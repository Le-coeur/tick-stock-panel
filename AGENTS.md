# AGENTS.md — AI 编码代理开发入口

本文件面向 AI 编码代理，假设读者对本项目一无所知。修改、调试或审查本仓库前，请先读本文件建立整体认知；**动手改代码前必须完整阅读根目录的 [`CONTRIBUTING.md`](CONTRIBUTING.md)**(项目贡献、数据契约、验证矩阵与 PR 复审的唯一权威规范)，涉及二次开发时再读 [`docs/secondary-development.md`](docs/secondary-development.md)。

## 1. 项目概览

**TSP(tick-stock-panel)**——自托管、零运维的 A 股「选股 + 监控 + 回测」量化工作台(个人开源，仅供学习研究，MIT License，当前版本 v0.2.3)。

- **后端**：`backend/`，Python ≥ 3.11 + FastAPI + Pydantic v2 + APScheduler + sse-starlette；包管理用 **uv**(`uv.lock` 锁定)。
- **数据**:Polars(计算引擎，版本锁定 `>=1.44,<1.45`，升级需重跑 `backend/scripts/stress_polars_concurrency.py` 压测)+ DuckDB + Parquet + PyArrow;pandas 仅限回测服务边界使用；`vectorbt` 为可选 extras。
- **前端**:`frontend/`,React 18 + TypeScript + Vite + Tailwind CSS + TanStack Query + ECharts + lightweight-charts + dnd-kit；包管理用 **pnpm 9**(`pnpm-lock.yaml` 锁定，无 npm/yarn)。
- **部署**:Docker 两阶段构建，前端 dist 拷进后端镜像，**单容器**运行，默认端口 `3018`;GitHub Actions 自动构建多架构镜像推送到 GHCR。
- **AI(可选)**:OpenAI 兼容接口(DeepSeek / 通义 / Ollama 等),`AI_API_KEY` 留空即关闭。

## 2. 架构与主数据流

```text
数据源或数据插件
  -> services 同步、校验与标准化
  -> DataStore / KlineRepository / Parquet
  -> indicators pipeline 生成 enriched 数据
  -> 策略 / 监控 / 回测 / 分析服务
  -> FastAPI API / SSE
  -> frontend/src/lib/api.ts 类型 + TanStack Query
  -> 页面和共享组件
```

关键机制：多数据源**能力路由矩阵**(`backend/app/data_providers/capabilities.py` 注册表，单一权威，fail-closed)、交易日探针(自动停掉节假日实时轮询)、财务多源合并(PIT 口径，公告前一律空值不填 0)、回测在 spawn worker 子进程隔离执行(SSE 流式进度)、分层缓存(enriched 存 15 列基础数据、读取时现算 68 列指标)。

## 3. 代码组织

### 后端 `backend/app/`(约 25 个一级模块)

| 目录 | 职责 |
| --- | --- |
| `api/` | HTTP/SSE 薄层，只做参数校验与响应映射，不放重计算 |
| `services/` | 业务编排(同步、实时行情、监控、挖掘、通知等 50+ 服务) |
| `tickflow/` | 存储、仓库、能力检测与 TickFlow 接入 |
| `data_providers/` | 数据源接口、标准化模型、能力路由、custom 自定义源 |
| `plugins/` | 插件发现与加载；内置 `fuyao`(同花顺 REST)与 `stocksdk` 示例插件 |
| `indicators/` | 指标计算与 enriched 流水线(保持向量化) |
| `strategy/` | 策略注册/执行/监控/AI 生成；`builtin/` 下 25 个内置策略 |
| `backtest/` | 回测/优化/步进优化/worker;信号生成与成交模拟严格分离 |
| `factors/` | 因子 DSL、注册表、因子库版本管理 |
| `extensions/`、`custom/` | 后端扩展加载器与自定义代码模板 |
| `jobs/`、`config.py`、`main.py` | 定时任务、配置、FastAPI 入口 |

### 前端 `frontend/src/`

- `pages/` — 页面级编排(看板、策略、回测、因子、监控、复盘、设置等 25+ 页);`lib/api.ts` 是所有请求的封装与类型契约；`lib/queryKeys.ts` 集中维护 TanStack Query 查询键；`components/` 放可复用组件；`extensions/` 是前端受控扩展插槽(registry/Slot/Boundary)。
- 约定：禁止组件散落请求地址；复杂逻辑下沉到 hook;契约变化必须同步更新 `api.ts` 类型。

### 其他

- `docs/` — 15 篇领域文档(部署、配置、数据源插件、策略、因子、挖掘、二次开发等，改动相关契约时需同步更新)。
- `backend/tests/` — 169 个 pytest 测试文件，含 `fixtures/`;`backend/scripts/` — 压测、探测、清理等运维脚本。
- `packaging/` — PyInstaller 桌面版打包;`docker-compose.yml` / `Dockerfile` — 单容器部署(默认不含 stock-sdk 插件，`--build-arg INCLUDE_STOCKSDK=1` 显式开启);`tiers.yaml` — TickFlow 套餐频率对照表(仅启动探测与 UI 标签用，业务代码不读)。

## 4. 构建与运行

前置:Python ≥ 3.11、Node ≥ 20、`uv`、`pnpm`。

```bash
cp .env.example .env        # 配置 TICKFLOW_API_KEY(留空=None 模式)、AI_API_KEY、PORT 等
./dev.sh                    # Windows 用 .\dev.ps1;一键起前后端(自动装依赖、释放端口)
# 后端 http://localhost:3018  前端 http://localhost:3011

docker compose up --build   # 或直接用 GHCR 现成镜像:ghcr.io/shy3130/tick-stock-panel:latest
```

所有配置从根目录 `.env` 读取；完整配置项见 `docs/configuration.md`。

## 5. 测试与验证

```bash
cd backend
uv run pytest tests/path/to/test_x.py -q   # 至少跑受影响模块的全部测试
uv run ruff check app/path.py tests/path.py  # 新改动不得引入 Ruff 告警
cd frontend && pnpm build                   # 任何 TS/组件/样式/API 类型改动必跑
git diff --check                            # 提交前必跑
```

验证矩阵(最低要求，详见 CONTRIBUTING §9):bug 修复先写复现测试；金融数值改动用固定样本断言并覆盖边界/降级路径；数据源改动补契约、缺能力、空数据、字段单位标准化测试；回测改动覆盖信号日/成交日、T+1、费用、滑点、不可成交；前端改动覆盖加载/空/错/切换/实时刷新，并按改动范围在桌面与窄屏尺寸检查。

## 6. 不可违反的数据契约(金融口径，出错不抛异常只给错误结果)

- **比例/百分比**:`change_pct`、`turnover_rate` 在不同边界有小数制与百分数制之分(见 CONTRIBUTING §3.1 对照表),跨边界映射必须加单位测试，禁止"小于 1 就乘 100"的启发式转换。
- **价格与复权**:enriched 的 `open/high/low/close` 是前复权价，`raw_*` 是不复权原始价；涨跌停/一字板/炸板判定必须基于原始价。
- **日期与时区**:窗口/前 N 日按实际交易日不用自然日；统一北京时间(naive 墙钟),服务器时区不得成为业务输入；分钟 K 的股票/ETF/指数分开存储路由。
- **PIT**:财务按公告日取最新，公告前空值绝不填 0；历史换手率优先用公告日不晚于目标交易日的历史股本。

## 7. 插件化与扩展红线

- 通用功能必须通过 `get_provider()` / `provider_has_dataset()` 与 preferences 路由访问数据；禁止把 TickFlow SDK 调用硬编码进策略、监控、回测、API 或前端。
- provider 负责供应商字段/单位/日期/代码格式到内部标准的转换；缺能力时明确提示、跳过或 fail-closed，禁止静默换用错误数据。
- 插件加载失败不得导致应用启动失败或其他数据源不可用；只有明确标注 TickFlow 专属的功能才能直连 TickFlow。
- 新增数据源适配须同步更新 `docs/custom-data-source.md` / `docs/plugin-development.md`。

## 8. 工作规则(对所有改动生效)

- 先定位真实调用链、数据来源、缓存层和现有测试再修改；不凭文件名或界面现象猜测。
- 最小范围修改：不顺手重构、重命名、格式化无关代码；保留工作区他人未提交的改动。
- 注释解释"为什么"，不复述代码行为。
- 写数据后同步检查：持久化文件、内存缓存、generation、SSE 事件、前端 query 失效。
- 共享状态用锁/不可变快照/原子替换；持锁期间不做网络请求或全量扫描；实时热路径不得随历史数据量线性增长。
- 不提交 `data/`、密钥、Token、Webhook、用户策略、日志、Parquet 样本和机器绝对路径；文件删除/覆盖/批量回算一律 fail-closed。
- 配置、策略 JSON、Parquet schema、API 字段变化默认向后兼容(新增字段给默认值，允许历史数据缺字段)。
- 不虚构测试、性能数据或审查结果；无法执行的验证必须明确标注。不自动执行 git 提交/推送/合并等操作。

## 9. 文档地图

| 场景 | 文档 |
| --- | --- |
| 贡献规范 / 数据契约 / PR 模板与复审 | `CONTRIBUTING.md`(必读) |
| 二次开发、前端插槽、后端策略接口 | `docs/secondary-development.md` |
| 部署、访问密码、老 CPU 兼容 | `docs/deployment.md`、`docs/deploy-password.md` |
| 全部 `.env` 配置项 | `docs/configuration.md` |
| 自定义数据源 / 插件开发 | `docs/custom-data-source.md`、`docs/plugin-development.md` |
| 策略体系 / AI 策略迭代 | `docs/strategy.md`、`docs/strategy-iteration.md`、`backend/app/strategy/prompts/strategy-guide.md` |
| 因子平台 / 因子挖掘 / 市场阶段 | `docs/factor-platform-plan.md`、`docs/mining.md`、`docs/market-phase.md` |
