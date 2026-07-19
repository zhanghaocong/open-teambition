# open-teambition 权限模型设计

> 本文档描述 open-teambition 的权限模型：资源树 + 自定义角色 + 权限点 + 字段级授权（含自定义字段）。
> 文中 SQL / 数字均在真实 PostgreSQL 16 上验证过。
> 状态：**草案（待评审）**。

## 1. 目标与核心原则

权限模型有两条硬约束：

1. **任何操作的授权判定，都归约为「单个权限点 × 单个目标资源」**，不出现复合布尔表达式。
2. **任何资源（含字段）都能被扣到一个权限点上**，且能沿资源树继承。

由此形成核心心脏——统一判定函数：

```
can(principal, permission_point, resource) -> boolean
```

Web / CLI / MCP / Agent 四类入口的鉴权全部复用它。

## 2. 核心概念

| 概念 | 说明 |
|------|------|
| **Resource（资源）** | 一切受管对象（org/space/project/task/file/comment…），成**树**，每个资源有且仅有一个父。 |
| **Permission Point（权限点）** | 授权最小单位，命名 `<type>.<action>`（见第 3 节）。 |
| **Role（角色）** | 权限点的具名集合。**DB 自定义角色**（数据驱动，可按组织自定义）。 |
| **Grant（授权）** | `(principal, role, resource)`，沿资源树**向下继承**。 |
| **Principal（主体）** | `user` / `agent` / `service` 统一为主体，走同一套 grant。 |
| **Token scope** | API Token / PAT 携带权限点子集 + 可选资源范围，对有效权限再收窄。 |

## 3. 权限点命名空间

字段**不进资源树**（否则「行×字段」爆炸），字段维度放进权限点命名空间：

| 层级 | 形式 | 例子 |
|------|------|------|
| 实体级 | `<type>.<action>` | `task.read`、`task.update`、`task.move` |
| 普通字段级 | `<type>.field.<key>.<read\|write>` | `task.field.title.write` |
| 敏感字段级 | `<type>.sfield.<key>.<read\|write>` | `task.sfield.salary.read` |

**通配（规范文本）**：段内 `*` 表示「恰好一段」，单独一个 `*` 表示「全通配」。
- `task.field.*.read` — 读所有普通字段（够不到 `sfield`，敏感字段默认隐藏）
- `*.read`、`task.*`、`*`

> 但字段权限点具体是哪一个，由**字段元数据配置**（见第 7 节），上面的 `field`/`sfield` 只是默认约定。

## 4. 数据库建模

标签用**去连字符的 UUID**（32 位十六进制），任何 PG 版本都是合法 ltree label。

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
  type       text not null,
  parent_id  uuid references resources(id) on delete cascade,
  path       ltree not null,                 -- parent.path || 去连字符的自身 id
  created_at timestamptz not null default now(),
  created_by uuid references principals(id)
);
create index on resources using gist (path);   -- 服务子树扫描
create index on resources (parent_id);          -- 列表查询必需
create table projects (id uuid primary key references resources(id) on delete cascade, name text not null);
create table tasks    (id uuid primary key references resources(id) on delete cascade,
                       project_id uuid not null references projects(id), title text not null,
                       stage_id uuid, due_at timestamptz);

-- 权限点目录：内建(来自能力注册表, 代码 seed) + 用户自定义
create table permission_points (
  point             text primary key,             -- 'task.read' | 'task.field.title.write' | 'task.hr.read'
  label             text not null,
  description       text,
  builtin           boolean not null default false,
  scope_resource_id uuid references resources(id)  -- 自定义点归属范围；null=全局
);

-- 自定义角色（数据驱动）
create table roles (
  id uuid primary key default gen_random_uuid(),
  scope_resource_id uuid references resources(id) on delete cascade,  -- null=内置全局角色
  name text not null
);
create table role_permissions (
  role_id          uuid not null references roles(id) on delete cascade,
  permission_point text not null,                 -- 规范通配文本，如 'task.field.*.write'
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
create index on grants (principal_id, resource_id);   -- 判定性能核心

-- 自定义字段：元数据里配置读/写权限点（可空 → 回退实体级）
create table custom_fields (
  id                uuid primary key default gen_random_uuid(),
  scope_resource_id uuid not null references resources(id) on delete cascade,
  applies_to        text not null,        -- 'task'
  key               text not null,
  label             text not null,
  kind              text not null,        -- 'text'|'number'|'date'|'select'|'multi_select'|'member'
  options           jsonb,
  read_point        text references permission_points(point),   -- 空 → <type>.read
  write_point       text references permission_points(point),   -- 空 → <type>.update
  unique (scope_resource_id, applies_to, key)
);
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
  scopes       text[] not null default '{}',      -- 权限点子集；'*' 不额外收窄
  resource_scope_id uuid references resources(id),
  expires_at   timestamptz,
  created_at   timestamptz not null default now()
);

-- 审计：只记写操作
create table audit_events (
  id uuid primary key default gen_random_uuid(),
  actor_id     uuid not null references principals(id),
  capability   text not null,       -- 'task.move'
  permission_point text not null,
  resource_id  uuid,
  input_summary jsonb,
  result       text not null,       -- 'allow' | 'deny' | 'error'
  idempotency_key text,
  created_at   timestamptz not null default now()
);
create index on audit_events (resource_id, created_at);
```

## 5. ltree 的 `@>` 运算符

`@>` 是 PostgreSQL「包含」运算符，对 `ltree` 表示**「左是右的祖先或自身」**（左是右的路径前缀）：

| 表达式 | 结果 |
|--------|------|
| `'org.space' @> 'org.space.proj.task'` | `t`（祖先） |
| `'org.space.proj.task' @> 'org.space'` | `f`（后代不是祖先） |
| `'org.space' @> 'org.space'` | `t`（含相等） |
| `'org.other' @> 'org.space.proj'` | `f`（旁支） |

`<@` 是反方向（后代或自身）。判定里 `gr.path @> target.path` 即「授权所挂资源是目标的祖先或自身」→ 权限沿树向下继承。GiST 索引可加速 `@>`/`<@`/`~`。

## 6. 判定逻辑（后端 TS + Drizzle，无存储过程）

`role_permissions.permission_point` 存规范通配文本；判定时用 `replace()`+`case` 转 `lquery`（段内 `*`→`*{1}`，单独 `*` 保持）。

```ts
import { and, eq, sql } from 'drizzle-orm';
import { grants, resources, rolePermissions } from './schema';

/** 单点判定 */
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

/** 列出有效权限点（供前端 / 批量判定，一次查询后本地 matchPoint） */
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
  return rows.map(r => r.point);   // Agent token 再与 token.scopes 求交
}
```

生成的核心 WHERE：

```sql
where g.principal_id = $1
  and gr.path @> $2::ltree
  and $3::ltree ~ (case when rp.permission_point='*' then '*'
                        else replace(rp.permission_point,'*','*{1}') end)::lquery
```

**性能优化形式**（拒绝/多 grant 场景）：ltree 标签就是去连字符的 uuid，目标 `path` 本身含全部祖先 id（深度约 5），可改成资源驱动，成本 O(树深)、与 grant 数无关：

```sql
where g.principal_id = :P and g.resource_id = any(:ancestorIds)   -- ancestorIds 由 path 解析, 无需额外查询
```

## 7. 字段级授权（含自定义字段）

- **权限点写进字段元数据**：`custom_fields.read_point / write_point`，用户可把字段扣到任意（含自定义）权限点；为空则回退实体级 `<type>.read` / `<type>.update`。
- **共享点**：多个字段可用同一个 `read_point`（如 `task.hr.read`），授一个点解锁一组字段。
- **内建字段**同理，有一份代码内字段注册表，也带 `read_point/write_point`。

执行层（对「一能力一权限点」的细化）：

- **实体读** 过 `task.read` 网关 → 逐字段按 `read_point` 脱敏（读不到的字段从响应剔除）。
- **实体写** 过 `task.update` 网关 → 对入参里实际改动的字段逐个按 `write_point` 把关。

```ts
const fieldReadPoint  = (type: string, f: FieldDef) => f.readPoint  ?? `${type}.read`;
const fieldWritePoint = (type: string, f: FieldDef) => f.writePoint ?? `${type}.update`;

export async function redactForRead(db, actor, type, entityRef, entity, fields: FieldDef[]) {
  const points = await effectivePoints(db, actor, entityRef);       // 一次
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

> 重要：**字段权限按「字段定义」判一次即可**，不要把权限谓词下推到「每一行字段值」——同一字段所有值共享同一 `read_point`（见第 9 节性能）。

## 8. 前端拦截（UX，非强制）

前端拦截只负责 UX（隐藏/禁用），真正强制在后端 `can()`。前后端共用 `matchPoint` / `pointForField`（`packages/shared`），语义与 lquery 一致。

```ts
// packages/shared/permission.ts
export function matchPoint(patterns: string[], required: string): boolean {
  const r = required.split('.');
  return patterns.some(p => {
    if (p === '*') return true;
    const ps = p.split('.');
    return ps.length === r.length && ps.every((s, i) => s === '*' || s === r[i]);  // 段级 *{1}
  });
}
export const pointForField = (type: string, f: FieldDef, act: 'read'|'write') =>
  (act === 'read' ? f.readPoint : f.writePoint) ?? `${type}.${act === 'read' ? 'read' : 'update'}`;
```

```tsx
function usePermissions(resourceRef: string) {
  return useQuery({ queryKey: ['permissions', resourceRef],
    queryFn: () => api.me.permissions.$get({ query: { resource: resourceRef } }).then(r => r.json()),
    staleTime: 60_000 });
}
function useCan(point: string, resourceRef: string) {
  const { data: points = [] } = usePermissions(resourceRef);
  return matchPoint(points, point);
}
function useCanField(resourceRef: string, type: string, field: FieldDef, action: 'read'|'write') {
  const { data: points = [] } = usePermissions(resourceRef);
  return matchPoint(points, pointForField(type, field, action));
}

// 声明式包裹
<Can point="member.invite" resource={`project:${projectId}`}><InviteButton /></Can>
```

## 9. 性能（真实 PG16 基准）

**数据集**：10 万 task（101,052 资源）、20 万 grants、2 万 principals、1 万自定义字段、100 万字段值。

| 查询 | 执行时间(热) |
|------|--------------|
| 单次 `can()` 命中（~20 grants） | 0.049 ms |
| 单次 `can()` 拒绝 | 0.198 ms |
| `can()` 命中（主体持 500 grants） | 0.41 ms |
| `effectivePoints`（单资源） | 0.09–0.13 ms |
| 单任务读取 + 字段级脱敏 | 0.18 ms |
| 看板 100 任务×10 字段 + 脱敏（补 `parent_id` 索引后） | 5.0 ms |

**结论**：
- 判定计划是**主体驱动**的，成本 ∝ 单主体 grant 数（有界，几十条），**与资源树/总 grant 规模无关**。
- 看板/列表的真正瓶颈是**列表查询索引**（`resources(parent_id)` 或 `tasks(project_id)`），不是权限。
- **字段权限按字段定义解析一次**（`effectivePoints` + `matchPoint` 对约 10 个字段判断），别下推到值行；净开销 ≈ 一次 `effectivePoints`（~0.1ms），与值行数无关。
- 实际系统中**网络往返 > 查询本身**；关键是减少往返（前端按资源上下文一次拉取、后端请求级 memo/短 TTL 缓存）。

**必备索引**：`grants(principal_id, resource_id)`、`custom_field_values(entity_id)`、`resources(parent_id)`（或 `tasks(project_id)`）、`resources.path` GiST。

## 10. 决策记录（ADR）

| # | 决策 | 说明 |
|---|------|------|
| 1 | 统一判定 `can(principal, point, resource)` | 四入口复用，一能力一权限点 |
| 2 | **DB 自定义角色** | `roles` + `role_permissions` 数据驱动 |
| 3 | **ltree** 承载资源树 + `@>` 继承 | 标签用去连字符 uuid；GiST 索引 |
| 4 | **字段级权限点写进字段元数据** | `read_point/write_point` 可配置、可共享、可回退 |
| 5 | **不写存储过程**，判定用 Drizzle 拼 SQL | ltree/lquery 用 `sql` 片段 + `replace()` |
| 6 | **审计只记写操作** | 读不落审计 |
| 7 | 纯 **allow** 语义，无显式 deny | 敏感字段用独立 `sfield` 命名空间实现默认隐藏 |
| 8 | 不引入 pgvector / 向量检索 | 一律标准查询（见选型文档 ADR-4） |

## 11. 待确认

1. `owner` 语义：资源创建者是否自动获得该资源 `owner` grant？（倾向：是）
2. 资源移动（改父）重写子树 `path` 的实现细节与频率约束。
3. Token scope 与 grants 求交的具体裁剪点（统一在 `effectivePoints` 出口处）。
