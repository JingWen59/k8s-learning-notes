# Day 6：调度策略与资源管理（精简版）

> 日期：2026-05-16
> 主题：Pod 调度、资源限制、QoS、LimitRange、ResourceQuota

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 5（15 分钟）
  ☐ 2. 调度策略 4 种方式（实验 1）
  ☐ 3. Taint + Toleration（实验 2）

下午（3h）：
  ☐ 4. 资源管理：requests / limits（实验 3）
  ☐ 5. QoS 等级 + LimitRange + ResourceQuota（实验 4）

晚上（1h）：
  ☐ 6. 面试题自测
  ☐ 7. push 到 GitHub
```

---

## 📚 模块 1：Pod 调度策略（60 分钟）

### 理论（20 分钟）

**Scheduler 怎么调度 Pod？**

```
1. 过滤（Filtering）：排除不满足条件的节点
   → 资源不够、taint 不容忍、nodeSelector 不匹配

2. 打分（Scoring）：给剩下的节点打分
   → 资源均衡、亲和性匹配等

3. 选择得分最高的节点
```

**4 种调度方式：**

```
1. nodeSelector（最简单）
   → Pod 只调度到有指定标签的节点
   → 硬限制，不匹配就 Pending

2. nodeAffinity（更灵活）
   → 硬限制（required）：必须满足
   → 软限制（preferred）：尽量满足
   → 支持 In / NotIn / Exists / Gt / Lt

3. podAffinity / podAntiAffinity
   → Pod 和其他 Pod 的亲和/反亲和
   → 亲和：把相关 Pod 放在一起（同节点/同区域）
   → 反亲和：把 Pod 分散开（高可用）

4. Taint + Toleration
   → 节点打 Taint（污点）→ 默认排斥所有 Pod
   → Pod 加 Toleration（容忍）→ 才能调度上去
   → 用途：专用节点、Master 不跑业务 Pod
```

### 🔨 实验 1：nodeSelector + nodeAffinity（40 分钟）

```bash
# 1.1 给节点打标签
kubectl label node k8s-worker1 disk=ssd
kubectl get nodes --show-labels | grep disk

# 1.2 nodeSelector：只调度到 disk=ssd 的节点
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disk: ssd
  containers:
  - name: nginx
    image: nginx:alpine
EOF

kubectl get pod ssd-pod -o wide
# 应该在 k8s-worker1 上 ✅

# 1.3 nodeAffinity：软限制
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values: ["ssd"]
  containers:
  - name: nginx
    image: nginx:alpine
EOF

kubectl get pod affinity-pod -o wide
# 尽量在 worker1，但不强制

# 1.4 podAntiAffinity：副本分散到不同节点
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread
  template:
    metadata:
      labels:
        app: spread
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["spread"]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
EOF

调度器给每个节点打分（针对每个新 Pod）
    ↓
检查这个节点上是否已经运行了 app=spread 的 Pod
    ↓
如果已经有 → 减 100 分（尽量避开）
如果没有 → 保持原分（欢迎调度）
    ↓
选择总分最高的节点运行 Pod

kubectl get pods -l app=spread -o wide
# 3 个 Pod 尽量在不同节点 ✅

# 清理
kubectl delete pod ssd-pod affinity-pod
kubectl delete deployment spread-app
kubectl label node k8s-worker1 disk-
```

**✍️ 记录：**
- nodeSelector 效果：__________
- 硬限制 vs 软限制区别：__________
- podAntiAffinity 用途：__________

- nodeSelector 效果：__________

强制将 Pod 调度到拥有指定标签的节点上。 如果没有匹配标签的节点，Pod 会处于 Pending 状态。例如实验中 disk=ssd 的 Pod 被强制调度到了 k8s-worker1。

- 硬限制 vs 软限制区别：__________

硬限制（required）是必须满足条件才能调度，否则 Pod 会一直 Pending；软限制（preferred）是优先尝试满足，但不强制，没有符合条件的节点时会调度到其他节点。 硬限制对应 requiredDuringScheduling...，软限制对应 preferredDuringScheduling...。

- podAntiAffinity 用途：__________

将具有相同标签的 Pod 副本分散到不同的节点（或拓扑域）上，避免多个副本集中在同一个节点，提高服务的高可用性和容错能力。 例如实验中的 spread-app 的 3 个副本被分散到了 3 个不同的 master 节点上。

---

## 📚 模块 2：Taint + Toleration（45 分钟）

### 理论（15 分钟）

**Taint（污点）= 节点排斥 Pod**

```
3 种效果：
  NoSchedule：不调度新 Pod（已有的不受影响）
  PreferNoSchedule：尽量不调度（软限制）
  NoExecute：不调度 + 驱逐已有 Pod
```

**Toleration（容忍）= Pod 容忍节点的 Taint**

```
Pod 只有加了对应的 Toleration，才能调度到有 Taint 的节点

典型场景：
  Master 节点有 Taint → 业务 Pod 不会调度上去
  GPU 节点有 Taint → 只有 GPU 任务能调度上去
```

### 🔨 实验 2：Taint + Toleration（30 分钟）

```bash
# 2.1 查看 Master 已有的 Taint
kubectl describe node k8s-master1 | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule

# 这就是为什么普通 Pod 不会跑在 Master 上

# 2.2 给 Worker1 加 Taint
kubectl taint node k8s-worker1 env=gpu:NoSchedule

# 2.3 部署一个普通 Pod
kubectl run no-tol --image=nginx:alpine
kubectl get pod no-tol -o wide
# 不会调度到 Worker1（因为有 Taint）

# 2.4 部署一个有 Toleration 的 Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-tol
spec:
  tolerations:
  - key: env
    operator: Equal
    value: gpu
    effect: NoSchedule
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: nginx
    image: nginx:alpine
EOF

kubectl get pod with-tol -o wide
# 在 Worker1 上 ✅（容忍了 Taint）

# 2.5 清理 Taint
kubectl taint node k8s-worker1 env=gpu:NoSchedule-
# 注意末尾的 - 表示删除

kubectl delete pod no-tol with-tol
```

**✍️ 记录：**
- Taint 3 种效果：__________
- Master 为什么不跑业务 Pod：__________

---

## ☕ 午休

---

## 📚 模块 3：资源管理 requests / limits（60 分钟）

### 理论（20 分钟）

**requests 和 limits 的区别：**

```
requests（请求量）：
  → Pod 调度时的依据
  → "我至少需要多少资源"
  → 节点 requests 总和不能超过节点资源

limits（限制量）：
  → Pod 运行时的上限
  → "我最多用多少资源"
  → 超过 CPU limits → 被限流（throttle）
  → 超过 Memory limits → 被杀掉（OOMKilled）
```

**CPU 单位：**

```
1 = 1 核
0.5 = 半核 = 500m
100m = 0.1 核
m = millicores（千分之一核）
```

**内存单位：**

```
128Mi = 128 MiB
1Gi = 1 GiB
100M = 100 MB（注意 Mi 和 M 不一样）
```

**面试关键点：**

```
只设 requests 不设 limits → Pod 可以"超卖"
只设 limits 不设 requests → requests 自动等于 limits
两个都设 → 最佳实践
都不设 → 最低优先级（BestEffort），资源紧张时最先被杀
```

### 🔨 实验 3：资源限制（40 分钟）

```bash
# 3.1 设置 requests 和 limits
cat <<EOF > /root/yaml/config/resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
EOF

kubectl apply -f /root/yaml/config/resource-pod.yaml
kubectl describe pod resource-pod | grep -A 6 "Limits"

# 3.2 模拟 OOMKilled
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "dd if=/dev/zero of=/tmp/fill bs=1M count=200"]
    resources:
      limits:
        memory: 64Mi
EOF

sleep 15
kubectl get pod oom-pod
# STATUS: OOMKilled 或 CrashLoopBackOff

kubectl describe pod oom-pod | grep -A 3 "Last State"
# Reason: OOMKilled ✅

# 清理
kubectl delete pod resource-pod oom-pod
```

**✍️ 记录：**
- requests 和 limits 区别：__________
- 超内存的后果：__________
- 超 CPU 的后果：__________

---

## 📚 模块 4：QoS + LimitRange + ResourceQuota（45 分钟）

### 理论（15 分钟）

**QoS 3 个等级（资源紧张时谁先被杀）：**

```
Guaranteed（最高优先级）：
  → requests = limits（CPU 和内存都设且相等）
  → 最后被杀

Burstable（中等）：
  → 设了 requests 或 limits，但不相等
  → 中间被杀

BestEffort（最低）：
  → 什么都没设
  → 最先被杀
```

**LimitRange（命名空间默认值）：**

```
给 namespace 设置默认的 requests/limits
Pod 没写 → 自动加上默认值
防止"忘了写资源限制"
```

**ResourceQuota（命名空间配额）：**

```
限制整个 namespace 的资源总量
例：default namespace 最多用 4 核 CPU、8Gi 内存
防止某个 namespace 吃掉集群所有资源
```

### 🔨 实验 4：LimitRange + ResourceQuota（30 分钟）

```bash
# 4.1 创建 LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      cpu: 200m
      memory: 128Mi
    defaultRequest:
      cpu: 100m
      memory: 64Mi
    type: Container
EOF

# 测试：创建一个没写资源限制的 Pod
kubectl run auto-limit --image=nginx:alpine
kubectl describe pod auto-limit | grep -A 6 "Limits"
# 自动加上了 200m CPU / 128Mi Memory ✅

# 4.2 创建 ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"
EOF

kubectl describe quota ns-quota
# 看到已用/总量

# 4.3 查看 QoS
kubectl run guaranteed --image=nginx:alpine --dry-run=client -o yaml | \
  sed 's/resources: {}/resources:\n          requests:\n            cpu: 100m\n            memory: 64Mi\n          limits:\n            cpu: 100m\n            memory: 64Mi/' | \
  kubectl apply -f -

kubectl describe pod guaranteed | grep "QoS Class"
# QoS Class: Guaranteed ✅

# 清理
kubectl delete pod auto-limit guaranteed
kubectl delete limitrange default-limits
kubectl delete quota ns-quota
```

**✍️ 记录：**
- QoS 3 个等级：__________
- LimitRange 的作用：__________
- ResourceQuota 的作用：__________

---

## ❓ 模块 5：面试题

### Q1: requests 和 limits 区别？

```
requests：调度依据，"最少需要多少"
limits：运行上限，"最多用多少"
超 CPU → 限流，超内存 → OOMKilled
```

### Q2: QoS 3 个等级？

```
Guaranteed：requests=limits，最后被杀
Burstable：设了但不相等，中间被杀
BestEffort：什么都没设，最先被杀
```

### Q3: nodeSelector 和 nodeAffinity 区别？

```
nodeSelector：简单的标签匹配，硬限制
nodeAffinity：更灵活
  - required（硬）/ preferred（软）
  - 支持 In/NotIn/Exists/Gt/Lt 操作符
```

### Q4: Taint 和 Toleration？

```
Taint：节点打"污点"，排斥 Pod
Toleration：Pod 加"容忍"，可以调度到有 Taint 的节点
3 种效果：NoSchedule / PreferNoSchedule / NoExecute
```

### Q5: Master 为什么不跑业务 Pod？

```
Master 有 Taint：node-role.kubernetes.io/control-plane:NoSchedule
普通 Pod 没有对应的 Toleration → 不会调度上去
DaemonSet（如 Calico）有 Toleration → 可以跑在 Master 上
```

### Q6: 怎么让 Pod 分散到不同节点？

```
podAntiAffinity：
  同一标签的 Pod 尽量不在同一节点
  topologyKey: kubernetes.io/hostname

用途：高可用部署，避免单节点故障影响所有副本
```

### Q7: LimitRange 和 ResourceQuota 区别？

```
LimitRange：单个 Pod/Container 的默认值和上下限
ResourceQuota：整个 Namespace 的资源总配额

LimitRange → 控制个体
ResourceQuota → 控制整体
```

### Q8: Pod 被 OOMKilled 怎么办？

```
1. kubectl describe pod → 确认 OOMKilled
2. 原因：内存使用超过 limits
3. 解决：
   - 调大 limits
   - 优化应用内存使用
   - 检查是否有内存泄漏
```

---

## 🎯 速记版

```
调度 4 方式：nodeSelector(标签) nodeAffinity(灵活) podAffinity(亲和) taint+toleration(排斥)
Taint 3 效果：NoSchedule / PreferNoSchedule / NoExecute
requests=调度依据，limits=运行上限
超CPU→限流，超内存→OOMKilled
QoS：Guaranteed(最高) Burstable(中) BestEffort(最低)
LimitRange→单个Pod默认值，ResourceQuota→Namespace总配额
```

---

## 📊 自测清单

```
☐ 1. Scheduler 调度 Pod 的 2 个步骤？
☐ 2. nodeSelector 和 nodeAffinity 区别？
☐ 3. Taint 3 种效果？
☐ 4. Master 不跑业务 Pod 的原理？
☐ 5. requests 和 limits 区别？
☐ 6. CPU 和 Memory 超限的后果分别是什么？
☐ 7. QoS 3 个等级？哪个最先被杀？
☐ 8. LimitRange 和 ResourceQuota 区别？
☐ 9. 怎么让 3 副本分散到不同节点？
☐ 10. Pod 被 OOMKilled 怎么排查？
```
