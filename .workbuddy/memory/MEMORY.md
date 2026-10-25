# 项目长期约定

## 分支
- `main`：fork 同步源（upstream），不部署。
- `deploy`：唯一部署分支，Cloudflare Workers Builds 监听它。
- `backup-main`：本地备份，不参与同步。

## wrangler.toml（mail-worker）
- 每轮 sync 必冲突。本地值固定为：D1 `jaspermail` / `0b186691-fb70-42ea-8944-5a8d2fcae097`，KV `e27cda1bbbef4fddb0726b63c75810e6`，R2 桶 `jaspermail`（binding `r2`）。
- 冲突处理原则：只替换 `[[d1_databases]]`、`[[kv_namespaces]]`、`[[r2_buckets]]` 三块的取值，其余保持上游。

## 部署拓扑（必须写进 wrangler.toml，否则会被 wrangler 摘除）
- 自定义域名：`[[routes]] pattern = "mail.jaspershadow.dpdns.org" custom_domain = true`。
- `jwt_secret` 不入库，走 Dashboard Secret（wrangler deploy 恒为 `keepSecrets: true`，不会被清空）。

## 本地补丁（需在 sync 时保留）
- `mail-worker/src/service/r2-service.js` 中带 `[local]` 注释的 KV 回退读取（R2 未命中时回落 KV），用于 R2/KV 双写过渡期兼容历史附件。上游如改动该文件，合并时保留此逻辑。
