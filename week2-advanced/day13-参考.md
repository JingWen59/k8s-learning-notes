# Day 13：面试题总结（精简版）

> 日期：2026-05-23
> 主题：高频面试题汇总、模拟面试、查漏补缺

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 按模块过一遍所有面试题（自测）
  ☐ 2. 标记"卡壳"的题目
  ☐ 3. 重点突破卡壳的知识点

下午（3h）：
  ☐ 4. 模拟面试（对着镜子/录音讲）
  ☐ 5. 项目经验包装
  ☐ 6. 场景题练习

晚上（1h）：
  ☐ 7. 再过一遍卡壳的题
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：高频面试题速查（按模块分类）

### 🏗️ 架构组件（Day 1）

```
Q: K8s 核心组件？
A: Master 4 个（apiserver/etcd/scheduler/controller）
   Node 3 个（kubelet/kube-proxy/containerd）

Q: kubectl apply 完整流程？
A: kubectl → apiserver → etcd → controller 创建 Pod
   → scheduler 分配节点 → kubelet 起容器

Q: 集群高可用怎么做？
A: 3 Master + keepalived(VIP) + haproxy(负载均衡)
   VIP 飘移 3-5 秒，etcd 多数派
```

### 🐳 Pod 与工作负载（Day 2）

```
Q: Pod 5 种状态？
A: Pending / Running / Succeeded / Failed / Unknown

Q: 3 种探针？
A: liveness(杀重启) readiness(摘流量) startup(保护期)

Q: Deployment vs StatefulSet？
A: Deploy：名字随机，共享存储，无状态
   STS：名字固定(xxx-0)，独立 PVC，有状态

Q: CrashLoopBackOff 排查？
A: describe → logs --previous → 看 Exit Code
```

### 🌐 网络（Day 3）

```
Q: Service 4 种类型？
A: ClusterIP(内部) NodePort(节点端口) LB(云上) ExternalName(DNS别名)

Q: ClusterIP 怎么实现？
A: 不是真实 IP，iptables DNAT 规则转发到 Pod IP

Q: Headless Service？
A: ClusterIP=None，DNS 直接解析到所有 Pod IP
   用途：StatefulSet 固定 DNS
```

### 🚪 Ingress + 存储（Day 4）

```
Q: Ingress 和 Service 区别？
A: Service 4 层(TCP/端口)，Ingress 7 层(HTTP/域名/路径)

Q: PV 和 PVC 关系？
A: PV=实际存储(管理员)，PVC=申请(用户)，自动匹配绑定

Q: 动态供给？
A: StorageClass → PVC 创建时自动创建 PV
```

### ⚙️ 配置与安全（Day 5-6）

```
Q: ConfigMap 和 Secret 区别？
A: ConfigMap 明文，Secret Base64 编码
   都可以挂载为文件或环境变量

Q: RBAC 4 个概念？
A: Role(命名空间权限) ClusterRole(集群权限)
   RoleBinding(绑定) ClusterRoleBinding(集群绑定)

Q: 资源限制？requests vs limits？
A: requests：调度依据（最少需要多少）
   limits：上限（最多用多少，超了被 kill）

Q: Pod 调度策略？
A: nodeSelector / nodeAffinity / taint+toleration / podAffinity
```

### 📊 监控（Day 8）

```
Q: Prometheus 架构？
A: Server(拉取+存储) + Exporters(暴露指标) + Alertmanager(告警) + Grafana(可视化)

Q: Pull vs Push？
A: Pull(主动拉，默认) Push(被动推，Pushgateway，短期任务)

Q: 4 种指标类型？
A: Counter(只增) Gauge(可变) Histogram(分桶) Summary(分位)

Q: 查 CPU 使用率？
A: 100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100
```

### 📝 日志（Day 9）

```
Q: 日志收集方案？
A: DaemonSet(最常用) / Sidecar / 应用直接推送

Q: EFK vs PLG？
A: EFK(ES全文索引,重) PLG(Loki标签索引,轻)
   小规模 → PLG，大规模 → EFK
```

### 🚀 CI/CD（Day 10）

```
Q: GitOps 是什么？
A: Git=唯一真相源，Agent(ArgoCD)在集群内监听 Git 并自动同步

Q: 和传统 CI/CD 区别？
A: 传统：CI push 到集群（有凭证风险）
   GitOps：Agent 从 Git pull（安全，有漂移检测）

Q: Helm 解决什么？
A: 打包多个 YAML / values 管理多环境 / 一键升级回滚
```

### 🛡️ 高可用（Day 11）

```
Q: etcd 备份怎么做？
A: etcdctl snapshot save，每天至少 1 次 + 异地存储

Q: Master 挂 1 台会怎样？
A: VIP 飘移 + etcd 多数派正常 + leader 重选 → 集群正常

Q: 集群升级流程？
A: Master 逐台升级 → Worker drain → 升级 → uncordon
```

### 🔧 故障排查（Day 12）

```
Q: Pod Pending？
A: describe → 资源不足 / taint / PVC

Q: Service 不通？
A: 检查 Endpoints → selector → kube-proxy → DNS

Q: Exit 137？
A: OOMKilled（128+9），内存超限
```

---

## 📚 模块 2：场景题练习（60 分钟）

### 场景 1："线上服务突然 503"

```
你怎么排查：

1. kubectl get pods -l app=xxx
   → 看 Pod 是否正常

2. kubectl get endpoints <svc>
   → Endpoints 有没有 IP

3. kubectl logs <pod>
   → 应用是否报错

4. kubectl describe pod
   → 是否在重启 / OOM

5. kubectl top pods
   → 是否资源打满

回答模板：
"先看 Pod 状态，再看 Service Endpoints，
然后看日志定位原因。
如果 Pod 在 CrashLoop，看 --previous 日志。
如果 OOM，调大 limits。
如果流量过大，先扩副本数应急。"
```

### 场景 2："部署更新后客户有报错"

```
你怎么处理：

1. 先回滚
   kubectl rollout undo deployment/xxx
   → 止血

2. 查看更新了什么
   kubectl rollout history deployment/xxx
   → 找到这次改了什么

3. 看新版 Pod 日志
   kubectl logs <新Pod> --previous
   → 为什么崩了

4. 修复后重新发布

回答模板：
"第一步止血：立即回滚。
第二步定位：看更新历史和新 Pod 日志。
第三步修复：修好代码/配置后重新发版。
第四步复盘：为什么测试没发现？加强灰度。"
```

### 场景 3："集群某个节点 NotReady"

```
你怎么排查：

1. kubectl describe node <node>
   → 看 Conditions 和 Events

2. SSH 到节点
   systemctl status kubelet
   journalctl -u kubelet

3. 常见原因 + 修复
   - kubelet 挂了 → restart
   - containerd 挂了 → restart
   - 磁盘满了 → 清理
   - 证书过期 → renew

4. 节点上的 Pod 怎么办？
   → 5 分钟后 K8s 自动迁移 Pod 到其他节点
```

### 场景 4："需要给集群做灾备方案"

```
你怎么设计：

1. etcd 备份
   → crontab 每天备份 + 保留 7 天 + 异地存储

2. 集群高可用
   → 3 Master + VIP + haproxy

3. 应用层高可用
   → Deployment replicas ≥ 2
   → 配置 PDB（Pod Disruption Budget）
   → 跨节点分散（podAntiAffinity）

4. 监控告警
   → Prometheus + AlertManager
   → 节点 / Pod / etcd 健康检查

5. 演练
   → 定期故障演练（关 Master / 关 Worker）
```

---

## 📚 模块 3：项目经验包装（60 分钟）

### 你的项目描述模板

```
项目名称：K8s 高可用集群搭建与运维

环境规模：
  3 Master + 1 Worker（可说"N 个 Worker"）
  K8s v1.30

我做了什么：
  1. 从零搭建高可用集群
     - 3 Master + keepalived + haproxy
     - Calico 网络方案
     
  2. 部署监控体系
     - Prometheus + Grafana 指标监控
     - Loki + Promtail 日志收集
     
  3. CI/CD 流程
     - Helm 管理应用部署
     - GitOps（ArgoCD）自动同步
     
  4. 运维保障
     - etcd 定时备份
     - 故障演练（Master 宕机验证）
     - 证书管理和集群升级
```

### 面试提问你的问法

```
Q: "说说你做过的 K8s 项目"

回答模板（STAR 法则）：

Situation（背景）：
"我们需要容器化部署微服务，搭建 K8s 集群..."

Task（任务）：
"我负责集群的搭建、监控和运维..."

Action（行动）：
"我用 kubeadm 搭建了 3 Master 高可用集群，
配置了 keepalived + haproxy 保证入口高可用，
用 Prometheus + Grafana 搭建了监控体系..."

Result（结果）：
"集群稳定运行 X 个月，做过 Master 宕机演练，
VIP 3-5 秒切换，业务无感知..."
```

---

## 📚 模块 4：模拟面试（60 分钟）

### 用法

**录音或对着镜子，每题 3 分钟**

```
第 1 轮：基础题（10 题，每题 2 分钟）
  1. K8s 核心组件有哪些？
  2. Pod 有几种状态？
  3. Deployment 和 StatefulSet 区别？
  4. Service 4 种类型？
  5. Ingress 和 Service 区别？
  6. PV 和 PVC 关系？
  7. 三种探针区别？
  8. ConfigMap 和 Secret 区别？
  9. Prometheus 架构？
  10. 日志收集方案？

第 2 轮：进阶题（5 题，每题 3 分钟）
  1. kubectl apply 完整流程？
  2. Service ClusterIP 怎么实现？
  3. 集群高可用怎么设计？
  4. etcd 备份恢复流程？
  5. GitOps 和传统 CI/CD 区别？

第 3 轮：场景题（3 题，每题 5 分钟）
  1. 线上 Pod 反复重启怎么排查？
  2. 部署更新后有报错怎么处理？
  3. 怎么设计一个灾备方案？
```

---

## 📚 模块 5：查漏补缺清单

### 自测结果记录

```
✅ = 流利回答（不用管了）
⚠️ = 能答但不流利（复习一遍）
❌ = 答不出来（重点突破）

架构组件：    ☐✅ ☐⚠️ ☐❌
Pod工作负载：  ☐✅ ☐⚠️ ☐❌
网络Service： ☐✅ ☐⚠️ ☐❌
Ingress存储： ☐✅ ☐⚠️ ☐❌
配置安全：    ☐✅ ☐⚠️ ☐❌
监控：       ☐✅ ☐⚠️ ☐❌
日志：       ☐✅ ☐⚠️ ☐❌
CI/CD：      ☐✅ ☐⚠️ ☐❌
高可用：     ☐✅ ☐⚠️ ☐❌
故障排查：    ☐✅ ☐⚠️ ☐❌
```

---

## 🎯 面试回答原则

```
1. 先说结论，再说细节
   ❌ "首先...然后...最后..."（太长）
   ✅ "有 4 种类型：ClusterIP、NodePort..."（先给答案）

2. 结合实际经验
   ❌ "理论上说..."
   ✅ "我在项目中用了..."

3. 不会的不要编
   ✅ "这个我了解不深，但我知道..."
   ✅ "我没用过，但原理是..."

4. 反问面试官
   "您这边用的是什么网络方案？"
   "您这边日志收集用的 EFK 还是 Loki？"
```

---

## 📊 今日自测清单

```
☐ 基础 10 题能否流利回答？
☐ 进阶 5 题能否讲清楚？
☐ 场景 3 题能否给出完整方案？
☐ "说说你的 K8s 项目"能否讲 3 分钟？
☐ 所有"卡壳"的题已补强？
```
