# 登录与鉴权

这里是登录方式和登录态的唯一定义。各类项目的 guide 只负责实现，不另外定义。

## 用户体系

- **默认每个项目独立**：一个项目有自己的 API 和自己的 `users` 表，不和其他项目共享用户。
- 登录方式按项目类型定：
  - 小程序项目：**微信登录**（见下文「小程序：微信登录」）。
  - Web 项目（H5/PC）：**账号密码登录**（见下文「Web：账号密码登录」）。
- 用户表只有一张 `users`，登录凭证直接放在上面：账号密码项目放 `username`（规范化后的用户名）和 `password_hash`；小程序项目放 `openid`。登录字段都加唯一约束。
- **一个 API 同时服务多个端**（比如同一个后端既给小程序用，也给 Web 用）的情况按项目而定。遇到时先和用户确认，确认要共用用户表时，仍然是这一张 `users`，两种凭证的字段都加上、都允许为空：
  ```
  users(id, user_id, username, password_hash, openid, nickname, avatar, token_version, created_at, updated_at, ...)
    unique(username)、unique(openid)    -- PostgreSQL 的唯一约束允许多行为 NULL
    check (username is not null or openid is not null)
    check ((username is null) = (password_hash is null))
  ```
  两种身份默认不互相关联：小程序里新建的用户 `username` 为空，Web 注册的账号 `openid` 为空。需要「同一个人两边通用」时，绑定流程和已有数据怎么合并要按业务单独设计，先问用户，不要自行实现。

## 小程序：微信登录

| 环节 | 小程序 | API |
|---|---|---|
| 登录 | `uni.login` → `POST /auth/mp-login { code }` | 用 code 换 openid（见 `api/wechat.md`），按 openid 找到或新建用户，签发 token |
| token 存放 | `uni.setStorageSync('token')` | 签发 JWT，不存储（见「token（JWT）」） |
| 发请求时怎么带 | `Authorization: Bearer <token>` | 鉴权中间件读 `Authorization` |
| 未登录（`code: 40100`）时 | 清 token → 重新 `uni.login` 静默登录 → 重放原请求（并发请求只登录一次） | 返回 `ErrorRes { code: 40100 }` |

- 小程序的请求不受 CORS 限制，API 不需要为它配置 CORS。

## Web：账号密码登录

### 账号和密码
- **用户名**：去掉首尾空格并转成小写后再存储和查询，加唯一约束。长度和字符集规则写在 zod schema 里，注册和登录共用。
- **密码**：至少 8 位。只存哈希，用 `Bun.password.hash`（默认 argon2id）生成，用 `Bun.password.verify` 校验。明文密码不能入库、不能写日志、不能出现在任何响应里。
- **登录失败**：账号不存在和密码错误统一返回 `{ code: 40010, message: '账号或密码错误' }`，不能让人分辨出账号是否存在。这里不用 `40100`，是因为前端遇到 `40100` 会走「清登录态、跳登录页」的流程。
- **防暴力破解**：同一个用户名连续失败 5 次，锁定 15 分钟，期间返回 `{ code: 42901 }`，这一层在 API 里做。同一个 IP 的失败次数也要计数限流，**这一层在 Next 的 `app/auth/login/route.ts` 里做**：登录请求经过 Next 转发，API 看到的是 Next 服务器的 IP，在 API 按 IP 限流等于所有人共用一个计数。Next 取客户端 IP 时只信自己的反向代理加的头（如 Nginx 设置的 `X-Real-IP`），不信客户端能伪造的值。两层计数都直接放在进程内存里（一个 `Map`，记录失败次数和解锁时间），不引入 Redis；过期的记录要定期清理，避免内存一直增长。进程重启后计数清零，可以接受。多实例部署时每个实例各自计数，实际允许的尝试次数会变成「实例数 × 5」，如果需要严格限制，再和用户确认要不要改成存数据库。
- **修改密码**：必须校验旧密码。改完后把该用户的 `token_version` 加 1，让其他已签发的 token 全部失效（见「token（JWT）」），再给当前设备签发新 token。
- **注册**：是开放注册还是由后台创建账号，按需求定，不清楚就问用户。开放注册时，注册接口同样要按 IP 限流。
- **找回密码**：暂不做。需要时由用户指定方案。

### 部署前提
- Web 和 API 放在**同一个主域名的不同子域名**下，例如 `m.example.com` 和 `api.example.com`。这样它们属于同站（same-site），cookie 可以用 `SameSite=Lax`。
- **不要**把 Web 和 API 放在不同的主域名下。跨站时 cookie 必须设置 `SameSite=None; Secure`，并且会被 iOS Safari 的 ITP 等第三方 cookie 限制拦截，登录态会不稳定。如果确实没办法做到同站，就让 Web 通过 Next.js 的 rewrites 同源转发到 API，客户端不直接跨域请求。

### 登录态

| 环节 | 浏览器端 | Next 服务端 | API |
|---|---|---|---|
| 登录 | 登录页 `/login` 的表单提交到同源的 `POST /auth/login` | `app/auth/login/route.ts` 转发 `POST /auth/password-login { username, password }` 给 API，拿到 token 后写 cookie | 校验密码，签发 token |
| token 存放 | httpOnly cookie `token`（JS 读不到） | 由 Route Handler 写 cookie：`HttpOnly; Secure; SameSite=Lax; Domain=.example.com; Path=/` | 签发 JWT，不存储（见「token（JWT）」） |
| 发请求时怎么带 | `fetch(url, { credentials: 'include' })`，浏览器自动带 cookie | 用 `cookies()` 读出 token，以 `Authorization: Bearer` 请求 API（服务端请求不会自动带浏览器 cookie） | 鉴权中间件**先读 `Authorization`，没有再读 cookie `token`** |
| 未登录（`code: 40100`）时 | 跳转 `/login?redirect=<当前路径>` | `redirect('/login?redirect=<当前路径>')` | 返回 `ErrorRes { code: 40100 }` |
| 退出 | 请求同源的 `POST /auth/logout` | `app/auth/logout/route.ts` 删除 cookie，删除时的 `Domain` 和 `Path` 必须和写入时完全一致（否则删不掉） | JWT 不在服务端作废；要让该用户所有设备下线时，把 `token_version` 加 1 |

- 登录成功后跳回 `redirect`，必须先校验是本站地址，防止开放重定向。**不要用字符串前缀判断**（`/\evil.com`、`/<Tab>/evil.com` 都以 `/` 开头且不以 `//` 开头，但浏览器会解析成 `https://evil.com`），用 URL 解析后比较 origin：
  ```ts
  const u = new URL(redirect ?? '/', origin)   // origin 用配置里的 WEB_ORIGIN，不要取请求头
  const target = u.origin === origin ? u.pathname + u.search + u.hash : '/'
  ```
- 登录页不能被缓存成公共页面，登录表单只通过 HTTPS 提交。
- 本地开发（`localhost`）时 cookie 不设 `Domain`（`.example.com` 在 localhost 上不生效）。cookie 的 `Domain`、`Secure` 从环境变量读，不要写死。Chrome 和 Firefox 允许 localhost 上的 `Secure` cookie，Safari 不一定，本地调试登录用 Chrome。

### API 侧的 CORS

```ts
app.use('*', cors({
  origin: [env.WEB_ORIGIN],    // 精确写出 Web 的 origin，不能用 '*'
  credentials: true,           // 允许带 cookie
}))
```

- 使用 `credentials: true` 时，`origin` 不能是 `*`，否则浏览器会拒绝。

## 职责边界

- 用到 AppSecret、access_token 的逻辑，以及密码校验和哈希，只在 API 里实现。
- Web 的 Route Handler（`app/auth/`）只负责写 cookie、删 cookie 和登录的 IP 限流，**不校验密码，也不持有任何密钥。**

## token（JWT）

- 登录 token 默认用 JWT，用 `hono/jwt` 的 `sign` / `verify`，算法 HS256。
- 签名密钥 `JWT_SECRET` 放环境变量，至少 32 字节的随机串，在 `env.ts` 里校验长度。换密钥会让所有人下线。
- payload 只放 `{ sub: userId, ver: tokenVersion, exp }`，不放手机号、openid 等个人信息（JWT 只是签名，内容谁都能解码）。
- `users` 上加 `token_version integer not null default 0`。鉴权中间件验签通过后，再按 `sub` 查一次 `users.token_version` 和 `ver` 比对，不一致就返回 `40100`。这是一次主键查询，换来的是「改密码、强制下线」能立刻生效。
- 默认**不做 refresh token**，`exp` 设为 7 天左右。
- 小程序过期后走 `40100` 静默重新登录，用户无感知。
- Web 过期后要重新输入账号密码。需要更长的登录保持（比如「记住我」）时再单独设计。
- 有特殊安全要求（后台管理、资金操作）时再单独设计。
