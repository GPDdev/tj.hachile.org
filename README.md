# 同济账号证明

https://tj.hachile.org — 仅提供生成证明码、验证证明码两个功能。没有校园导航、搜索、账户主页或其他栏目。

生成：用户在学校官网登录，后端验证学校签名的 ID token 后生成一个 10 分钟有效的一次性匿名码。
验证：输入证明码，后端原子核验并消耗；只返回认证状态和时间，不返回姓名、学号。

**当前为安全模板，认证默认关闭。学校应用尚未批准，Worker 和生产 D1 尚未部署，不生成模拟有效码。**

## 需要向同济开放平台申请什么

1. 申请一个接入同济统一身份认证的 Web 应用，使用 **OAuth 2.0 / OIDC 授权码模式**，服务端保管学校分配的 `client_id` 和 `client_secret`。
2. 网站地址登记为 `https://tj.hachile.org`；精确回调地址登记为 `https://tj-api.hachile.org/auth/callback`。不是前端首页，也不是旧的 tongji 域名。
3. 仅申请 `openid` 身份认证范围；本实现不调用学生个人信息、成绩、学籍、手机号、邮箱等数据接口，不需要申请这些接口权限。授权请求指定 `kc_idp_hint=tjiam`。
4. 请信息办确认该应用允许匿名认证证明码这个用途，支持 S256 PKCE、`nonce`、`prompt=login` / `max_age=0`，并能返回可验证签名的 ID token，含 `sub`、`iss`、`aud`、`exp`、`iat`、`nonce`、`auth_time`。获批后仍须联调，不能删除安全校验来绕过不兼容。

申请用途可填写：

> 用户主动通过同济统一身份认证登录后，本应用生成有效期 10 分钟的一次性匿名证明码，供他人核验一次同济账号认证是否完成。不获取姓名、学号、成绩或联系方式；不存储学校密码、用户标识或令牌；仅存储随机证明码的 SHA-256 哈希及有效期。不作为实名或在读学籍证明。

[官方学生申请指南](https://api.tongji.edu.cn/docs/intro/apply/student_apply)目前不向个人开发者开放。学生组织应请指导教师按[教职工申请指南](https://api.tongji.edu.cn/docs/intro/apply/teacher_apply)申请；需先在信息系统建设与管理平台备案，并与信息办确认需求，再通过 `system.tongji.edu.cn` 的“数据服务申请”办理。校外域名、个人项目的可接受性应先与负责老师确认，不能保证获批。

认证方案依据[官方授权码文档](https://api.tongji.edu.cn/docs/intro/develop/authentication_code)，不是申请 Client Credentials 批量数据权限，也不借用演示客户端。

## 部署

前端：GitHub Pages；后端：Cloudflare Worker；证明码哈希和临时认证会话：Cloudflare D1。

GitHub 仓库 `GPDdev/tj.hachile.org` → Settings → Pages → Source 选 GitHub Actions；工作流只发布 `public/`。
Cloudflare DNS 添加 DNS only 的 CNAME `tj` → `gpddev.github.io`，Pages Custom domain 填 `tj.hachile.org`，待证书可用后启用 Enforce HTTPS。

获批后，在 `worker/` 执行：

```powershell
npm ci
npx wrangler login
npx wrangler d1 create tj-identity
Copy-Item wrangler.example.jsonc wrangler.jsonc
```

在被 gitignore 忽略的 `wrangler.jsonc` 填 D1 UUID、CLIENT_ID；确认 Rate Limiter namespace_id 在账号内唯一，保持 `AUTH_ENABLED=false`。

```powershell
npx wrangler secret put CLIENT_SECRET
npx wrangler d1 migrations apply tj-identity --remote
npm run deploy
```

CLIENT_SECRET 只在终端输入、保存为 Worker Secret，**不可填进前端或提交 GitHub**。
Worker Custom Domain 为 `tj-api.hachile.org`。在受控环境启用 `AUTH_ENABLED=true`，联调成功登录、拒绝授权、state/nonce/PKCE、签名与声明、过期、回调重放及并发核验。通过后再把 `public/config.js` 中的公开 API 地址填为 `https://tj-api.hachile.org` 并推送前端。

## 检查

需要 Node.js 24（测试使用内置 SQLite）。

```powershell
cd D:\Desktop\hachile-workspace\tj.hachile.org\worker
npm ci
npm test
npm run check -- --config wrangler.example.jsonc
cd ..
node tools/check.mjs
```

测试模拟学校端点，覆盖关闭状态、真实 SQLite 迁移、RSA/JWT、state/Cookie、PKCE、nonce、issuer/audience/时效、来源与长度限制、并发一次性消费、过期清理和错误脱敏。未测试真实获批应用。

## 安全边界

证明码有 128 位随机熵，有效 600 秒，`DELETE … RETURNING` 保证最多一次核验成功。D1 只存码哈希及时间，临时会话存 state 哈希、nonce、PKCE verifier 及有效期；每 10 分钟清理过期记录。学校令牌只在后端内存使用，不保存、不输出。

回调证明码通过 URL fragment 到达浏览器并立即从地址栏移除，只在当前页内存显示。刷新即丢失，勿公开传播。认证 Cookie 使用 Secure、HttpOnly、SameSite=Lax；核验仅接受本站 Origin；输入绑定 SQL 参数；错误脱敏；每 IP/路由/Cloudflare 机房限流 30 次/分钟。校园共享 NAT 可能误触限流，该限额不是全局精确限额。Worker 关闭应用请求日志，但基础设施日志与备份受提供商政策约束。

码可被转交，不能证明出示者是账号本人，不能证明仍在读或院系。核验请求中途断网也可能已消耗该码，应重新生成。正式学籍用途需另行申请学校允许的在读状态核验接口，不能从登录成功推断。
