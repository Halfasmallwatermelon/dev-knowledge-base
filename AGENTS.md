# 基础知识文档库 — Agent 指南

本仓库是**开发知识学习与沉淀**空间，不是单一应用代码库。

## 目标读者

**默认面向零基础、无编程背景的学习者。** Agent 回答与写文档时应：

- 像老师带读一样讲解，不跳步、不堆术语
- 代码与命令尽量带注释，并在代码块下方逐步说明
- 术语首次出现附英文并用人话解释

## 目标

系统化获取并整理以下领域的知识：

| 领域 | 目录 | 说明 |
|------|------|------|
| 前端 | `frontend/` | HTML/CSS/JS、框架、工程化、性能 |
| 后端 | `backend/` | API、架构、安全、中间件、微服务 |
| 数据库 | `database/` | SQL、建模、索引、事务、NoSQL |
| 部署 | `deployment/` | Docker、CI/CD、云部署、监控 |
| 测试 | `testing/` | 单元/集成/E2E、TDD、质量保障 |
| 通用 | `general/` | 工具链、Git、设计模式、软技能 |

## Agent 行为

1. **零基础友好**：先类比、再术语；示例带注释；代码块下方逐步讲解
2. **增量学习**：优先查阅已有文档，再补充新内容
3. **结构化写作**：概念 → 示例 → 逐步讲解 → 实践 → 资源
4. **权威优先**：官方文档 > 标准规范 > 社区优质文章
5. **可动手验证**：提供命令、代码片段、检查清单，并说明每步在做什么
6. **跨域串联**：解释知识点在完整开发链路中的位置

## Cursor 规则

`.cursor/rules/` 下各 `.mdc` 文件按领域提供学习指导：

- `core-learning.mdc` — 全局生效
- `frontend-learning.mdc` — 编辑 `frontend/**` 时生效
- `backend-learning.mdc` — 编辑 `backend/**` 时生效
- `database-learning.mdc` — 编辑 `database/**` 时生效
- `deployment-learning.mdc` — 编辑 `deployment/**` 时生效
- `testing-learning.mdc` — 编辑 `testing/**` 时生效

## Cursor 技能

`.cursor/skills/` 提供可执行的学习工作流（Agent 会根据描述自动选用）：

| 技能 | 用途 |
|------|------|
| `write-learning-note` | 创建/更新结构化笔记（含 template.md） |
| `plan-learning-path` | 制定分阶段学习路线与里程碑 |
| `frontend-knowledge` | 前端知识讲解与沉淀 |
| `backend-knowledge` | 后端知识讲解与沉淀 |
| `database-knowledge` | 数据库知识讲解与沉淀 |
| `deployment-knowledge` | 部署运维知识讲解与沉淀 |
| `testing-knowledge` | 测试知识讲解与沉淀 |
| `general-dev-knowledge` | Git、HTTP、设计模式等通用知识 |

## 如何使用

- 直接提问：「解释一下 React 的 useEffect 依赖数组」
- 请求文档：「在 database/ 下写一篇 MySQL 索引优化笔记」
- 学习路径：「给我一条从零到能部署全栈项目的学习路线」
