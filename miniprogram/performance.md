# 小程序性能

## 包体积

限制：主包 ≤ 2MB，单个分包 ≤ 2MB，所有包加起来 ≤ 20MB。

排查顺序：
1. 用微信开发者工具「详情 → 基本信息 → 本地代码」和「代码依赖分析」，找出体积最大的文件。
2. `static/` 里的图片移到 CDN，这通常是收益最大的一步。
3. 只被某个分包用到的组件和依赖，挪进该分包目录。并在 `manifest.json` 中开启：
   ```json
   "mp-weixin": {
     "optimization": { "subPackages": true },
     "lazyCodeLoading": "requiredComponents"
   }
   ```
4. 检查是否整包引入了大型库（如完整的 lodash、moment、echarts 全量）。换成按需引入或更轻的替代品（`lodash-es` 按函数引入，`dayjs`，echarts 自定义构建）。
5. 构建产物要用发行模式（`build:mp-weixin`）来看体积，开发模式的产物没有压缩，不能作为参考。

分包规则：
- tabBar 页面必须在主包里。
- 按业务模块分包，比如 `subpkg-survey`、`subpkg-user`。
- 用户很可能马上进入的分包，在 `pages.json` 里配置 `preloadRule` 预下载。

## 启动与首屏

- 首页请求在 `onLoad` 里发出，不要放在 `onReady` 或 `onMounted`，越早越好。
- 用骨架屏代替全屏 loading。
- 首屏只加载首屏需要的数据，其余部分分批或懒加载。
- 能缓存的配置数据（分类、字典）先读本地缓存渲染，再后台刷新。

## 渲染与数据更新

uni-app 的响应式数据变更最终会通过 setData 从逻辑层传到渲染层，数据量和频率决定了性能。

- 响应式数据只保留模板会用到的字段。接口返回的大对象先裁剪再赋值；不需要响应式的数据用普通变量或 `shallowRef` / `markRaw`。
- 不要在 `onPageScroll` 里修改响应式数据。需要吸顶、曝光判断的场景，用 `uni.createIntersectionObserver` 或 CSS `position: sticky`。
- 高频事件（输入、滚动、拖动）做节流或防抖。
- 长列表：分页加载，每次追加，不要整个数组替换。超过数百条用虚拟列表（例如 z-paging 的虚拟列表模式）。
- 列表的 `v-for` 必须写稳定的 `:key`（用业务 id，不用 index）。
- 显示隐藏频繁切换的用 `v-show`，很少显示的用 `v-if`。

## 图片

- `<image>` 加上 `lazy-load`，并设置正确的 `mode`（一般用 `aspectFill` 或 `widthFix`）。
- 头像、列表缩略图不要直接用原图。

## 内存与泄漏

- `setInterval`、`setTimeout`、`uni.$on`、WebSocket、IntersectionObserver，在 `onUnload` 里清理。
- 不要在全局 store 里堆积页面级的大数据，页面卸载后要释放。
