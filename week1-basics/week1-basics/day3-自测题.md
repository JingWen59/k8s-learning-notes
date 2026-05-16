### Q1: Service 4 种类型？
ClusterIP: 集群内部IP
NodePort：端口，外部访问
LoadBalancer：外部云负载均衡器
[ExternalName: DNS别名，指向外部域名]

### Q2: ClusterIP 是真实 IP 吗？怎么实现的？
不是。只存在于iptables规则中
数据包目标是ClusterIP时，iptables做DNAT改写为Pod IP

### Q3: kube-proxy iptables 模式 vs IPVS 模式？
iptables模式：默认，简单，service多了性能下降
ipvs模式：负载均衡

### Q4: 从集群外访问 Pod 有几种方式？
NodePort 
Ingress 
LoadBalancer 

### Q5: Service 的 DNS 名格式？
<svc-name>.<namespace>.svc.cluster.local

### Q6: Headless Service 是什么？和普通 Service 区别？
没有ClusterIP，所有pod都有单独的Hostmame


### Q7: Endpoints 是什么？
service背后的pod的实际ip列表


### Q8: Pod 之间怎么通信？
同一台节点内：通过linux bridge直接通信
不同节点之间：通过CNI


### Q9: 为什么 Pod IP 不能直接用？要用 Service？
pod ip直接用的话没有负载均衡，service相当于一个主机可以负责转接，而且对外只需要暴露service的ip


### Q10: NodePort 的缺点？生产环境用什么替代？
缺点就是端口太少，每个service占一个端口，直接暴露端口不安全，没有域名路由能力
