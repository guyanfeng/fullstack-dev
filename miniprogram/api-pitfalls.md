# 小程序平台 API 易错点

只写客户端这一侧。服务端对应的部分（换 openid、解手机号）见 `api/wechat.md`。

## 登录

流程：`uni.login()` 拿到 `code` → 发给自己的 API → API 调 `code2session` 换取 openid → API 按 openid 找到或新建用户（见 `auth.md`「用户体系」），签发自己的 token 返回给小程序。

- `code` 只能使用一次，有效期 5 分钟。不要缓存 code，也不要在多个请求里重复使用同一个 code。
- `session_key` 永远不能下发到客户端。小程序只持有自己后端签发的 token。
- 静默登录放在 `App.onLaunch` 或请求封装里，在首次需要登录态时触发。并发请求时只登录一次，其余请求等待同一个 Promise。
- 不要做「打开就必须授权才能看」的强制登录，审核会驳回。先让用户浏览，到了需要身份的操作时再引导登录。

## 用户头像和昵称

- `uni.getUserProfile` / `wx.getUserProfile` 已经不能拿到真实头像和昵称了（只会返回默认的灰色头像和「微信用户」）。不要再用它们。
- 正确做法是「头像昵称填写能力」：
  ```vue
  <button open-type="chooseAvatar" @chooseavatar="onChooseAvatar">
    <image :src="avatar" />
  </button>
  <input type="nickname" v-model="nickname" @blur="onNicknameBlur" />
  ```
- `chooseavatar` 返回的是临时路径，必须先用 `uni.uploadFile` 上传到自己的存储，再保存 URL。直接保存临时路径，下次打开就失效了。
- 昵称会经过微信的安全检测，不合规时会被清空，所以要以 `blur` 之后的值为准。

## 手机号

- 只能通过按钮触发：`<button open-type="getPhoneNumber" @getphonenumber="onGetPhone">`。
- 回调里拿 `e.detail.code`，发给 API，由 API 调 `getuserphonenumber` 换取手机号。不要再用旧的 `encryptedData + iv` 解密方式。
- 用户拒绝时 `e.detail.errMsg` 不是 `ok`，要给出可以继续使用的路径，不能卡死流程。
- 这个能力需要小程序已完成认证，并且是收费的。开发前提醒用户确认账号资质。

## 订阅消息

- `uni.requestSubscribeMessage({ tmplIds })` 必须在用户的点击事件里**同步**调用。如果先 `await` 一个请求再调用，就不算点击触发了，会直接失败。
- 一次最多传 3 个模板 ID。
- 一次性订阅：用户每同意一次，服务端只能发一条。要发多条就需要多次引导订阅。
- 用户勾选「总是保持以上选择」之后不会再弹窗，回调会直接返回上次的结果。所以要以回调的结果为准，不要假设弹窗一定出现过。

## 分享

- 页面里定义了 `onShareAppMessage` 才会出现「转发」菜单；同时定义 `onShareTimeline` 才能分享到朋友圈。
  ```ts
  onShareAppMessage(() => ({ title, path: `/subpkg-survey/detail?id=${id}`, imageUrl }))
  ```
- `path` 必须带上恢复页面所需的全部参数。从分享卡片进入时没有页面栈，目标页要能独立加载数据，并提供回首页的入口。
- 分享图建议比例 5:4。不填 `imageUrl` 时默认截取当前页面上半部分。
- 不要写「分享后才能查看」「分享到群才能领取」这类逻辑，属于诱导分享，审核会驳回。

## 文件上传下载

- `uni.uploadFile` 和 `uni.downloadFile` 的域名需要单独配置在后台的 uploadFile / downloadFile 合法域名里，和 request 的合法域名分开配置。
- 上传前先压缩图片（`uni.compressImage`，或在 `chooseMedia` 时设置 `sizeType: ['compressed']`）。
- 大文件优先考虑由 API 签发直传凭证，直接上传到对象存储（OSS / COS），不要经过自己的 API 中转。

## 其他高频坑

- `uni.showToast` 和 `uni.showLoading` 不能同时显示，后调用的会顶掉前一个。用完 loading 一定要 `hideLoading`，包括出错的分支。
- `uni.setClipboardData` 成功后微信会自动弹出「内容已复制」，不要再额外弹一次 toast。
- 扫码进入、分享进入、普通打开时，参数的位置不一样：普通参数在 `onLoad(query)` 里；小程序码的 `scene` 参数要 `decodeURIComponent(query.scene)`。
