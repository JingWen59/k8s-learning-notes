# Day 1：K8s 架构与组件

> 日期：2026-05-11
> 学习时长：X 小时
> 状态：⏳ 进行中

---

## 🎯 今日目标

- [ ] 理解 K8s 整体架构
- [ ] 搞清楚 6 大核心组件各自职责
- [ ] 吃透高可用设计（keepalived + haproxy）
- [ ] 画出 kubectl apply 完整流程

---

## 📚 1. K8s 整体架构

### 1.1 我的理解（用自己的话）

K8s 集群是一套"管理系统"，分两类节点：

- **Master**：大脑，负责"决策"
- **Worker**：手脚，负责"执行"

### 1.2 集群拓扑

```
                      ┌─────────────────────┐
                      │ VIP: 192.168.105.200│
                      │  (keepalived+haproxy)│
                      └──────────┬──────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
        ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
        │  Master1  │      │  Master2  │      │  Master3  │
        │  .128     │      │  .129     │      │  .130     │
        └───────────┘      └───────────┘      └───────────┘
                                 │
                          ┌──────▼──────┐
                          │   Worker1   │
                          │    .131     │
                          └─────────────┘
```

### 1.3 关键点

- 3 Master + 1 Worker
- 通过 VIP (192.168.105.200) 统一入口
- 高可用：任意 1 台 Master 挂了不影响集群

---

## 📚 2. 六大核心组件

### 2.1 Master 组件

| 组件 | 职责 | 类比 |
|------|------|------|
| kube-apiserver | 唯一入口，所有请求的网关 | 前台接待 |
| etcd | 集群状态存储 | 账本 |
| kube-scheduler | 决定 Pod 去哪个节点 | 领班 |
| controller-manager | 各种控制循环 | 经理 |

### 2.2 Node 组件（Master 和 Worker 都有）

| 组件 | 职责 |
|------|------|
| kubelet | 节点代理，启动和管理 Pod |
| kube-proxy | Service 网络转发 |
| containerd | 容器运行时 |

### 2.3 容易混淆的点

**apiserver 和 etcd 的关系**

- apiserver 是"前台"，负责接收请求
- etcd 才是"账本"，存所有数据
- 只有 apiserver 能直接读写 etcd
- 其他组件想读数据，都要通过 apiserver

**scheduler 和 controller-manager 的区别**

- scheduler 只干一件事：调度（决定 Pod 去哪个 Node）
- controller-manager 管所有控制逻辑：
  - Deployment Controller：保证副本数
  - Node Controller：监控节点健康
  - 还有几十个 Controller

**kubelet 和 kube-proxy 的区别**

- kubelet：管 Pod 的生命周期
- kube-proxy：管 Service 的网络转发

---

## 📚 3. 高可用设计（重点 ⭐）

### 3.1 为什么需要高可用？

单 Master 的问题：

```
┌─────────┐
│ Master1 │ ← 客户端
└─────────┘
   ↓ 挂了
   集群无法管理 ❌
```

3 Master 也有问题：客户端连哪台？

### 3.2 核心思路：VIP（虚拟 IP）

给客户端一个**统一入口**，客户端不关心具体连哪台机器。

```
类比：
  普通做法：找小张请打 13800001（他的手机）
  VIP做法：找客服请打 4001234567（自动转接）
```

VIP `192.168.105.200` 就是这个"客服热线"。

### 3.3 keepalived：决定 VIP 在哪台

**核心机制**：

- 通过 VRRP 协议在 3 台 Master 之间互发心跳
- priority 决定谁拿 VIP（优先级最高的 MASTER 拿）
- MASTER 挂了，BACKUP 自动抢 VIP

**配置**：

| 节点 | priority | 状态 |
|------|----------|------|
| Master1 | 100 | MASTER |
| Master2 | 90 | BACKUP |
| Master3 | 80 | BACKUP |

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

### 3.8 为什么 priority 要不一样？

**防止脑裂**：

```
如果两台 priority 都是 100：
  Master1 觉得"我应该拿 VIP"
  Master2 觉得"我应该拿 VIP"
  → 两台都配置 VIP
  → 网络冲突 → 灾难
```

不同 priority 保证"只有一台是 MASTER"。

### 3.9 check_haproxy 脚本的作用

防止"VIP 在但 haproxy 不工作"的情况：

```
haproxy 挂了
   ↓
脚本检测到（exit 1）
   ↓
keepalived 降低本机 priority（-2）
   ↓
另一台 Master 优先级变高
   ↓
抢 VIP，由它的 haproxy 接管
```

---

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

## 💡 今日收获

### "啊哈"时刻

- VIP 是虚拟的，不属于任何机器，是网卡上的"额外别名"
- keepalived 和 haproxy 各司其职，缺一不可
  - keepalived 解决"找谁"
  - haproxy 解决"怎么分"
- 组件间通过 etcd 间接通信，实现解耦

### 遗留疑问

- ？leader election 怎么工作？
- ？云上 K8s 的高可用是怎么实现的？
- ？为什么 etcd 必须奇数节点？

### 明天要深入

- Pod 生命周期
- 工作负载（Deployment / StatefulSet / DaemonSet）

---

## ❓ 面试题整理

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
7. kubelet 调用 CRI（containerd）启动容器
8. CNI（Calico）配置 Pod 网络
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

**haproxy 监听 8443 端口**，把请求负载到 3 台 Master 的 6443（apiserver），策略 roundrobin，并对后端做健康检查。

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
