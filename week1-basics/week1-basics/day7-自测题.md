
**Pod 与工作负载（Day 2）：**   done


4. Pod 5 种状态？
ready
failed
succeeded
pending
unknown

5. 3 种探针？各自失败后的后果？
livenessProbe 失败则杀掉重启
startupProbe  完成前不检查其他探针
readinessProbe  失败则从Service移除

6. Deployment 和 StatefulSet 的 3 个核心区别？
应用有无状态，一个是headless一个用clusterip，Pod名称有无顺序


7. DaemonSet 使用场景？
每个节点上跑的任务，例如收集日志


8. CrashLoopBackOff 怎么排查？
kubectl describe pod 看events
端口号写错 配置文件写错  OOMKilled  NodeSelector没有找到匹配的节点    命令写错



**Service 与网络（Day 3）：**     

```
9. Service 4 种类型？
ClusterIP、NodePort、LoadBalancer、ExternalName

10. ClusterIP 是真实 IP 吗？怎么实现的？
不是真实IP。只存在于Iptables中，DNAT把Iptables改写为Pod IP

12. Service 的 DNS 名格式？
<svc-name>.<namespace>.svc.cluster.local

```

**Ingress 与存储（Day 4）：**

```
13. Ingress 和 Service（NodePort）的区别？
Ingress是七层HTTP，供外部访问，按域名访问；NodePort是四层TCP/UDP，内部访问，按端口访问

14. PV 和 PVC 的关系？绑定流程？
PV是存储；PVC是申请存储

15. 静态供给 vs 动态供给？

静态：管理员手动创建PV -> PVC绑定
动态：管理员创建 StorageClass -> PVC触发自动创建PV

16. PV 回收策略 Retain 和 Delete 区别？
Retain是 删除PVC不删除PV
Delete是 删除PVC也删除PV

```

**配置与安全（Day 5）：**

```
17. ConfigMap 和 Secret 区别？
configmap 普通配置 明文存储 环境变量 / 文件挂载
secret base64加密

18. RBAC 4 个概念？
Role：命名空间权限
ClusterRole：集群权限
RoleBinding：绑定Role给用户
ClusterRoleBinding：绑定ClusterRole给用户

19. Secret 的 Base64 是加密吗？
不是

20. NetworkPolicy 默认行为？
默认pod全部互通，加 NetworkPolicy 后限制其他Pod的访问

```

**调度与资源（Day 6）：**

```
21. 4 种调度方式？
nodeSelector 硬限制
nodeAffinity 硬限制required 软限制preferred 加分
podAntiAffinity pod和其他pod的反亲和 减分
Taint + Toleration  污点和容忍

22. Taint 3 种效果？
NoSchedule  不调度新pod，已有的不受影响
PreferNoSchedule 尽量不调度（软限制）
NoExecute 不调度+驱逐已有pod

23. requests 和 limits 区别？超内存会怎样？
requests是需求下限
limits是能使用的最大资源

24. QoS 3 个等级？
Guaranteed   requests=limits   最后被杀
Burstable    requests不等于limits   中间被杀
BestEffort   什么都没设   最先被杀

```

