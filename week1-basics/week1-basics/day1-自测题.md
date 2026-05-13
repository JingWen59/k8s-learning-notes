Q1: K8s 核心组件？
APISERVER,ETCD,CONTROLLER,SCHEDULER,KUBELET,CONTAINERD,KUBEPROXY

Q2: kubectl apply 后流程？
解析，apiserver接受，etcd,controller创建pod对象，scheduler分配pod到节点，kubelet调用containerd起容器

Q3: 高可用设计？
keepalived用vip对外界暴露统一的访问地址，外界的访问统一转发的主控制器上；算法根据优先级确定主master，通过在不同master主机之间发送心跳确保当前主master没挂，如果挂了做主备切换 + haproxy负责轮询，负载均衡请求到不同的Master节点上

Q4: Master1 挂了？
vip飘到其他节点上，目前还剩两台主机，那么仍然符合etcd多数节点，controller重新选出新的主控制器
客户端 发送请求到 VIP的8443端口 → haproxy转发到某台 Master的apiserver 6443端口

Q5: 为什么 keepalived + haproxy 都要？
keepalived做故障切换，haproxy做负载均衡，负责不同的功能

Q6: VIP 是真实 IP 吗？
不是。是网卡上的虚拟IP。
