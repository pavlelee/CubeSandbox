# CubeSandbox Chart 还原 One Click 能力最终改造方案

## 1. 目标

将 `deploy/k8s/chart` 改造成在 Kubernetes/TKE 集群中尽可能完整还原
`deploy/images/.work/extract/cube-sandbox-one-click-v0.4.0` One Click 包能力的
Chart 交付形态。

约束：

- 不在 Chart 中自动给节点打 label，计算节点 label 由使用方提前打好。
- 不引入 demo 节点 IP、demo SQL seed、演示数据。
- 不交付独立 `cube-db-migrate` 镜像；数据库 schema 迁移由 CubeMaster 内置逻辑完成。
- `cubemastercli` 只允许通过独立 `cubemastercli` 运维镜像交付，禁止混入 Master/Node 运行镜像，禁止伪造 `ctl` wrapper。
- 初始化镜像与运行镜像职责分离。
- Master、Node、Proxy、Egress、WebUI 镜像职责分离。
- `cube-node` 镜像只包含 Cube Node 所需组件。
- MySQL / Redis 支持第三方服务；配置第三方服务时不安装内置 MySQL / Redis。
- 后续实现和验证不能通过缓存、hack、跳过构建/拉取等方式规避问题。

## 2. 总体改造项

本轮最终改造包含 6 类能力：

1. `cube-dns` 直接对 `cube-node` DaemonSet 生效。
2. MySQL / Redis 默认值、持久化、端口、健康检查整体对齐 One Click。
3. readiness / liveness / Helm test 增强。
4. external control plane 支持，覆盖 One Click compute-only 场景。
5. CIDR 冲突检测对齐 One Click。
6. `values.yaml` 中已识别的默认值全部调成 One Click 默认行为。

## 3. cube-dns 对 cube-node DaemonSet 生效方案

### 3.1 结论

可以让 Chart 提供的 `cube-dns` 直接对 `cube-node` DaemonSet 生效，但不能依赖
ClusterIP Service 自动生效。Kubernetes 不会自动让 Pod 使用 Chart 自带 DNS Service。

最终采用 **node-local cube-dns**：

- `cube-dns` 以 DaemonSet 跑在 Cube 计算节点。
- `cube-dns` 使用 `hostNetwork: true`。
- `cube-dns` 监听 `127.0.0.54:53`，对齐 One Click 默认
  `CUBE_PROXY_COREDNS_BIND_ADDR=127.0.0.54`。
- `cube-node` Pod 使用 `dnsPolicy: None`。
- `cube-node` Pod 显式设置 `dnsConfig.nameservers: [127.0.0.54]`。
- 只影响 `cube-node` Pod，不修改宿主机 `/etc/resolv.conf`、NetworkManager、
  systemd-resolved，也不影响非 Cube Pod。

### 3.2 values 设计

```yaml
cubeDns:
  enabled: true
  mode: nodeLocal
  bindAddress: 127.0.0.54
  domain: cube.app
  answerIP: ""
  forward:
    upstreams: []
  service:
    type: ClusterIP
    port: 53
```

```yaml
cubeProxy:
  hostNetwork: true
```

```yaml
cubeNode:
  dns:
    useCubeDns: true
    nameserver: 127.0.0.54
    clusterDomain: cluster.local
    waitTimeoutSeconds: 120
    checkImage:
      repository: busybox
      tag: "1.36"
      pullPolicy: IfNotPresent
```

### 3.3 DNS 解析行为

`cubeDns.mode=nodeLocal` 时：

- `cube.app` 返回当前节点 HostIP。
- `*.cube.app` 返回当前节点 HostIP。
- 这等价于 One Click 默认：`CUBE_PROXY_DNS_ANSWER_IP=${CUBE_SANDBOX_NODE_IP:-}`。
- 如果 `cubeDns.answerIP` 非空，则固定返回 `answerIP`。
- 其他域名转发到 `cubeDns.forward.upstreams`；为空时使用 `/etc/resolv.conf`。

### 3.4 cube-node Pod 配置

`cubeNode.dns.useCubeDns=true` 时，`cube-node` DaemonSet 生成：

```yaml
dnsPolicy: None
dnsConfig:
  nameservers:
    - 127.0.0.54
  searches:
    - <namespace>.svc.cluster.local
    - svc.cluster.local
    - cluster.local
  options:
    - name: ndots
      value: "5"
```

### 3.5 启动顺序

在 `cube-node` DaemonSet 增加 DNS wait initContainer：

- 等待 `127.0.0.54:53` 可用。
- 校验 `nslookup cube.app 127.0.0.54`。
- 校验 `nslookup wildcard-check.cube.app 127.0.0.54`。
- 校验 `nslookup kubernetes.default.svc.cluster.local 127.0.0.54`。

DNS 不可用时，`cube-node` 主容器不启动。

### 3.6 涉及文件

- `deploy/k8s/chart/templates/dns.yaml`
- `deploy/k8s/chart/templates/node-daemonset.yaml`
- `deploy/k8s/chart/templates/validate.yaml`
- `deploy/k8s/chart/values.yaml`

## 4. MySQL / Redis 整体对齐方案

### 4.1 默认值对齐

| 配置项 | 当前默认 | 改造后默认 | One Click 对应项 |
| --- | --- | --- | --- |
| `mysql.image.repository` | `mysql` | `mysql` | `mysql:8.0` |
| `mysql.image.tag` | `8.0` | `8.0` | `mysql:8.0` |
| `mysql.port` | `3306` | `3306` | `CUBE_SANDBOX_MYSQL_PORT=3306` |
| `mysql.database` | `cube_mvp` | `cube_mvp` | `CUBE_SANDBOX_MYSQL_DB=cube_mvp` |
| `mysql.user` | `cube` | `cube` | `CUBE_SANDBOX_MYSQL_USER=cube` |
| `mysql.password` | `cube` | `cube_pass` | `CUBE_SANDBOX_MYSQL_PASSWORD=cube_pass` |
| `mysql.rootPassword` | `cube-root` | `cube_root` | `CUBE_SANDBOX_MYSQL_ROOT_PASSWORD=cube_root` |
| `redis.image.repository` | `redis` | `redis` | `redis:7-alpine` |
| `redis.image.tag` | `7.2` | `7-alpine` | `redis:7-alpine` |
| `redis.port` | `6379` | `6379` | `CUBE_SANDBOX_REDIS_PORT=6379` |
| `redis.password` | `cube` | `ceuhvu123` | `CUBE_SANDBOX_REDIS_PASSWORD=ceuhvu123` |

### 4.2 持久化对齐

One Click 默认使用 Docker named volume：

- `cube-sandbox-mysql-data`
- `cube-sandbox-redis-data`

Kubernetes 中默认采用 one-click 风格 hostPath，避免安装测试环境必须具备默认 StorageClass：

```yaml
mysql:
  persistence:
    enabled: true
    existingClaim: ""
    hostPath: /data/cube-mysql
    size: 20Gi
    storageClassName: ""
    accessModes:
      - ReadWriteOnce
```

```yaml
redis:
  persistence:
    enabled: true
    existingClaim: ""
    hostPath: /data/cube-redis
    size: 10Gi
    storageClassName: ""
    accessModes:
      - ReadWriteOnce
```

支持：

1. `existingClaim`：复用已有 PVC。
2. `hostPath`：使用指定宿主机路径。
3. `hostPath: ""` 且未指定 `existingClaim` 时，Chart 创建 PVC。

### 4.3 端口对齐

- MySQL Service `port` 使用 `.Values.mysql.port`。
- Redis Service `port` 使用 `.Values.redis.port`。
- 容器内端口仍为 MySQL `3306` / Redis `6379`。
- CubeMaster / CubeAPI / CubeProxy 配置继续使用 `.Values.mysql.port` /
  `.Values.redis.port`。

### 4.4 MySQL 启动参数对齐

MySQL container 增加 One Click compose 中的参数：

```yaml
args:
  - --default-authentication-plugin=mysql_native_password
  - --skip-name-resolve
```

### 4.5 健康检查

MySQL：

```bash
mysqladmin ping -h 127.0.0.1 -u${MYSQL_USER} -p${MYSQL_PASSWORD} --silent
```

Redis：

```bash
redis-cli -a "${REDIS_PASSWORD}" ping | grep -x PONG
```

MySQL / Redis 均增加：

- startupProbe
- readinessProbe
- livenessProbe

### 4.6 第三方服务模式

保持现有逻辑：

- `mysql.host` 非空时，不安装 `cube-mysql`。
- `redis.host` 非空时，不安装 `cube-redis`。
- 不创建内置 Deployment / Service / PVC。
- Master / API / Proxy 使用第三方地址。

### 4.7 涉及文件

- `deploy/k8s/chart/templates/mysql.yaml`
- `deploy/k8s/chart/templates/redis.yaml`
- `deploy/k8s/chart/templates/_helpers.tpl`
- `deploy/k8s/chart/templates/validate.yaml`
- `deploy/k8s/chart/values.yaml`

## 5. readiness / liveness / Helm test 增强方案

### 5.1 CubeMaster

增加：

```yaml
readinessProbe:
  httpGet:
    path: /notify/health
    port: cubemaster
livenessProbe:
  httpGet:
    path: /notify/health
    port: cubemaster
```

### 5.2 CubeAPI

增加：

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: http-api
livenessProbe:
  httpGet:
    path: /health
    port: http-api
```

### 5.3 Cube Node

- readiness 默认开启，检查 cubelet 9999 端口。
- liveness 默认开启，检查 cubelet 9999 端口。
- 后续可扩展 exec 检查 network-agent `/healthz` / `/readyz`。

### 5.4 CubeEgress / cube-egress-net

- `cube-egress` 继续检查 `/admin/v1/health`。
- `cube-egress-net` 增加 exec 检查：
  - `cube-dev` interface 存在；
  - ip rule / iptables 关键规则存在；
  - 网络初始化脚本状态正常。

### 5.5 Helm test 覆盖

Helm test 至少覆盖：

1. CubeMaster `/notify/health`。
2. CubeAPI `/health`。
3. WebUI `/` 和 `/cubeapi/v1/health`。
4. 内置 MySQL / Redis 健康。
5. CubeProxy HTTP / HTTPS 数据面。
6. CubeDNS `cube.app` / wildcard 解析。
7. Cube Node DaemonSet ready 数量。
8. Cube Node 注册信息非空。
9. Pod 内 runtime asset：
   - `/dev/kvm`
   - `/data/cubelet/cubelet.sock`
   - `/tmp/cube/network-agent-grpc.sock`
   - `Cubelet/config/config.toml`
   - `cube-kernel-scf/vmlinux`
   - `cube-image/cube-guest-image-cpu.img`
10. CubeEgress sidecar ready 和 admin health。

CubeProxy 数据面保持 One Click 的 host-network 语义：默认
`cubeProxy.hostNetwork=true`，nginx 监听节点 `80/443`，这样 node-local
`cube-dns` 返回节点 HostIP 后，流量进入本节点 CubeProxy，CubeProxy 可以直连
本机 sandbox bridge IP。Chart 同时写入 nginx `resolver`，保证 Lua Redis
客户端可以解析内置或第三方 Redis DNS 名称。Chart 不创建
`cube-proxy-node` ClusterIP Service，避免 Kubernetes 对多个 CubeProxy Pod
做随机分流并产生不同于 One Click 的数据面入口语义。

### 5.6 涉及文件

- `deploy/k8s/chart/templates/master.yaml`
- `deploy/k8s/chart/templates/api.yaml`
- `deploy/k8s/chart/templates/mysql.yaml`
- `deploy/k8s/chart/templates/redis.yaml`
- `deploy/k8s/chart/templates/node-daemonset.yaml`
- `deploy/k8s/chart/templates/tests/*.yaml`
- `deploy/k8s/chart/values.yaml`

## 6. external control plane 支持方案

### 6.1 目标

对齐 One Click compute-only：

```bash
ONE_CLICK_DEPLOY_ROLE=compute
ONE_CLICK_CONTROL_PLANE_CUBEMASTER_ADDR=<ip>:8089
```

Chart 需要支持只部署 Cube Node，并让 Cube Node 注册到外部 CubeMaster。

### 6.2 values 设计

```yaml
externalControlPlane:
  enabled: false
  masterEndpoint: ""
  apiEndpoint: ""
```

### 6.3 行为

- `externalControlPlane.enabled=false`：使用 Chart 内部 CubeMaster Service。
- `externalControlPlane.enabled=true`：所有需要 CubeMaster 的地方使用
  `externalControlPlane.masterEndpoint`。
- `controlPlane.enabled=false && cubeNode.enabled=true` 时，必须配置
  `externalControlPlane.masterEndpoint`。
- `externalControlPlane.apiEndpoint` 可选，用于 Helm test 查询外部 API；
  未配置时跳过 API 侧测试。

### 6.4 helper

`templates/_helpers.tpl` 中 `cube.masterEndpoint` 改为：

```gotemplate
{{- define "cube.masterEndpoint" -}}
{{- if .Values.externalControlPlane.enabled -}}
{{- .Values.externalControlPlane.masterEndpoint -}}
{{- else -}}
{{- printf "%s.%s.svc.cluster.local:%v" (include "cube.masterName" .) .Release.Namespace .Values.controlPlane.master.service.port -}}
{{- end -}}
{{- end -}}
```

新增 `cube.apiEndpoint` helper，用于 Helm test 和 WebUI upstream 默认判断。

### 6.5 validate 规则

- `externalControlPlane.enabled=true` 时必须设置 `masterEndpoint`。
- `controlPlane.enabled=false && cubeNode.enabled=true` 时必须设置
  `externalControlPlane.enabled=true`。
- `externalControlPlane.enabled=true && controlPlane.enabled=true` 允许，但文档注明仅用于混合测试。

### 6.6 涉及文件

- `deploy/k8s/chart/templates/_helpers.tpl`
- `deploy/k8s/chart/templates/validate.yaml`
- `deploy/k8s/chart/templates/api.yaml`
- `deploy/k8s/chart/templates/node-daemonset.yaml`
- `deploy/k8s/chart/templates/tests/*.yaml`
- `deploy/k8s/chart/values.yaml`

## 7. CIDR 冲突检测对齐方案

### 7.1 当前差异

当前 `deploy/images/scripts/cube-node-init.sh` 只检查是否存在完全相同 route，
弱于 One Click。

One Click 会检查：

- IPv4 CIDR 格式；
- mask 范围 `/8` 到 `/30`；
- CIDR 是否为网络地址；
- 与 host interface 地址重叠；
- 与 host route 重叠；
- `CUBE_SANDBOX_NETWORK_CIDR_SKIP_CONFLICT_CHECK=1` 跳过冲突检测。

### 7.2 改造目标

将 One Click 的 CIDR 检查逻辑移植到 `cube-node-init.sh`，保证节点初始化阶段
fail-fast。

### 7.3 检测规则

实现：

1. `validate_cidr_format`：校验 `A.B.C.D/M`。
2. `ip_to_int`：IPv4 转整数。
3. `network_addr`：计算网络地址。
4. `cidr_overlap`：判断两个 CIDR 是否重叠。
5. `collect_host_networks`：收集 `ip -o -4 addr show` 和 `ip -o -4 route show`。
6. `check_cidr_conflict`：输出冲突 interface / route 明细。

跳过规则：

```bash
CUBE_SANDBOX_NETWORK_CIDR_SKIP_CONFLICT_CHECK=1
```

仅跳过冲突检测，不跳过 CIDR 格式和网络地址校验。

### 7.4 涉及文件

- `deploy/images/scripts/cube-node-init.sh`
- `deploy/images/cube-node-init/Dockerfile`，如需额外命令依赖则补充。

## 8. values.yaml 默认值对齐清单

| values 路径 | 当前默认 | 改造后默认 | 说明 |
| --- | --- | --- | --- |
| `mysql.password` | `cube` | `cube_pass` | 对齐 One Click |
| `mysql.rootPassword` | `cube-root` | `cube_root` | 对齐 One Click |
| `mysql.persistence.enabled` | `false` | `true` | One Click 默认 Docker volume 持久化 |
| `redis.image.tag` | `7.2` | `7-alpine` | 对齐 One Click |
| `redis.password` | `cube` | `ceuhvu123` | 对齐 One Click |
| `redis.persistence.enabled` | `false` | `true` | One Click 默认 Docker volume 持久化 |
| `controlPlane.master.persistence.enabled` | `false` | `true` | 对齐 `/data/CubeMaster/storage` 持久化语义 |
| `placement.nodeLabelSelector.cube.tencent.com/allow-pvm-bootstrap` | 无 | `"true"` | PVM bootstrap 默认开启时要求节点显式授权 |
| `cubeNode.updateStrategy.type` | `OnDelete` | `RollingUpdate` | 对齐 PVM 安装验证 profile |
| `cubeNode.pvmGuestKernel.enabled` | `false` | `true` | 默认进入 PVM profile，减少安装 values |
| `bootstrap.pvmHostKernel.enabled` | `false` | `true` | 默认初始化 PVM host kernel |
| `bootstrap.nodeInit.loadKVMModule` | `true` | `false` | One Click 不默认加载 `kvm_pvm` |
| `bootstrap.nodeInit.dataCubelet.loopback.enabled` | `false` | `true` | 开箱安装时自动准备 `/data/cubelet` XFS loopback |
| `cubeDns.domain` | 空，继承 proxy domain | `cube.app` | 对齐 One Click 默认域名 |
| `cubeDns.mode` | 无 | `nodeLocal` | 让 cube-node DS 直接使用 cube-dns |
| `cubeNode.dns.useCubeDns` | 无 | `true` | 默认让 cube-node 使用 node-local cube-dns |
| `cubeNode.probes.liveness.enabled` | `false` | `true` | 增强运行期自愈 |

PVM Host Kernel InitContainer 默认开启。安装前必须只给允许被初始化和重启的计算节点打上选择标签：

```yaml
placement:
  nodeLabelSelector:
    cube.tencent.com/role: compute
    cube.tencent.com/cube-node: "true"
    cube.tencent.com/allow-pvm-bootstrap: "true"
```

`cube-node-init` 需要执行与 One Click 等价的 PVM 一致性预检：

- 若宿主机已加载 `kvm_pvm`，但 `cubeNode.pvmGuestKernel.enabled=false`，则 fail-fast，避免错误选择 `vmlinux-bm`。
- 若 `cubeNode.pvmGuestKernel.enabled=true`，但宿主机未加载 `kvm_pvm`，则由 `pvm-host-bootstrap` 尝试安装 host kernel 并按配置重启；重启后仍不满足时 fail-fast。

## 9. 实施步骤

### 阶段一：values 与 validate

1. 调整 `values.yaml` 默认值。
2. 增加 `externalControlPlane` 和 `cubeNode.dns` values。
3. 增加 validate 规则，避免错误组合。

### 阶段二：DNS 与 external control plane

1. `cube-dns` 支持 `nodeLocal` 和 `service` 两种模式。
2. `cube-node` 支持 `dnsPolicy: None` + `dnsConfig`。
3. 增加 DNS wait initContainer。
4. 更新所有 Master endpoint 引用。

### 阶段三：DB 与探针

1. MySQL / Redis PVC 模板。
2. MySQL / Redis Service port values 化。
3. MySQL / Redis startup/readiness/liveness。
4. Master / API probes。
5. 增强 Helm test。

### 阶段四：CIDR 检测

1. 移植 One Click CIDR 检查。
2. 增加冲突详情输出。
3. 本地验证合法、非法、冲突、跳过四类场景。
4. 重新构建并推送 `cube-node-init:v0.4.0`。

### 阶段五：集群验证

1. 重新构建并推送变化镜像。
2. 删除旧 release。
3. 按最新 Chart 安装。
4. 执行 `helm test`。
5. 逐项执行功能验证。
6. 输出验证报告。

## 10. 集群验证标准

最终必须在集群环境验证通过。

### 10.1 静态验证

```bash
helm lint deploy/k8s/chart
helm template cube deploy/k8s/chart -n cube-system
```

检查：

- 不包含 demo IP。
- 不包含 nodePrepare 打标签 Job。
- 不包含 `cube-db-migrate`。
- 包含独立 `cubemastercli` 运维 Deployment；不在 Master/Node 运行镜像中混入 `cubemastercli`，不提供 `ctl` wrapper。
- MySQL/Redis 第三方模式下不渲染内置 Deployment / Service / PVC。

### 10.2 安装验证

```bash
helm upgrade --install cube deploy/k8s/chart \
  -n cube-system \
  --create-namespace \
  --wait \
  --timeout 30m
```

通过标准：

- 所有 Deployment Ready。
- 所有 DaemonSet Ready。
- 所有 initContainer 成功。
- 无 ImagePullBackOff / CrashLoopBackOff。
- Cube Node 只在 label 命中的节点上启动。

### 10.3 DNS 验证

在 cube-node Pod 内执行：

```bash
cat /etc/resolv.conf
nslookup cube.app 127.0.0.54
nslookup wildcard-check.cube.app 127.0.0.54
nslookup kubernetes.default.svc.cluster.local 127.0.0.54
```

通过标准：

- `/etc/resolv.conf` nameserver 为 `127.0.0.54`。
- `cube.app` 返回本节点 IP 或配置的 `answerIP`。
- wildcard 返回本节点 IP 或配置的 `answerIP`。
- 集群 Service 域名仍可解析。

### 10.4 DB 验证

```bash
kubectl get pvc -n cube-system
kubectl exec -n cube-system deploy/cube-mysql -- \
  mysqladmin ping -h 127.0.0.1 -ucube -pcube_pass --silent
kubectl exec -n cube-system deploy/cube-redis -- \
  redis-cli -a ceuhvu123 ping
```

通过标准：

- MySQL / Redis PVC Bound。
- MySQL health OK。
- Redis 返回 `PONG`。

第三方 DB 模式：

- 内置 MySQL / Redis Deployment、Service、PVC 不存在。
- Master / API / Proxy 使用第三方地址。
- 健康检查通过。

### 10.5 Node 验证

```bash
kubectl get ds -n cube-system
kubectl get pod -n cube-system -l app.kubernetes.io/component=cube-node -o wide
```

进入 cube-node Pod：

```bash
test -e /dev/kvm
test -S /data/cubelet/cubelet.sock
test -S /tmp/cube/network-agent-grpc.sock
test -f /usr/local/services/cubetoolbox/Cubelet/config/config.toml
test -f /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux
test -f /usr/local/services/cubetoolbox/cube-image/cube-guest-image-cpu.img
```

通过标准：

- Node DaemonSet ready 数等于期望计算节点数。
- Cube Node 向 CubeMaster 注册成功。
- 必要 socket 和 runtime asset 存在。

### 10.6 CIDR 验证

至少验证：

1. 合法且不冲突 CIDR：init 成功。
2. 格式非法 CIDR：init 失败并输出明确错误。
3. 与 host route/interface 重叠 CIDR：init 失败并输出冲突明细。
4. 设置 `CUBE_SANDBOX_NETWORK_CIDR_SKIP_CONFLICT_CHECK=1`：仅跳过冲突检测。

### 10.7 Helm test 验证

```bash
helm test cube -n cube-system --timeout 20m
```

通过标准：

- Master / API / WebUI / Proxy / DNS / DB / Node / Egress 关键路径全部通过。
- test Pod 退出码为 0。
- test 日志中有明确检查结果。

### 10.8 external control plane 验证

```bash
helm upgrade --install cube-compute deploy/k8s/chart \
  -n cube-system \
  --set controlPlane.enabled=false \
  --set externalControlPlane.enabled=true \
  --set externalControlPlane.masterEndpoint=<external-master>:8089 \
  --wait \
  --timeout 30m
```

通过标准：

- 不安装 Chart 内部 Master / API / MySQL / Redis / WebUI，除非显式开启。
- Cube Node 使用外部 Master endpoint。
- 外部 Master 能看到新注册节点。
- Node init connectivity check 通过。

## 11. 最终验证报告格式

实现并完成集群验证后，按如下格式输出：

```text
# CubeSandbox Chart One Click 对齐验证报告

## 验证环境
- 集群：
- Namespace：cube-system
- Chart commit：
- 镜像 tag：v0.4.0
- 计算节点 label selector：

## 验证命令和结果
- helm lint：通过/失败
- helm template：通过/失败
- helm install/upgrade：通过/失败
- kubectl get pod：通过/失败
- DNS 验证：通过/失败
- MySQL/Redis 验证：通过/失败
- Node 验证：通过/失败
- CIDR 验证：通过/失败
- helm test：通过/失败
- external control plane：通过/失败/未执行原因

## Pod 状态
<kubectl get pod -n cube-system -o wide 输出摘要>

## 关键日志
<失败或关键成功日志摘要>

## 遗留问题
- 无 / 列表

## 结论
- 通过 / 不通过
```

## 12. 风险与取舍

1. **不默认修改宿主机 DNS**
   node-local DNS 只影响 cube-node Pod，避免 Chart 隐式修改宿主机网络配置。
   外部客户端访问 `cube.app` 仍需用户配置 DNS/LB。

2. **PVM Host Kernel 默认开启**
   默认 profile 面向 PVM 开箱安装，会修改 host kernel/bootloader 并按租约重启
   选中的计算节点。节点必须带 `cube.tencent.com/allow-pvm-bootstrap=true`
   才会被选中；`cube-node-init` 会校验 `kvm_pvm` 与
   `cubeNode.pvmGuestKernel.enabled` 一致。

3. **hostPath 默认开启**
   对齐 One Click host-local 持久化语义，避免依赖默认 StorageClass。
   生产环境可按需切换到 `existingClaim` 或 PVC。

4. **SQL seed 不默认执行**
   遵守不内置 demo 数据的要求，不渲染 One Click 的
   `sql/002_seed_single_node.sql`，节点数据依赖真实 Cube Node 注册。

5. **external control plane 需要明确 endpoint**
   compute-only 模式不猜测控制面地址，避免错误注册。
