# 知识卡片：keepalived + haproxy 高可用

> 这是一份"标准答案"，可以反复复习。
> 
> 复习时，先合上文件凭记忆答，再对照。

---

## 🃏 卡片 1：keepalived 是什么？

**问**：keepalived 是什么？解决什么问题？

**标准答**：

keepalived 是一个**用 VRRP 协议管理虚拟 IP（VIP）**的工具。

**解决的问题**：
- 多台服务器之间，让一个"虚拟 IP"在它们之间漂移
- 当持有 VIP 的机器宕机，VIP 自动飘到其他机器
- 实现高可用：客户端通过 VIP 访问，无感切换

**核心机制**：
- 通过 VRRP 协议互发心跳
- priority（优先级）决定谁拿 VIP
- 高优先级 = MASTER，拿 VIP
- 低优先级 = BACKUP，待命

**记忆点**：keepalived = "保持活着" = 监控 + 切换

---

## 🃏 卡片 2：haproxy 是什么？

**问**：haproxy 是什么？为什么 K8s 高可用要用它？

**标准答**：

haproxy 是一个**高性能的 TCP/HTTP 负载均衡器**。

**在 K8s 高可用中的作用**：
- 监听 8443 端口
- 把请求 roundrobin 转发到 3 台 Master 的 6443 端口（apiserver）
- 对后端做健康检查（挂掉的 Master 不转发）

**为什么不用 nginx？**
- nginx 主要做 7 层（HTTP）
- haproxy 在 4 层（TCP）性能更好
- K8s apiserver 是 HTTPS，4 层转发更合适

**记忆点**：haproxy = "高可用代理" = 负载均衡 + 健康检查

---

## 🃏 卡片 3：为什么两个都要用？

**问**：只用 keepalived 不行吗？只用 haproxy 不行吗？

**标准答**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 只用 keepalived | VIP 切换实现高可用 | 所有请求打 MASTER 一台，浪费 BACKUP |
| 只用 haproxy | 负载均衡 | 客户端必须知道某台 IP，那台挂了就完蛋 |
| **两者结合** | 高可用 + 负载均衡 | （完美方案） |

**类比**：
- keepalived = 公司前台（永远有人接电话）
- haproxy = 分派员（把客户分配给不同部门）
- 两者结合 = 客户打总机，总是有人接，且自动分流

---

## 🃏 卡片 4：VIP 工作原理

**问**：VIP 是什么？是真实存在的 IP 吗？

**标准答**：

VIP（Virtual IP，虚拟 IP）：
- **不是真实的物理 IP**
- 是 keepalived 在某台机器的网卡上"额外添加的 IP 别名"

**验证方法**：
\```bash
ip addr show eno16777736
# inet 192.168.105.128/24 ...           ← 真实 IP
# inet 192.168.105.200/24 secondary ... ← VIP（虚拟）
\```

**关键特性**：
- 同一时刻**只有一台机器**拥有 VIP
- 客户端通过 ARP 找到 VIP 对应的 MAC 地址
- 对客户端来说，访问 VIP 和访问普通 IP 完全一样

**漂移过程**：
1. Master1 宕机
2. keepalived 在 Master2 上感知到
3. Master2 把 VIP 加到自己的网卡上
4. 同时发 ARP 广播："192.168.105.200 现在归我管"
5. 客户端的 ARP 缓存更新，请求自动到 Master2

---

## 🃏 卡片 5：priority 的作用

**问**：keepalived 的 priority 是什么？为什么要不一样？

**标准答**：

**priority 是 VRRP 协议中的"优先级"**：
- 数值范围 1-255
- 数值越大，优先级越高
- 优先级最高的节点（且健康）拿 VIP

**为什么必须不一样？**

防止**脑裂**：
- 如果两台 priority 都是 100
- 两台都认为"我应该拿 VIP"
- 两台都给自己的网卡加 VIP
- 网络上出现两个 192.168.105.200
- ARP 表混乱，请求时通时不通
- **灾难**

**配置示例**：
\```
Master1: priority 100  (MASTER)
Master2: priority 90   (BACKUP)
Master3: priority 80   (BACKUP)
```

---

## 🃏 卡片 6：完整请求流程

**问**：客户端执行 `kubectl get nodes`，请求是怎么走的？

**标准答**：

\```
1. kubectl 读取 ~/.kube/config，发请求到 https://192.168.105.200:8443
       ↓
2. 客户端 ARP 找到 VIP 的 MAC（当前在 Master1）
       ↓
3. 请求到达 Master1 的网卡
       ↓
4. Master1 的 haproxy 监听 8443，接到请求
       ↓
5. haproxy 按 roundrobin，比如转发到 Master2:6443
       ↓
6. Master2 的 apiserver 处理请求
       ↓
7. apiserver 查 etcd 拿到节点列表
       ↓
8. 返回结果（沿路返回客户端）
\```

**关键点**：
- VIP 在哪台机器，请求就先到哪台
- haproxy 再把请求分散
- apiserver 处理的可能不是 VIP 所在的机器

---

## 🃏 卡片 7：故障切换流程

**问**：Master1 宕机后会发生什么？详细描述切换过程。

**标准答**：

**T0：正常状态**
\```
Master1: VIP, priority 100, 每秒广播心跳
Master2: priority 90, 监听心跳
Master3: priority 80, 监听心跳
\```

**T1：Master1 宕机**
\```
Master1: 💀 不再广播
Master2/3: 还在收最后一次心跳
\```

**T2：3 秒后（advert_int=1，fall=3）**
\```
Master2: 3 秒没收到 Master1 心跳 → "Master1 挂了"
Master3: 3 秒没收到 Master1 心跳 → "Master1 挂了"
\```

**T3：选举**
\```
Master2 (priority=90) > Master3 (priority=80)
Master2 决定抢占 VIP
\```

**T4：VIP 切换**
\```
Master2: 把 192.168.105.200 加到自己的网卡
Master2: 发免费 ARP 广播
"192.168.105.200 → 我的 MAC"
\```

**T5：流量切换**
\```
客户端的 ARP 缓存更新
后续请求到达 Master2
Master2 的 haproxy 接管处理
\```

**总耗时**：约 3-5 秒，客户端可能感知到一两次请求失败/重试。

---

## 🃏 卡片 8：check_haproxy 脚本

**问**：keepalived 配置里的 check_haproxy 脚本是干什么的？

**标准答**：

**问题场景**：
\```
VIP 在 Master1 上
但 Master1 的 haproxy 进程挂了
keepalived 还以为 Master1 正常（VRRP 心跳还在发）
VIP 不会切换
→ 客户端请求到 Master1:8443 → 拒绝连接
→ 服务实际不可用
\```

**脚本作用**：
- 定期检测 haproxy 是否监听 8443
- 如果监听 → exit 0（正常）
- 如果不监听 → exit 1（异常）
- weight=-2 表示异常时把 priority 降低 2

**生效流程**：
\```
正常时：Master1 priority = 100
haproxy 挂了：priority = 100 - 2 = 98
如果 Master2 priority = 99 → Master2 抢 VIP
\```

**配置示例**：
\```conf
vrrp_script check_haproxy {
    script "/bin/bash -c 'if [[ $(ss -tlnp | grep 8443) ]]; then exit 0; else exit 1; fi'"
    interval 3       # 每 3 秒检查一次
    weight -2        # 失败时降低 2
    fall 10          # 连续 10 次失败才确认
    rise 2           # 连续 2 次成功才恢复
}
\```

---

## 🃏 卡片 9：端口 8443 vs 6443

**问**：为什么 haproxy 用 8443，apiserver 用 6443？

**标准答**：

**端口分配**：
- 6443：K8s apiserver 默认监听端口
- 8443：haproxy 监听端口（VIP 入口）

**为什么不一样？**

haproxy 和 apiserver **在同一台机器上**（都装在 Master）。

如果 haproxy 也监听 6443：
- 它会和 apiserver 抢 6443 端口
- 同一端口只能被一个进程监听
- 冲突 → haproxy 起不来

**所以**：
- haproxy 监听 8443
- apiserver 监听 6443
- haproxy 把 8443 的请求转发到 6443

**客户端的 kubeconfig** 里要写 VIP:8443，不是 6443。

---

## 🃏 卡片 10：面试综合回答模板

**问**：你的 K8s 集群高可用是怎么设计的？

**标准答**（3 分钟版本）：

> "我的集群是 3 Master + 1 Worker 的高可用架构，
> control-plane 的高可用通过 **keepalived + haproxy** 实现。
> 
> 首先有一个 **VIP（192.168.105.200）作为统一入口**，
> 所有客户端（包括 kubectl 和 kubelet）都通过 VIP 访问 apiserver。
> 
> **keepalived** 通过 VRRP 协议在 3 台 Master 之间维护这个 VIP：
> - Master1 优先级 100（MASTER）
> - Master2 优先级 90（BACKUP）
> - Master3 优先级 80（BACKUP）
> 
> 当持有 VIP 的 Master 宕机，keepalived 会在 3-5 秒内把 VIP 飘到另一台。
> 
> **haproxy** 在每台 Master 上监听 8443 端口，
> 把请求 roundrobin 到三台 Master 的 6443（apiserver），
> 并对后端做健康检查。
> 
> 我还配置了 **check_haproxy 检测脚本**：
> 当本机 haproxy 挂了，自动降低 keepalived 的 priority，
> 触发 VIP 切换到其他节点，避免单点失效。
> 
> 整个架构实现了**故障切换 + 负载均衡**，
> 任意 1 台 Master 故障不影响集群可用性。"

---

## 复习方法

### 第一次复习（学完当天晚上）
✅ 全部卡片过一遍，重点理解

### 第二次复习（24 小时后）
✅ 合上文件，凭记忆回答 → 对照

### 第三次复习（3 天后）
✅ 只看问题，凭记忆讲出来 → 录音回听

### 第四次复习（1 周后）
✅ 挑薄弱卡片重点过

### 第五次复习（面试前）
✅ 全部过一遍，重点是综合回答（卡片 10）


## 复习节奏：艾宾浩斯遗忘曲线

>学完当天：    第 1 次复习（晚上 20 分钟）
>24 小时后：   第 2 次复习（10 分钟）
>3 天后：      第 3 次复习（10 分钟）
>1 周后：      第 4 次复习（10 分钟）
>1 个月后：    第 5 次复习（10 分钟）

>每次间隔拉大，每次都"主动回忆"

