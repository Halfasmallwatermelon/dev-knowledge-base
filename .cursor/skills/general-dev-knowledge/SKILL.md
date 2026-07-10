---
name: general-dev-knowledge
description: 获取与沉淀通用开发知识（Git、命令行、HTTP、设计模式、调试、协作流程）。用户问工具链、版本控制、代码规范、软技能或写入 general/ 目录时使用。
---

# 通用开发知识技能

## 零基础讲解

- 工具链概念先用人话：Git 像「带时光机的存档」、HTTP 像「浏览器和后端的对话规则」
- 命令示例带注释，说明每条命令做什么、在什么目录执行

## 触发后动作

1. Read `general/` 已有笔记
2. 落盘时用 `write-learning-note` 模板

## 常见主题

| 主题 | 要点 |
|------|------|
| Git | 分支策略、rebase vs merge、冲突解决 |
| HTTP | 方法、状态码、Cookie、CORS 概览 |
| 调试 | 复现 → 隔离 → 假设 → 验证 |
| 设计模式 | 场景驱动，避免为模式而模式 |
| 协作 | PR 描述、Code Review、文档即代码 |

## Git 命令备忘

```bash
git status          # 查看哪些文件被修改了
git diff            # 查看具体改了什么内容
git log --oneline -10  # 查看最近 10 条提交记录（简短格式）
git switch -c feature/xxx  # 创建并切换到新分支 feature/xxx
git add -p          # 交互式选择要暂存的部分改动
git commit -m "type(scope): message"  # 提交暂存的改动并写说明
```

## 权威资源

- [Pro Git](https://git-scm.com/book/zh/v2)
- [RFC 9110 HTTP](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Refactoring Guru — 设计模式](https://refactoringguru.cn/design-patterns)

## 与其他技能协作

全栈问题常需联合 `frontend-knowledge`、`backend-knowledge` 等；先用本技能打基础，再深入领域技能。
