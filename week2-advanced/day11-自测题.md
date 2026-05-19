### Q1: etcd 备份怎么做？多久备一次？
etcdctl snapshot save 命令

- 生产环境：每天至少1次
- 关键操作前：手动备份一次
- 建议crontab 自动备份 + 异地存储

### Q2: etcd 恢复流程？

单Master：
1、停apiserver和etcd
2、备份旧/var/lib/etcd
3、etcdctl snapshot restore -> 新data-dir
4、启动etcd和apiserver

多master：
1、所有master停etcd
2、每台用同一个备份恢复
3、修正集群配置（initial-cluster）
4、逐台启动

### Q3: Master 挂了会怎样？

挂1台：
VIP漂移
etcd多数派还在
controller/scheduler重新选主
集群正常

挂2台：
etcd无法形成多数派
集群冻结（只读，不能改）

挂3台：
集群完全不可用
需要从etcd备份恢复


### Q4: K8s 证书过期了怎么办？

症状：
apiserver无法启动
kubectl 报 x509 certificate expired
节点notReady

解决：
kubeadm certs renew all
systemctl restart kubelet

预防：
定期检查 kubeadm certs check-expiration
提前续签 到期前30天
监控告警


### Q9: 你做过哪些高可用和灾备措施？
1、集群架构：3 Master + keepalived + haproxy
2、etcd备份：crontab 每天自动备份
3、故障演练：模拟Master宕机，验证VIP漂移
4、证书管理：定期检查到期时间
5、监控告警：Prometheus 监控etcd和节点健康


### Q10: 集群完全崩溃怎么恢复？

前提：有etcd备份

步骤：
1、重建一台Master
kubeadm init 不加入旧集群
2、恢复etcd数据
etcdctl snapshot restore
3、启动控制面
apiserver/controller/scheduler
4、加入其他Master
kubeadm join --control-plane
5、加入Worker
kubeadm join
6、验证所有资源恢复


