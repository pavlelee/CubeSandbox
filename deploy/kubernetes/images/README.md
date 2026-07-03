# CubeSandbox delivery images

This directory contains image build definitions used by the Kubernetes/TKE chart.

## Build entrypoint

```bash
PUSH=1 REGISTRY=<registry>/<namespace> IMAGE_TAG=v0.4.0 ./deploy/images/build-cube-images.sh
```

Use `NO_CACHE=1` when every Docker image layer must be rebuilt instead of
using Docker's build cache:

```bash
NO_CACHE=1 PUSH=1 REGISTRY=<registry>/<namespace> IMAGE_TAG=v0.4.0 ./deploy/images/build-cube-images.sh
```

The script reuses valid artifacts already present under `deploy/images/.work/downloads`
and does not require a `.complete` marker.

## Image source policy

- `cube-node` continues to use `deploy/images/cube-node/Dockerfile`.
  It is a Kubernetes delivery image that bundles the node-side runtime components required by the Cube Node Big Pod, including `Cubelet`, `network-agent`, `cube-shim`, `cube-kernel-scf`, `cube-image`, `cube-vs`, and `cube-snapshot`. `cube-egress` is intentionally not bundled in this image because it is delivered as a separate sidecar image.
- `cube-node-init` and `cube-pvm-host-bootstrap` are Kubernetes Init Container images and stay in this delivery image directory.
- `cube-master` is built directly from `CubeMaster/docker/Dockerfile`. The build script prepares a temporary Docker context with the release-package `cubemaster` binary and the `CubeMaster/docker/tools` directory expected by that Dockerfile.
- `cube-api` is built from `CubeAPI/Dockerfile`; no duplicate Dockerfile is kept here.
- `cubemastercli` is an operational CLI image. It packages only the
  release-package `CubeMaster/bin/cubemastercli` binary and minimal runtime
  dependencies. It is separate from `cube-master` and `cube-node` so runtime
  image responsibilities remain clean.
- `cube-proxy-node` is built from `CubeProxy/Dockerfile`; no duplicate Dockerfile is kept here.
- `cube-egress` is built from `CubeEgress/Dockerfile`; no duplicate Dockerfile is kept here. Its `cube-egress/openresty:1.29.2.5-tproxy` base image is built first from `CubeEgress/openresty/Dockerfile`, because that patched OpenResty base is part of the upstream CubeEgress build chain rather than a public pull-only dependency.
- `cube-egress-net` is a Kubernetes helper image that owns the host TPROXY
  iptables/ip-rule setup for CubeEgress. It packages the upstream
  `CubeEgress/scripts/cube-proxy-iptables-init.sh` plus a small idempotent
  entrypoint that waits for `cube-dev`, applies rules, and removes them on
  termination.
- `cube-webui` packages the one-click `webui/dist` static assets and reuses
  the one-click WebUI nginx layout. The chart mounts a rendered nginx config so
  `/cubeapi/` proxies to the chart-managed CubeAPI Service.
- The template builder sidecar uses a dind image by default; no duplicate Dockerfile is kept here.

The Helm chart stays under `deploy/k8s/chart`; image build logic stays here to avoid coupling chart templates with image construction.

`build-cube-images.sh` copies only the scripts required by each image into that image's build context. Do not add generic helper scripts here unless they are referenced by a Dockerfile or explicitly copied by the build script.

CubeMaster runtime configuration is delivered by the Helm chart from `deploy/k8s/chart/files/cube-master/conf.yaml` as a Secret mounted at `/usr/local/services/cubemaster/conf.yaml`. CubeMaster schema migrations are embedded in the `cubemaster` binary at compile time from `CubeMaster/pkg/base/dao/migrate/migrations/mysql`; this image build does not package a second SQL copy.
