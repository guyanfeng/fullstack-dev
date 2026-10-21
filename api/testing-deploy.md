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
- 每个新增接口至少覆盖：正常路径、参数校验失败、未登录或无权限。
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
- 提供 `GET /health` 健康检查接口，用于负载均衡探活和发布后确认。
- 启动时先校验环境变量（`env.ts`），缺失就直接退出，不要带着错误配置运行。
- 迁移作为独立步骤执行（部署流水线里在启动新版本之前执行；服务器上没有 Bun 和源码，`drizzle-kit migrate` 在 CI 或有源码的机器上连生产库执行），不要在每个实例启动时自动执行。`db/manual/` 下需要在事务外执行的 SQL 也在这一步执行（见 `api/database.md`）。
- 处理 `SIGTERM`：停止接收新请求，等待正在处理的请求完成后再退出。
- 多实例部署时，需要共享的状态放在数据库里，锁用数据库行锁，不能放在进程内存里。不引入 Redis。登录失败计数例外，放在进程内存里（见 `auth.md`「账号和密码」）。
- 在 Nginx 或负载均衡后面时，从 `X-Forwarded-For` 获取真实 IP，并且只信任自己代理加的头。
- 微信消息推送地址必须是公网可访问的 HTTPS 地址，不能带鉴权中间件（它靠验签来保证安全）。
