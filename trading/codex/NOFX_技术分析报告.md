# NOFX 技术分析报告

> 分析对象路径：`/home/zcxggmu/workspace/hello-projs/posp/trading/nofx`
>
> 报告目标：基于代码实现进行深度技术结构剖析（后端/前端/数据/安全/部署）。

---

## 一、项目概述

NOFX 是一个面向加密货币合约交易的 AI 自动化交易平台，集 **策略配置、AI 决策、实盘交易、回测与对比、AI 辩论** 于一体。平台提供 Web 界面进行配置和监控，后端以 Go 为核心，前端以 React + Vite 为核心。系统支持多家交易所与多家 AI 模型服务的接入。

**核心技术栈（从代码与构建信息归纳）：**
- **后端**：Go + Gin；SQLite（modernc.org/sqlite）；自定义日志（logrus）。
- **前端**：React 18 + TypeScript + Vite + TailwindCSS；SWR / Zustand / axios。
- **AI 客户端**：DeepSeek、Qwen、OpenAI、Claude、Gemini、Grok、Kimi（`mcp/`）。
- **交易所**：Binance、Bybit、OKX、Bitget、Hyperliquid、Aster、Lighter（`trader/`）。
- **部署**：Docker Compose（前后端分离、Nginx 反代）。

> 入口主函数：`main.go`。API 路由在 `api/server.go`。

---

## 二、代码结构与职责边界

> 以目录维度拆分系统模块

- `api/`：HTTP API 与路由编排（Gin），包含交易、模型、策略、回测、辩论等接口。
- `auth/`：JWT、密码哈希、TOTP OTP。
- `backtest/`：回测引擎（数据馈送、AI 调用、账户仿真、指标、持久化）。
- `config/`：全局配置（环境变量）。
- `crypto/`：AES-GCM 存储加密 + RSA-OAEP 传输解密。
- `debate/`：多 AI 辩论引擎（多轮投票、共识）。
- `decision/`：策略引擎、提示词构造、AI 输出解析与校验。
- `experience/`：匿名遥测（Google Analytics 事件）。
- `hook/`：可插拔钩子（HTTP Client、特定交易所对象）。
- `logger/`：统一日志封装。
- `manager/`：多交易员管理、竞赛缓存。
- `market/`：行情与指标计算，Kline 拉取。
- `mcp/`：AI 模型客户端封装（多提供商）。
- `provider/`：外部数据源（AI500、OI Top、CoinAnk）。
- `security/`：URL/SSRF 保护。
- `store/`：SQLite 数据层（实体 + 迁移 + 加密）。
- `trader/`：交易执行与各交易所适配。
- `web/`：前端 UI。包含 Vite、Tailwind、组件/页面/状态管理。
- `docker/`、`nginx/`：部署构建与反向代理配置。
- `scripts/`：运维或数据修复脚本（清理订单、迁移加密等）。

---

## 三、整体运行架构（部署视角）

```
┌─────────────────────────┐        ┌──────────────────────────┐
│        浏览器            │        │        外部服务           │
│  React SPA + WebCrypto  │        │ AI 模型 / 交易所 / 数据源 │
└─────────────┬───────────┘        └────────────┬──────────────┘
              │ HTTP(S)                               │
              ▼                                       │
┌─────────────────────────┐   /api/      ┌────────────▼────────────┐
│         Nginx            │────────────►│       Go 后端            │
│  静态资源 + API 反代      │             │ Gin API + AI/交易/回测   │
└─────────────┬───────────┘             └────────────┬────────────┘
              │                                        │
              │                                        │
              ▼                                        ▼
      /usr/share/nginx/html                   SQLite 数据库 (data/db)

```

关键特点：
- 前端与后端容器分离，前端 Nginx 负责静态资源与 `/api/` 反代。
- 后端为单体进程，内部包含 API、交易、回测、辩论、存储等子系统。
- 数据持久化主要为 SQLite（本地文件，挂载到 `./data`）。

---

## 四、后端核心子系统深度分析

### 4.1 启动与生命周期（`main.go`）

启动流程：
1. 加载 `.env`（`godotenv`）。
2. 初始化日志（`logger.Init`）。
3. 初始化配置（`config.Init`），读取 `JWT_SECRET`、`API_SERVER_PORT` 等。
4. 初始化 SQLite，创建/迁移表并注入加密函数（`store.New`）。
5. 初始化加密服务（RSA 私钥 + AES 数据密钥）。
6. 设置 JWT Secret（`auth.SetJWTSecret`）。
7. 初始化 TraderManager、BacktestManager、PositionSyncManager。
8. 从数据库加载交易员配置并（可）自动恢复运行。
9. 启动 Gin API 服务器。
10. 捕获 SIGINT/SIGTERM，优雅关闭所有交易员。

**关键点：**
- 数据库路径默认 `data/data.db`，便于 Docker 挂载。
- 位置同步 `PositionSyncManager` 默认 10s 轮询。

### 4.2 API 网关层（`api/server.go`）

- Gin Router，CORS 放开 `*`。
- 公共 API：健康检查、系统配置、加密公钥、公开竞赛数据、Kline 数据等。
- 认证 API：注册、登录、OTP 校验、完成注册、密码重置。
- 受保护 API：交易员管理、策略管理、模型/交易所配置、账户/仓位/订单/决策/统计、回测、辩论。

**认证流程**：
- 注册 → 返回 OTP Secret + QR → `complete-registration` → 发放 JWT。
- 登录 → 验证密码 → 返回 `requires_otp` → OTP 校验 → 发放 JWT。
- JWT 通过 `Authorization: Bearer` 校验；注销将 token 写入内存黑名单。

> 见 `api/server.go` 中 `handleRegister/handleLogin/handleVerifyOTP`。

### 4.3 配置系统（`config/`）

- 支持项：`JWT_SECRET`、`API_SERVER_PORT`、`REGISTRATION_ENABLED`、`MAX_USERS`、`TRANSPORT_ENCRYPTION`、`EXPERIENCE_IMPROVEMENT`。
- `ExperienceImprovement` 默认开启（触发 GA4 遥测）。

### 4.4 数据存储层（`store/`）

- SQLite 单文件数据库，`journal_mode=DELETE`，`synchronous=FULL`，`busy_timeout=5000`。
- 主要实体：
  - 用户 `users`
  - AI 模型 `ai_models`
  - 交易所 `exchanges`
  - 交易员 `traders`
  - 策略 `strategies`
  - 决策记录 `decision_records`
  - 仓位 `trader_positions`
  - 订单/成交 `trader_orders` / `trader_fills`
  - 权益曲线 `trader_equity_snapshots`
  - 回测数据 `backtest_*`
  - 系统配置 `system_config`（例如安装 ID）

- **加密策略**：AI 模型 / 交易所敏感字段可通过 AES-GCM 存储加密（`crypto.EncryptForStorage`），由 `Store.SetCryptoFuncs` 注入。

### 4.5 加密与安全（`crypto/`, `security/`）

- **传输加密**：浏览器端使用 WebCrypto（AES-GCM + RSA-OAEP）加密敏感字段。
- **存储加密**：AES-GCM（`DATA_ENCRYPTION_KEY`），存储值带 `ENC:v1:` 前缀。
- **防重放**：解密 payload 校验时间戳（5 分钟窗口）。
- **SSRF 防护**：`security.ValidateURL` + `SafeHTTPClient`，用于外部数据 API 调用。

### 4.6 交易执行与调度（`trader/`）

- `Trader` 接口统一交易操作（开/平仓、止盈止损、查询订单状态等）。
- `AutoTrader` 是核心 AI 交易执行器：
  - 周期性 `runCycle` 拉取行情/账户数据，构建上下文。
  - 调用 `decision.GetFullDecisionWithStrategy` 获取 AI 决策。
  - 对决策进行多层校验与风险控制后执行交易。
  - 记录决策日志、订单、成交与权益快照。

- 支持交易所：Binance/Bybit/OKX/Bitget/Hyperliquid/Aster/Lighter。

**风险控制（部分代码强制）：**
- 最大持仓数、最小仓位、单币仓位价值上限（`AutoTrader`）。
- AI 输出解析时也做校验（杠杆上限、止盈止损、风报比等）。

### 4.7 策略与决策引擎（`decision/`）

- `StrategyEngine` 用 `StrategyConfig` 构造系统提示词与用户提示词。
- 支持多语言提示模板（中/英）。
- AI 返回解析策略：
  - 支持 `<reasoning>` / `<decision>` 标签提取。
  - 支持 ```json 代码块提取。
  - 失败则进入安全等待 `wait` 决策。

- **决策校验**：
  - 动作合法性（open/close/hold/wait）。
  - 杠杆上限与仓位上限校验。
  - 止盈止损关系校验（多空方向）。
  - 风报比检查（内置 >=3.0）。

### 4.8 行情与数据（`market/`, `provider/`）

- **实时 Kline**：使用 CoinAnk API（`market/data.go`），内置 API Key。
- **历史数据**：回测通过 Binance Futures Kline HTTP 拉取（`market/historical.go`）。
- 技术指标：EMA / MACD / RSI 等为自实现计算（`market/data.go`）。
- **外部数据源**：AI500 币池、OI Top、OI Ranking 等（`provider/`）。

### 4.9 回测引擎（`backtest/`）

- `Manager` 管理多个回测任务，`Runner` 驱动单次回测。
- `DataFeed` 在多个 timeframes 下加载历史 K 线并驱动周期推进。
- 支持 AI 结果缓存（`AICache`）、断点恢复与运行锁。
- 指标/权益/交易记录可落库（SQLite）与文件（run dir）。

### 4.10 辩论竞技场（`debate/`, `api/debate.go`）

- 多 AI 参与（多轮）→ 票选 → 共识输出。
- 可绑定策略上下文与交易员执行器（`TraderExecutor`）。
- SSE 推送辩论实时状态（`/debates/:id/stream`）。

### 4.11 日志与遥测（`logger/`, `experience/`）

- 日志输出到 stdout + `data/nofx_YYYY-MM-DD.log`。
- 遥测默认开启，发送到 GA4（可通过 `EXPERIENCE_IMPROVEMENT=false` 关闭）。

---

## 五、核心业务流程（时序概览）

### 5.1 实盘交易决策周期

1. `AutoTrader.Run()` 周期触发。
2. 拉取账户/仓位/行情/外部指标（`market`/`provider`）。
3. `StrategyEngine` 构建系统+用户 Prompt。
4. `mcp.AIClient` 请求 AI → 返回决策。
5. 解析 + 校验 → 若通过则执行交易（`Trader`）。
6. 写入决策日志、订单、成交与权益曲线。
7. `PositionSyncManager` 定期对账，发现手动平仓/触发止损等。

### 5.2 回测流程

1. API `POST /backtest/start` 提交参数。
2. `Backtest.Manager` 创建 `Runner`，构建 `DataFeed`。
3. 时间推进：逐步生成 market snapshots，调用 AI 决策。
4. 模拟账户成交与收益曲线，存储到 backtest 表。

### 5.3 辩论流程

1. 创建 Debate Session，配置参与模型、策略与轮次。
2. `DebateEngine` 获取策略上下文 → 轮询生成讨论。
3. 投票与共识输出，可触发执行。
4. SSE 推送辩论消息/投票结果。

---

## 六、数据模型概要（关键表）

> 仅列核心字段用于理解数据流

- `users`：email、password_hash、otp_secret、otp_verified。
- `ai_models`：provider、api_key（加密）、custom_api_url。
- `exchanges`：exchange_type、api_key/secret_key（加密）、钱包/私钥字段。
- `traders`：ai_model_id、exchange_id、strategy_id、scan_interval、is_running。
- `strategies`：config(JSON)、is_active、is_default。
- `decision_records`：system_prompt、input_prompt、cot_trace、decision_json。
- `trader_positions`：symbol、side、entry/exit、realized_pnl、close_reason。
- `trader_orders` / `trader_fills`：订单生命周期与成交明细。
- `trader_equity_snapshots`：权益曲线时间序列。
- `backtest_runs` / `backtest_equity` / `backtest_trades` / `backtest_metrics`：回测数据。

---

## 七、前端架构分析（`web/`）

### 7.1 技术栈
- React 18 + TypeScript + Vite。
- TailwindCSS + Radix UI + Framer Motion。
- 状态管理：Zustand。
- 数据获取：SWR + axios（封装在 `httpClient`）。

### 7.2 页面与组件
- `App.tsx` 主路由逻辑（hash + path 混用）。
- 页面：`CompetitionPage`、`AITradersPage`、`StrategyStudioPage`、`DebateArenaPage`、`BacktestPage` 等。
- 组件：仪表盘卡片、图表、配置表单、对话框。

### 7.3 安全与通信
- 统一 `httpClient` 自动注入 JWT；401 自动登出与跳转。
- 传输加密：`CryptoService` 使用 RSA 公钥加密敏感信息。
- `api.ts` 统一封装后端接口。

### 7.4 i18n
- `i18n/translations.ts` 内置多语言文案。

---

## 八、部署与运维

- **Docker Compose**：前后端分离服务，前端 Nginx 代理 `/api/`。
- **健康检查**：后端 `/api/health`；前端 `/health`。
- **环境变量**：`.env` 中配置 JWT/加密密钥/端口等。
- **脚本**：`start.sh` 自动生成密钥与启动容器。

---

## 九、测试现状

- 后端测试覆盖 `api/`、`decision/`、`mcp/`、`provider/`、`trader/` 等模块。
- 使用 `testify`、`gomonkey` 等进行 mock。
- 前端仅有基础测试 scaffolding（`src/test/setup.ts`）。

**整体评价：** 后端测试覆盖有一定基础，但未形成端到端交易/回测的自动化回归链路；前端测试较弱。

---

## 十、风险点与改进建议

### 10.1 架构/实现风险

1. **文档与代码存在偏差**：文档中提及 `strategy/` 模块、`go-talib`，但代码中未见实际使用；可能存在过期资料。
2. **硬编码 API Key**：`market/data.go` 中 CoinAnk API Key 内置，存在泄露与滥用风险。
3. **安全策略不一致**：策略配置声明部分风险控制“代码强制”，但实现中未全量覆盖（如 `MaxMarginUsage` 未在交易执行层强制执行）。
4. **风险控制参数不一致**：`decision` 层对风报比阈值使用固定值（3.0），未读取策略配置。
5. **管理员登录接口缺失**：前端调用 `/api/admin-login`，后端未实现路由，存在功能断层。
6. **CORS 全开放**：生产环境可能存在跨站风险；建议限制 Origin。
7. **JWT 黑名单仅内存**：多实例部署或重启后失效；分布式场景不安全。
8. **遥测默认开启**：可能引发合规/隐私问题，应在 UI 中显式提示并允许关闭。
9. **交易所适配不完整**：部分交易所功能在代码中标注 TODO（如 Lighter 的 SetLeverage/SetMarginMode）。

### 10.2 建议优先级（可执行）

**优先级 P0（安全/一致性）**
- 移除硬编码 API Key → 通过环境变量或数据库配置。
- 在 `AutoTrader` 层补齐 `MaxMarginUsage` 风控强制校验。
- 将决策校验中的风报比阈值与 `StrategyConfig` 对齐。

**优先级 P1（可靠性/部署）**
- 实现或移除 `/api/admin-login` 前后端对齐。
- 引入持久化 Token Blacklist（Redis/SQLite 表）。
- 增加 API 速率限制/请求频控。

**优先级 P2（可维护性）**
- 对文档与代码进行同步梳理。
- 回测/实盘差异（数据源、风控规则）增加明确标注。
- 前端补充关键路径测试（交易配置、回测启动、辩论流程）。

---

## 十一、总结

NOFX 的核心价值在于：**多 AI 模型 + 多交易所 + 策略/回测/辩论一体化**。代码结构清晰、模块边界合理，后端包含丰富的交易与回测能力，前端提供完整的可视化配置与运营界面。主要技术风险集中在配置一致性、安全默认值、文档同步与少数未实现的功能点上。

若聚焦于生产可用性，建议优先完善安全与风险控制一致性，并补齐配置与接口缺失。

---

**报告完成**
