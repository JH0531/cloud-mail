# 项目长期约定

## 分支
- `main`：fork 同步源（upstream），不部署。
- `deploy`：唯一部署分支，Cloudflare Workers Builds 监听它。
- `backup-main`：本地备份，不参与同步。
- `OPC-design`：OPC 隔离实例分支（自 deploy 创建，2026-10-25），部署 Worker `cloud-mail-opc` @ `mail.supracept.work`；永不合入 main，详见下方专节。

## wrangler.toml（mail-worker）
- 每轮 sync 必冲突。本地值固定为：D1 `jaspermail` / `0b186691-fb70-42ea-8944-5a8d2fcae097`，KV `e27cda1bbbef4fddb0726b63c75810e6`，R2 桶 `jaspermail`（binding `r2`）。
- 冲突处理原则：只替换 `[[d1_databases]]`、`[[kv_namespaces]]`、`[[r2_buckets]]` 三块的取值，其余保持上游。

## 部署拓扑（必须写进 wrangler.toml，否则会被 wrangler 摘除）
- 自定义域名：`[[routes]] pattern = "mail.jaspershadow.dpdns.org" custom_domain = true`。
- `jwt_secret` 不入库，走 Dashboard Secret（wrangler deploy 恒为 `keepSecrets: true`，不会被清空）。

## 数据库升级约定
- 每次 fork sync 合并上游新版本并部署后，必须手动访问 `https://<站点域名>/api/init/<jwt_secret>` 执行一次迁移（幂等，可重复）。
- 路径必须带 `/api` 前缀：不带前缀会被 SPA 兜底成前端页面，迁移不会执行。
- 弹窗「请按照文档更新数据库」= `hono.js:28` 捕获到 `D1_ERROR: no such column`，说明代码引用了迁移尚未添加的列。

## 本地补丁（需在 sync 时保留）
- `mail-worker/src/service/r2-service.js` 中带 `[local]` 注释的 KV 回退读取（R2 未命中时回落 KV），用于 R2/KV 双写过渡期兼容历史附件。上游如改动该文件，合并时保留此逻辑。

## OPC 隔离实例（OPC-design 分支，2026-10-25 建）
- 分支 `OPC-design`：自 deploy 创建并推送 origin；**永不合入 main**（main 是 fork 同步源，且 push main 会触发 `.github/workflows/deploy-cloudflare.yml` 用 wrangler-action.toml 误部署）。同步链路固定 `upstream/main → deploy → OPC-design`，wrangler-opc.toml 为新文件、sync 零冲突，`[local]` 补丁经 deploy 自动继承。
- Worker `cloud-mail-opc`，域名 `mail.supracept.work`，`workers_dev = false`，cron 每小时。配置在 `mail-worker/wrangler-opc.toml`；package.json 有 `deploy:opc` script。
- **唯一合法部署命令：`wrangler deploy -c wrangler-opc.toml`**；严禁不带 `-c`（wrangler.toml 仍指向生产 jaspermail 资源，误部署会把代码推到生产 Worker）。
- 资源与生产完全隔离：D1 `opcmail` / `add32c26-0da8-4e93-b6fb-7ad047aeb682`（APAC）；KV `opcmail-kv` / `ec79b98867aa426a8ed060bddbb6f507`；R2 桶 `jasper-opcmail`。`jwt_secret` 走 cloud-mail-opc 的 Dashboard Secret（值与生产不同，勿记录到记忆文件）。
- 迁移约定：OPC 实例每次部署后访问 `https://mail.supracept.work/api/init/<OPC_SECRET>`（幂等，带 /api 前缀）。
- supracept.work apex 预留给另一 OPC web SaaS：SaaS **不得使用 `*.supracept.work` 通配 Worker 路由**（通配 DNS 无害，具体记录优先）；Email Routing catch-all 为 zone 级唯一规则，如需收件归 cloud-mail-opc，与 SaaS 协商。
- 本地部署坑：pnpm 10 的 `pnpm --prefix ../mail-vue install` 静默 no-op（不建 node_modules），wrangler `[build]` 段会 vite not found；先 `cd mail-vue && pnpm install` 再部署。Cloudflare Linux 构建环境无此问题。
- 遗留（需用户操作）：Dashboard 连接 Workers Builds 监听 OPC-design（Root=mail-worker，Deploy=`npx wrangler deploy -c wrangler-opc.toml`）；浏览器注册 admin@supracept.work 后关闭注册；如需收件再配 Email Routing。
