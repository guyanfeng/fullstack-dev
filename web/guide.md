# Web 开发规范（H5 / PC，Next.js）

适用：Web 项目，Next.js App Router + TypeScript。H5 和 PC 是同一个应用，用响应式适配：移动端优先写样式，再用断点（Tailwind 的 `md:` / `lg:`）适配 PC。页面可能在手机浏览器、微信内置浏览器、PC 浏览器里打开。

## 按需继续读取

| 涉及 | 读取 |
|---|---|
| 微信内打开、微信里的分享卡片、小程序 web-view、业务域名校验文件 | `web/wechat-browser.md` |
| iOS/Android 表现不一致、键盘遮挡、滚动穿透、安全区、日期、音视频、1px | `web/mobile-compat.md` |
| 首屏慢、包体积大、图片、缓存、弱网 | `web/performance.md` |
| 页面、组件、样式、颜色字号间距、shadcn/ui 主题、加载/空/错误状态、toast 和确认弹窗 | `design.md` |

## 项目结构

新项目按这个模板建；现有项目以现状为准（见 `SKILL.md`「第零步：先看现状」）。

```
<项目根>/
├── app/
│   ├── layout.tsx        # viewport、全局样式、Provider
│   ├── (pages)/...       # 路由分组
│   └── auth/             # 只放写 cookie 的 Route Handler（账号密码登录、退出，见 auth.md）；有独立 API 项目时，业务接口一律放在 API 里
├── components/           # 分层见 design.md「组件化」
│   ├── ui/               # shadcn 生成的组件
│   ├── common/           # 通用组件：PageContainer、StateView、ConfirmDialog 等
│   └── <模块>/           # 业务组件
├── hooks/                # usePagedList 等
├── lib/
│   ├── request.ts        # 唯一的请求封装
│   └── env.ts            # 判断是否在微信内等
└── public/               # 放小程序业务域名校验文件等需要根路径访问的文件
```

## 写法约定

- 默认写 Server Component。只有用到 state、effect、事件、浏览器 API 时才加 `'use client'`，并且把 client 部分拆成尽量小的叶子组件。
- `window`、`document`、`localStorage`、`navigator.userAgent` 只能在 client 组件的 `useEffect` 或事件回调里访问。在渲染阶段访问会导致 SSR 报错或 hydration mismatch。
- 判断是否在微信内：服务端用 `headers()` 读 UA，或者客户端在 mount 之后判断。不要在渲染阶段用 `typeof window` 做分支来渲染不同的 UI。
- viewport 用 `export const viewport` 配置，不要手写 `<meta name="viewport">`：
  ```ts
  export const viewport: Viewport = {
    width: 'device-width',
    initialScale: 1,
    viewportFit: 'cover', // 才能用 env(safe-area-inset-*)
  }
  ```
  不要设置 `maximumScale: 1` 或 `userScalable: false`，这会禁止双指缩放，属于无障碍问题。iOS 输入框聚焦时放大的问题，用 16px 字号解决（见下文）。
- 样式默认用 Tailwind，设计稿宽 375，直接用 px。颜色、字号、圆角用 `design.md` 的语义 token，不写死色值。布局用 flex 和百分比做自适应，不做 rem 整体缩放。
- 环境变量：只有带 `NEXT_PUBLIC_` 前缀的才会进入客户端包，所以密钥绝不能用这个前缀。

## 网络请求

登录态怎么存、怎么带、未登录（`40100`）怎么处理，以 `auth.md` 为准；错误分流规则见 `SKILL.md`「响应与错误」。这里只写 Web 的实现要求：

- `lib/request.ts` 是唯一的请求封装：拼 API 地址、`credentials: 'include'`、超时（10 秒）、按 `code` 分流错误，HTTP 非 200 或 body 解析失败按通用错误处理。接口类型按 `SKILL.md`「类型来源」：和 API 同仓库时用 schema 的 `z.infer`，否则写在 `types/api.ts`。
- Server Component 取数据：在服务端用 `cookies()` 读 token，以 `Authorization` 头请求 API，把结果作为 props 传给 client 组件。不要在 client 里重复请求一次。
- 不把 token 放进 localStorage，也不在 client 代码里读写 token。

## 交互底线

### 通用与移动端
- 点击区域、加载态、空状态、出错重试、操作反馈见 `design.md`「交互规则」。
- 表单输入框 `font-size` 不小于 16px，否则 iOS 聚焦时会自动放大页面。
- 底部固定栏：新项目统一用 `BottomBar`（见 `design.md`「组件化」）；现有项目沿用已有写法，没处理安全区的补上。安全区的写法见 `web/mobile-compat.md`「安全区」。

### PC 端
- 布局按断点重新排（比如列表变成多列、加侧边导航），不是把移动端布局拉宽或放大。
- 内容区设最大宽度并居中（具体值按设计稿），宽屏上不要铺满整屏。
- hover 只能是增强，不能是唯一的触发方式：只有鼠标悬停才出现的菜单或操作按钮，在手机上点不出来。Tailwind v4 的 `hover:` 只在支持悬停的设备上生效；v3 项目需要时自己包一层 `@media (hover: hover)`。
- 键盘能完成所有操作：可点击的元素用 `<button>` 或 `<a>`，不用 `div` 加 `onClick`；不要去掉焦点样式，用 `focus-visible:` 给键盘焦点加明显的样式；弹窗打开时焦点移进弹窗，`Esc` 能关闭，关闭后焦点回到触发它的按钮。
- 宽表格包在 `overflow-x-auto` 的容器里，只让表格自己横向滚动，不要撑破整个页面。

## 验收命令

在 Web 项目根目录下执行，并在回复里贴出输出：

```bash
npx tsc --noEmit                       # 类型检查
npm run lint                           # 按项目实际脚本
npm run build                          # next build 能通过，才说明没有在渲染阶段访问 window/document
grep -rnE "NEXT_PUBLIC_[A-Z_]*(SECRET|KEY|TOKEN)" . --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='.env*' --exclude-dir=node_modules --exclude-dir=.next  # 命中的逐条确认是可以公开的值（如地图 key），不能是密钥
grep -rn "localStorage" app components lib | grep -i token   # 应为空
grep -rnE "<(div|span)[^>]*onClick" app components            # 命中的逐条确认，能改成 <button> 或 <a> 的要改
```

命令无法覆盖、需要在回复里提醒用户确认的：
- 在 iOS Safari 和微信内置浏览器里实际打开过（涉及微信能力时，对照 `web/wechat-browser.md`）
- 在 PC 宽屏（1440px 以上）下看过布局，并且只用键盘（Tab、Enter、Esc）走过一遍主要流程
- 底部固定元素处理了安全区，输入框字号不小于 16px
- 加载态、空状态、错误态都有
