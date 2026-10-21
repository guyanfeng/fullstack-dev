# 移动端兼容性

## 视口高度

- `100vh` 在移动端会包含地址栏，导致底部被遮挡。用 `100dvh`，并用 `100vh` 作为回退：
  ```css
  .full { height: 100vh; height: 100dvh; }
  ```
- `dvh` 需要 iOS 15.4+，更老的系统用回退值。

## 安全区

- viewport 要设置 `viewportFit: 'cover'`（见 `web/guide.md`），否则 `env()` 拿到的都是 0。
- 底部固定栏：`padding-bottom: env(safe-area-inset-bottom)`。
- 全屏页面的顶部也要考虑 `env(safe-area-inset-top)`。

## 软键盘

- **iOS**：键盘弹出时不会改变视口高度，而是把整个页面往上推，`position: fixed` 的元素会错位。
- **Android**：键盘弹出时会压缩视口高度，底部固定元素会被顶上来，挡住内容。
- 处理方法：
  - 用 `window.visualViewport` 的 `resize` 事件计算键盘高度，手动调整布局。
  - 输入框聚焦时用 `scrollIntoView({ block: 'center' })` 确保输入框可见。
  - 带输入框的底部弹层，键盘弹出时改为相对定位或者跟随 visualViewport 调整。
- iOS 上键盘收起后页面可能没有回弹，留下一块空白：在 `blur` 时执行 `window.scrollTo(0, 0)` 或滚动到原来的位置。
- 输入框 `font-size` 小于 16px 时 iOS 会自动放大页面（见 `web/guide.md`）。

## 滚动

- 弹窗打开时背景页面还能滚动（滚动穿透）：
  - 弹窗打开时给 `body` 设置 `overflow: hidden`。iOS 上这样不够，还需要 `position: fixed` 并记录和恢复滚动位置。
  - 弹窗内部的滚动容器加 `overscroll-behavior: contain`。
- 局部滚动容器加 `-webkit-overflow-scrolling: touch`（老版本 iOS 需要）。
- 下拉页面时出现微信或浏览器的背景（橡皮筋效果），可以对 `html, body` 设置 `overscroll-behavior-y: none`，但会影响原生下拉刷新，要按需使用。

## 日期

- 接口返回的时间是 `yyyy-MM-dd HH:mm:ss` 格式的北京时间（见 `SKILL.md`「数据格式」）。
- iOS 上 `new Date("2026-10-21 10:00:00")` 会返回 `Invalid Date`，所以不能直接用 `new Date` 解析，统一用 `dayjs(str)` 解析和格式化。
- `dayjs(str)` 按设备本地时区解析。用户设备不在东八区时会有偏差，需要支持海外用户时，用 dayjs 的 timezone 插件按 Asia/Shanghai 解析。
- 提交给接口的时间也用同一格式，用 dayjs 格式化，不要手动拼字符串。

## 1px 边框

- 高分屏上 `1px` 会显得比较粗。需要真正的细线时，用 `transform: scaleY(0.5)` 的伪元素，或者 `box-shadow: 0 0 0 0.5px`。
- 大多数情况下直接用 `1px` 就可以，不要为了细线把代码弄得很复杂。

## 点击与手势

- 去掉点击高亮：`-webkit-tap-highlight-color: transparent`。
- 去掉长按菜单和选中（只在按钮等元素上用）：`-webkit-touch-callout: none; user-select: none`。
- iOS 上非可点击元素（`div`、`span`）的 `click` 事件可能不触发事件委托，给它们加上 `cursor: pointer`。
- 需要阻止默认行为的 `touchmove` 监听要写 `{ passive: false }`，否则 `preventDefault` 无效。

## 媒体

- 视频内联播放（不自动全屏）：`<video playsInline webkit-playsinline="true">`。
- 自动播放必须同时 `muted`，有声音的播放要等用户交互后才能触发。
- 微信里的自动播放见 `web/wechat-browser.md`。

## 图片与文件

- `<input type="file" accept="image/*">` 在部分 Android 机型上只能打开相册或只能拍照。需要拍照时加 `capture="environment"`，但这样 iOS 会直接打开相机，不能选相册。
- iOS 拍出的照片可能有 EXIF 旋转信息。现代浏览器显示时会自动处理，但用 canvas 压缩时要注意方向。
- 上传前用 canvas 压缩大图，控制在 1~2MB 以内。

## 字体

- 不要加载完整的中文 Web 字体（几 MB），使用系统字体栈：
  ```css
  font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Helvetica Neue", "Microsoft YaHei", sans-serif;
  ```
- 必须用特殊字体时（比如标题），只做子集化，包含用到的字。
