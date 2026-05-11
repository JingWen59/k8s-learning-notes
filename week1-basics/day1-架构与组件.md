# D1：K8s 架构与组件

##  1. K8s 架构

### 1.1 k8s 节点 与 职责（2）

- Master节点：负责决策
- Worker节点：负责执行


##  2. K8s 组件

### 2.1 Master节点 的组件 与 职责（4）

- apiserver：负责作为唯一入口接受所有请求
- etcd：负责集群状态存储 
- scheduler：负责决定 Pod 去哪个节点 
- manager：负责控制循环

### 2.2 Node 组件（Master 和 Worker 都有）

- kubelet：节点代理，启动和管理 Pod 
- kube-proxy：Service 网络转发
- containerd：容器运行时


## 3. 高可用设计

### 3.1 为什么需要高可用？

因为单节点宕机后，集群无法管理


### 3.2 高可用实现 方法 和 内容

- 方法：虚拟IP（VIP）
- 内容：给客户端提供一个统一入口（VIP）


### 3.3 keepalived：决定 VIP 在哪台

**核心机制**：

- 通过 VRRP 协议在 3 台 Master 之间互发心跳
- priority 决定谁拿 VIP（优先级最高的 MASTER 拿）
- MASTER 挂了，BACKUP 自动抢 VIP


### 3.4 haproxy：负载均衡

仅有 keepalived 的问题：

```
client → VIP → Master1
         所有请求都打 Master1
         Master2/3 闲着
```
**这只是主备切换，不是负载均衡。**

haproxy 解决：

- 监听 8443 端口
- 把请求 roundrobin（轮询）到 3 台 Master 的 6443
- 对后端做健康检查


### 3.5 端口设计

| 端口 | 用途 |
|------|------|
| 8443 | haproxy 监听端口（VIP 入口） |
| 6443 | apiserver 真实端口 |

为什么不一样？因为 haproxy 和 apiserver 都在同一台机器上，监听同一端口会冲突。



### 3.6 完整请求流程

```
1. kubectl apply 发请求
       ↓
2. 请求到 192.168.105.200:8443 (VIP:8443)
       ↓
3. VIP 当前在 Master1 上（keepalived 管理）
       ↓
4. 请求到达 Master1 网卡
       ↓
5. Master1 上的 haproxy 接到请求
       ↓
6. haproxy roundrobin 转发到 Master2:6443
       ↓
7. Master2 的 apiserver 处理
       ↓
8. 返回结果
```


### 3.7 故障切换流程

```
正常状态：
[Master1] ← VIP
[Master2] BACKUP（监听心跳）
[Master3] BACKUP（监听心跳）

Master1 宕机：
[Master1] 💀 不再广播
[Master2] ⏰ 3 秒收不到心跳
[Master3] ⏰ 3 秒收不到心跳

切换：
[Master2] (priority=90) 抢到 VIP
[Master3] (priority=80) 让位

完成（3-5 秒内）：
[Master1] 💀
[Master2] ← VIP（接管）
[Master3] BACKUP
```

整个过程**客户端无感知**。


## 📚 4. kubectl apply 完整流程

```
你执行 kubectl apply -f nginx.yaml
       ↓
请求到 VIP:8443
       ↓
haproxy 转发到某台 apiserver:6443
       ↓
apiserver 验证请求 → 写入 etcd
       ↓
controller-manager 监听到变化
       ↓
创建 ReplicaSet → 创建 Pod（状态 Pending）
       ↓
scheduler 监听到 Pending Pod
       ↓
执行调度算法（预选+优选）→ 分配节点
       ↓
目标节点的 kubelet 监听到任务
       ↓
kubelet 调用 containerd（CRI）
       ↓
containerd 调用 runc 创建容器
       ↓
Calico 给 Pod 配网络（CNI）
       ↓
Pod 运行 ✅
```

### 关键点

- 所有操作都经过 apiserver
- etcd 是状态的"唯一真相源"
- 组件间通过"监听 etcd"协同（不是直接调用）

---



