# One Click 非 Owner 链路部署与验证记录

## 目的

本文记录当前 One Click 两节点环境的部署方式和非 owner 链路验证方式，用于和 K8S 场景
`CUBEVS_HOSTPORT_CHECKSUM` 问题进行对比。

重点对比链路：

```text
client / curl
  -> control CubeProxy
  -> compute HostIP:hostPort
  -> compute CubeVS
  -> compute SandboxIP:containerPort
```

## 登录方式

跳板机通过 SSH key 登录，不需要密码：

```bash
ssh root@106.53.31.91
```

从跳板机登录两个 Cube 节点时使用密码：

```bash
# control 节点
ssh root@10.2.122.117
# password: <omitted>

# compute 节点
ssh root@10.2.122.65
# password: <omitted>
```

节点角色：

```text
control: 10.2.122.117
compute: 10.2.122.65
```

注意：当前 One Click 标准拓扑中，CubeProxy 只部署在 control 节点。compute 节点不监听
80 端口，已确认：

```text
http://10.2.122.65/ -> connection refused
```

## 部署包来源

本次部署不是使用 checksum fix 后的 CubeNet，而是先还原 CubeNet 相关改动，再重新编译并打包。

已还原的文件：

```text
CubeNet/src/tcp.h
CubeNet/cubevs/mvmtap_x86_bpfel.o
```

重新编译后生成的 compat One Click 包：

```text
deploy/one-click/.work/cube-sandbox-one-click-v0.4.0-compat-el8.tar.gz
sha256: e61fd6c5a83d2ad6396eb20ff9b9702c6205e3417297e9b98eaae0374b9426ab
```

两台节点上部署后的 `network-agent` 和本地重新编译产物一致：

```text
sha256: b07dce8dbe9e622008ef47ec5d49550facea530e4a1e061d47e2205f12cb8518
BuildID: 7097032c822b2317e2cb3ff886f811951bee2d87
```

## 组件分布

control `10.2.122.117` 部署组件：

```text
cubemaster
cubemastercli
cube-api
cubelet
network-agent
cube-runtime
containerd-shim-cube-rs
cube-proxy
coredns
redis
mysql
webui
cube-egress
```

compute `10.2.122.65` 部署组件：

```text
cubelet
network-agent
cube-runtime
containerd-shim-cube-rs
cube-agent
```

两台节点均已安装 PVM host kernel，并通过 One Click smoke：

```text
control smoke: OK
compute smoke: OK
CubeMaster node view: both healthy=true, Ready=True
```

## 部署步骤

将 compat One Click 包上传到跳板机：

```bash
scp deploy/one-click/.work/cube-sandbox-one-click-v0.4.0-compat-el8.tar.gz \
  root@106.53.31.91:/root/cube-install/
```

从跳板机分发到两个 Cube 节点：

```bash
scp /root/cube-install/cube-sandbox-one-click-v0.4.0-compat-el8.tar.gz \
  root@10.2.122.117:/root/cube-install/

scp /root/cube-install/cube-sandbox-one-click-v0.4.0-compat-el8.tar.gz \
  root@10.2.122.65:/root/cube-install/
```

在目标节点执行部署：

```bash
# control
/root/cube-install/cube_run_oneclick.sh control 10.2.122.117

# compute
/root/cube-install/cube_run_oneclick.sh compute 10.2.122.65 10.2.122.117
```

部署后在 control 上应用了两个 runtime 运维补丁：

```text
1. DNS backend patch
   处理 resolvectl 存在但 systemd-resolved 未启用导致 dns/coredns quickcheck 失败的问题。

2. CubeProxy Redis key compatibility patch
   v0.4.0 CubeProxy Lua 只读取 legacy Redis key；
   当前 CubeMaster 写入 namespaced key。
   该 patch 只影响 CubeProxy 读取路由元数据，不修改 CubeVS/checksum 数据面。
```

## 当前 Sandbox

control sandbox：

```text
sandbox_id: f63a54d46c4546bb9b6dc59c129d2dc8
HostIP:     10.2.122.117
SandboxIP:  192.168.0.6
HostPort:   10.2.122.117:20001 -> 49999
```

compute sandbox：

```text
sandbox_id: de6cf867cea54d949e5d0a1f32d23e6a
HostIP:     10.2.122.65
SandboxIP:  192.168.0.64
HostPort:   10.2.122.65:20001 -> 49999
```

## 重点验证链路

本次重点关注非 owner 链路。One Click 当前可验证的非 owner 链路是：

```text
client / curl
  -> control CubeProxy 10.2.122.117:80
  -> compute HostIP:hostPort 10.2.122.65:20001
  -> compute CubeVS
  -> compute sandbox 192.168.0.64:49999
```

与 K8S 报告的 `sslip.io` host-based CubeProxy URL 形式对齐，访问 compute sandbox：

```bash
curl -sS -m 5 -D - \
  http://49999-de6cf867cea54d949e5d0a1f32d23e6a.10.2.122.117.sslip.io/health
```

验证结果：

```text
HTTP/1.1 200 OK
"OK"
```

重复访问统计：

```text
http://49999-de6cf867cea54d949e5d0a1f32d23e6a.10.2.122.117.sslip.io/health
50/50 OK
```

同一个 control CubeProxy 入口访问 control sandbox 也正常：

```bash
curl -sS -m 5 -D - \
  http://49999-f63a54d46c4546bb9b6dc59c129d2dc8.10.2.122.117.sslip.io/health
```

验证结果：

```text
HTTP/1.1 200 OK
"OK"
```

重复访问统计：

```text
http://49999-f63a54d46c4546bb9b6dc59c129d2dc8.10.2.122.117.sslip.io/health
50/50 OK
```

辅助 path-based CubeProxy URL 也正常：

```bash
curl -sS -m 5 -D - \
  http://10.2.122.117/sandbox/de6cf867cea54d949e5d0a1f32d23e6a/49999/health
```

结果：

```text
50/50 OK
```

## 与 K8S 场景的差异

K8S 报告中的验证是：

```text
同一个 sandbox:
  owner 节点 CubeProxy -> 200
  非 owner 节点 CubeProxy -> owner HostIP:hostPort -> 504/200
```

当前 One Click 标准环境没有 compute 节点 CubeProxy，因此不能做 owner CubeProxy 与
non-owner CubeProxy 的双入口对照。

当前已验证的是 One Click 标准模型中的非 owner 链路：

```text
control CubeProxy, 非 owner
  -> compute owner HostIP:hostPort
  -> compute CubeVS
  -> compute sandbox
```

该链路在还原 CubeNet 后重新编译的组件上正常，未复现 504/checksum 问题。
