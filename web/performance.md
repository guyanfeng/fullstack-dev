# Web 性能

目标：在 4G 中低端安卓机上，首屏内容 2 秒内可见。

## JS 体积

- 默认用 Server Component，只有交互部分才是 client 组件。client 组件越少，要下载和执行的 JS 就越少。
- 只在部分页面用到的重型组件（图表、富文本编辑器、地图、二维码生成），用 `next/dynamic` 动态加载，并设置 `ssr: false` 和加载占位。
- 用 `@next/bundle-analyzer` 检查产物，找出意外被打进来的大依赖。
- 不要整包引入工具库（`lodash` 改成按函数引入，`moment` 换成 `dayjs`）。
- 第三方脚本（统计、客服）用 `next/script` 的 `lazyOnload`，不要阻塞首屏。

## 数据获取

- 首屏数据在服务端获取，直接渲染出 HTML，不要等客户端 JS 加载完再请求。
- 用 `loading.tsx` 和 `<Suspense>` 做流式渲染，慢的部分先显示骨架屏。
- 多个互不依赖的请求用 `Promise.all` 并行，不要串行 `await`。
- 合理使用 Next.js 的缓存（`fetch` 的 `revalidate`），但登录用户的个人数据不能被缓存成公共数据。

## 图片

- 用 `next/image`，必须设置 `width` / `height` 或 `fill`，避免布局抖动。
- 设置 `sizes`，让移动端加载合适尺寸的图片。
- 只有首屏最重要的那一张图加 `priority`。
- 如果图片走 CDN 并自带图片处理，可以用自定义 `loader` 对接 CDN 的处理参数，不必经过 Next.js 的图片优化服务。

## 缓存

- HTML：`Cache-Control: no-cache`（特别是在微信里，见 `web/wechat-browser.md`）。
- 带 hash 的静态资源（`/_next/static/`）：`public, max-age=31536000, immutable`，Next.js 默认已经这样处理。
- `public/` 下没有 hash 的文件，不要设置长缓存。

## 弱网

- 请求设置超时（建议 10 秒），超时后显示可以重试的错误状态，不要一直转圈。
- 提交类操作在弱网下要防止重复提交（按钮禁用 + 服务端幂等）。
- 避免首屏依赖多个串行请求。

## 检查方式

- Chrome DevTools 用「Slow 4G」+ 4 倍 CPU 降速来模拟。
- 看 Lighthouse 的移动端报告，重点关注 LCP、CLS、INP。
- 真机测试：至少在一台中低端安卓机的微信里打开看看。
