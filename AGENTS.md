# AGENTS.md

社区增强版 WeKnora：腾讯开源的知识管理与问答框架（RAG + Agent），本仓库在其之上增加一套
**A 股投研工具链**（行情/财务查询、形态识别、HALO 基本面评分）。

上游：https://github.com/Tencent/WeKnora ｜ 许可与第三方组件：`NOTICE.md` ｜ 上游 README：`README-upstream.md`

**默认部署与上游等价**：投研栈是可选 profile（`docker compose --profile finance`），不启用时
不启动 python-service、不挂载任何数据目录，知识库与对话主流程完全不受影响。

Monorepo：根 Go module 是主服务；另有若干独立构建的模块（各自 lockfile、各自 workflow）。
除下面的投研栈之外，其余目录基本沿用上游——本仓库不携带上游提交历史，对比差异的做法见
「PR & commit conventions」。

## Setup commands

Dependencies are per-module. Run commands from the module root, not the repo root, except `make`.

```bash
# Root Go module (server) — cmd/server
make deps             # go mod download
make build            # go build -o WeKnora ./cmd/server (binary lands at repo root)
make build-prod       # versioned, stripped, CGO on
make build-lite       # single binary: sqlite + sqlite_fts5 + local file storage
make build-anydoc     # links the in-process Rust parser (needs cargo)
make run              # build, then run
make fmt && make lint # go fmt ./... ; golangci-lint run

# cli/ — separate Go module, Cobra CLI. make lint is `go vet ./...`, not golangci-lint.
cd cli && make build && make test && make test-coverage && make lint

# client/ — separate Go module, REST SDK
cd client && go build ./... && go test -race ./...

# frontend/ — Vue 3 + Vite + tdesign-vue-next. npm, not pnpm, despite pnpm-workspace.yaml.
cd frontend && npm install && npm run dev      # also: npm test, npm run type-check, npm run build

# docreader/ — Python parsing service (:50051), uv-managed
uv sync --project docreader
uv run --project docreader python -m unittest discover -s docreader/tests -p "test_*.py" -v

# python-service/ — 投研数据服务，pip/pytest（仅 finance profile 需要）
cd python-service && pip install -r requirements.txt && python -m pytest
```

Local dev loop: `make dev-start` (docker-compose.dev.yml infra) → `make dev-app` (Air hot-reload on
`:18080`) → `make dev-frontend`; `make dev-stop` tears down.

## Project layout

- `cmd/` — binaries: `server` (primary), `desktop` (Wails), `download`
- `internal/` — all server code: `handler` → `application/service` → `application/repository`, plus `agent`, `models`, `stream`, `mcp`, `container` (dig DI), `router`, `middleware`, `types`
- `migrations/versioned/` — `golang-migrate`, sequential numeric filenames, per-DB subdirs (sqlite/mysql/paradedb)
- `config/` — `config.yaml`, `builtin_models.yaml` (+ `models.json` overlay)
- `cli/`, `client/`, `frontend/`, `docreader/`, `python-service/`, `mcp-server/`, `miniprogram/`, `packages/` — separate modules (see above)
- `scripts/` — repo utilities, `scripts/git-hooks/`, `scripts/model-catalog/`
- `deploy/`, `docker/`, `helm/` — container and chart packaging
- `testdata/`, `third_party/`, `licenses/` — fixtures, vendored code, third-party notices
- `docs/`, `website-docs/` — long-form documentation and the docs site

## Code style

- Go: `gofmt` + `gofumpt`, 120-char line limit (`.golangci.yml`, linters `lll`/`govet`/`revive`)
- CI checks formatting on **changed lines only** (`base...head`), not the whole tree
- Vue/TS: `vue-tsc --build` for type-check; there is no ESLint/Prettier config — match surrounding
  component style
- Python: stdlib `unittest` in `docreader`, `pytest` in `python-service`; no ruff/mypy config present
- All dependency injection goes through `uber/dig` in `internal/container/container.go` — never
  construct services inline
- Config: Viper reads `config/config.yaml` with `${ENV}` substitution. `.env.example` (sections A–J)
  is the canonical env reference — add new env vars there

## 投研栈（本 fork 的增量）

Go 侧工具族：`internal/agent/tools/hithink_finance/**`（薄封装）、`halo/**`、`zettaranc/**`；
Python 侧：`python-service/**`（含 `halo/`、`zettaranc/`）；前端：`frontend/src/components/workspace/kline/`。

不变量，每条都有回归测试兜底：

- **`period` 不是报告期**。在 `v_balance_sheet` / `v_income_statement` / `v_cash_flow_statement`
  里它装的是*报表口径*（`annual` / `quarterly`，约 17.7 万行同值）。真正的报告期是
  `fiscal_year` + `fiscal_period`（`Q1`–`Q4`、`FY`），排序用 `period_end_ms`；
  `report_date_ms` 被同步污染（多个报告期共享同一时间戳）。共用排序在
  `financial/period.go`，有回归测试——**不要在某个查询里重新内联 `ORDER BY period`**。
- **`v_financial_indicators_detail.report` 是 `YYYY-N`**（`2026-2`），不是 `YYYY-QN`；
  `FY` 必须映射成 `-4`，否则 join 静默返回空。
- **Python 拥有每一个数字**。`AI` 只填七个定性维度（moat / stag / ESG / management /
  shareholder / valuation / risk），`halo.analyze` 只返回定量锚点，`halo.verify` 复算合成分。
  LLM 做算术是 bug 来源——风险项以 `(10 − risk) × 0.10` 进入，符号写错会静默改变总分。
- **缺失 ≠ 零**。要么 `halo.ok=false` 带原因，要么 `⚠️ 缺失` 标记；**绝不**换一个口径顶替。
  员工人数只在年报里，因此**HALO 在半年报上本来就不可计算**——如实说，不要外推。
- **读表格的单位**。分部营收表有的公司用元、有的用百万；没有 `单位：` 声明就**跳过该页**，
  不要按 1e6 硬猜。
- **对账是过滤器不是法官**。`status` 只有 `verified`（两路一致，或抽取字段自身与 DuckDB 对账）、
  `disputed`（被证伪，**绝不**因为"抽取器通常是对的"而提升）、`pending`。
- **`analyze()` 永远返回同样的形状**，包括最常见的"还没同步到财报"路径，否则调用方会
  拿到 `KeyError` 而不是"暂无数据"。
- **工具注册是白名单**（`agent_service.go`）：不在 agent 的 `allowed_tools` 里的工具既不在
  模型 schema 里也不可调用。
- **同步窗口可配**。`CheckSyncWindow` 在窗口内直接拒绝查询（避免撞 DuckDB 写锁），窗口由
  `HITHINK_SYNC_WINDOWS` 指定（`HH:MM-HH:MM[,…]`）：未设置用代码默认值，显式置空 = 不拦截。

`testdata/schema.json` 是 schema 快照，由 `schema_contract_test.go` 把关：它扫描工具源码里的
反引号 SQL，校验每个被引用的表/列。改任何硬编码 SQL 后都要跑它——不需要活的 DuckDB 就能
发现漂移。

外部 HTTP：cninfo 有限速（`HALO_CNINFO_MIN_INTERVAL`，默认 0.5s），因为它会封 IP——
社区实测阈值是 5 分钟 300 次请求。注意 `datacenter-web.eastmoney.com` 与
`push2his.eastmoney.com` 在**不同 WAF** 后面：一个被封不代表另一个不可用。

数据不随仓库分发：默认挂载仓库内的空占位目录 `.hithink-placeholder`，
设 `HITHINK_DB_DIR` 指向本机库目录后才启用。

## Testing instructions

- Root module: `make test` (`go test -v ./...`); single package with
  `go test ./internal/agent/... -run TestName -count=1`
- Build-tagged: `go test -tags anydoc ./...` and `go test -tags desktop ./internal/container`
- CI excludes `docreader` from the root sweep; its Python and Go-client tests run separately
- Concurrency-sensitive changes: `go test -race`
- In-memory infra only — `miniredis` for Redis, `sqlmock` for SQL, `gorm.io/driver/sqlite` +
  `sqlite-vec` for vectors. **No testcontainers.**
- Tests use `testify`: `require` for setup/teardown, `assert` for assertions
- Integration tests use the `_integration_test.go` suffix and respect `-short`
- Before pushing: `make model-catalog-check` if you touched `config/` or `scripts/model-catalog/`

## PR & commit conventions

本仓库是 Tencent/WeKnora 的社区分叉，独立维护，**不携带上游的提交历史**。

- **工作分支就是 `main`** —— 这是本仓库自己的代码，直接在上面提交
- **不要向上游推送任何东西**。面向上游核心的修复（`internal/`、`frontend/`、`docreader/` 等
  共通部分）请直接提给 [Tencent/WeKnora](https://github.com/Tencent/WeKnora) 的 issue/PR
- 需要对比或合并上游时，自己挂远端，**用 merge 不要 rebase**：

  ```bash
  git remote add upstream https://github.com/Tencent/WeKnora.git
  git fetch upstream
  git diff upstream/main...HEAD     # 看差异
  git merge upstream/main           # 或同步
  ```

- 本仓库增量功能的 issue / PR 欢迎
- Conventional Commits，带 scope：`feat(agent/tools):`、`fix(frontend):`、`chore:`、`docs:`
- Pre-commit / pre-push 钩子与 CI 对齐。跳过：`HOOK_SKIP_TEST=1`（跳测试）或
  `SKIP_HOOKS=1`（全跳）——见 `scripts/git-hooks/`
- CI 只 lint 新增代码（相对基线），无关的重新格式化请单独开一个提交

## Security

- **绝不提交密钥**。`.gitignore` 忽略所有点文件，只放行 `.env.example`、`.gitignore`、
  `.github/` 与 `.hithink-placeholder/`（finance profile 的占位挂载目录）；`.env`、`.env.lite`
  等一律留在本地
- **警告**：由于上面这条 `.*` 规则，往一个**全新 clone** 里 `git add` 点文件会被静默忽略。
  仓库里现有的点文件（`.golangci.yml`、`.gitattributes`、`.air.toml`、`.env.lite.example` 等）
  已进索引、不受影响；新增点文件时需要 `git add -f` 或补一条 `!` 例外
- `cli/` 的 CI 会跑 `scripts/check-secret-tokens.sh` 扫描文档中的凭据
- `scripts/check-license-bundle.sh` 与 `test-license-bundle.sh` 把关 `THIRD_PARTY_NOTICES.md`——
  改依赖后要跑
- 漏洞请按 `SECURITY.md` 报告，不要开公开 issue
