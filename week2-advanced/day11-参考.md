# Day 11：高可用与灾备（精简版）

> 日期：2026-05-21
> 主题：集群高可用、etcd 备份恢复、故障演练、集群升级

---

## ✅ 今日学习清单

```
上午（2h）：
  ☐ 1. 复习 Day 10（15 分钟）
  ☐ 2. K8s 高可用架构回顾
  ☐ 3. etcd 集群原理 + 备份恢复（实验 1）

下午（4h）：
  ☐ 4. 故障演练：Master 宕机（实验 2）
  ☐ 5. 故障演练：etcd 数据恢复（实验 3）
  ☐ 6. 证书管理 + 集群升级（理论）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：K8s 高可用架构回顾（30 分钟）

### 理论（30 分钟）

**你的集群高可用架构：**

```
                    VIP: 192.168.105.200
                    (keepalived 管理)
                          ↓
                    haproxy:8443
                    (负载均衡)
                    ↓     ↓     ↓
              Master1  Master2  Master3
              (.128)   (.129)   (.130)
              
              每台 Master 上运行：
              - apiserver
              - etcd
              - controller-manager
              - scheduler
              - kubelet
              - kube-proxy
```

**高可用的 4 个层次：**

```
1. 网络层：keepalived + VIP
   → Master 挂了 VIP 自动飘移

2. API 层：haproxy 负载均衡
   → 请求分散到 3 台 apiserver

3. 数据层：etcd 3 节点集群
   → Raft 协议，挂 1 台数据不丢

4. 控制层：leader election
   → controller-manager / scheduler 自动选主
```

**能容忍的故障：**

```
挂 1 台 Master → 集群正常 ✅
挂 2 台 Master → etcd 无法形成多数派 → 集群冻结 ❌
挂全部 Worker  → 控制面正常，但没有节点跑 Pod
```

---

## 📚 模块 2：etcd 备份与恢复（90 分钟）

### 理论（20 分钟）

**为什么 etcd 备份最重要？**

```
etcd 里存了什么：
  - 所有 Pod / Deployment / Service 定义
  - 所有 ConfigMap / Secret
  - 所有 RBAC 配置
  - 集群状态和元数据

etcd 丢了 = 集群从零开始
所以 etcd 备份 = K8s 的"保命操作"
```

**备份方式：**

```
etcdctl snapshot save → 生成一个快照文件
这个文件包含 etcd 所有数据
定时备份 + 异地存储 = 灾备方案
```

**恢复场景：**

```
场景 1：误删资源（kubectl delete 删错了）
  → 从备份恢复 etcd → 资源回来了

场景 2：etcd 数据损坏
  → 用备份重建 etcd

场景 3：整个集群崩溃
  → 用备份 + kubeadm 重建
```

### 🔨 实验 1：etcd 备份与恢复（70 分钟）

#### 1.1 查看 etcd 集群状态

```bash
# 设置环境变量（方便后续使用）
export ETCDCTL_API=3
export ETCD_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCD_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCD_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCD_ENDPOINTS=https://127.0.0.1:2379

# 查看集群成员
etcdctl member list \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  -w table

# 查看集群健康状态
etcdctl endpoint health \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# 查看集群状态（谁是 leader）
etcdctl endpoint status \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  -w table
```

**✍️ 记录：**
- etcd 有几个成员：3

- 谁是 leader：k8s-master1（192.168.105.128）

---

#### 1.2 执行备份

```bash
# 创建备份目录
mkdir -p /root/etcd-backup

# 执行快照备份
etcdctl snapshot save /root/etcd-backup/etcd-$(date +%Y%m%d%H%M).db \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# 验证备份文件
etcdctl snapshot status /root/etcd-backup/etcd-*.db \
  --write-out=table

# 查看文件大小
ls -lh /root/etcd-backup/
```

**✍️ 记录：**
- 备份文件大小：__________
- 备份包含的 revision：__________


备份文件大小：23 MB（24 MB）

备份包含的 revision：500501

备份包含的 key 总数：1646

备份文件路径：/root/etcd-backup/etcd-202605180235.db

备份 HASH：7de35485

备份时间：2026-05-18 02:35

---

#### 1.3 模拟灾难 + 恢复

```bash
# 1.3.1 先创建一个测试资源（用于验证恢复）
kubectl create deployment backup-test --image=nginx:alpine --replicas=2
kubectl get deployment backup-test
# 记住：backup-test 存在 ✅

# 1.3.2 执行一次新的备份（包含 backup-test）
etcdctl snapshot save /root/etcd-backup/etcd-with-test.db \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# 1.3.3 模拟灾难：删掉 backup-test
kubectl delete deployment backup-test
kubectl get deployment backup-test
# 已经没了 ❌

# 1.3.4 从备份恢复（⚠️ 这会重置 etcd，请确认你在实验环境）

# 停 apiserver 和 etcd
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 等 10 秒让 Pod 停掉
sleep 10
crictl ps | grep -E "etcd|apiserver"
# 应该没有了

# 备份旧数据
mv /var/lib/etcd /var/lib/etcd.bak

# 从快照恢复
etcdctl snapshot restore /root/etcd-backup/etcd-with-test.db \
  --data-dir=/var/lib/etcd

# 恢复 manifests
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 等 1-2 分钟让组件起来
sleep 60

# 1.3.5 验证恢复
kubectl get deployment backup-test
# backup-test 回来了 ✅
kubectl get pods -l app=backup-test
# Pod 也在 ✅

# 清理
kubectl delete deployment backup-test
```

**⚠️ 注意：这个恢复操作在生产环境要非常谨慎！**

```
生产环境恢复步骤（多 Master）：
1. 所有 Master 停 etcd
2. 每台从同一个备份恢复
3. 恢复后修正 etcd 集群配置
4. 逐台启动
```

**✍️ 记录：**
- 恢复后 backup-test 是否存在：__________
- 恢复过程中集群是否可用：__________


恢复后 backup-test 是否存在：存在 ✅（2 个 Pod 都是 Running 状态）

恢复过程中集群是否可用：不可用 ❌（恢复期间 kubectl delete deployment backup-test 报错 Unable to connect to the server: EOF，因为 etcd 和 apiserver 都被停止了）

---

#### 1.4 写一个定时备份脚本

```bash
# 创建备份脚本
cat > /root/etcd-backup/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR=/root/etcd-backup
ETCDCTL_API=3
DATE=$(date +%Y%m%d%H%M)

etcdctl snapshot save ${BACKUP_DIR}/etcd-${DATE}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 只保留最近 7 天的备份
find ${BACKUP_DIR} -name "etcd-*.db" -mtime +7 -delete

echo "Backup completed: etcd-${DATE}.db"
EOF

chmod +x /root/etcd-backup/backup.sh

# 加入 crontab（每天凌晨 2 点备份）
echo "0 2 * * * /root/etcd-backup/backup.sh >> /var/log/etcd-backup.log 2>&1" | crontab -

# 验证 crontab
crontab -l

# 手动测试一次
/root/etcd-backup/backup.sh
ls -lh /root/etcd-backup/
```

---

## ☕ 午休

---

## 📚 模块 3：故障演练（60 分钟）

### 🔨 实验 2：Master 宕机演练

```bash
# 2.1 确认当前状态
kubectl get nodes
# 3 Master + 1 Worker 全部 Ready

# 确认 VIP 在哪
ip addr | grep 192.168.105.200
# 假设在 Master1

# 2.2 模拟 Master1 宕机
# 方法：停止 kubelet（不用真关机，方便恢复）
systemctl stop kubelet

# 2.3 在 Master2 上观察（SSH 到 Master2）
ssh root@192.168.105.129

# VIP 是否飘过来了？
ip addr | grep 192.168.105.200
# 应该能看到 VIP ✅

# kubectl 是否还能用？
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes
# Master1 变成 NotReady，其他正常

kubectl get pods -A | head -20
# 大部分 Pod 正常运行

# 2.4 恢复 Master1（回到 Master1）
# 在 Master1 上
systemctl start kubelet

# 等 1 分钟
sleep 60

kubectl get nodes
# Master1 重新变成 Ready

# VIP 是否飘回来？
ip addr | grep 192.168.105.200
# 如果 keepalived 配置了 preempt，VIP 会飘回 Master1
```

**✍️ 记录：**
- Master1 停止后多久 VIP 飘移：__________
- 飘移后 kubectl 是否可用：__________
- Master1 恢复后 VIP 是否回来：__________

---

### 🔨 实验 3：Worker 节点故障演练

```bash
# 3.1 在 Worker 上跑一些 Pod
kubectl create deployment ha-test --image=nginx:alpine --replicas=3
kubectl get pods -o wide -l app=ha-test
# 看有没有 Pod 在 Worker1 上

# 3.2 模拟 Worker1 故障
ssh root@192.168.105.131 "systemctl stop kubelet"

# 3.3 观察 Pod 迁移（需要等 5 分钟）
# K8s 默认 5 分钟后才判定节点不可达
kubectl get nodes -w
# Worker1 变成 NotReady

# 等 5 分钟后
kubectl get pods -o wide -l app=ha-test
# 原来在 Worker1 上的 Pod 会被重新调度到其他节点

# 3.4 恢复 Worker1
ssh root@192.168.105.131 "systemctl start kubelet"

# 等 1 分钟
kubectl get nodes
# Worker1 恢复 Ready

# 清理
kubectl delete deployment ha-test
```

**✍️ 记录：**
- Worker 故障后 Pod 多久迁移：__________
- Pod 迁移到了哪个节点：__________



---

## 📚 模块 4：证书管理（30 分钟）

### 理论（20 分钟）

**K8s 证书体系：**

```
K8s 使用 TLS 证书保证组件间通信安全

主要证书：
  - CA 证书（根证书）
  - apiserver 证书
  - apiserver-kubelet-client 证书
  - etcd 证书（ca/server/peer）
  - front-proxy 证书
  - kubelet 客户端证书

证书位置：/etc/kubernetes/pki/
```

**证书有效期：**

```
kubeadm 生成的证书默认有效期：1 年
CA 证书：10 年

过期后果：
  - apiserver 无法启动
  - kubelet 无法注册
  - 组件间通信失败
  - 集群瘫痪
```

### 🔨 实操：查看和续签证书（10 分钟）

```bash
# 查看所有证书到期时间
kubeadm certs check-expiration

# 续签所有证书（到期前执行）
kubeadm certs renew all

# 续签后重启控制面组件
# 方法 1：重启 kubelet
systemctl restart kubelet

# 方法 2：移走再放回 manifests（强制重建 Pod）
mv /etc/kubernetes/manifests/*.yaml /tmp/
sleep 10
mv /tmp/*.yaml /etc/kubernetes/manifests/
sleep 60

# 验证
kubeadm certs check-expiration
```

**✍️ 记录：**
- 证书到期时间：__________
- CA 证书到期时间：__________

证书到期时间：May 18, 2027（364天后）

CA 证书到期时间：May 06, 2036（9年后）

---

## 📚 模块 5：集群升级（20 分钟）

### 理论（20 分钟）

**升级策略：**

```
K8s 支持跨 1 个小版本升级：
  1.29 → 1.30 ✅
  1.29 → 1.31 ❌（需要先升到 1.30）
```

**升级顺序：**

```
1. 升级 kubeadm
2. 升级控制面（Master 逐台升）
   → Master1 先升，验证 OK
   → Master2 升
   → Master3 升
3. 升级 Worker（逐台升）
   → 先 drain（驱逐 Pod）
   → 升级 kubelet + kubeadm
   → uncordon（恢复调度）
```

**升级命令概览（了解即可）：**

```bash
# Master 升级流程
yum install -y kubeadm-1.31.0
kubeadm upgrade plan
kubeadm upgrade apply v1.31.0
yum install -y kubelet-1.31.0 kubectl-1.31.0
systemctl restart kubelet

# Worker 升级流程
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
yum install -y kubeadm-1.31.0 kubelet-1.31.0
kubeadm upgrade node
systemctl restart kubelet
kubectl uncordon <node>
```

---

## ❓ 模块 6：面试题

### Q1: etcd 备份怎么做？多久备一次？

```
命令：
  etcdctl snapshot save /path/backup.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=... --cert=... --key=...

频率：
  - 生产环境：每天至少 1 次
  - 关键操作前：手动备份一次
  - 建议 crontab 自动备份 + 异地存储
```

### Q2: etcd 恢复流程？

```
单 Master：
  1. 停 apiserver 和 etcd
  2. 备份旧 /var/lib/etcd
  3. etcdctl snapshot restore → 新 data-dir
  4. 启动 etcd 和 apiserver

多 Master（生产）：
  1. 所有 Master 停 etcd
  2. 每台用同一个备份恢复
  3. 修正集群配置（initial-cluster）
  4. 逐台启动
```

### Q3: Master 挂了会怎样？

```
挂 1 台：
  - VIP 飘移（3-5 秒）
  - etcd 多数派还在
  - controller/scheduler 重新选主
  - 集群正常 ✅

挂 2 台：
  - etcd 无法形成多数派
  - 集群冻结（只读，不能改）❌

挂 3 台：
  - 集群完全不可用 ❌
  - 需要从 etcd 备份恢复
```

### Q4: K8s 证书过期了怎么办？

```
症状：
  - apiserver 无法启动
  - kubectl 报 x509 certificate expired
  - 节点 NotReady

解决：
  kubeadm certs renew all
  systemctl restart kubelet

预防：
  - 定期检查：kubeadm certs check-expiration
  - 提前续签（到期前 30 天）
  - 监控告警
```

### Q5: 集群升级流程？

```
1. 先升 Master（逐台）
   → kubeadm upgrade apply
   → 升 kubelet + kubectl
   
2. 再升 Worker（逐台）
   → drain（驱逐 Pod）
   → 升 kubeadm + kubelet
   → uncordon（恢复调度）

注意：
  - 只能跨 1 个小版本
  - 先在测试环境验证
  - 升级前 etcd 备份
```

### Q6: drain 和 cordon 区别？

```
cordon：
  → 标记节点不可调度
  → 已有 Pod 不受影响
  → kubectl cordon <node>

drain：
  → cordon + 驱逐所有 Pod
  → Pod 会被重新调度到其他节点
  → kubectl drain <node> --ignore-daemonsets

uncordon：
  → 恢复节点可调度
  → kubectl uncordon <node>
```

### Q7: etcd 为什么必须奇数节点？

```
Raft 协议需要多数派（> 50%）同意才能写入

3 节点：多数派 = 2，容忍挂 1 台
4 节点：多数派 = 3，容忍挂 1 台（和 3 节点一样！）
5 节点：多数派 = 3，容忍挂 2 台

4 节点比 3 节点多一台机器但容忍故障数相同
→ 浪费资源，所以用奇数
```

### Q8: 生产环境 etcd 部署建议？

```
1. 独立部署（不和 Master 混部）
   → 避免资源争抢
   
2. SSD 磁盘
   → etcd 对磁盘 IOPS 敏感
   
3. 3 或 5 节点
   → 3 节点容忍 1 故障
   → 5 节点容忍 2 故障
   
4. 定期备份 + 异地存储
   → crontab + S3/NFS

5. 监控 etcd 性能
   → 延迟、磁盘使用、集群健康
```

### Q9: 你做过哪些高可用和灾备措施？

```
（结合你的实际经验回答）

1. 集群架构：3 Master + keepalived + haproxy
2. etcd 备份：crontab 每天自动备份
3. 故障演练：模拟 Master 宕机，验证 VIP 飘移
4. 证书管理：定期检查到期时间
5. 监控告警：Prometheus 监控 etcd 和节点健康
```

### Q10: 集群完全崩溃怎么恢复？

```
前提：有 etcd 备份

步骤：
  1. 重建一台 Master
     → kubeadm init（不加入旧集群）
  2. 恢复 etcd 数据
     → etcdctl snapshot restore
  3. 启动控制面
     → apiserver / controller / scheduler 自动拉起
  4. 加入其他 Master
     → kubeadm join --control-plane
  5. 加入 Worker
     → kubeadm join
  6. 验证所有资源恢复

没有 etcd 备份 = 从零开始
所以备份是生命线
```

---

## 🎯 速记版（考前 1 分钟）

```
高可用 4 层：VIP(网络) haproxy(API) etcd(数据) leader-election(控制)
容忍故障：3 Master 挂 1 台正常，挂 2 台冻结
etcd 备份：etcdctl snapshot save，每天至少 1 次
etcd 恢复：停 etcd → snapshot restore → 启动
证书有效期：1 年（CA 10 年），kubeadm certs renew all
升级顺序：Master 逐台 → Worker 逐台（drain → 升级 → uncordon）
升级限制：只能跨 1 个小版本
drain vs cordon：drain 驱逐 Pod，cordon 只标记不调度
```

---

## 📊 自测清单

```
☐ 1. K8s 高可用的 4 个层次？
☐ 2. 3 Master 能容忍几台故障？为什么？
☐ 3. etcd 备份命令？（完整写出来）
☐ 4. etcd 恢复的步骤？
☐ 5. Master 挂了后 VIP、etcd、controller 分别怎么处理？
☐ 6. K8s 证书默认有效期？怎么续签？
☐ 7. 集群升级的顺序？
☐ 8. drain 和 cordon 区别？
☐ 9. 生产环境 etcd 部署建议？
☐ 10. 没有 etcd 备份，集群崩了怎么办？
```
