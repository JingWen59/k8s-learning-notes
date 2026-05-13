# Day 3：Service 与网络（精简版）

> 日期：2026-05-13
> 主题：Service 4 种类型、kube-proxy、DNS 解析、网络模型

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 2（15 分钟，自测面试题）
  ☐ 2. K8s 网络模型 + Service 基础（实验 1）
  ☐ 3. Service 4 种类型对比（实验 2）

下午（3h）：
  ☐ 4. kube-proxy 工作原理（实验 3）
  ☐ 5. DNS 解析（实验 4）
  ☐ 6. Headless Service + Endpoints（实验 5）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. 更新 PROGRESS.md 并 push
```

---

## 📚 模块 1：K8s 网络模型 + Service 基础（60 分钟）

### 理论（20 分钟）

**K8s 网络的 3 个基本要求：**

```
1. Pod 与 Pod 之间可以直接通信（不需要 NAT）
2. Node 与 Pod 之间可以直接通信
3. Pod 看到自己的 IP 和别人看到它的 IP 是一样的
```

**为什么需要 Service？**

```
问题：
  Pod IP 不固定（Pod 重建后 IP 变了）
  多副本时，客户端不知道连哪个 Pod

Service 解决：
  提供一个稳定的 ClusterIP（不变）
  自动负载均衡到后端多个 Pod
  通过 label selector 关联 Pod

类比：
  Pod = 员工（可能离职换人）
  Service = 工位电话（号码不变，谁坐这个位置谁接）


Pod (web-0, web-1, web-2)	3 个服务员
Pod IP (10.244.194.102 等)	服务员的手机号
Service (web-svc)	餐厅的柜台电话（一个号码）
Service IP (10.96.0.1)	柜台电话号码
你打柜台电话，前台自动转给空闲的服务员。你不关心是谁接的电话，只要能服务就行。

```

**Service 核心概念：**

```
Service        → 稳定的访问入口（ClusterIP + 端口）
Endpoints      → Service 后面真正的 Pod IP 列表
Label Selector → Service 通过标签找到对应的 Pod
```

**流量路径：**

```
客户端 → Service (ClusterIP:Port) → kube-proxy 转发 → 某个 Pod
```

### 🔨 实验 1：Service 基础（40 分钟）

```bash
# 1.1 先部署一个 3 副本 Deployment
cat <<EOF > /root/yaml/service/nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
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
        ports:
        - containerPort: 80
EOF


1. API 版本和资源类型
yaml
apiVersion: apps/v1
kind: Deployment
使用 Kubernetes 的 apps/v1 API

资源类型是 Deployment（不是 Pod、Service 等）

2. Deployment 的元数据
yaml
metadata:
  name: web
这个 Deployment 的名字叫 web

3. 期望状态（spec）
yaml
spec:
  replicas: 3           # 要运行 3 个 Pod 副本
  selector:             # Deployment 管理哪些 Pod？
    matchLabels:
      app: web          # 管理所有标签为 app=web 的 Pod
4. Pod 模板
yaml
  template:              # Pod 的"蓝图"
    metadata:
      labels:
        app: web         # Pod 的标签（让 selector 能匹配到）
    spec:
      containers:
      - name: nginx      # 容器名字
        image: nginx:alpine  # 使用的镜像
        ports:
        - containerPort: 80  # 容器内部监听 80 端口
创建后的效果
text
Deployment: web
     │
     │ 管理
     ▼
  3 个 Pod:
    web-xxxxx-yyyy1    IP: 10.244.194.102    运行 nginx:alpine
    web-xxxxx-yyyy2    IP: 10.244.194.103    运行 nginx:alpine
    web-xxxxx-yyyy3    IP: 10.244.194.104    运行 nginx:alpine



kubectl apply -f /root/yaml/service/nginx-deploy.yaml
kubectl get pods -l app=web -o wide
# 记下 3 个 Pod 的 IP



# 1.2 创建 Service
cat <<EOF > /root/yaml/service/web-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web          # 通过标签找 Pod
  ports:
  - port: 80          # Service 端口
    targetPort: 80    # Pod 端口
  type: ClusterIP     # 默认类型
EOF

这个 Service 给之前创建的 3 个 nginx Pod 提供了一个固定的内部入口（10.96.0.1:80），客户端访问这个地址，会自动负载均衡到 3 个 Pod 上。

kubectl apply -f /root/yaml/service/web-svc.yaml
kubectl get svc web-svc
# 记下 ClusterIP

CLUSTER-IP:  10.96.238.46

客户端 ──→ 记 Service IP ( 10.96.238.46)
                    ↓
                web-svc (固定入口)
                    ↓
         ┌──────────┼──────────┐
         ↓          ↓          ↓
      web-0      web-1      web-2
    (102:80)   (103:80)   (104:80)


# 1.3 验证 Service 负载均衡
# 多次 curl，观察返回的 Pod 不同
kubectl run test-pod --image=busybox --rm -it -- sh
# 进入后执行：
wget -qO- http://web-svc
wget -qO- http://web-svc
wget -qO- http://web-svc
exit

1. Service 的 DNS 解析

你没有写 IP，直接写 http://web-svc，能解析出来是因为 CoreDNS。在集群内部，Service 名字就是域名，格式是：

<service名>.<namespace>.svc.cluster.local


同一个 namespace 下简写成 web-svc 就行。跨 namespace 就要写 web-svc.default。

2. ClusterIP 的虚拟 IP

web-svc 的 CLUSTER-IP 是 10.96.238.46，这个 IP 是虚拟的，不在任何网卡上，kube-proxy 通过 iptables/ipvs 规则把发往这个 IP:80 的流量转发到后端 Pod。

3. 负载均衡

你连续 wget 了 3 次，如果 Deployment 是 replicas: 3，这 3 次请求大概率分发到了不同的 Pod（RR 或随机，取决于 kube-proxy 模式）。可以验证：

# 给每个 Pod 写一个不同的 index.html
kubectl get pods -l app=web -o name | while read pod; do
  kubectl exec $pod -- sh -c "echo '${pod##*/}' > /usr/share/nginx/html/index.html"
done

# 然后再进 test-pod curl 几次就能看到不同结果




意思是你 exit 退出了 shell 会话，--rm 自动把 Pod 删掉了。那句 resume using kubectl attach... 是 kubectl 的通用提示，但因为你加了 --rm，Pod 已经被删了，用不上。忽略就行。

小结

这一整套验证了：mirror 配置生效 + Deployment 正常 + Service ClusterIP 可达 + CoreDNS 解析正常 + kube-proxy 转发正常


# 1.4 查看 Endpoints
kubectl get endpoints web-svc
# 应该看到 3 个 Pod IP

# 1.5 删一个 Pod，观察 Endpoints 自动更新
kubectl delete pod $(kubectl get pods -l app=web -o name | head -1)
kubectl get endpoints web-svc
# 新 Pod 的 IP 自动加入
```

**✍️ 记录：**
- Service ClusterIP：__________
- Endpoints 列表：__________
- 删 Pod 后 Endpoints 变化：__________

---

## 📚 模块 2：Service 4 种类型（60 分钟）

### 理论（20 分钟）

```
ClusterIP（默认）：
  → 只在集群内部可访问
  → 分配一个虚拟 IP
  → 用途：内部服务间通信

NodePort：
  → 在每个节点上开一个端口（30000-32767）
  → 外部可以通过 节点IP:NodePort 访问
  → 用途：开发测试、简单暴露

LoadBalancer：
  → 在 NodePort 基础上，向云厂商申请一个外部负载均衡器
  → 用途：生产环境对外暴露（需要云环境）
  → 裸机环境用不了（除非装 MetalLB）

ExternalName：
  → 不代理流量，只做 DNS 别名
  → 把 Service 名解析到外部域名
  → 用途：访问集群外的服务（如外部数据库）
```

**对比表：**

```
类型           访问范围      端口          适用场景
─────────────────────────────────────────────────
ClusterIP     集群内部      任意          内部通信
NodePort      集群外部      30000-32767   开发测试
LoadBalancer  公网          自动分配      生产（云上）
ExternalName  DNS 别名      无            访问外部服务
```

**从外部访问 Pod 的 3 种方式：**

```
1. NodePort Service
2. Ingress（Day 4 学）
3. kubectl port-forward（调试用）
```

### 🔨 实验 2：ClusterIP vs NodePort（40 分钟）

```bash
# 2.1 当前 web-svc 是 ClusterIP，集群外访问不了
kubectl get svc web-svc
# TYPE: ClusterIP

# 在 Master1 上 curl ClusterIP（集群内可以）
curl http://$(kubectl get svc web-svc -o jsonpath='{.spec.clusterIP}')
# 能访问 ✅

# 2.2 创建 NodePort Service
cat <<EOF > /root/yaml/service/web-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080     # 指定端口（不指定会随机分配）
  type: NodePort
EOF

kubectl apply -f /root/yaml/service/web-nodeport.yaml
kubectl get svc web-nodeport
# TYPE: NodePort, PORT: 80:30080

# 2.3 从集群外访问（用任意节点 IP + 30080）
curl http://192.168.105.128:30080
curl http://192.168.105.131:30080
# 任何节点 IP 都能访问 ✅

# 2.4 对比两种 Service
kubectl get svc
# web-svc       ClusterIP   → 只能集群内访问
# web-nodeport  NodePort    → 集群外也能访问
```

**✍️ 记录：**
- ClusterIP 访问方式：__________
- NodePort 访问方式：__________
- NodePort 端口范围：__________

1. Service ClusterIP
kubectl get svc web-svc


看 CLUSTER-IP 列，填进去，比如：10.96.238.46

2. Endpoints 列表
kubectl get endpoints web-svc


会看到类似：

NAME      ENDPOINTS                                      AGE
web-svc   10.244.1.5:80,10.244.1.6:80,10.244.1.7:80     5m


把 ENDPOINTS 列的内容填进去。

3. 删 Pod 后 Endpoints 变化

先删一个 Pod：

kubectl get pods -l app=web
# 随便挑一个删
kubectl delete pod <任意一个pod名>


等几秒（Deployment 会自动拉起新 Pod），再看：

kubectl get endpoints web-svc


你会观察到：

被删的那个 Pod 的 IP 消失了

新拉起的 Pod 有了新的 IP，出现在 Endpoints 里

总数还是 3 个（因为 replicas=3）

填写示例：旧 Pod IP 10.244.1.5 消失，新 Pod IP 10.244.1.8 加入，总数保持 3 个



Service ClusterIP：10.96.238.46

Endpoints 列表：10.244.194.104:80, 10.244.194.73:80, 10.244.194.88:80

删 Pod 后 Endpoints 变化：旧 Pod IP 10.244.194.104 消失，新 Pod IP 10.244.194.68 加入，其余两个 10.244.194.73、10.244.194.88 不变，总数保持 3 个

Endpoints 的作用

简单说：Endpoints 就是 Service 背后的"真实地址簿"，告诉 kube-proxy 流量该转发到哪些 Pod。

流量链路
客户端请求
    ↓
Service（ClusterIP: 10.96.238.46:80）
    ↓
kube-proxy 查 Endpoints 表
    ↓
转发到真实 Pod
    ├── 10.244.194.68:80
    ├── 10.244.194.73:80
    └── 10.244.194.88:80
---

## ☕ 午休

---

## 📚 模块 3：kube-proxy 工作原理（45 分钟）

### 理论（15 分钟）

**kube-proxy 是什么？**

```
每个节点上运行的网络代理
负责实现 Service → Pod 的流量转发

它不是真正的"代理"（不经过它转发）
而是维护 iptables/IPVS 规则
```

**两种模式：**

```
iptables 模式（默认）：
  → 为每个 Service 创建 iptables 规则
  → 流量匹配 ClusterIP → DNAT 到某个 Pod IP
  → 随机选择后端 Pod
  → 优点：简单可靠
  → 缺点：Service 多了规则多，性能下降

IPVS 模式：
  → 用 Linux 内核的 IPVS 模块
  → 支持更多负载均衡算法（rr/lc/wlc...）
  → 优点：大规模集群性能好
  → 缺点：需要额外内核模块
```

**ClusterIP 的本质：**

```
ClusterIP 不是真实 IP（没有网卡绑定它）
它只存在于 iptables 规则中
当数据包目标是 ClusterIP 时，iptables 把目标改写为某个 Pod IP
这就是 DNAT（目标地址转换）
```

### 🔨 实验 3：查看 iptables 规则（30 分钟）

```bash
# 3.1 查看 kube-proxy 模式
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# mode: "" 表示 iptables（默认）

# 3.2 查看 Service 对应的 iptables 规则
# 先拿到 web-svc 的 ClusterIP
SVC_IP=$(kubectl get svc web-svc -o jsonpath='{.spec.clusterIP}')
echo $SVC_IP

# 查看 KUBE-SERVICES 链中关于这个 IP 的规则
iptables -t nat -L KUBE-SERVICES -n | grep $SVC_IP

# 3.3 追踪完整链路
# 找到对应的 KUBE-SVC-xxx 链
iptables -t nat -L -n | grep -A 5 "KUBE-SVC"
# 会看到 KUBE-SEP-xxx（每个 Pod 一条）

# 3.4 验证：Service 的 ClusterIP 不在任何网卡上
ip addr | grep $SVC_IP
# 没有输出 → 说明 ClusterIP 是虚拟的，只在 iptables 里
```

**✍️ 记录：**
- kube-proxy 模式：__iptables（mode: "" 即默认 iptables 模式）________
- ClusterIP 是否在网卡上：__不在（ip addr | grep 无输出），ClusterIP 是纯虚拟 IP，只存在于 iptables 规则中________
- iptables 规则的作用：_将发往 ClusterIP 的流量按概率均匀分发到后端 Pod_________

---

## 📚 模块 4：DNS 解析（45 分钟）

### 理论（15 分钟）

**CoreDNS 的作用：**

```
集群内的 DNS 服务器
让 Pod 可以通过"名字"访问 Service，不用记 IP
```

**DNS 解析规则：**

```
Service DNS 格式：
  <service-name>.<namespace>.svc.cluster.local

简写（同 namespace 内）：
  直接用 service-name 就行

例：
  web-svc                                    → 同 namespace
  web-svc.default                            → 指定 namespace
  web-svc.default.svc.cluster.local          → 完整 FQDN
```

**Pod DNS 格式（StatefulSet）：**

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

例：
  web-0.web-svc.default.svc.cluster.local
```

### 🔨 实验 4：DNS 解析验证（30 分钟）

```bash
# 4.1 进入一个测试 Pod
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh

# 4.2 在 Pod 内测试 DNS
# 短名（同 namespace）
nslookup web-svc
# 应该解析到 ClusterIP

# 带 namespace
nslookup web-svc.default

# 完整 FQDN
nslookup web-svc.default.svc.cluster.local

# 4.3 查看 Pod 的 DNS 配置
cat /etc/resolv.conf
# nameserver 10.96.0.10  ← CoreDNS 的 ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local

# 4.4 通过 DNS 名访问 Service
wget -qO- http://web-svc
# 能访问 ✅

exit

结果解析
nslookup 三种写法都解析到同一个 IP
web-svc                              → 10.96.238.46  ✓（短名，靠 search 域补全）
web-svc.default                      → 10.96.238.46  ✓（带 namespace）
web-svc.default.svc.cluster.local    → 10.96.238.46  ✓（完整 FQDN）

DNS 服务器
Server:    10.96.0.10（CoreDNS 的 ClusterIP）

/etc/resolv.conf 的含义
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

配置	作用
nameserver 10.96.0.10	DNS 请求发给 CoreDNS
search default.svc.cluster.local ...	短名自动补后缀尝试解析
ndots:5	名字里小于 5 个点就先走 search 补全

所以你写 web-svc，实际解析顺序是：

web-svc.default.svc.cluster.local → 命中，返回 ✓

wget 验证

DNS 解析到 ClusterIP → iptables 转发到后端 Pod → 返回 nginx 页面，整条链路畅通


```

**✍️ 记录：**
- Service DNS 名：__web-svc.default.svc.cluster.local________
- resolv.conf 内容：_
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5_________
- CoreDNS 的 IP：___10.96.0.10（即 kube-dns Service 的 ClusterIP）_______

---

## 📚 模块 5：Headless Service（30 分钟）

### 理论（10 分钟）

```
普通 Service：
  → 有 ClusterIP
  → DNS 解析到 ClusterIP（一个 IP）
  → kube-proxy 负载均衡到后端 Pod

Headless Service（ClusterIP: None）：
  → 没有 ClusterIP
  → DNS 直接解析到所有 Pod IP（多个 A 记录）
  → 客户端自己选择连哪个 Pod

用途：
  → StatefulSet 需要（给每个 Pod 固定 DNS）
  → 客户端需要知道所有后端 Pod IP 的场景
```

### 🔨 实验 5：对比普通 Service 和 Headless（20 分钟）

```bash
# 5.1 创建 Headless Service（指向同一组 Pod）
cat <<EOF > /root/yaml/service/web-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None       # 关键：None = Headless
  selector:
    app: web
  ports:
  - port: 80
EOF

kubectl apply -f /root/yaml/service/web-headless.yaml
kubectl get svc web-headless
# CLUSTER-IP 列显示 None

# 5.2 对比 DNS 解析结果
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh

# 普通 Service → 解析到 1 个 ClusterIP
nslookup web-svc
# Address: 10.96.x.x（一个 IP）

# Headless Service → 解析到所有 Pod IP
nslookup web-headless
# Address: 10.244.x.x
# Address: 10.244.x.x
# Address: 10.244.x.x（多个 IP = 每个 Pod 一个）

exit

# 清理所有资源
kubectl delete deployment web
kubectl delete svc web-svc web-nodeport web-headless
```

**✍️ 记录：**
- 普通 Service DNS 解析结果：__________
- Headless Service DNS 解析结果：__________
- Headless 的使用场景：__________

---

## ❓ 模块 6：面试题

### Q1: Service 4 种类型？

```
ClusterIP：集群内访问，默认类型
NodePort：节点 IP + 端口，集群外可访问
LoadBalancer：云上外部负载均衡器
ExternalName：DNS 别名，指向外部域名
```

### Q2: ClusterIP 是真实 IP 吗？怎么实现的？

```
不是真实 IP，没有网卡绑定它
只存在于 iptables 规则中
数据包目标是 ClusterIP 时，iptables 做 DNAT 改写为 Pod IP
```

### Q3: kube-proxy iptables 模式 vs IPVS 模式？

```
iptables：默认，简单可靠，Service 多了性能下降
IPVS：大规模集群用，支持更多负载均衡算法，性能好
```

### Q4: 从集群外访问 Pod 有几种方式？

```
1. NodePort Service（节点IP:30000-32767）
2. Ingress（域名路由，Day 4 学）
3. LoadBalancer（云上）
4. kubectl port-forward（调试用）
```

### Q5: Service 的 DNS 名格式？

```
<svc-name>.<namespace>.svc.cluster.local
同 namespace 内可以直接用 svc-name
```

### Q6: Headless Service 是什么？和普通 Service 区别？

```
Headless：ClusterIP=None
普通 Service DNS → 解析到 1 个 ClusterIP
Headless DNS → 直接解析到所有 Pod IP

用途：StatefulSet 固定 DNS、客户端需要知道所有后端
```

### Q7: Endpoints 是什么？

```
Service 后面真正的 Pod IP 列表
Pod 创建/删除时自动更新
kubectl get endpoints <svc-name> 可以查看
```

### Q8: Pod 之间怎么通信？

```
同节点 Pod：通过 Linux bridge 直接通信
跨节点 Pod：通过 CNI 插件（Calico）
  → Calico 用 BGP/IPIP 打通跨节点网络
  → 每个 Pod 有独立 IP，可以直接互访
```

### Q9: 为什么 Pod IP 不能直接用？要用 Service？

```
Pod IP 不固定（重建后变）
多副本时不知道连哪个
Service 提供：稳定 IP + 负载均衡 + DNS 名
```

### Q10: NodePort 的缺点？生产环境用什么替代？

```
缺点：
  - 端口范围有限（30000-32767）
  - 每个 Service 占一个端口
  - 没有域名路由能力
  - 安全性差（直接暴露节点端口）

生产替代：Ingress（支持域名路由、TLS、路径匹配）
```

---

## 🎯 速记版（考前 1 分钟）

```
4 种 Service：ClusterIP(内部) NodePort(节点端口) LB(云上) ExternalName(DNS别名)
ClusterIP 本质：iptables DNAT 规则，不是真实 IP
kube-proxy：维护 iptables/IPVS 规则，不是真正代理
DNS 格式：svc-name.namespace.svc.cluster.local
Headless：ClusterIP=None，DNS 直接解析到 Pod IP
Endpoints：Service 后面的 Pod IP 列表，自动更新
外部访问：NodePort / Ingress / LoadBalancer
```

---

## 📊 自测清单（晚上合上笔记做）

```
☐ 1. Service 4 种类型各自的访问范围？
☐ 2. ClusterIP 是怎么实现的？（iptables DNAT）
☐ 3. kube-proxy 两种模式的区别？
☐ 4. Service 的 DNS 名格式？
☐ 5. Headless Service 和普通 Service 的 DNS 解析区别？
☐ 6. 从集群外访问 Pod 有几种方式？
☐ 7. 为什么不能直接用 Pod IP？
☐ 8. Endpoints 是什么？什么时候更新？
☐ 9. NodePort 的缺点？
☐ 10. 画出 client → Service → Pod 的完整流量路径
```
