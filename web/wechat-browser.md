# 在微信里打开的 Web 页面（微信内置浏览器、web-view）

只在 Web 页面会在微信内置浏览器里打开，或者会被小程序的 `<web-view>` 打开时才需要读。

不使用 JSSDK（`wx.config`、自定义分享、开放标签），因此不需要服务号和 jsapi_ticket。

## 环境判断

```ts
export const isWeChat = (ua: string) => /MicroMessenger/i.test(ua)
export const isWeChatMiniProgram = (ua: string) => /miniProgram/i.test(ua) // 在小程序 web-view 里
```

UA 判断按 `web/guide.md` 的规则来：服务端读 headers，或者客户端 mount 之后再判断。

## 分享

- 在微信里分享网页，卡片显示的是页面的 `<title>` 和链接，不能自定义描述和缩略图（需要 JSSDK，不在本 skill 范围内）。所以每个页面都要设置有意义的 `<title>`（Next 的 `metadata.title`）。
- 分享出去的链接要带上恢复页面需要的参数，但不要带 token 之类的登录凭证。

## 在小程序 web-view 里运行时

Web 页面被小程序的 `<web-view>` 打开时，和小程序之间的通信能力很有限，设计功能前要先确认能不能实现。

### 判断与准备
- `wx.miniProgram.*` 来自 `jweixin-1.6.0.js`：用 `next/script` 加载 `https://res.wx.qq.com/open/js/jweixin-1.6.0.js`（`strategy="afterInteractive"`，只在小程序 web-view 里加载）。这些接口不需要 `wx.config`，也不需要签名。
- 用 `wx.miniProgram.getEnv(res => res.miniprogram)` 判断是否在小程序里。UA 里包含 `miniProgram` 也可以作为判断依据。

### H5 → 小程序
- **跳转**：`wx.miniProgram.navigateTo / redirectTo / switchTab / reLaunch / navigateBack`，目标是小程序的页面路径，可以带 query。这是 H5 通知小程序「立刻做某件事」的**唯一可靠方式**。
- **postMessage 不是实时的**：`wx.miniProgram.postMessage({ data })` 发出的消息，小程序不会马上收到。只有在**小程序后退、web-view 组件销毁、分享、复制链接**这些时机，`<web-view @message>` 才会一次性收到之前所有消息（`e.detail.data` 是数组）。
  - 适合：分享时带上 H5 当前的状态（标题、路径），离开页面时回传结果。
  - 不适合：点击按钮后让小程序立刻响应。这种场景要用跳转（例如跳到小程序的某个页面，用 query 带参数），或者经过服务端中转。

### 小程序 → H5
- 只能通过 web-view 的 `src` 传参（修改 URL），没有向 H5 实时发送消息的接口。
- 需要 H5 刷新数据时，可以在 H5 里监听 `visibilitychange` 或 `pageshow`，在页面重新可见时重新拉取数据。

### 其他限制
- web-view 会铺满整个页面，并且覆盖其他组件。
- H5 域名必须配置在小程序的业务域名里；配置时要下载校验文件，放在 Web 项目的 `public/` 下，确保 `https://域名/校验文件名` 能直接访问到（不能被中间件重定向或改写）。个人主体的小程序不支持 web-view。

## 微信内的其他坑

- **缓存**：微信内置浏览器对 HTML 缓存比较激进，发版后用户可能看到旧页面。HTML 响应要设置 `Cache-Control: no-cache`，静态资源用带 hash 的文件名长期缓存。
- **音视频自动播放**：iOS 微信里需要在 `WeixinJSBridgeReady` 事件里调用 `play()` 才能自动播放。
- **下载文件**：微信内不能直接下载文件。需要引导用户「在浏览器中打开」，或者改成预览。
