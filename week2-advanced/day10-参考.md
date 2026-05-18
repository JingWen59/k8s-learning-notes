# Day 10：CI/CD + Helm（精简版）

> 日期：2026-05-20
> 主题：Helm 包管理、GitOps 理念、ArgoCD

---

## ✅ 今日学习清单

```
上午（3h）：
  ☐ 1. 复习 Day 9（15 分钟）
  ☐ 2. Helm 核心概念 + Chart 结构
  ☐ 3. Helm 常用操作（实验 1）

下午（3h）：
  ☐ 4. 创建自己的 Helm Chart（实验 2）
  ☐ 5. GitOps 理念 + CI/CD 流程
  ☐ 6. ArgoCD 部署 + 演示（实验 3）

晚上（1h）：
  ☐ 7. 面试题自测
  ☐ 8. push 到 GitHub
```

---

## 📚 模块 1：Helm 核心概念（45 分钟）

### 理论（30 分钟）

**Helm 是什么？**

```
K8s 的"包管理器"（类似 yum/apt）

没有 Helm：
  手动写 Deployment.yaml + Service.yaml + ConfigMap.yaml + ...
  改配置要逐个文件改
  不同环境（dev/test/prod）要维护多份 YAML

有了 Helm：
  一个命令装一整套应用
  用 values.yaml 管理不同环境配置
  一键升级/回滚
```

**3 个核心概念：**

```
Chart（图表）：
  → 一个应用的"安装包"
  → 包含所有 YAML 模板 + 默认配置
  → 类比：rpm/deb 包

Release（发布）：
  → Chart 安装到集群后的一个实例
  → 同一个 Chart 可以安装多次（不同 Release 名）
  → 类比：把 nginx.rpm 装了 3 次，叫 nginx-prod/nginx-dev/nginx-test

Repository（仓库）：
  → 存放 Chart 的地方
  → 类比：yum 源
```

**Chart 目录结构：**

```
mychart/
├── Chart.yaml          # Chart 的元信息（名字、版本）
├── values.yaml         # 默认配置值
├── templates/          # YAML 模板目录
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # 模板辅助函数
└── charts/             # 子依赖（可选）
```

**模板语法（Go template）：**

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: {{ .Values.containerPort }}
```

```yaml
# values.yaml
replicas: 3
image:
  repository: nginx
  tag: alpine
containerPort: 80
```

**不同环境怎么管理：**

```
values.yaml          → 默认值
values-dev.yaml      → 开发环境覆盖
values-prod.yaml     → 生产环境覆盖

安装时：
  helm install myapp ./mychart -f values-prod.yaml
  → 生产配置覆盖默认配置
```

### 🔨 实验 1：Helm 常用操作（15 分钟）

```bash
# 1.1 查看已安装的 Release
helm list -A

# 1.2 搜索 Chart
helm search repo nginx

# 1.3 查看某个 Chart 的默认 values
helm show values stable/nginx-ingress 2>/dev/null || echo "尝试其他 chart"

# 1.4 查看 Release 历史
helm history loki -n monitoring 2>/dev/null

# 1.5 常用命令速记
# helm install <name> <chart>         安装
# helm upgrade <name> <chart>         升级
# helm rollback <name> <revision>     回滚
# helm uninstall <name>               卸载
# helm list                           列出
# helm status <name>                  状态
# helm template <chart>               渲染模板（不安装）
```

---

## 📚 模块 2：创建自己的 Helm Chart（60 分钟）

### 🔨 实验 2：从零创建 Chart

```bash
# 2.1 创建 Chart 骨架
cd /root
helm create myapp
ls myapp/
# Chart.yaml  charts/  templates/  values.yaml

# 2.2 看生成的默认模板
cat myapp/Chart.yaml
cat myapp/values.yaml
cat myapp/templates/deployment.yaml

# 2.3 简化：删掉不需要的，只留核心
rm -f myapp/templates/tests/*
rm -f myapp/templates/hpa.yaml
rm -f myapp/templates/serviceaccount.yaml

# 2.4 修改 values.yaml
cat > myapp/values.yaml << 'EOF'
replicaCount: 2

image:
  repository: nginx
  tag: alpine

service:
  type: NodePort
  port: 80
  nodePort: 30088

ingress:
  enabled: false
EOF

# 2.5 渲染模板（预览生成的 YAML，不安装）
helm template myapp ./myapp

# 2.6 安装
helm install myapp ./myapp -n default

# 2.7 验证
kubectl get deploy,svc,pods | grep myapp
curl http://192.168.105.128:30088

# 2.8 修改配置升级（改副本数）
helm upgrade myapp ./myapp --set replicaCount=5
kubectl get pods | grep myapp
# 应该变成 5 个

# 2.9 回滚
helm rollback myapp 1
kubectl get pods | grep myapp
# 回到 2 个

# 2.10 查看历史
helm history myapp

# 2.11 卸载
helm uninstall myapp
```

**✍️ 记录：**
- Chart 目录结构：__________
- values.yaml 的作用：__________
- helm template 和 helm install 的区别：__________

Chart 目录结构：

myapp/
├── Chart.yaml          # Chart 的元数据（名称、版本、描述）
├── values.yaml         # 默认配置值（副本数、镜像、端口等）
├── templates/          # K8s 资源模板目录
│   ├── _helpers.tpl    # 模板辅助函数（如名称、标签定义）
│   ├── deployment.yaml # Deployment 资源模板
│   ├── service.yaml    # Service 资源模板
│   ├── serviceaccount.yaml  # ServiceAccount 资源模板
│   ├── hpa.yaml        # HPA 自动伸缩模板
│   └── NOTES.txt       # 安装后的提示信息
└── charts/             # 依赖的子 Chart


values.yaml 的作用：
定义 Chart 的默认配置参数（如 replicaCount、image、service.type、autoscaling 等），模板文件通过 .Values.xxx 引用这些值。用户可以在安装/升级时通过 --set 或 -f 自定义覆盖，实现一套模板、多套配置的灵活部署。

helm template 和 helm install 的区别：

	helm template	helm install
作用	仅在本地渲染模板，输出生成的 YAML	渲染模板并实际部署到 K8s 集群
是否连接集群	❌ 不需要	✅ 需要
是否创建资源	❌ 不创建	✅ 创建 Pod、Service 等资源
用途	调试模板、检查语法错误	正式安装应用

---

## ☕ 午休

---

## 📚 模块 3：CI/CD + GitOps（45 分钟）

### 理论（30 分钟）

**传统 CI/CD 流程：**

```
开发 push 代码
    ↓
CI（持续集成）：
  → 拉代码 → 编译 → 跑测试 → 构建镜像 → 推镜像到仓库
    ↓
CD（持续部署）：
  → kubectl apply / helm upgrade 部署到 K8s
    ↓
应用更新 ✅
```

**传统 CD 的问题：**

```
1. 谁执行的 kubectl？需要集群凭证
   → 凭证泄露风险
2. 部署状态只在 CI 系统里
   → 集群实际状态和期望状态可能不一致
3. 手动回滚麻烦
```

**GitOps 理念：**

```
核心思想：
  Git 仓库 = 唯一真相源（Single Source of Truth）
  集群状态必须和 Git 仓库一致

怎么做：
  1. 所有 K8s 配置（YAML/Helm Chart）存在 Git 仓库
  2. 一个 Agent 在集群里监听 Git 仓库
  3. Git 有变化 → Agent 自动同步到集群
  4. 集群被手动改了 → Agent 检测到不一致 → 自动修复

优点：
  ✅ Git 审计（谁改了什么，什么时候改的）
  ✅ 回滚简单（git revert）
  ✅ 集群凭证不外泄（Agent 在集群内部）
  ✅ 多集群统一管理
```
Git 仓库是远程代码托管平台，不在集群里

**传统 CI/CD vs GitOps：**

```
               传统 CI/CD              GitOps
──────────────────────────────────────────────────
触发方式      CI 系统 push 到集群       Agent 从 Git pull
凭证位置      CI 系统持有集群凭证       Agent 在集群内
真相源       CI 流水线配置             Git 仓库
回滚方式      重新跑流水线              git revert
漂移检测      无                       自动检测并修复
代表工具      Jenkins / GitLab CI      ArgoCD / Flux
```

**ArgoCD 工作流程：**

```
1. 开发者 push YAML 到 Git 仓库
       ↓
2. ArgoCD（集群内）监听到 Git 变化
       ↓
3. ArgoCD 对比 Git 内容和集群实际状态
       ↓
4. 有差异 → 自动/手动同步
       ↓
5. 集群状态 = Git 仓库内容
```

---

### 🔨 实验 3：部署 ArgoCD（60 分钟）

```bash
# 3.1 安装 ArgoCD
kubectl create namespace argocd

# 下载安装文件（如果 GitHub 不通，用浏览器下载后传上来）
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 如果上面不行，手动下载
# 浏览器访问：https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# 保存后 scp 到 Master1，再 apply

# 3.2 等待 Pod 起来（需要 3-5 分钟）
kubectl get pods -n argocd -w

# 3.3 改 Service 为 NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30443}]}}'

# 3.4 获取初始密码
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
echo
# 账号：admin，密码：上面输出的

# 3.5 浏览器访问
# https://192.168.105.128:30443
# 注意是 HTTPS，浏览器会提示不安全，点"继续访问"

# 3.6 创建一个 Application（连接 Git 仓库）
# 在 ArgoCD UI 中：
#   + New App
#   Application Name: demo
#   Project: default
#   Repository URL: https://github.com/JingWen59/k8s-learning-notes（你的仓库）
#   Path: yaml/deployment（仓库里的 YAML 目录）
#   Cluster URL: https://kubernetes.default.svc
#   Namespace: default
#   → Create

# 3.7 点 Sync → ArgoCD 会把 Git 仓库里的 YAML 部署到集群
```

**⚠️ 如果 ArgoCD 镜像拉不到**（国内常见）：

```bash
# 用 daocloud 拉取核心镜像（在所有节点执行）
crictl pull docker.m.daocloud.io/argoproj/argocd:v2.10.0
ctr -n k8s.io images tag docker.m.daocloud.io/argoproj/argocd:v2.10.0 quay.io/argoproj/argocd:v2.10.0
```

**如果实在装不上 ArgoCD**（资源/网络限制），理解概念即可，面试能讲清楚流程就行。

**✍️ 记录：**
- ArgoCD 做了什么：__________
- GitOps 和传统 CI/CD 区别：__________

---

## 📚 模块 4：CI/CD 完整流程图（15 分钟）

### 面试时画的图

```
完整 GitOps 流程：

开发者
  ↓ git push（代码）
┌──────────────┐
│  代码仓库     │  ← 应用代码
│  (GitLab)    │
└──────┬───────┘
       ↓ webhook 触发
┌──────────────┐
│  CI 流水线    │  ← GitLab CI / Jenkins
│  编译→测试    │
│  →构建镜像    │
│  →推镜像仓库  │
└──────┬───────┘
       ↓ 修改镜像 tag
┌──────────────┐
│  配置仓库     │  ← K8s YAML / Helm Chart
│  (Git)       │
└──────┬───────┘
       ↓ ArgoCD 监听
┌──────────────┐
│  ArgoCD      │  ← 集群内 Agent
│  对比+同步    │
└──────┬───────┘
       ↓
┌──────────────┐
│  K8s 集群    │  ← 自动更新
└──────────────┘
```

**关键点：代码仓库和配置仓库分离**

```
代码仓库：应用源码（Java/Go/...）
配置仓库：K8s YAML / Helm Chart

CI 修改配置仓库的镜像 tag
ArgoCD 监听配置仓库
→ 职责分离，安全
```



一个 Chart 解决一个“独立部署单元”的需求 —— 可以是一个微服务、一个完整应用、或一套技术栈（如监控、日志、数据库）。你按功能边界划分：什么功能应该一起部署、一起升级、一起回滚，就放进一个 Chart。

helm是用来根据chart来安装应用的

values.yaml 让同一个 Chart 能部署到不同环境（dev/prod）、不同配置（小副本/大副本）、不同版本（v1/v2），而不需要复制整个 Chart。

---

## ❓ 模块 5：面试题

### Q1: Helm 解决了什么问题？

```
1. 复杂应用需要很多 YAML（Deployment+Service+ConfigMap+...）
   → Helm 把它们打包成一个 Chart，一键安装
   
2. 不同环境（dev/test/prod）配置不同
   → values.yaml 管理差异，模板复用

3. 升级和回滚困难
   → helm upgrade / helm rollback 一条命令

4. 应用依赖管理
   → Chart 可以依赖其他 Chart
```

### Q2: Helm 3 和 Helm 2 区别？

```
Helm 2：需要在集群里装 Tiller（有安全风险）
Helm 3：去掉了 Tiller，直接用 kubeconfig 操作

Helm 3 的优点：
  - 更安全（不需要集群内的高权限 Pod）
  - 更简单（不用管理 Tiller）
  - Release 信息存在 Secret 里（而不是 ConfigMap）
```

### Q3: helm template 和 helm install 区别？

```
helm template：只渲染模板，输出 YAML，不部署
helm install：渲染 + 部署到集群

用途：
  helm template 用于 CI 中检查生成的 YAML 是否正确
  helm install 真正部署
```

### Q4: GitOps 和传统 CI/CD 区别？

```
传统 CI/CD：
  CI 系统持有集群凭证 → push 到集群
  凭证外泄风险，无漂移检测

GitOps：
  Agent 在集群内 → 从 Git pull
  Git = 唯一真相源
  自动检测漂移并修复
  回滚 = git revert

代表工具：ArgoCD / Flux
```

### Q5: ArgoCD 怎么工作？

```
1. 在集群内运行（不需要外部凭证）
2. 连接 Git 仓库（监听变化）
3. 定期对比 Git 内容 vs 集群实际状态
4. 有差异时：
   - 自动同步（auto sync）
   - 或通知人工确认（manual sync）
5. 支持 Helm Chart / Kustomize / 纯 YAML
```

### Q6: 多环境怎么管理？

```
方案 1：多 values 文件
  values-dev.yaml / values-prod.yaml
  helm install -f values-prod.yaml

方案 2：多分支
  Git 分支 dev → 部署到 dev 集群
  Git 分支 main → 部署到 prod 集群
  ArgoCD 分别监听不同分支

方案 3：多目录
  /envs/dev/values.yaml
  /envs/prod/values.yaml
  ArgoCD ApplicationSet 管理
```

### Q7: CI/CD 流水线中的镜像安全？

```
1. 镜像扫描（Trivy / Snyk）
   → CI 阶段扫描漏洞
   → 高危漏洞阻止部署

2. 镜像签名
   → 只允许签名过的镜像运行
   → Cosign + Admission Controller

3. 私有镜像仓库
   → Harbor 自建
   → 不依赖外网
```

### Q8: Helm Chart 的 values 优先级？

```
从低到高：
  1. Chart 内 values.yaml（默认值）
  2. -f values-custom.yaml（文件覆盖）
  3. --set key=value（命令行覆盖，最高）

例：
  helm install myapp ./chart \
    -f values-prod.yaml \
    --set image.tag=v2.0
  
  image.tag 最终是 v2.0（--set 最高）
```



---

## 🎯 速记版（考前 1 分钟）

```
Helm 3 概念：Chart(包) Release(实例) Repo(仓库)
Chart 结构：Chart.yaml + values.yaml + templates/
Helm 命令：install/upgrade/rollback/uninstall/template/list
GitOps：Git=唯一真相，Agent(ArgoCD)在集群内拉取并同步
vs 传统CI/CD：Push→集群(有凭证风险) vs Pull←Git(安全)
ArgoCD：监听 Git → 对比集群 → 自动/手动同步
多环境：多 values 文件 / 多分支 / 多目录
CI/CD 全流程：代码→CI(编译测试构建)→改配置仓库→ArgoCD→集群
```

---

## 📊 自测清单

```
☐ 1. Helm 解决了什么问题？
☐ 2. Chart 目录结构？values.yaml 的作用？
☐ 3. helm template vs helm install 区别？
☐ 4. Helm 3 去掉了什么？为什么？
☐ 5. GitOps 的核心理念（一句话）？
☐ 6. ArgoCD 怎么工作？画流程图
☐ 7. 传统 CI/CD 和 GitOps 的 3 个区别？
☐ 8. 多环境配置怎么管理？
☐ 9. 完整 CI/CD 流程画图（代码→镜像→配置仓库→集群）
☐ 10. 为什么代码仓库和配置仓库要分离？
```
