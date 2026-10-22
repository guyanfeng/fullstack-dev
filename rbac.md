# 后台管理与角色权限（RBAC）

做后台管理、管理员账号、角色、权限点、菜单和按钮权限时读本文件。`auth.md` 管 C 端用户「是谁」，本文件管后台管理员「是谁、能做什么」。

- 只用于后台。C 端用户之间的资源归属仍按 `security.md`「越权」，不套角色。
- 后台接口按权限点授权，不按 `userId` 过滤：有 `survey:read` 的管理员能看所有用户的问卷。
- 不做数据范围（「只能看自己部门的」）。需要时先问用户。

## 整体形态

- **后台是独立的 Web 项目**（Next.js，PC 优先），如 `admin.example.com`，按 `web/guide.md`、`design.md` 和 `web/design.md` 写，和 C 端 Web 分开部署。
- **复用同一个 API**：后台接口统一挂在 `/admin` 下。后台登录 `POST /auth/admin-login` 直接挂在 `app` 上，不放进任何挂了鉴权中间件的子路由（`/admin` 或 C 端的路由组）。
- **管理员单独一张表 `admins`**，不进 `users`（这是 `auth.md`「用户表只有一张」的例外）。C 端账号和后台账号互不相通。
- **token 分开签**：后台用 `ADMIN_JWT_SECRET`，**必须和 `JWT_SECRET` 不同**，两个都至少 32 字节，在 `env.ts` 里校验：
  ```ts
  .refine((e) => e.JWT_SECRET !== e.ADMIN_JWT_SECRET, { message: 'ADMIN_JWT_SECRET 不能和 JWT_SECRET 相同', path: ['ADMIN_JWT_SECRET'] })
  ```
  这样 C 端的 token 在后台验签必然失败，后台 token 在 C 端也一样。payload 同样是 `{ sub: adminId, ver: tokenVersion, exp }`，`exp` 约 8 小时。

## C 端和后台的 origin 分开放行

两边的 cookie 都设在 `.example.com`（见 `auth.md`），浏览器访问 `api.example.com` 时 `token` 和 `admin_token` 都会带上。如果 `cors` 和 `csrf` 的白名单合并成 `[WEB_ORIGIN, ADMIN_WEB_ORIGIN]`，C 端页面上的任何脚本（XSS、被篡改的依赖）都能带着管理员的 cookie 调 `/admin`。所以**按路径选 origin，不合并白名单**：`/admin` 和 `/auth/admin-login` 只放行 `ADMIN_WEB_ORIGIN`，其余只放行 `WEB_ORIGIN`。有后台时，`auth.md` 的 `cors` 和 `security.md` 的 `csrf` 换成（这样能挡住浏览器里的跨端调用；`admin_token` 仍会随请求发到 C 端的 Next 服务端，所以 C 端服务端不能记录请求的 cookie 头）：

```ts
// src/app.ts
const isAdminPath = (path: string) => path === '/admin' || path.startsWith('/admin/') || path === '/auth/admin-login'
const allowedOrigin = (c: Context) => (isAdminPath(c.req.path) ? env.ADMIN_WEB_ORIGIN : env.WEB_ORIGIN)

app.use('*', cors({ origin: (origin, c) => (origin === allowedOrigin(c) ? origin : null), credentials: true }))
const csrfMw = csrf({ origin: (origin, c) => origin === allowedOrigin(c) })
app.use('*', (c, next) => (c.req.header('Authorization') ? next() : csrfMw(c, next)))
```

## 表结构

`id()`、`timestamps` 见 `api/database.md`「表设计」里的公共列 helper。

```ts
// src/db/schema.ts
export const admins = pgTable('admins', {
  id: id(),
  adminId: uuid('admin_id').notNull().unique().$defaultFn(() => Bun.randomUUIDv7()),
  username: text('username').notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  isSuper: boolean('is_super').notNull().default(false),
  status: text('status', { enum: ['active', 'disabled'] }).notNull().default('active'),
  tokenVersion: integer('token_version').notNull().default(0),
  ...timestamps,
})

export const roles = pgTable('roles', {
  id: id(),
  roleId: uuid('role_id').notNull().unique().$defaultFn(() => Bun.randomUUIDv7()),
  name: text('name').notNull().unique(),
  description: text('description').notNull().default(''),
  ...timestamps,
})

export const permissions = pgTable('permissions', {
  id: id(),
  permissionId: uuid('permission_id').notNull().unique().$defaultFn(() => Bun.randomUUIDv7()),
  key: text('key').notNull().unique(),                    // 代码里引用的标识，如 'survey:read'，创建后不能改
  name: text('name').notNull(),                           // 后台显示的名称，如「查看问卷」
  groupName: text('group_name').notNull().default(''),    // 角色编辑页按它分组（不用 group：PostgreSQL 保留字）
  isSystem: boolean('is_system').notNull().default(false), // 种子迁移写入的（代码引用的）为 true，不能删
  ...timestamps,
})

export const rolePermissions = pgTable('role_permissions', {
  roleId: uuid('role_id').notNull().references(() => roles.roleId, { onDelete: 'cascade' }),
  permissionId: uuid('permission_id').notNull().references(() => permissions.permissionId, { onDelete: 'cascade' }),
}, (t) => [primaryKey({ columns: [t.roleId, t.permissionId] }), index('role_permissions_permission_id_idx').on(t.permissionId)])

export const adminRoles = pgTable('admin_roles', {
  adminId: uuid('admin_id').notNull().references(() => admins.adminId, { onDelete: 'cascade' }),
  roleId: uuid('role_id').notNull().references(() => roles.roleId, { onDelete: 'restrict' }),
}, (t) => [primaryKey({ columns: [t.adminId, t.roleId] }), index('admin_roles_role_id_idx').on(t.roleId)])

export const adminAuditLogs = pgTable('admin_audit_logs', {
  id: id(),
  adminId: uuid('admin_id').notNull().references(() => admins.adminId),
  action: text('action').notNull(),          // 如 'role.permissions.update'、'survey.export'
  targetType: text('target_type').notNull(), // 如 'role'
  targetId: text('target_id'),
  detail: jsonb('detail'),
  ip: text('ip'),
  requestId: text('request_id'),             // 和日志里的 requestId 对上
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('admin_audit_logs_admin_id_created_at_idx').on(t.adminId, t.createdAt),
  index('admin_audit_logs_action_created_at_idx').on(t.action, t.createdAt),
  index('admin_audit_logs_created_at_idx').on(t.createdAt),
])
```

- 两张关联表（`role_permissions`、`admin_roles`）用复合主键，没有 `id`、UUID 和时间列；审计日志只增不改，没有 UUID 和 `updated_at`。这是 `api/database.md`「每张表都有」的例外。复合主键的第一列已经能当索引用，所以只给第二列单独建索引。
- 管理员**只停用、不删除**（审计日志要一直能关联到人）。

## 权限点

权限点、角色、角色有哪些权限、管理员有哪些角色，全部存在数据库里，在后台界面上配置。

- `key` 命名 `<模块>:<动作>`，动作用 `read`、`create`、`update`、`delete`、`export`，或具体的业务动作（`user:ban`）。
- 接口用 `requirePermission('survey:read')` 按 `key` 判断，所以：
  - **`key` 创建后不能改**：修改接口只接受 `name` 和 `groupName`。
  - **系统权限点（`is_system = true`）不能删**，接口返回 `40300`。后台新建的（`is_system = false`）可以删，删除时 `role_permissions` 级联删除。
  - 后台新建的权限点，只有被代码（接口的 `requirePermission`、前端的菜单和 `<Can>`）引用后才起作用；被引用了就要写进种子迁移（见下），变成系统权限点。
- **代码新用到的权限点，用迁移写进库**，保证每个环境部署后都有：`bunx drizzle-kit generate --custom --name <名称>` 生成空迁移文件，写入：
  ```sql
  -- key 已存在时只把它标成系统权限点，不覆盖后台改过的名称和分组
  INSERT INTO "permissions" ("permission_id", "key", "name", "group_name", "is_system") VALUES
    (gen_random_uuid(), 'survey:read', '查看问卷', '问卷', true),
    (gen_random_uuid(), 'survey:delete', '删除问卷', '问卷', true)
  ON CONFLICT ("key") DO UPDATE SET "is_system" = true;
  ```
  不能用 `DO NOTHING`：后台先建了同 `key` 的权限点（`is_system = false`）时，整行被跳过，它就一直能被删除。
  SQL 里没有应用层的 `Bun.randomUUIDv7()`，用 PostgreSQL 自带的 `gen_random_uuid()`（v4），UUID 列只要求唯一，不影响使用。
- 第一次建表时，种子迁移至少写入后台自己用的：`role:manage`、`admin:manage`、`permission:manage`、`audit:read`。
- 给角色设置权限时，入参是 `permissionIds`，先查出这些权限点，数量对不上返回 `40000`（「权限点不存在」）。给管理员分配角色（包括创建管理员时的 `roleIds`）同样先查一遍，数量对不上返回 `40000`（「角色不存在」），不要等外键报错变成 `50000`。

## API：鉴权与授权

全局中间件按 `api/guide.md`「中间件顺序」，`cors`、`csrf` 按上面「origin 分开放行」。`/admin` 子路由挂 `adminAuth`，每个路由再挂 `requirePermission`。

```ts
// src/middlewares/admin-auth.ts
type AdminEnv = { Variables: { adminId: string; isSuper: boolean; permissions: Set<string> } }

export const adminAuth = createMiddleware<AdminEnv>(async (c, next) => {
  // 放在最前面：鉴权失败的 40100、后续抛错的响应也都带上，不让浏览器和代理缓存
  c.header('Cache-Control', 'private, no-store')
  const token = c.req.header('Authorization')?.replace(/^Bearer /, '') ?? getCookie(c, 'admin_token')
  if (!token) throw new BizError(40100)
  const payload = await verify(token, env.ADMIN_JWT_SECRET, 'HS256').catch(() => { throw new BizError(40100) })
  const admin = await loadAdmin(String(payload.sub))
  if (!admin || admin.status !== 'active' || admin.tokenVersion !== payload.ver) throw new BizError(40100)
  c.set('adminId', admin.adminId)
  c.set('isSuper', admin.isSuper)
  c.set('permissions', admin.permissions)
  await next()
})

// 多个参数是「全部都要有」；不传参数在定义路由时就报错
export const requirePermission = (...perms: [string, ...string[]]) => {
  if (perms.length === 0) throw new Error('requirePermission 至少要一个权限点')
  return createMiddleware<AdminEnv>(async (c, next) => {
    if (!c.get('isSuper') && !perms.every((p) => c.get('permissions').has(p))) throw new BizError(40300)
    await next()
  })
}
```

```ts
// src/services/admin.ts：只查需要的列，不查 passwordHash
export async function loadAdmin(adminId: string) {
  const [admin] = await db
    .select({ adminId: admins.adminId, username: admins.username, isSuper: admins.isSuper, status: admins.status, tokenVersion: admins.tokenVersion })
    .from(admins).where(eq(admins.adminId, adminId))
  if (!admin) return null
  const rows = await db.selectDistinct({ key: permissions.key }).from(adminRoles)
    .innerJoin(rolePermissions, eq(rolePermissions.roleId, adminRoles.roleId))
    .innerJoin(permissions, eq(permissions.permissionId, rolePermissions.permissionId))
    .where(eq(adminRoles.adminId, adminId))
  return { ...admin, permissions: new Set(rows.map((r) => r.key)) }
}
```

```ts
// src/routes/admin/surveys.ts
export const adminSurveys = new Hono<AdminEnv>()
  .get('/', requirePermission('survey:read'), zValidator('query', PageQuery), async (c) => { ... })
  .delete('/:surveyId', requirePermission('survey:delete'), async (c) => { ... })

// src/routes/admin/index.ts：所有后台路由只从这里挂
// adminAuth 必须是第一条：Hono 按注册顺序执行，写在它前面的路由不会经过鉴权。所以在创建时链式挂上
export const admin = new Hono<AdminEnv>().use('*', adminAuth)
admin.get('/me', async (c) => { ... })   // 只要求登录，不挂 requirePermission 的唯一例外
admin.route('/surveys', adminSurveys)

// src/app.ts
app.route('/admin', admin)
app.route('/auth/admin-login', adminLogin)   // 直接挂在 app 上
```

- **每次请求都查一次权限**，不把权限放进 token。后台请求量小，多两次查询换来的是：改角色、改权限、停用账号立即生效，不用让人重新登录。
- `token_version` 只在改密码、重置密码、停用时加 1（强制下线），改角色和权限不用加。
- 每个 `/admin` 路由都要挂 `requirePermission`。唯一例外是 `GET /admin/me`（只要求登录，返回当前管理员信息和权限列表，给前端渲染菜单）。
- 没有权限返回 `40300`，未登录或 token 失效返回 `40100`。

## 防止权限提升

- **授出的权限不能超过自己拥有的**：非超级管理员编辑角色权限、给别人分配角色时，涉及的每个权限点都必须在自己的权限集合里，否则返回 `40300`。
- **不能修改权限比自己高的角色**：编辑、删除角色前，角色**原有的**权限也必须都在自己的权限集合里。否则有 `role:manage` 的低权限管理员能把高权限角色里的权限删掉，让持有它的管理员被降权。
- **不能管理权限比自己高的管理员**：重置密码、停用、改角色、编辑另一个管理员之前，加载对方的权限集合，要求对方的权限是自己的子集。否则低权限的人重置高权限管理员的密码，就能登录对方的账号拿到更高的权限。
- **不能操作自己和超级管理员**：不能改自己的角色、不能停用自己（改自己的密码走单独的接口，要校验旧密码）；不能通过接口停用、重置、编辑超级管理员。
  ```ts
  type Actor = { adminId: string; isSuper: boolean; permissions: Set<string> }   // 从 c.get(...) 取

  function assertCanGrant(actor: Actor, keys: Iterable<string>) {
    if (actor.isSuper) return
    for (const k of keys) if (!actor.permissions.has(k)) throw new BizError(40300, '不能授予自己没有的权限')
  }

  async function assertCanEditRole(actor: Actor, roleId: string) {
    if (actor.isSuper) return
    const old = await db.select({ key: permissions.key }).from(rolePermissions)
      .innerJoin(permissions, eq(permissions.permissionId, rolePermissions.permissionId))
      .where(eq(rolePermissions.roleId, roleId))
    for (const { key } of old) if (!actor.permissions.has(key)) throw new BizError(40300, '不能修改权限比自己高的角色')
  }

  async function assertCanManage(actor: Actor, targetAdminId: string) {
    if (targetAdminId === actor.adminId) throw new BizError(40300, '不能操作自己')
    const target = await loadAdmin(targetAdminId)
    if (!target) throw new BizError(40400)
    if (target.isSuper) throw new BizError(40300, '不能操作超级管理员')
    if (actor.isSuper) return
    for (const k of target.permissions) if (!actor.permissions.has(k)) throw new BizError(40300, '不能操作权限比自己高的管理员')
  }
  ```
- **超级管理员（`is_super`）只能直接在数据库里设置**（见「账号」），没有任何接口能修改这个字段。这样超级管理员不会被降到一个都不剩。
- 角色还有管理员在用时不能删，返回 `40900`（外键 `restrict` 兜底）。
- **入参 schema 只收能改的字段**（zod 默认丢掉 schema 里没有的字段，见 `api/guide.md`「路由」）：创建管理员只收 `username`、`roleIds`；编辑只收 `username`；角色、状态、密码各走专门的接口。任何入参 schema 都不收 `id`、`adminId`、`passwordHash`、`tokenVersion`、`isSuper`；改权限点不收 `key`、`isSystem`。

## 账号

- 登录、密码、防暴力破解的规则同 `auth.md`「Web：账号密码登录」（用户名规范化、argon2id、`40010`、5 次锁 15 分钟、IP 限流在后台 Next 的 Route Handler 里做），只是查的是 `admins`、签的是 `ADMIN_JWT_SECRET`。
- **不开放注册**：管理员由有 `admin:manage` 的人在后台创建，初始密码由服务端随机生成、只显示一次，管理员登录后自己修改。
- **重置密码**同样生成随机密码只显示一次，并把对方的 `token_version` 加 1。
- **超级管理员手动在数据库里插入**，代码里没有创建超级管理员的逻辑。先在本地生成密码哈希（`read -rs` 不回显，也不进 shell 历史）：
  ```bash
  read -rs PW && printf '%s' "$PW" | bun -e 'console.log(await Bun.password.hash(await Bun.stdin.text()))'; unset PW
  ```
  再在数据库里执行：
  ```sql
  INSERT INTO admins (admin_id, username, password_hash, is_super)
  VALUES (gen_random_uuid(), 'admin', '<上面输出的哈希>', true);
  ```
  `username` 按 `auth.md` 的规则写成规范化后的小写。

## 操作审计

- 后台的写操作（增删改、分配角色、改权限、重置密码、停用），以及**导出、下载敏感数据**，在 service 里和业务操作**放在同一个事务**里写一条 `admin_audit_logs`：谁（`adminId`）、做了什么（`action`）、对象（`targetType` + `targetId`）、关键参数（`detail`）、请求 IP（取法见 `api/testing-deploy.md`「真实 IP」）、`requestId`。
- `detail` 里不放密码、token 等日志禁止记录的内容（见 `api/testing-deploy.md`「日志」）。
- 审计日志没有修改和删除接口；后台只提供按管理员、动作、时间查询的列表（需要 `audit:read`）。

## 后台前端

- 登录态同 `auth.md`「Web：账号密码登录」，cookie 名改成 `admin_token`，避免和 C 端的 `token` 混在一起。
- 布局（`app/(admin)/layout.tsx`，Server Component）请求一次 `GET /admin/me`，显式传 `cache: 'no-store'`，不依赖 Next 版本的默认缓存行为：`40100` 跳登录页；拿到的权限列表通过 Provider 传给 client 组件。
- 菜单配置写成一份 `{ path, label, permission }` 列表，菜单和页面权限都从这里读，不在各处另写判断。
- 按钮级用通用组件 `<Can permission="survey:delete">`，放在 `components/common/`（见 `design.md`「组件化」）。
- 没有权限的页面显示 `ErrorState`（文案「没有访问权限」），不跳转、不白屏；操作时 API 返回 `40300` 用 toast 提示。
- **前端按权限隐藏只是体验**，真正的权限以 API 的判断为准；不能因为按钮藏起来了就省掉 API 的 `requirePermission`。

## 测试

每个 `/admin` 接口至少覆盖：
- 不带 token、带 C 端 token → `40100`（`/admin/me` 也要测）
- 已登录但没有对应权限 → `40300`；有权限 → 成功
- 停用后、改密码后旧 token → `40100`

只需要测一次的：
- 后台 token 调 C 端接口 → `40100`
- 带 `Origin: WEB_ORIGIN` 预检 `/admin/me`，响应没有 `Access-Control-Allow-Origin`；带 `Origin: WEB_ORIGIN` 发 `text/plain` 的 POST 到 `/admin/...` → `40300`
- 反过来，带 `Origin: ADMIN_WEB_ORIGIN` 调 C 端接口同样不放行；`/administrators` 这类前缀相同的路径不算后台路径
- `/admin` 的响应带 `Cache-Control: private, no-store`，鉴权失败的 `40100` 也带
- 设置不存在的权限点、分配不存在的角色 → `40000`
- `ADMIN_JWT_SECRET` 和 `JWT_SECRET` 相同时，`env.ts` 校验失败

角色和管理员管理接口再加：
- 授予自己没有的权限、分配包含自己没有的权限的角色、改自己的角色 → `40300`
- 低权限管理员重置、停用、改角色高权限管理员或超级管理员 → `40300`
- 编辑、删除含有自己没有的权限的角色 → `40300`
- 删除系统权限点 → `40300`（超级管理员也一样）
- 后台先建一个 `is_system = false` 的权限点，再执行同 `key` 的种子迁移：变成 `is_system = true`，名称和分组保持后台的值
- 改完角色权限后，对方不用重新登录，下一次请求就按新权限判断
- 写操作和导出后 `admin_audit_logs` 里有对应记录

## 验收命令

在 API 项目根目录执行，并在回复里贴出输出：

```bash
# 每个后台路由都声明了权限；只应剩 GET /admin/me。路由写成多行时会误报，逐个确认
grep -rnE "\.(get|post|put|patch|delete)\(" src/routes/admin | grep -v "requirePermission"
# adminAuth 在创建后台路由时链式挂上（保证是第一条）；/admin 只挂这一个子路由
grep -n "new Hono<AdminEnv>().use('\*', adminAuth)" src/routes/admin/index.ts   # 应有 1 行
grep -rnE "route\('/admin" src                               # 应只有 app.route('/admin', admin)
# 入参和出参都不该有的字段，应为空
grep -rnE "passwordHash|password_hash|tokenVersion|token_version" src/schemas
# 只应出现在 /admin/me 等出参里，不能出现在入参 schema 里，逐个确认
grep -rnE "isSuper|is_super|isSystem|is_system" src/schemas
# 权限点的种子迁移不能用 DO NOTHING；有结果时确认不是 permissions 表的种子
grep -rn "DO NOTHING" src/db/migrations
# 代码引用了、但种子迁移里没有的权限点，应为空
comm -23 <(grep -rhoE "requirePermission\([^)]*\)" src | grep -oE "'[a-z_]+:[a-z_]+'" | sort -u) \
         <(grep -rhoE "'[a-z_]+:[a-z_]+'" src/db/migrations | sort -u)
```

在后台 Web 项目根目录执行（`API_DIR` 换成 API 项目的路径）：

```bash
# 菜单和 <Can> 用到了、但 API 种子迁移里没有的权限点，应为空
comm -23 <(grep -rshoE "permission(=|: ?)['\"][a-z_]+:[a-z_]+['\"]" app components lib | grep -oE "[a-z_]+:[a-z_]+" | sort -u) \
         <(grep -rhoE "'[a-z_]+:[a-z_]+'" "$API_DIR/src/db/migrations" | tr -d "'" | sort -u)
```

命令无法覆盖、需要在回复里说明的：
- 新增的权限点写进了种子迁移，并告诉用户去后台给需要的角色分配
- 后台写操作和导出都写了审计日志
- `ADMIN_JWT_SECRET`、`ADMIN_WEB_ORIGIN` 已加到 `env.ts` 和 `.env.example`
