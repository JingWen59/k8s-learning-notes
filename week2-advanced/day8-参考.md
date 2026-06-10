# Day 8：监控体系（精简版）

> 日期：2026-05-18
> 主题：Prometheus 架构、PromQL、Grafana、告警

---

## ✅ 今日学习清单

```
上午（2h）：
  ☐ 1. 复习 Week 1（30 分钟，自测面试题）
  ☐ 2. Prometheus 架构 + Pull 模型（理论）
  ☐ 3. PromQL 语法基础（理论）

下午（4h）：
  ☐ 4. Helm 安装 kube-prometheus-stack（实验 1）
  ☐ 5. 访问 Grafana + 导入 Dashboard（实验 2）
  ☐ 6. PromQL 实战（实验 3）
  ☐ 7. 配置告警（实验 4）

晚上（1h）：
  ☐ 8. 面试题自测
  ☐ 9. push 到 GitHub
```

---

## 📚 模块 1：Prometheus 架构（45 分钟）

### 理论（30 分钟）

**Prometheus 是什么？**

```
开源的监控告警系统
存储时序数据（TSDB）
通过 PromQL 查询
是 CNCF 第二个毕业项目（K8s 是第一个）
```

**核心架构：**

```
┌─────────────────────────────────────────────────┐
│  Prometheus Server                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Scraper  │→ │  TSDB    │← │  PromQL      │   │
│  │ (拉取)   │  │ (时序库)  │  │  Engine      │   │
│  └────┬─────┘  └──────────┘  └──────────────┘   │
└───────┼─────────────────────────────────────────┘
        │ HTTP Pull
        ↓
   /metrics 端点
        ↑
   ┌────┴─────┬─────────┬──────────┐
   │          │         │          │
node-export kube-state app1     ingress
 (节点)    (K8s资源)   (业务)     (网关)
```

**5 大组件：**

```
1. Prometheus Server   → 拉取 + 存储 + 查询
2. Exporters           → 暴露指标的小程序
   - node-exporter：节点指标（CPU/内存/磁盘）
   - kube-state-metrics：K8s 资源状态
   - cAdvisor：容器指标（kubelet 内置）
3. Alertmanager        → 告警管理（去重、分组、通知）
4. Pushgateway         → 短期任务推送（如 CronJob 指标）
5. Grafana             → 可视化（非 Prometheus 官方，但常配套）
```

### Pull vs Push 模型

```
Pull 模型（Prometheus 默认）：
  Prometheus 主动去拉取 /metrics
  优点：
    - 集中控制采集频率
    - 容易发现 target 是否在线
    - 适合长期运行的服务
  缺点：
    - 短期任务不好采集（任务结束就拉不到了）

Push 模型：
  应用主动推送指标到 Pushgateway
  适用：短期任务（CronJob、批处理）
```

**为什么 Prometheus 选 Pull？**

```
1. 控制权在 Prometheus，配置统一
2. 服务发现机制可自动发现新 target
3. 网络中间故障容易发现（target down）
```

---

## 📚 模块 2：PromQL 基础（45 分钟）

### 理论（30 分钟）

**4 种数据类型：**

```
Instant Vector（瞬时向量）：某个时刻的一组值
Range Vector（区间向量）：一段时间内的一组值
Scalar（标量）：单个数字
String（字符串）：少用
```

**基本查询：**

```promql
# 瞬时值：所有 CPU 时间累计值
node_cpu_seconds_total

# 加标签过滤
node_cpu_seconds_total{mode="idle"}
node_cpu_seconds_total{instance="k8s-master1"}

# 模糊匹配
node_cpu_seconds_total{instance=~"k8s-.*"}

# 区间向量：过去 5 分钟的数据
node_cpu_seconds_total[5m]
```

**常用函数：**

```promql
rate()      → 每秒平均增长率（counter 用）
irate()     → 瞬时增长率
increase()  → 区间内增长总量
sum()       → 求和
avg()       → 平均
max()/min() → 最大/最小
by (label)  → 按标签分组
without (label) → 排除标签分组
```

**经典查询：**

```promql
# CPU 使用率（%）
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100

# 内存使用率（%）
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Pod CPU 使用率
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Pod 内存使用
container_memory_working_set_bytes{namespace="default"}

# K8s 节点状态
kube_node_status_condition{condition="Ready"}

# Pod 重启次数
kube_pod_container_status_restarts_total
```

---

## ☕ 午休

---

## 📚 模块 3：部署 Prometheus + Grafana（90 分钟）

### 🔨 实验 1：用 Helm 安装 kube-prometheus-stack

```bash
# 1.1 安装 Helm（如果还没装）
curl -k https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 或下载二进制
wget https://mirrors.huaweicloud.com/helm/v3.14.0/helm-v3.14.0-linux-amd64.tar.gz
tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/

helm version

# 1.2 添加仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 如果访问慢，用国内源
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 1.3 创建命名空间
kubectl create namespace monitoring

# 1.4 准备 values.yaml（简化配置，避免资源不足）
cat <<EOF > /root/prometheus-values.yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 400Mi
        cpu: 100m
      limits:
        memory: 1Gi
        cpu: 500m
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

grafana:
  adminPassword: admin123
  service:
    type: NodePort
    nodePort: 30030
  resources:
    requests:
      memory: 200Mi
      cpu: 50m

alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        memory: 100Mi
        cpu: 50m
EOF

# 1.5 安装
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f /root/prometheus-values.yaml

# 等待启动（5-10 分钟，需要拉很多镜像）
kubectl get pods -n monitoring -w
```

**⚠️ 资源不足的话**：

集群每台 Master 只有 2G 内存，可能跑不起完整的 kube-prometheus-stack。

**简化方案**：只装 Prometheus + Grafana（不装 Alertmanager 和 Operator）

```bash
# 单独装 Prometheus
helm install prometheus prometheus-community/prometheus \
  -n monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false \
  --set pushgateway.enabled=false

# 单独装 Grafana
helm install grafana grafana/grafana \
  -n monitoring \
  --set service.type=NodePort \
  --set service.nodePort=30030 \
  --set adminPassword=admin123 \
  --set persistence.enabled=false
```

**✍️ 记录：**
- 装好后 Pod 数：__________
- Grafana NodePort：__________

---

### 🔨 实验 2：访问 Grafana

```bash
# 2.1 查看 Grafana Service
kubectl get svc -n monitoring | grep grafana

# 2.2 拿到密码（如果用的是 stack）
kubectl get secret -n monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
# 默认账号：admin

# 2.3 浏览器访问
# http://192.168.105.128:30030
# 账号 admin，密码上面查到的（或者 admin123）
```

**导入 Dashboard：**

```
登录 Grafana → 左侧 + 号 → Import

输入 Dashboard ID：
  - 7249：Kubernetes Cluster 监控
  - 13770：Kubernetes Views（推荐）
  - 1860：Node Exporter Full（节点监控）

数据源选 Prometheus → Import
```

**✍️ 记录：**
✍️ 记录：

- 你导入了哪些 Dashboard：
  ① 1860 - Node Exporter Full（节点CPU/内存/磁盘/网络监控，正常显示）
  ② 315 - Kubernetes cluster monitoring（K8s集群概览）
  ③ 13332 - kube-state-metrics（Pod/Deployment状态，因cluster变量不匹配显示N/A）

- 看到的关键指标：
  ① CPU使用率：15.8%（k8s-master1）
  ② 系统负载（5分钟平均）：正常
  ③ 内存使用率：正常采集
  ④ 磁盘使用率：正常采集
  ⑤ 4个节点的Node Exporter均在线采集数据
  ⑥ Prometheus数据源连接正常


---

## 📚 模块 4：PromQL 实战（45 分钟）

### 🔨 实验 3：在 Prometheus UI 查询

```bash
# 访问 Prometheus UI
kubectl get svc -n monitoring | grep prometheus
# 找到 prometheus-server，类型改成 NodePort

kubectl edit svc -n monitoring prometheus-server
# 把 type: ClusterIP 改成 type: NodePort
# 加上 nodePort: 30090

# 浏览器访问
# http://192.168.105.128:30090
```

**在 Prometheus 的 Graph 页面练习：**

```promql
# 1. 看所有节点 CPU 使用率
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100

{instance="192.168.105.128:9100"}	16.488882962766155
{instance="192.168.105.131:9100"}	6.599359147972621
{instance="192.168.105.130:9100"}	15.145795311547744
{instance="192.168.105.129:9100"}	15.244809439882019


# 2. 看所有节点内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.128:9100", job="kubernetes-service-endpoints", namespace="monitoring", node="k8s-master1", service="prometheus-prometheus-node-exporter"}	61.72134399840874
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.131:9100", job="kubernetes-service-endpoints", namespace="monitoring", node="k8s-worker1", service="prometheus-prometheus-node-exporter"}	71.43378626767216
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.130:9100", job="kubernetes-service-endpoints", namespace="monitoring", node="k8s-master3", service="prometheus-prometheus-node-exporter"}	50.932536044281875
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.129:9100", job="kubernetes-service-endpoints", namespace="monitoring", node="k8s-master2", service="prometheus-prometheus-node-exporter"}

# 3. 看所有 Pod 数量
count(kube_pod_info)

38

# 4. 看每个 namespace 的 Pod 数
count(kube_pod_info) by (namespace)

{namespace="kube-system"}	23
{namespace="monitoring"}	7
{namespace="default"}	7
{namespace="ingress-nginx"}	1


# 5. 看 Pod 重启次数（找有问题的）
kube_pod_container_status_restarts_total > 0

kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-controller-manager", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-controller-manager-k8s-master1", service="prometheus-kube-state-metrics", uid="e502dfd4-43e1-429a-9f4b-133c31e2d0fa"}	5
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-scheduler", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-scheduler-k8s-master3", service="prometheus-kube-state-metrics", uid="e9dd4e0d-e459-4216-b47e-2898deb98850"}	24
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="etcd", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="etcd-k8s-master3", service="prometheus-kube-state-metrics", uid="d10b860c-9864-4c89-b0e4-34f149494e4d"}	4
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="etcd", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="etcd-k8s-master2", service="prometheus-kube-state-metrics", uid="cb966747-b840-4bbf-bb54-63cc96bcfa9d"}	4
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="nginx", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="web-74958745-49jfw", service="prometheus-kube-state-metrics", uid="41551427-d907-4098-81fa-0e55e6c6dc32"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-proxy", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-proxy-mlnt8", service="prometheus-kube-state-metrics", uid="3d1c5a32-a948-4c3c-9ef0-47d75b785ef1"}	3
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="coredns", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="coredns-cb4864fb5-c7r5w", service="prometheus-kube-state-metrics", uid="97af471c-40b1-4665-805f-d703bb223316"}	3
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="calico-node", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="calico-node-4x27x", service="prometheus-kube-state-metrics", uid="00f969a4-51cd-477a-8456-19c76f4fa412"}	6
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="nginx", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="web-74958745-m9lvh", service="prometheus-kube-state-metrics", uid="79bd63fb-284b-4366-a1ea-a2ab004e99f8"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-proxy", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-proxy-gnszx", service="prometheus-kube-state-metrics", uid="11cfa745-fec9-408b-8459-71f3a9f34719"}	4
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="nginx", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="web-74958745-8n58x", service="prometheus-kube-state-metrics", uid="90fd37ec-bea6-4228-b650-1f0b844dc5be"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="calico-node", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="calico-node-tkkf6", service="prometheus-kube-state-metrics", uid="e19a2f8f-365f-435f-bddd-8b3ba379ec63"}	3
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="httpd", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="httpd-app-c689bcddc-bhqkf", service="prometheus-kube-state-metrics", uid="f01cb246-c6bc-47c6-8cc4-56e5703bf900"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="httpd", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="httpd-app-c689bcddc-jflhl", service="prometheus-kube-state-metrics", uid="7367a7df-d265-4997-93ae-0a6a3c4c0030"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-proxy", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-proxy-2wcwp", service="prometheus-kube-state-metrics", uid="cc0dbaba-c8b3-4fcb-b82f-a4188e4edc32"}	5
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-scheduler", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-scheduler-k8s-master2", service="prometheus-kube-state-metrics", uid="434e57bd-7dec-4870-84f6-aa290f60572a"}	25
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="calico-node", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="calico-node-bkxz6", service="prometheus-kube-state-metrics", uid="be7fb414-6aaf-43cc-809e-8f3d1bcfe65c"}	23
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="node-exporter", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="monitoring", node="k8s-master3", pod="prometheus-prometheus-node-exporter-stltz", service="prometheus-kube-state-metrics", uid="d629a8be-5b98-4cde-a0ac-da81e1ed3292"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-proxy", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-proxy-lx28b", service="prometheus-kube-state-metrics", uid="22d979d5-d772-49f2-8b45-06685604dbb6"}	3
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-scheduler", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-scheduler-k8s-master1", service="prometheus-kube-state-metrics", uid="c7129c2d-e977-48f7-9fd1-4275255b3415"}	25
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-controller-manager", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-controller-manager-k8s-master3", service="prometheus-kube-state-metrics", uid="8c465c53-81de-4abe-ad62-b41cab10db49"}	6
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-apiserver", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-apiserver-k8s-master3", service="prometheus-kube-state-metrics", uid="18a8e96c-77b0-40b3-a719-46858b51b18a"}	7
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-controller-manager", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-controller-manager-k8s-master2", service="prometheus-kube-state-metrics", uid="d6b132e4-f999-4321-a307-56fe4cef1f00"}	8
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="etcd", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="etcd-k8s-master1", service="prometheus-kube-state-metrics", uid="984c5b99-3a24-4d41-bb00-1fbabe5a1bad"}	5
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="calico-kube-controllers", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="calico-kube-controllers-564985c589-rq86t", service="prometheus-kube-state-metrics", uid="a2be82fc-5305-4fc5-874f-835a4e135260"}	18
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="nginx", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="nginx-app-7fdcbb8c5-8q58c", service="prometheus-kube-state-metrics", uid="0cd0529d-844d-4cc8-b6e2-6bfcd81b74d2"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-apiserver", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-apiserver-k8s-master2", service="prometheus-kube-state-metrics", uid="497bba56-1bc7-4d3d-9f76-75f7430d88b3"}	11
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="coredns", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="coredns-cb4864fb5-54fbs", service="prometheus-kube-state-metrics", uid="c202a2f6-b942-4329-898e-f7b22f22e69f"}	3
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="nginx", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="default", node="k8s-master3", pod="nginx-app-7fdcbb8c5-j9586", service="prometheus-kube-state-metrics", uid="0d5aad0e-df76-4cb4-b7d2-450a1573b6ab"}	1
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="kube-apiserver", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="kube-apiserver-k8s-master1", service="prometheus-kube-state-metrics", uid="e6e4eb4b-99ce-4bd1-aaa9-ee8ac60341bc"}	28
kube_pod_container_status_restarts_total{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="kube-state-metrics", app_kubernetes_io_part_of="kube-state-metrics", app_kubernetes_io_version="2.18.0", container="calico-node", helm_sh_chart="kube-state-metrics-7.3.0", instance="10.244.135.196:8080", job="kubernetes-service-endpoints", namespace="kube-system", node="k8s-master3", pod="calico-node-qdzqw", service="prometheus-kube-state-metrics", uid="69aa7658-9464-4f96-8cb0-b21d4392cf6b"}

# 6. 看节点磁盘使用率
(node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_avail_bytes{mountpoint="/"}) 
  / node_filesystem_size_bytes{mountpoint="/"} * 100

{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", device="/dev/sda3", fstype="xfs", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.128:9100", job="kubernetes-service-endpoints", mountpoint="/", namespace="monitoring", node="k8s-master1", service="prometheus-prometheus-node-exporter"}	46.339515444787814
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", device="/dev/sda3", fstype="xfs", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.131:9100", job="kubernetes-service-endpoints", mountpoint="/", namespace="monitoring", node="k8s-worker1", service="prometheus-prometheus-node-exporter"}	86.34670668699299
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", device="/dev/sda3", fstype="xfs", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.130:9100", job="kubernetes-service-endpoints", mountpoint="/", namespace="monitoring", node="k8s-master3", service="prometheus-prometheus-node-exporter"}	48.63073229954197
{app_kubernetes_io_component="metrics", app_kubernetes_io_instance="prometheus", app_kubernetes_io_managed_by="Helm", app_kubernetes_io_name="prometheus-node-exporter", app_kubernetes_io_part_of="prometheus-node-exporter", app_kubernetes_io_version="1.11.1", device="/dev/sda3", fstype="xfs", helm_sh_chart="prometheus-node-exporter-4.55.0", instance="192.168.105.129:9100", job="kubernetes-service-endpoints", mountpoint="/", namespace="monitoring", node="k8s-master2", service="prometheus-prometheus-node-exporter"}
```

**✍️ 记录：**
- 你的集群 CPU 使用率：__________
- 内存使用率：__________
- 哪个 Pod 重启过：__________

集群 CPU 使用率：
节点	CPU 使用率
k8s-master1 (192.168.105.128)	16.49%
k8s-master2 (192.168.105.129)	15.24%
k8s-master3 (192.168.105.130)	15.15%
k8s-worker1 (192.168.105.131)	6.60%
集群平均	约 13.4%
内存使用率：
节点	内存使用率
k8s-master1 (192.168.105.128)	61.72%
k8s-master2 (192.168.105.129)	有数据（未显示具体值）
k8s-master3 (192.168.105.130)	50.93%
k8s-worker1 (192.168.105.131)	71.43% ⚠️ 最高
哪个 Pod 重启过：
Pod	容器	重启次数	说明
kube-apiserver-k8s-master1	kube-apiserver	28	⚠️ 最多，需关注
kube-scheduler-k8s-master1	kube-scheduler	25	控制面组件频繁重启
kube-scheduler-k8s-master2	kube-scheduler	25	同上
kube-scheduler-k8s-master3	kube-scheduler	24	同上
calico-node-bkxz6 (worker1)	calico-node	23	⚠️ worker1 网络组件不稳
calico-kube-controllers	calico-kube-controllers	18	网络控制器重启多
kube-apiserver-k8s-master2	kube-apiserver	11	
kube-controller-manager-k8s-master2	kube-controller-manager	8	
kube-apiserver-k8s-master3	kube-apiserver	7	
kube-controller-manager-k8s-master3	kube-controller-manager	6	
calico-node-4x27x (master3)	calico-node	6	
kube-controller-manager-k8s-master1	kube-controller-manager	5	
etcd-k8s-master1	etcd	5	
kube-proxy (多个)	kube-proxy	3~5	正常范围
etcd-k8s-master2	etcd	4	
etcd-k8s-master3	etcd	4	
calico-node-tkkf6 (master1)	calico-node	3	
coredns (×2)	coredns	3	
web-xxx (×3)	nginx	1	业务 Pod，正常
nginx-app-xxx (×2)	nginx	1	业务 Pod，正常
httpd-app-xxx (×2)	httpd	1	业务 Pod，正常
node-exporter-stltz (master2)	node-exporter	1	正常（刚重启过节点）
📊 额外记录 - 磁盘使用率：
节点	根分区使用率	状态
k8s-master1	46.34%	✅ 正常
k8s-master2	有数据（未显示具体值）	
k8s-master3	48.63%	✅ 正常
k8s-worker1	86.35%	🔴 危险！接近驱逐阈值
⚠️ 需要关注的风险

worker1 磁盘 86%：kubelet 默认 85% 触发 DiskPressure，随时可能再次驱逐 Pod

worker1 内存 71%：也较高

kube-apiserver/scheduler 重启 25-28 次：集群运行 7 天，重启次数偏多，可能跟之前 DiskPressure 和节点重启有关

calico-node (worker1) 重启 23 次：跟 worker1 反复 DiskPressure 驱逐相关

---

## 📚 模块 5：配置告警（30 分钟）

### 🔨 实验 4：写一个告警规则

```bash
# 5.1 创建一个 PrometheusRule（用 Operator 部署的话）
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-alerts
  namespace: monitoring
spec:
  groups:
  - name: my-alerts
    rules:
    - alert: HighCPUUsage
      expr: 100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100 > 80
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "节点 {{ \$labels.instance }} CPU 使用率超过 80%"
        description: "当前 CPU 使用率：{{ \$value }}%"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ \$labels.pod }} 反复重启"
EOF

# 5.2 查看告警
kubectl get prometheusrule -n monitoring

# 5.3 在 Prometheus UI 看告警状态
# 浏览器：http://192.168.105.128:30090/alerts
```

**告警的 3 个状态：**

```
Inactive：正常，没触发
Pending：触发条件满足，但还在 for 时间内
Firing：持续超过 for 时间，正式告警
```

**记录**
- HighCPUUsage：状态 inactive（当前所有节点 CPU < 80%，规则正常待触发）
- PodCrashLooping：状态 inactive（当前没有 Pod 在 15 分钟内反复重启）
- 规则健康状态：ok
- 评估间隔：60 秒
- 规则文件位置：/etc/config/alerting_rules.yml


🎉 整个监控系统现在的能力
✅ 数据采集：Prometheus + Node Exporter + kube-state-metrics
✅ 可视化：  Grafana Dashboard（1860/315）
✅ 告警规则：HighCPUUsage / PodCrashLooping（在 Prometheus 中生效）
❌ 告警通知：未安装 Alertmanager（不会发邮件/钉钉通知）


告警规则
HighCPUUsage

一句话：某个节点 CPU 超过 80% 持续 2 分钟就报警

触发条件：
  节点 CPU > 80%
  持续时间：2 分钟

当前状态：inactive（所有节点 CPU 都在 6%~16%，远没到 80%）


类比：体温超过 38.5°C 持续 2 分钟就报警

PodCrashLooping

一句话：某个 Pod 在 15 分钟内反复重启就报警

触发条件：
  15 分钟内有 Pod 重启
  持续时间：5 分钟

当前状态：inactive（当前没有 Pod 在反复重启）


类比：病人反复晕倒，5 分钟内发生多次就发紧急通知


整体协作关系

Node Exporter（采集机器指标）──┐
                               ├──→ Prometheus（存储+计算）──→ 告警规则判断
kube-state-metrics（采集K8s指标）┘          │                    ├── HighCPUUsage
                                           │                    └── PodCrashLooping
                                           ↓
                                      Grafana（图表展示）


---

## ❓ 模块 6：面试题

### Q1: Prometheus 怎么采集 K8s 指标？

```
Pull 模型：
  Prometheus 主动 HTTP 请求 target 的 /metrics 端点

K8s 集群指标来源：
  - node-exporter：节点 CPU/内存/磁盘（每节点一个 Pod，DaemonSet）
  - kube-state-metrics：K8s 资源状态（Deployment 副本数、Pod 状态等）
  - cAdvisor：容器指标（kubelet 内置）
  - kubelet：节点自身指标

  Node Exporter 告诉你：“机器哪里不舒服”（磁盘I/O高、网络丢包）。

kube-state-metrics 告诉你：“调度器在干什么”（Pod待调度、副本数不足）。

cAdvisor 告诉你：“哪个容器在捣乱”（这个Pod内存超标）。

```

### Q2: Prometheus 的服务发现机制？

```
Prometheus 通过配置文件自动发现 target：

K8s 服务发现：
  - kubernetes_sd_configs
  - 支持发现：node / pod / service / endpoints / ingress
  - 通过 label selector 筛选
  - 新增 Pod 自动加入采集，删除自动移除

其他方式：
  - file_sd_configs（文件）
  - dns_sd_configs（DNS）
  - consul_sd_configs（Consul）
```

### Q3: Pull 模型 vs Push 模型？

```
Pull（Prometheus 默认）：
  - Prometheus 主动拉取
  - 优点：集中控制、容易发现 target 故障
  - 缺点：短期任务采集不到

Push（用 Pushgateway）：
  - 应用主动推送指标
  - 适用：短期任务、批处理
  - 缺点：单点故障、数据可能丢失

K8s 监控选 Pull，符合声明式理念
```

### Q4: PromQL 查 Pod CPU 使用率？

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="default", pod=~"nginx.*"}[5m])) by (pod)

解释：
  container_cpu_usage_seconds_total - cAdvisor 暴露的 CPU 累计秒数
  rate()[5m] - 每秒增长率，即 CPU 使用核数
  sum() by (pod) - 按 Pod 聚合
```

### Q5: Alertmanager 怎么配置？

```
Alertmanager 负责：
  - 告警去重（同一告警不重复发）
  - 告警分组（相关告警合并）
  - 告警路由（不同告警发不同人）
  - 告警抑制（高级别告警抑制低级别）

通知方式：
  - Email
  - Webhook（钉钉/企微/Slack）
  - PagerDuty

配置流程：
  1. Prometheus 触发告警 → 发给 Alertmanager
  2. Alertmanager 根据 route 路由
  3. 选择 receiver 发送
```

### Q6: Counter / Gauge / Histogram / Summary？

```
Counter（计数器）：只增不减（HTTP 请求总数、错误总数）
  → 配合 rate() 用

Gauge（仪表盘）：可增可减（当前内存、当前连接数）
  → 直接看就行

Histogram（直方图）：分桶统计（请求延迟分布）
  → 配合 histogram_quantile() 算分位数

Summary（摘要）：客户端预聚合的分位数
  → 直接拿 0.5 / 0.9 / 0.99 分位
```

### Q7: rate() 和 irate() 区别？

```
rate()：区间内的平均增长率（平滑）
  → 适合做监控图表

irate()：最后两个数据点的瞬时增长率
  → 适合故障排查（反应敏感）

例：
  rate(http_requests_total[5m])   - 5 分钟平均 QPS
  irate(http_requests_total[5m])  - 最近瞬时 QPS
```

### Q8: 监控 K8s 集群的核心指标？

```
节点层面：
  - CPU 使用率
  - 内存使用率
  - 磁盘使用率
  - 网络流量

集群资源层面：
  - 节点是否 Ready
  - Pod 是否 Running
  - Pod 重启次数
  - Deployment 副本数是否符合预期

容器层面：
  - 容器 CPU/内存
  - 容器是否被 OOMKilled

业务层面：
  - QPS / 错误率 / 响应时间（RED 指标）
```

### Q9: 怎么监控自己写的应用？

```
方案 1：暴露 /metrics 端点（推荐）
  - 用 Prometheus 客户端库（go/python/java）
  - 暴露 HTTP 端口 + /metrics
  - 配置 ServiceMonitor 让 Prometheus 自动发现

方案 2：用 Exporter
  - 如果应用本身不支持 /metrics
  - 部署对应的 exporter（mysql-exporter / redis-exporter）
```

### Q10: Prometheus 数据存储？

```
TSDB（时序数据库）：
  - 本地磁盘存储
  - 默认保留 15 天
  - 高压缩比（约 1-2 字节/样本）

长期存储方案：
  - Thanos：横向扩展、长期存储
  - VictoriaMetrics：高性能替代品
  - Cortex：多租户
```

---

## 🎯 速记版（考前 1 分钟）

```
架构：Prometheus(拉取+存储+查询) + Exporters + Alertmanager + Grafana
采集：Pull 模型，HTTP GET /metrics
K8s 监控：node-exporter(节点) + kube-state-metrics(资源) + cAdvisor(容器)
4 种指标：Counter(只增) Gauge(可变) Histogram(分桶) Summary(分位)
核心函数：rate() irate() sum() by() 
服务发现：kubernetes_sd_configs 自动发现 K8s 资源
告警状态：Inactive → Pending → Firing
长期存储：Thanos / VictoriaMetrics
```

---

## 📊 自测清单

```
☐ 1. Prometheus 5 大组件？    
☐ 2. Pull 和 Push 模型的区别和使用场景？
☐ 3. node-exporter / kube-state-metrics / cAdvisor 各自采集什么？
☐ 4. PromQL 4 种数据类型？
☐ *5. rate() 和 irate() 区别？
☐ *6. Counter / Gauge / Histogram / Summary 各自适合什么？
☐ 7. 写一个查询：所有节点 CPU 使用率
☐ 8. Alertmanager 的 4 个核心功能？
☐ 9. 怎么监控自己写的应用？
☐ 10. K8s 集群必须监控的 5 个核心指标？
```
