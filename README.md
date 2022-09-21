# WeOps Lab Helm Repositories

![](https://wedoc.canway.net/imgs/img/嘉为蓝鲸.jpg)

WeOps Lab Helm Chart仓库

* weops-kuberbetes-collect 用于kubernetes的资源与监控数据采集。集成[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics),[node_exporter](https://github.com/prometheus/node_exporter)和[cadvisor](https://github.com/google/cadvisor)，通过prometheus-agent采集数据，remotewrite到外部服务。


## 使用示例

```shell
git clone https://github.com/WeOps-Lab/WeOps-Charts.git
cd WeOps-Charts/weops/weops-kubernetes-collect
helm install weops-collecter . -n weops --create-namespace
```

参数说明

参数|描述|必填|默认值
---|----|----|----
image.repository|采集数据使用的prometheus-agent镜像仓库|是|bitnami/prometheus
image.tag|采集数据使用的prometheus-agent镜像tag|否|2.38.0
image.pullPolicy|镜像拉取策略|否|IfNotPresent
resources.limits.cpu|prometheus-agent的cpu资源限制|否|512m
resources.limits.memory|prometheus-agent的内存资源限制|否|512Mi
resources.requests.cpu|prometheus-agent的cpu资源要求|否|10m
resources.requests.memory|prometheus-agent的cpu资源要求|否|32Mi
remoteWrtieUrl|remoteWrtie的目标地址|是|http://admin:admin@10.10.10.10:9090/api/v1/write
kubeStateMetricsUrl|使用已有kube-state-metrics时填写，仅在enableKubeStateMetrics为false时生效|否|prometheus-kube-state-metrics:8080
kubeClusterLabel|k8s集群标签，用于区分多集群接入|是|a-kubernetes-cluster
scrape_interval|数据刮削时间间隔|否|60s
evaluation_interval|数据计算时间间隔|否|60s
enableKubeStateMetrics|启用内置 kube-state-metrics|是|true
kube-state-metrics.image.repository|kube-state-metrics镜像仓库|是|registry.k8s.io/kube-state-metrics/kube-state-metrics
kube-state-metrics.image.tag|kube-state-metrics镜像tag|否|v2.5.0
kube-state-metrics.image.pullPolicy|kube-state-metrics镜像拉取策略|否|IfNotPresent
kube-state-metrics.image.pullPolicy.imagePullSecrets|kube-state-metrics镜像拉取凭据|否|[]
kube-state-metrics.resources.limit.cpu|kube-state-metrics的cpu资源限制|否|100m
kube-state-metrics.resources.limit.memory|kube-state-metrics的内存资源限制|否|256Mi
kube-state-metrics.resources.cpu|kube-state-metrics的cpu资源要求|否|10m
kube-state-metrics.resources.memory|kube-state-metrics的内存资源要求|否|64Mi
