# CubeSandbox Chart 与 one-click 能力对齐差异计划

## 背景

本计划记录 `deploy/k8s/chart` 与 one-click 安装包之间的能力对齐差异，来源于对以下内容的逐项对比：

- `deploy/images/.work/extract/cube-sandbox-one-click-v0.4.0/install.sh`
- `deploy/images/.work/extract/cube-sandbox-one-click-v0.4.0/env.example`
- `deploy/images/.work/sandbox-package/scripts/one-click/*`
- `deploy/images/.work/sandbox-package/scripts/systemd/*`
- `deploy/images/.work/sandbox-package/systemd/*`
- `deploy/images/.work/sandbox-package/CubeMaster/conf.yaml`
- `deploy/k8s/chart/values.yaml`
- `deploy/k8s/chart/templates/*`
- `deploy/images/*`

目标是把 one-click 已具备、但 Chart 交付仍未完全等价的能力显式沉淀为计划，便于后续逐项修复和验收。本文只记录能力差异、Kubernetes 化后的目标形态和验收标准，不包含 demo 节点、demo 数据，也不要求 Chart 给节点自动打标签。

## 当前已对齐或已明确的原则

- Cube Node 按 DaemonSet Big Pod 交付。
- Cube Node 只在使用方已打好 label 的节点上启动，Chart 仅通过 `placement.nodeLabelSelector` 选择节点。
- 初始化镜像和运行镜像职责分离：PVM Host Kernel 安装/重启、节点预检、Cube Node 运行分别由独立镜像承担。
- Master 组件、Node 组件、Proxy、Egress、WebUI 镜像职责分离，Node 镜像只包含 Cube Node 必需组件。
- CubeMaster 复用 `CubeMaster/docker/Dockerfile`，数据库 schema 迁移由 CubeMaster 内置逻辑负责，不在 Chart 中交付独立迁移镜像或 Master CLI。
- Chart 复用 one-click 包中的 `CubeMaster/conf.yaml`，通过 ConfigMap/Secret 方式挂载给 CubeMaster。
- MySQL / Redis 支持第三方服务；配置第三方服务时，不安装 Chart 内置 MySQL / Redis。
- `cube-proxy-node` 属于 Cube Chart 能力范围，需要随 Chart 一起交付。
- `cube-egress` 属于 Cube Node Big Pod sidecar，网络初始化能力由独立网络初始化容器承担。
- 不在 Chart 中放置固定节点 IP、demo seed 数据、节点打标签 Job。
- 镜像正常构建、推送、拉取，不通过临时绕过方式交付。

## 能力不对齐清单

| 优先级 | 能力项 | one-click 行为 | Chart / 镜像目标形态 | 未对齐影响 | 修复动作 |
| --- | --- | --- | --- | --- | --- |
| P0 | Cube Node runtime 工具固定路径 | 安装时创建 `/usr/local/bin/containerd-shim-cube-rs`、`/usr/local/bin/cube-runtime`、`/usr/local/bin/cubecli`、`/usr/local/bin/cubevsmapdump` 软链 | `cube-node` 镜像内直接具备等价软链 | 创建 sandbox、snapshot 或 runtime 调用固定路径时失败 | 在 `deploy/images/cube-node/Dockerfile` 中创建软链，并在启动脚本中做预检 |
| P0 | Cube Node 镜像职责边界 | one-click 本机安装会复制 compute 节点全部目录，其中包含 egress 相关内容 | Kubernetes 下 egress 已是 sidecar，`cube-node` 镜像只保留 Node 必需组件 | 镜像职责混乱，后续排障和升级边界不清晰 | 从 `cube-node` 镜像和 build context 中移除 egress 目录 |
| P0 | PVM guest kernel 选择 | 根据 `CUBE_PVM_ENABLE` 选择 `cube-kernel-scf/vmlinux -> vmlinux-bm` 或 `vmlinux-pvm` | Chart 暴露明确 value，由 `cube-node` 启动时选择 guest kernel 软链 | 非 PVM 环境或 PVM 环境可能使用错误 guest kernel | 增加 `cubeNode.pvmGuestKernel.enabled`，启动时设置并校验软链 |
| P0 | PVM Host Kernel 安装和节点重启 | one-click 在宿主机上安装 host kernel，并在需要时提示/执行重启 | Chart 通过独立 Init Container 执行 host kernel 安装与重启控制 | 节点未完成 host kernel 初始化时直接启动 Cube Node，后续 VM 启动失败 | 保持 `cube-pvm-host-bootstrap` 为独立 init 镜像，状态可观测，失败即阻断主容器 |
| P0 | CubeMaster artifact storage 持久化 | one-click 使用宿主机 `/data/CubeMaster/storage` | Chart 默认使用 hostPath，并支持 PVC、existingClaim 或 emptyDir | CubeMaster Pod 重建后模板/rootfs artifact 丢失 | 增加 `controlPlane.master.persistence` 并在 README 中标明生产建议 |
| P0/P1 | Cube Node 初始化预检完整性 | 检查 KVM、内存、XFS、cgroup v2 cpu、glibc、cubecow 依赖、CIDR、主网卡等 | `cube-node-init` 覆盖关键前置条件，支持显式配置或自动探测主网卡 | 异常节点进入运行期才失败，定位成本高 | 补齐预检项；失败时在 init 日志中输出明确原因 |
| P1 | CubeProxy 生命周期 | one-click 将 cube proxy 作为核心能力启动，并提供 80/443 入口，入口天然是显式节点 host proxy | Chart 以 DaemonSet 交付 `cube-proxy-node`，通过 hostNetwork 提供节点本地入口，不创建 ClusterIP Service | 缺少统一入口时，sandbox 域名访问不可用；多节点 ClusterIP 随机分流会偏离 one-click 的明确 CubeProxy host 入口模型 | 将 `cube-proxy-node` 纳入 Chart 默认交付，并要求外部 DNS/LB 指向明确 CubeProxy 入口 |
| P1 | CubeProxy TLS 证书 | one-click 通过本地证书目录和 mkcert 生成/挂载证书，支持自定义证书文件 | Chart 需要支持自签、用户 Secret、关闭 TLS 三种模式；生产推荐用户 Secret | TLS 证书不可控会导致域名访问、浏览器信任链和运维替换不完整 | 增加 `cubeProxy.tls.mode`、SAN 配置、existingSecret/secretName，并在 README 明确生产配置 |
| P1 | DNS / CoreDNS / host DNS routing | one-click 部署 CoreDNS，并调整 host DNS routing，让 `cube.app` / wildcard 解析到 CubeProxy | Kubernetes 默认不接管宿主机 DNS；可选交付 CoreDNS Service，并要求使用方配置上游 DNS 或客户端 DNS | 离线/内网环境需要额外 DNS 配置；Chart 体验不完全等价 one-click | 新增可选 `cubeDns`，默认不修改宿主机 DNS，在 README 明确接入方式 |
| P1 | WebUI 交付 | one-click 默认启动 WebUI nginx，并代理 `/cubeapi` 到 CubeAPI | Chart 新增 WebUI Deployment / Service / 镜像配置，并代理到 Chart 内 CubeAPI | 缺少控制台入口，管理体验不完整 | 增加 `webui.enabled`、镜像、Service、nginx 配置和健康检查 |
| P1 | CubeEgress 与透明代理网络 | one-click 包含 egress worker、证书准备、TPROXY、ip rule、sysctl 规则 | Chart 中 egress worker 作为 sidecar，网络规则由独立 init 容器安装，证书由 Secret 管理 | egress 出站、回包路径和动态证书能力不完整 | 保持 worker 与网络初始化分离，补齐 CA/placeholder 证书 Secret 和健康检查 |
| P1 | Helm test / quickcheck 覆盖度 | one-click `quickcheck.sh` 覆盖服务进程、健康接口、节点注册、runtime asset、proxy、egress、WebUI | Chart test 应覆盖 Master、API、Node、Proxy、Egress、WebUI、DNS 的关键路径 | 安装完成但关键链路不可用时无法及时发现 | 增强 `templates/tests`，并补充人工验收命令 |
| P2 | 诊断脚本交付 | one-click 提供 `cube-diag` 依赖检查、进程检查、日志收集 | Chart 未完整交付 Kubernetes 化诊断工具 | 故障排查体验不如 one-click | 后续设计 debug image 或 ConfigMap 脚本，使用 kubectl 收集 Pod/Node 诊断信息 |
| P2 | 停止/卸载语义 | one-click stop 会停止本机服务并回滚 host DNS routing | Helm uninstall 删除 K8s 资源，但不应隐式修改使用方节点标签或外部 DNS | 卸载后遗留规则、证书或 hostPath 数据需要明确处理 | README 增加卸载后清理说明；HostPath 数据和节点 label 由使用方处理 |

## 分阶段修复计划

### 第一阶段：核心运行链路必须对齐

1. **Cube Node 镜像修正**
   - 创建 one-click 等价 runtime 工具软链。
   - 移除 Node 镜像中的 egress 内容。
   - 启动时校验 `cube-runtime`、`containerd-shim-cube-rs`、`cubecli`、`cubevsmapdump` 等命令存在。
2. **PVM guest kernel 选择**
   - 新增 Chart value 控制是否启用 PVM guest kernel。
   - `cube-node-entrypoint.sh` 根据 value 设置 `cube-kernel-scf/vmlinux` 指向。
   - 启动日志打印最终使用的 guest kernel。
3. **PVM Host Kernel bootstrap**
   - 保持独立 init 镜像。
   - Init 成功后主容器才启动。
   - 需要重启时通过 init 容器日志和 Pod 状态显式呈现。
4. **CubeMaster storage 持久化**
   - 支持 `emptyDir`、`hostPath`、`existingClaim`、Chart 创建 PVC。
   - 默认使用 hostPath；生产环境可切换到 PVC 或 existingClaim。

### 第二阶段：安装前置检查和安装验收对齐

1. **增强 `cube-node-init`**
   - 检查 KVM、XFS、内存、glibc、cgroup v2 cpu。
   - 检查 cubecow 依赖命令。
   - 校验网络 CIDR 格式，必要时检测冲突。
   - 支持主网卡自动探测，也支持 value 显式指定。
2. **增强 Helm test**
   - CubeMaster `/notify/health`。
   - CubeAPI `/health` 和节点列表。
   - Cube Node DaemonSet ready 状态。
   - CubeEgress admin health。
   - WebUI 首页和 `/cubeapi` 代理。
   - DNS 解析链路。
   - CubeProxy 数据面入口；admin health 如仅监听 loopback，则通过 Pod 内检查。

### 第三阶段：one-click 周边能力 Kubernetes 化

1. **CubeProxy TLS**
   - 默认可使用自签证书用于测试环境。
   - 生产环境使用用户提供的 Secret。
   - Secret key 名、域名 SAN、IP SAN 可配置。
2. **DNS 能力**
   - Chart 可选部署 CoreDNS Service。
   - 不默认修改宿主机 `/etc/resolv.conf` 或 NetworkManager 配置。
   - 由使用方将 wildcard DNS 指向 CubeProxy 入口或将上游 DNS 指到 `cubeDns` Service。
3. **WebUI**
   - 以独立镜像交付静态资源和 nginx 配置。
   - `/cubeapi` 代理到 CubeAPI Service。
4. **诊断工具**
   - 将 one-click 的诊断思路转换为 Kubernetes 诊断脚本。
   - 输出 Pod、DaemonSet、Service、事件、init 日志、Node 预检结果、proxy/egress 关键健康信息。

## 当前修复状态

- P0/P1 核心运行链路已落到 Chart 和镜像中：Cube Node Big Pod、PVM Host Kernel bootstrap、Cube Node 初始化、PVM guest kernel 选择、runtime 工具软链、CubeMaster 持久化配置、CubeProxy、CubeEgress、WebUI、CubeDNS、TLS Secret/自签/证书管理模式均已纳入 Chart。
- CubeProxy 已按 One Click 数据面语义默认使用 `hostNetwork=true` 并监听节点 `80/443`，保证 node-local `cube-dns` 返回节点 HostIP 后，Proxy 可以同时接入外部流量并直连本机 sandbox bridge IP；Chart 不创建 proxy ClusterIP Service，避免随机分流形成第二套数据面语义。同时 Chart 在 nginx `global.conf` 中写入 resolver，支持 Lua Redis 客户端解析 Kubernetes/第三方 Redis DNS 名称。
- `cube-node` 镜像已移除 `cube-egress` 内容，`cube-egress` 和 `cube-egress-net` 作为 Big Pod sidecar 独立交付。
- CubeMaster 不再依赖独立 DB 迁移 Job 或 Master CLI；迁移由 CubeMaster 内置逻辑执行。
- Helm test 已覆盖 CubeMaster、CubeAPI、节点注册、WebUI、CubeProxy 数据面、CubeDNS 解析；CubeProxy/CubeEgress 的 loopback admin 健康通过 Pod readiness/liveness 和人工验证命令检查，不再暴露无效 ClusterIP admin Service。
- one-click 的 `cube-diag` 能力已转换为 `diagnostics.enabled=true` 下的 Kubernetes 诊断 ConfigMap，使用 `kubectl`/`helm` 收集 release 状态与组件日志。
- 模板构建和沙箱生命周期已完成真实集群验证：通过 CubeAPI 从可访问镜像仓库创建模板，等待模板构建 `ready`，创建 sandbox，验证查询、暂停、恢复/连接、删除流程，最后清理模板。
- Kubernetes 场景仍不默认修改宿主机 DNS，这是有意差异：默认 node-local `cube-dns` 只对显式使用 `dnsConfig.nameservers: [127.0.0.54]` 的 `cube-node` Pod 生效；宿主机 DNS、外部 DNS、Ingress 或负载均衡接入由使用方显式配置。
- `cube-node-init` 已补齐 PVM 一致性预检：`kvm_pvm` 与 `cubeNode.pvmGuestKernel.enabled` 不匹配时 fail-fast。

## 非目标

- 不在 Chart 中自动给节点打 label、taint 或修改使用方节点规划。
- 不在 Chart 中内置 demo IP、One Click 单节点 seed SQL 或固定环境数据。
- 不在 Kubernetes 默认安装路径中强行接管宿主机 DNS。
- 不把 Master、Node、Proxy、Egress、WebUI 混合到同一个镜像中。
- 不通过手工导入、缓存兜底或跳过构建/拉取的方式交付镜像。
- 不修改 `deploy/` 目录之外的代码来完成本轮 Chart 能力对齐。

## 验收标准

- `helm lint deploy/k8s/chart` 通过。
- `helm template` 输出不包含 demo IP、节点打标签 Job、独立 DB 迁移资源或 Master CLI 交付资源。
- 所有 Chart 使用的自研镜像均按规范 tag 构建并推送，集群可正常拉取。
- `cube-node` 镜像内不存在 egress 目录，存在 one-click 等价 runtime 工具软链。
- `cube-node` 启动日志能看到最终 guest kernel 选择结果。
- `cube-node-init` 能在异常节点上提前失败，并输出明确失败原因。
- `cube-master` 可配置持久化 storage，Pod 重建后 artifact 不因默认临时卷而误用于生产。
- 启用 `cubeProxy.enabled=true` 时，CubeProxy Pod Ready，TLS 配置符合 value。
- 启用 `cubeEgress.enabled=true` 时，egress worker 和网络初始化均 Ready，健康检查通过。
- 启用 `webui.enabled=true` 时，WebUI 首页和 `/cubeapi` 代理可访问。
- 启用 `cubeDns.enabled=true` 时，目标域名解析到 CubeProxy 入口或配置的目标地址。
- `helm test` 覆盖 Master、API、Node、Proxy、Egress、WebUI、DNS 的关键健康路径。
