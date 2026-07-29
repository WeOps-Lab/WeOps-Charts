# WeOps Kubernetes 采集器 Chart

Helm Chart 目录 `weops-kubernetes-collect`（版本 `3.11.3`），部署 cAdvisor、node-exporter、kube-state-metrics 并把指标写入远端 Prometheus / VictoriaMetrics。本文只覆盖 CNI netns 占用修复与安装直接相关的字段。

## 本次修复

- cAdvisor 不再挂载 `/var/run/netns`、移除空操作 `preStop`；node-exporter 不再把宿主机 `/` 递归挂到容器 `/host`，改为 `/etc → /host`、`/proc → /host-proc`、`/sys → /host-sys`，保留文件系统容量等指标。
- node-exporter 通过 nginx sidecar 仅暴露 `/metrics`；`/debug/pprof/` 返回 `403`，pprof 被阻断。
- cAdvisor 与 node-exporter 使用 `values.yaml` 中的国内镜像源，容器与文件系统相关指标保留。
- Docker 默认不显式设置 runtime；containerd 需 `--set cadvisor-exporter.container_runtime=containerd`。
- 升级不会自动清理升级前已残留的 CNI namespace 或长期 `Terminating` Pod，需按运行时/CNI 流程单独处理。

## 查看 containerd 路径

仅当节点使用 containerd 时需要确认。所有查询在**每个将运行 cAdvisor 的节点**上执行，必要时加 `sudo`。

### 节点运行时

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion'
```

### root / state

```bash
ps -ef | grep [c]ontainerd
containerd config dump | grep -E '^root =|^state ='
```

### socket

```bash
find /run /var/run -type s -name containerd.sock 2>/dev/null
```

如不可用，从 kubelet 的 `--container-runtime-endpoint` 或 `containerRuntimeEndpoint` 反查。

### 常见默认值

| 项目 | 常见默认路径 | Chart 字段 |
| --- | --- | --- |
| root | `/var/lib/containerd` | `cadvisor-exporter.containerd.dataRoot.hostPath` |
| state | `/run/containerd` | `cadvisor-exporter.containerd.state.hostPath` |
| socket | `/run/containerd/containerd.sock` | `cadvisor-exporter.containerd.socketPath` |

以每个节点实际结果为准，所有目标节点的 root、state、socket 路径应保持一致。

## Helm 安装

命令在仓库根目录的 `weops/` 下执行；Chart 目录名 `weops-kubernetes-collect`，以 `./weops-kubernetes-collect` 引用。`remoteWrtieUrl` 是项目历史拼写，命令中不要改写。

### Docker 默认

`cadvisor-exporter.container_runtime` 默认 `docker`，无需显式设置：

```bash
helm install weops-collecter ./weops-kubernetes-collect \
  -n weops --create-namespace \
  --set-string remoteWrtieUrl='http://<用户名>:<密码>@<地址>:9093/api/v1/write' \
  --set-string kubeClusterLabel='<集群标识>' \
  --wait --timeout 15m
```

### containerd 默认路径

所有目标节点 root/state/socket 与上文默认值一致时：

```bash
helm install weops-collecter ./weops-kubernetes-collect \
  -n weops --create-namespace \
  --set-string remoteWrtieUrl='http://<用户名>:<密码>@<地址>:9093/api/v1/write' \
  --set-string kubeClusterLabel='<集群标识>' \
  --set cadvisor-exporter.container_runtime=containerd \
  --wait --timeout 15m
```

### containerd 自定义路径

模板只把 `state` 目录以 `hostPath → mountPath` 形式挂入容器，`socketPath` 必须位于 `state.mountPath` 之下。假设宿主机 root `/data/containerd`、state `/data/containerd-run`、socket `/data/containerd-run/containerd.sock`，容器内 `mountPath` 与 `hostPath` 保持同路径：

```bash
helm install weops-collecter ./weops-kubernetes-collect \
  -n weops --create-namespace \
  --set-string remoteWrtieUrl='http://<用户名>:<密码>@<地址>:9093/api/v1/write' \
  --set-string kubeClusterLabel='<集群标识>' \
  --set cadvisor-exporter.container_runtime=containerd \
  --set cadvisor-exporter.containerd.dataRoot.hostPath=/data/containerd \
  --set cadvisor-exporter.containerd.dataRoot.mountPath=/data/containerd \
  --set cadvisor-exporter.containerd.state.hostPath=/data/containerd-run \
  --set cadvisor-exporter.containerd.state.mountPath=/data/containerd-run \
  --set cadvisor-exporter.containerd.socketPath=/data/containerd-run/containerd.sock \
  --wait --timeout 15m
```

> URL 含凭据会写入 shell 历史，生产环境建议改用私有 values 文件。release `weops-collecter` 已存在时，把 `helm install` 改为 `helm upgrade --install` 即可。

## 安装后确认

```bash
kubectl get pods -n weops -o wide
sudo grep -E 'netns|cni-' "/proc/<cAdvisor 容器宿主机 PID>/mountinfo"
journalctl -u kubelet --since '<安装时间>' | grep -E 'device or resource busy|remove netns|PLEG'
```

期望：Pods `Running`；mountinfo 无 `netns`/`cni-` 行；kubelet 自本次安装起无新增 `device or resource busy` / `remove netns` / PLEG 异常。