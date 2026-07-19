# open-teambition Tech Stack & Architecture

> 技术选型与关键架构决策。
> 状态：**草案（待评审）**。标注「待确认」的条目尚未拍板。
>
> 相关文档：[data-model.md](./data-model.md) · [permission-model.md](./permission-model.md)

## 1. Background & Goals

open-teambition 是一个开源的项目协作工具（Teambition 类），核心是**项目 / 任务协作 + 实时协同**。

与一般协作工具不同，本项目从第一天起就要求 **Agent 友好**：

- 内建 **CLI** 与 **MCP** 能力（与 Web 一起交付）。
- 内建**项目管理特化的 Agent**，形态为**响应式（live query 触发）+ 对话式**双模。

因此架构设计的第一目标是：**让业务能力可被 Web / CLI / MCP / Agent 四类入口无差别复用，避免后期为 Agent 化返工。**

## 2. Core Architecture

> **把「能力（Capability）」做成传输无关的一等公民内核，Web API / CLI / MCP / Agent 全部退化成薄适配器。**

不按 REST 路由组织业务，而是维护一个中心 **Capability Registry（能力注册表）**。每个能力是一个自描述对象，声明：

- `name`：如 `task.create`、`task.move`、`project.listMembers`
- `description`：面向人与 LLM 的自然语言说明（一等公民，定义时强制）
- `input` / `output`：Zod schema（机器可读，跨端复用）
- `scopes`：所需权限
- `idempotent`：是否幂等
- `handler(input, ctx)`：真正的业务逻辑，`ctx` 携带 actor / db / 事件发射器

一个注册表，四种适配器**自动派生**：

| 入口 | 派生方式 |
|------|----------|
| Web API | 遍历注册表生成 Hono 路由 + `zod-validator` + Hono RPC 类型 |
| CLI | Zod schema → 命令参数 / flags + `--help` |
| MCP | Zod → JSON Schema → MCP Tool 定义（复用 `description`） |
| Agent | 同一份定义即 function-calling 的 tool schema |

**收益**：业务逻辑只写一次；新增能力时四个入口同时具备，无需分别实现。

## 3. Tech Stack Overview

| 层次 | 选型 | 说明 |
|------|------|------|
| 仓库结构 | pnpm workspaces + Turborepo | Monorepo，前后端共享类型 |
| 语言 | TypeScript（全栈） | 端到端类型安全 |
| 运行时 | **Node.js（自托管）** | 已定稿 |
| 后端框架 | **Hono** + Hono RPC | 轻量、类型友好 |
| ORM | **Drizzle ORM** | TS-first、SQL 可控 |
| 数据库 | **PostgreSQL** | 关系型数据天然契合 |
| 事件流 / 实时 | **Postgres `LISTEN/NOTIFY`** + WebSocket | MVP 不引入 Redis |
| 鉴权 | **API Token / PAT + scope** | 人与 Agent 均为 principal |
| 前端构建 | Vite + TypeScript | |
| 前端 UI | **shadcn/ui + Tailwind CSS** | 可控、无锁定 |
| 服务端状态 | **TanStack Query** | |
| 客户端状态 | Zustand | 轻量 |
| 表单 | React Hook Form + Zod | 复用共享 schema |
| 看板拖拽 | **dnd-kit** | |
| 路由 | TanStack Router 或 React Router v7 | 待确认 |
| 校验 | Zod + `@hono/zod-validator` | 单一事实来源 |

## 4. Layer Details

### 4.1 Core (`packages/core`)

- 纯 TypeScript，**传输无关**，不依赖 Hono / HTTP。
- 承载领域模型 + Capability Registry。
- `packages/shared` 存放 Zod schema 与共享类型，供前后端与各适配器复用。

### 4.2 API (`apps/api`)

- Hono + `@hono/node-server`。
- 从注册表生成 REST 路由与 Hono RPC 客户端类型。
- WebSocket 提供实时推送（看板/任务变更广播）。
- 通过 `ctx.actor` 贯穿身份，写操作落审计日志。

### 4.3 Data Layer

- Drizzle ORM + `pg`（node-postgres）。
- 迁移用 `drizzle-kit`。
- 事件流与 live query 触发基于 Postgres `LISTEN/NOTIFY`。
- 领域模型详见 [data-model.md](./data-model.md)。

### 4.4 Web (`apps/web`)

- React + Vite + Tailwind + shadcn/ui。
- TanStack Query 管理服务端状态；Zustand 管理本地 UI 状态。
- 看板拖拽用 dnd-kit。

### 4.5 Agent (`apps/agent`)

- **响应式**：基于 core 注册的命名 live query，数据变更经 `LISTEN/NOTIFY` 触发重算，结果集变化时驱动 Agent 行为（如任务逾期自动提醒/改期）。
- **对话式**：用户通过 MCP client（IDE/客户端）调用 MCP 工具指挥 Agent。
- Agent 作为一个 **principal**，通过受限 scope 的 token 调用与人类相同的 Capability 内核。

### 4.6 CLI (`apps/cli`) & MCP (`apps/mcp`)

- 均为**薄适配器**，不含业务逻辑，仅做协议转换。
- 与 Web 一起在 MVP 交付。

## 5. Architecture Decision Records (ADR)

### ADR-1: Capability Registry as Core, Thin Adapters

见第 2 节。业务逻辑只写一次，防止为 CLI/MCP/Agent 重复实现。

### ADR-2: Postgres LISTEN/NOTIFY for Events, No Redis in MVP

- 所有状态变更发领域事件，同时驱动：前端实时更新、审计日志、Agent 响应式触发。
- MVP 用 Postgres `LISTEN/NOTIFY` 承载事件总线与 live query 触发。
- Redis 作为后续横向扩展项（多节点 pub/sub + 缓存）预留抽象层。

### ADR-3: Multi-Principal Identity + Audit + Idempotency + Structured Errors

- 人与 Agent 都是 principal；支持 API Token / PAT，Agent token 走受限 scope。
- 写操作记录 actor 审计日志。
- 写操作支持 Idempotency-Key（Agent 会重试/并发）。
- 错误统一为机器可读结构 `{ code, message, retryable, details }`。
- 权限模型详见 [permission-model.md](./permission-model.md)。

### ADR-4: Standard SQL Only, No pgvector

- 项目/任务检索与 live query 一律用标准 SQL 查询（Drizzle）实现。
- 不引入 `pgvector` 依赖或向量数据结构。
- 不为向量检索预留任何接口或抽象。

## 6. Directory Structure (Planned)

```
packages/
  core/          # 领域模型 + Capability Registry（传输无关，纯 TS）
  shared/        # Zod schema / 共享类型
apps/
  api/           # Hono：注册表 → HTTP 适配器 + WebSocket
  web/           # React 前端
  cli/           # 注册表 → CLI 适配器
  mcp/           # 注册表 → MCP server 适配器
  agent/         # PM Agent：response(live query) + 对话式
docs/
  README.md
  tech-stack.md
  data-model.md
  permission-model.md
```

> `cli` / `mcp` / `agent` 均不含业务逻辑，只做协议转换。

## 7. Open Questions

1. **前端路由**：TanStack Router（类型更强）还是 React Router v7？
2. **鉴权范围**：MVP 是否只做本地账号 + Token，OAuth/第三方登录延后？
3. **首期功能范围**：项目 / 看板任务（建、移动、查）+ 一个响应式 Agent 触发（任务逾期）+ 一个对话式 MCP 工具入口。是否加/减？
4. **交付节奏**：先产出「架构骨架 + 一个可跑通的垂直切片」评审，还是一次铺开整套 MVP？

## 8. Deferred Alternatives

| 方案 | 结论 | 理由 |
|------|------|------|
| Cloudflare Workers + Durable Objects | **未采用** | 已定 Node 自托管；更符合开源可自托管预期 |
| Redis（事件总线/缓存） | **延后** | MVP 用 Postgres `LISTEN/NOTIFY` 即可 |
| pgvector / 语义检索 | **不采用** | 一律用标准查询；不预设、不预留接口 |
| Yjs（CRDT 文档协同） | **延后** | 首期聚焦项目/看板 |

## Changelog

| Date | Change |
|---|---|
| 2026-07-16 | 初稿 |
| 2026-07-19 | 合并重复文档，统一英文文件名，补充交叉引用 |
