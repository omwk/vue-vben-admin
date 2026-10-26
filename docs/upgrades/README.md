# Vben 上游升级规范

本目录保存每次 Vben 上游升级的操作规则和审查记录。规则的目标是让 `apps/web-antd` 的业务扩展与 Vben 核心保持清晰边界，使上游升级可以逐次合并、验证和回滚。

## 远端约定

当前远端约定：

```text
origin   -> https://github.com/omwk/vue-vben-admin
upstream -> https://github.com/vbenjs/vue-vben-admin.git
```

团队 fork 已建立。修改远端后必须用 `git remote -v` 和 `git ls-remote origin main` 验证，禁止把官方仓库当作可推送的 `origin`。

## 两周升级一次

1. 从团队 `main` 创建 `upgrade/vben/<YYYY-MM-DD>-<full-sha>`。
2. `git fetch upstream`，查看 release notes 和变更范围。
3. 固定一个完整上游 commit SHA，记录升级前 SHA、root 版本和 `apps/web-antd` 版本。
4. 将该 SHA 合并到升级分支，保留 Git 历史，不使用整体复制覆盖。
5. 逐个解决冲突。业务适配优先放在 `apps/web-antd/src/**`；共享包冲突必须说明原因和清理计划。
6. 复制 `template.md` 创建本次升级记录，填满所有字段。
7. 完成前端、后端和真实 API 业务验证后，创建 PR 合并到团队 `main`。
8. 升级失败时关闭升级 PR，保留上一条可部署 `main`，不得强行合并。

## 目录边界

业务代码放在 `apps/web-antd/src/api`、`src/views`、`src/router/routes/modules`、`src/locales` 和应用配置中。`packages/**` 与 `internal/**` 是上游维护区，只有经过单独审查的必要修复才能修改。

## API 配套

真实后端使用 `/api/v1` 和 Go API。mock 只用于离线测试，真实开发必须关闭 `VITE_NITRO_MOCK`。OpenAPI 变更必须记录配套 `api-go` 完整 SHA，并重新生成客户端类型，禁止手工编辑生成物。

## 验证命令

```bash
pnpm install --frozen-lockfile
pnpm check
pnpm run build:antd
```

涉及 API 或认证时，在 `api-go` 仓库追加 `go test ./...`、`go vet ./...`，并验证登录、用户信息、权限码、动态菜单和失效 token。所有跳过项都必须写明原因。

每次升级都必须填写 [template.md](./template.md)，不能依赖聊天记录解释冲突或验证结果。
