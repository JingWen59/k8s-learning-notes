### Q1: Pod 和容器什么区别？
容器是一个运行环境
pod是一组容器，是调度的最小单位


### Q2: Pod 有几种状态？
pending, running, succeeded, failed, unknown


### Q3: Deployment 和 StatefulSet 区别？
deployment: 无状态（哪个pod处理请求都一样），pod无顺序启动，每个pod无独立存储，pod名随机
statefulSet: 有状态（每个Pod之间有区别），pod有顺序启动，每个pod独立PVC存储，pod名固定


### Q4: 三种探针的区别？
livenessProbe：活着吗？失败则杀掉重启
readinessProbe：准备好了吗？失败则从Service移除
startupProbe：启动完了吗？完成前不检查其他探针


### Q5: maxSurge 和 maxUnavailable？
maxSurge：更新时允许多出的节点最多有几个
maxUnavailable：更新时允许挂掉的节点最多有几个


### Q6: DaemonSet 使用场景？
日志采集
每个节点运行一个pod


### Q7: CrashLoopBackOff 怎么排查？
kubectl describe pod，看events。
一般情况：命令错误，配置错误，端口号写错，OOM


### Q8: 为什么不用裸 Pod？
因为裸pod删除之后不会自动备份，不安全


### Q9: Job 的重启策略？
never或onFailure，不能用always


### Q10: StatefulSet 为什么需要 Headless Service？
因为每个pod需要有独立hostname（为什么statefulset需要pod有独立的hostname？因为外部要 按名字找 Pod，不能随机访问。）
