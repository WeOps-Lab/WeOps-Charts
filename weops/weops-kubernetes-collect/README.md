# WeOps Kubernetes 采集器部署与验证指南

本文面向负责 Kubernetes 集群部署与故障处理的运维人员，说明采集器的 CNI namespace 占用问题、修复内容、运行时路径确认方法、Helm 安装升级步骤和安装后验证方法。

> **适用范围**：本文对应当前目录中的 Chart。目录名为 `weops-kubernetes-collect`，但 `Chart.yaml` 中的实际 Chart 名称是历史拼写 `weops-kubenetes-collect`，版本为 `3.11.3`。执行命令时以当前目录和实际 `Chart.yaml` 为准，不要自行改写 Chart 名称。

## 1. 问题与修复说明

### 1.1 问题表现和故障链

旧版采集器存在以下宿主机挂载：

- cAdvisor 显式挂载宿主机 `/var/run/netns`，可能持续持有 CNI 创建的 network namespace。
- node-exporter 将宿主机 `/` 递归挂载到容器内 `/host`，可能间接持有 `/host/run/netns/cni-*`。

当容器运行时或 CNI 删除 namespace 时，仍被采集器进程持有的挂载可能导致以下故障链：

1. namespace 删除失败，日志出现 `device or resource busy`、`unlinkat` 或 `remove netns` 等错误。
2. Pod 长期停留在 `Terminating`，并可能出现 `KillPodSandbox` 或 PLEG 异常。
3. 异常持续累积后，节点可能无法创建新容器。

### 1.2 当前修复

当前 Chart 已实施以下调整：

- 删除 cAdvisor 对宿主机 `/var/run/netns` 的显式挂载。
- 删除 cAdvisor 中仅执行 `sleep 15` 的无效 `preStop`。
- node-exporter 不再将宿主机 `/` 递归挂载到 `/host`，改为：
  - 宿主机 `/etc` 挂载到容器 `/host`，用于报告宿主机根文件系统所在设备的容量指标。
  - 宿主机 `/proc` 挂载到容器 `/host-proc`。
  - 宿主机 `/sys` 挂载到容器 `/host-sys`。
- node-exporter 保留 `node_filesystem_*` 文件系统指标，并排除未显式挂载的虚拟目录和容器运行时目录。
- node-exporter 继续通过 nginx sidecar 对外提供 `/metrics`，`/debug`（包括 pprof）返回 HTTP `403`。
- cAdvisor、node-exporter 和 nginx 使用当前 `values.yaml` 中配置的国内镜像地址。
- `/boot` 和 `/data` 改为可选挂载，默认均关闭。

> **重要说明**：升级 Chart 只能更新采集器工作负载，不会自动处理升级前已经残留的 CNI namespace 或长期 `Terminating` Pod。历史残留必须单独确认占用者，并按照集群的容器运行时和 CNI 运维流程处理。

## 2. 安装前检查

### 2.1 检查 Helm 和 Kubernetes 访问

以下命令应在能够访问目标集群、并已配置正确 kubeconfig 的终端中执行：

```bash
helm version --short
kubectl version --client
kubectl cluster-info
kubectl get nodes
kubectl auth can-i create daemonsets.apps --namespace weops
kubectl auth can-i create deployments.apps --namespace weops
kubectl auth can-i create configmaps --namespace weops
kubectl auth can-i create services --namespace weops
kubectl auth can-i create clusterroles.rbac.authorization.k8s.io
kubectl auth can-i create clusterrolebindings.rbac.authorization.k8s.io
```

预期结果：

- `helm` 和 `kubectl` 命令可正常执行。
- `kubectl cluster-info` 能访问目标集群。
- 节点处于预期状态。
- 上述 `kubectl auth can-i` 命令均返回 `yes`。默认部署包含集群级 RBAC 资源；如果返回 `no`，应先由集群管理员核对并补充所需权限，不要继续安装。

仓库根目录的现有示例使用 namespace `weops`，本文沿用该名称。如果实际环境必须使用其他 namespace，请将本文所有命令中的 `weops` 统一替换为实际名称。

### 2.2 判断节点使用 Docker 还是 containerd

`kubectl get nodes -o wide` 在部分 Kubernetes 版本中不会直接显示容器运行时。Kubernetes 1.23 可使用 `nodeInfo.containerRuntimeVersion` 查询。

查询单个节点：

```bash
NODE='<节点名称>'
kubectl get node "${NODE}" -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}{"\n"}'
```

批量查询所有节点：

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion'
```

返回值通常以 `docker://` 或 `containerd://` 开头。应检查所有目标节点，不能只检查一个节点后假定全群一致。

## 3. 查询 containerd 实际路径

本节命令需要登录到**每个将运行 cAdvisor 的节点**执行。根据节点权限配置决定是否需要 `sudo`。

### 3.1 查看进程和 systemd 启动参数

```bash
ps -ef | grep [c]ontainerd
systemctl cat containerd
```

重点检查：

- `containerd` 进程是否通过 `--config` 或 `-c` 指定配置文件。
- systemd unit 的 `ExecStart` 是否指定自定义配置文件、root、state 或 socket 相关参数。
- 是否存在发行版或安装工具注入的 drop-in 配置。

### 3.2 查看配置文件、root 和 state

如果当前 containerd 版本支持，可以先查看配置输出：

```bash
containerd config dump
```

如输出内容较多，可筛选 `root` 和 `state`：

```bash
containerd config dump 2>/dev/null | grep -E '^\s*(root|state)\s*='
```

对于默认配置文件位置，可执行：

```bash
grep -E '^\s*(root|state)\s*=' /etc/containerd/config.toml
```

如果 `containerd config dump` 不可用，或者 `/etc/containerd/config.toml` 不存在，应回到进程参数和 `systemctl cat containerd` 的结果，确认实际加载的配置文件。未显式配置时，常见默认值为：

| 项目 | 常见默认路径 | 对应 Chart 字段 |
| --- | --- | --- |
| root（数据目录） | `/var/lib/containerd` | `cadvisor-exporter.containerd.dataRoot.hostPath` |
| state（运行状态目录） | `/run/containerd` | `cadvisor-exporter.containerd.state.hostPath` |

这些路径只是常见默认值，不能替代节点上的实际检查。

### 3.3 查看 kubelet 的 CRI endpoint 和 socket

先检查 kubelet 进程参数和配置：

```bash
ps -ef | grep [k]ubelet
grep -R 'containerRuntimeEndpoint' /var/lib/kubelet/config.yaml /etc/systemd/system/kubelet.service.d 2>/dev/null
```

重点检查以下两种配置形式：

- kubelet 参数中的 `--container-runtime-endpoint=unix:///.../containerd.sock`。
- kubelet 配置文件中的 `containerRuntimeEndpoint: unix:///.../containerd.sock`。

如果节点安装了 `crictl` 且当前用户具有运行时访问权限，可继续检查：

```bash
crictl info
crictl config
```

`crictl` 不存在、版本不支持 `config` 子命令或权限不足时，以 kubelet 参数、kubelet 配置和 socket 实际位置为准。检查监听中的 Unix socket 和常见目录：

```bash
ss -lx | grep containerd.sock
find /run /var/run -type s -name 'containerd.sock' 2>/dev/null
```

核对关系：

- `cadvisor-exporter.containerd.state.hostPath` 是节点上的目录。
- `cadvisor-exporter.containerd.state.mountPath` 是该目录挂载到 cAdvisor 容器内的位置。
- `cadvisor-exporter.containerd.socketPath` 是 cAdvisor 容器内可见的 socket 路径，必须与 `state.mountPath` 下的实际 socket 相符。
- 默认配置中三者分别为 `/run/containerd`、`/run/containerd` 和 `/run/containerd/containerd.sock`。

### 3.4 确认所有节点路径一致

当前 Chart 使用一个 cAdvisor DaemonSet 和一组 values，不能按节点分别设置容器运行时类型或 containerd 的 root、state、socket。必须逐节点记录查询结果并比较：

- 如果所有目标节点都使用 containerd 且路径一致，可以使用同一组 Helm values。
- 如果集群混用 Docker 和 containerd，或 containerd 路径不一致，应先统一运行时配置，或者拆分为不同的 workload/values 并限定各自的目标节点。
- 不要在未拆分工作负载的情况下承诺一个 DaemonSet 能自动适配每个节点的不同运行时或路径。

## 4. Helm 安装、升级和卸载

### 4.1 进入 Chart 目录并核对 Chart

从仓库根目录执行：

```bash
cd weops/weops-kubernetes-collect
helm show chart .
helm lint .
```

`helm show chart .` 应显示 Chart 名称 `weops-kubenetes-collect`。如果 `helm lint .` 失败，应先处理其报告的问题，不要继续部署。

### 4.2 准备业务参数

当前 Chart 使用历史字段名 `remoteWrtieUrl`，拼写必须与 `values.yaml` 保持一致。建议将远程写入地址和集群标签保存在仓库外的私有 values 文件中，例如 `${HOME}/.config/weops/weops-values.yaml`：

```bash
install -d -m 700 "${HOME}/.config/weops"
cat > "${HOME}/.config/weops/weops-values.yaml" <<'EOF'
remoteWrtieUrl: "<REMOTE_WRITE_URL>"
kubeClusterLabel: "<CLUSTER_LABEL>"
EOF
chmod 600 "${HOME}/.config/weops/weops-values.yaml"
VALUES_FILE="${HOME}/.config/weops/weops-values.yaml"
```

执行安装前，必须编辑该文件，将 `<REMOTE_WRITE_URL>` 和 `<CLUSTER_LABEL>` 替换为实际值，然后执行：

```bash
test -r "${VALUES_FILE}"
helm lint . -f "${VALUES_FILE}"
```

不要将包含用户名、密码、Token 或其他凭据的 values 文件提交到仓库。当前模板会将 `remoteWrtieUrl` 渲染到 ConfigMap，还应通过 Kubernetes RBAC 限制非必要读取权限。

对于不含逗号等 Helm 分隔符的临时值，可以在 Helm 命令末尾追加 `--set-string remoteWrtieUrl='<REMOTE_WRITE_URL>'` 和 `--set-string kubeClusterLabel='<CLUSTER_LABEL>'`。这两个参数不能脱离 `helm install` 或 `helm upgrade` 单独执行。

URL 可能包含 `:`, `/`, `?`, `&`, `=`, `,` 等特殊字符。即使使用 `--set-string`，也需要正确处理 shell 引号和 Helm 转义；因此含凭据或复杂查询参数时优先使用私有 values 文件，避免泄露到 shell 历史、进程参数或提交记录。

### 4.3 Docker 默认安装

`cadvisor-exporter.container_runtime` 的默认值是 `docker`。Docker 节点不需要显式设置运行时。

推荐使用 `upgrade --install`，可同时覆盖首次安装和后续幂等更新：

```bash
helm upgrade --install weops-collecter . \
  --namespace weops \
  --create-namespace \
  -f "${VALUES_FILE}"
```

仅执行首次安装：

```bash
helm install weops-collecter . \
  --namespace weops \
  --create-namespace \
  -f "${VALUES_FILE}"
```

成功信号：Helm 命令返回成功，且后续“安装后验证”中的 Deployment 和两个 DaemonSet 均能正常就绪。

### 4.4 containerd 默认路径安装

只有确认目标节点使用 containerd 后，才显式设置运行时：

```bash
helm upgrade --install weops-collecter . \
  --namespace weops \
  --create-namespace \
  -f "${VALUES_FILE}" \
  --set cadvisor-exporter.container_runtime=containerd
```

该命令使用当前 Chart 的 containerd 默认值：

- data root：`/var/lib/containerd`，默认挂载。
- state：`/run/containerd`。
- socket：`/run/containerd/containerd.sock`。

如果任一目标节点与这些路径不一致，应使用下一节的自定义路径命令，而不是直接安装。

### 4.5 containerd 自定义 root、state 和 socket

先将以下变量替换为各节点检查后确认一致的实际路径：

```bash
CONTAINERD_ROOT='<CONTAINERD_ROOT>'
CONTAINERD_STATE='<CONTAINERD_STATE>'
CONTAINERD_SOCKET='<CONTAINERD_SOCKET_IN_CONTAINER>'
```

例如，若节点 socket 位于 `${CONTAINERD_STATE}/containerd.sock`，并且 state 以相同路径挂载到容器，则 `CONTAINERD_SOCKET` 应设置为该容器内路径。安装命令如下：

```bash
helm upgrade --install weops-collecter . \
  --namespace weops \
  --create-namespace \
  -f "${VALUES_FILE}" \
  --set cadvisor-exporter.container_runtime=containerd \
  --set cadvisor-exporter.containerd.dataRoot.enabled=true \
  --set cadvisor-exporter.containerd.dataRoot.hostPath="${CONTAINERD_ROOT}" \
  --set cadvisor-exporter.containerd.dataRoot.mountPath="${CONTAINERD_ROOT}" \
  --set cadvisor-exporter.containerd.dataRoot.hostPathType=Directory \
  --set cadvisor-exporter.containerd.state.hostPath="${CONTAINERD_STATE}" \
  --set cadvisor-exporter.containerd.state.mountPath="${CONTAINERD_STATE}" \
  --set cadvisor-exporter.containerd.state.hostPathType=Directory \
  --set cadvisor-exporter.containerd.socketPath="${CONTAINERD_SOCKET}"
```

字段含义：

| 字段 | 用途 |
| --- | --- |
| `cadvisor-exporter.container_runtime` | 运行时选择；只接受 `docker` 或 `containerd` |
| `cadvisor-exporter.containerd.dataRoot.enabled` | 是否渲染并挂载 containerd 数据目录 |
| `cadvisor-exporter.containerd.dataRoot.hostPath` | 节点上的 containerd root |
| `cadvisor-exporter.containerd.dataRoot.mountPath` | cAdvisor 容器内的数据目录挂载路径 |
| `cadvisor-exporter.containerd.dataRoot.hostPathType` | Kubernetes `hostPath.type`；当前默认值为 `Directory` |
| `cadvisor-exporter.containerd.state.hostPath` | 节点上的 containerd state 目录 |
| `cadvisor-exporter.containerd.state.mountPath` | cAdvisor 容器内的 state 目录挂载路径 |
| `cadvisor-exporter.containerd.state.hostPathType` | Kubernetes `hostPath.type`；当前默认值为 `Directory` |
| `cadvisor-exporter.containerd.socketPath` | 传给 cAdvisor `--containerd` 参数的容器内 socket 路径 |

`dataRoot.enabled` 的行为必须按模板理解：

- `true`（默认）：渲染 `containerd-data` volume 和 volumeMount。所有目标节点上的 `dataRoot.hostPath` 必须存在；由于类型是 `Directory`，路径不存在会导致 Pod 无法正常创建。
- `false`：不渲染、不挂载 `containerd-data`；`containerd-run` 仍会挂载，cAdvisor 仍通过其中的 `socketPath` 连接运行时。只有在确认当前环境不需要 cAdvisor 访问 data root，并已验证所需容器指标正常后才关闭。

关闭 data root 时，应将下列参数追加到完整的 containerd 安装或升级命令中：

```bash
--set cadvisor-exporter.containerd.dataRoot.enabled=false
```

该代码块是 Helm 参数片段，不能单独执行。如果路径包含逗号等 Helm 特殊字符，应改用 values 文件，避免 `--set` 解析错误。

### 4.6 可选 node-exporter 文件系统挂载

当前默认配置如下：

| 目录 | 开关字段 | hostPath 字段 | mountPath 字段 | 默认值 |
| --- | --- | --- | --- | --- |
| `/boot` | `node-exporter.filesystem.mounts[0].enabled` | `node-exporter.filesystem.mounts[0].hostPath`，默认 `/boot` | `node-exporter.filesystem.mounts[0].mountPath`，默认 `/boot` | `false` |
| `/data` | `node-exporter.filesystem.mounts[1].enabled` | `node-exporter.filesystem.mounts[1].hostPath`，默认 `/data` | `node-exporter.filesystem.mounts[1].mountPath`，默认 `/data` | `false` |

模板会将可选目录挂载到 `filesystem.rootMount.mountPath + mountPath`。按默认值，宿主机 `/boot` 和 `/data` 分别出现在容器内 `/host/boot` 和 `/host/data`。

只有所有目标节点都存在相应目录时才启用。由于 Helm 的数组参数容易整体覆盖列表，推荐在私有 values 文件中保留完整条目后修改 `enabled`，例如：

```yaml
node-exporter:
  filesystem:
    mounts:
      - name: boot
        hostPath: /boot
        mountPath: /boot
        enabled: true
      - name: data
        hostPath: /data
        mountPath: /data
        enabled: false
```

如果目标节点目录不一致，应保持关闭，或者拆分工作负载和 values；当前 DaemonSet 不能按节点自动选择不同 hostPath。

### 4.7 升级、查看历史和回滚

升级现有 Docker release：

```bash
helm upgrade weops-collecter . \
  --namespace weops \
  -f "${VALUES_FILE}"
```

containerd 环境每次升级都应继续传入相同的运行时和路径参数，或者将这些参数固化到受控的私有 values 文件中。不要在升级时遗漏 `cadvisor-exporter.container_runtime=containerd`，否则本次渲染会恢复 Chart 的 Docker 默认值。

查看当前配置和发布历史：

```bash
helm get values weops-collecter --namespace weops --all
helm history weops-collecter --namespace weops
```

从 `helm history` 输出中选择确认可用的 revision，替换变量值后执行回滚：

```bash
REVISION='<目标 revision>'
helm rollback weops-collecter "${REVISION}" --namespace weops --wait
```

回滚后必须重新执行安装后验证。若升级失败且新 Pod 无法就绪，可先保留现场信息，再回滚到上一可用 revision。

### 4.8 卸载

```bash
helm uninstall weops-collecter --namespace weops
```

卸载前确认该 release 确实属于当前集群和 namespace。卸载不会自动清理由旧版本或其他组件造成的历史 netns 残留。

## 5. 安装后验证

### 5.1 检查工作负载和 Pod

```bash
kubectl get deployment,daemonset --namespace weops
kubectl get pods --namespace weops -o wide
kubectl rollout status daemonset/cadvisor-exporter --namespace weops
kubectl rollout status daemonset/node-exporter --namespace weops
kubectl rollout status deployment/weops-flow --namespace weops
```

预期结果：

- `cadvisor-exporter` 和 `node-exporter` DaemonSet 在所有目标节点均就绪。
- `weops-flow` Deployment 就绪。
- 相关 Pod 为 `Running`，容器 Ready 数符合 Pod 定义。

如果未就绪，先执行以下命令保留事件和日志，再决定回滚：

```bash
kubectl describe daemonset cadvisor-exporter --namespace weops
kubectl describe daemonset node-exporter --namespace weops
kubectl get events --namespace weops --sort-by='.lastTimestamp'
kubectl logs --namespace weops --selector app=cadvisor-exporter --tail=200
kubectl logs --namespace weops --selector app=node-exporter --all-containers --tail=200
```

### 5.2 核对渲染后的参数和挂载

查看 Helm 实际发布的 manifest，以及集群中的 DaemonSet 和 Pod：

```bash
helm get manifest weops-collecter --namespace weops
kubectl get daemonset cadvisor-exporter --namespace weops -o yaml
kubectl get daemonset node-exporter --namespace weops -o yaml
kubectl get pods --namespace weops --selector app=cadvisor-exporter -o yaml
kubectl get pods --namespace weops --selector app=node-exporter -o yaml
```

至少确认：

- cAdvisor 不包含 `/var/run/netns` volume 或 volumeMount，也不包含 `sleep 15` 的 `preStop`。
- Docker 环境包含 `--docker=unix:///var/run/docker.sock`；containerd 环境包含与预期一致的 `--containerd=<socketPath>`。
- node-exporter 参数包含 `--path.rootfs=/host`、`--path.procfs=/host-proc` 和 `--path.sysfs=/host-sys`。
- node-exporter 不再把宿主机 `/` 挂载到 `/host`；`filesystem-root` 的宿主机路径应为 `/etc`。
- 未启用可选挂载时，不应出现 `/host/boot` 或 `/host/data`。

### 5.3 检查采集器进程的 mountinfo

在每个目标节点上查找 cAdvisor 和 node-exporter 容器。containerd/CRI 环境可使用：

```bash
sudo crictl ps --name cadvisor-exporter
sudo crictl ps --name node-exporter
CONTAINER_ID='<上一步查到的容器 ID>'
sudo crictl inspect "${CONTAINER_ID}" | grep -m 1 '"pid"'
```

Docker 环境可使用：

```bash
docker ps --filter name=cadvisor-exporter
docker ps --filter name=node-exporter
CONTAINER_ID='<上一步查到的容器 ID>'
docker inspect --format '{{.State.Pid}}' "${CONTAINER_ID}"
```

取得容器宿主机 PID 后，分别检查两个采集器进程：

```bash
PID='<inspect 输出中的宿主机 PID>'
sudo grep -E '(/var)?/run/netns|cni-' "/proc/${PID}/mountinfo"
```

预期结果：命令无输出，即采集器进程未持有 CNI netns 挂载。如果仍有输出，先记录对应 PID、容器、挂载点和节点，不要直接删除 namespace 或强制卸载；应确认是否仍运行旧 Pod、是否存在其他持有进程，再按运行时/CNI 运维流程处置。

### 5.4 验证 Pod 创建、Ready 和正常删除

选择一个集群能够拉取的最小测试镜像并替换 `<TEST_IMAGE>`，同时将 `<NODE_NAME>` 替换为目标节点名称。以下流程使用唯一的临时 namespace；只有 namespace 创建成功后才注册清理函数，异常退出时也只清理本次创建的测试资源。

```bash
set -euo pipefail

NODE_NAME="<NODE_NAME>"
TEST_IMAGE="<TEST_IMAGE>"
VERIFY_NS="weops-collect-verify-$(date +%s)-$$"
TEST_POD="netns-delete-check"
NS_CREATED=false

if [[ "${NODE_NAME}" == "<NODE_NAME>" || -z "${NODE_NAME}" ]]; then
  echo "错误：请将 NODE_NAME 替换为目标节点名称" >&2
  exit 1
fi
if [[ "${TEST_IMAGE}" == "<TEST_IMAGE>" || -z "${TEST_IMAGE}" ]]; then
  echo "错误：请将 TEST_IMAGE 替换为集群能够拉取的测试镜像" >&2
  exit 1
fi

cleanup() {
  status=$?
  trap - EXIT INT TERM
  if [[ "${NS_CREATED}" == true ]]; then
    kubectl delete pod "${TEST_POD}" --namespace "${VERIFY_NS}" \
      --ignore-not-found --wait=true >/dev/null 2>&1 || true
    kubectl delete namespace "${VERIFY_NS}" \
      --ignore-not-found --wait=true >/dev/null 2>&1 || true
  fi
  exit "${status}"
}

kubectl create namespace "${VERIFY_NS}"
NS_CREATED=true
trap cleanup EXIT INT TERM

kubectl run "${TEST_POD}" \
  --namespace "${VERIFY_NS}" \
  --image="${TEST_IMAGE}" \
  --restart=Never \
  --overrides="{\"spec\":{\"nodeName\":\"${NODE_NAME}\"}}"
kubectl wait "pod/${TEST_POD}" \
  --namespace "${VERIFY_NS}" \
  --for=condition=Ready \
  --timeout=120s
kubectl delete pod "${TEST_POD}" \
  --namespace "${VERIFY_NS}" \
  --wait=true
kubectl delete namespace "${VERIFY_NS}" --wait=true
NS_CREATED=false
trap - EXIT INT TERM
```

成功信号：测试 Pod 能在指定节点创建、进入 `Ready`，并在删除后正常消失；本次创建的临时 namespace 也正常删除。若测试失败，清理函数会在保留原始退出状态的同时尝试删除本次创建的 Pod 和 namespace。清理失败时，记录命令输出并使用已输出的唯一 namespace 名称检查残留资源；不要删除其他 namespace，也不要把镜像拉取失败、调度失败等其他原因直接归因于 netns。

### 5.5 检查 kubelet 和运行时日志

在每个目标节点上，从安装或升级开始时间检查日志：

```bash
journalctl -u kubelet --since '<INSTALL_TIME>' | grep -E 'device or resource busy|unlinkat|remove netns|KillPodSandbox|PLEG'
journalctl -u containerd --since '<INSTALL_TIME>' | grep -E 'device or resource busy|unlinkat|remove netns|KillPodSandbox'
```

Docker 节点将第二条命令中的 unit 名称替换为该节点实际使用的 Docker systemd unit。预期结果是测试期间没有新增匹配日志。历史日志仍可能被查到，应结合时间戳区分升级前残留和升级后新增问题。

### 5.6 验证 node-exporter HTTP 和文件系统指标

先选择一个 node-exporter Pod 所在节点并取得其 InternalIP：

```bash
NODE='<节点名称>'
NODE_IP=$(kubectl get node "${NODE}" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
```

从能够访问该节点 `9100` 端口的主机执行：

```bash
curl -sS -o /dev/null -w 'metrics=%{http_code}\n' "http://${NODE_IP}:9100/metrics"
curl -sS -o /dev/null -w 'pprof=%{http_code}\n' "http://${NODE_IP}:9100/debug/pprof/"
curl -fsS "http://${NODE_IP}:9100/metrics" | grep -E '^node_filesystem_(size|avail)_bytes'
curl -fsS "http://${NODE_IP}:9100/metrics" | grep -E '^node_filesystem_device_error'
```

预期结果：

- `/metrics` 返回 HTTP `200`。
- `/debug/pprof/` 返回 HTTP `403`。
- 输出中存在 `node_filesystem_size_bytes` 和 `node_filesystem_avail_bytes`。
- `node_filesystem_device_error` 应结合标签逐项检查；值为 `1` 表示该文件系统采集失败，需要继续检查挂载和权限。

如果执行端无法直接访问节点 IP，可使用受控的端口转发或在集群内的诊断环境执行等效 HTTP 检查，不要为了验证临时放宽生产网络访问策略。

### 5.7 检查历史残留

Chart 更新完成后仍需单独检查历史对象：

```bash
kubectl get pods --all-namespaces | grep Terminating
sudo findmnt -R /run/netns
sudo ls -l /var/run/netns
```

这些命令只用于检查。发现残留时，应先确认 namespace 的引用者、对应 Pod、CNI 和运行时状态；不要直接执行批量删除、强制卸载或手工移除 netns 文件。

## 6. 版本与兼容性提醒

- 当前 Chart 默认运行时是 Docker；containerd 必须显式设置 `cadvisor-exporter.container_runtime=containerd`。
- 当前 Chart 未通过 `kubeVersion` 声明 Kubernetes 版本范围。本文给出的运行时查询方式兼容 Kubernetes 1.23，但不代表所有 Kubernetes、containerd、Docker 或 CNI 版本均已验证。
- 升级前应在与生产环境相同的 Kubernetes、运行时和 CNI 组合中完成渲染检查、Pod 生命周期测试、指标检查和日志观察。
- containerd 路径、systemd unit 名称、`crictl` 能力和节点网络可达性可能因发行版及安装方式不同而变化，应以目标节点实际结果为准。
