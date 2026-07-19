# open-teambition Permission Model

> 权限模型：资源树 + 自定义角色 + 权限点 + 字段级授权。
> 文中 SQL / 数字均在真实 PostgreSQL 16 上验证过。
> 状态：**草案（待评审）**。
>
> 相关文档：[tech-stack.md](./tech-stack.md) · [data-model.md](./data-model.md)

## 1. Goals & Principles

权限模型有两条硬约束：

1. **任何操作的授权判定，都归约为「单个权限点 × 单个目标资源」**，不出现复合布尔表达式。
2. **任何资源（含字段）都能被扣到一个权限点上**，且能沿资源树继承。

由此形成核心心脏——统一判定函数：

```
can(principal, permission_point, resource) -> boolean
```

Web / CLI / MCP / Agent 四类入口的鉴权全部复用它。

### Terminology Mapping

本文 SQL 示例与 [data-model.md](./data-model.md) 的对应关系：

| data-model | 本文 | 说明 |
|---|---|---|
| `field-definition` | `custom_fields` 表 | 字段 schema，含 `read_point` / `write_point` |
| `field-value` | `custom_field_values` 表 | 字段实例值 |
| `workspace` | 资源树根节点 | `type = 'workspace'` |
| `project` | 资源树中间节点 | `type = 'project'` |
| `task` | 资源树叶子节点 | `type = 'task'` |
| ULID | UUID（SQL 示例） | 领域层用 ULID；SQL 示例沿用 UUID 以兼容 PG 原生类型 |

## 2. Core Concepts

| 概念 | 说明 |
|------|------|
| **Resource（资源）** | 一切受管对象（workspace/project/task/file/comment…），成**树**，每个资源有且仅有一个父。 |
| **Permission Point（权限点）** | 授权最小单位，命名 `<type>.<action>`（见第 3 节）。 |
| **Role（角色）** | 权限点的具名集合。**DB 自定义角色**（数据驱动，可按 workspace 自定义）。 |
| **Grant（授权）** | `(principal, role, resource)`，沿资源树**向下继承**。 |
| **Principal（主体）** | `user` / `agent` / `service` 统一为主体，走同一套 grant。 |
| **Token scope** | API Token / PAT 携带权限点子集 + 可选资源范围，对有效权限再收窄。 |

### Resource Tree

```
workspace (root)
  └── project
        └── task
              └── comment / file (future)
```

## 3. Permission Point Namespace

字段**不进资源树**（否则「行×字段」爆炸），字段维度放进权限点命名空间：

| 层级 | 形式 | 例子 |
|------|------|------|
| 实体级 | `<type>.<action>` | `task.read`、`task.update`、`task.move` |
| 普通字段级 | `<type>.field.<key>.<read\|write>` | `task.field.title.write` |
| 敏感字段级 | `<type>.sfield.<key>.<read\|write>` | `task.sfield.salary.read` |

**通配（规范文本）**：段内 `*` 表示「恰好一段」，单独一个 `*` 表示「全通配」。

- `task.field.*.read` — 读所有普通字段（够不到 `sfield`，敏感字段默认隐藏）
- `*.read`、`task.*`、`*`

字段权限点具体是哪一个，由**字段元数据配置**（见第 7 节）；上面的 `field`/`sfield` 只是默认约定。

## 4. Database Schema

标签用**去连字符的 UUID**（32 位十六进制），任何 PG 版本都是合法的 ltree label。

```sql
create extension if not exists ltree;

-- 主体：人/Agent/服务统一
create table principals (
  id uuid primary key default gen_random_uuid(),
  kind text not null check (kind in ('user','agent','service')),
  created_at timestamptz not null default now()
);
create table users  (id uuid primary key references principals(id) on delete cascade,
                      email text unique not null, name text not null, password_hash text);
create table agents (id uuid primary key references principals(id) on delete cascade,
                      display_name text not null, owner_user_id uuid references users(id));

-- 资源树；领域表与资源共享主键
create table resources (
  id         uuid primary key default gen_random_uuid(),
  type       text not null,                -- 'workspace' | 'project' | 'task' | ...
  parent_id  uuid references resources(id) on delete cascade,
  path       ltree not null,               -- parent.path || 去连字符的自身 id
  created_at timestamptz not null default now(),
  created_by uuid references principals(id)
);
create index on resources using gist (path);
create index on resources (parent_id);
create table projects (id uuid primary key references resources(id) on delete cascade, name text not null);
create table tasks    (id uuid primary key references resources(id) on delete cascade,
                       project_id uuid not null references projects(id), title text not null,
                       status_id uuid, due_at timestamptz);

-- 权限点目录：内建(来自能力注册表, 代码 seed) + 用户自定义
create table permission_points (
  point             text primary key,
  label             text not null,
  description       text,
  builtin           boolean not null default false,
  scope_resource_id uuid references resources(id)
);

-- 自定义角色（数据驱动）
create table roles (
  id uuid primary key default gen_random_uuid(),
  scope_resource_id uuid references resources(id) on delete cascade,
  name text not null
);
create table role_permissions (
  role_id          uuid not null references roles(id) on delete cascade,
  permission_point text not null,
  primary key (role_id, permission_point)
);

-- 授权
create table grants (
  id uuid primary key default gen_random_uuid(),
  principal_id uuid not null references principals(id) on delete cascade,
  role_id      uuid not null references roles(id) on delete cascade,
  resource_id  uuid not null references resources(id) on delete cascade,
  created_by   uuid references principals(id),
  created_at   timestamptz not null default now(),
  unique (principal_id, role_id, resource_id)
);
create index on grants (principal_id, resource_id);

-- 字段定义（= data-model 中的 field-definition）
create table custom_fields (
  id                uuid primary key default gen_random_uuid(),
  scope_resource_id uuid not null references resources(id) on delete cascade,
  applies_to        text not null,        -- 'task' | 'project'
  key               text not null,
  label             text not null,
  kind              text not null,        -- field-type key: 'text'|'tag'|'attachment'|'sprint'|...
  options           jsonb,
  read_point        text references permission_points(point),
  write_point       text references permission_points(point),
  unique (scope_resource_id, applies_to, key)
);

-- 字段值（= data-model 中的 field-value）
create table custom_field_values (
  entity_id       uuid references resources(id) on delete cascade,
  custom_field_id uuid references custom_fields(id) on delete cascade,
  value           jsonb,
  updated_at      timestamptz not null default now(),
  primary key (entity_id, custom_field_id)
);
create index on custom_field_values (entity_id);

-- API Token / Agent scope
create table api_tokens (
  id uuid primary key default gen_random_uuid(),
  principal_id uuid not null references principals(id) on delete cascade,
  token_hash   text not null,
  scopes       text[] not null default '{}',
  resource_scope_id uuid references resources(id),
  expires_at   timestamptz,
  created_at   timestamptz not null default now()
);

-- 审计：只记写操作
create table audit_events (
  id uuid primary key default gen_random_uuid(),
  actor_id     uuid not null references principals(id),
  capability   text not null,
  permission_point text not null,
  resource_id  uuid,
  input_summary jsonb,
  result       text not null,
  idempotency_key text,
  created_at   timestamptz not null default now()
);
create index on audit_events (resource_id, created_at);
```

## 5. ltree `@>` Operator

`@>` 是 PostgreSQL「包含」运算符，对 `ltree` 表示**「左是右的祖先或自身」**：

| 表达式 | 结果 |
|--------|------|
| `'ws.proj' @> 'ws.proj.task'` | `t`（祖先） |
| `'ws.proj.task' @> 'ws.proj'` | `f`（后代不是祖先） |
| `'ws.proj' @> 'ws.proj'` | `t`（含相等） |
| `'ws.other' @> 'ws.proj.task'` | `f`（旁支） |

判定里 `gr.path @> target.path` 即「授权所挂资源是目标的祖先或自身」→ 权限沿树向下继承。

## 6. Authorization Logic

`role_permissions.permission_point` 存规范通配文本；判定时用 `replace()`+`case` 转 `lquery`。

```ts
import { and, eq, sql } from 'drizzle-orm';
import { grants, resources, rolePermissions } from './schema';

export async function can(db: Db, principalId: string, point: string, resourceId: string): Promise<boolean> {
  const [target] = await db.select({ path: resources.path })
    .from(resources).where(eq(resources.id, resourceId)).limit(1);
  if (!target) return false;

  const [row] = await db.select({ ok: sql<number>`1` })
    .from(grants)
    .innerJoin(resources, eq(resources.id, grants.resourceId))
    .innerJoin(rolePermissions, eq(rolePermissions.roleId, grants.roleId))
    .where(and(
      eq(grants.principalId, principalId),
      sql`${resources.path} @> ${target.path}::ltree`,
      sql`${point}::ltree ~ (case when ${rolePermissions.permissionPoint} = '*'
                                  then '*'
                                  else replace(${rolePermissions.permissionPoint}, '*', '*{1}')
                             end)::lquery`,
    ))
    .limit(1);
  return Boolean(row);
}

export async function effectivePoints(db: Db, principalId: string, resourceId: string): Promise<string[]> {
  const [target] = await db.select({ path: resources.path })
    .from(resources).where(eq(resources.id, resourceId)).limit(1);
  if (!target) return [];
  const rows = await db.selectDistinct({ point: rolePermissions.permissionPoint })
    .from(grants)
    .innerJoin(resources, eq(resources.id, grants.resourceId))
    .innerJoin(rolePermissions, eq(rolePermissions.roleId, grants.roleId))
    .where(and(
      eq(grants.principalId, principalId),
      sql`${resources.path} @> ${target.path}::ltree`,
    ));
  return rows.map(r => r.point);
}
```

**性能优化形式**（拒绝/多 grant 场景）：ltree 标签就是去连字符的 uuid，目标 `path` 本身含全部祖先 id，可改成资源驱动：

```sql
where g.principal_id = :P and g.resource_id = any(:ancestorIds)
```

## 7. Field-Level Authorization

- **权限点写进字段元数据**：`custom_fields.read_point / write_point`，为空则回退实体级 `<type>.read` / `<type>.update`。
- **共享点**：多个字段可用同一个 `read_point`（如 `task.hr.read`），授一个点解锁一组字段。
- **内建字段**（`title`, `status_id`, `assignee_id` 等）在代码内字段注册表中也带 `read_point` / `write_point`。

执行层：

- **实体读** 过 `task.read` 网关 → 逐字段按 `read_point` 脱敏。
- **实体写** 过 `task.update` 网关 → 对入参里实际改动的字段逐个按 `write_point` 把关。

```ts
const fieldReadPoint  = (type: string, f: FieldDef) => f.readPoint  ?? `${type}.read`;
const fieldWritePoint = (type: string, f: FieldDef) => f.writePoint ?? `${type}.update`;

export async function redactForRead(db, actor, type, entityRef, entity, fields: FieldDef[]) {
  const points = await effectivePoints(db, actor, entityRef);
  for (const f of fields)
    if (!matchPoint(points, fieldReadPoint(type, f))) delete entity.customFields[f.key];
  return entity;
}

export async function assertWritableFields(db, actor, type, entityRef, changedKeys: string[], fields) {
  const points = await effectivePoints(db, actor, entityRef);
  for (const key of changedKeys) {
    const f = fields.find(x => x.key === key)!;
    if (!matchPoint(points, fieldWritePoint(type, f))) throw new Forbidden(fieldWritePoint(type, f));
  }
}
```

> **字段权限按「字段定义」判一次即可**，不要把权限谓词下推到「每一行字段值」。

## 8. Frontend Integration (UX Only)

前端拦截只负责 UX（隐藏/禁用），真正强制在后端 `can()`。前后端共用 `matchPoint` / `pointForField`（`packages/shared`）。

```ts
// packages/shared/permission.ts
export function matchPoint(patterns: string[], required: string): boolean {
  const r = required.split('.');
  return patterns.some(p => {
    if (p === '*') return true;
    const ps = p.split('.');
    return ps.length === r.length && ps.every((s, i) => s === '*' || s === r[i]);
  });
}
```

```tsx
function useCanField(resourceRef: string, type: string, field: FieldDef, action: 'read'|'write') {
  const { data: points = [] } = usePermissions(resourceRef);
  return matchPoint(points, pointForField(type, field, action));
}
```

## 9. Performance (PostgreSQL 16 Benchmarks)

**数据集**：10 万 task（101,052 资源）、20 万 grants、2 万 principals、1 万字段定义、100 万字段值。

| 查询 | 执行时间(热) |
|------|--------------|
| 单次 `can()` 命中（~20 grants） | 0.049 ms |
| 单次 `can()` 拒绝 | 0.198 ms |
| `can()` 命中（主体持 500 grants） | 0.41 ms |
| `effectivePoints`（单资源） | 0.09–0.13 ms |
| 单任务读取 + 字段级脱敏 | 0.18 ms |
| 看板 100 任务×10 字段 + 脱敏 | 5.0 ms |

**结论**：

- 判定成本 ∝ 单主体 grant 数，**与资源树/总 grant 规模无关**。
- 看板/列表瓶颈是**列表查询索引**，不是权限。
- 字段权限按字段定义解析一次，净开销 ≈ 一次 `effectivePoints`（~0.1ms）。

**必备索引**：`grants(principal_id, resource_id)`、`custom_field_values(entity_id)`、`resources(parent_id)`、`resources.path` GiST。

## 10. ADR Summary

| # | 决策 | 说明 |
|---|------|------|
| 1 | 统一判定 `can(principal, point, resource)` | 四入口复用，一能力一权限点 |
| 2 | **DB 自定义角色** | `roles` + `role_permissions` 数据驱动 |
| 3 | **ltree** 承载资源树 + `@>` 继承 | 标签用去连字符 uuid；GiST 索引 |
| 4 | **字段级权限点写进字段元数据** | `read_point/write_point` 可配置、可共享、可回退 |
| 5 | **不写存储过程**，判定用 Drizzle 拼 SQL | ltree/lquery 用 `sql` 片段 |
| 6 | **审计只记写操作** | 读不落审计 |
| 7 | 纯 **allow** 语义，无显式 deny | 敏感字段用独立 `sfield` 命名空间 |
| 8 | 不引入 pgvector | 见 [tech-stack.md ADR-4](./tech-stack.md#adr-4-standard-sql-only-no-pgvector) |

## 11. Open Questions

1. `owner` 语义：资源创建者是否自动获得该资源 `owner` grant？（倾向：是）
2. 资源移动（改父）重写子树 `path` 的实现细节与频率约束。
3. Token scope 与 grants 求交的具体裁剪点（统一在 `effectivePoints` 出口处）。

## Changelog

| Date | Change |
|---|---|
| 2026-07-19 | 初稿 |
| 2026-07-19 | 对齐 data-model 术语（workspace、field-definition/field-value、status_id） |
