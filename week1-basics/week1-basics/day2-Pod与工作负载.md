# Day 2：Pod 与工作负载（参考答案）

> 日期：2026-05-12
> 主题：Pod 生命周期、5 种工作负载、健康检查、滚动更新

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 1（15 分钟，自测面试题）
  ☐ 2. Pod 基础 + 生命周期 → 立即实操验证
  ☐ 3. 重启策略 → 立即实操验证
  ☐ 4. 三种探针 → 立即实操验证

下午（3h）：
  ☐ 5. Deployment → 立即实操 + 滚动更新实验
  ☐ 6. StatefulSet / DaemonSet / Job / CronJob → 各部署一个
  ☐ 7. 状态排查练习（故意制造故障）

晚上（1h）：
  ☐ 8. 整理面试题（合上笔记自测）
  ☐ 9. 更新 PROGRESS.md
  ☐ 10. push 到 GitHub
```

---

## 📚 模块 1：Pod 基础 + 生命周期（45 分钟）

### 理论（15 分钟）

**Pod 是什么？**

```
Pod = K8s 最小运行单位
一个 Pod 里可以有 1 个或多个容器
这些容器共享：网络（同一个 IP）、存储（同一个 Volume）

类比：
  Pod = 一间宿舍
  容器 = 宿舍里的人
  同宿舍的人共享地址和水电
```

**Pod 5 种状态：**

```
Pending    → 等调度或拉镜像
Running    → 至少一个容器在跑
Succeeded  → 所有容器正常退出（exit 0）
Failed     → 至少一个容器异常退出
Unknown    → 节点失联
```

**常见子状态（Events 里看到的）：**

```
ContainerCreating   → 正在拉镜像/创建容器
ImagePullBackOff    → 镜像拉不到，退避重试
CrashLoopBackOff    → 反复崩溃，重启间隔越来越长
OOMKilled           → 内存超限被杀
```

### 🔨 立即实操（30 分钟）

#### 实验 1：创建一个 Pod，观察状态变化

```yaml
# yaml/pod/simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

```bash
# 部署
kubectl apply -f yaml/pod/simple-pod.yaml

# 观察状态变化
kubectl get pods -w
# Pending → ContainerCreating → Running

# 看详细信息
kubectl describe pod simple-pod

# 看日志
kubectl logs simple-pod

# 进入容器
kubectl exec -it simple-pod -- sh
# 退出：exit
```

**✍️ 记录**：你看到的状态变化过程

---

#### 实验 2：故意制造 ImagePullBackOff

```bash
kubectl run bad-image --image=这个镜像不存在:v999

# 观察
kubectl get pods -w
# Pending → ErrImagePull → ImagePullBackOff

# 看原因
kubectl describe pod bad-image | tail -10

# 清理
kubectl delete pod bad-image
```

**✍️ 记录**：Events 里报了什么错误？

---

#### 实验 3：制造 CrashLoopBackOff

```bash
kubectl run crash-pod --image=busybox -- sh -c "exit 1"

# 观察
kubectl get pods -w
# Running → Error → CrashLoopBackOff
# 注意重启间隔越来越长

# 看日志
kubectl logs crash-pod --previous

# 清理
kubectl delete pod crash-pod
```

**✍️ 记录**：重启间隔是多少？（10s → 20s → 40s → ...）

---

#### 实验 4：对比裸 Pod 和 Deployment

```bash
# 裸 Pod
kubectl run bare-pod --image=nginx:alpine
kubectl delete pod bare-pod
kubectl get pods
# → 没了，永远不会回来 ❌

# Deployment
kubectl create deployment deploy-test --image=nginx:alpine
kubectl delete pod $(kubectl get pods -l app=deploy-test -o name | head -1)
kubectl get pods
# → 自动重建了 ✅

# 清理
kubectl delete deployment deploy-test
```

**✍️ 记录**：裸 Pod 和 Deployment 管理的 Pod 区别是什么？

---

## 📚 模块 2：重启策略（20 分钟）

### 理论（5 分钟）

```
Always     → 无论怎么退出都重启（Deployment 默认）
OnFailure  → exit 非 0 才重启（Job 常用）
Never      → 不重启
```

**退避机制（CrashLoopBackOff）：**

```
第 1 次：立即重启
第 2 次：10 秒后
第 3 次：20 秒后
第 4 次：40 秒后
...
最长间隔：5 分钟
```

### 🔨 立即实操（15 分钟）

#### 实验 5：对比 Always vs Never

```yaml
# yaml/pod/restart-always.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-always
spec:
  restartPolicy: Always
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "echo hello; exit 0"]

---
# yaml/pod/restart-never.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-never
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "echo hello; exit 0"]
```

```bash
kubectl apply -f yaml/pod/restart-always.yaml
kubectl apply -f yaml/pod/restart-never.yaml

# 等 30 秒后观察
kubectl get pods

# restart-always → CrashLoopBackOff（反复重启）
# restart-never  → Completed（正常退出，不重启）

# 清理
kubectl delete pod restart-always restart-never
```

**✍️ 记录**：两种策略的区别在实际行为上是什么？

---

## 📚 模块 3：三种探针（50 分钟）

### 理论（15 分钟）

```
livenessProbe（存活）：
  → 容器活着吗？
  → 失败 → 杀掉重启
  → 类比：检查心跳

readinessProbe（就绪）：
  → 容器准备好了吗？
  → 失败 → 从 Service 移除（不接流量）
  → 类比：医生准备好接诊了吗

startupProbe（启动）：
  → 容器启动完了吗？
  → 启动完成前不检查 liveness/readiness
  → 适用：启动慢的应用（Java）
```

**检测方式：**

```
httpGet     → 发 HTTP 请求，2xx/3xx 算成功
tcpSocket   → TCP 连接上算成功
exec        → 执行命令，exit 0 算成功
```

**关键参数：**

```
initialDelaySeconds → 启动后等多久开始检测
periodSeconds       → 每隔多久检测一次
failureThreshold    → 连续失败几次算失败
```

### 🔨 立即实操（35 分钟）

#### 实验 6：存活探针 - 故意让它失败

```yaml
# yaml/pod/liveness-fail.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
spec:
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy; sleep 30; rm /tmp/healthy; sleep 600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f yaml/pod/liveness-fail.yaml
kubectl get pods -w

# 前 30 秒：Running（/tmp/healthy 存在）
# 30 秒后：探针失败（文件被删了）
# 连续 3 次失败 → 容器被杀 → 重启
# 重启后又好了 30 秒 → 又失败 → 循环

kubectl describe pod liveness-fail
# Events 里看到：Liveness probe failed → Killing → Started
```

**✍️ 记录**：
- 探针失败后多久容器被杀？
- describe 的 Events 里看到了什么？

---

#### 实验 7：就绪探针 - 控制流量

```yaml
# yaml/pod/readiness-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-test
  labels:
    app: readiness-test
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 5
```

```bash
kubectl apply -f yaml/pod/readiness-test.yaml

# 创建 Service
kubectl expose pod readiness-test --port=80

# 立即查看（Pod 还没 Ready）
kubectl get pods
kubectl get endpoints readiness-test
# Endpoints 为空！Service 不会转发流量给它

# 等 15 秒后再看
kubectl get pods
# READY 变成 1/1
kubectl get endpoints readiness-test
# 出现了 Pod IP
```

**✍️ 记录**：
- readinessProbe 通过前，Pod 的 READY 列是什么？
- Endpoints 从空变成有 IP 的时间点？

---

#### 实验 8：对比 liveness 和 readiness 的后果

```
liveness 失败  → 容器被杀重启（激进）
readiness 失败 → 只是不接流量（温和）

场景选择：
  应用挂死了（无响应）    → 需要 liveness 重启它
  应用在启动/预热中       → 需要 readiness 不转发流量
  应用短暂过载           → 用 readiness（别杀它，等它恢复）
```

```bash
# 清理
kubectl delete pod liveness-fail readiness-test
kubectl delete svc readiness-test
```

---

## ☕ 休息 10 分钟

---

## 📚 模块 4：Deployment + 滚动更新（50 分钟）

### 理论（10 分钟）

**Deployment 核心：**

```
Deployment → 管理 → ReplicaSet → 管理 → Pod

Deployment 不直接管 Pod
它通过 ReplicaSet 间接管理
```

**滚动更新关键参数：**

```
maxSurge：更新时最多多出几个 Pod
maxUnavailable：更新时最多不可用几个 Pod

例：3 副本，maxSurge=1，maxUnavailable=1
  → 更新中最多 4 个 Pod（3+1）
  → 更新中最少 2 个可用（3-1）
```

### 🔨 立即实操（40 分钟）

#### 实验 9：部署 Deployment

```yaml
# yaml/deployment/nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f yaml/deployment/nginx-deploy.yaml

# 看全景
kubectl get deploy,rs,pods
```

**✍️ 记录**：Deployment、ReplicaSet、Pod 三者的关系

---

#### 实验 10：滚动更新 + 回滚

```bash
# 终端 1：watch Pod 变化
kubectl get pods -w

# 终端 2：执行更新
kubectl set image deployment/nginx-deploy nginx=nginx:1.21

# 观察终端 1：
# 新 Pod 创建（1.21）→ 旧 Pod 删除 → 逐个替换

# 查看更新状态
kubectl rollout status deployment/nginx-deploy

# 查看历史版本
kubectl rollout history deployment/nginx-deploy

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deploy

# 确认回滚成功
kubectl describe deployment nginx-deploy | grep Image
# 应该回到 nginx:alpine
```

**✍️ 记录**：
- 更新过程中，Pod 是怎么逐步替换的？
- 回滚后镜像版本变回了什么？

这个实验在干什么？
简单说：演示如何无停机地升级应用，以及升级出问题后如何快速恢复。

核心概念（就两句）
滚动更新：不一次性把所有 Pod 都换掉，而是一个一个换 → 升级过程中服务不会中断

回滚：新版本有问题，一键恢复到旧版本

操作对应实际效果
命令	实际效果	类比
kubectl set image ... nginx=1.21	把 nginx 从 alpine 版升级到 1.21 版	微信从 8.0 升级到 9.0
kubectl get pods -w	实时看 Pod 一个变一个	盯着进度条
kubectl rollout undo	回到上一个版本	微信 9.0 闪退，退回 8.0
你观察到的现象（终端1）
text
旧Pod Running → 新Pod创建 → 新Pod Running → 旧Pod删除
旧Pod Running → 新Pod创建 → 新Pod Running → 旧Pod删除
旧Pod Running → 新Pod创建 → 新Pod Running → 旧Pod删除
3个Pod，逐个替换，整个过程服务不中断。

记录答案
✍️ 更新过程中，Pod是怎么逐步替换的？

先创建一个新版本Pod，等它启动成功后，再删除一个旧版本Pod。重复这个过程，直到全部替换成新版本。

✍️ 回滚后镜像版本变回了什么？

变回 nginx:alpine（升级前的版本）

一句话总结
滚动更新 = 逐个替换 Pod，不中断服务；回滚 = 一键回到旧版本。

---

#### 实验 11：扩缩容

```bash
# 扩到 5 个
kubectl scale deployment nginx-deploy --replicas=5
kubectl get pods

# 缩到 2 个
kubectl scale deployment nginx-deploy --replicas=2
kubectl get pods
```

---

## 📚 模块 5：其他工作负载（50 分钟）

### DaemonSet（15 分钟）

**理论：** 每个节点跑一个 Pod，用于日志/监控/网络插件

```yaml
# yaml/daemonset/log-agent.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: agent
        image: busybox
        command: ["sh", "-c", "while true; do echo collecting logs; sleep 60; done"]
```

```bash
kubectl apply -f yaml/daemonset/log-agent.yaml
kubectl get pods -o wide
# 每个节点一个 Pod ✅

kubectl delete daemonset log-agent
```

---

### StatefulSet（15 分钟）

**理论：**

```
和 Deployment 的区别：
  - Pod 名字固定（xxx-0, xxx-1, xxx-2）
  - 每个 Pod 独立存储
  - 有序启动/删除
  - 需要 Headless Service
```

```yaml
# yaml/statefulset/web-sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  clusterIP: None         # Headless Service
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-svc    # 必须关联 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
```

```bash
kubectl apply -f yaml/statefulset/web-sts.yaml

# 观察 Pod 名字
kubectl get pods
# web-0  web-1  web-2  ← 固定有序

# 观察启动顺序（如果删除后重建）
kubectl delete pod web-1
kubectl get pods -w
# web-1 重建，名字不变

# DNS 验证
kubectl exec web-0 -- nslookup web-1.web-svc
# 能解析到 web-1 的 IP

kubectl delete statefulset web
kubectl delete svc web-svc
```
这个 kubectl exec web-0 -- nslookup web-1.web-svc 命令，其实是在验证 StatefulSet 的三大核心特性：

稳定的网络标识 (Stable Network Identity)

有序的部署和扩展 (Ordered Deployment & Scaling)

稳定的持久化存储 (Stable Storage)（虽然你这次的 YAML 没涉及，但概念上是一体的）


1. 验证：Pod 名字固定有序 (web-0, web-1, web-2)
你做的：

bash
kubectl get pods
# web-0  web-1  web-2
验证的目的：
这验证了 StatefulSet 的第一个核心特性——稳定的、可预测的 Pod 名称。

对比 Deployment：Deployment 创建的 Pod 名字后面是一串随机哈希（例如 nginx-deploy-866996f4f4-4jfmm）。Pod 重建后，这个随机串会变，身份就变了。

StatefulSet 的行为：StatefulSet 管理的 Pod 拥有固定的、从 0 到 N-1 的序号。web-0、web-1、web-2 这个名字就是它们在集群中的“身份证”，永远不会变。这对于需要明确知道“同伴”是谁的应用（比如数据库集群）至关重要。

一句话：验证了 StatefulSet 的 Pod 有“固定名字”，不随机。

2. 验证：删除后重建，名字不变，且重建顺序可控
你做的：

bash
kubectl delete pod web-1
kubectl get pods -w
# web-1 重建，名字不变
验证的目的：
这验证了 StatefulSet 的第二个核心特性——Pod 状态和身份的“粘性”。

核心价值：web-1 这个 Pod 因为某种原因（比如节点故障、手动删除）挂掉了，但 StatefulSet 控制器会把它重新拉起来，并且新 Pod 的名字还叫 web-1。

这意味着什么？ 对于有状态应用，这意味着：

新的 web-1 会试图挂载和之前 web-1 一样的数据存储卷（如果你配置了持久化存储）。

应用集群里的其他成员（比如 web-0）知道，联系 web-1 就能找到那个“邻居”。

对比实验：你之前在 Deployment 上做删除 Pod 的实验，新 Pod 的名字是完全不同的。

一句话：验证了 StatefulSet 的 Pod 即便“死了重来”，它的“身份”也保持不变。

3. 验证：Pod 之间能够通过“稳定的网络标识”互相访问
你做的：

bash
kubectl exec web-0 -- nslookup web-1.web-svc
# 能解析到 web-1 的 IP
验证的目的：
这验证了第三个核心特性——稳定的、可预测的 DNS 名称。

你正在做的事情：让 web-0 这个 Pod 去问集群的 DNS 服务器：“web-1.web-svc 这个地址是谁？”

成功的意义：

web-1.web-svc 这个 DNS 名字是稳定且可预测的。你不需要知道 web-1 这个 Pod 此刻的 IP 地址（它重启后可能会变），你只需要知道它的名字。
web-0 成功解析到了 web-1 的 IP，这说明在 StatefulSet 里，Pod 之间可以通过 <pod-name>.<service-name> 这种格式的域名互相找到对方。
应用场景：这对于有状态集群是生命线。比如，一个数据库的主节点需要知道所有从节点的地址来同步数据。现在，主节点不需要记录一堆变化的 IP，只需要知道 db-0, db-1, db-2 这几个固定的名字就行了。

一句话：验证了 StatefulSet 的 Pod 之间可以通过“固定的名字”互相找到对方。



---

### Job 和 CronJob（20 分钟）

**Job：跑一次就结束**

```yaml
# yaml/job/pi-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  template:
    spec:
      containers:
      - name: pi
        image: busybox
        command: ["sh", "-c", "echo 任务完成; sleep 5"]
      restartPolicy: OnFailure
```

```bash
kubectl apply -f yaml/job/pi-job.yaml
kubectl get jobs
kubectl get pods
# 等 5 秒
kubectl get pods
# 状态变成 Completed ✅

kubectl delete job pi-job
```

 Job 的任务一次性完成（Completed）特性！

你看到的关键现象
bash
pi-job-m2v7h   0/1    Completed   0    22m   # ✅ Job 完成了
状态是 Completed，不是 Running！




**CronJob：定时跑**

```yaml
# yaml/job/hello-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"    # 每分钟一次
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["sh", "-c", "echo Hello $(date)"]
          restartPolicy: OnFailure
```

```bash
kubectl apply -f yaml/job/hello-cronjob.yaml

# 等 2-3 分钟
kubectl get cronjob
kubectl get jobs       # 多个 Job
kubectl get pods       # 每个 Job 对应一个 Pod

kubectl delete cronjob hello-cron
```
这个实验想告诉你：

CronJob 会按照 schedule 自动创建 Job

每次触发时间戳不同（29642871 → 29642872 → 29642873）

任务成功完成后 Pod 变成 Completed

历史 Job 和 Pod 会被保留（方便查看日志）

CronJob 是「定时器 + Job」的组合

你创建的 hello-cron CronJob 本身不在节点上，但它每分钟创建的 Pod 会运行在工作节点（如 worker1）上
---

## 📚 模块 6：Pod 排查练习（30 分钟）

### 排查 3 种常见错误

```bash
# 1. ImagePullBackOff
kubectl run img-err --image=不存在:v1
kubectl describe pod img-err | grep -A 5 Events

# 2. CrashLoopBackOff
kubectl run crash --image=busybox -- sh -c "exit 1"
kubectl logs crash --previous

# 3. Pending（资源不足）
# 创建一个请求 999G 内存的 Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        memory: "999Gi"
EOF

kubectl describe pod big-pod | grep -A 5 Events
# FailedScheduling: Insufficient memory

这算 Pending！

从哪里看出来的？
从 FailedScheduling 这个事件看出来的！

text
Events:
  Type     Reason            Age   Message
  ----     ------            ----  -------
  Warning  FailedScheduling  9s    0/4 nodes are available: 1 Insufficient memory, 3 node(s) had untolerated taint
FailedScheduling = 调度失败 = Pod 处于 Pending 状态


# 清理
kubectl delete pod img-err crash big-pod
```

---

## 📚 模块 7：清理所有资源

```bash
kubectl delete deployment nginx-deploy 2>/dev/null
kubectl delete pod simple-pod 2>/dev/null
kubectl get pods
# 应该干干净净
```

---

## ❓ 模块 8：面试题整理

### Q1: Pod 和容器什么区别？

```
容器 = 一个进程
Pod = 一组容器的壳，共享网络和存储
Pod 是调度的最小单位
```

### Q2: Pod 有几种状态？

```
Pending / Running / Succeeded / Failed / Unknown
```

### Q3: Deployment 和 StatefulSet 区别？

```
Deployment：Pod 名随机、共享存储、并行启动、无状态
StatefulSet：Pod 名固定（xxx-0）、独立 PVC、有序启动、有状态
```

### Q4: 三种探针的区别？

```
liveness：活着吗？失败→杀掉重启
readiness：准备好了吗？失败→从 Service 移除
startup：启动完了吗？完成前不检查其他探针
```

### Q5: maxSurge 和 maxUnavailable？

```
maxSurge：更新时最多多几个 Pod
maxUnavailable：更新时最多挂几个 Pod
例：3副本，1/1 → 最多4个，最少2个可用
```

### Q6: DaemonSet 使用场景？

```
日志采集、监控 agent、网络插件
每个节点一个，自动部署和清理
```

### Q7: CrashLoopBackOff 怎么排查？

```
kubectl describe pod → 看 Events
kubectl logs --previous → 看崩溃前的日志
常见原因：命令错误、配置错误、依赖没启动、OOM
```

### Q8: 为什么不用裸 Pod？

```
裸 Pod 删了没人管
Deployment 有自愈、滚动更新、回滚、扩缩容
```

### Q9: Job 的重启策略？

```
只能用 OnFailure 或 Never，不能用 Always
因为 Job 本来就要结束的，Always 会无限重启
```

### Q10: StatefulSet 为什么需要 Headless Service？

```
给每个 Pod 提供固定 DNS：
  pod名.service名.namespace.svc.cluster.local
普通 Service 只有一个 ClusterIP（无法区分具体 Pod）
Headless（ClusterIP=None）直接解析到每个 Pod IP
```

---

## 🎯 精简速记版（考前 1 分钟）

```
Pod 5状态：Pending/Running/Succeeded/Failed/Unknown
重启策略：Always(默认) OnFailure(Job) Never(调试)
3探针：liveness(杀重启) readiness(摘流量) startup(保护期)
5工作负载：Deploy(无状态) STS(有状态) DS(每节点) Job(一次) CronJob(定时)
滚动更新：maxSurge(多几个) maxUnavailable(挂几个)
排查3连：describe→logs→logs --previous
```

---

## 📊 自测清单（晚上合上笔记做）

```
☐ 1. Pod 5 种状态？
☐ 2. CrashLoopBackOff 含义和排查步骤？
☐ 3. 3 种探针各自的"失败后果"？
☐ 4. Deployment vs StatefulSet 3 个区别？
☐ 5. 3 副本 maxSurge=1 的滚动更新过程？
☐ 6. DaemonSet 适用场景？
☐ 7. Job 为什么不能用 restartPolicy: Always？
☐ 8. 裸 Pod 和 Deployment 区别？
☐ 9. readiness 失败 vs liveness 失败的区别？
☐ 10. 用 3 分钟讲完 Pod 生命周期
```
