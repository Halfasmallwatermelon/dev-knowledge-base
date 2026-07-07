# 基础知识文档库

> 系统化积累前端、后端、数据库、部署、测试等开发知识的学习笔记仓库。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 简介

本仓库是一个**开发知识学习与沉淀**空间，不是单一应用代码库。内容按领域分目录组织，配合 Cursor Agent 规则与技能，支持结构化学习、增量笔记与跨领域串联。

## 目录结构

| 目录 | 说明 |
|------|------|
| [`frontend/`](frontend/) | HTML/CSS/JS、框架、工程化、性能 |
| [`backend/`](backend/) | API、架构、安全、中间件、微服务（含 Node/Python/Go/Java/C#/Rust） |
| [`database/`](database/) | SQL、建模、索引、事务、NoSQL |
| [`deployment/`](deployment/) | Docker、CI/CD、云部署、监控 |
| [`testing/`](testing/) | 单元/集成/E2E、TDD、质量保障 |
| [`general/`](general/) | 工具链、Git、设计模式、软技能、学习路线 |

## 如何使用

### 直接阅读

浏览各目录下的 Markdown 笔记，按主题学习。

### 配合 Cursor Agent

仓库内置 `.cursor/rules/` 与 `.cursor/skills/`，在 Cursor 中打开本项目后，可直接提问，例如：

- 「解释一下 React 的 useEffect 依赖数组」
- 「在 database/ 下写一篇 MySQL 索引优化笔记」
- 「给我一条从零到能部署全栈项目的学习路线」

详见 [AGENTS.md](AGENTS.md)。

## 笔记规范

单篇笔记推荐结构：

1. 概述（解决什么问题）
2. 核心概念
3. 典型用法 / 示例
4. 常见误区与最佳实践
5. 与其他领域的关联
6. 参考资源

模板见 [`.cursor/skills/write-learning-note/template.md`](.cursor/skills/write-learning-note/template.md)。

## 贡献

欢迎 Issue 与 Pull Request：

- 修正错误、补充示例、完善链接
- 新增笔记请遵循上述结构与「由浅入深、可验证」原则
- 引用请优先官方文档与标准规范

## 许可证

本项目采用 [MIT License](LICENSE) 开源。
