# 授权协议与生命周期

## 1. SHOPLINE 入口与签名

入口查询参数使用 `appkey`、`handle`、`timestamp`、`sign`，回调另含 `code`；内嵌入口可带 `lang`。发起 OAuth 的参数名称则是 `appKey`、`responseType=code`、`scope`、`redirectUri`。保留大小写，不套用 Shopify 的 HMAC query、`shop`、refresh-token grant 或其他平台的端点。

查询验签顺序：

1. 在转换为普通对象前检查 `URLSearchParams`，拒绝任何重复 key，包括 `sign` 和未知参数，防止验签值与执行值不同。
2. 检查 `appkey` 等于后端配置；handle 是单个店铺域名前缀，可采用 `^[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?$`。不接受完整 URL、斜杠、点号或凭证。
3. 时间戳要求 13 位十进制 Unix 毫秒整数，并检查 `abs(now - timestamp) <= 600000`，同时拒绝过旧与过远的未来时间。
4. 去除 `sign`，其余全部参数按 key 字典升序，以 `key=value` 拼接并用 `&` 连接。依据官方算法和实际解码行为构造原文；不要重新 URL 编码或只签白名单字段。
5. 使用 APP Secret 做 HMAC-SHA256，校验 64 位 hex 签名，优先用 `crypto.subtle.verify` 而非普通字符串比较。

上面的双向窗口、严格格式和去重是应用的输入保护；不要把签名正确等同于应用身份、店铺和时间戳均合法。含空格、`+`、`%`、Unicode、空值和额外参数的情况须有确定性签名测试。

根入口只要出现任何 SHOPLINE 签名相关参数就进入验签流程，参数不完整也拒绝。`/app?handle=...` 与 `/auth/embedded?...` 是 UI 页面，它们不能自行建立会话。只对 API 路由做必要的尾部斜杠规范化，不重写签名 query。

## 2. OAuth 与网络

独立访问的授权地址：

```text
https://{handle}.myshopline.com/admin/oauth-web/#/oauth/authorize
  ?appKey={appKey}&responseType=code&scope={scope}&redirectUri={redirectUri}
```

使用 URL API 正确编码查询参数，`redirectUri` 固定来自后端 `APP_URL` 和已登记回调路径。handle 只能来自已验签入口。OAuth code 仅服务端使用，交换后跳转到干净的 `/app`，不继续携带 code、sign 或 token。

内嵌授权必须走官方 App Bridge，使后台父窗口处理 OAuth，不能用 iframe 中的 `location.href` / `window.open` 替代。以下为官方 `Oauth` 接口的用法示例，使用前核对安装版本：

```ts
import Client, { Oauth, shared } from '@shoplineos/app-bridge'

const app = Client.createApp({ appKey, host: shared.getHost() })
Oauth.create(app).invoke({ appKey, scope, redirectUri })
```

核对 App Bridge 安装版本的导出、类型与当前官方文档，再验证真实后台行为。若使用其他官方授权接口，也需确认它能在父窗口完成授权。Bridge 初始化失败时显示可操作的错误，避免无限跳转；开发 React StrictMode 下同一次授权仅派发一次。

令牌请求契约：

| 操作 | POST 路径 | 实际 body |
| --- | --- | --- |
| code 换令牌 | `/admin/oauth/token/create` | `JSON.stringify({ code })` |
| 刷新 | `/admin/oauth/token/refresh` | 空字符串，通常省略 `body` 属性 |

固定目标为 `https://{handle}.myshopline.com`。请求头含 `Content-Type: application/json`、`appkey`、`timestamp`、`sign`；`sign` 为 `HMAC-SHA256(secret, actualBody + timestamp)` 的 hex。先生成一次 body 再签名并发送同一字符串。刷新不发送 `{}`，也不依赖 refreshToken。

所有携带凭证的出站请求使用 `redirect: 'manual'` 并拒绝 300–399，不能未经 workerd 验证就改用其他重定向模式。为 token 请求设置取消/超时，建议默认 8 秒；订阅请求建议默认 5 秒。一次性 code 不自动重试，失败或结果未知时提示从正常入口重新授权，不能重放旧回调 URL。

同时检查 HTTP 状态与响应 JSON：成功 `code === 200`，`data.accessToken` 为非空字符串、`scope` 为字符串、`expireTime` 可解析且晚于现在。以响应到期时间为准，不从示例推断固定有效期。对 scope 做拆分、去空、去重和排序后比较；默认要求与所需 scope 集合一致，不一致要求重新授权。

官方 token 响应以 accessToken、expireTime、scope 为准；不解析 accessToken 来猜 storeId，不要求响应存在 refreshToken。浏览器登录由应用的随机 Cookie 会话实现。

## 3. ShopAuth 状态与并发

在单一应用、单一环境的 DO namespace 中，以校验过的 handle 选取对象。同一 namespace 承载多应用时标识还需包含 app identity。SQLite-backed DO 可通过 `ctx.storage.get/put` 保存授权记录，无须额外 D1。

持久记录至少包括：

| 记录 | 必要内容 |
| --- | --- |
| installation | handle、accessToken、expireTime、scope、installed、authorizedAt、持久安装标识/版本 |
| 刷新计划 | refreshAt、refreshRetryAt、refreshAttempts |
| subscription | 绑定当前 authorizedAt、ready、failures、nextAttemptAt |
| sessions | 会话摘要与 expires；有数量和过期清理上限 |
| 撤销标记 | lastUninstallAt / 当前安装代次，不能只删数据后忘记撤销事实 |
| code 防重放 | code 的摘要和有界有效期/数量，不保存原始 code |

在 code 有效窗口内保留有界摘要，重复请求不得再调用 token/create，也不得创建新会话。只保存最近一次摘要不足以覆盖窗口内所有已消费 code；过期摘要可随正常操作清理。

DO 单线程不等于跨网络 await 不交错。令牌创建、刷新、会话签发、Alarm 使用每店铺 Promise 操作队列，队列在任务失败后也能继续。卸载走独立短撤销队列，令牌操作开始前等待此前撤销完成，但卸载不等待上游网络。

每个异步操作捕获授权 epoch/安装代次，网络返回后与写入令牌、会话、订阅状态或 Alarm 前重新校验。卸载先使旧 epoch 失效，再删除令牌、会话、刷新计划与 Alarm，持久化未安装状态及事件时间。初始化只在必要时用短 storage gate；不能用覆盖整个网络请求的 `blockConcurrencyWhile` 将卸载卡住。

持久安装标识用于跨请求、重启和未来业务任务判断当前授权，不能只用进程内 epoch 代替。状态清理不得删除仍在有效窗口内的已消费 code 摘要；否则卸载、重装或重启后同一 code 会重新触发交换。代码/会话数量限制应有明确拒绝或过期清理策略，不能提前淘汰仍有效的防重放证据。

code 回调还需与最后卸载时间比较，旧签发回调不能使已撤销安装复活。晚到卸载事件若不晚于已处理撤销，或早于当前新授权，则幂等忽略。`INSTALL` 事件仅确认接收，不能单独签发令牌/会话；重装由新的已验签 OAuth 完成。

具体事务与 gate 写法需用真实 workerd 并发测试证明满足上述不变量，授权类保持独立于业务模块。

## 4. Alarm、订阅与远端核验

令牌按剩余有效期的约 90% 设置提前刷新点。授权成功先持久化安装与会话，然后安排 Alarm 异步维护 `apps/installed_uninstalled`；补订阅失败不能使已完成的 OAuth 返回失败。

订阅通过 Admin API 的 `/admin/openapi/{version}/webhooks.json` 查询/创建；创建 body 形状为 `{ webhook: { topic, address, api_version } }`，Authorization 为 Bearer accessToken。版本来自统一 `SHOPLINE_API_VERSION` 配置，选择时与当前官方定义一起核对，不把无版本文档默认跳到的未来版本或 `Latest` 直接复制到配置。

按 topic、address、api_version 判断已有订阅，避免每次入口重复创建。`webhooks: null` 按空列表处理，其他异常结构应报错。以下默认退避参数是可调整的应用策略：

| 计划 | 首次失败 | 上限 | 成功后 |
| --- | --- | --- | --- |
| token 刷新 | 30 秒 | 5 分钟 | 按新 expireTime 计算 |
| 订阅维护 | 30 秒 | 30 分钟 | 约 6 小时复查 |

分别保存尝试次数与下一次时间，用一个 Alarm 调度最近的有效任务，不依赖平台有限次数的异常重试。过期令牌的会话请求同样遵守已保存的刷新退避，不能绕开计划触发请求风暴。

签名入口可通过只读订阅查询强制核验远端令牌，会话查询可缓存核验结果约 30 秒。明确 401 或 token API 的 `STORE_NOT_INSTALL_APP` 触发撤销；超时、网络故障、5xx 仅表示暂时无法核实，不把它们当成卸载。前端可在可见时约 15 秒检查会话，聚焦时再检查；这是有延迟的漏通知补偿，不保证实时发现卸载。

## 5. 生命周期 Webhook

仅允许 POST。限制请求体大小，读取原始 body 并验签后再解析 JSON。`X-Shopline-Hmac-Sha256` 的官方示例使用 Base64；支持正确长度的 Base64 HMAC-SHA256。验证原始字节，不能把 JSON parse/stringify 后的结果拿来验签。

检查 `X-Shopline-Topic === 'apps/installed_uninstalled'`，以及已签名 payload 中的 appkey、合法 handle、operate、timestamp。字段结构随订阅 API 版本核对，处理 INSTALL/UNINSTALL；timestamp 要求正安全整数毫秒值，拒绝明显未来事件。延迟通知用持久事件时间/安装代次判断，不能机械套用入口十分钟窗口。

只有完成本地持久撤销后才确认卸载成功；重复已处理事件返回 200，撤销暂时失败返回可重试错误。处理不能等待 token 请求超时。Webhook 头不是原始 body 签名的一部分，不能仅凭 topic 或 domain 头选择其他店铺；若使用头部 shop 信息，应与签名 payload 身份一致。别记录 raw body、签名或令牌。

## 6. 浏览器会话

Cookie 可命名为 `__Host-sl_session`，值包含合法 handle 和高熵随机会话值。属性为 `Path=/; HttpOnly; Secure; SameSite=None; Partitioned`，不设置 Domain。默认有效期 6 小时，每店铺至多 32 个未过期会话；这些容量和时长是应用策略。后端保存 HMAC/摘要而不是原始会话值。

会话认证同时检查摘要、到期时间、安装状态和 scope。`GET /api/auth/session` 可返回 handle、scope、expireTime、installed，不返回 token、Secret、会话摘要或完整安装记录。401 时清除 Cookie；暂时上游不可用返回适当的 503，前端显示核验失败，不继续展示旧授权状态。

API 与重定向添加 `Cache-Control: no-store`、`Referrer-Policy: no-referrer` 等必要响应头。未来新增 Cookie 认证的写接口时，再做同源 Origin 与内容类型检查；不为“空白应用”预先暴露通用 Admin API 代理。

Partitioned Cookie 需要浏览器支持，完整授权必须通过 HTTPS 开发域名测试。若目标浏览器不支持，应明确缺口并单独设计方案，不能临时把会话或 accessToken 放进 URL/localStorage。
