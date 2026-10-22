# 数据库（默认 Drizzle + PostgreSQL）

> 如果项目用的不是 Drizzle 或 PostgreSQL，以项目现有代码为准，下面的原则仍然适用。

## 结构

```
src/db/
├── schema.ts      # 所有表定义（表多了可以拆成 schema/ 目录，再 index 导出）
├── client.ts      # 导出 db 实例
└── migrations/    # drizzle-kit 生成，提交到 git
drizzle.config.ts
```

## 表设计

- 表名用复数 snake_case（`survey_answers`）。TS 里用 camelCase，在 drizzle 配置里设置 `casing: 'snake_case'` 自动转换。
- 每张表都有：`id`、`<单数表名>Id`、`created_at`、`updated_at`。时间用 `timestamp with time zone`。
- **两个 id，分工不同：**
  - `id`：`bigint` 自增主键，只在数据库内部用，不出现在外键、接口和 URL 里。Drizzle 写 `bigint('id', { mode: 'number' }).generatedAlwaysAsIdentity().primaryKey()`（`mode: 'bigint'` 时值是 JS 的 BigInt，`JSON.stringify` 会直接报错）。
  - `<单数表名>Id`：UUID v7，加唯一约束，是这一行对外的标识。比如 `users.userId`、`surveys.surveyId`（数据库列名经 `casing` 转成 `user_id`、`survey_id`）。Drizzle 写 `uuid('user_id').notNull().unique().$defaultFn(() => Bun.randomUUIDv7())`，在应用里生成，不依赖 PostgreSQL 版本。
  - **外键引用对方的 UUID，不引用自增 `id`**，列名和对方的 UUID 列同名：`surveys.userId` → `users.userId`。这样数据导出、合并到另一个库时，关联关系不依赖自增值，不用重新映射。
  - 接口的入参、出参、URL 都只用 UUID，响应 schema 里是 `z.string().uuid()`，不返回自增 `id`。
- 公共列写成 helper，所有表复用（`src/db/columns.ts`）：
  ```ts
  export const id = () => bigint('id', { mode: 'number' }).generatedAlwaysAsIdentity().primaryKey()
  export const timestamps = {
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow().$onUpdate(() => new Date()),
  }
  ```
  `$onUpdate` 要求 drizzle-orm ≥ 0.30.5，而且只在通过 Drizzle 执行 `update` 时由应用填值，迁移里不会生成数据库触发器；手写 SQL 的 `UPDATE` 要自己设置 `updated_at`。现有项目版本更低时沿用项目原来的写法。
- 金额用整数存「分」（`integer` 或 `bigint`），不要用 `float` 或 `real`。
- 枚举优先用 `text` + zod / 应用层校验，或者 pg enum。pg enum 增加值比较容易，删除和改名很麻烦，要慎用。
- 外键列一定要建索引。
- 业务上要唯一的字段加唯一约束（用户名、openid、订单号、「同一个用户对同一份问卷只能提交一次」），不要只在代码里判断。
- 不做软删除：不加 `deleted_at`，删除就是 `delete`。

## 迁移

- 改表流程：修改 `schema.ts` → `drizzle-kit generate` 生成 SQL → 检查生成的 SQL → `drizzle-kit migrate` 执行。
- **禁止**在生产环境使用 `drizzle-kit push`。`push` 只能用于个人本地的临时数据库。
- 生成的迁移 SQL 一定要人工看一遍，重点关注：删列、改类型、改名（drizzle 可能会生成「删列再加列」，导致数据丢失）。
- 线上表结构变更要向后兼容，分步进行（先加新列 → 代码同时兼容新旧 → 迁移数据 → 再删旧列），不要一步完成。
- 给线上大表加索引要用 `CREATE INDEX CONCURRENTLY`（否则会锁表）。但 PostgreSQL 不允许它在事务里执行，而 drizzle-kit 的 migrate 会把迁移包在事务里，直接写进迁移文件会报 `cannot run inside a transaction block`。做法：
  1. 从 drizzle 生成的迁移里去掉这条建索引语句（`schema.ts` 里仍然保留索引定义，保持 schema 和数据库一致）。
  2. 把 `CREATE INDEX CONCURRENTLY IF NOT EXISTS ...` 单独放进 `db/manual/` 下的 SQL 文件，在部署时用 `psql -f` 在事务外单独执行。
  3. 验证：执行 `\d 表名` 确认索引存在，并检查 `pg_index.indisvalid` 为 true（CONCURRENTLY 失败时会留下一个无效索引，需要删掉后重建）。
  4. 以上基于 drizzle-kit 迁移默认包事务的行为，实际以项目所用版本为准，第一次使用时先在测试库验证。

## 查询

- 查询逻辑写在 service 里，routes 不直接操作 db。
- 不要把数据库行直接返回给客户端，先映射成响应 schema 定义的结构，避免泄露内部字段（openid、session 信息、内部状态字段）。
- 只查需要的列，不要所有地方都 `select *`。
- 避免 N+1：列表里需要关联数据时，用 join、Drizzle 的关系查询（`with`），或者批量 `inArray` 查询。
- 分页：普通列表用 `limit / offset`；数据量很大或者需要无限滚动的，用游标分页，游标用 UUID v7（它按生成时间有序：`where survey_id < cursor order by survey_id desc`），不把自增 `id` 暴露出去。
- 多个写操作要保证一致性时，用 `db.transaction(async (tx) => { ... })`，事务内部全部使用 `tx`，不要混用 `db`。
- 并发扣减（库存、名额）用条件更新（`update ... set n = n - 1 where id = ? and n > 0`）或行锁（`for update`），不要先查再改。

## 连接

- 整个进程共用一个 db 实例，在 `client.ts` 里创建并导出，不要每次请求新建连接。
- 连接串从 `env.ts` 读取。
- 测试环境使用单独的数据库（见 `api/testing-deploy.md`）。
