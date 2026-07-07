---
name: testing-knowledge
description: 获取、讲解与沉淀测试知识（单元/集成/E2E、TDD、Mock、测试金字塔、CI 门禁）。用户学习测试、写 testing/ 笔记、问 Jest/pytest/Playwright/覆盖率时使用。
---

# 测试知识技能

## 触发后动作

1. Read `testing/` 已有笔记
2. 遵循 `.cursor/rules/testing-learning.mdc`
3. 落盘时用 `write-learning-note` 模板

## 讲解顺序

测什么 → 测哪一层 → 怎么写 → 怎么 Mock → 怎么进 CI

## 测试金字塔（默认比例）

```
        / E2E \        少量，关键用户路径
       /集成测试\      中等，API/模块协作
      /  单元测试  \    大量，纯逻辑与边界
```

## 用例结构（AAA）

```text
Arrange — 准备数据与依赖
Act     — 执行被测行为
Assert  — 断言结果与副作用
```

## 必含元素（按主题选用）

- **选型**：单元 vs 集成 vs E2E 边界
- **Mock**：何时 Mock 外部 IO，何时用 Testcontainers
- **Flaky**：异步等待、时间、随机数、共享状态
- **CI**：失败即阻断合并

## 权威资源

详见 [resources.md](resources.md)

## 实践原则

修 Bug 先写失败测试；每个核心模块至少一条 happy path + 一条边界用例
