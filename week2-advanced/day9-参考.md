# Day 9：日志收集（精简版）

> 日期：2026-05-19
> 主题：K8s 日志架构、EFK/PLG 方案、Loki + Promtail

---

## ✅ 今日学习清单

```
上午（2h）：
  ☐ 1. 复习 Day 8 监控面试题（15 分钟）
  ☐ 2. K8s 日志架构（3 种方案对比）
  ☐ 3. EFK vs PLG 选型

下午（4h）：
  ☐ 4. 部署 Loki + Promtail（实验 1）
  ☐ 5. Grafana 查询日志（实验 2）
  ☐ 6. 日志排查实战（实验 3）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：K8s 日志架构（45 分钟）

### 理论（30 分钟）

**K8s 日志的 3 个层级：**

```
1. 容器日志（stdout/stderr）
   → kubectl logs 能看到的
   → 存在节点的 /var/log/containers/
   → 容器重建后丢失

2. 节点日志
   → kubelet、containerd 的日志
   → systemd journal

3. 集群级别日志
   → 所有 Pod 的日志集中收集
   → 持久化存储 + 搜索
   → 这就是日志收集系统要做的事
```

**为什么需要集中收集？**

```
问题：
  - kubectl logs 只能看当前 Pod
  - Pod 重建后日志丢失
  - 多副本时不知道看哪个 Pod
  - 无法搜索历史日志
  - 无法跨 Pod 关联分析

解决：
  → 集中收集所有 Pod 日志
  → 持久化存储
  → 统一搜索和分析
```

**3 种日志收集方案：**

```
方案 1：节点级 DaemonSet（最常用）⭐
  → 每个节点跑一个日志 Agent
  → Agent 读取 /var/log/containers/ 下的日志文件
  → 发送到集中存储

  优点：对应用无侵入，部署简单
  缺点：只能收集 stdout/stderr

方案 2：Sidecar 容器
  → 每个 Pod 里加一个日志采集容器
  → 适合需要收集文件日志（不走 stdout）的场景

  优点：灵活，可以收集文件日志
  缺点：每个 Pod 都要加，资源开销大

方案 3：应用直接推送
  → 应用自己把日志发到收集系统
  → 类似 Push 模型

  优点：最灵活
  缺点：侵入应用代码，耦合高
```

**生产环境选择：方案 1（DaemonSet）覆盖 90% 场景。**

---

### EFK vs PLG 对比

```
EFK（重量级）：
  E = Elasticsearch（存储 + 搜索）
  F = Fluentd / Filebeat（采集）
  K = Kibana（可视化）

  优点：全文搜索强大，功能丰富
  缺点：资源消耗大（ES 需要大量内存），运维复杂

PLG（轻量级）⭐：
  P = Promtail（采集）
  L = Loki（存储 + 查询）
  G = Grafana（可视化）

  优点：资源消耗小，和 Prometheus 生态融合
  缺点：不支持全文搜索（只索引标签）
```

**对比表：**

```
             EFK                PLG
─────────────────────────────────────────
存储引擎    Elasticsearch       Loki
采集器      Fluentd/Filebeat    Promtail
可视化      Kibana              Grafana
资源消耗    大（ES 需 8G+）      小（Loki 轻量）
全文搜索    ✅ 支持              ❌ 只查标签
适合规模    大型集群             中小型集群
运维难度    高                   低
```

**面试建议回答**：小规模用 PLG（Loki），大规模用 EFK（ES）。

---

## 📚 模块 2：Loki + Promtail 架构（30 分钟）

### 理论（30 分钟）

**Loki 核心理念：**

```
"Like Prometheus, but for logs"

Prometheus：按标签索引指标
Loki：按标签索引日志（不索引日志内容）

标签示例：
  {namespace="default", pod="nginx-xxx", container="nginx"}

查询时：
  先通过标签筛选 → 再在结果中 grep
  这样存储成本极低
```

**组件职责：**

```
Promtail（采集器，DaemonSet）：
  → 每个节点一个
  → 读取 /var/log/pods/ 目录下的日志
  → 自动给日志打标签（namespace、pod、container）
  → 推送到 Loki

Loki（存储 + 查询）：
  → 接收 Promtail 推送的日志
  → 按标签索引，日志内容压缩存储
  → 提供 LogQL 查询接口

Grafana（可视化）：
  → Explore 页面查询日志
  → 可以和监控指标关联
```

**LogQL 基本语法：**

```logql
# 按标签查
{namespace="default"}
{namespace="default", pod=~"nginx.*"}
{container="nginx"}

# 按内容过滤
{namespace="default"} |= "error"        # 包含 error
{namespace="default"} != "healthcheck"   # 不包含 healthcheck
{namespace="default"} |~ "err|warn"      # 正则匹配

# 按时间
{namespace="default"} |= "error"         # 最近显示的日志里有 error

# 统计
count_over_time({namespace="default"} |= "error" [5m])   # 5 分钟内 error 数
rate({namespace="default"}[1m])                           # 每秒日志产生率
```

---

## ☕ 午休

---

## 📚 模块 3：部署 Loki + Promtail（90 分钟）

### 🔨 实验 1：Helm 安装 Loki Stack

```bash
# 1.1 添加 Grafana 仓库
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 如果 grafana 仓库访问不了，试国内镜像
# helm repo add grafana http://mirror.azure.cn/kubernetes/charts/

# 1.2 搜索确认
helm search repo grafana/loki

# 1.3 安装 Loki（轻量配置）
cat <<EOF > /root/loki-values.yaml
loki:
  persistence:
    enabled: false
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
promtail:
  resources:
    requests:
      memory: 128Mi
      cpu: 50m
    limits:
      memory: 256Mi
EOF

helm install loki grafana/loki-stack \
  -n monitoring \
  -f /root/loki-values.yaml \
  --set grafana.enabled=false \
  --set prometheus.enabled=false

# 1.4 等待启动
kubectl get pods -n monitoring -l app=loki
kubectl get pods -n monitoring -l app=promtail
# loki-0：Running（StatefulSet）
# promtail-xxx：每个节点一个（DaemonSet）
```

**⚠️ 如果 Helm 源不可用**，手动部署方案：

```bash
# 手动部署 Loki（StatefulSet）
cat <<EOF > /root/yaml/logging/loki.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: loki
  namespace: logging
spec:
  serviceName: loki
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
      - name: loki
        image: grafana/loki:2.9.0
        args:
        - -config.file=/etc/loki/loki.yaml
        ports:
        - containerPort: 3100
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        resources:
          requests:
            memory: 256Mi
          limits:
            memory: 512Mi
      volumes:
      - name: config
        configMap:
          name: loki-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: logging
data:
  loki.yaml: |
    auth_enabled: false
    server:
      http_listen_port: 3100
    common:
      path_prefix: /loki
      storage:
        filesystem:
          chunks_directory: /loki/chunks
          rules_directory: /loki/rules
      replication_factor: 1
      ring:
        kvstore:
          store: inmemory
    schema_config:
      configs:
      - from: 2020-10-24
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h
---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: logging
spec:
  selector:
    app: loki
  ports:
  - port: 3100
    targetPort: 3100
EOF

mkdir -p /root/yaml/logging
kubectl apply -f /root/yaml/logging/loki.yaml

# 手动部署 Promtail（DaemonSet）
cat <<EOF > /root/yaml/logging/promtail.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: logging
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: promtail
        image: grafana/promtail:2.9.0
        args:
        - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: pods-log
          mountPath: /var/log/pods
          readOnly: true
        resources:
          requests:
            memory: 128Mi
          limits:
            memory: 256Mi
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: pods-log
        hostPath:
          path: /var/log/pods
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: logging
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    positions:
      filename: /tmp/positions.yaml
    clients:
    - url: http://loki.logging.svc:3100/loki/api/v1/push
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
EOF

kubectl apply -f /root/yaml/logging/promtail.yaml

# 验证
kubectl get pods -n logging
# loki-0：Running
# promtail-xxx：每个节点一个
```

**✍️ 记录：**
- Loki Pod 状态：__________
- Promtail Pod 数量：__________（应该 = 节点数）

项目	结果
Loki Pod 状态	1/1 Running（正常运行，111分钟）
Promtail Pod 数量	4 个（= 节点数 4，✅ 完全匹配）
详细说明
节点	Promtail Pod	状态
k8s-master1	promtail-79wx4	✅ Running
k8s-master2	promtail-8xcnr	✅ Running
k8s-master3	promtail-kk7jq	✅ Running
k8s-worker1	promtail-wnrjq	✅ Running

🎉 Loki + Promtail 日志收集栈运行完全正常！

Loki（日志存储/查询）：1个 Pod，Running ✅

Promtail（日志采集）：4个 Pod = 4个节点，全部 Running ✅

命名空间：logging

---

### 🔨 实验 2：Grafana 查询日志（30 分钟）

```bash
# 2.1 在 Grafana 中添加 Loki 数据源
# 浏览器访问 Grafana：http://192.168.105.128:30030
#
# 操作步骤：
# 1. 左侧菜单 → Configuration → Data Sources → Add data source
# 2. 选择 Loki
# 3. URL 填：http://loki.logging.svc:3100
#    （或 http://loki.monitoring.svc:3100，看你装在哪个 namespace）
# 4. Save & Test

# 2.2 查询日志
# 左侧菜单 → Explore → 数据源选 Loki
#
# 试以下查询：

# 所有 default namespace 的日志
{namespace="default"}

# nginx Pod 的日志
{namespace="default", pod=~"nginx.*"}

# 包含 error 的日志
{namespace="default"} |= "error"

# kube-system 的日志
{namespace="kube-system"}
```

**如果 Grafana 还没装**（NodePort 访问）：

```bash
# 查看 Grafana Service
kubectl get svc -n monitoring | grep grafana

# 如果没有 NodePort，改一下
kubectl edit svc -n monitoring prometheus-grafana
# 改 type: ClusterIP 为 type: NodePort
# 加 nodePort: 30030
```

**✍️ 记录：**
- Grafana 上能查到日志吗：__________
- 查询语句示例：__________

---

### 🔨 实验 3：日志排查实战（30 分钟）

```bash
# 3.1 制造一些日志
# 部署一个会报错的 Pod
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-demo
  template:
    metadata:
      labels:
        app: log-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "while true; do echo INFO normal log; sleep 3; echo ERROR something failed; sleep 7; done"]
EOF

# 等 Pod 起来
sleep 15

# 3.2 用 kubectl logs 看
kubectl logs -l app=log-demo --tail=10

# 3.3 在 Grafana Explore 中查询
# {namespace="default", app="log-demo"} |= "ERROR"

# 3.4 统计错误频率
# count_over_time({namespace="default", app="log-demo"} |= "ERROR" [5m])

# 3.5 对比传统方式 vs Loki
# 传统：kubectl logs -l app=log-demo | grep ERROR
# Loki：{app="log-demo"} |= "ERROR"（可以看历史，有时间线）

# 清理
kubectl delete deployment log-demo
```

**✍️ 记录：**
- kubectl logs vs Loki 的区别：__________
- LogQL 查错误日志的语句：__________

---

## ❓ 模块 4：面试题

### Q1: K8s 日志收集有哪几种方案？

```
方案 1：节点级 DaemonSet（最常用）
  → Agent（Promtail/Filebeat）读取 /var/log/pods/
  → 对应用无侵入

方案 2：Sidecar 容器
  → 每个 Pod 加一个日志容器
  → 收集文件日志（不走 stdout 的场景）

方案 3：应用直接推送
  → 应用内集成 SDK 推送
  → 侵入代码，耦合高

生产选择：DaemonSet 覆盖 90%
```

### Q2: EFK 和 PLG（Loki）区别？

```
EFK：
  - ES 做全文索引（存储大、搜索强）
  - 资源消耗大（ES 需 8G+ 内存）
  - 适合大规模、需要全文搜索

PLG（Loki）：
  - 只索引标签，日志压缩存储
  - 资源消耗小
  - 和 Prometheus/Grafana 生态融合
  - 适合中小规模

选型：
  小团队 / 中小集群 → PLG
  大团队 / 大集群 / 需要全文搜索 → EFK
```

### Q3: Loki 为什么比 ES 省资源？

```
ES：对日志全文建索引 → 索引本身就很大
Loki：只索引标签（namespace/pod/container），日志内容压缩存储

查询方式不同：
  ES：直接搜"error" → 从索引找
  Loki：先按标签找到日志流 → 再 grep 内容

代价：Loki 搜索大量日志时比 ES 慢
收益：存储成本降低 10 倍以上
```

### Q4: Promtail 怎么知道日志属于哪个 Pod？

```
K8s 的日志文件路径包含了元信息：
  /var/log/pods/<namespace>_<pod-name>_<uid>/<container>/0.log

Promtail 通过 kubernetes_sd_configs 自动发现 Pod
并根据路径 + K8s API 的元数据，给日志打标签：
  namespace, pod, container, node 等
```

### Q5: kubectl logs 和集中日志系统的区别？

```
kubectl logs：
  - 只能看当前正在运行的 Pod
  - Pod 重建后日志丢失
  - 多副本时要逐个看
  - 没有搜索功能

集中日志系统（Loki/ES）：
  - 持久化存储（Pod 删了还能看）
  - 统一搜索所有 Pod
  - 支持时间范围查询
  - 支持告警
```

### Q6: 容器日志存在哪里？

```
容器 stdout/stderr → 由 containerd 写入磁盘
路径：/var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log

软链接：/var/log/containers/<pod>_<namespace>_<container>-<id>.log

kubelet 负责日志轮转（logrotate）
默认：单文件 10MB，保留 5 个
```

### Q7: 日志级别怎么配置？

```
K8s 本身不管日志级别
日志级别是应用自己的事

最佳实践：
  - 应用输出结构化日志（JSON 格式）
  - 包含 level 字段（INFO/WARN/ERROR）
  - Loki/ES 按 level 字段过滤

JSON 格式示例：
  {"time":"2024-01-01T00:00:00Z","level":"ERROR","msg":"connection refused"}

查询：
  {app="myapp"} | json | level="ERROR"
```

### Q8: 日志采集的性能影响？

```
DaemonSet 方案对节点的影响：
  - CPU：很低（Promtail 约 50-100m）
  - 内存：约 128-256Mi
  - 磁盘 IO：读取日志文件，影响很小

优化方式：
  - 限制采集速率
  - 过滤不需要的日志（如 healthcheck）
  - 设置资源 limits
  - 日志文件轮转（避免磁盘满）
```

### Q9: 怎么处理日志量过大？

```
1. 应用侧：减少无意义日志（如每个请求的 DEBUG 日志）
2. 采集侧：过滤规则（drop 不需要的）
3. 存储侧：
   - 设置保留时间（如 7 天）
   - 压缩存储
   - 按 namespace/重要度分层存储
4. 查询侧：
   - 限制查询时间范围
   - 用标签缩小范围再搜内容
```

### Q10: 日志告警怎么配？

```
Loki 支持 Ruler 组件做日志告警：

规则示例：
  groups:
  - name: log-alerts
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate({namespace="default"} |= "ERROR" [5m])) > 10
      for: 5m
      annotations:
        summary: "default namespace 每秒超过 10 条 ERROR 日志"

也可以在 Grafana 里配置 Alert：
  查询 → 设阈值 → 通知渠道
```

---

## 🎯 速记版（考前 1 分钟）

```
3 种方案：DaemonSet(最常用) Sidecar(文件日志) 直接推送(侵入)
EFK vs PLG：ES(全文索引,资源大) Loki(标签索引,资源小)
PLG 组件：Promtail(采集DaemonSet) Loki(存储) Grafana(查询)
Loki 理念：只索引标签，内容压缩存储，查询时 grep
日志路径：/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log
LogQL：{标签过滤} |= "内容过滤"
kubectl logs 缺点：临时、单Pod、无搜索、Pod删了就没了
```

---

## 📊 自测清单

```
☐ 1. K8s 日志收集 3 种方案？推荐哪种？
☐ 2. EFK 和 PLG 的核心区别？怎么选？
☐ 3. Loki 为什么比 ES 省资源？
☐ 4. Promtail 怎么自动发现 Pod 日志？
☐ 5. 容器日志存在节点的哪个目录？
☐ 6. kubectl logs 的局限性？
☐ 7. LogQL 怎么查某个 Pod 的 ERROR 日志？
☐ 8. 日志量过大怎么处理？
☐ 9. JSON 结构化日志的好处？
☐ 10. 画出 Promtail → Loki → Grafana 的数据流
```
