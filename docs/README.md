# open-teambition Documentation

> 项目设计文档索引。状态均为**草案（待评审）**。

## Documents

| Document | Description |
|---|---|
| [tech-stack.md](./tech-stack.md) | 技术选型、架构理念、Capability Registry、ADR |
| [data-model.md](./data-model.md) | 领域模型：meta/instance 分层、`custom_fields` 体系、实体关系 |
| [permission-model.md](./permission-model.md) | 权限模型：资源树、角色、权限点、字段级授权 |

## Reading Order

1. **tech-stack.md** — 了解整体架构目标与技术栈
2. **data-model.md** — 了解领域实体与数据结构设计
3. **permission-model.md** — 了解授权如何与领域模型衔接

## Terminology Cross-Reference

跨文档共享的核心命名。**领域模型名与 DB 表名一致**（snake_case），DB schema 以 permission-model.md 为准。

| 模型 / 列 | 说明 |
|---|---|
| `workspace` / `project` / `task` | 业务实体，与 `resources` 表共享主键 |
| `custom_fields` | 字段 schema / 元数据 |
| `custom_field_values` | 字段实例值 |
| `principal` / `api_tokens` | 统一主体与 PAT |
| `audit_events` | 写操作审计 |
| `tasks.due_at` | 任务截止日期（内置列） |
| `roles` / `permission_points` / `grants` | 授权子系统 |
| UUID（`gen_random_uuid()`） | 全局 ID 策略；ltree label 用去连字符 uuid |
