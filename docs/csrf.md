# 通过“CSRF 思路”调用酷狗官方 API（简版说明）

> 本项目后端充当中间层：利用浏览器携带的 Cookie 与伪造官方客户端参数（UA/签名/必备查询参数），将请求转发到酷狗官方接口。该方式在上游文档中被称为“CSRF（跨站伪造）调用”。

## 一句话原理
- 浏览器先从后端拿到登录 Cookie（`token`、`userid` 等）；
- 之后所有业务请求打到后端，浏览器自动带上这些 Cookie；
- 后端把 Cookie 中的 `token`、`userid`、`dfid` 等注入官方 API 所需参数，并伪造客户端特征与签名，转发给官方网关；
- 官方返回的 `Set-Cookie` 由后端继续透传给浏览器。

## 请求链路（最小化）
1) 登录：`/api/login_cellphone` → 后端调用 `login.user.kugou.com`，解密 `secu_params` 后写回 `Set-Cookie: token, userid, ...`。
2) 业务：`/api/**` → 浏览器自动携带 Cookie → 后端读取 `req.cookies` 注入参数 → 转发至 `gateway.kugou.com`（或登录域名）。
3) 同步：官方返回的 `Set-Cookie` 继续由后端写回，维持会话所需信息。

## 关键实现点
- 统一参数注入：`util/request.js`
  - 从 `options.cookie` 取 `dfid / token / userid`，构造 `defaultParams`：`dfid、mid(md5 dfid)、uuid、appid/clientver（随 platform）、userid、clienttime` 等；若有 `token` 则一并加入。
  - 根据场景选择签名：`signatureAndroidParams` / `signatureWebParams` / `signatureRegisterParams`，默认走 Android。
  - 伪造头：固定 Android UA；必要时加 `X-Real-IP`/`X-Forwarded-For`；登录接口用 `headers: { 'x-router': 'login.user.kugou.com' }` 指示路由。
- Cookie 进出：`server/server.js`
  - 读入：解析请求头为 `req.cookies`；
  - 写回：将模块返回的 `cookie` 数组写入响应 `Set-Cookie`；HTTPS 场景附带 `SameSite=None; Secure`。
- 登录获取凭证：`server/module/login_cellphone.js`、`server/module/login_token.js`
  - 登录/刷新时从官方响应的 `secu_params` 解出 `token`，并把 `token / userid / vip_*` 推入 `res.cookie` 以写回浏览器。

## 与“CSRF”的对应关系（本项目语境）
- 这里的“CSRF”指“跨站携带 Cookie + 伪造官方客户端请求”的调用方式，而非讲述 Web 防御策略的安全章节。
- 本后端不保存会话，仅依赖浏览器 Cookie 与官方返回值完成转发与签名。

## 相关文件（便于溯源）
- `server/util/request.js`：参数注入、签名、UA 与路由头设置；
- `server/server.js`：Cookie 解析与 `Set-Cookie` 透传；
- `server/module/login_cellphone.js`、`server/module/login_token.js`：登录/刷新，提取并回写凭证。

## 配置与使用速记
- `.env`：`platform=lite`（概念版），`API_PREFIX`（默认 `/api`），`CORS_ALLOW_ORIGIN`；
- 登录：调用 `/api/login_cellphone?mobile=&code=`（或体现在请求体）；
- 刷新：`/api/login_token`；
- 其余接口：直接请求 `/api/**`，浏览器会自动带上登录时写入的 Cookie。
