# D1：K8s 架构与组件

### Q1: K8s 有哪些核心组件？各自职责？

**答**：

K8s 分控制平面和工作节点。

控制平面（Master）：
- **kube-apiserver**：所有请求的入口，唯一能访问 etcd 的组件
- **etcd**：分布式 KV 数据库，存储集群所有状态
- **kube-scheduler**：负责 Pod 调度，决定 Pod 运行在哪个节点
- **kube-controller-manager**：运行各种控制器，保证实际状态趋近期望状态

工作节点（Worker）：
- **kubelet**：节点代理，管理 Pod 生命周期
- **kube-proxy**：维护节点上的网络规则，实现 Service
- **容器运行时**（containerd）：真正运行容器


### Q2: kubectl apply 后发生了什么？

**答**：

1. kubectl 解析 YAML，发请求到 apiserver（通过 VIP + haproxy）
2. apiserver 验证、鉴权后，把数据写入 etcd
3. controller-manager 监听到 etcd 变化，比如发现新建了 Deployment：
   - Deployment Controller 创建 ReplicaSet
   - ReplicaSet Controller 创建 Pod 对象（无节点）
4. scheduler 监听到无节点的 Pod，执行调度算法（预选 + 优选）
5. scheduler 把节点信息写回 apiserver
6. 目标节点的 kubelet 监听到分配给自己的 Pod
7. **kubelet 调用 CRI（containerd）启动容器**
8. **CNI（Calico）配置 Pod 网络**
9. Pod 状态变为 Running，kubelet 上报状态到 apiserver

### Q3: 集群高可用怎么设计的？

**答**：

我用 keepalived + haproxy 实现 control-plane 的高可用。

VIP 是 192.168.105.200，作为统一入口。

**keepalived 通过 VRRP 协议管理 VIP**：
- Master1 优先级 100（MASTER）
- Master2 优先级 90（BACKUP）
- Master3 优先级 80（BACKUP）

某台 Master 故障时，VIP 自动飘到优先级次高的节点，3-5 秒切换。

**haproxy 监听 8443 端口**，把请求负载到 3 台 Master 的 **6443（apiserver）**，策略 roundrobin，并对后端做健康检查。

我配置了 check_haproxy 脚本，haproxy 挂了会降低本机 priority，触发 VIP 切换，避免单点失效。

### Q4: 如果 Master1 挂了会怎样？

**答**：

1. **VIP 切换**：keepalived 检测到 Master1 心跳丢失，3-5 秒内 VIP 飘到 Master2
2. **etcd**：3 副本中挂 1 个，剩 2 个仍能形成多数派，集群状态可用
3. **controller-manager / scheduler**：通过 leader election 重新选主
4. **结果**：客户端无感切换，集群继续工作

但如果挂 2 台 Master：
- etcd 只剩 1 个，无法形成多数派 → 集群冻结

### Q5: 为什么需要 keepalived 和 haproxy 两个？只用一个不行吗？

**答**：

**只用 keepalived**：实现了主备切换，但所有请求都打 MASTER 一台，浪费 BACKUP 节点。这是"高可用"但不是"负载均衡"。

**只用 haproxy**：实现了负载均衡，但客户端必须知道某台 haproxy 的 IP，那台挂了就完蛋。

**两者结合**：
- keepalived 提供稳定的入口（VIP）
- haproxy 把流量分散到多台后端
- = 高可用 + 负载均衡

### Q6: VIP 是真实存在的 IP 吗？

**答**：

不是。VIP 是 keepalived 在某台机器的网卡上"额外加的别名"。

```bash
ip addr show eno16777736
# inet 192.168.105.128/24 ...           ← 真实 IP
# inet 192.168.105.200/24 secondary ... ← VIP（虚拟）
```

同一时刻，**只有一台机器拥有 VIP**。客户端通过 ARP 找到 VIP 对应的 MAC，本质和访问普通 IP 一样。