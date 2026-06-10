# Day 14：综合实战（精简版）

> 日期：2026-05-24
> 主题：综合项目部署、复盘总结、面试准备

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 综合实战项目：部署一套完整应用（实验 1）
  ☐ 2. 包含：Deployment + Service + Ingress + PVC + ConfigMap

下午（3h）：
  ☐ 3. 给应用加监控（实验 2）
  ☐ 4. 给应用加日志查询（实验 3）
  ☐ 5. 故障演练 + 恢复（实验 4）

晚上（1h）：
  ☐ 6. 14 天复盘总结
  ☐ 7. 最终 push + 整理 GitHub 仓库
  ☐ 8. 制定后续计划
```

---

## 📚 模块 1：综合实战项目（3 小时）

### 目标

```
部署一个"生产级"应用，覆盖 14 天所有知识点：

✅ Deployment（多副本）
✅ Service（ClusterIP）
✅ Ingress（域名访问）
✅ ConfigMap（配置注入）
✅ Secret（敏感信息）
✅ PVC（数据持久化）
✅ 健康检查（liveness + readiness）
✅ 资源限制（requests + limits）
✅ HPA（自动扩缩容，可选）
```

### 🔨 实验 1：部署 Nginx + Redis 应用

```bash
# 创建 namespace
kubectl create namespace demo

# 1.1 ConfigMap（Nginx 自定义配置）
cat <<'EOF' > /root/yaml/demo/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: demo
data:
  default.conf: |
    server {
        listen 80;
        server_name _;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            return 200 "ok";
        }
    }
  index.html: |
    <h1>K8s Learning Demo</h1>
    <p>Deployed by: Your Name</p>
    <p>Date: 2026-05-24</p>
EOF

# 1.2 Secret（模拟数据库密码）
cat <<'EOF' > /root/yaml/demo/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
data:
  redis-password: cGFzc3dvcmQxMjM=    # password123 的 base64
EOF

# 1.3 PVC（Redis 数据持久化）
cat <<'EOF' > /root/yaml/demo/pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/redis-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: demo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# 1.4 Redis Deployment
cat <<'EOF' > /root/yaml/demo/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: 64Mi
            cpu: 50m
          limits:
            memory: 128Mi
            cpu: 100m
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 3
          periodSeconds: 5
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: demo
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
EOF

# 1.5 Nginx Deployment（前端）
cat <<'EOF' > /root/yaml/demo/nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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
        resources:
          requests:
            memory: 64Mi
            cpu: 50m
          limits:
            memory: 128Mi
            cpu: 100m
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        - name: nginx-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF

# 1.6 Ingress
cat <<'EOF' > /root/yaml/demo/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
  - host: demo.test.com
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

# 1.7 一键部署
mkdir -p /root/yaml/demo
kubectl apply -f /root/yaml/demo/

# 1.8 验证
kubectl get all -n demo
kubectl get pvc -n demo
kubectl get ingress -n demo

# 测试访问
curl -H "Host: demo.test.com" http://192.168.105.128:30080
# 应该看到 "K8s Learning Demo" ✅

# 测试 Redis
kubectl exec -n demo $(kubectl get pods -n demo -l app=redis -o name) -- redis-cli ping
# PONG ✅
```

**✍️ 记录：**
- 部署了哪些资源：__________
- curl 返回的内容：__________

---

## 📚 模块 2：给应用加监控（30 分钟）

### 🔨 实验 2：在 Grafana 看应用指标

```bash
# 2.1 如果有 Prometheus，它已经在自动采集 demo namespace 的 Pod 指标

# 在 Grafana 中查询：
# Pod CPU 使用：
# sum(rate(container_cpu_usage_seconds_total{namespace="demo"}[5m])) by (pod)

# Pod 内存使用：
# container_memory_working_set_bytes{namespace="demo"}

# Nginx QPS（如果有流量）：
# rate(nginx_http_requests_total[5m])

# 2.2 模拟流量
kubectl run load-test --image=busybox --rm -it -- sh -c "while true; do wget -qO- http://nginx-svc.demo.svc; sleep 0.5; done"
# Ctrl+C 停止
```

---

## 📚 模块 3：给应用查日志（20 分钟）

### 🔨 实验 3：在 Grafana Loki 查日志

```
Grafana → Explore → Loki

查询：
  {namespace="demo", app="nginx"}
  {namespace="demo", app="redis"}
  {namespace="demo"} |= "error"
```

---

## 📚 模块 4：故障演练（60 分钟）

### 🔨 实验 4：模拟故障 + 恢复

```bash
# 4.1 删一个 Pod，观察自愈
kubectl delete pod -n demo $(kubectl get pods -n demo -l app=nginx -o name | head -1)
kubectl get pods -n demo -w
# 自动重建 ✅

# 4.2 模拟滚动更新
kubectl set image deployment/nginx -n demo nginx=nginx:1.21
kubectl rollout status deployment/nginx -n demo
# 逐个替换 ✅

# 4.3 回滚
kubectl rollout undo deployment/nginx -n demo
kubectl describe deployment nginx -n demo | grep Image
# 回到 alpine ✅

# 4.4 扩容应对"流量高峰"
kubectl scale deployment nginx -n demo --replicas=5
kubectl get pods -n demo -l app=nginx
# 5 个 Pod ✅

# 4.5 缩容
kubectl scale deployment nginx -n demo --replicas=3

# 4.6 验证 PVC 持久化
# 写数据到 Redis
kubectl exec -n demo $(kubectl get pods -n demo -l app=redis -o name) -- redis-cli set test-key "hello from day14"

# 删掉 Redis Pod
kubectl delete pod -n demo -l app=redis

# 等重建
sleep 15

# 数据还在吗？
kubectl exec -n demo $(kubectl get pods -n demo -l app=redis -o name) -- redis-cli get test-key
# "hello from day14" ✅ 数据持久化成功
```

---

## 📚 模块 5：14 天复盘总结

### 知识地图

```
Week 1（基础）：
  Day 1: 架构组件 → 能画集群架构图
  Day 2: Pod 工作负载 → 5 种控制器
  Day 3: Service 网络 → 4 种 Service + DNS
  Day 4: Ingress 存储 → 7 层路由 + PV/PVC
  Day 5: 配置安全 → ConfigMap/Secret/RBAC
  Day 6: 调度资源 → nodeSelector/affinity/taint
  Day 7: 复盘实战 → Week 1 综合

Week 2（进阶）：
  Day 8: 监控 → Prometheus + Grafana
  Day 9: 日志 → Loki + Promtail
  Day 10: CI/CD → Helm + ArgoCD + GitOps
  Day 11: 高可用 → etcd 备份 + 故障演练
  Day 12: 故障排查 → 万能三板斧
  Day 13: 面试题 → 分模块 + 场景题
  Day 14: 综合实战 → 全流程部署
```

### 能力自评

```
☐ 能从零搭建 K8s 高可用集群
☐ 能部署和管理各种工作负载
☐ 能配置网络（Service/Ingress）
☐ 能做持久化存储
☐ 能搭建监控和日志系统
☐ 能做 CI/CD 流程
☐ 能做 etcd 备份恢复
☐ 能排查常见故障
☐ 能回答 80% 的面试题
```

---

## 📚 模块 6：后续学习方向

```
如果面试 K8s 运维/SRE：
  → 深入 Prometheus + AlertManager
  → 学习 Terraform（IaC）
  → 学习 Service Mesh（Istio）

如果面试 K8s 开发：
  → 学习 Operator 开发
  → 学习 CRD + Controller
  → 深入 client-go

如果面试 DevOps：
  → 深入 CI/CD（Jenkins / GitLab CI）
  → 学习 ArgoCD + Flux
  → 学习云原生安全
```

---

## 📚 模块 7：清理 + 整理 GitHub

```bash
# 清理 demo 项目
kubectl delete namespace demo
kubectl delete pv redis-pv

# 整理 GitHub 仓库
cd /d/code/k8s-learning-notes

# 更新 README
notepad README.md
# 加上"14 天学习已完成"

# 更新 PROGRESS.md
notepad PROGRESS.md
# 标记 Day 14 完成

# 最终推送
git add .
git commit -m "docs: Day14 综合实战 + 14天学习完成"
git push
```

---

## 🎯 最终速记（面试前 5 分钟过一遍）

```
架构：3 Master + VIP + haproxy，6 组件各司其职
Pod：5 状态，3 探针，5 种工作负载
网络：4 种 Service，ClusterIP 是 iptables DNAT
存储：PV/PVC 绑定，StatefulSet + volumeClaimTemplates
监控：Prometheus Pull + Grafana 可视化
日志：DaemonSet(Promtail) + Loki + Grafana
CI/CD：Helm 管理 + ArgoCD GitOps
高可用：etcd 备份 + 多 Master + VIP 飘移
排查：describe → logs → logs --previous
```

---

## 📊 最终自测（终极版）

```
☐ 画 K8s 架构图（5 分钟内）
☐ 讲 kubectl apply 完整流程（3 分钟）
☐ 讲集群高可用设计（3 分钟）
☐ 排查一个 Pod 不正常的场景（3 分钟）
☐ 讲你的 K8s 项目经验（5 分钟）
☐ 现场写一个 Deployment YAML（5 分钟）
☐ 讲 etcd 备份恢复流程（3 分钟）
☐ 讲 Service 到 Pod 的流量路径（3 分钟）
☐ 讲 GitOps 流程（3 分钟）
☐ 讲监控告警方案（3 分钟）
```

**全部能讲清楚 = 面试 80% 没问题 ✅**
