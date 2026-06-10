# Day 15：ELK 日志系统 + 缓冲队列（最终版）

> 日期：2026-05-22 ~ 05-23
> 主题：Elasticsearch + Kibana + Filebeat + Redis + Logstash
> 实际部署：极简方案 Filebeat → ES → Kibana
> 资源限制：Master1 4G 内存，ELK 全部跑在 Master1

---

## 📚 模块 1：ELK 架构

### 完整数据流（生产方案）

```
┌─────────┐    ┌──────────┐    ┌─────────────┐    ┌───────────┐    ┌──────┐
│ Pod 日志 │ → │ Filebeat │ → │ Redis/Kafka │ → │ Logstash  │ → │  ES  │
│/var/log/ │    │(DaemonSet)│  │  (缓冲队列)  │    │(解析清洗)  │    │(存储)│
└─────────┘    └──────────┘    └─────────────┘    └───────────┘    └──┬───┘
                                                                      │
                                                                 ┌────▼───┐
                                                                 │ Kibana │
                                                                 │(搜索)  │
                                                                 └────────┘
```

### 极简方案（学习环境，省内存）

```
┌─────────┐    ┌──────────┐    ┌──────┐    ┌────────┐
│ Pod 日志 │ → │ Filebeat │ →  │  ES  │ →  │ Kibana │
└─────────┘    └──────────┘    └──────┘    └────────┘

省掉 Redis + Logstash = 省 800M 内存
```

### 各组件职责（一句话版）

```
Filebeat：   采集日志文件，发到队列或 ES（轻量，DaemonSet）
Redis/Kafka：缓冲队列，削峰填谷，防丢数据
Logstash：   从队列消费，解析/清洗/转换日志
ES：         全文索引 + 存储
Kibana：     Web 界面搜索和可视化
```

### 为什么加缓冲队列？

```
不加：
  Filebeat → Logstash → ES
  问题：
    - 日志高峰 → ES 扛不住 → 数据丢失
    - ES 挂了 → Filebeat 积压 → 节点磁盘满

加了：
  Filebeat → 缓冲队列 → Logstash → ES
  优点：
    1. 削峰填谷（高峰缓冲，低峰消化）
    2. 容错（ES 挂了日志在队列里不丢）
    3. 多消费者（同一份日志给 ES + 告警 + 归档）
```

---

## 📚 模块 2：Kafka vs Redis 对比（面试必知）

```
              Kafka                    Redis
──────────────────────────────────────────────
持久化       ✅ 磁盘持久化             ⚠️ 内存为主
吞吐量       百万级/秒                 万级/秒
消息保留     可配保留天数               消费后删除
多消费者     ✅ Consumer Group         ❌ 一次消费
适用场景     大流量生产环境             小流量/测试
运维难度     较高                       低

选型建议：
  生产大流量（>10GB/天）→ Kafka
  小流量/测试/学习      → Redis
```

### Kafka 核心概念（面试时能讲就行）

```
Topic：   消息分类（如 k8s-logs）
Partition：一个 Topic 可分多个分区（并行处理）
Consumer Group：消费者组（同一组内消息不重复消费）
Offset：  消费位置（可以回溯重新消费）
Broker：  Kafka 服务器节点
KRaft：   新版去掉了 ZooKeeper，用 Raft 协议选主
```

### 面试回答模板

```
Q: 你们日志架构怎么设计的？

A: "Filebeat 采集 → Kafka 做缓冲 → Logstash 解析 → ES 存储 → Kibana 查询。
   加 Kafka 是为了削峰填谷、防止 ES 挂了丢日志、支持多消费者。
   Kafka 用的 KRaft 模式，3 Broker，k8s-logs Topic 3 分区 2 副本。"

Q: 为什么不用 Redis？

A: "Redis 内存为主，大流量下内存压力大，且消费后消息就没了，不支持多消费者。
   Kafka 磁盘持久化，支持按时间回溯，Consumer Group 支持多个消费者。
   小规模或测试环境可以用 Redis，生产我们选 Kafka。"
```

---

## ⚙️ 模块 3：部署前准备

### 3.1 Master1 加内存

```
VMware → 关机 → 设置 → 内存 4096 MB（建议 6G 以上跑完整链路）
```

### 3.2 磁盘扩容（如果 containerd 空间不足）

```bash
# VMware 里硬盘扩到 40G 后
fdisk -l /dev/sda | grep sda3    # 记住 Start 扇区

fdisk /dev/sda
# d → 3 → n → p → 3 → 输入原 Start → 回车 → w

reboot
xfs_growfs /
df -h /    # 确认变大

# 重建 containerd.img（如果损坏）
umount /var/lib/containerd
rm -f /var/lib/containerd.img
dd if=/dev/zero of=/var/lib/containerd.img bs=1M count=10240
mkfs.ext4 -F /var/lib/containerd.img
mount -o loop /var/lib/containerd.img /var/lib/containerd
systemctl start containerd && sleep 10 && systemctl start kubelet
```

### 3.3 kubelet eviction 阈值（防止频繁 disk-pressure）

```bash
cat >> /var/lib/kubelet/config.yaml << 'EOF'
evictionHard:
  imagefs.available: "5%"
  nodefs.available: "5%"
  memory.available: "100Mi"
EOF
systemctl restart kubelet
```

### 3.4 给节点打标签 + 创建命名空间

```bash
kubectl label node k8s-master1 elk=true
kubectl create namespace elk
mkdir -p /root/yaml/elk
```

---

## 🔨 模块 4：部署 Elasticsearch

### 4.1 拉镜像（网络不好时手动拉）

```bash
crictl pull docker.m.daocloud.io/library/busybox:latest
ctr -n k8s.io images tag docker.m.daocloud.io/library/busybox:latest docker.io/library/busybox:latest

crictl pull docker.m.daocloud.io/library/elasticsearch:7.17.0
ctr -n k8s.io images tag docker.m.daocloud.io/library/elasticsearch:7.17.0 docker.elastic.co/elasticsearch/elasticsearch:7.17.0
```

### 4.2 部署

```bash
cat <<'EOF' > /root/yaml/elk/elasticsearch.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elk
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      nodeSelector:
        elk: "true"
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms256m -Xmx256m"
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
        resources:
          requests:
            memory: 400Mi
            cpu: 200m
          limits:
            memory: 512Mi
            cpu: 500m
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elk
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
EOF

kubectl apply -f /root/yaml/elk/elasticsearch.yaml
```

### 4.3 验证

```bash
kubectl exec -n elk elasticsearch-0 -- curl -s http://localhost:9200
# 看到 "You Know, for Search" ✅
```

---

## 🔨 模块 5：部署 Kibana

### 5.1 拉镜像

```bash
crictl pull docker.m.daocloud.io/library/kibana:7.17.0
ctr -n k8s.io images tag docker.m.daocloud.io/library/kibana:7.17.0 docker.elastic.co/kibana/kibana:7.17.0
```

### 5.2 部署

```bash
cat <<'EOF' > /root/yaml/elk/kibana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      nodeSelector:
        elk: "true"
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.0
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch.elk.svc:9200"
        - name: NODE_OPTIONS
          value: "--max-old-space-size=128"
        ports:
        - containerPort: 5601
        resources:
          requests:
            memory: 200Mi
            cpu: 100m
          limits:
            memory: 256Mi
            cpu: 300m
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
spec:
  type: NodePort
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
    nodePort: 30561
EOF

kubectl apply -f /root/yaml/elk/kibana.yaml
```

### 5.3 验证

```
浏览器：http://192.168.105.128:30561
Kibana 启动慢（2-3 分钟），可能因 ES 没完全就绪而重启 1-2 次
```

---

## 🔨 模块 6：部署 Redis 缓冲队列（可选）

> 如果内存够（6G+），可以加 Redis + Logstash 完整链路

```bash
cat <<'EOF' > /root/yaml/elk/redis-queue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-queue
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-queue
  template:
    metadata:
      labels:
        app: redis-queue
    spec:
      nodeSelector:
        elk: "true"
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: 128Mi
          limits:
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-queue
  namespace: elk
spec:
  selector:
    app: redis-queue
  ports:
  - port: 6379
    targetPort: 6379
EOF

kubectl apply -f /root/yaml/elk/redis-queue.yaml
```

验证：

```bash
kubectl exec -n elk <redis-pod-name> -- redis-cli ping
# PONG ✅

# 查看消息队列长度
kubectl exec -n elk <redis-pod-name> -- redis-cli llen k8s-logs
# 数字 > 0 说明 Filebeat 在写入 ✅
```

---

## 🔨 模块 7：部署 Filebeat

### 极简版（直连 ES）

```bash
cat <<'EOF' > /root/yaml/elk/filebeat.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: elk
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"

    output.elasticsearch:
      hosts: ["http://elasticsearch.elk.svc:9200"]
      index: "k8s-logs-%{+yyyy.MM.dd}"

    setup.template.name: "k8s-logs"
    setup.template.pattern: "k8s-logs-*"
    setup.ilm.enabled: false
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elk
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      serviceAccountName: filebeat
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.17.0
        args: ["-c", "/etc/filebeat/filebeat.yml", "-e"]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat
        - name: varlog
          mountPath: /var/log
          readOnly: true
        resources:
          requests:
            memory: 64Mi
            cpu: 50m
          limits:
            memory: 128Mi
            cpu: 100m
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: varlog
        hostPath:
          path: /var/log
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elk
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
- apiGroups: [""]
  resources: ["namespaces","pods","nodes"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: elk
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: filebeat
EOF

kubectl apply -f /root/yaml/elk/filebeat.yaml
```

### Redis 版（output 换成 Redis）

```yaml
# 只改 ConfigMap 的 output 部分
    output.redis:
      hosts: ["redis-queue.elk.svc:6379"]
      key: "k8s-logs"
      datatype: list
```

### 拉镜像（所有节点都需要，DaemonSet）

```bash
# Master1
crictl pull m.daocloud.io/docker.io/elastic/filebeat:7.17.0
ctr -n k8s.io images tag m.daocloud.io/docker.io/elastic/filebeat:7.17.0 docker.elastic.co/beats/filebeat:7.17.0

# 其他节点 SSH 执行同样命令
ssh root@<IP> "crictl pull ... && ctr -n k8s.io images tag ..."
```

---

## 🔨 模块 8：部署 Logstash（可选，需 6G+ 内存）

```bash
cat <<'EOF' > /root/yaml/elk/logstash.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.conf: |
    input {
      redis {
        host => "redis-queue.elk.svc"
        port => 6379
        data_type => "list"
        key => "k8s-logs"
        codec => json
      }
    }

    filter {
      if [kubernetes] {
        mutate {
          add_field => {
            "k8s_namespace" => "%{[kubernetes][namespace]}"
            "k8s_pod" => "%{[kubernetes][pod][name]}"
            "k8s_container" => "%{[kubernetes][container][name]}"
          }
        }
      }
    }

    output {
      elasticsearch {
        hosts => ["http://elasticsearch.elk.svc:9200"]
        index => "k8s-logs-%{+YYYY.MM.dd}"
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      nodeSelector:
        elk: "true"
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.0
        command:
        - /bin/bash
        - -c
        - |
          export LS_JAVA_OPTS="-Xms256m -Xmx256m"
          /usr/share/logstash/bin/logstash -f /usr/share/logstash/pipeline/logstash.conf --pipeline.workers 1 --pipeline.batch.size 125
        volumeMounts:
        - name: pipeline
          mountPath: /usr/share/logstash/pipeline/logstash.conf
          subPath: logstash.conf
        resources:
          requests:
            memory: 384Mi
            cpu: 100m
          limits:
            memory: 512Mi
            cpu: 500m
      volumes:
      - name: pipeline
        configMap:
          name: logstash-config
          items:
          - key: logstash.conf
            path: logstash.conf
EOF

kubectl apply -f /root/yaml/elk/logstash.yaml
```

---

## 📊 模块 9：实验验证结果

### 我们实际验证通过的

```
✅ ES 启动成功，返回 "You Know, for Search"
✅ Kibana 启动成功，浏览器可访问 30561 端口
✅ Redis 启动成功，llen k8s-logs 返回 33000+ 条（证明 Filebeat → Redis 链路通）
✅ Logstash 启动成功
✅ Filebeat DaemonSet 每节点一个，全部 Running
✅ Filebeat 通过 Pod IP 能连通 ES（curl 测试通过）
```

### 未完成（因内存不足 4G）

```
❌ Filebeat 通过 Service DNS 连 ES（kube-proxy/DNS 在内存紧张时不稳定）
❌ 在 Kibana 上看到完整日志
❌ 完整链路 Filebeat → Redis → Logstash → ES → Kibana

原因：Master1 只有 4G，ELK 全链路需要约 3G，加上 K8s 系统组件 1.5G = 4.5G
解决：Master1 加到 6G 即可跑通全链路
```

---

## 📚 模块 10：Logstash Filter 常用配置

```
Logstash 3 大部分：input → filter → output

常用 filter：

1. grok（正则解析非结构化日志）
   filter {
     grok {
       match => { "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:level} %{GREEDYDATA:msg}" }
     }
   }

2. json（解析 JSON 格式日志）
   filter {
     json { source => "message" }
   }

3. mutate（字段操作）
   filter {
     mutate {
       remove_field => ["agent", "ecs"]
       add_field => { "env" => "production" }
     }
   }

4. date（时间解析）
   filter {
     date {
       match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
       target => "@timestamp"
     }
   }
```

---

## 📚 模块 11：Filebeat vs Logstash 分工

```
Filebeat（采集端）：
  - Go 写的，轻量（几十 MB 内存）
  - 只做采集和转发
  - DaemonSet 部署（每节点一个）

Logstash（处理端）：
  - JVM，重量（512M+ 内存）
  - 强大解析能力（grok/json/mutate）
  - 集中部署 1-3 个实例

最佳分工：
  Filebeat 负责"搬运"
  Logstash 负责"加工"
```

---

## 📚 模块 12：ES 索引管理

```
按天建索引：k8s-logs-2026.05.22
ILM 管理：Hot(7天) → Warm(30天) → Cold(90天) → Delete

手动操作：
  # 查看索引
  curl http://localhost:9200/_cat/indices?v
  # 删除旧索引
  curl -X DELETE http://localhost:9200/k8s-logs-2026.05.20
  # 集群健康
  curl http://localhost:9200/_cluster/health?pretty
```

---

## 📚 模块 13：ELK vs PLG 选型

```
             ELK + 缓冲队列           PLG (Loki)
─────────────────────────────────────────────────
组件数量    5 个                      3 个
资源消耗    大（ES 需 8G+）            小（Loki 轻量）
全文搜索    ✅ 强大                    ❌ 只有标签过滤
适用规模    大型（TB 级日志）           中小型
运维难度    高                         低

选型：
  小团队/中小集群 → PLG
  大团队/需要全文搜索 → ELK
```

---

## 📚 模块 14：踩坑记录

### 坑 1：disk-pressure taint 反复出现

```
原因：kubelet 默认 imagefs 阈值 15%，containerd 分区小容易触发
解决：调整 eviction 阈值 + 扩容 containerd.img
```

### 坑 2：containerd.img 用 dd 追加导致文件系统损坏

```
错误操作：dd if=/dev/zero bs=1M count=5120 >> containerd.img
正确操作：删掉重建 → dd of=containerd.img → mkfs.ext4 → mount
```

### 坑 3：Logstash 128m 堆内存启动失败

```
原因：JVM 默认 1G + 自定义 128m 冲突，且 128m 不够
解决：用 command 显式指定 LS_JAVA_OPTS="-Xms256m -Xmx256m"
```

### 坑 4：Kafka OOM

```
原因：Kafka LogCleaner 需要较大堆内存，128m 不够
解决：改为 Redis 代替（学习环境），或给 Kafka 512m+
```

### 坑 5：重建 containerd 后系统组件镜像丢失

```
现象：etcd/apiserver 全部 CrashLoop
解决：手动拉 K8s 系统镜像并 tag
```

---

## ❓ 模块 15：面试题

### Q1: ELK 各组件的作用？

```
Filebeat：轻量采集（DaemonSet），读日志发到队列
Kafka/Redis：缓冲队列，削峰填谷，防丢数据
Logstash：解析（grok/json）、清洗、转换
ES：全文索引 + 存储
Kibana：Web 搜索 + Dashboard
```

### Q2: 为什么加 Kafka/Redis 缓冲？

```
1. 削峰填谷（高峰缓冲，低峰消化）
2. 容错（ES 挂了日志在队列不丢）
3. 多消费者（ES + 告警 + 归档）
```

### Q3: Kafka 和 Redis 做缓冲的区别？

```
ka：磁盘持久化、百万级吞吐、多消费者 → 大流量生产
Redis：内存、万级吞吐、简单队列 → 小流量测Kaf试
```

### Q4: Kafka 核心概念？

```
Topic：消息分类
Partition：分区（并行）
Consumer Group：消费者组（不重复消费）
Offset：消费位置（可回溯）
Broker：服务节点
KRaft：新版选主协议（取代 ZooKeeper）
```

### Q5: Filebeat 和 Logstash 区别？

```
Filebeat：轻量（Go，几十MB），只采集转发
Logstash：重量（JVM，512M+），能解析能转换
分工：Filebeat 搬运 → Logstash 加工
```

### Q6: ES 索引策略怎么设计？

```
按天建索引：k8s-logs-2026.05.22
ILM 管理生命周期：Hot(7天) → Warm(30天) → Delete
```

### Q7: ELK vs PLG 怎么选？

```
ELK：大规模、需要全文搜索、有运维能力
PLG：中小规模、标签过滤够用、轻量低成本
```

### Q8: Logstash grok 怎么用？

```
日志：2024-01-01 10:30:00 ERROR Connection refused
表达式：%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}
结果：3 个字段
```

### Q9: Java 多行日志怎么处理？

```
方案 1（推荐）：应用输出 JSON 格式（一行一条）
方案 2：Filebeat multiline 配置
  multiline.pattern: '^\d{4}-\d{2}-\d{2}'
  multiline.negate: true
  multiline.match: after
```

### Q10: 日志容量怎么规划？

```
ES 存储 = 日均日志量 × 保留天数 × 1.5
例：10GB/天 × 30天 × 1.5 = 450GB
```

---

## 🎯 速记版（考前 1 分钟）

```
全链路：Filebeat(采集) → Kafka/Redis(缓冲) → Logstash(解析) → ES(存储) → Kibana(查询)
加缓冲原因：削峰 + 容错 + 多消费者
Kafka vs Redis：Kafka(大流量/持久/多消费) Redis(小流量/简单)
Filebeat vs Logstash：FB 轻量只采集，LS 重量做解析
Kafka概念：Topic/Partition/ConsumerGroup/Offset/KRaft
ES 索引：按天建，ILM 管生命周期
Logstash Filter：grok(正则) json(JSON解析) mutate(字段) date(时间)
ELK vs PLG：ELK(全文搜索/大) PLG(标签索引/轻)
面试回答：生产用 Kafka + ELK，学习环境资源有限用极简方案验证
```

---

## 📚 模块 16：清理资源

```bash
kubectl delete namespace elk
kubectl delete clusterrole filebeat 2>/dev/null
kubectl delete clusterrolebinding filebeat 2>/dev/null
```

---

## 📊 内存规划参考

```
组件                  最低内存     推荐内存
──────────────────────────────────────────
ES                   512Mi       1Gi
Kibana               256Mi       512Mi
Redis                128Mi       256Mi
Logstash             384Mi       512Mi
Filebeat(每节点)      64Mi        128Mi
Kafka                512Mi       1Gi

极简方案(ES+Kibana+FB)：  ~900Mi
完整方案(+Redis+LS)：     ~1.8Gi
完整方案(+Kafka+LS)：     ~2.3Gi

建议 Master 节点 6G+ 内存跑完整链路
```
