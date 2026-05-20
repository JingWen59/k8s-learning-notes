### Q1: requests 和 limits 区别？
requests：调度依据，最少需要多少
limits：运行上限，最多用多少

### Q2: QoS 3 个等级？
Guaranteed：requests = limits，最后被杀

Burstable：设了但不相等，中间被杀

BestEffort：什么都没设，最先被杀


### Q3: nodeSelector 和 nodeAffinity 区别？
nodeSelector：标签匹配，硬限制
nodeAffinity：灵活
- required（硬）/ preferred（软）

### Q4: Taint 和 Toleration？
Taint 污点：给节点打污点，排斥Pod
Toleration 容忍：给Pod加容忍，可以调度到有污点的节点


### Q5: Master 为什么不跑业务 Pod？
Master有污点，所以一般的pod分配不上去


### Q6: 怎么让 Pod 分散到不同节点？
使用podAntiAffinity：同一标签的Pod尽量不在同一节点


### Q7: LimitRange 和 ResourceQuota 区别？
LimitRange：单个pod的默认值和上下限
ResourceQuota：整个Namespace的资源总配额


### Q8: Pod 被 OOMKilled 怎么办？
kubectl describe pod   确认OOMKilled
原因是内存超过Limits
解决方法是：调大limits，优化应用内存使用，检查是否有内存泄漏
