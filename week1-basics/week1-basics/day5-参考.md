# Day 5：配置管理与安全（精简版）

> 日期：2026-05-15
> 主题：ConfigMap、Secret、RBAC、ServiceAccount

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 4（15 分钟）
  ☐ 2. ConfigMap 概念 + 使用方式（实验 1）
  ☐ 3. Secret 概念 + 对比 ConfigMap（实验 2）

下午（3h）：
  ☐ 4. RBAC 权限模型（实验 3）
  ☐ 5. ServiceAccount（实验 4）
  ☐ 6. NetworkPolicy 网络策略（实验 5）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：ConfigMap（60 分钟）

### 理论（15 分钟）

**ConfigMap 是什么？**

```
把配置和镜像解耦
应用代码打在镜像里，配置放在 ConfigMap 里
不同环境（dev/test/prod）只需要换 ConfigMap，不用重新构建镜像

类比：
  镜像 = 毛坯房（不变）
  ConfigMap = 装修方案（可以随时换）
```

**3 种使用方式：**

```
1. 环境变量注入
   → Pod 启动时读取
   → 修改后需要重启 Pod

2. 挂载为文件
   → ConfigMap 的内容变成容器里的文件
   → 修改后自动更新（约 1 分钟）

3. 命令行参数
   → 用 $(ENV_NAME) 引用
```

### 🔨 实验 1：ConfigMap 全流程（45 分钟）

```bash
# 1.1 创建 ConfigMap（3 种方式）

# 方式 A：命令行创建
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080

# 方式 B：从文件创建
echo "server.port=8080" > /tmp/app.properties
kubectl create configmap file-config --from-file=/tmp/app.properties

# 方式 C：YAML 创建
cat <<EOF > /root/yaml/config/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  nginx.conf: |
    server {
        listen 80;
        location / {
            return 200 "Hello from ConfigMap";
        }
    }
EOF

mkdir -p /root/yaml/config
kubectl apply -f /root/yaml/config/configmap.yaml

# 查看
kubectl get configmap
kubectl describe configmap demo-config

# 1.2 使用方式 1：环境变量
cat <<EOF > /root/yaml/config/pod-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo APP_ENV=\$APP_ENV APP_PORT=\$APP_PORT; sleep 3600"]
    envFrom:
    - configMapRef:
        name: demo-config
EOF

kubectl apply -f /root/yaml/config/pod-env.yaml
sleep 5
kubectl logs env-pod
# 输出：APP_ENV=production APP_PORT=8080 ✅

# 1.3 使用方式 2：挂载为文件
cat <<EOF > /root/yaml/config/pod-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: config-vol
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-vol
    configMap:
      name: demo-config
      items:
      - key: nginx.conf
        path: default.conf
EOF

kubectl apply -f /root/yaml/config/pod-volume.yaml
sleep 10
kubectl exec volume-pod -- cat /etc/nginx/conf.d/default.conf
# 看到 nginx.conf 的内容 ✅

# 1.4 验证热更新（修改 ConfigMap）
kubectl edit configmap demo-config
# 把 "Hello from ConfigMap" 改成 "Hello Updated"
# 等 1-2 分钟
kubectl exec volume-pod -- cat /etc/nginx/conf.d/default.conf
# 文件已更新 ✅（但 nginx 需要 reload 才生效）

# 清理
kubectl delete pod env-pod volume-pod
kubectl delete configmap app-config file-config demo-config
```

**✍️ 记录：**
- 3 种创建方式：_--from-literal、--from-file、YAML 声明式_________
- 环境变量 vs 文件挂载区别：环境变量：重启 Pod 才生效
文件挂载：自动更新，但应用需重载__________
- 热更新：文件可以，环境变量不行

---

## 📚 模块 2：Secret（45 分钟）

### 理论（10 分钟）

**Secret vs ConfigMap：**

```
                ConfigMap          Secret
──────────────────────────────────────────
存储内容      普通配置             敏感信息（密码/证书/Token）
编码方式      明文                 Base64 编码
使用方式      环境变量/文件         环境变量/文件
大小限制      1MB                  1MB
etcd 存储     明文                 可配置加密（EncryptionConfig）
```

**⚠️ Secret 的 Base64 不是加密！**

```
Base64 只是编码，任何人都能解码
真正的安全需要：
  1. etcd 启用加密（EncryptionConfig）
  2. RBAC 限制谁能读 Secret
  3. 考虑 Vault 等外部密钥管理
```

**Secret 3 种类型：**

```
Opaque：      通用（用户自定义，最常用）
kubernetes.io/dockerconfigjson：镜像仓库认证
kubernetes.io/tls：TLS 证书
```

### 🔨 实验 2：Secret 使用（35 分钟）

```bash
# 2.1 创建 Secret
# 命令行方式
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=P@ssw0rd123

# 查看（注意是 base64 编码的）
kubectl get secret db-secret -o yaml
# data:
#   username: YWRtaW4=        ← base64(admin)
#   password: UEBzc3cwcmQxMjM= ← base64(P@ssw0rd123)

# 解码验证
echo "YWRtaW4=" | base64 -d
# admin

# 2.2 YAML 方式创建
cat <<EOF > /root/yaml/config/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_HOST: bXlzcWwuZGVmYXVsdC5zdmM=    # mysql.default.svc
  DB_PASS: UEBzc3cwcmQxMjM=             # P@ssw0rd123
EOF

kubectl apply -f /root/yaml/config/secret.yaml

# 2.3 Pod 使用 Secret（环境变量）
cat <<EOF > /root/yaml/config/pod-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo DB_HOST=\$DB_HOST DB_PASS=\$DB_PASS; sleep 3600"]
    envFrom:
    - secretRef:
        name: app-secret
EOF

kubectl apply -f /root/yaml/config/pod-secret.yaml
sleep 5
kubectl logs secret-pod
# DB_HOST=mysql.default.svc DB_PASS=P@ssw0rd123
# 注入到容器里的是解码后的明文 ✅

# 2.4 创建镜像仓库 Secret（了解）
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --dry-run=client -o yaml
# 看生成的 YAML 结构

# 清理
kubectl delete pod secret-pod
kubectl delete secret db-secret app-secret
```

**✍️ 记录：**
- Secret 和 ConfigMap 核心区别：__Secret 存敏感数据（密码、Token），ConfigMap 存普通配置（端口、环境变量）。Secret 有 Base64 编码（防肉眼直接看），ConfigMap 明文。________
- Base64 是加密吗：_ 不是，只是编码，任何人都能解码（base64 -d）_________
- imagePullSecrets 什么时候用：Pod 需要从私有镜像仓库（如 Harbor、私有 Docker Hub）拉取镜像时使用__________

---

## ☕ 午休

---

## 📚 模块 3：RBAC 权限控制（60 分钟）

### 理论（20 分钟）

**RBAC 4 个核心概念：**

```
Role：
  → 一组权限（能对什么资源做什么操作）
  → 命名空间级别

ClusterRole：
  → 和 Role 一样，但是集群级别
  → 跨命名空间或集群范围的资源

RoleBinding：
  → 把 Role 绑定给用户/组/ServiceAccount
  → 命名空间级别

ClusterRoleBinding：
  → 把 ClusterRole 绑定给用户/组/SA
  → 集群级别
```

**类比理解：**

```
Role = 工牌（能进哪些门）
RoleBinding = 把工牌发给某个人

ClusterRole = VIP 工牌（能进所有门）
ClusterRoleBinding = 把 VIP 工牌发给某个人
```

**权限三要素：**

```
apiGroups：资源所属的 API 组（""=核心组，apps，batch...）
resources：资源类型（pods，deployments，services...）
verbs：操作（get，list，watch，create，update，delete）
```

### 🔨 实验 3：RBAC 实操（40 分钟）

```bash
# 3.1 创建一个只能看 Pod 的 Role
cat <<EOF > /root/yaml/config/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-viewer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: pod-viewer
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
EOF

kubectl apply -f /root/yaml/config/rbac.yaml

# 3.2 测试权限
# 用 pod-viewer 身份查看 Pod（应该能看）
kubectl auth can-i get pods --as=system:serviceaccount:default:pod-viewer
# yes ✅

# 执行了一个 RBAC（基于角色的访问控制）配置，创建了三个资源，最终让 pod-viewer 这个服务账号（ServiceAccount）拥有了查看 Pod 的权限。

# 用 pod-viewer 身份删除 Pod（应该不能）
kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-viewer
# no ✅

# 用 pod-viewer 身份查看 Deployment（应该不能）
kubectl auth can-i get deployments --as=system:serviceaccount:default:pod-viewer
# no ✅

# 3.3 查看集群内置的 ClusterRole
kubectl get clusterrole | head -20
# admin / cluster-admin / edit / view 等
# ClusterRole 是集群级别的角色，可以跨所有命名空间生效

# 查看 cluster-admin 有什么权限
kubectl describe clusterrole cluster-admin
# 所有资源的所有操作 → 超级管理员
# 这个输出显示的是 cluster-admin 这个集群角色的权限定义，可以看到它拥有 最高级别的权限。

# 3.4 查看 view 角色权限
kubectl describe clusterrole view
# 只有 get/list/watch 权限

# 清理
kubectl delete -f /root/yaml/config/rbac.yaml
```

简单来说，就是给某个账号分配权限：

创建身份：pod-viewer（一个服务账号）

定义规则：pod-reader（只能 get/list/watch Pod，不能删也不能动别的）

绑定：把身份和规则绑一起

最终效果：pod-viewer 这个账号只能看 Pod，不能删 Pod，也不能看 Deployment。

后面看 cluster-admin 和 view 是为了对比：

cluster-admin：超级管理员，啥都能干

view：只读账号，跟你刚建的 pod-viewer 类似

**✍️ 记录：**
- Role vs ClusterRole 区别：

Role 只在单个命名空间内生效；

ClusterRole 在整个集群级别生效（可访问集群资源如 Node、CRD，或跨命名空间访问）。

- RBAC 三要素：

Subject（主体）：谁要操作（User、Group、ServiceAccount）
Role/ClusterRole（角色）：能做什么（定义操作权限，如 get pod）
RoleBinding/ClusterRoleBinding（绑定）：把主体和角色绑在一起


- 内置角色有哪些：

cluster-admin：集群超级管理员（最高权限）
admin：命名空间管理员（可管理命名空间内大多数资源及 RBAC）
edit：命名空间内可读写（修改资源，但不能改 RBAC 权限）
view：命名空间内只读（不能看 Secret）

---

## 📚 模块 4：ServiceAccount（30 分钟）

### 理论（10 分钟）

```
ServiceAccount = Pod 的"身份证"

每个 Pod 都有一个 ServiceAccount
默认使用 default（权限很小）
可以给 Pod 指定不同的 SA → 不同的权限

用途：
  - Pod 需要调用 K8s API（如 Operator）
  - 不同应用不同权限（最小权限原则）
```

### 🔨 实验 4：ServiceAccount 验证（20 分钟）

```bash
# 4.1 查看默认 SA
kubectl get serviceaccount
# default ← 每个 namespace 自动创建

# 4.2 Pod 使用自定义 SA
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
---
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /var/run/secrets/kubernetes.io/serviceaccount/token | head -c 50; echo; sleep 3600"]
EOF

sleep 10
kubectl logs sa-pod
# 能看到 Token 的前 50 个字符

# 4.3 查看 Pod 里自动挂载的 SA Token
kubectl exec sa-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt  namespace  token


每个 Pod 都有一个“身份证”（ServiceAccount），K8s 通过这个身份证上的 Token 来识别 Pod 的身份，并配合 RBAC 决定它能做什么（比如访问 API）。

4.1：K8s 会给每个命名空间自动创建一个默认的“身份证”（default ServiceAccount）。

4.2 & 4.3：你可以在 Pod 里指定使用一个自定义的“身份证”（my-app-sa），K8s 会自动把这张“身份证”（Token 文件）挂载进 Pod 里，Pod 里的程序就可以拿着它去跟 K8s API 证明“我是谁”了。


# 清理
kubectl delete pod sa-pod
kubectl delete sa my-app-sa
```

---

## 📚 模块 5：NetworkPolicy 网络策略（30 分钟）

### 理论（10 分钟）

```
NetworkPolicy = Pod 级别的防火墙

默认：所有 Pod 之间可以互相访问
加了 NetworkPolicy：只允许指定的流量

例：只允许 frontend 访问 backend，其他 Pod 不能访问
```

### 🔨 实验 5：NetworkPolicy 实操（20 分钟）

```bash
# 5.1 准备测试环境
kubectl run backend --image=nginx:alpine --labels="app=backend"
kubectl expose pod backend --port=80

kubectl run frontend --image=busybox --labels="app=frontend" -- sleep 3600
kubectl run hacker --image=busybox --labels="app=hacker" -- sleep 3600

# 默认：所有人都能访问 backend
kubectl exec frontend -- wget -qO- -T 3 http://backend
# 成功 ✅
kubectl exec hacker -- wget -qO- -T 3 http://backend
# 成功 ✅

# 5.2 创建 NetworkPolicy：只允许 frontend 访问 backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
EOF

# 5.3 测试
kubectl exec frontend -- wget -qO- -T 3 http://backend
# 成功 ✅（frontend 被允许）

kubectl exec hacker -- wget -qO- -T 3 http://backend
# 超时 ❌（hacker 被拒绝）

# 清理
kubectl delete pod frontend backend hacker
kubectl delete svc backend
kubectl delete networkpolicy backend-policy
```

**✍️ 记录：**
- NetworkPolicy 默认行为：_默认允许所有_________
- 配置后效果：__按规则放行，拒绝其他________

---

## ❓ 模块 6：面试题

### Q1: ConfigMap 和 Secret 区别？

```
ConfigMap：普通配置，明文存储
Secret：敏感信息，Base64 编码
两者使用方式相同：环境变量 / 文件挂载
Base64 不是加密，真正安全需要 etcd 加密 + RBAC
```

### Q2: ConfigMap 修改后 Pod 怎么生效？

```
环境变量方式：Pod 不会自动更新，需要重启
文件挂载方式：约 1 分钟自动更新文件
  但应用可能需要 reload（如 nginx reload）
  
最佳实践：修改 ConfigMap 后滚动重启
  kubectl rollout restart deployment/xxx
```

### Q3: RBAC 4 个概念？

```
Role：命名空间级权限
ClusterRole：集群级权限
RoleBinding：绑定 Role 给用户/SA
ClusterRoleBinding：绑定 ClusterRole 给用户/SA
```

### Q4: ServiceAccount 是什么？

```
Pod 的身份证，用于调用 K8s API
每个 Pod 默认用 default SA
可以自定义 SA 并配合 RBAC 控制权限
最小权限原则：只给需要的权限
```

### Q5: NetworkPolicy 怎么用？

```
Pod 级别的防火墙
默认全通，加 NetworkPolicy 后限制流量
通过 podSelector 选择目标 Pod
通过 ingress/egress 规则控制进出流量
需要 CNI 支持（Calico 支持 ✅）
```

### Q6: 怎么存储敏感信息？

```
基本方案：K8s Secret + RBAC 限制访问
进阶方案：
  - etcd 加密（EncryptionConfig）
  - 外部密钥管理（HashiCorp Vault）
  - Sealed Secrets（加密后存 Git）
  
生产建议：Vault + CSI Driver
```

---

## 🎯 速记版

```
ConfigMap：明文配置，环境变量/文件挂载，文件可热更新
Secret：Base64 编码（不是加密），3 种类型（Opaque/docker/tls）
RBAC：Role+Binding（命名空间）ClusterRole+Binding（集群）
ServiceAccount：Pod 身份证，配合 RBAC 控制 API 访问
NetworkPolicy：Pod 防火墙，默认全通，加策略后限制
```

---

## 📊 自测清单

```
☐ 1. ConfigMap 3 种创建方式？
☐ 2. ConfigMap 环境变量和文件挂载的热更新区别？
☐ 3. Secret 的 Base64 是加密吗？
☐ 4. RBAC 的 4 个核心概念？
☐ 5. Role 和 ClusterRole 区别？
☐ 6. ServiceAccount 的作用？
☐ 7. NetworkPolicy 默认行为？
☐ 8. 生产环境怎么存储密码？
☐ 9. kubectl auth can-i 怎么用？
☐ 10. imagePullSecrets 什么时候需要？
```
