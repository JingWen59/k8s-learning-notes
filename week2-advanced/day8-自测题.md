### Q1: Prometheus 怎么采集 K8s 指标？

Pull模型：
prometheus 拉取 target的 /metrics 端点

k8s集群指标：
-node-exporter 节点 CPU/内存/磁盘 资源
-kube-state-metrics   K8s 资源状态（Deployment 副本数、Pod 状态等）
-cAdvisor  容器指标（kubelet 内置）
-kubelet  节点自身指标

### Q2: Prometheus 的服务发现机制？

服务发现 = Prometheus 主动问 K8s “现在有哪些 Pod/节点？”然后自动开始采集，你不用手动维护任何 IP 地址。K8s服务发现模式下，Target信息不会写入任何本地文件，而是直接缓存在Prometheus的内存中。文件服务发现模式下，Prometheus 读取文件内容到内存。 


### Q3: Pull 模型 vs Push 模型？

Pull（Prometheus 默认）：
  - Prometheus 主动拉取
  - 优点：集中控制、容易发现 target 故障
  - 缺点：短期任务采集不到

Push（用 Pushgateway）：
  - 应用主动推送指标
  - 适用：短期任务、批处理
  - 缺点：单点故障、数据可能丢失

K8s 监控选 Pull，符合声明式理念


### Q4: PromQL 查 Pod CPU 使用率？

ontainer_cpu_usage_seconds_total - cAdvisor 暴露的 CPU 累计秒数

### Q5: Alertmanager 怎么配置？

Alertmanager 负责：
  - 告警去重（同一告警不重复发）
  - 告警分组（相关告警合并）
  - 告警路由（不同告警发不同人）
  - 告警抑制（高级别告警抑制低级别）

配置流程：
  1. Prometheus 触发告警 → 发给 Alertmanager
  2. Alertmanager 根据 route 路由
  3. 选择 receiver 发送

### Q8: 监控 K8s 集群的核心指标？




### Q9: 怎么监控自己写的应用？




### Q10: Prometheus 数据存储？

