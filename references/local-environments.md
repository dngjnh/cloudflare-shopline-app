# 本地运行、Tunnel 与环境隔离

## 最小项目组成

```text
worker/index.ts                  Worker 路由、ShopAuth 导出
worker/security.ts               Web Crypto 签名与输入校验
src/App.tsx                     授权状态、Bridge 页面、空白后台
scripts/local.mjs               固定本地环境的 dev/build/preview
scripts/production.mjs          生产隔离、构建与产物校验
scripts/tunnel.mjs              需要命名 Tunnel 时的运行管理
tests/auth.test.mjs             确定性签名与路由测试
tests/auth-runtime.test.mjs     隔离 workerd/SQLite/Alarm 测试
tests/fixtures/auth-worker.ts   仅测试使用的控制入口
wrangler.jsonc
vite.config.ts
.dev.vars.example              APP Secret 占位说明
.dev.vars.production           只有注释，阻止加载开发密钥
```

结构可随应用规模调整，不要求为单文件逻辑制造额外层。按目标环境选择依赖并生成锁文件。使用 Web Crypto、Request/Response/fetch，不默认引入 Express、Redis、Node 后端 SDK 或 `nodejs_compat`。

## Wrangler 与 Vite

最小绑定只有 `ASSETS`、`SHOP_AUTH`。`ShopAuth` 从 `cloudflare:workers` 的 `DurableObject` 派生并由 Worker 模块导出；新 SQLite DO 使用 `new_sqlite_classes: ['ShopAuth']` 的初始 migration。

配置决策：

| 配置 | 要求 |
| --- | --- |
| `main` | 指向实际 Worker 模块 |
| `compatibility_date` | 使用目标 workerd 确实支持的日期，实际启动确认 |
| `assets.binding` | `ASSETS`，与代码一致 |
| `assets.not_found_handling` | React SPA 可用 `single-page-application` |
| `assets.run_worker_first` | 覆盖签名根入口 `/` 与 `/api/*`，避免静态资源吞掉认证 |
| vars | SHOPLINE_APP_KEY、APP_URL、SHOPLINE_SCOPES、SHOPLINE_API_VERSION、APP_ENV，以及实际采用的内嵌模式配置 |
| `secrets.required` | 使用此配置时必须包含 SHOPLINE_APP_SECRET |
| DO | SHOP_AUTH → ShopAuth；不要指向其他环境的 script_name |
| 默认云端暴露 | 可预留 `workers_dev: false`、`preview_urls: false`，开发无需云端路由 |

scope 由平台登记和实际需要决定，不预设业务权限或编造无需权限的 scope。若空白应用需要一个有效 scope 才能授权，先核对当前平台要求并由已有配置或用户确定；配置缺失时明确拒绝授权，仍可完成本地无凭证验证。scope 变更要求重新授权。

Vite 使用 React 与 `@cloudflare/vite-plugin`，Worker 环境名称与 TypeScript 配置一致。默认回环地址、固定端口、`strictPort: true`，`allowedHosts` 只包含用户确定的开发域名，不能设为全部允许。缺少域名时先完成本地开发与配置说明。

兼容日期须在所安装 workerd 的支持范围内；变更兼容日期或运行时依赖后实际启动验证。build 成功不能替代启动。

## 密钥与日志

忽略 `.dev.vars*`、`.env*`、`.wrangler/`、`.cloudflared/`、`dist/` 和日志；按需放行只含占位值的 example，以及只有注释的 `.dev.vars.production`。保留已有文件，不读取或复制真实 Secret、Tunnel 凭证、账户证书到新产物。

使用 `secrets.required` 后，未声明的密钥可能被 Wrangler 过滤，因此“文件里已有 Secret”不代表 Worker binding 已加载。修改声明后重新生成类型，并用隔离虚拟值验证。APP Secret 不进入 `VITE_` 变量或浏览器 bundle。

空白应用默认只记录脱敏事件、状态码和尝试次数；关闭会暴露原始授权 URL 的自动 invocation logs，不记录授权 code、Token、Cookie 或 Secret。

## 命令职责

```sh
npm run dev          # 仅本机 Vite/Workers
npm run tunnel       # 仅运行已配置的开发 Tunnel
npm run dev:tunnel   # 二者都未运行时联合启动
npm run build        # 固定默认本地配置
npm run preview      # 按本地配置检查构建结果
npm run cf-typegen   # 根据实际 Wrangler 配置生成类型
```

仅给实际实现的脚本记录命令。空白应用的本地启动脚本只负责环境选择与 Vite/Workers 进程管理；授权 Alarm 由 workerd 处理，不要求额外业务数据库初始化或任务调度。

本地脚本显式选择默认环境，不被父 shell 中 `CLOUDFLARE_ENV=production` 意外切到生产；并核对 Vite `.env` 加载规则。生产脚本独立选择 `CLOUDFLARE_ENV=production`，禁止开发 `.env` 密钥回退。

## HTTPS 开发 Tunnel

完整 OAuth 路径为“SHOPLINE → 用户开发 HTTPS 域名 → 命名 Tunnel → 本机固定端口 → Vite/Workers”。这不需要部署 Worker。已有服务时先检查端口并只启动缺少的 Tunnel，不终止用户进程、不随机换端口。

日常使用复用已有路由。登录、创建/修改 Tunnel、添加/覆盖 DNS 必须在用户明确要求的范围内；脚本中的 setup 步骤不是执行授权。新项目若还没有 Tunnel，先完成配置方案和本地验证，再按用户范围处理云端初始化。

Tunnel 配置应：

- 仅允许指定 hostname；阻断 `/__debug`、`/cdn-cgi/local` 及子路径；最后以 404 兜底。
- 凭证以受限权限直接写文件，不打印 token；已有文件只校验，不覆盖。ID 不一致或 DNS 冲突时停止，不自动重建或覆盖。
- 指标监听回环地址，端口可动态分配；不通过 debug 日志采集真实授权流量。
- 联合启动脚本仅管理自己启动的子进程，退出时不能终止已有用户服务。

同一命名 Tunnel 在多台电脑同时运行可能把请求分流到不同本地状态。默认同一时间只在当前电脑连接；独立并行联调应使用各自的域名/Tunnel/应用身份。

## 生产边界

空白应用应带有可填写的 `env.production` 及真实实现的生产脚本；APP KEY、HTTPS 域名和所选暴露方式的配置缺失时阻止部署。使用独立 Worker、应用身份、Secret 和 DO namespace，不绑定开发或其他 Worker 的授权对象。公开地址采用用户确定的自定义域名/路由，或已登记的 workers.dev HTTPS 地址；不要未经要求强制购买域名或创建 DNS 记录。沿用目标项目更严格的生产约定。

默认 `npm run deploy` 拒绝未明确选择环境的部署，并提供明确的生产命令。生产构建校验只检查空白应用实际依赖。部署本身只在用户明确授权后执行。

Cloudflare Vite 环境在构建时决定。生产部署必须重新构建，读取此次生成的 Wrangler 配置并核对 name、APP_ENV、APP KEY、APP_URL、路由及 SHOP_AUTH，最后部署这份配置。不能仅运行 `wrangler deploy --env production` 并假定已有本地产物自动切换。

保留只有注释的 `.dev.vars.production`，避免回退加载开发 `.dev.vars`；生产 Secret 使用明确授权的远端 Secret 流程。可先在无生产配置时测试部署保护是否于联网前拒绝，无需为验证创建云端资源。

生产脚本至少区分配置检查、生产构建、Secret 输入与部署，例如 `check:production`、`build:production`、`secret:production`、`deploy:production`。只在脚本确实存在时把命令写入 README。检查模式不上传、不写 Secret、不申请域名；Secret 通过交互安全输入或已授权的密钥机制传递，不能把真实值拼入命令参数。

用户要求实际部署时，在授权范围内核实账户、目标 Worker 与生产配置，配置独立 Secret，构建并部署本次校验产物，再用生产 HTTPS 检查页面、未登录 session 及错误路由。首次创建 Worker/DO 与 Secret 的先后顺序以当前 Wrangler 支持的流程为准；确需先创建目标时不对外暴露缺少 Secret 的应用。只读检查和本地部署准备无需重复请求确认，遇到账户权限或配置缺失则明确具体阻碍。写 Secret 与部署失败后，先核实远端状态，再决定是否重试；不要盲目循环或自动换账户、目标、域名。生产 HTTP 可达仍不能证明真实店铺已完成 OAuth。
