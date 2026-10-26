# web-vben Agent Rules

本文件是 `web-vben` 的强制工作约定。任何 Agent 在修改本仓库前必须先阅读本文件，并在完成后按验证清单执行。违反这些边界会直接增加 Vben 上游升级冲突，禁止以“先改了再说”作为理由绕过。

## 1. 仓库身份与当前事实

- 本仓库是 Vben Admin monorepo，当前实际应用是 `apps/web-antd`，组件库为 Ant Design Vue。
- `api-go` 是独立 Go API 仓库；前端通过 `/api/v1` 调用它。
- 团队 fork 已确定为 `https://github.com/omwk/vue-vben-admin`。约定：`origin` = 团队 fork，`upstream` = 官方 Vben；修改 remote 后必须用 `git remote -v` 和 `git ls-remote` 验证。
- 上游基线必须记录完整 Git SHA。package version 只能辅助说明，不能代替 SHA。

## 2. 目录所有权

业务默认只允许写入：

- `apps/web-antd/src/api/**`：业务 API 适配
- `apps/web-antd/src/views/**`：业务页面
- `apps/web-antd/src/router/routes/modules/**`：业务路由
- `apps/web-antd/src/locales/**`：业务文案
- `apps/web-antd/src/preferences.ts`：应用级偏好覆盖
- `apps/web-antd/.env.*`、`apps/web-antd/vite.config.ts`：应用环境和开发代理
- `docs/upgrades/**`：升级记录和方案

以下目录默认视为上游维护区，不得用于承载业务代码：

- `packages/**`
- `internal/**`
- `apps/web-antd/src/typings/**` 中的共享 Vben 类型
- Vben 核心路由、权限、请求客户端和布局实现

确需修改上游维护区时，必须单独提交，说明：上游是否已有等价实现、修改原因、影响范围、对应测试、未来清理或回迁计划。业务需求不得通过复制或长期 fork 核心包实现。

## 3. 前后端契约

- API base 固定为 `/api/v1`；开发代理必须保留 `/api/v1` 原始路径并指向 Go API。
- 真实后端开发时 `VITE_NITRO_MOCK=false`。mock 只用于明确的离线测试，不得与真实 API 混用或作为“已经接通”的证明。
- Vben 核心接口保持兼容：`POST /auth/login`、`POST /auth/refresh`、`POST /auth/logout`、`GET /auth/codes`、`GET /user/info`、`GET /menu/all`。
- 成功响应使用 `{ code: 0, data, error: null, message: "ok" }`；认证失败使用 401，权限不足使用 403，不得用 HTTP 200 隐藏失败。
- OpenAPI 以 `api-go/api/openapi/v1.yaml` 为契约源；生成的 TypeScript/Go 文件不得手工修改。契约变更必须记录配套 `api-go` 完整 commit SHA。
- `/user/info` 必须符合 Vben `UserInfo` 类型；动态菜单必须返回真实存在的 `apps/web-antd/src/views/**` 页面映射。
- 前端隐藏菜单和权限码只是展示控制；真正的授权必须由 Go API 按用户、角色、资源和动作执行。

## 4. 新业务开发

1. 从 `main` 创建 `feat/<domain>/<short-description>` 分支。
2. API 调用放在应用 API 层，页面放在业务 views，路由放在业务 routes；不要把业务逻辑写进共享包。
3. 复用现有 Vben/Ant Design Vue 组件、请求客户端和类型；新增依赖前检查 workspace 是否已有能力。
4. 不要修改 `pnpm-lock.yaml`、共享包或 Vben 核心配置来解决单个页面问题；确需修改时写明升级影响。
5. 使用 Conventional Commits，例如 `feat(web): add customer list`、`fix(web): handle expired access token`。

## 5. 上游升级流程

- 每两周检查 `upstream/main`、release notes、安全修复和阻塞性 bug。
- 每次升级先固定一个完整 SHA，创建：`upgrade/vben/<YYYY-MM-DD>-<full-sha>`。
- 从团队 `main` 开始，执行 `git fetch upstream`，将选定 SHA 合并到升级分支；保留 Git 历史，禁止把上游目录整体复制覆盖本地改动。
- 上游合并与本地兼容修复尽量分成独立提交。逐文件解决冲突，优先保留 `apps/web-antd/src/**` 的扩展。
- 每次升级必须复制 `docs/upgrades/template.md`，记录旧/新 SHA、版本、冲突文件、适配理由、API 契约 SHA、检查结果、跳过原因和部署结果。
- 升级失败或验证失败时，`main` 保持上一条可部署基线；不得把未验证分支推到生产分支。
- 升级完成后通过 PR 合并到团队 `main`，禁止直接在 `main` 上拉取上游或解决大规模冲突。

## 6. 必须执行的验证

前端改动至少执行：

```bash
pnpm install --frozen-lockfile
pnpm check
pnpm run build:antd
```

涉及 API、认证、菜单、权限或请求客户端时，还必须：

```bash
go test ./...                  # 在 api-go 仓库
go vet ./...
```

并验证真实业务链路：登录、用户信息、权限码、菜单、过期 token、无权限请求。每次升级记录所有检查结果；跳过检查必须写明具体原因，必需检查失败不得标记为完成。

## 7. 禁止事项与停止条件

- 禁止提交 `.env`、token、JWT 密钥、数据库密码或私有仓库凭据。
- 禁止删除或重写团队 `main` 历史，禁止 `git push --force` 作为升级手段。
- 禁止把 mock 数据、mock 服务响应或本地绕过逻辑带入生产配置。
- 禁止在不知道 fork URL 时修改 remote 或声称远端已更新。
- 发现工作树有与当前任务无关的用户改动时，先保留并避开；不得 reset、checkout 或删除。
- 发现上游升级会要求长期修改 `packages/**` 或 `internal/**` 时，停止并先写出替代方案与冲突记录。

## 8. 完成前自检

- [ ] 改动是否集中在应用扩展区？
- [ ] 是否误开启 Nitro mock 或改动 `/api/v1` 契约？
- [ ] 是否记录了 API 配套 commit SHA？
- [ ] 是否执行并记录了必要验证？
- [ ] 是否检查 `git diff --check`、`git status` 和实际 remote？
- [ ] 是否留下可重复的升级记录，而不是只依赖聊天上下文？
