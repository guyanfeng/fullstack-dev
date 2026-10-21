# API 开发规范（Hono + Bun）

适用：API 项目（或仓库里的 API 目录），Hono + TypeScript，用 Bun 运行。

## 按需继续读取

| 涉及 | 读取 |
|---|---|
| 用户表、小程序登录和账号密码登录的流程、注册、token、鉴权中间件 | `auth.md` |
| 小程序登录、手机号、access_token、内容安全检测 | `api/wechat.md` |
| 建表、改表、迁移、查询、事务、索引 | `api/database.md` |
| 写测试、本地运行、部署、日志、环境变量 | `api/testing-deploy.md` |

> 数据库默认用 **Drizzle + PostgreSQL**。如果项目实际用的不是这套，以项目现有代码为准，`api/database.md` 里的原则仍然适用。

## 项目结构

新项目按这个模板建；现有项目以现状为准（见 `SKILL.md`「第零步：先看现状」）。

```
src/
├── index.ts            # 入口：组装 app，export default
├── app.ts              # 创建 Hono 实例、挂全局中间件、挂路由、onError
├── schemas/            # 每个接口的入参、出参 zod schema，pagination.ts 也在这里（和前端同仓库时放仓库根的 shared/）
├── errors.ts           # 错误码表（见 SKILL.md「响应与错误」）
├── env.ts              # 用 zod 校验环境变量，启动时校验失败直接退出
├── routes/             # 每个业务模块一个文件，只负责参数校验和调用 service
│   └── surveys.ts
├── services/           # 业务逻辑，不依赖 Hono 的 Context
├── db/
│   ├── schema.ts
│   └── client.ts
├── middlewares/        # auth、requestId 等
└── lib/                # 微信 SDK 封装、工具函数
```

分层规则：`routes` → `services` → `db`。service 不接收 `c`（Context），只接收普通参数，这样才能单独测试和复用。

## 写法约定

### 入口
```ts
// index.ts
import { app } from './app'
import { env } from './env'

export default { port: env.PORT, fetch: app.fetch }
```

### 路由
- 每个模块 `new Hono()` 后用 `app.route('/surveys', surveys)` 挂载。
- 示例写法：
  ```ts
  export const surveys = new Hono<AppEnv>()
    .get('/', zValidator('query', ListQuery), async (c) => { ... })
    .post('/', zValidator('json', CreateSurvey), async (c) => { ... })
  ```
- 所有入参（`json`、`query`、`param`、`form`）都必须经过校验，schema 从 `src/schemas/` 引入。handler 里只用 `c.req.valid('json')` 读取，不用 `c.req.json()`。
- `@hono/zod-validator` 默认的失败响应不是 `ErrorRes` 格式。在 `lib/validator.ts` 里包一层，传入 hook，把校验失败统一转成 `{ code: 40000, message }`（HTTP 200，见 `SKILL.md`「响应与错误」），所有路由都用这个包装后的 `zValidator`。
- 出参：统一用 `lib/response.ts` 的 `ok(c, Schema, data)` 返回，它先 `Schema.parse(data)` 再返回 `c.json({ code: 0, data })`。zod 默认会去掉 schema 里没有的字段，所以这一步同时防止内部字段（openid、password_hash 等）泄露；返回的数据和 schema 对不上时会抛错，进 `onError` 返回 `50000` 并写日志，schema 和实现不一致能立刻被发现。不导出 `AppType`，前端怎么拿类型见 `SKILL.md`「类型来源」。

### Context 类型
在 `AppEnv` 里声明 `Variables`（例如 `userId`、`requestId`），中间件用 `c.set` 写入，handler 用 `c.get` 读取。不要用 `any` 绕过类型。

### 错误处理
- 错误响应格式和错误码表见 `SKILL.md`「响应与错误」，错误码统一从 `src/errors.ts` 引用，不在各个模块里另外定义。
- 业务错误抛 `BizError(code, message)`（`lib/errors.ts`）。
- `app.onError` 统一转成 HTTP 200 的 `ErrorRes { code, message, requestId }`：`BizError` 按它自己的 code 返回；`HTTPException`（包括 hono 中间件抛的，如 jwt 校验失败）按它的 status 映射到对应的通用码（401 → `40100` 等）；其他未知错误返回 `{ code: 50000 }` 和通用文案，完整错误只写进日志。
- `app.notFound` 返回 `{ code: 40400, message: '接口不存在' }`。
- 不要在每个 handler 里写 try/catch 再返回错误 JSON，交给 `onError` 统一处理。

### 中间件顺序
`requestId` → `logger` → `secureHeaders` → `cors`（配置见 `auth.md`：精确的 Web origin + `credentials: true`）→ 路由级 `auth`（先读 `Authorization`，再读 cookie）。
鉴权中间件挂在需要登录的路由组上，不要全局挂载后再写一堆排除路径。

### Bun 相关
- 环境变量统一从 `env.ts` 导出的对象读取，业务代码不直接读 `Bun.env` 或 `process.env`。
- 能用 Web 标准 API（`fetch`、`crypto.subtle`、`Request`）的就用标准 API，少依赖 `node:*` 模块。

## 接口设计

- RESTful：`GET /surveys`、`POST /surveys`、`GET /surveys/:id`、`PATCH /surveys/:id`、`DELETE /surveys/:id`。
- HTTP 一律 200，错误码按 `SKILL.md` 的错误码表。
- 列表接口必须分页，从 `SKILL.md`「分页（两种，按场景选）」的两种里选一种。
- 只能操作自己资源的接口，在 service 里校验归属（`where userId = ?`），不能只靠前端隐藏按钮。
- 写接口考虑幂等：提交答卷、消息推送回调之类的场景，用唯一约束或幂等键防止重复写入。

## 验收命令

在 API 项目根目录下执行，并在回复里贴出输出：

```bash
bunx tsc --noEmit                        # 类型检查
bun test                                 # 新增或修改的接口必须有测试，见 api/testing-deploy.md
grep -rn "c.req.json()" src              # 应为空，入参一律走 validator
grep -rnE "process\.env|Bun\.env" src | grep -v "src/env.ts"   # 应为空
bunx drizzle-kit generate                # 改了 schema 时执行；没改表时输出应为 "No schema changes"
```

改了表结构时，贴出新生成的迁移 SQL，并标出其中的删列、改类型、改名操作（见 `api/database.md`）。

命令无法覆盖、需要在回复里说明的：
- 需要登录的接口挂了 auth，并在 service 里校验了资源归属
- 新增的环境变量已加到 `env.ts` 和 `.env.example`
