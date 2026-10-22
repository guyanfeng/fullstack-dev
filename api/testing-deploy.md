# 测试、运行与部署

## 本地运行

- 开发：`bun --hot src/index.ts`（或者 package.json 里对应的 `dev` 脚本）。
- Bun 会自动加载 `.env`，不需要 dotenv。
- `.env` 不提交到 git；`.env.example` 要提交，并且和 `env.ts` 的 schema 保持同步。

## 测试

用 `bun test`。

### 接口测试
不需要启动真实的 HTTP 服务，直接调用 `app.request`：
```ts
import { describe, expect, test } from 'bun:test'
import { app } from '../src/app'

test('创建问卷需要登录', async () => {
  const res = await app.request('/surveys', { method: 'POST', body: JSON.stringify({}) })
  expect(res.status).toBe(200)
  expect((await res.json()).code).toBe(40100)
})
```
需要类型提示时，可以用 `hono/testing` 的 `testClient(app)`。

### 测试什么
- 每个新增接口至少覆盖：正常路径、参数校验失败、未登录或无权限；仅本人可访问的按 id 接口再加一条越权测试；公开接口测未发布的资源：详情返回 `40400`，列表里不出现（见 `security.md`「越权」）。
- 业务规则（资源归属、幂等、状态流转）要单独测 service。
- 签名、验签、金额计算这类不能出错的逻辑，必须有单元测试。

### 外部依赖
- 测试中**不能**请求真实的微信接口。`lib/wechat` 的函数要支持注入 `fetch` 或者在测试中被替换，返回固定数据。
- 数据库用单独的测试库：测试开始前执行迁移，每个测试之间清空相关表（或者每个测试在事务里执行，结束后回滚）。

## 日志

- 输出结构化 JSON 日志，每条带上 `requestId`。
- **不能**记录：密码（包括登录失败时的请求体）、token、AppSecret、密钥、完整手机号、`session_key`。openid、手机号需要记录时要脱敏。
- 错误日志带上请求路径、用户 id、错误堆栈，便于排查。

## 部署

不用 Docker。部署的是 `bun build --compile` 编译出的单个可执行文件，用 systemd 守护。服务器上不需要装 Bun，也不需要 `node_modules`。

### 编译
```bash
bun build src/index.ts --compile --minify --sourcemap \
  --target=bun-linux-x64 --no-compile-autoload-dotenv --outfile dist/server
```
- `--target` 按服务器选：`bun-linux-x64`、`bun-linux-arm64`。在 macOS 上编译也能直接产出 Linux 文件。服务器 CPU 较老、启动报 `Illegal instruction` 时，改用 `bun-linux-x64-baseline`。
- `--no-compile-autoload-dotenv`：编译出的文件默认会读取工作目录下的 `.env`，关掉它，环境变量只从 systemd 的 `EnvironmentFile` 来，避免两处来源。
- `--sourcemap` 会把 sourcemap 嵌进可执行文件，报错堆栈显示源码的文件和行号；同时生成的 `.map` 文件不用上传。
- 编译用的 Bun 版本就是线上运行的版本，锁定版本号（CI 里指定版本，不用 `latest`）。

### systemd
```ini
# /etc/systemd/system/api.service
[Unit]
Description=api
After=network.target

[Service]
# 专用的非 root 用户
User=api
WorkingDirectory=/opt/api
ExecStart=/opt/api/server
# 权限 600，不进 git
EnvironmentFile=/etc/api/env
Restart=always
RestartSec=3
# 留时间处理完正在进行的请求（见下面的 SIGTERM）
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```
- systemd 的注释只能单独占一行，不能写在值的后面（`User=api # xxx` 会把 `# xxx` 当成值的一部分）。
- 发布：上传新文件到 `/opt/api/server.new` → `chmod +x` → `mv` 覆盖 `server` → `systemctl restart api` → 请求 `/health` 确认。不要直接覆盖正在运行的文件。
- 日志直接输出到 stdout，用 `journalctl -u api` 查看，不在应用里写日志文件。
- 改了 `.service` 文件要先 `systemctl daemon-reload`；首次部署 `systemctl enable --now api`。

### 其他
- 提供 `GET /health` 健康检查接口。服务只监听本机或内网地址，所以负载均衡探活请求的是 Nginx 的 `/health`，由 Nginx 转发给 API；发布后在服务器上用 `curl http://<HOST>:<PORT>/health` 确认（同机部署就是 `127.0.0.1`）。
- 启动时先校验环境变量（`env.ts`），缺失就直接退出，不要带着错误配置运行。
- 迁移作为独立步骤执行（部署流水线里在启动新版本之前执行；服务器上没有 Bun 和源码，`drizzle-kit migrate` 在 CI 或有源码的机器上连生产库执行），不要在每个实例启动时自动执行。`db/manual/` 下需要在事务外执行的 SQL 也在这一步执行（见 `api/database.md`）。
- 处理 `SIGTERM`：停止接收新请求，等待正在处理的请求完成后再退出。
- 多实例部署时，需要共享的状态放在数据库里，锁用数据库行锁，不能放在进程内存里。不引入 Redis。登录失败计数例外，放在进程内存里（见 `auth.md`「账号和密码」）。
- **真实 IP（限流、审计日志都按这里取，API 和 Next 一样）**：只读 `X-Real-IP`，由自己的 Nginx 设置 `proxy_set_header X-Real-IP $remote_addr;`（会覆盖客户端自己带的同名头）。
  - **API 和 Next 只允许自己的 Nginx 访问**，端口不对公网开放，否则客户端可以绕过 Nginx，自己带一个 `X-Real-IP`，限流和审计日志都能伪造。按 Nginx 的位置二选一：
    - **Nginx 和服务在同一台机器**：API 的 `HOST` 用默认的 `127.0.0.1`（见 `api/guide.md`「入口」），Next 用 `next start -H 127.0.0.1`。
    - **Nginx 在另一台机器**：不能绑 `127.0.0.1`（远程的 Nginx 连不上本机回环地址）。`HOST` 和 `-H` 设成本机的内网 IP，再用防火墙让这个端口只放行 Nginx 的内网地址。
  - 应用代码不读 `X-Forwarded-For`：它最左边的值是客户端自己能填的。
  - Nginx 前面还有 CDN 或负载均衡时，`$remote_addr` 是它们的地址，要先在 Nginx 里按它们的文档用 realip 模块还原（`set_real_ip_from` 填 CDN/LB 的地址段，`real_ip_header` 按文档可以是 `X-Forwarded-For`），还原后的 `$remote_addr` 仍然只通过 `X-Real-IP` 传给应用。不清楚部署结构就问用户。
  - 取不到（本地开发没有 Nginx）时按 `'unknown'` 处理，不回退去读别的头。
- 微信消息推送地址必须是公网可访问的 HTTPS 地址，不能带鉴权中间件（它靠验签来保证安全）。
