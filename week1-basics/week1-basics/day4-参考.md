# Day 4：Ingress 与存储（精简版）

> 日期：2026-05-14
> 主题：Ingress 流量入口、PV/PVC 持久化存储、StorageClass

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 3（15 分钟，自测面试题）
  ☐ 2. Ingress 概念 + 部署 Ingress Controller（实验 1）
  ☐ 3. Ingress 路由规则（实验 2）

下午（3h）：
  ☐ 4. Volume 基础（emptyDir / hostPath）（实验 3）
  ☐ 5. PV / PVC / StorageClass（实验 4）
  ☐ 6. StatefulSet + PVC 综合实验（实验 5）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. 更新 PROGRESS.md 并 push
```

---

## 📚 模块 1：Ingress 概念（60 分钟）

### 理论（20 分钟）

**为什么需要 Ingress？**

```
Day 3 学了 NodePort，但它有缺点：
  - 端口有限（30000-32767）
  - 没有域名路由
  - 每个 Service 占一个端口
  - 没有 TLS 终结

Ingress 解决：
  - 用域名区分不同服务（nginx.test.com → nginx，api.test.com → api）
  - 用路径区分（/api → 后端，/ → 前端）
  - 统一 TLS 证书管理
  - 只需要暴露一个入口端口（80/443）
```

**Ingress 的两个概念：**

```
Ingress 资源（规则）：
  → 一段 YAML，定义"域名/路径 → 哪个 Service"
  → 只是规则，本身不处理流量

Ingress Controller（执行者）：
  → 真正监听 80/443 的 Pod
  → 读取 Ingress 规则，生成 nginx/traefik 配置
  → 常用：Nginx Ingress Controller

类比：
  Ingress 资源 = 交通规则（左转去A，右转去B）
  Ingress Controller = 交警（执行规则，指挥交通）
```

**Ingress vs Service 区别：**

```
Service（NodePort）：4 层（TCP），按端口区分
Ingress：7 层（HTTP），按域名/路径区分

                    ┌→ nginx-svc service名称（nginx.test.com  域名）
client → Ingress ──┤
                    └→ api-svc（api.test.com）
```

### 🔨 实验 1：部署 Ingress Controller（40 分钟）

```bash
# 1.1 下载 Nginx Ingress Controller 的 YAML
# 方法 A：在 Windows 浏览器下载后传到 Master1
# 方法 B：直接在 Master1 用 curl（可能需要代理）

# 使用阿里云镜像版本
cat <<'EOF' > /root/yaml/ingress/nginx-ingress-controller.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
    spec:
      containers:
      - name: controller
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.10.0
        args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
      serviceAccountName: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods","secrets","configmaps","events"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses","ingressclasses","ingresses/status"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get","list","watch"]
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get","list","watch","create","update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    nodePort: 30443
  selector:
    app: ingress-nginx
EOF

mkdir -p /root/yaml/ingress
kubectl apply -f /root/yaml/ingress/nginx-ingress-controller.yaml

# 1.2 等待 Controller 启动
kubectl get pods -n ingress-nginx -w
# 等到 Running（可能需要 1-3 分钟拉镜像）

# 1.3 验证
kubectl get svc -n ingress-nginx
# 看到 ingress-nginx-controller，NodePort 30080/30443
```

> **注意**：如果镜像拉不到，用 `crictl pull` + tag 的方式（和之前 Calico 一样）

**✍️ 记录：**
- Ingress Controller 的 Pod 状态：_Running（Pod 名称 ingress-nginx-controller-68d57ff555-ndzdv，状态为 Running，就绪容器 1/1）_________
- NodePort 端口：_80:30080/TCP,443:30443/TCP_________

---

## 📚 模块 2：Ingress 路由规则（60 分钟）

### 理论（10 分钟）

**两种路由方式：**

```
基于域名：
  nginx.test.com → nginx-svc
  api.test.com   → api-svc

基于路径：
  test.com/      → frontend-svc
  test.com/api   → backend-svc
```

### 🔨 实验 2：域名路由 + 路径路由（50 分钟）

```bash
# 2.1 准备两个后端 Service
cat <<EOF > /root/yaml/ingress/backends.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx-app
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd-app
  template:
    metadata:
      labels:
        app: httpd-app
    spec:
      containers:
      - name: httpd
        image: httpd:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    app: httpd-app
  ports:
  - port: 80
EOF

kubectl apply -f /root/yaml/ingress/backends.yaml
kubectl get pods
# 4 个 Pod（2 nginx + 2 httpd）

# 2.2 创建 Ingress 规则（基于域名）
cat <<EOF > /root/yaml/ingress/ingress-host.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  - host: httpd.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
EOF

kubectl apply -f /root/yaml/ingress/ingress-host.yaml
kubectl get ingress

# 2.3 测试域名路由
# 用 curl -H 模拟域名访问
curl -H "Host: nginx.test.com" http://192.168.105.128:30080
# 返回 nginx 欢迎页 ✅

curl -H "Host: httpd.test.com" http://192.168.105.128:30080
# 返回 httpd 欢迎页（It works!）✅

# 同一个端口（30080），不同域名，路由到不同 Service
```

**✍️ 记录：**
- nginx.test.com 返回：_Welcome to nginx! (Nginx 欢迎页)_________
- httpd.test.com 返回：_It works! Apache httpd (Apache httpd 测试页)_________
- 同一端口不同域名如何区分：

通过 HTTP 请求头中的 Host 字段（域名）进行区分，Ingress Controller 根据 Host 规则将请求路由到不同的后端服务

 Ingress 的核心功能——基于域名的虚拟主机

_________

---

## ☕ 午休

---

## 📚 模块 3：Volume 基础（45 分钟）

### 理论（15 分钟）

**为什么需要 Volume？**

```
容器的文件系统是临时的
容器重启 → 数据丢失

Volume 解决：
  把数据存到容器之外的地方
  容器重启/重建后数据还在
```

**3 种常见 Volume 类型：**

```
emptyDir：
  → 临时目录，Pod 删除后数据消失
  → 用途：同 Pod 多容器共享数据、缓存

hostPath：
  → 挂载节点的本地目录
  → Pod 删除后数据还在（在节点上）
  → 用途：日志收集、开发测试
  → 缺点：Pod 漂移到其他节点就找不到了

PV/PVC（下个模块学）：
  → 持久化存储的标准方案
  → 和节点无关
```

### 🔨 实验 3：emptyDir 和 hostPath（30 分钟）

```bash
# 3.1 emptyDir：两个容器共享数据
cat <<EOF > /root/yaml/storage/emptydir-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /data/log.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

mkdir -p /root/yaml/storage
kubectl apply -f /root/yaml/storage/emptydir-demo.yaml

# 等 Pod Running 后
sleep 10
kubectl logs emptydir-demo -c reader
# 能看到 writer 写入的时间 → 两个容器共享了 /data

# 3.2 hostPath：挂载节点目录
cat <<EOF > /root/yaml/storage/hostpath-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: host-log
      mountPath: /usr/share/nginx/html
  volumes:
  - name: host-log
    hostPath:
      path: /tmp/k8s-test
      type: DirectoryOrCreate
EOF

kubectl apply -f /root/yaml/storage/hostpath-demo.yaml

# 在节点上写文件
# 先看 Pod 在哪个节点
kubectl get pod hostpath-demo -o wide
# 假设在 k8s-worker1

# SSH 到那个节点，写个文件
# ssh root@192.168.105.131
# echo "hello from host" > /tmp/k8s-test/index.html

# 或者直接 exec 进 Pod 看
kubectl exec hostpath-demo -- ls /usr/share/nginx/html

# 清理
kubectl delete pod emptydir-demo hostpath-demo
```

**✍️ 记录：**
- emptyDir 用途：__同 Pod 内多容器共享数据_______
- hostPath 的缺点：__Pod 重新调度到其他节点后，读不到原来的数据____

---

## 📚 模块 4：PV / PVC / StorageClass（60 分钟）

### 理论（20 分钟）

**三个概念：**

```
PV（PersistentVolume）：
  → 集群级别的存储资源（管理员创建）
  → 代表一块"实际的存储"
  → 类比：仓库里的柜子

PVC（PersistentVolumeClaim）：
  → 用户的存储申请（开发者创建）
  → "我要一个 10G 的存储"
  → 类比：领用申请单

StorageClass：
  → 自动创建 PV 的"模板"
  → 动态供给：PVC 一创建，PV 自动生成
  → 类比：自动分配柜子的系统
```

**绑定流程：**

```
静态供给：
  管理员创建 PV → 开发者创建 PVC → K8s 自动匹配绑定

动态供给：
  管理员创建 StorageClass → 开发者创建 PVC → K8s 自动创建 PV 并绑定
```

**PV 的回收策略：**

```
Retain：   PVC 删除后 PV 保留（数据不丢）
Delete：   PVC 删除后 PV 一起删（数据丢失）
Recycle：  已废弃
```

**访问模式：**

```
ReadWriteOnce (RWO)：单节点读写
ReadOnlyMany (ROX)： 多节点只读
ReadWriteMany (RWX)：多节点读写
```

### 🔨 实验 4：PV + PVC 手动绑定（40 分钟）

```bash
# 4.1 创建 PV
cat <<EOF > /root/yaml/storage/pv-demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/pv-data
EOF

kubectl apply -f /root/yaml/storage/pv-demo.yaml
kubectl get pv
# STATUS: Available（等待绑定）

# 4.2 创建 PVC
cat <<EOF > /root/yaml/storage/pvc-demo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f /root/yaml/storage/pvc-demo.yaml
kubectl get pvc
# STATUS: Bound（已绑定到 pv-demo）

kubectl get pv
# STATUS: Bound, CLAIM: default/pvc-demo

# 4.3 Pod 使用 PVC
cat <<EOF > /root/yaml/storage/pod-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
EOF

kubectl apply -f /root/yaml/storage/pod-pvc.yaml

# 写入数据
kubectl exec pvc-pod -- sh -c "echo 'PV data' > /usr/share/nginx/html/index.html"

# 删除 Pod 再重建
kubectl delete pod pvc-pod
kubectl apply -f /root/yaml/storage/pod-pvc.yaml

# 数据还在
kubectl exec pvc-pod -- cat /usr/share/nginx/html/index.html
# 输出：PV data ✅（数据持久化成功）

# 4.4 观察删除 PVC 后 PV 的状态
kubectl delete pod pvc-pod
kubectl delete pvc pvc-demo
kubectl get pv
# STATUS: Released（因为 Retain 策略，PV 保留但不可再绑定）

# 清理
kubectl delete pv pv-demo
```

**✍️ 记录：**
- PV 和 PVC 的绑定过程：__________
- Pod 删除后数据是否保留：__________
- Retain 策略下 PVC 删除后 PV 状态：__________

---

## 📚 模块 5：StatefulSet + PVC 综合（30 分钟）

### 理论（5 分钟）

```
StatefulSet 的 volumeClaimTemplates：
  → 自动为每个 Pod 创建独立的 PVC
  → Pod 重建后自动重新绑定原来的 PVC
  → 数据跟着"身份"走（web-0 永远用 web-0 的数据）
```

### 🔨 实验 5：StatefulSet + 独立存储（25 分钟）

```bash
# 5.1 部署带存储的 StatefulSet
cat <<EOF > /root/yaml/storage/sts-storage.yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-svc
spec:
  clusterIP: None
  selector:
    app: sts-demo
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-demo
spec:
  serviceName: sts-svc
  replicas: 2
  selector:
    matchLabels:
      app: sts-demo
  template:
    metadata:
      labels:
        app: sts-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
EOF

kubectl apply -f /root/yaml/storage/sts-storage.yaml

# 5.2 查看自动创建的 PVC
kubectl get pvc
# data-sts-demo-0  ← Pod sts-demo-0 的专属 PVC
# data-sts-demo-1  ← Pod sts-demo-1 的专属 PVC

# 5.3 写数据验证独立性
kubectl exec sts-demo-0 -- sh -c "echo 'I am pod-0' > /usr/share/nginx/html/index.html"
kubectl exec sts-demo-1 -- sh -c "echo 'I am pod-1' > /usr/share/nginx/html/index.html"

kubectl exec sts-demo-0 -- cat /usr/share/nginx/html/index.html
# I am pod-0
kubectl exec sts-demo-1 -- cat /usr/share/nginx/html/index.html
# I am pod-1
# 两个 Pod 的数据互不干扰 ✅

# 5.4 删 Pod 验证数据持久
kubectl delete pod sts-demo-0
kubectl get pods -w
# sts-demo-0 重建后
kubectl exec sts-demo-0 -- cat /usr/share/nginx/html/index.html
# I am pod-0 ← 数据还在 ✅

# 清理
kubectl delete statefulset sts-demo
kubectl delete svc sts-svc
kubectl delete pvc data-sts-demo-0 data-sts-demo-1

# 清理 Ingress 相关（模块 1-2 的）
kubectl delete ingress demo-ingress
kubectl delete deployment nginx-app httpd-app
kubectl delete svc nginx-svc httpd-svc
```

**✍️ 记录：**
- volumeClaimTemplates 自动创建了什么：__________
- 每个 Pod 的存储是否独立：__________
- Pod 重建后数据是否保留：__________

---

## ❓ 模块 6：面试题

### Q1: Ingress 和 Service 的区别？

```
Service（NodePort）：4 层（TCP），按端口区分
Ingress：7 层（HTTP），按域名/路径区分

Ingress 优点：
  - 多个 Service 共用一个入口端口
  - 支持域名路由
  - 支持 TLS
```

### Q2: Ingress 资源和 Ingress Controller 的关系？

```
Ingress 资源 = 规则（YAML 定义的路由）
Ingress Controller = 执行者（真正处理流量的 Pod）

没有 Controller，Ingress 规则不会生效
常用 Controller：Nginx Ingress Controller
```

### Q3: emptyDir 和 hostPath 区别？

```
emptyDir：
  - Pod 级别，Pod 删了数据没了
  - 用途：同 Pod 多容器共享临时数据

hostPath：
  - 节点级别，Pod 删了数据还在节点上
  - 缺点：Pod 漂移到其他节点就找不到数据
  - 用途：日志收集、开发测试
```

### Q4: PV 和 PVC 的关系？

```
PV = 实际存储（管理员提供）
PVC = 存储申请（用户请求）

PVC 声明"我要 10G"
K8s 找到匹配的 PV 自动绑定

类比：
  PV = 仓库里的柜子
  PVC = 领用申请单
```

### Q5: 静态供给 vs 动态供给？

```
静态：管理员手动创建 PV → PVC 绑定
动态：管理员创建 StorageClass → PVC 触发自动创建 PV

生产环境几乎都用动态供给（效率高）
```

### Q6: PV 回收策略有哪些？

```
Retain：PVC 删除后 PV 保留（数据安全）
Delete：PVC 删除后 PV 一起删（数据丢失）
生产环境用 Retain，防止误删
```

### Q7: StatefulSet 的 volumeClaimTemplates？

```
自动为每个 Pod 创建独立 PVC
Pod 重建后自动绑定原来的 PVC
数据跟着"身份"走

例：
  sts-demo-0 → data-sts-demo-0（PVC）
  sts-demo-1 → data-sts-demo-1（PVC）
```

### Q8: 访问模式 RWO / ROX / RWX？

```
RWO（ReadWriteOnce）：单节点读写（最常用）
ROX（ReadOnlyMany）：多节点只读
RWX（ReadWriteMany）：多节点读写（需要 NFS 等支持）
```

### Q9: 有状态应用（MySQL）怎么部署？

```
StatefulSet + Headless Service + PVC

StatefulSet：固定 Pod 名和启动顺序
Headless Service：固定 DNS
volumeClaimTemplates：每个 Pod 独立存储

mysql-0 永远是 master，数据在 data-mysql-0
mysql-1 永远是 slave，数据在 data-mysql-1
```

### Q10: 容器数据持久化的最佳实践？

```
开发测试：emptyDir / hostPath
生产无状态：不需要持久化
生产有状态：PVC + StorageClass（动态供给）
数据库：StatefulSet + PVC + 定期备份
```

### Q11: PV (PersistentVolume) 的存储系统包括哪些类型

本地存储	hostPath、local	
    hostPath：直接挂载节点上的目录（仅测试用）
    local：挂载节点上的磁盘或分区（生产可用）。对读写延迟要求极高的高性能数据库或缓存系统。
网络文件系统	
    NFS	通过网络共享文件，支持多节点同时读写 (RWX)
    需要多Pod共享文件的场景，如文件存储、内容管理系统。
云存储/块存储	
    AWS EBS、GCE PD、Azure Disk、Ceph RBD 等

---

## 🎯 速记版（考前 1 分钟）

```
Ingress：7 层路由（域名/路径），需要 Ingress Controller 才生效
Volume 3 种：emptyDir(临时) hostPath(节点) PV/PVC(持久)
PV=存储 PVC=申请，自动匹配绑定
供给：静态(手动建PV) 动态(StorageClass自动建)
回收：Retain(保留) Delete(删除)
访问：RWO(单节点) ROX(多只读) RWX(多读写)
StatefulSet 存储：volumeClaimTemplates 每 Pod 独立 PVC
```

---

## 📊 自测清单（晚上合上笔记做）

```
☐ 1. Ingress 和 Service（NodePort）的区别？
☐ 2. Ingress 资源和 Ingress Controller 的关系？
☐ 3. emptyDir 和 hostPath 的区别和使用场景？
☐ 4. PV 和 PVC 的关系？绑定流程？
☐ 5. 静态供给和动态供给的区别？
☐ 6. PV 回收策略 Retain 和 Delete 的区别？
☐ 7. RWO / ROX / RWX 分别是什么？
☐ 8. StatefulSet 的 volumeClaimTemplates 做了什么？
☐ 9. 生产环境有状态应用怎么部署？
☐ 10. 画出 client → Ingress → Service → Pod 的流量路径
```
