# 架构概览

本文档补充 `AGENTS.md` 第 3 节"仓库速览"的高层指针，沉淀本仓库关键架构模式的实现语义、入口与边界，供新贡献者快速理解底层约束。

> 维护原则：本文档对应的是实现语义和边界条件；如果实现变化但本文档没有同步更新，以实际可执行代码为准并顺手修正文档。

## 数据源架构

`data_provider/` 使用优先级链式 Strategy 模式。

- 入口：`DataFetcherManager.get_daily_data()`（`data_provider/base.py`）。
- 按优先级依次尝试各 fetcher，单源失败自动 failover；每个市场（A 股 / 港股 / 美股）有独立的数据源优先级链。
- 内置熔断器 `CircuitBreaker`（`data_provider/realtime_types.py`）跟踪各源健康状态，自动跳过近期失败的源。
- 标准输出列：`['date', 'open', 'high', 'low', 'close', 'volume', 'amount', 'pct_chg']`。

修改 `data_provider/` 时必须保留：provider 优先级、字段标准化、超时 / 重试 / 缓存策略、graceful degradation。详见 `AGENTS.md` §7 稳定性护栏 → "数据源与 fallback"。

## LLM 路由

- 使用 `litellm` 统一路由到多个 LLM provider（Gemini、Anthropic、OpenAI、DeepSeek、MiniMax 等）。
- 通过 `LLM_CHANNELS` 环境变量配置渠道；具体 provider / model / Base URL / fallback 由 `LLM_CHANNELS` 解析后写入运行时 config。
- 各 provider 的 preset 与诊断建议详见 `docs/llm-providers.md`、`docs/LLM_CONFIG_GUIDE.md`。

## 配置热加载

- 配置单例通过 `src.config.get_config()` 获取，初始化逻辑在 `setup_env()`（`src/config.py`）。
- 定时调度模式下，每次进入 `scheduled_task()` 都会调用 `_reload_runtime_config()`（`main.py`）：
  1. 读取当前 `.env` 文件值；
  2. 对 `.env` 管理的键更新 `os.environ`，但**保留进程环境变量的覆盖**（启动前已在 `os.environ` 里的键不会被 `.env` 覆盖）；
  3. 调用 `Config.reset_instance()` 让单例重建，下游 `get_config()` 重新加载；
  4. `run_full_analysis(runtime_config, ...)` 用新 config 重新构造 pipeline。
- 非调度模式（单次 / API）下配置在进程启动时初始化一次，运行时不变。

## 数据库

- 默认 SQLite 数据库路径：`./data/stock_analysis.db`。
- 通过 SQLAlchemy 2.0+ ORM 访问。
- 默认启用 WAL 模式以支持并发读写，由 `Config.sqlite_wal_enabled`（默认 `true`，环境变量 `SQLITE_WAL_ENABLED`）控制；具体 PRAGMA 在 `src/storage.py::_initialize_sqlite_connection`。
- `DatabaseManager` 是基于 `metaclass=_DatabaseManagerMeta` 的单例（`src/storage.py`），全进程共享一个连接池入口。

## 前后端关系

- **开发模式**：Vite 开发服务器（默认端口 5173）通过代理 `/api` → `http://127.0.0.1:8000` 转发 API 请求到 FastAPI（配置见 `apps/dsa-web/vite.config.ts`）。
- **生产构建**：Vite 产物输出到 `/static/`，由 FastAPI 直接托管静态文件；前端路由 `/api/*` 由 FastAPI 路由处理。
- 桌面端（`apps/dsa-desktop/`）内嵌 Web UI 构建产物，并由 Electron 主进程拉起 Python 后端子进程；本地后端 URL 仍为 `http://127.0.0.1:8000`。