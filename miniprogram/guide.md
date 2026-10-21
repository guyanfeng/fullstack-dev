# 微信小程序开发规范（uni-app）

适用：uni-app（Vue 3 + Vite + TypeScript），只发布到微信小程序（`MP-WEIXIN`）。

## 按需继续读取

只有任务涉及下列内容时才读对应文件：

| 涉及 | 读取 |
|---|---|
| 登录、手机号、用户信息、订阅消息、分享、文件上传下载 | `miniprogram/api-pitfalls.md` |
| 包体积超限、分包、首屏慢、列表卡顿、内存告警 | `miniprogram/performance.md` |

## 项目结构

新项目按这个模板建；现有项目以现状为准（见 `SKILL.md`「第零步：先看现状」）。

```
src/
├── pages/            # 主包页面（tabBar 页面必须在主包）
├── subpkg-*/         # 分包，按业务模块拆，如 subpkg-survey/
├── components/       # 全局组件，见下方 easycom 规则
├── composables/      # useXxx 组合式函数
├── stores/           # Pinia
├── services/         # 接口请求，按业务模块一个文件
├── utils/
│   └── request.ts    # 唯一的 uni.request 封装
├── static/           # 只放小图标，大图走 CDN
├── pages.json
└── manifest.json
```

- 新页面必须同时在 `pages.json` 注册，否则编译不报错但运行时白屏。
- **easycom 免注册**只对 `components/组件名/组件名.vue` 这种目录结构生效（例如 `components/SurveyCard/SurveyCard.vue`）。写成 `components/SurveyCard.vue` 不会被自动注册，必须手动 import。现有项目沿用它已有的方式。
- 非 tabBar、非首屏的页面默认放分包。

## 写法约定

- 使用 `<script setup lang="ts">` + 组合式 API。不写选项式 API。
- 页面生命周期从 `@dcloudio/uni-app` 引入：`onLoad`、`onShow`、`onReachBottom`、`onPullDownRefresh`、`onShareAppMessage`。不要用 Vue 的 `onMounted` 去拿页面参数，参数只在 `onLoad(query)` 里拿。
- 调用 API 优先用 `uni.*`。只有 uni 没有封装的微信能力才用 `wx.*`，并包在条件编译里：
  ```ts
  // #ifdef MP-WEIXIN
  wx.xxx()
  // #endif
  ```
- 尺寸单位用 `rpx`（设计稿宽 750）。1px 边框直接写 `1px`，不要写 `1rpx`，部分机型会消失。
- 状态管理：
  - **只有跨页面共享的数据才放 Pinia**（`stores/`），比如当前用户信息、购物车数量。
  - 只在一个页面里用的数据放在页面自己的 `ref` / `reactive` 里，不进 store。
  - 列表数据每次进入页面重新请求，不缓存在 store 里。
  - 需要持久化的少量数据（token、用户基础信息）用 `uni.setStorageSync`，启动时读出来放回 store。storage 总量不超过 10MB，不存大对象和列表。

## 小程序和 Web 的差异（最常出错的地方）

- 没有 DOM 和 `window` / `document`。不能用依赖 DOM 的库（大部分 Web UI 库、富文本编辑器、直接操作 DOM 的图表库）。引入任何 npm 包前先确认它支持小程序。
- 模板里不支持 `<component :is>` 动态组件，用 `v-if` 分支代替。
- 不支持 `v-html`，富文本用 `<rich-text>` 或 `mp-html` 之类的组件。
- 样式：不能用 `*` 通配选择器；组件样式默认隔离，跨组件改样式用 `:deep()` 或 `options: { styleIsolation: 'shared' }`，谨慎使用。
- 页面栈最多 10 层。连续跳转详情页用 `uni.redirectTo`，回到列表用 `uni.navigateBack`，不要无限 `navigateTo`。
- 跳 tabBar 页面只能用 `uni.switchTab`，并且不能带 query 参数（要传参用 store 或 storage）。
- 所有请求域名必须是 HTTPS，并在小程序后台配置合法域名。开发者工具勾了"不校验合法域名"时能跑，真机会失败，这个要提醒用户。

## 网络请求

登录态和未登录（`40100`）的处理以 `auth.md` 为准，错误分流规则见 `SKILL.md`「响应与错误」。这里只写小程序的实现要求：

- 只通过一个请求封装发请求（新项目放在 `utils/request.ts`；现有项目用它已有的封装，没有就先问用户要不要加）。
- 封装负责：拼 baseURL、带 `Authorization` 头、超时、按 `code` 分流、`40100` 时静默重新登录并重放请求（并发请求只触发一次登录）。
- 注意：`uni.request` 网络失败时走 `fail`，HTTP 非 200（比如网关返回的 502）仍然走 `success`，要先判断 `statusCode === 200`，再看 `code`。

## 性能底线

完整内容见 `miniprogram/performance.md`，以下几条每次都要遵守：

- 主包不超过 2MB，单个分包不超过 2MB，总包不超过 20MB。
- 长列表分页加载；超过 100 条考虑虚拟列表。
- 响应式数据只放模板需要的字段，不要把整个接口返回的大对象塞进 `ref`，每次改动都会序列化后传给渲染层。

## 验收命令

在小程序项目根目录下执行，并在回复里贴出输出：

```bash
npx vue-tsc --noEmit                    # TS 项目；JS 项目跳过并说明
grep -rnwE "window|document|localStorage|sessionStorage" src --include=*.vue --include=*.ts --include=*.js   # 应为空，有结果要逐条说明
grep -rn "wx\." src --include=*.vue --include=*.ts --include=*.js   # 每处都要在 #ifdef MP-WEIXIN 内
npm run build:mp-weixin                 # 发行构建，必须能通过
du -sh dist/build/mp-weixin             # 总包体积（HBuilderX 项目的产物在 unpackage/dist/build/mp-weixin）
find dist/build/mp-weixin -maxdepth 1 -mindepth 1 -type d -name 'subpkg-*' -exec du -sh {} \;   # 各分包
find dist/build/mp-weixin -mindepth 1 -maxdepth 1 ! -name 'subpkg-*' -exec du -sk {} + | awk '{s+=$1} END {printf "%.2f MB\n", s/1024}'   # 近似主包体积，需 ≤ 2MB（macOS / Linux 通用；精确值以开发者工具「代码依赖分析」为准）
```

命令无法覆盖、需要在回复里提醒用户确认的：
- 新页面已注册到 `pages.json`，放在正确的主包或分包
- 订阅消息、手机号、头像昵称等 API 是由按钮点击触发的
- 在 iOS 和 Android 真机上预览过，并且关闭了「不校验合法域名」
