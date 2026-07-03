# Cube On K8S 非 Owner CubeProxy 504 根因与修复报告

## 结论

K8S 非 owner CubeProxy 访问 sandbox 返回 504 的直接原因是：`cube-node:v0.4.0` 中的 network-agent/CubeVS BPF 产物来自官方 One Click v0.4.0 包，而该包构建于 hostPort checksum 修复之前，跨节点 hostPort 回包在 owner 节点离开 `eth0` 时 TCP checksum 仍然错误，客户端节点直接丢包，三次握手无法完成。

这不是 TKE 安全组、`fileInject`、CubeProxy Redis 路由字段或 `proxyBaseURL` 唯一性的根因。之前验证中“One Click 能正常工作”的环境运行的是本地后续重新构建过的 `_output/compat-el8` 兼容产物，不是官方 2026-06-15 构建的 One Click v0.4.0 原始包；K8S Chart 当时拉取的旧 `cube-node:v0.4.0` 与官方 One Click v0.4.0 包组件 hash 完全一致，所以真正差异是官方 v0.4.0 产物和 2026-06-22/23 之后的修复产物不同。

旧报告 `CUBEVS_HOSTPORT_CHECKSUM_FIX_REPORT.md` 中“删除 TCP 源端口 checksum 更新”的判断不再作为最终方案。当前源码正确形态是保留端口 checksum 更新，但 helper flags 不能带 `BPF_F_PSEUDO_HDR`：

```c
/* update TCP csum for port change (not part of pseudo-header) */
flags = sizeof(listen_port);
err = bpf_l4_csum_replace(skb, offset, listen_port, host_port, flags);
```

对应提交为 `7c2dd1f CubeVS: fix invalid TCP checksum on port-mapped sandbox replies`，随后 `07295df CubeVS: Regenerate BPF objects` 重新生成了 BPF object。K8S 旧镜像没有运行到等价产物。

## 请求路径

目标架构保持 One Click 语义：

```text
Browser / Agent client
  -> control or selected CubeProxy
  -> compute HostIP:hostPort
  -> compute CubeVS
  -> SandboxIP:containerPort
```

CubeProxy 内置逻辑会从 Redis 读取 `bypass_host_proxy:<sandboxId>` 元数据。如果当前 CubeProxy 节点 IP 等于 `HostIP`，直接转发到 `SandboxIP:containerPort`；否则转发到 `HostIP:hostPort`。代码位置：

- `CubeProxy/lua/sandbox_backend.lua`: `resolve_backend()` 读取 Redis 元数据。
- `CubeProxy/lua/sandbox_backend.lua`: owner 分支使用 `SandboxIP:containerPort`，非 owner 分支使用 `HostIP:hostPort`。

network-agent 在创建 sandbox 网络时把端口映射写入 CubeVS map：

- `network-agent/internal/service/tap_lifecycle.go`: `configurePortMappings()` 调用 `cubevsAddPortMap()`，并把 `HostIP` 写成节点 IP。

CubeVS 跨节点 hostPort 回包路径：

- `CubeNet/src/mvmtap.bpf.c`: `from_cube()` 在 sandbox TAP ingress 上处理回包。
- `CubeNet/src/mvmtap.bpf.c`: 命中 `local_port_mapping` 后调用 `snat_tcp()`，再 `bpf_redirect(nodenic_ifindex, 0)`。
- `CubeNet/src/tcp.h`: `snat_tcp()` 更新源 IP、源端口和 L3/L4 checksum。

## 现场对比

K8S 旧镜像：

```text
image: ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0
image digest: sha256:95d62f0ef4fecec561e05f3e547f19e8272f509e1c1db115557b93a82abbdab5
network-agent sha256: 9c56efe20fc476a289d902e4ba382dcd8fa4dcd9beab57b88d9b1ec0cdeaf688
cubevsmapdump sha256: 3923b84b907b8baacd996a27becb8e37b19dd0ddf645dc15dcc15be365a94716
```

官方 One Click v0.4.0 包：

```text
package: deploy/images/.work/downloads/cube-sandbox-one-click-v0.4.0.tar.gz
release_version: v0.4.0
git_commit: 4004a6ec34a9d045a9789a1fd438d6518eedb3d3
built_at: 2026-06-15T12:43:09Z
network-agent sha256: 9c56efe20fc476a289d902e4ba382dcd8fa4dcd9beab57b88d9b1ec0cdeaf688
cubevsmapdump sha256: 3923b84b907b8baacd996a27becb8e37b19dd0ddf645dc15dcc15be365a94716
```

旧 K8S 镜像与官方 One Click v0.4.0 包组件 hash 一致，说明 `cube-node:v0.4.0` 确实是基于官方 One Click v0.4.0 包组件打出来的。

之前工作正常的 One Click 验证环境：

```text
control: 10.2.122.117
compute: 10.2.122.65
network-agent sha256: b07dce8dbe9e622008ef47ec5d49550facea530e4a1e061d47e2205f12cb8518
cubevsmapdump sha256: 77ee821a99312633ababd844cd4632d25973259596d03f21c4c7dff0d69af9ba
non-owner/control CubeProxy -> compute HostIP:hostPort: 200
```

结论：出问题时，K8S 旧镜像和官方 One Click v0.4.0 包是同一批组件产物；之前不复现的 One Click 环境不是这批官方原始产物，而是后续从当前代码重建的兼容产物。

## 修复进入时间

官方 v0.4.0 tag / package 停在：

```text
commit: 4004a6ec34a9d045a9789a1fd438d6518eedb3d3
package built_at: 2026-06-15T12:43:09Z
```

hostPort checksum 源码修复在 v0.4.0 之后进入：

```text
7c2dd1f 2026-06-22 19:26:15 +0800 CubeVS: fix invalid TCP checksum on port-mapped sandbox replies
07295df 2026-06-23 09:58:33 +0800 CubeVS: Regenerate BPF objects
```

两者之间的关键代码差异只有 `CubeNet/src/tcp.h` 中端口 checksum 更新的 helper flags：

```diff
- flags = BPF_F_PSEUDO_HDR | sizeof(listen_port);
+ /* update TCP csum for port change (not part of pseudo-header) */
+ flags = sizeof(listen_port);
```

端口不属于 TCP pseudo-header，旧 v0.4.0 把 `BPF_F_PSEUDO_HDR` 用在端口增量更新上，会让跨节点 hostPort 回包 checksum 错误。修复后还必须重新生成 `CubeNet/cubevs/*_bpfel.o`，否则 network-agent 里嵌入的 BPF object 仍是旧逻辑。

## 抓包证据

旧 K8S 复现 Agent：

```text
agent: cube-checksum-regression
sandboxId: 1f8a90b9896d49a6991bca1fd321ad37
HostIP: 10.2.5.238
SandboxIP: 192.168.1.40
49999 -> 20001
```

访问结果：

```text
non-owner proxy 10.2.5.138 -> 504
owner proxy     10.2.5.238 -> 200
```

旧 K8S 抓包要点：

```text
10.2.5.138 -> 10.2.5.238:20001 SYN: checksum correct
10.2.5.238:20001 -> 10.2.5.138 SYN-ACK: checksum incorrect
client never ACKs
```

One Click 抓包要点：

```text
10.2.122.117 -> 10.2.122.65:20001 SYN: checksum correct
10.2.122.65:20001 -> 10.2.122.117 SYN-ACK: checksum correct
HTTP 200 "OK"
```

两边 sandbox 侧都能看到 sandbox 原始回包 checksum 呈现 partial/offload 形态；差异发生在 CubeVS hostPort SNAT 后离开 owner 节点时：旧 K8S 产物输出 incorrect，One Click/新产物输出 correct。

两边节点 `eth0` offload 配置一致，安全组也已确认放通，因此不是系统 offload 或安全组导致。

## 已实施修复

基于当前源码重新生成 CubeVS BPF object，并重新构建 network-agent 与 cubevsmapdump：

```bash
go generate ./...
make build
go build -o ../../network-agent/bin/cubevsmapdump ./cmd/cubevsmapdump
```

将两个二进制替换进 `cube-node:v0.4.0` 基础镜像后推送验证镜像：

```text
ccr.ccs.tencentyun.com/pavleli/cube-node:samecode-20260627124356
digest: sha256:9fd575a09d2cdbb5eea6904796d8e7c504f88c59c37a806c8048347d87f3697e
```

为 Chart 使用补了稳定验证 tag：

```text
ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0-cubevsfix-20260627
digest: sha256:9fd575a09d2cdbb5eea6904796d8e7c504f88c59c37a806c8048347d87f3697e
```

Chart 默认值已更新：

```yaml
images:
  node:
    repository: ccr.ccs.tencentyun.com/pavleli/cube-node
    tag: v0.4.0-cubevsfix-20260627
    pullPolicy: Always
```

当前 live 环境也已滚动到该修复产物。

## 修复验证

当前 K8S compute 节点：

```text
10.2.5.138
10.2.5.238
```

`cube-node` DaemonSet：

```text
image: ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0-cubevsfix-20260627
desired: 2
ready: 2
updated: 2
```

Pod 内二进制 hash：

```text
network-agent: 9a6de4b7021115259086947ad3c9e940a7dfd25f560d313e917de014d8719a6e
cubevsmapdump: 19f4a7d03a3e979079ba3583360517e77d74146ebb9beb8f04bc09f44b744bd5
```

最终 fresh Agent：

```text
namespace: agent-way-system
name: cube-chartfix-verify
sandboxId: 66d957886a5d4d9b8dcd989d9e3d0637
HostIP: 10.2.5.138
SandboxIP: 192.168.0.71
49999 -> 20005
49983 -> 20004
```

访问结果，连续 5 次 owner / non-owner 都成功：

```text
owner     10.2.5.138: http=200 body="OK"
non-owner 10.2.5.238: http=200 body="OK"
```

修复后 owner 节点 `eth0` 抓包：

```text
10.2.5.238 -> 10.2.5.138:20005 SYN: checksum correct
10.2.5.138:20005 -> 10.2.5.238 SYN-ACK: checksum correct
10.2.5.138:20005 -> 10.2.5.238 HTTP response packets: checksum correct
```

这证明 `Browser -> CubeProxy -> compute HostIP:hostPort -> CubeVS -> SandboxIP:containerPort` 链路在 K8S 下已经恢复，不再触发旧的 hostPort 回包 checksum 问题。

## 本地测试

```text
CubeNet/cubevs:
  go test ./... passed
  coverage total: 11.6%
  html: /tmp/cubevs-samecode-coverage.html

network-agent:
  go test ./... passed
  coverage total: 41.4%
  html: /tmp/network-agent-samecode-coverage.html
```

## 最终修复方案

1. Chart 不再使用旧 `cube-node:v0.4.0` 作为默认 node 镜像，改为已验证的 `v0.4.0-cubevsfix-20260627`。
2. 正式发布时，应把 One Click package 和 K8S `cube-node` 镜像纳入同一构建产物链路：先生成 CubeVS BPF object，再构建 network-agent/cubevsmapdump，再把同一组二进制打进 One Click package 与 Chart 镜像。
3. 每次涉及 `CubeNet/src/*.bpf.c` 或 `CubeNet/src/*.h` 后，必须执行 `go generate ./...` 并确认 `CubeNet/cubevs/*_bpfel.o` 已同步，否则源码修复不会进入运行时 BPF object。
4. Chart 发布后必须新建 fresh Agent 验证 owner / non-owner 两条链路；滚动 `cube-node` 后旧 Agent 可能出现 `NotFoundAtCubelet`，不能作为唯一验收对象。
5. 保持 One Click 路由语义：CubeProxy 根据 Redis 元数据选择本地 `SandboxIP:containerPort` 或跨节点 `HostIP:hostPort`，不要再引入 `cube-proxy-node` ClusterIP Service 分流。

## 验收步骤

在本仓库执行远程命令时使用 `rtk` 前缀：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get ds cube-node -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,READY:.status.numberReady,DESIRED:.status.desiredNumberScheduled,UPDATED:.status.updatedNumberScheduled'
```

创建 fresh Agent 后查 Redis：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'REDIS=$(kubectl -n cube-system get pod -l app.kubernetes.io/component=redis -o jsonpath="{.items[0].metadata.name}"); kubectl -n cube-system exec "$REDIS" -- sh -ec '\''redis-cli -a "$REDIS_PASSWORD" hgetall bypass_host_proxy:66d957886a5d4d9b8dcd989d9e3d0637'\'''
```

分别访问 owner / non-owner：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'curl -sS -o /tmp/non_owner_body -w "non_owner_http=%{http_code} total=%{time_total}\n" --max-time 8 http://49999-66d957886a5d4d9b8dcd989d9e3d0637.10.2.5.238.sslip.io/health; cat /tmp/non_owner_body; echo'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'curl -sS -o /tmp/owner_body -w "owner_http=%{http_code} total=%{time_total}\n" --max-time 8 http://49999-66d957886a5d4d9b8dcd989d9e3d0637.10.2.5.138.sslip.io/health; cat /tmp/owner_body; echo'
```

期望结果：

```text
non_owner_http=200
owner_http=200
body: "OK"
```
