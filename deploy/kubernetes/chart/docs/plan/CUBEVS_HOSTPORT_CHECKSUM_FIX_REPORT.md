# CubeVS HostPort Checksum 修复结论

## 背景

在按 One Click 方案调整 Cube Chart 时，目标数据链路为：

```text
Browser
  -> control CubeProxy
  -> compute HostIP:hostPort
  -> compute CubeVS
  -> SandboxIP:containerPort
```

实测发现跨节点访问会失败：

- sandbox owner 节点 `10.2.5.238` 本地 CubeProxy 访问返回 `200`。
- 非 owner 节点 `10.2.5.138` CubeProxy 访问同一 sandbox 返回 `504`。
- 安全组已经确认放通，不是 TKE 安全组问题。
- 抓包显示 owner 节点能收到来自非 owner 节点的 SYN，也能通过 CubeVS DNAT 到 sandbox；sandbox 返回 SYN-ACK 后，客户端侧看到 checksum 异常，TCP 三次握手无法完成。

## 结论

根因不是 `fileInject` 字段，也不是 CubeProxy Redis 元数据解析错误，而是 CubeVS hostPort 回包路径的 TCP checksum 修正逻辑在部分内核 checksum offload 状态下重复计算了 source port delta。

具体表现是：

1. 跨节点 CubeProxy 根据 Redis 元数据找到 sandbox owner 节点。
2. 非 owner CubeProxy 转发到 `HostIP:hostPort`。
3. owner 节点 CubeVS 将 `hostPort` 转发到 `SandboxIP:containerPort`。
4. sandbox 回包经 CubeVS SNAT 回 `HostIP:hostPort`。
5. CubeVS 在 SNAT source port 时额外调用一次非 pseudo-header 的 `bpf_l4_csum_replace()`。
6. 由于该回包带 checksum offload 状态，最终 checksum 引擎会基于已改写的 TCP header 再计入 source port，导致 source port delta 被计算两次。
7. 客户端收到 checksum 错误的 SYN-ACK，不发送 ACK，CubeProxy 最终返回 `504`。

之前抓包中 checksum 差值与 `hostPort - listenPort` 对应，也支持这个判断。

## 修复方案

只修 CubeVS hostPort 回包路径，保持改动最小。

核心改动：

- 文件：`CubeNet/src/tcp.h`
- 函数：`snat_tcp()`
- 行为：保留 source IP 的 checksum 修正；去掉 source port 改写前额外的 `bpf_l4_csum_replace()`；仍通过 `bpf_skb_store_bytes()` 将 TCP source port 写成 hostPort。

修复后语义：

```text
sandbox reply:
  SandboxIP:containerPort
    -> CubeVS snat_tcp()
    -> HostIP:hostPort
    -> remote CubeProxy / client
```

source port 的最终 checksum 交给内核最终 checksum 引擎基于改写后的 TCP header 处理，避免重复叠加 port delta。

重新生成的 BPF object：

- `CubeNet/cubevs/mvmtap_x86_bpfel.o`

## 验证拓扑

测试集群通过跳板机访问，命名空间为 `cube-system` / `agent-way-system`。

`cube-node` DaemonSet 热修镜像：

```text
ccr.ccs.tencentyun.com/pavleli/cube-node:checksumfix-20260627144604
```

DaemonSet 状态：

```text
cube-node READY 2 / DESIRED 2
```

新建 smoke Agent：

```text
agent-way-system/cube-checksum-smoke
sandboxId: 9e10d404c8074c278c88c414aa515ba2
HostIP: 10.2.5.138
SandboxIP: 192.168.0.167
port mapping: 49999 -> 20003
```

Redis 路由元数据：

```text
bypass_host_proxy:9e10d404c8074c278c88c414aa515ba2
HostIP    10.2.5.138
SandboxIP 192.168.0.167
49999     20003
```

## 访问验证

owner 节点 CubeProxy：

```text
http://49999-9e10d404c8074c278c88c414aa515ba2.10.2.5.138.sslip.io/health
=> 200, "OK"
```

非 owner 节点 CubeProxy，强制跨节点访问 owner hostPort：

```text
http://49999-9e10d404c8074c278c88c414aa515ba2.10.2.5.238.sslip.io/health
=> 200, "OK"
```

这证明以下链路已恢复：

```text
Browser / curl
  -> 10.2.5.238 CubeProxy
  -> 10.2.5.138:20003 hostPort
  -> 10.2.5.138 CubeVS
  -> 192.168.0.167:49999 sandbox
```

## 本地验证

已执行：

```text
CubeNet/cubevs go test: 39 passed, coverage 11.6%
network-agent make build: success
network-agent go test: 87 passed, coverage 41.4%
git diff --check: passed
```

## 对 Chart 方案的影响

该 bug 修复后，Chart 可以继续向 One Click 的数据面模型收敛：

```text
Browser
  -> control CubeProxy / wildcard DNS
  -> compute HostIP:hostPort
  -> compute CubeVS
  -> SandboxIP:containerPort
```

Chart 层仍应遵循 One Click 的分工：

- CubeProxy / DNS 使用 control 节点 labelSelector 部署。
- CubeProxy 不通过 `ClusterIP Service` 做随机分流，避免产生与 One Click 不一致的第二套数据面语义。
- compute 节点只承载 cube-node、CubeVS、cubelet、network-agent 和 sandbox。
- CubeProxy 根据 Redis 中的 `HostIP`、`SandboxIP`、`containerPort -> hostPort` 元数据转发。

## 注意事项

旧 Agent 在滚动 `cube-node` 后可能出现 Cubelet 侧 `NotFoundAtCubelet`，不能再作为 checksum 修复的验证对象。本次结论以新建的 `cube-checksum-smoke` Agent 为准。

该修复当前只覆盖 hostPort 回包路径，未扩大到其它 NAT 路径，避免引入无关行为变化。
