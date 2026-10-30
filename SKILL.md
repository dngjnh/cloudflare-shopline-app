---
name: cloudflare-shopline-app
description: "搭建可部署到 Cloudflare Workers 的空白 SHOPLINE 应用，完成配置入口、OAuth/App Bridge 授权、会话、令牌自动续期、卸载撤销及业务开发接口。用于创建或完善空白应用的授权底座；不用于 Shopify、SHOPLINE 主题/Storefront 或具体业务功能。"
---

# Cloudflare Workers 接入 SHOPLINE 空白应用

在用户指定目录交付可运行、可构建、可部署的空白 SHOPLINE 应用。用户填写应用配置、登记平台地址并完成授权后，可以直接编写业务页面和服务端路由；日常令牌刷新、订阅维护和卸载撤销由底座处理。默认基线为 React + TypeScript + Vite、Cloudflare Workers/Static Assets、每店铺一个 SQLite-backed `ShopAuth` Durable Object，以及官方 `@shoplineos/app-bridge`。

本技能提供生成应用所需的实现契约与验收依据，不附带预制代码模板。调用技能搭建应用时，应在目标目录实现完整代码、配置和测试，而不是只交付接入方案或留下授权存储、刷新等核心存根。

## 开始工作

1. 默认在用户指定的空目录中创建独立应用。若目标目录已有项目，则按需阅读现有配置和协作说明，保留已有改动、凭证和进程。
2. 根据用户要求和已提供的信息确定目标目录、应用名称、开发 HTTPS 域名、SHOPLINE 应用身份、内嵌模式、scope、端口与 Tunnel；目标目录已有配置时复用适用值。先读 [app-contract.md](references/app-contract.md)，集中配置与业务接入边界。可先完成虚拟配置下的实现与测试，缺少真实配置只阻止相应联调或部署，不成为核心代码留空的理由。
3. 实现或修改授权、签名、App Bridge 或 Webhook 前，查阅当前 SHOPLINE 官方文档。有可用的 SHOPLINE 授权/文档技能时使用；不要求安装特定插件。SDK 和示例需核对 Workers 适用性；官方资料与版本选择见 [sources.md](references/sources.md)。
4. 实现授权前阅读 [auth-lifecycle.md](references/auth-lifecycle.md)；实现启动、Tunnel 或部署时阅读 [local-environments.md](references/local-environments.md)；交付前按 [verification.md](references/verification.md) 选择验证。

## 空白应用边界

交付范围：签名入口、授权回调、App Bridge 中间页、空白后台、最小会话接口、安装/卸载通知、令牌与会话持久化、Alarm 刷新与订阅维护、服务端鉴权及 SHOPLINE 请求辅助接口、隔离测试、配置和部署文档。

授权数据存储于 DO 的 SQLite-backed storage，Alarm 承担每店铺的刷新和订阅维护。空白应用无需额外业务数据表、D1、Queues、Cron 或业务专用密钥与权限；这些由后续具体业务需求决定。

界面仅展示授权状态、必要的店铺信息和空白业务区域。UI 库与语言支持按用户要求或目标目录现有约定选择。

## 实现顺序

### 1. 固定最小路由契约

| 路由 | 职责 |
| --- | --- |
| `GET /` | 带 SHOPLINE 参数时完整验签；普通访问提供说明页 |
| `GET /api/auth/callback` | 验签、一次性 code 换令牌、建立会话、跳转 |
| `GET /auth/embedded` | 通过官方 App Bridge 发起 OAuth 的独立前端页 |
| `GET /app` | 空白应用页面，不以 URL 参数作为登录凭据 |
| `GET /api/auth/session` | 验证 Cookie 与当前安装，只返回安全的授权信息 |
| `POST /api/webhooks/app` | 原始 body 验签、校验生命周期事件并撤销卸载店铺 |

这些路径是可复用的设计选择，不是 SHOPLINE 固定路径。与已有应用集成时可保留原路径，但后台登记、代码、订阅和文档必须一致。API 不应回落成 SPA 的 200 HTML。未知 API 返回 404，已知路由的方法不匹配返回 405。

### 2. 实现签名与授权

使用 Web Crypto，先拒绝重复查询参数，再校验 appkey、handle、毫秒时间戳和签名。使用签名中的店铺构造固定 SHOPLINE HTTPS 目标；不接受调用方传入任意 token URL 或回调地址。

code 换令牌与刷新按店铺串行。code 请求不自动重试；刷新发送空 body，并对实际发送的 body 加 timestamp 签名。Workers 出站使用 `redirect: 'manual'` 并拒绝 3xx，设置超时。

内嵌 OAuth 通过目标 App Bridge 版本支持的官方接口发起，处理开发 StrictMode 重复 effect。独立的 `/auth/embedded` 避免带 handle 的页面被根路由误判为签名请求。具体协议、响应校验与 Bridge 版本差异见授权参考。

### 3. 完成生命周期，再展示空白后台

`ShopAuth` 是安装、令牌和会话的唯一持久来源。用 Alarm 在令牌到期前刷新，并维护安装/卸载事件订阅；失败的下一次执行时间落盘。授权成功不等待订阅创建成功。

卸载独立于令牌网络等待，立即撤销令牌、会话、Alarm 并保留未安装状态。每个旧异步操作在提交安装数据、会话或 Alarm 前检查授权版本，防止卸载后恢复。拒绝旧安装延迟到达的撤销事件，不把新 `INSTALL` 通知当作重新授权。

浏览器使用随机 HttpOnly 会话 Cookie，后端只存会话摘要；不返回 accessToken 或 APP Secret。空白页面区分检查中、未授权、已授权和暂时无法核实，授权失效时清除旧店铺展示。

### 4. 配置本地开发与生产隔离

本地 Vite 通过 Cloudflare 插件运行真实 Workers。默认回环监听、固定端口、`strictPort`；沿用已有端口与命名 Tunnel。新项目可用 `127.0.0.1:5173`，但不能因此改动已有用户进程。

密钥放入被忽略的 `.dev.vars`，示例只放占位值；不覆盖已有密钥文件。显式声明 `secrets.required` 时，空白应用仅需声明自己的 `SHOPLINE_APP_SECRET`。生产身份、域名、密钥与 DO 隔离，环境在 Vite 构建时选择，部署使用本次校验过的产物。

“可部署”要求交付有效生产配置、构建校验及明确部署命令，不代表本次必须上线。用户仅要求搭建/整理应用不构成创建云资源、修改 Tunnel/DNS、配置生产 Secret 或部署的授权；用户已明确要求并授权的部署则完成相应步骤，无需重复询问。具体环境边界以目标项目与本次授权为准。

### 5. 留出业务开发入口

把授权服务与空白业务页分开。实现服务端会话鉴权、受当前安装保护的 SHOPLINE 请求辅助函数，在 README 给出如何增加页面和受保护路由的最小示例，接口职责见 [app-contract.md](references/app-contract.md)。业务代码无需复制验签、读取 DO 私有存储或自行刷新令牌。后续 scope 变化通过正常入口重新授权。

## 交付要求

- 交付实现代码、隔离测试、配置清单和可执行命令；README 列出配置位置、平台登记地址、本地 HTTPS、生产部署与业务接入方法。把代码缺陷与尚待用户填写的身份/域名/密钥区分开。
- 修改 bindings/vars/Secret 声明后通过 `wrangler types` 或项目的 `cf-typegen` 命令生成类型，不手写生成文件。更换兼容日期或运行时依赖后实际启动 workerd。
- 使用当前项目的测试/lint/build 命令，按验证参考覆盖授权竞态；只有文档/技能改动时检查结构、引用和 diff 即可。
- 区分单元、workerd 集成、本地 HTTP、开发 HTTPS 和真实 SHOPLINE 联调；只报告当前应用实际完成的验证。
