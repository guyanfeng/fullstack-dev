# 安全审查

写接口、上传、渲染用户内容，或者用户要求做安全审查时读本文件。本文件只写其他文件没写的规则；已有的规则在下表，审查时一并对照，不在这里重复。

不在本文件范围内：支付、个人信息合规（隐私保护指引、注销等）、后台管理与角色权限。遇到时先问用户。

| 已有规则 | 位置 |
|---|---|
| 密钥只在服务端、服务端校验输入、报错不暴露内部信息 | `SKILL.md`「安全」 |
| 密码哈希、防暴力破解、账号不可枚举、cookie 属性、开放重定向、CORS、JWT | `auth.md` |
| 入参全部走 validator、出参经 schema 过滤内部字段、中间件顺序 | `api/guide.md` |
| 日志不记录密码、token、密钥、手机号 | `api/testing-deploy.md`「日志」 |
| `session_key` 不下发、AppSecret 不进日志、内容安全检测 | `miniprogram/api-pitfalls.md`、`api/wechat.md` |
| `NEXT_PUBLIC_` 不放密钥、token 不进 localStorage | `web/guide.md` |
| `.env` 不进 git，只提交 `.env.example` | `api/testing-deploy.md`「本地运行」 |

## 越权

- **只能本人读写的资源**（我的问卷、我的订单、草稿）按 id 读、改、删时，查询条件**同时带资源 id 和当前用户**（`c.get('userId')`），在 service 里一条查询完成，不要先查出来再在代码里比对。
- 查不到一律返回 `40400`，不区分「不存在」和「不是你的」，不让人探测别人的资源是否存在。
- **公开资源**（已发布的问卷、答题页、分享页）不套 `userId`，按它自己的访问规则校验（如 `status = 'published'`），不满足同样返回 `40400`。同一个资源的「作者管理」和「公开访问」分成两个接口，各自校验，不要在一个接口里用 if 分支混在一起。
- 个人数据的列表（我的问卷）只查当前用户的数据，不接受前端传来的 `userId` 作为过滤条件。
- 写入时归属字段（`userId`）从 token 取。入参 schema 里不出现 `userId`、`role`、`status` 这类归属和权限字段；状态变更走专门的接口（如 `POST /surveys/:id/publish`），在 service 里校验状态流转。
- 对外 id 用 UUID（见 `api/database.md`）只是不可枚举，不能代替上面的校验。
- 每个仅本人可访问的按 id 接口加一条测试：用户 B 请求用户 A 的资源，返回 `40400`。公开接口也要测：公开详情（答题页、分享页）请求未发布的资源返回 `40400`；公开列表里不出现未发布的资源。

## CSRF（API 服务 Web、用 cookie 鉴权时）

Web 的 cookie 设在 `.example.com`、`SameSite=Lax`（见 `auth.md`）。这个 cookie 会发给 `example.com` 下的**所有**子域，任何子域发起的请求也都会带上它。所以：

- `example.com` 下不能有不受我们控制的子域（第三方托管的落地页、客服系统、用户自定义子域等），也不能有能被上传内容执行脚本的子域。
- 公开文件、对象存储的自定义域名用**另一个主域名**（如 `example-static.com`），不用 `static.example.com`，否则它也会收到 token。

在这个前提下，JSON 请求会触发 CORS 预检，被精确 origin 挡住；但表单类请求（`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`）不预检，会直接到达 API。因此：

- API 用 Hono 自带的 `csrf` 中间件，只对**没有 `Authorization` 头**的请求生效（带 Bearer 的请求来自小程序或 Next 服务端，浏览器跨域无法伪造这个头）。小程序用 `uni.uploadFile` 上传到 API 中转时，请求是 multipart 且没有 `Origin`，**必须手动带 `Authorization`**（见 `miniprogram/api-pitfalls.md`「文件上传下载」），否则会被拒；直传对象存储的请求不带 API token：
  ```ts
  import { csrf } from 'hono/csrf'
  const csrfMw = csrf({ origin: [env.WEB_ORIGIN] })
  app.use('*', (c, next) => (c.req.header('Authorization') ? next() : csrfMw(c, next)))
  ```
  挂在 `cors` 之后、`auth` 之前。它只检查非 GET 的表单类请求，被拒时抛 403，由 `onError` 转成 `40300`。
- GET 请求不能有副作用（`SameSite=Lax` 下，别的站点通过链接跳转发起的 GET 也会带 cookie）。
- Next 的 `app/auth/login/route.ts`、`app/auth/logout/route.ts` 同理：先校验请求头 `Origin` 等于 `WEB_ORIGIN`，不等就拒绝。

## XSS

- 用户输入的内容一律用插值渲染（React 的 `{}`、Vue 的 `{{ }}`）。不用 `dangerouslySetInnerHTML`、`v-html`、`rich-text`、`mp-html` 渲染用户内容。需要用户输入富文本时，过滤方案先问用户。
- 用户填写的链接（个人主页、跳转地址等）在入参 schema 里限定协议，`javascript:`、`http:` 都会被拒（Zod 3 和 4 都能用；`z.url()` 只有 Zod 4 有）：
  ```ts
  z.string().url().refine((u) => u.startsWith('https://'), '链接必须以 https:// 开头')
  ```
- Web 默认不配 CSP：Next 的 nonce 方案要求页面全部动态渲染。有安全要求需要 CSP 时，和用户确认后按 Next 文档「Content Security Policy」配置。

## 限流

登录和注册的限流见 `auth.md`。下面这些接口也要限流，超限返回 `42900`：

- 调用有配额或收费的第三方接口的（内容安全检测、订阅消息等）。
- 签发上传凭证、发表内容、导出数据的。

计数方式和 `auth.md` 一样放进程内存。登录后的接口按 `userId` 计数，未登录的按 IP 计数（只信自己反向代理设置的头，如 `X-Real-IP`）。阈值按业务定，不清楚就问用户。

## 文件上传

- 存储路径由服务端生成（如 `<userId>/<uuid>.<扩展名>`），不用用户的原始文件名。
- 类型用白名单（按业务定，如 `jpg`、`png`、`webp`），不接受 `svg`、`html`、`xml` 这类浏览器会执行脚本的格式。客户端传来的扩展名和 `Content-Type` 都不可信，服务端要再校验。
- 直传对象存储时（见 `miniprogram/api-pitfalls.md`），凭证限定对象路径、大小上限和类型，有效期尽量短。上传请求只带签发的凭证，不带 API 的 token。
- 经过 API 中转时用 `hono/body-limit` 限制大小。它默认抛 `HTTPException(413)`，进入 `onError` 的通用映射，前端拿不到明确的错误码和中文提示，要传 `onError`：
  ```ts
  bodyLimit({ maxSize: 5 * 1024 * 1024, onError: () => { throw new BizError(40000, '文件不能超过 5MB') } })
  ```
- 只给本人看的文件放私有存储，用短期签名 URL 访问。公开文件用另一个主域名访问（见「CSRF」）。

## 数据库查询

- Drizzle 的查询构建器和 `` sql`...` `` 模板会把变量作为参数传，不会注入。`sql.raw()` 原样拼接，不能传入任何来自用户的值。
- 动态排序、动态列名用白名单映射（`{ createdAt: surveys.createdAt }[sort]`），不把用户传的字符串拼进 SQL。

## 服务端请求外部地址

- API 不请求用户提供的 URL（抓取链接预览、按 URL 下载图片等）。确实需要时先问用户，要限定域名白名单、禁止内网地址，并且不自动跟随重定向（`fetch` 传 `redirect: 'manual'`，否则白名单域名可以 302 到内网）。

## 依赖与密钥

- 锁文件（`bun.lock`、`package-lock.json`、`pnpm-lock.yaml`）要提交。
- `.env` 曾经提交过（下面的历史检查有结果）：里面的密钥全部轮换，再用 `git filter-repo` 清理历史。只删文件不轮换没有用，历史和别人的克隆里还在。
- 依赖漏洞审计：high 和 critical 级别的漏洞要升级修复；修不了的，在回复里说明为什么不受影响。国内镜像不支持审计接口（npmmirror 返回 404 / `NOT_IMPLEMENTED`），审计命令一律指定官方源：`npm audit` 用 `--registry`，`bun audit` 用环境变量 `BUN_CONFIG_REGISTRY`（会覆盖 `bunfig.toml` 里的 `registry`）。输出里出现 `error` 不算通过；出现 `skipped`（私有 scoped registry 没有审计接口时会这样）要在回复里列出没审计的包，不能当成通过。

## 验收命令

在对应项目根目录执行，并在回复里贴出输出。

所有任务都执行（从仓库根查，monorepo 的子目录里执行也能查到根目录的 `.env`）：
```bash
git -C "$(git rev-parse --show-toplevel)" ls-files | grep -E '(^|/)\.env' | grep -v '\.env\.example$'   # 应为空
git -C "$(git rev-parse --show-toplevel)" log --all --name-only --format= | grep -E '(^|/)\.env' | grep -v '\.env\.example$' | sort -u   # 应为空，有结果见「依赖与密钥」
```

下面的命令在改了接口、上传、用户内容的渲染、用户填的链接、依赖，或用户要求安全审查时执行：

API：
```bash
BUN_CONFIG_REGISTRY=https://registry.npmjs.org bun audit --audit-level=high
grep -rn "sql.raw(" src             # 有结果时逐个确认参数不来自用户
grep -rn "hono/csrf" src            # API 服务 Web（cookie 鉴权）时应有结果
```

Web：
```bash
npm audit --audit-level=high --omit=dev --registry=https://registry.npmjs.org   # 用 bun 的项目改用上面 API 的 bun audit 命令
grep -rn "dangerouslySetInnerHTML" . --include='*.tsx' --include='*.jsx' --exclude-dir=node_modules --exclude-dir=.next   # 应为空
```

小程序：
```bash
npm audit --audit-level=high --omit=dev --registry=https://registry.npmjs.org   # 用 bun 的项目改用上面 API 的 bun audit 命令；HBuilderX 项目没有锁文件时跳过并说明
grep -rnE "v-html|<rich-text|<mp-html" . --include='*.vue' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage   # 有结果时确认内容不是用户输入
```

命令无法覆盖、需要在回复里逐条说明的：
- 仅本人可访问的按 id 接口都在查询里带了当前用户，公开接口按访问规则校验，都有对应测试
- 入参 schema 里没有归属和权限字段
- 新增的上传、第三方调用、发表类接口有限流和大小限制
- GET 接口没有副作用
- Web 用 `.example.com` cookie 时：该主域下没有不受控的子域，公开文件和对象存储用另一个主域名
