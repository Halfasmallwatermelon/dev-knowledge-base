---
name: deployment-knowledge
description: 获取、讲解与沉淀部署运维知识（Docker、K8s、CI/CD、Nginx、HTTPS、监控）。用户学习部署、写 deployment/ 笔记、问容器/流水线/发布/回滚时使用。
---

# 部署知识技能

## 触发后动作

1. Read `deployment/` 已有笔记
2. 遵循 `.cursor/rules/deployment-learning.mdc`
3. 落盘时用 `write-learning-note` 模板

## 讲解顺序

目标环境 → 构建产物 → 运行方式 → 网络/证书 → 监控 → 回滚

## 必含元素（按主题选用）

- **环境**：dev / staging / prod 配置差异
- **容器**：Dockerfile 多阶段、健康检查、资源限制
- **CI/CD**：build → test → image → deploy 流水线
- **可观测性**：日志、指标、链路追踪

## 部署清单模板

```markdown
## 部署前
- [ ] 环境变量已配置
- [ ] 数据库迁移已执行
- [ ] 健康检查端点可用

## 部署后
- [ ]  smoke test 通过
- [ ] 日志无 ERROR 激增
- [ ] 回滚命令已确认
```

## 故障排查顺序

日志 → 健康检查 → 端口/防火墙 → 权限/密钥 → 依赖服务

## 权威资源

详见 [resources.md](resources.md)
