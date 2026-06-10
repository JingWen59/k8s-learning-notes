# Day 12：故障排查（精简版）

> 日期：2026-05-22
> 主题：K8s 常见故障排查思路、排查命令、实战案例

---

## ✅ 今日学习清单

```
上午（2h）：
  ☐ 1. 复习 Day 11（15 分钟）
  ☐ 2. 排查思路 + 核心命令
  ☐ 3. Pod 故障排查（实验 1）

下午（4h）：
  ☐ 4. 网络故障排查（实验 2）
  ☐ 5. 节点故障排查（实验 3）
  ☐ 6. 集群组件故障排查（实验 4）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：排查思路 + 核心命令（45 分钟）

### 理论（30 分钟）

**排查万能三板斧：**

```
1. kubectl describe <resource> <name>   → 看 Events
2. kubectl logs <pod>                   → 看日志
3. kubectl logs <pod> --previous        → 看崩溃前的日志
```

**排查流程图：**

```
Pod 不正常
    ↓
什么状态？
    ├── Pending        → 调度问题
    │   → describe 看 Events
    │   → 常见：资源不足 / 节点 taint / PVC 未绑定
    │
    ├── ImagePullBackOff → 镜像问题
    │   → 镜像名错了 / 仓库不通 / 认证失败
    │
    ├── CrashLoopBackOff → 启动崩溃
    │   → logs --previous 看原因
    │   → 常见：命令错 / 配置错 / 依赖没启动
    │
    ├── Running 但不工作 → 应用层问题
    │   → exec 进去调试
    │   → 检查端口、进程、配置
    │
    └── Evicted → 资源不足被驱逐
        → 节点磁盘/内存不足
        → 清理资源或扩容
```

**核心排查命令速查：**

```bash
# Pod 层面
kubectl get pods -o wide                 # 看状态和节点
kubectl describe pod <name>              # 看 Events
kubectl logs <pod>                       # 看日志
kubectl logs <pod> --previous            # 上次崩溃日志
kubectl logs <pod> -c <container>        # 多容器指定
kubectl exec -it <pod> -- sh             # 进入调试

# 节点层面
kubectl get nodes                        # 看状态
kubectl describe node <name>             # 看资源/条件/Events
kubectl top nodes                        # 看资源使用（需要 metrics-server）

# Service/网络层面
kubectl get svc                          # 看 Service
kubectl get endpoints <svc>              # 看后端 Pod 列表
kubectl exec <pod> -- nslookup <svc>     # DNS 是否正常
kubectl exec <pod> -- wget -qO- <svc>    # 访问是否通

# 组件层面
crictl ps                                # 看容器
crictl logs <container-id>               # 看容器日志
journalctl -u kubelet --no-pager -l      # kubelet 日志
journalctl -u containerd --no-pager -l   # containerd 日志
```

---

### 🔨 实验 1：Pod 故障排查大全（60 分钟）

#### 场景 1：Pending（资源不足）

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        memory: "999Gi"
EOF

# 排查
kubectl get pod pending-pod
# STATUS: Pending

kubectl describe pod pending-pod | grep -A 5 Events
# FailedScheduling: Insufficient memory

# 修复：降低 requests 或扩容节点
kubectl delete pod pending-pod
```

#### 场景 2：Pending（PVC 未绑定）

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pending
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: not-exist-pvc
EOF

kubectl describe pod pvc-pending | grep -A 5 Events
# persistentvolumeclaim "not-exist-pvc" not found

kubectl delete pod pvc-pending
```

#### 场景 3：ImagePullBackOff

```bash
kubectl run img-fail --image=nginx:不存在的tag
kubectl describe pod img-fail | grep -A 5 Events
# Failed to pull image

# 排查：
# 1. 镜像名/tag 是否正确？
# 2. 仓库是否可达？（crictl pull 测试）
# 3. 是否需要 imagePullSecrets？

kubectl delete pod img-fail
```

#### 场景 4：CrashLoopBackOff

```bash
# 4a: 命令错误
kubectl run crash1 --image=busybox -- 不存在的命令
sleep 20
kubectl logs crash1 --previous
# exec: "不存在的命令": executable file not found

# 4b: 依赖未就绪
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash2
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "wget -q http://not-exist-service && echo ok || exit 1"]
EOF

sleep 30
kubectl logs crash2 --previous
# wget: bad address 'not-exist-service'

kubectl delete pod crash1 crash2
```

#### 场景 5：Running 但访问不通

```bash
# 部署 nginx 但把端口配错
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-port
  labels:
    app: wrong-port
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 8080    # 错的！nginx 监听 80
---
apiVersion: v1
kind: Service
metadata:
  name: wrong-port-svc
spec:
  selector:
    app: wrong-port
  ports:
  - port: 80
    targetPort: 8080         # 转发到 8080，但 nginx 在 80
EOF

# 测试
kubectl run test --image=busybox --rm -it -- wget -qO- -T 3 http://wrong-port-svc
# 超时！

# 排查
kubectl exec wrong-port -- ss -tlnp
# 看到 nginx 监听的是 80，不是 8080

# 修复：targetPort 改成 80
kubectl delete pod wrong-port
kubectl delete svc wrong-port-svc
```

**✍️ 记录排查清单：**
```
Pending     → describe → 看 Events（资源/taint/PVC）
ImagePull   → describe → 镜像名/仓库/认证
Crash       → logs --previous → 命令/配置/依赖
Running不通 → exec 进去检查端口和进程
```

---

## ☕ 午休

---

## 📚 模块 2：网络故障排查（60 分钟）

### 🔨 实验 2：网络问题排查

#### 场景 1：Service 无 Endpoints

```bash
# 故意把 selector 写错
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web-wrong    # 故意写错！和 Pod 标签不匹配
  ports:
  - port: 80
EOF

# 排查
kubectl get endpoints web-svc
# ENDPOINTS: <none>  ← 没有后端！

kubectl get pods --show-labels | grep web
# 标签是 app=web

kubectl get svc web-svc -o yaml | grep -A 2 selector
# selector: app=web-wrong  ← 不匹配！

# 修复
kubectl patch svc web-svc -p '{"spec":{"selector":{"app":"web"}}}'
kubectl get endpoints web-svc
# 出现 Pod IP ✅

kubectl delete deployment web
kubectl delete svc web-svc
```

#### 场景 2：DNS 不通

```bash
# 测试 DNS
kubectl run dns-test --image=busybox:1.28 --rm -it -- nslookup kubernetes
# 如果超时 → CoreDNS 有问题

# 排查 CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
# 是否 Running？

kubectl logs -n kube-system -l k8s-app=kube-dns
# 看有没有报错
```

#### 场景 3：Pod 之间不通

```bash
# 创建两个 Pod
kubectl run pod-a --image=busybox -- sleep 3600
kubectl run pod-b --image=nginx:alpine

# 获取 pod-b 的 IP
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

# 从 pod-a ping pod-b
kubectl exec pod-a -- ping -c 3 $POD_B_IP
# 如果不通 → CNI 插件（Calico）问题

# 排查
kubectl get pods -n kube-system | grep calico
# Calico Pod 是否 Running？

kubectl delete pod pod-a pod-b
```

**✍️ 网络排查清单：**
```
Service 不通 → 检查 Endpoints → 检查 selector 匹配
DNS 不通    → 检查 CoreDNS Pod 状态和日志
Pod 互不通  → 检查 CNI 插件（Calico）状态
NodePort 不通 → 检查 kube-proxy + iptables 规则
```

---

## 📚 模块 3：节点故障排查（45 分钟）

### 🔨 实验 3：节点问题排查

```bash
# 3.1 查看节点状态
kubectl get nodes
kubectl describe node k8s-worker1

# 关注 Conditions 部分：
# MemoryPressure    → 内存不足
# DiskPressure      → 磁盘不足
# PIDPressure       → 进程数不足
# Ready             → 节点是否正常

# 3.2 查看节点资源使用
kubectl describe node k8s-worker1 | grep -A 10 "Allocated resources"
# 看 CPU / Memory 的 Requests 和 Limits

# 3.3 常见节点问题

# 节点 NotReady 排查
ssh root@<node-ip>
systemctl status kubelet
journalctl -u kubelet --no-pager | tail -30
# 常见原因：
#   - kubelet 崩了
#   - containerd 挂了
#   - 证书过期
#   - 内存不足 OOM

# 节点 DiskPressure
df -h
# /var/lib/containerd 满了
# 清理：crictl rmi --prune

# 节点 MemoryPressure
free -h
# 看是否真的内存不足
# 解决：驱逐 Pod 或扩容
```

**✍️ 节点排查清单：**
```
NotReady    → SSH 到节点 → systemctl status kubelet → journalctl
DiskPressure → df -h → crictl rmi --prune
MemoryPressure → free -h → 找到占内存的 Pod
```

---

## 📚 模块 4：集群组件故障排查（30 分钟）

### 🔨 实验 4：控制面组件问题

```bash
# 4.1 查看控制面 Pod
kubectl get pods -n kube-system

# 4.2 查看组件日志
# apiserver
crictl ps | grep apiserver
crictl logs <apiserver-container-id> 2>&1 | tail -20

# etcd
crictl ps | grep etcd
crictl logs <etcd-container-id> 2>&1 | tail -20

# controller-manager
kubectl logs -n kube-system kube-controller-manager-k8s-master1

# scheduler
kubectl logs -n kube-system kube-scheduler-k8s-master1

# 4.3 kubelet 日志
journalctl -u kubelet --no-pager | tail -30

# 4.4 常见组件故障
# apiserver 不响应 → 检查 etcd 连接 / 证书
# scheduler 不工作 → Pod 一直 Pending
# controller 不工作 → Deployment 创建后没有 ReplicaSet
```

---

## ❓ 模块 5：面试题

### Q1: Pod 一直 Pending 怎么排查？

```
1. kubectl describe pod → 看 Events
2. 常见原因：
   - Insufficient cpu/memory（资源不足）
   - 节点有 taint，Pod 没有 toleration
   - PVC 未绑定（存储问题）
   - 没有可用节点（全部 cordon 了）
3. 修复：扩容 / 加 toleration / 创建 PV
```

### Q2: Pod CrashLoopBackOff 怎么排查？

```
1. kubectl logs <pod> --previous → 看崩溃日志
2. kubectl describe pod → 看 Exit Code
3. 常见原因：
   - Exit 1：命令或配置错误
   - Exit 137：OOMKilled（内存超限）
   - Exit 143：被 SIGTERM 杀掉
4. 进阶：kubectl exec 进去调试
```

### Q3: Service 访问不通怎么排查？

```
排查链路：client → Service → Endpoints → Pod

1. kubectl get endpoints <svc>
   → 为空？selector 和 Pod 标签不匹配
   → 有 IP？Pod 没问题

2. kubectl exec <pod> -- wget -qO- http://<svc>
   → 能通？问题在集群外访问方式
   → 不通？检查 kube-proxy / iptables

3. 直接 curl Pod IP
   → 通？问题在 Service 层
   → 不通？问题在 Pod 本身
```

### Q4: 节点 NotReady 怎么排查？

```
1. SSH 到节点
2. systemctl status kubelet → 是否在运行
3. journalctl -u kubelet → 看报错
4. systemctl status containerd → 容器运行时
5. 常见原因：
   - kubelet 挂了 → systemctl restart kubelet
   - containerd 挂了 → systemctl restart containerd
   - 证书过期 → kubeadm certs renew
   - 磁盘满 → 清理
   - 内存 OOM → 找原因
```

### Q5: 怎么排查 Pod 网络不通？

```
分层排查：
1. 同节点 Pod 互通？
   → 不通：bridge 问题
2. 跨节点 Pod 互通？
   → 不通：CNI 插件问题（Calico）
3. Pod → Service 通？
   → 不通：kube-proxy / DNS
4. Pod → 外网通？
   → 不通：节点网关 / NAT

工具：
  ping / wget / nslookup / traceroute
  iptables -t nat -L / ip route
```

### Q6: Exit Code 代表什么？

```
0    → 正常退出
1    → 应用错误
126  → 命令不可执行
127  → 命令未找到
137  → OOMKilled（128 + 9 = SIGKILL）
143  → SIGTERM 优雅终止（128 + 15）
```

### Q7: 如何查看 Pod 被驱逐的原因？

```
kubectl describe pod <evicted-pod>
kubectl describe node <node>

Events 里看到：
  The node was low on resource: ephemeral-storage
  The node was low on resource: memory

原因：节点资源不足，kubelet 主动驱逐低优先级 Pod
```

### Q8: 如何实时排查正在运行的 Pod？

```
# 进入 Pod
kubectl exec -it <pod> -- sh

# 在 Pod 里排查
ps aux            → 看进程
ss -tlnp          → 看端口
cat /etc/resolv.conf → DNS 配置
wget -qO- http://xxx → 测试连接
env               → 看环境变量
df -h             → 看磁盘
free -m           → 看内存
```


### 记一下
Service 不通 → 
检查 Endpoints → 检查 selector 匹配

DNS 不通    → 
检查 CoreDNS Pod 状态和日志

Pod 互不通  → 
检查 CNI 插件（Calico）状态

NodePort 不通 → 
检查 kube-proxy + iptables 规则

---

## 🎯 速记版（考前 1 分钟）

```
万能三板斧：describe → logs → logs --previous
Pending：资源不足 / taint / PVC
ImagePull：镜像名错 / 仓库不通 / 认证
Crash：logs --previous（命令错/配置错/依赖没启动）
Service 不通：检查 Endpoints → selector 匹配
DNS 不通：检查 CoreDNS Pod
节点 NotReady：SSH → kubelet 状态 → journalctl
Exit 137 = OOM，Exit 143 = SIGTERM
```

---

## 📊 自测清单

```
☐ 1. Pod Pending 排查步骤？
☐ 2. CrashLoopBackOff 排查步骤？
☐ 3. Service 访问不通的排查链路？
☐ 4. 节点 NotReady 怎么排查？
☐ 5. Exit Code 137 / 143 分别是什么？
☐ 6. Endpoints 为空说明什么？
☐ 7. DNS 不通怎么排查？
☐ 8. 怎么进入 Pod 内部调试？
☐ 9. 节点 DiskPressure 怎么解决？
☐ 10. 画出"Pod 不正常"的完整排查流程图
```
