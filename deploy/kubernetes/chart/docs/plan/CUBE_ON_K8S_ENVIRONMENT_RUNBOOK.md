# Cube On K8S 当前环境操作手册

## 目的

本文记录当前 Cube On K8S 验证环境的现场信息和操作步骤，目标是让一个全新的 agent 不依赖历史上下文，也能知道：

- 如何通过跳板机操作当前 TKE/K8S 集群。
- 如何操作当前 Cube compute 节点。
- 当前部署了哪些 Cube / AgentWay 组件。
- 当前环境如何删除旧 Chart、渲染并部署 v0.5.0 Chart。
- 如何通过 API 创建 sandbox 并执行 Hello 代码完成端到端验收。

本文记录当前测试环境的操作方式和现场状态，方便后续 agent 直接接手排查。所有 K8S 操作必须先通过跳板机 `106.53.31.91` 执行。

## 环境入口

当前所有 K8S 操作都通过跳板机执行。

```text
跳板机公网 IP: 106.53.31.91
跳板机对应集群节点内网 IP: 10.2.16.125
```

本地工作区已有用于免交互执行跳板机命令的 expect 脚本：

```bash
/tmp/cube_ssh.exp
```

在本仓库中执行远程命令时必须加 `rtk` 前缀：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 'kubectl get nodes -o wide'
```

注意：

- 跳板机上可以直接使用 `kubectl` 操作当前集群。
- 跳板机当前没有可用的 `helm` 命令；现场日常验证以 `kubectl` 为主。
- 跳板机偶发快速并发 SSH 握手失败，表现为 `kex_exchange_identification: Connection closed by remote host`。遇到时等待 5 到 10 秒后串行重试。

## 登录方式

### 登录跳板机

本地 agent 推荐使用 expect 脚本执行一次性命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 'kubectl get pod -A -o wide | head'
```

需要交互时，可直接 SSH 到跳板机：

```bash
ssh root@106.53.31.91
```

跳板机认证按当前环境配置。

### 操作 Cube compute 节点

当前 Cube On K8S 的 compute 节点是：

```text
10.2.16.125
10.2.16.157
10.2.222.49
10.2.222.82
```

优先通过 `kubectl` 在跳板机上操作节点和 Pod：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl get nodes -l cube.tencent.com/role=compute -o wide'
```

如果需要进入宿主机 namespace，可在目标节点创建临时 privileged debug Pod，
挂载宿主机根目录到 `/host` 后执行 `chroot /host ...`。新弹性节点
`10.2.222.49` / `10.2.222.82` 未验证可用的 SSH 密码登录；此前
`10.2.5.16` / `10.2.5.189` 的 root 密码记录不再代表当前 compute
节点登录方式。

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system exec <debug-pod> -- chroot /host /bin/sh -c "uname -r; cat /proc/cmdline"'
```

常见用途：

```bash
# 在节点上看监听端口
ss -lntp | grep -E ':80|:443|:200'

# 在节点上抓跨节点 hostPort 流量，替换为当前目标 compute IP
tcpdump -ni any host 10.2.222.49 or host 10.2.222.82

# 看 Cube 相关进程
ps -ef | grep -E 'cubelet|network-agent|cube-proxy|openresty' | grep -v grep
```

## 当前 K8S 节点

关键节点：

```text
10.2.16.125   跳板机 / kubectl 操作入口，公网 IP 106.53.31.91，Cube compute 节点
10.2.16.157   Cube compute 节点
10.2.36.21    AgentWay 组件当前运行节点
10.2.5.44     Cube control 节点
10.2.5.54     Cube control 节点
10.2.222.49   Cube compute 节点，TencentOS Server 4，PVM host kernel
10.2.222.82   Cube compute 节点，TencentOS Server 4，PVM host kernel
```

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 'kubectl get nodes -o wide'
```

Cube Chart 当前将调度拆成控制面和计算面两组 placement。

控制面 Deployment 使用以下 nodeSelector：

```yaml
cube.tencent.com/role: control
cube.tencent.com/cube-control: "true"
```

控制面 toleration：

```yaml
- key: cube.tencent.com/control
  operator: Equal
  value: "true"
  effect: NoSchedule
```

Cube compute 相关 DaemonSet 使用以下 nodeSelector：

```yaml
cube.tencent.com/role: compute
cube.tencent.com/cube-node: "true"
cube.tencent.com/allow-pvm-bootstrap: "true"
```

Compute toleration：

```yaml
- key: cube.tencent.com/compute
  operator: Equal
  value: "true"
  effect: NoSchedule
```

其中 `cube-node`、`cube-dns` 复用 compute placement；`cube-master`、`cube-api`、`cube-webui`、`cube-cubemastercli`、`cube-mysql`、`cube-redis`、`cube-proxy-node` 复用 control placement。Chart 默认通过 `global.timezone=Asia/Shanghai` 给 Cube 容器、initContainer 和 sidecar 注入 `TZ`。

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get deploy,sts,ds,pod -o wide'
```

## Namespace 与部署组件

### cube-system

Cube Chart 安装在 `cube-system` namespace，Helm release 名称为 `cube`。

已确认存在 Helm release secret：

```text
sh.helm.release.v1.cube.v1
sh.helm.release.v1.cube.v2
```

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get secret -l owner=helm,name=cube'
```

v0.5.0 目标控制面 Deployment / StatefulSet：

```text
cube-master        docker.io/liv1020/cube-master:v0.5.0
cube-api           docker.io/liv1020/cube-api:v0.5.0
cube-webui         docker.io/liv1020/cube-webui:v0.5.0
cube-cubemastercli docker.io/liv1020/cubemastercli:v0.5.0
cube-proxy-node    docker.io/liv1020/cube-proxy-node:v0.5.0
cube-mysql         StatefulSet, mysql:8.0
cube-redis         StatefulSet, redis:7-alpine
```

这些 Pod 当前按 `placement.controlPlane` 运行在 `10.2.5.44` / `10.2.5.54`。

v0.5.0 目标计算面 DaemonSet：

```text
cube-node
  - docker.io/liv1020/cube-node:v0.5.0
  - docker.io/liv1020/cube-egress:v0.5.0
  - docker.io/liv1020/cube-egress-net:v0.5.0
  - docker.io/liv1020/cube-node-init:v0.5.0
  - docker.io/liv1020/cube-pvm-host-bootstrap:v0.5.0

cube-dns
  - ccr.ccs.tencentyun.com/tkeimages/coredns:v1.11.1-tke.1
```

`cube-node`、`cube-dns` 部署在 `10.2.16.125`、`10.2.16.157`、
`10.2.222.49`、`10.2.222.82` 四台 compute 节点上。`cube-proxy-node`
从 v0.5.0 起不再是 compute DaemonSet，而是 control 节点上的 Deployment。

### Sandbox 内 Kubernetes Service DNS

`cube-node` Pod 的 `/etc/resolv.conf` 与 Cube sandbox guest 内的
`/etc/resolv.conf` 是两层配置。Chart 中的 node-local `cube-dns`
监听 `127.0.0.54`，只适合 `cube-node` Pod 自身使用；不要把
`127.0.0.54` 作为 sandbox guest 的 nameserver，因为在 guest 内它会变成
sandbox 自己的 loopback。

如果需要 sandbox 内解析并访问 Kubernetes Service 名称，例如
`agent-way-model-gateway.agent-infra.svc.cluster.local`，不要假设 sandbox
直接访问 Service ClusterIP 一定可行。CubeVS eBPF egress 可能绕过宿主机
kube-proxy 的 Service VIP DNAT；节点上 `curl <ClusterIP>` 能通，不代表
sandbox 里访问 ClusterIP 能通。

v0.5.0 Chart 已删除 sandbox service proxy 资源，不再在 Chart 内为这类
Service 自动生成 hostNetwork HTTP proxy 或 DNS override。需要暴露给
sandbox 的 in-cluster Service 应由平台网络层、AgentWay provider 配置或
operator 管理的外部代理处理。

Service：

```text
cube-master ClusterIP 8089
cube-api    ClusterIP 3000
cube-webui  ClusterIP 12088
cube-mysql  Headless Service 3306, backs StatefulSet cube-mysql
cube-redis  Headless Service 6379, backs StatefulSet cube-redis
```

注意：当前 Chart 不创建 `cube-proxy-node` ClusterIP Service。CubeProxy 使用 control 节点 Deployment + hostNetwork 直接监听节点 `80/443`，避免 ClusterIP 随机分流破坏 One Click 数据面语义。渲染 live values 时必须设置 `cubeProxy.advertiseIP` 或 `cubeDns.answerIP` 指向 control CubeProxy 入口。

状态存储：

```text
StorageClass: cube-cbs-wffc
volumeBindingMode: WaitForFirstConsumer

PVC:
cube-master-storage       20Gi
mysql-data-cube-mysql-0   20Gi
redis-data-cube-redis-0   10Gi
```

当前控制面状态组件默认使用 PVC，不再使用 hostPath。`cube-cbs-wffc` 使用 WFFC，目的是让 CBS 盘在 Pod 选中 control 节点后再创建，避免多可用区 TKE 集群中 PV zone 与 Pod zone 不匹配。

### agent-way-system

AgentWay 安装在 `agent-way-system` namespace。

关键 CRD：

```text
agents.agent.agentway.io
agentsandboxproviders.agent.agentway.io
```

当前 Cube provider：

```yaml
kind: AgentSandboxProvider
metadata:
  name: cube
spec:
  cube:
    masterEndpoint: http://cube-master.cube-system.svc.cluster.local:8089
    proxyAccessMode: host
    proxyBaseURL: http://43.144.18.240.sslip.io
    distributionScope:
      - 10.2.16.125
      - 10.2.16.157
      - 10.2.222.49
      - 10.2.222.82
    allowOut:
      - 172.16.0.0/12
      - 10.2.0.0/16
    networkType: tap
    instanceType: cubebox
    envdPort: 49983
    allowInternetAccess: true
```

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl get agentsandboxproviders.agent.agentway.io cube -o yaml'
```

注意：

- `proxyBaseURL` 必须指向当前 control 节点 CubeProxy 入口，用于生成公网可访问的 Agent `accessURL`。
- 判断 sandbox owner 节点不要只看 `accessURL`，必须查 Redis 中的 `HostIP`。
- `distributionScope` 当前按 K8S compute selector 对齐为：
  `10.2.16.125`、`10.2.16.157`、`10.2.222.49`、`10.2.222.82`。

## 部署方式

当前 Cube 资源是 Helm release `cube` 安装出来的，Chart 源在本仓库：

```text
deploy/kubernetes/chart
```

跳板机上没有可用的 `helm`，live 环境使用本地 `helm template` 渲染，再把渲染结果传到跳板机执行 `kubectl apply`。v0.5.0 会把 `cube-proxy-node` 从 DaemonSet 改为 Deployment，把 MySQL/Redis 改为 StatefulSet；现场部署前先删除旧 `cube-system` namespace，避免旧资源和旧 PVC 名称残留影响验证。

删除旧 Chart：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl delete ns cube-system --wait=true --timeout=20m || true'
```

确认 control 入口 IP。`cubeProxy.advertiseIP` 应设置为要暴露给 `cube.app` / wildcard 的 control CubeProxy IP 或 LB IP：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl get nodes -l cube.tencent.com/role=control -o wide'
```

本地渲染并传输：

```bash
CONTROL_PROXY_IP=<control-node-or-lb-ip>

rtk rm -rf /tmp/cube-rendered-v050
rtk helm template cube deploy/kubernetes/chart \
  -n cube-system \
  --output-dir /tmp/cube-rendered-v050 \
  --set cubeProxy.advertiseIP="${CONTROL_PROXY_IP}" \
  --set cubeDns.answerIP="${CONTROL_PROXY_IP}"

rtk tar -czf /tmp/cube-rendered-v050.tgz -C /tmp cube-rendered-v050
rtk scp /tmp/cube-rendered-v050.tgz root@106.53.31.91:/tmp/cube-rendered-v050.tgz
```

跳板机 apply 并等待：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'set -e
   rm -rf /tmp/cube-rendered-v050
   tar -xzf /tmp/cube-rendered-v050.tgz -C /tmp
   kubectl create ns cube-system --dry-run=client -o yaml | kubectl apply -f -
   find /tmp/cube-rendered-v050/cube/templates -maxdepth 1 -type f -name "*.yaml" -exec kubectl -n cube-system apply -f {} \;'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system rollout status deploy/cube-master --timeout=20m && kubectl -n cube-system rollout status deploy/cube-api --timeout=20m && kubectl -n cube-system rollout status deploy/cube-proxy-node --timeout=20m && kubectl -n cube-system rollout status sts/cube-mysql --timeout=20m && kubectl -n cube-system rollout status sts/cube-redis --timeout=20m && kubectl -n cube-system rollout status ds/cube-dns --timeout=20m && kubectl -n cube-system rollout status ds/cube-node --timeout=30m'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get deploy,sts,ds,pod -o wide'
```

注意：如果现场把 MySQL 从旧 hostPath 切到新 PVC，或删除重建 MySQL PVC，`cube-node` 可能已经在旧库注册过，随后只继续上报 `t_cube_node_status`，不会自动补 `t_cube_node_registration`。这时创建 sandbox 会报：

```text
cube ret_code=130597 ret_msg=no more resource
```

处理方式是重启 compute DaemonSet，让 cube-node 重新向新库注册：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system rollout restart ds/cube-node && kubectl -n cube-system rollout status ds/cube-node --timeout=5m'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system exec cube-mysql-0 -- sh -lc '\''mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" -e "select node_id,host_ip,quota_cpu,quota_mem_mb,max_mvm_num,updated_at from t_cube_node_registration;"'\'''
```

当前已确认新 compute 节点重新注册成功：

```text
10.2.222.49  quota_cpu=16000  quota_mem_mb=19512  max_mvm_num=38
10.2.222.82  quota_cpu=16000  quota_mem_mb=19512  max_mvm_num=38
```

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'for p in $(kubectl -n cube-system get pod -l app.kubernetes.io/component=cube-node -o name); do echo "$p"; kubectl -n cube-system describe "$p" | sed -n "/  cube-node:/,/  cube-egress:/p" | grep -E "Image:|Image ID:|Ready:"; done'
```

## 新 TencentOS 4 compute 节点 /dev/kvm 排障记录

2026-07-02 将 compute 扩到 `10.2.222.49`、`10.2.222.82` 后，
`cube-node-init` 曾在两台节点上报错：

```text
[cube-node-init] loading kvm_pvm module
[cube-node-init] ERROR: /dev/kvm does not exist
```

根因不是 `/dev/kvm` 权限或挂载问题，而是节点已经运行 PVM host kernel，
但启动参数缺少 `nopti pti=off`。`kvm_pvm` 当前不支持 host KPTI，
`modprobe kvm_pvm` 会失败：

```text
modprobe: ERROR: could not insert 'kvm_pvm': Operation not supported
kvm_pvm: Support for host KPTI is not included yet.
```

诊断命令示例：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system logs <cube-node-pod> -c cube-node-init --previous --tail=80'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system exec <host-debug-pod> -- chroot /host /bin/sh -c '\''cat /proc/cmdline; modprobe kvm_pvm 2>&1 || true; dmesg | tail -120 | grep -iE "kvm|pvm|pti" || true'\'''
```

现场修复方式是在对应节点的 PVM kernel 启动项补充 `nopti pti=off`，
然后重启节点：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system exec <host-debug-pod> -- chroot /host /bin/sh -c '\''set -e; pvm_kernel=$(ls /boot/vmlinuz-*pvm.host* 2>/dev/null | sort | tail -1); test -n "$pvm_kernel"; grubby --set-default "$pvm_kernel"; grubby --update-kernel "$pvm_kernel" --args "nopti pti=off"; systemctl reboot || reboot'\'''
```

修复后验证：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system logs <cube-node-pod> -c cube-node-init --tail=80'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system rollout status ds/cube-node --timeout=5m'
```

当前验证结果：

```text
10.2.222.49  cmdline contains nopti pti=off; kvm_pvm loaded; /dev/kvm exists; cube-node 3/3 Running
10.2.222.82  cmdline contains nopti pti=off; kvm_pvm loaded; /dev/kvm exists; cube-node 3/3 Running
cube-node DaemonSet 4/4 READY
cube-node DaemonSet pvm-host-bootstrap env contains PVM_KERNEL_BOOT_ARGS=nopti pti=off
```

长期修复已经落到 `deploy/kubernetes/images/scripts/pvm-host-bootstrap.sh`：
当节点已经运行 PVM kernel 但缺少 `PVM_KERNEL_BOOT_ARGS` 时，不再提前
退出，而是补 bootloader 参数并通过独立的 `boot-args-reboot-count`
触发一次协调重启。发布时需要用该脚本重新构建并推送
`cube-pvm-host-bootstrap` 镜像；只给 DaemonSet 增加 env 不能让旧镜像
具备这段自愈逻辑。

## OpenClaw Agent CR 验收记录

当前已用 Agent CR 部署并访问成功：

```text
namespace: agent-way-system
name: cube-openclaw-placement-verify
phase: Running
serviceHealth.ready: true
sandboxId: b4ff0f14e207496f9a46e24c1bedf230
owner host: 10.2.5.138
templateId: tpl-8617535d681744f5ac2e5c82
gatewayToken: openclaw-placement-verify-token
accessURL: http://18789-b4ff0f14e207496f9a46e24c1bedf230.43.144.18.240.sslip.io/
```

验证命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n agent-way-system get agent cube-openclaw-placement-verify -o jsonpath="phase={.status.phase}{\"\n\"}sandbox={.status.sandboxId}{\"\n\"}url={.status.accessURL}{\"\n\"}ready={.status.serviceHealth.ready}{\"\n\"}message={.status.serviceHealth.message}{\"\n\"}"'

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'URL=$(kubectl -n agent-way-system get agent cube-openclaw-placement-verify -o jsonpath="{.status.accessURL}"); curl -L --connect-timeout 10 --max-time 30 -sS -D /tmp/openclaw.headers -o /tmp/openclaw.body -w "http=%{http_code} total=%{time_total} size=%{size_download}\n" "$URL"; grep -i "<title>" /tmp/openclaw.body'
```

当前结果：

```text
http=200
<title>OpenClaw Control</title>
```

本次没有使用体积过大的 all-in-one OpenClaw 镜像。源镜像为：

```text
cube-sandbox-image.tencentcloudcr.com/demo/lightweight-openclaw-deepseek-wecom:latest
digest: sha256:2dd779fe91862e2b4a3d2b60e2bdb2606f3b8519422fd7e8aaef0bc35cfc7239
```

注意：当前 `10.2.5.x` control/compute 节点可以访问 `ccr.ccs.tencentyun.com`，但访问 `cube-sandbox-image.tencentcloudcr.com:443` 超时。因此本次为了完成 K8S 环境验收，把同 digest 的轻量 OpenClaw 镜像临时镜像到当前可匿名访问的 CCR tag：

```text
ccr.ccs.tencentyun.com/pavleli/cube-master:openclaw-lite-k8s-verify-20260629
digest: sha256:2dd779fe91862e2b4a3d2b60e2bdb2606f3b8519422fd7e8aaef0bc35cfc7239
```

后续如果要直接使用源 Harbor 域名，需要给 `10.2.5.44`、`10.2.5.54`、`10.2.5.16`、`10.2.5.189` 补通到 `cube-sandbox-image.tencentcloudcr.com` 的 443 出网路径，或把 OpenClaw 镜像正式发布到 compute/control 节点可达的 CCR/TCR 仓库。

## Agent accessURL 502 定位与修复

本节记录 `cube-dns-proxy-verify` 的一次现场 502 排查，避免把
sandbox 出站 DNS 问题和 Agent 入站端口问题混淆。

### 当前验证对象

```text
namespace: agent-way-system
name: cube-dns-proxy-verify
sandboxId: 3fd0c222cffb4866ac67ff25d1583655
owner host: 10.2.5.16
accessURL: http://18789-3fd0c222cffb4866ac67ff25d1583655.43.144.18.240.sslip.io/
```

Redis 中的 CubeProxy 路由元数据：

```text
HostIP:    10.2.5.16
SandboxIP: 192.168.0.21
18789 -> 20038
49983 -> 20039
```

查询命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'REDIS=$(kubectl -n cube-system get pod -l app.kubernetes.io/component=redis -o jsonpath="{.items[0].metadata.name}"); kubectl -n cube-system exec "$REDIS" -- sh -ec '\''redis-cli -a "$REDIS_PASSWORD" hgetall bypass_host_proxy:3fd0c222cffb4866ac67ff25d1583655'\'''
```

### 根因

这次 `502` 不是
`agent-way-model-gateway.agent-infra.svc.cluster.local` 的 DNS 解析失败。
此前已经确认 sandbox 内能解析并访问：

```text
http://agent-way-model-gateway.agent-infra.svc.cluster.local:4000/
```

`502` 的直接原因是 Agent 应用只监听 sandbox 内的
`127.0.0.1:18789`。CubeProxy 入站路径是：

```text
browser
  -> CubeProxy
  -> owner HostIP:hostPort, e.g. 10.2.5.16:20038
  -> CubeVS hostPort map
  -> sandbox inner IP:containerPort
```

这条路径不会进入 guest loopback。因此应用只绑定
`127.0.0.1:<access.port>` 时，sandbox 内本地访问
`http://127.0.0.1:18789/` 可以成功，但外部 accessURL 会在 CubeProxy
反代到 hostPort 后收到拒绝或超时，最终表现为 `502`。

### 长期修复原则

首选修复 Agent 镜像或启动命令，让 Web 入口监听
`0.0.0.0:<access.port>`，或监听 sandbox 的 inner IP。不要只监听
`127.0.0.1:<access.port>`。

如果第三方镜像只能监听 loopback，可在 Agent CR 中注入
`bootstrapScripts`，在 sandbox 内启动一个轻量 TCP proxy，把 sandbox
inner IP 的入口端口转发到 `127.0.0.1:<access.port>`。当前验证镜像带
`python3`，现场热修使用了下面的脚本：

```yaml
spec:
  bootstrapScripts:
  - name: expose-loopback-web
    interpreter: /bin/sh
    script: |
      set -eu
      port=18789
      inner_ip="$(python3 - <<'PY'
      import socket
      s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
      try:
          s.connect(("169.254.68.5", 9))
          print(s.getsockname()[0])
      finally:
          s.close()
      PY
      )"
      proxy="/tmp/cube-loopback-proxy-${port}.py"
      cat > "${proxy}" <<'PY'
      import errno, select, socket, sys, threading
      bind = (sys.argv[1], int(sys.argv[2]))
      target = ("127.0.0.1", int(sys.argv[2]))
      s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
      s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
      try:
          s.bind(bind)
      except OSError as e:
          if e.errno == errno.EADDRINUSE:
              sys.exit(0)
          raise
      s.listen(128)
      def pump(a, b):
          try:
              while True:
                  r, _, _ = select.select([a, b], [], [])
                  for x in r:
                      data = x.recv(65536)
                      if not data:
                          return
                      (b if x is a else a).sendall(data)
          finally:
              for sock in (a, b):
                  try:
                      sock.close()
                  except Exception:
                      pass
      while True:
          client, _ = s.accept()
          upstream = socket.create_connection(target, timeout=5)
          threading.Thread(target=pump, args=(client, upstream), daemon=True).start()
      PY
      pkill -f "${proxy}" 2>/dev/null || true
      nohup python3 "${proxy}" "${inner_ip}" "${port}" >/tmp/cube-loopback-proxy-${port}.log 2>&1 &
```

生产使用时更推荐把等价逻辑固化到 Agent 镜像或启动脚本中；如果镜像中没有
`python3`，应改用镜像已有的 `socat`/`nc`，或直接让应用监听
`0.0.0.0`。

### 验证结果

热修后，跨节点 hostPort 和公网 accessURL 均已恢复：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system exec cube-node-hwgf5 -c cube-node -- sh -lc '\''curl -sS --connect-timeout 3 --max-time 8 -I http://10.2.5.16:20038/ | head'\'''

rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'curl -sS --connect-timeout 5 --max-time 15 -i http://18789-3fd0c222cffb4866ac67ff25d1583655.43.144.18.240.sslip.io/ | head -n 15'
```

结果：

```text
HTTP/1.1 200 OK
X-Cube-Retcode: 310200
<title>OpenClaw Control</title>
```

## 历史关键验证场景（旧 compute）

以下记录来自 compute 仍为 `10.2.5.138` / `10.2.5.238` 时的历史验证，用于保留 CubeProxy 跨节点链路排查方法。

历史环境用于验证和复现：

```text
非 owner CubeProxy
  -> owner HostIP:hostPort
  -> owner CubeVS
  -> SandboxIP:containerPort
```

历史复现 Agent：

```text
namespace: agent-way-system
name: cube-checksum-regression
sandboxId: 1f8a90b9896d49a6991bca1fd321ad37
```

滚动 `cube-node` 后旧 Agent 可能在 Cubelet 侧失效，应优先使用 fresh Agent 验证。本次修复验证 Agent：

```text
namespace: agent-way-system
name: cube-chartfix-verify
sandboxId: 66d957886a5d4d9b8dcd989d9e3d0637
```

Redis 元数据：

```text
HostIP:    10.2.5.138
SandboxIP: 192.168.0.71
49983 -> 20004
49999 -> 20005
```

因此：

- owner proxy 是 `10.2.5.138`。
- 非 owner proxy 是 `10.2.5.238`。
- Agent status 的 URL 后缀当前来自 provider `proxyBaseURL=http://43.144.18.240.sslip.io`，不是 sandbox owner 节点 IP。

## 复现步骤

### 1. 确认组件状态

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get deploy,ds,pod -o wide'
```

关键状态应为：

```text
cube-node         2/2 Ready
cube-proxy-node   2/2 Ready
cube-dns          2/2 Ready
cube-master       1/1 Ready
cube-api          1/1 Ready
cube-redis        1/1 Ready
cube-mysql        1/1 Ready
cube-webui        1/1 Ready
```

### 2. 确认 Agent 和 sandboxId

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n agent-way-system get agent cube-checksum-regression -o yaml'
```

如果旧 Agent 失效，例如滚动 `cube-node` 后出现 Cubelet 侧 `NotFoundAtCubelet`，不要继续使用旧 URL 验证，应新建一个 fresh Agent。

最小 Agent 示例：

```yaml
apiVersion: agent.agentway.io/v1alpha1
kind: Agent
metadata:
  name: cube-checksum-regression
  namespace: agent-way-system
  labels:
    agentway.io/provider: cube
    test: checksum-regression
  annotations:
    agentway.io/instance-name: Cube Checksum Regression
spec:
  accessToken: checksum-regression-token
  sandboxProviderRef: cube
  virtualAPIKey: sk-checksum-regression
  resources:
    cpuCores: 2
    memoryGi: 2
  profile:
    image: cube-sandbox-cn.tencentcloudcr.com/cube-sandbox/sandbox-code:latest
    access:
      port: 49999
      path: health
    ports:
    - name: health
      port: 49999
      protocol: TCP
```

应用命令：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 'kubectl -n agent-way-system apply -f /path/to/agent.yaml'
```

### 3. 查 Redis 路由元数据

不要直接输出 Redis 密码。Redis Pod 内已经有 `REDIS_PASSWORD` 环境变量，可在 Pod 内使用：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'REDIS=$(kubectl -n cube-system get pod -l app.kubernetes.io/component=redis -o jsonpath="{.items[0].metadata.name}"); kubectl -n cube-system exec "$REDIS" -- sh -ec '\''redis-cli -a "$REDIS_PASSWORD" hgetall bypass_host_proxy:66d957886a5d4d9b8dcd989d9e3d0637'\'''
```

重点字段：

```text
HostIP
SandboxIP
<containerPort>
<hostPort>
```

### 4. 分别访问 owner / 非 owner proxy

历史修复验证 Agent 的 owner 是 `10.2.5.138`。

访问非 owner proxy `10.2.5.238`：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'curl --connect-timeout 5 --max-time 15 -sS -o /tmp/reg_non_owner.out -w "non_owner_http=%{http_code} total=%{time_total}\n" http://49999-66d957886a5d4d9b8dcd989d9e3d0637.10.2.5.238.sslip.io/health; head -c 200 /tmp/reg_non_owner.out || true; echo'
```

修复镜像下，结果为：

```text
non_owner_http=200
"OK"
```

访问 owner proxy `10.2.5.138`：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'curl --connect-timeout 5 --max-time 15 -sS -o /tmp/reg_owner.out -w "owner_http=%{http_code} total=%{time_total}\n" http://49999-66d957886a5d4d9b8dcd989d9e3d0637.10.2.5.138.sslip.io/health; head -c 200 /tmp/reg_owner.out || true; echo'
```

当前结果为：

```text
owner_http=200
"OK"
```

旧 `cube-node:v0.4.0` 下，`cube-checksum-regression` 曾表现为非 owner 504、owner 200；修复镜像下，fresh Agent 的 owner / non-owner 均为 200。问题集中在旧 K8S `cube-node` 镜像的 CubeVS hostPort 回包产物，而不是安全组或 CubeProxy Redis 路由逻辑。

## 常用排查命令

查看 Cube 组件：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get deploy,ds,svc,pod -o wide'
```

查看 AgentWay 对象：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n agent-way-system get agents.agent.agentway.io -o wide'
```

查看 Cube provider：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl get agentsandboxproviders.agent.agentway.io cube -o yaml'
```

查看 `cube-node` 日志：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system logs -l app.kubernetes.io/component=cube-node -c cube-node --tail=200'
```

查看 `cube-proxy-node` 日志：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system logs -l app.kubernetes.io/component=cube-proxy-node --tail=200'
```

查看 DaemonSet 镜像：

```bash
rtk /tmp/cube_ssh.exp 106.53.31.91 \
  'kubectl -n cube-system get ds cube-node cube-proxy-node cube-dns -o custom-columns=NAME:.metadata.name,READY:.status.numberReady,DESIRED:.status.desiredNumberScheduled,IMAGES:.spec.template.spec.containers[*].image'
```

在节点上确认 hostPort 监听：

```bash
ssh root@106.53.31.91
ssh root@10.2.5.16
ss -lntp | grep 20005
```

抓非 owner 到 owner hostPort 链路：

```bash
# owner 节点 10.2.5.16 上执行
tcpdump -ni any 'host 10.2.5.189 and port 20005'

# 非 owner 节点 10.2.5.189 上执行
tcpdump -ni any 'host 10.2.5.16 and port 20005'
```

## 当前已知状态

当前集群已经切到已验证修复镜像：

```text
cube-node image: ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0-cubevsfix-20260627
current compute nodes: 10.2.5.16 / 10.2.5.189
historical non-owner proxy 10.2.5.238 -> 200
historical owner proxy     10.2.5.138 -> 200
OpenClaw Agent cube-openclaw-placement-verify -> Running / serviceHealth.ready=true / HTTP 200
OpenClaw Agent cube-dns-proxy-verify -> Running / serviceHealth.ready=true / HTTP 200 after loopback access proxy
```

当前 Chart 方案状态：

```text
control placement: cube-master / cube-api / cube-webui / cube-cubemastercli / cube-mysql / cube-redis
compute placement: cube-node / cube-proxy-node / cube-dns
timezone: TZ=Asia/Shanghai
state storage: cube-cbs-wffc PVC
```

如果需要重新复现旧问题，需要显式把 `cube-node` 回滚到旧 `ccr.ccs.tencentyun.com/pavleli/cube-node:v0.4.0`，并新建 fresh Agent 后再测 owner / non-owner 两条链路。不要使用滚动前的旧 Agent 作为唯一判断依据。

相关背景文档：

```text
deploy/kubernetes/chart/docs/plan/CUBEVS_HOSTPORT_CHECKSUM_FIX_REPORT.md
deploy/kubernetes/chart/docs/plan/CUBE_ON_K8S_NON_OWNER_ROOT_CAUSE_REPORT.md
deploy/kubernetes/chart/docs/plan/ONE_CLICK_NON_OWNER_LINK_DEPLOYMENT_REPORT.md
deploy/kubernetes/chart/docs/plan/FINAL_ONE_CLICK_PARITY_PLAN.md
deploy/kubernetes/chart/docs/ARCHITECTURE.md
```
