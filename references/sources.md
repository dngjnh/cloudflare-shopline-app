# 官方资料与版本选择

## 按任务查阅

实现或修改相应功能时查阅当前官方文档：

| 功能 | 官方资料 |
| --- | --- |
| 入口、授权参数、code 换令牌、刷新 | [SHOPLINE 应用授权](https://developer.shopline.com/docs/apps/api-instructions-for-use/app-authorization/) |
| 查询签名、POST body 与 timestamp 签名 | [SHOPLINE 签名算法](https://developer.shopline.com/docs/apps/api-instructions-for-use/generate-and-verify-signatures?lang=en) |
| 内嵌父窗口授权与 SDK 调用 | [App Bridge Authorization](https://developer.shopline.com/docs/apps/development-tool/app-bridge/actions/authorization/) |
| 原始 body HMAC、请求头与重送 | [Webhook 概述](https://developer.shopline.com/docs/apps/api-instructions-for-use/webhooks/overview/) |
| 安装/卸载 topic 与 payload | [安装状态变更事件](https://developer.shopline.com/docs/webhook/application/app-installation-status-changed/) |
| 订阅 API 契约 | [订阅 Webhook](https://developer.shopline.com/docs/admin-rest-api/webhook/subscribe-to-a-webhook) |
| 构建时环境选择、配置与 Secret 加载 | [Cloudflare Vite 环境](https://developers.cloudflare.com/workers/vite-plugin/reference/cloudflare-environments/) |
| 持久状态协调与异步交错 | [Durable Objects 设计规则](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/) |
| Alarm 调度、重复执行与重试 | [Durable Objects Alarm](https://developers.cloudflare.com/durable-objects/api/alarms/) |

## 版本与依赖

- 明确选择受支持的 SHOPLINE API 版本，统一用于请求路径、订阅和事件解析。无版本文档可能默认展示预发布版本，不直接采用 `Latest` 或页面默认值。
- App Bridge 的授权调用以安装版本导出、类型和对应官方文档为准，并验证真实内嵌行为。
- 引入 SHOPLINE 后端 SDK 前，在真实 workerd 中验证输入校验、Webhook 签名格式、超时和重定向行为。SDK 不替代应用的安装持久化、会话、撤销与刷新协调。
- 兼容日期与依赖版本由实际运行环境确定，生成锁文件并启动验证。
- 本技能中的会话容量、时长、提前刷新点、退避和复查间隔是应用策略，不是平台统一要求；调整时保留撤销、持久重试和并发保护。

若文档搜索不可用，可直接访问官方页面；无法核对的版本或字段明确记录为待确认，不将示例或推测当成已验证协议。
