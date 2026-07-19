# open-teambition Documentation

> 项目设计文档索引。状态均为**草案（待评审）**。

## Documents

| Document | Description |
|---|---|
| [tech-stack.md](./tech-stack.md) | 技术选型、架构理念、Capability Registry、ADR |
| [data-model.md](./data-model.md) | 领域模型：meta/instance 分层、field 体系、实体关系 |
| [permission-model.md](./permission-model.md) | 权限模型：资源树、角色、权限点、字段级授权 |

## Reading Order

1. **tech-stack.md** — 了解整体架构目标与技术栈
2. **data-model.md** — 了解领域实体与数据结构设计
3. **permission-model.md** — 了解授权如何与领域模型衔接

## Terminology Cross-Reference

两份领域文档中的概念对应关系：

| data-model.md | permission-model.md | 说明 |
|---|---|---|
| `workspace` | 资源树根节点（`type: workspace`） | 租户边界 |
| `project` | `type: project` 资源节点 | 项目实例 |
| `task` | `type: task` 资源节点 | 任务实例 |
| `field-definition` | `custom_fields` 表 | 字段 schema / 元数据 |
| `field-value` | `custom_field_values` 表 | 字段实例值 |
| `principal` | `principals` 表 | 统一主体（user / agent / service） |
| `api-token` | `api_tokens` 表 | PAT + scope |
| `audit-event` | `audit_events` 表 | 写操作审计 |
| ULID（领域 ID） | UUID（SQL 示例） | 实现层可统一为 ULID 文本存储 |
