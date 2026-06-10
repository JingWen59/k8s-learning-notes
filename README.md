# K8s 学习笔记

Kubernetes 学习记录，包含学习笔记、实验操作、踩坑记录。

基于自建高可用集群（3 Master + Worker）实战，环境为阿里云 ECS / 本地 VMware，CentOS 7 + containerd + Calico。

## 集群环境

```
节点规划：
  3 Master（control-plane，kubeadm + keepalived + haproxy 高可用）
  多 Worker

技术栈：
  K8s 1.30 / containerd / Calico
  Helm / ArgoCD / Prometheus / Grafana / Loki / ELK
```

## 目录结构

```
k8s-learning-notes/
├── week1-basics/          # 基础篇
│   ├── day00-集群搭建.md
│   ├── day01-高可用.md
│   └── ...
├── week2-advanced/        # 进阶篇
│   ├── day08-Prometheus监控.md
│   ├── day09-Loki日志.md
│   ├── day10-Helm-ArgoCD.md
│   ├── day11-etcd备份恢复.md
│   ├── day15-ELK日志系统.md
│   └── ...
└── README.md
```

> 注：以上结构为示例，请根据实际文件调整。

## 学习路线

### 基础篇
- 集群搭建（kubeadm 部署多 Master 高可用）
- Keepalived + HAProxy 代理 APIServer
- Pod / Deployment / Service / Ingress
- 存储（PV / PVC / StorageClass）
- 调度（NodeSelector / Affinity / Taint / Toleration）

### 进阶篇
- Prometheus + Grafana 监控告警
- Loki 日志收集（轻量方案）
- ELK 日志系统（ES + Filebeat + Redis + Logstash）
- Helm 应用管理 + ArgoCD GitOps
- etcd 备份与恢复
- HPA 自动扩缩容
- StatefulSet 有状态服务

## 笔记特点

- 每篇包含：学习清单 + 理论模块 + 动手实验 + 面试题 + 速记版
- 真实踩坑记录（不是教程里的理想环境）
- 面试导向，每个知识点配面试回答模板

## 实战踩坑记录

- CentOS 7 xfs ftype=0 不支持 overlayfs → ext4 loop 设备绕过
- runc / libseccomp 版本冲突 → 升级系统库
- Docker Hub 拉取超时 → 镜像加速 + 内部 Harbor
- kubelet disk-pressure 频繁驱逐 → 调整 eviction 阈值
- etcd 端口残留导致启动失败 → 清理僵尸进程

## 使用说明

每篇笔记可独立阅读，建议按 day 顺序学习。代码块可直接复制执行（注意替换 IP 等环境变量）。

## License

仅供个人学习使用。
