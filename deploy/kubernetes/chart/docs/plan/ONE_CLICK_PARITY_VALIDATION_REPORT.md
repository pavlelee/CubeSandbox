# CubeSandbox Chart One Click 对齐验证报告

## 验证环境

- 验证日期：2026-06-26
- 集群操作入口：验证跳板机
- Namespace：`cube-system`
- Helm release：`cube`
- 最终 release revision：`24`
- Chart 基线 commit：`0d6b55a`
- Chart 状态：本报告生成时为待提交工作树，提交后以对应 commit 为准
- 镜像 tag：`v0.4.0`
- 计算节点 label selector：

```yaml
cube.tencent.com/role: compute
cube.tencent.com/cube-node: "true"
cube.tencent.com/allow-pvm-bootstrap: "true"
```

> 说明：`allow-pvm-bootstrap` 是本验证集群已有节点标签，Chart 不负责给节点打标签。

## 验证结论

通过。

本轮按 `FINAL_ONE_CLICK_PARITY_PLAN.md` 对 Chart 做了完整实现，并在集群上完成：

1. 默认完整 Chart 部署验证。
2. `cube-dns` node-local 对 `cube-node` Pod 生效验证。
3. MySQL / Redis 内置持久化和健康验证。
4. CubeMaster / CubeAPI / WebUI / CubeProxy / CubeNode / CubeEgress / cube-egress-net Helm test 与探针验证。
5. external control plane / compute-only 模式集群验证。
6. CIDR 检测脚本合法、非法、冲突、跳过四类场景验证。
7. 第三方 MySQL / Redis 渲染验证。
8. 禁止 demo IP、nodePrepare、`cube-db-migrate` 渲染验证；`cubemastercli` 仅允许通过独立 `cubemastercli` 运维 Deployment 交付，禁止伪造 `ctl` wrapper。

## 本地静态验证

### helm lint

```bash
helm lint deploy/k8s/chart
```

结果：

```text
1 chart(s) linted, 0 chart(s) failed
```

### helm template

```bash
helm template cube deploy/k8s/chart -n cube-system
```

结果：通过。

额外检查：

- 未渲染 demo IP。
- 未渲染 nodePrepare / node-prepare。
- 未渲染 `cube-db-migrate`。
- 未在 Master/Node 运行镜像或资源中混入 `cubemastercli`；独立 `cubemastercli` 运维 Deployment 是预期交付资源。
- `mysql.host` / `redis.host` 配置第三方服务时，未渲染内置 MySQL / Redis Deployment、Service、PVC。
- `controlPlane.enabled=false` + `externalControlPlane.enabled=true` 时，仅渲染 compute-only 所需的 `cube-node` 和 `cube-dns` 等资源，不渲染内置 Master / API / MySQL / Redis / WebUI / Proxy。

### CIDR 脚本验证

命令覆盖：

```bash
sh deploy/images/scripts/cube-node-init.sh
```

通过场景：

- `198.18.0.0/15`：合法且不冲突，init 通过。
- `192.168.1.1/24`：非网络地址，失败并提示 `did you mean 192.168.1.0/24`。
- `127.0.0.0/8`：与 host interface 冲突，失败并输出冲突明细。
- `127.0.0.0/8` + `CUBE_SANDBOX_NETWORK_CIDR_SKIP_CONFLICT_CHECK=1`：仅跳过冲突检测，init 通过。

## 镜像验证

最终集群拉取的核心镜像均为规范 tag `v0.4.0`。

`cube-node` Pod 中验证到：

```text
cube-node-init ccr.ccs.tencentyun.com/pavleli/cube-node-init:v0.4.0
  sha256:2a9e1114db8df884659292723c1df6f5a1b94b4b7157cf41de7f6e689945207d
cube-node ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0
  sha256:95d62f0ef4fecec561e05f3e547f19e8272f509e1c1db115557b93a82abbdab5
cube-egress ccr.ccs.tencentyun.com/pavleli/cube-egress:v0.4.0
  sha256:c8846b65577b4cfc1d393969e692ed944edadee7988d6ade547cdfd5bb99e16f
cube-egress-net ccr.ccs.tencentyun.com/pavleli/cube-egress-net:v0.4.0
  sha256:c4f9a038a040edd9c899a7f17cbd75424551aa062b0f91cbdbd71ee53f7661df
```

## 完整 Chart 集群验证

### 安装 / 升级命令

```bash
helm upgrade --install cube /tmp/cube-chart-current/chart \
  -n cube-system \
  --create-namespace \
  -f /tmp/cube-validation-values.yaml \
  --wait \
  --timeout 30m
```

验证集群中 `/tmp/cube-validation-values.yaml`：

```yaml
cubeNode:
  updateStrategy:
    type: RollingUpdate
  pvmGuestKernel:
    enabled: true
placement:
  nodeLabelSelector:
    cube.tencent.com/role: compute
    cube.tencent.com/cube-node: "true"
    cube.tencent.com/allow-pvm-bootstrap: "true"
bootstrap:
  pvmHostKernel:
    enabled: false
```

结果：

- Helm release：`deployed`
- 最终 revision：`24`
- 所有 Deployment rollout 成功。
- 所有 DaemonSet rollout 成功。

### Pod / DaemonSet / PVC 状态

最终关键状态：

```text
deployment/cube-master  1/1 Available
deployment/cube-api     1/1 Available
deployment/cube-mysql   1/1 Available
deployment/cube-redis   1/1 Available
deployment/cube-webui   1/1 Available

daemonset/cube-dns        2/2 Ready
daemonset/cube-node       2/2 Ready
daemonset/cube-proxy-node 2/2 Ready

pvc/cube-master-storage Bound 20Gi
pvc/cube-mysql-data     Bound 20Gi
pvc/cube-redis-data     Bound 10Gi
```

`cube-node` 只调度到命中 label 的两个计算节点。

### DNS 验证

在 `cube-node` Pod 内验证：

```text
search cube-system.svc.cluster.local svc.cluster.local cluster.local
nameserver 127.0.0.54
options ndots:5
```

解析结果：

```text
cube.app                         -> 当前节点 HostIP
wildcard-check.cube.app          -> 当前节点 HostIP
kubernetes.default.svc.cluster.local -> ClusterIP
```

说明 `cube-dns` node-local 模式已直接对 `cube-node` DaemonSet 生效，同时不破坏集群 Service 域名解析。

### DB 验证

```bash
kubectl exec -n cube-system deploy/cube-mysql -- \
  mysqladmin ping -h 127.0.0.1 -ucube -pcube_pass --silent
kubectl exec -n cube-system deploy/cube-redis -- \
  redis-cli -a ceuhvu123 ping
```

结果：

```text
mysqld is alive
PONG
```

### Node runtime 验证

在 `cube-node` Pod 内验证：

```bash
test -e /dev/kvm
test -S /data/cubelet/cubelet.sock
test -S /tmp/cube/network-agent-grpc.sock
test -f /usr/local/services/cubetoolbox/Cubelet/config/config.toml
test -f /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux
test -f /usr/local/services/cubetoolbox/cube-image/cube-guest-image-cpu.img
```

结果：全部通过。

当前验证集群启用 PVM guest kernel，`vmlinux` 指向：

```text
/usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux-pvm
```

### Helm test

```bash
helm test cube -n cube-system --timeout 20m --logs
```

最终结果：

```text
TEST SUITE: cube-health-test        Phase: Succeeded
TEST SUITE: cube-mysql-test         Phase: Succeeded
TEST SUITE: cube-redis-test         Phase: Succeeded
TEST SUITE: cube-dns-test           Phase: Succeeded
TEST SUITE: cube-node-image-test    Phase: Succeeded
TEST SUITE: cube-node-runtime-test  Phase: Succeeded
```

覆盖项：

- CubeMaster `/notify/health`
- CubeAPI `/health`
- Cube Node 注册列表
- WebUI `/` 和 `/cubeapi/v1/health`
- CubeProxy HTTP / HTTPS 数据面
- CubeNode DaemonSet ready 状态
- CubeEgress / cube-egress-net 容器存在性
- cube-egress-net 规则级 readiness/liveness 探针命令通过
- MySQL / Redis 健康
- CubeDNS `cube.app`、wildcard、Kubernetes Service 域名解析
- cube-node 镜像内 runtime asset
- 计算节点 host runtime socket / KVM

### cube-egress-net 探针验证

`cube-egress-net` 增加 readiness/liveness exec 探针，检查：

- `cube-dev` interface 存在；
- `ip rule` 包含 `cube-dev` 上 tcp/80、tcp/443 到 table 100 的规则；
- table 100 包含 local route 到 `lo`；
- mangle `TRANSPROXY` chain 包含 80/443 TPROXY 规则。

集群内手工执行同等探针命令结果：

```text
ok
```

同时为 `cube-node` 增加 startupProbe，避免慢节点启动期间 liveness 在 cubelet 9999 端口未就绪前提前杀死容器。最终 `cube-node` DaemonSet `2/2 Ready`。

## external control plane / compute-only 集群验证

### 控制面 release

临时安装 `cube-control` release 到 `cube-control` namespace：

```bash
helm upgrade --install cube-control /tmp/cube-chart-current/chart \
  -n cube-control \
  --create-namespace \
  -f /tmp/cube-control-values.yaml \
  --wait \
  --timeout 30m
```

`cube-control` 禁用 Node / Proxy / DNS / Egress，仅部署控制面：

```yaml
cubeNode:
  enabled: false
cubeProxy:
  enabled: false
cubeDns:
  enabled: false
cubeEgress:
  enabled: false
placement:
  nodeLabelSelector: {}
```

`helm test cube-control -n cube-control --timeout 20m --logs` 通过：

```text
cube-control-health-test Phase: Succeeded
cube-control-mysql-test  Phase: Succeeded
cube-control-redis-test  Phase: Succeeded
```

### compute-only release

将 `cube` release 切换为 external control plane：

```bash
helm upgrade --install cube /tmp/cube-chart-current/chart \
  -n cube-system \
  -f /tmp/cube-compute-values.yaml \
  --wait \
  --timeout 30m
```

关键 values：

```yaml
controlPlane:
  enabled: false
externalControlPlane:
  enabled: true
  masterEndpoint: cube-control-master.cube-control.svc.cluster.local:8089
  apiEndpoint: http://cube-control-api.cube-control.svc.cluster.local:3000
```

验证结果：

- `cube-system` 中只保留 compute-only 所需 `cube-dns` 和 `cube-node` 等资源。
- 不安装内置 Master / API / MySQL / Redis / WebUI / Proxy。
- `cube-node` init connectivity check 通过。
- `helm test cube -n cube-system --timeout 20m --logs` 通过。
- 外部 `cube-control-api` 查询到两个 healthy Cube Node。

完成 external control plane 验证后，已恢复 `cube-system` 到完整 Chart 形态，并卸载临时 `cube-control` release，删除临时 namespace。

## 遗留问题

无阻塞遗留问题。

注意项：

- 验证用 test hook Pod 采用 `before-hook-creation` 删除策略，便于保留最近一次 `helm test --logs` 的 Pod 日志；下一次 `helm test` 会自动清理并重建。
- PVM Host Kernel bootstrap 能力保留且当前默认开启；只有带 `cube.tencent.com/allow-pvm-bootstrap=true` 的计算节点会被选中。
- Chart 不修改宿主机 DNS，不给节点打标签；节点规划和外部 DNS 接入仍由使用方负责。
