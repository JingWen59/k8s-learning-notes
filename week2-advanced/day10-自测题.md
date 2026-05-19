### Q1: Helm 解决了什么问题？

复杂应用需要很多yaml，helm把它们打包成一个chart，一键安装；而且不同环境可以用values.yaml管理差异


### Q3: helm template 和 helm install 区别？

helm template 只渲染模版，输出yaml，不部署
helm install 渲染 + 部署到集群

用途：
helm template 用于 CI 中检查生成的yaml是否正确
helm install 真正部署

### Q4: GitOps 和传统 CI/CD 区别？

传统CI/CD：
CI系统持有集群凭证 -> push到集群
凭证外泄风险，无漂移监测

GitOps：
Agent 在集群内
Git = 唯一真相源
自动监测漂移并修复
回滚=git revert

### Q5: ArgoCD 怎么工作？

定期对比Git内容 vs 集群实际状态
有差异时：自动同步


### Q6: 多环境怎么管理？

方案 1：多 values 文件

方案 2：多分支

方案 3：多目录

### Q7: CI/CD 流水线中的镜像安全？




### Q8: Helm Chart 的 values 优先级？


