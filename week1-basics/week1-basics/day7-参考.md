# Day 7：Week 1 复盘实战（精简版）

> 日期：2026-05-17
> 主题：Week 1 复盘、知识串联、综合实操、面试模拟

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. Week 1 知识串联测试（闭卷自测）
  ☐ 2. 查漏补缺：标记不熟的知识点

下午（3h）：
  ☐ 3. 综合实操（实验 1：从零部署一套完整应用）
  ☐ 4. 面试模拟（录音/对镜子讲）

晚上（1h）：
  ☐ 5. 整理 Week 1 笔记
  ☐ 6. 更新 PROGRESS.md 并 push
```

---

## 📚 模块 1：闭卷自测（60 分钟）

### 合上所有笔记，逐题回答

**架构组件（Day 1）：**

```
1. K8s 6 大组件分别是什么？各自的作用？
2. kubectl apply 从发出到 Pod 运行的完整流程？
3. 高可用集群怎么设计？VIP 的作用？
```

**Pod 与工作负载（Day 2）：**

```
4. Pod 5 种状态？
5. 3 种探针？各自失败后的后果？
6. Deployment 和 StatefulSet 的 3 个核心区别？
7. DaemonSet 使用场景？
8. CrashLoopBackOff 怎么排查？
```

**Service 与网络（Day 3）：**

```
9. Service 4 种类型？
10. ClusterIP 是真实 IP 吗？怎么实现的？
11. Headless Service 和普通 Service 的 DNS 解析区别？
12. Service 的 DNS 名格式？
```

**Ingress 与存储（Day 4）：**

```
13. Ingress 和 Service（NodePort）的区别？
14. PV 和 PVC 的关系？绑定流程？
15. 静态供给 vs 动态供给？
16. PV 回收策略 Retain 和 Delete 区别？
```

**配置与安全（Day 5）：**

```
17. ConfigMap 和 Secret 区别？
18. RBAC 4 个概念？
19. Secret 的 Base64 是加密吗？
20. NetworkPolicy 默认行为？
```

**调度与资源（Day 6）：**

```
21. 4 种调度方式？
22. Taint 3 种效果？
23. requests 和 limits 区别？超内存会怎样？
24. QoS 3 个等级？
```

---

### 自测结果记录

```
每题标记：
  ✅ = 流利回答
  ⚠️ = 能答但犹豫
  ❌ = 答不出

架构(1-3)：     ☐ ☐ ☐
Pod(4-8)：      ☐ ☐ ☐ ☐ ☐
网络(9-12)：    ☐ ☐ ☐ ☐
存储(13-16)：   ☐ ☐ ☐ ☐
安全(17-20)：   ☐ ☐ ☐ ☐
调度(21-24)：   ☐ ☐ ☐ ☐

统计：
  ✅ _____ 个
  ⚠️ _____ 个（需要复习）
  ❌ _____ 个（需要重点补强）
```

---

## 📚 模块 2：查漏补缺（60 分钟）

**把标记 ⚠️ 和 ❌ 的题目对照参考答案重新学一遍**

```
Day 1 参考 → week1-basics/day1-参考.md
Day 2 参考 → week1-basics/day2-参考.md
Day 3 参考 → week1-basics/day3-参考.md
Day 4 参考 → week1-basics/day4-参考.md
Day 5 参考 → week1-basics/day5-参考.md
Day 6 参考 → week1-basics/day6-参考.md
```

重点：**不是看一遍就行，要能用自己的话讲出来**

---

## ☕ 午休

---

## 📚 模块 3：综合实操（90 分钟）

### 🔨 实验 1：不看笔记，从零部署一套应用

**要求**：覆盖 Week 1 所有知识点

```
目标：部署一个 Nginx 应用
  ✅ Deployment（3 副本）
  ✅ Service（ClusterIP + NodePort）
  ✅ Ingress（域名路由）
  ✅ ConfigMap（自定义 nginx 配置）
  ✅ Secret（模拟密码）
  ✅ PVC（挂载持久化存储）
  ✅ livenessProbe + readinessProbe
  ✅ resources requests/limits
```

**尽量不看笔记，自己写 YAML：**

```bash
mkdir -p /root/yaml/week1-test

# 1. 写 ConfigMap
vi /root/yaml/week1-test/configmap.yaml

# 2. 写 Secret
vi /root/yaml/week1-test/secret.yaml

# 3. 写 PV + PVC
vi /root/yaml/week1-test/storage.yaml

# 4. 写 Deployment（含探针、资源限制、挂载 CM/Secret/PVC）
vi /root/yaml/week1-test/deployment.yaml

# 5. 写 Service
vi /root/yaml/week1-test/service.yaml

# 6. 写 Ingress
vi /root/yaml/week1-test/ingress.yaml

# 7. 一键部署
kubectl apply -f /root/yaml/week1-test/

# 8. 验证
kubectl get all
kubectl get pvc
kubectl get ingress
curl -H "Host: test.example.com" http://192.168.105.128:30080
```

**写不出来的部分，标记下来，说明那个知识点还没掌握。**

---

### 参考答案（实在写不出来再看）

```bash
cat <<'EOF' > /root/yaml/week1-test/all-in-one.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cm
data:
  index.html: |
    <h1>Week 1 Final Test</h1>
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: UEBzc3cwcmQ=
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/test-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-final
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-final
  template:
    metadata:
      labels:
        app: nginx-final
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: data
          mountPath: /data
        env:
        - name: SECRET_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: password
      volumes:
      - name: html
        configMap:
          name: nginx-cm
      - name: data
        persistentVolumeClaim:
          claimName: test-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx-final
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx-final
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30088
  type: NodePort
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
EOF

kubectl apply -f /root/yaml/week1-test/all-in-one.yaml

# 验证
kubectl get deploy,svc,pvc,ingress
curl http://192.168.105.128:30088
curl -H "Host: test.example.com" http://192.168.105.128:30080
```

---

## 📚 模块 4：面试模拟（60 分钟）

### 录音或对着镜子，每题 2-3 分钟

```
第 1 轮：快速问答（12 题，每题 1 分钟）
  1. K8s 核心组件？
  2. Pod 5 种状态？
  3. Service 4 种类型？
  4. Deployment vs StatefulSet？
  5. 3 种探针？
  6. PV 和 PVC 关系？
  7. ConfigMap 和 Secret 区别？
  8. RBAC 4 个概念？
  9. requests 和 limits？
  10. QoS 3 个等级？
  11. Taint 3 种效果？
  12. Headless Service？

第 2 轮：深度问题（5 题，每题 3 分钟）
  1. kubectl apply 完整流程画出来
  2. ClusterIP 怎么实现的
  3. 集群高可用设计
  4. 滚动更新过程（3 副本 maxSurge=1）
  5. Pod CrashLoopBackOff 排查步骤
```

---

## 📚 模块 5：Week 1 知识地图

```
Day 1: 架构 → 6 组件 + 高可用 + kubectl 流程
Day 2: Pod → 5 状态 + 3 探针 + 5 工作负载
Day 3: 网络 → 4 Service + DNS + iptables + Headless
Day 4: 存储 → Ingress 7 层路由 + PV/PVC + StorageClass
Day 5: 安全 → ConfigMap/Secret + RBAC + NetworkPolicy
Day 6: 调度 → nodeSelector/affinity + taint + QoS + quota
Day 7: 复盘 → 综合实操 + 面试模拟

关系串联：
  架构（骨架）→ Pod（血肉）→ 网络（神经）→ 存储（记忆）
  → 安全（免疫）→ 调度（大脑）→ 综合（行动）
```

---

## 🎯 Week 1 速记终极版

```
架构：3M+VIP+haproxy，6组件各司其职
Pod：5状态，3探针(liveness杀/readiness摘/startup保护)
工作负载：Deploy/STS/DS/Job/CronJob
网络：ClusterIP(iptables DNAT) NodePort LB ExternalName
DNS：svc.ns.svc.cluster.local
Ingress：7层路由(域名/路径)，需要Controller
存储：emptyDir(临时) hostPath(节点) PV/PVC(持久)
ConfigMap明文 Secret Base64(不是加密)
RBAC：Role+Binding(ns) ClusterRole+Binding(集群)
调度：nodeSelector/affinity/taint+toleration
资源：requests(调度) limits(上限) 超内存OOM
QoS：Guaranteed > Burstable > BestEffort
```

---

## 📊 最终清单

```
☐ 1. 24 道自测题能答对 20+ 道
☐ 2. 综合实操能不看笔记写 80% 的 YAML
☐ 3. 面试模拟能流利回答
☐ 4. 能画 K8s 架构图 + kubectl 流程图
☐ 5. PROGRESS.md 已更新
☐ 6. 所有笔记已 push 到 GitHub
```
