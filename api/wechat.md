# 微信服务端接口

用户表设计和登录流程见 `auth.md`。小程序的客户端部分见 `miniprogram/api-pitfalls.md`。

## 基本原则

- 所有微信接口调用都封装在 `lib/wechat/` 里，routes 和 services 不直接请求 `api.weixin.qq.com`。
- 微信接口出错时 HTTP 状态码大多仍然是 200，要检查返回体里的 `errcode`，不为 0 就是失败。
- AppSecret 只放在环境变量里，日志里不能打印出来。
- 只对接小程序：appid、AppSecret 都是小程序的。不对接服务号（JSSDK、网页授权不在本 skill 范围内）。
- openid 按 appid 区分，小程序用户用小程序的 openid 识别（存法见 `auth.md`「用户体系」）。

## access_token（接口调用凭证）

- 用 `POST https://api.weixin.qq.com/cgi-bin/stable_token`（`grant_type: client_credential`，不传 `force_refresh`，即普通模式）获取。
- **不缓存，每次用到时现取。** 普通模式下，token 有效期内重复调用返回的是同一个 token，不会让别处正在用的失效；频率限制是每分钟 1 万次、每天 50 万次，足够每次现取。不需要存数据库，也不需要处理多实例之间的同步。
- **不要用 `force_refresh: true`**：它会让之前的 token 立刻失效，每天限 20 次。现取的 token 仍然报 `40001` / `42001` 时，直接按微信接口失败处理并写日志，不自动强制刷新。
- 不要用旧接口 `GET /cgi-bin/token`：它每次调用都会让上一个 token 失效，必须集中缓存，和上面的做法冲突。

## 小程序登录

```
GET https://api.weixin.qq.com/sns/jscode2session
  ?appid=&secret=&js_code=CODE&grant_type=authorization_code
→ { openid, session_key }
```

- `session_key` 只存在服务端（如果确实需要），绝不返回给客户端。
- 登录成功后签发自己的 JWT（见 `auth.md`「token（JWT）」），返回给小程序。
- `js_code` 用过就失效，客户端重试时必须重新 `uni.login`。

## 手机号

```
POST https://api.weixin.qq.com/wxa/business/getuserphonenumber?access_token=TOKEN
body: { code }
→ { phone_info: { phoneNumber, purePhoneNumber, countryCode } }
```

- 这里的 `code` 来自 `getPhoneNumber` 按钮，和 `uni.login` 的 code 不是同一个东西。
- 手机号写入数据库前要和已有用户做唯一性检查，处理「同一个手机号已经绑定了别的账号」的情况。

## 内容安全

只能用于**小程序用户**：接口要求传小程序的 `openid`，并且这个用户近 2 小时内访问过小程序。Web 项目的账号密码用户没有 openid，不能用这套接口；Web 的 UGC 用第三方内容安全服务还是人工审核，先问用户。

### 文本（同步）
```
POST https://api.weixin.qq.com/wxa/msg_sec_check?access_token=TOKEN
body: { content, version: 2, scene, openid }
→ { result: { suggest: 'pass' | 'review' | 'risky', label } }
```
- `scene`：1 资料、2 评论、3 论坛、4 社交日志。
- 要求 `openid` 对应的用户在近 2 小时内访问过小程序。
- `risky` 直接拒绝；`review` 按业务决定是进人工审核，还是先隐藏。

### 图片和音频（异步）
- `POST /wxa/media_check_async`，传 `media_url`、`media_type`（1 音频、2 图片）、`version: 2`、`scene`、`openid`。
- 结果通过消息推送异步返回，需要在小程序后台配置消息推送 URL，并在 API 里实现对应的接收路由。
- 拿到结果之前，内容可以先标记为「审核中」，对他人不可见。
