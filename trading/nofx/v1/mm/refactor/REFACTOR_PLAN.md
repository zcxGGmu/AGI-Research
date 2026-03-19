# NOFX 项目重构方案

## 项目概述

将 NOFX 加密货币交易系统重构为港股/A股交易系统，并使用 LangGraph 重构 Agent 架构。

| 项目信息 | 说明 |
|---------|------|
| **项目路径** | `/home/zq/work-space/repo/ai-projs/trade/nofx` |
| **重构目标** | 加密货币 → 港股/A股 + LangGraph Agent |
| **预估工期** | 19-27 周 (含测试) |
| **重构范围** | 删除 10 个加密货币模块，新增 5 个券商适配器 |

---

## 目录结构

```
nofx/
├── adapter/                          # [新增] 券商适配器层
│   ├── broker.go                    # BrokerAdapter 接口
│   ├── hk/
│   │   ├── futu/                   # 富途港股适配器
│   │   ├── tiger/                  # 老虎证券港股适配器
│   │   └── longbridge/             # 长桥证券港股适配器
│   └── cn/
│       ├── eastmoney/               # 东方财富A股适配器
│       └── ths/                     # 同花顺A股适配器
├── market/                          # [新增] 市场数据层
│   ├── hk_market.go                # 港股行情服务
│   └── cn_market.go                # A股行情服务
├── trader/types/                    # [修改] 类型定义
│   ├── interface.go                # 统一Trader接口
│   ├── market_type.go              # MarketType枚举
│   ├── account.go                  # AccountBalance
│   ├── position.go                 # Position
│   ├── order.go                    # OrderRequest/Result
│   ├── hk_position.go              # HKPosition/WarrantData
│   └── cn_position.go              # CNPosition/CNTrade/T1Lock
├── store/                           # [修改] 数据存储层
│   ├── hk_position.go              # 港股持仓表
│   ├── hk_order.go                # 港股订单表
│   ├── cn_position.go              # A股持仓表
│   ├── cn_trade.go                # A股成交表
│   └── cn_t1_lock.go              # T+1锁定表
├── kernel/                          # [修改] 策略引擎
│   ├── t1_manager.go              # T+1管理器
│   ├── engine_hk.go               # 港股策略引擎
│   └── engine_cn.go               # A股策略引擎
└── telegram/agent/langgraph/       # [新增] LangGraph Agent
    ├── graph/
    │   ├── state.go               # TradingState
    │   ├── nodes.go               # 节点定义
    │   ├── edges.go               # 边定义
    │   └── workflow.go            # 工作流构建
    └── tools/
        ├── market_tools.go        # 市场工具
        ├── order_tools.go         # 订单工具
        └── account_tools.go        # 账户工具
```

---

## 一、重构范围

### 1.1 删除范围 (10 个加密货币模块)

| 模块路径 | 说明 | 关联依赖 |
|---------|------|----------|
| `trader/binance/` | Binance CEX 适配器 | trader/types, kernel |
| `trader/bybit/` | Bybit CEX 适配器 | trader/types, kernel |
| `trader/okx/` | OKX CEX 适配器 | trader/types, kernel |
| `trader/bitget/` | Bitget CEX 适配器 | trader/types, kernel |
| `trader/gate/` | Gate.io CEX 适配器 | trader/types, kernel |
| `trader/kucoin/` | Kucoin CEX 适配器 | trader/types, kernel |
| `trader/indodax/` | Indodax CEX 适配器 | trader/types, kernel |
| `trader/hyperliquid/` | Hyperliquid DEX | trader/types, kernel |
| `trader/aster/` | Aster DEX | trader/types, kernel |
| `trader/lighter/` | Lighter DEX | trader/types, kernel |

**同步删除的 Provider 模块**:
- `provider/hyperliquid/` - Hyperliquid 行情
- `provider/coinank/` - CoinAnk 聚合数据
- `provider/alpaca/` - Alpaca 数据 (美股)
- `provider/twelvedata/` - TwelveData 外汇

### 1.2 新增范围

| 模块路径 | 说明 | 优先级 |
|---------|------|--------|
| `adapter/hk/futu/` | 富途港股适配器 | P0 |
| `adapter/hk/tiger/` | 老虎证券港股 | P1 |
| `adapter/hk/longbridge/` | 长桥证券港股 | P1 |
| `adapter/cn/eastmoney/` | 东方财富A股 | P0 |
| `adapter/cn/ths/` | 同花顺A股 | P2 |
| `market/hk_market.go` | 港股行情服务 | P0 |
| `market/cn_market.go` | A股行情服务 | P0 |
| `store/hk_position.go` | 港股持仓存储 | P0 |
| `store/cn_position.go` | A股持仓存储 | P0 |
| `kernel/t1_manager.go` | T+1管理器 | P0 |
| `telegram/agent/langgraph/` | LangGraph Agent | P1 |

### 1.3 保留范围 (需适配)

| 模块路径 | 说明 | 适配内容 |
|---------|------|----------|
| `api/` | HTTP API | 新增港A股路由 |
| `kernel/engine.go` | 策略引擎 | 支持港A股选股 |
| `kernel/grid_engine.go` | 网格引擎 | 港A股费率计算 |
| `store/` | 数据持久化 | 新增持仓/订单表 |
| `auth/` | JWT 认证 | 无需修改 |
| `config/` | 配置管理 | 新增券商配置 |
| `web/` | React 前端 | 完全重构 UI |

---

## 二、重构步骤

### Phase 1: 类型定义与数据层

**工期**: 4-6 周
**目标**: 定义统一的 Go 类型和数据库 Schema

#### 1.1 Go 类型定义

| Task ID | 文件 | 类型/函数 | 优先级 |
|---------|------|----------|--------|
| T-101 | `trader/types/market_type.go` | `MarketType` 枚举 | P0 |
| T-102 | `trader/types/account.go` | `AccountBalance` 结构 | P0 |
| T-103 | `trader/types/position.go` | `Position` 结构 | P0 |
| T-104 | `trader/types/order.go` | `OrderRequest/OrderResult` | P0 |
| T-105 | `trader/types/market_data.go` | `MarketData/KLine` | P0 |
| T-106 | `trader/types/hk_position.go` | `HKPosition/WarrantData` | P1 |
| T-107 | `trader/types/cn_position.go` | `CNPosition/CNTrade/T1Lock` | P0 |

#### 1.2 数据库 Schema

| Task ID | 文件 | 表名 | 优先级 |
|---------|------|------|--------|
| T-501 | `store/hk_position.go` | `hk_stock_positions` | P0 |
| T-502 | `store/hk_order.go` | `hk_orders` | P0 |
| T-503 | `store/cn_position.go` | `cn_stock_positions` | P0 |
| T-504 | `store/cn_trade.go` | `cn_trades` | P0 |
| T-505 | `store/cn_t1_lock.go` | `cn_t1_locks` | P0 |

#### 1.3 T1Manager 核心

| Task ID | 文件 | 函数 | 优先级 |
|---------|------|------|--------|
| T-601 | `kernel/t1_manager.go` | `T1Manager` 结构体 | P0 |
| T-602 | `kernel/t1_manager.go` | `AllowedSellQty()` | P0 |
| T-603 | `kernel/t1_manager.go` | `OnTradeExecute()` | P0 |
| T-604 | `kernel/t1_manager.go` | `OnTradingDayEnd()` | P0 |

---

### Phase 2: 市场数据层

**工期**: 2-3 周
**目标**: 实现港股/A股实时行情服务

#### 2.1 港股行情服务

| Task ID | 文件 | 功能 | 优先级 |
|---------|------|------|--------|
| T-401 | `market/hk_market.go` | `HKMarketDataService` | P0 |
| T-402 | `market/hk_market.go` | WebSocket 实时行情 | P0 |
| T-403 | `adapter/hk/futu/market.go` | 富途行情适配 | P0 |

#### 2.2 A股行情服务

| Task ID | 文件 | 功能 | 优先级 |
|---------|------|------|--------|
| T-407 | `market/cn_market.go` | `CNMarketDataService` | P0 |
| T-408 | `market/cn_market.go` | WebSocket 实时行情 | P0 |
| T-409 | `adapter/cn/eastmoney/market.go` | 东方财富行情适配 | P0 |

---

### Phase 3: 券商适配器

**工期**: 3-4 周
**目标**: 实现完整的券商交易接口

#### 3.1 BrokerAdapter 接口

| Task ID | 文件 | 内容 | 优先级 |
|---------|------|------|--------|
| T-201 | `adapter/broker.go` | `BrokerAdapter` 接口定义 | P0 |

#### 3.2 港股适配器

| Task ID | 文件 | 方法 | 优先级 |
|---------|------|------|--------|
| T-202 | `adapter/hk/futu/adapter.go` | `GetBalance()` | P0 |
| T-203 | `adapter/hk/futu/adapter.go` | `GetPositions()` | P0 |
| T-204 | `adapter/hk/futu/adapter.go` | `PlaceOrder()` | P0 |
| T-205 | `adapter/hk/futu/adapter.go` | `CancelOrder()` | P0 |
| T-206 | `adapter/hk/futu/adapter.go` | `GetMarketData()` | P0 |
| T-207 | `adapter/hk/tiger/adapter.go` | 老虎证券适配器 | P1 |
| T-208 | `adapter/hk/longbridge/adapter.go` | 长桥证券适配器 | P1 |

#### 3.3 A股适配器

| Task ID | 文件 | 方法 | 优先级 |
|---------|------|------|--------|
| T-301 | `adapter/cn/eastmoney/adapter.go` | `GetBalance()` | P0 |
| T-302 | `adapter/cn/eastmoney/adapter.go` | `PlaceOrder()` + T+1检查 | P0 |
| T-303 | `adapter/cn/ths/adapter.go` | 同花顺适配器 | P2 |

---

### Phase 4: 策略引擎适配

**工期**: 2-3 周
**目标**: 适配港股/A股交易策略

| Task ID | 文件 | 内容 | 优先级 |
|---------|------|------|--------|
| T-613 | `kernel/engine_hk.go` | 港股选股策略 | P0 |
| T-614 | `kernel/engine_hk.go` | 港股风控逻辑 | P0 |
| T-619 | `kernel/engine_cn.go` | A股选股策略 | P0 |
| T-620 | `kernel/engine_cn.go` | A股T+1风控 | P0 |
| T-621 | `kernel/grid_engine.go` | 港股网格上下文 | P1 |
| T-622 | `kernel/grid_engine.go` | A股网格上下文 | P1 |

---

### Phase 5: LangGraph Agent

**工期**: 3-4 周
**目标**: 重构为状态化 Agent

#### 5.1 状态与图结构

| Task ID | 文件 | 内容 | 优先级 |
|---------|------|------|--------|
| T-701 | `.../graph/state.go` | `TradingState` 定义 | P0 |
| T-702 | `.../graph/state.go` | `IntentType/WorkflowStep` | P0 |
| T-703 | `.../graph/nodes.go` | `InputNode/IntentNode` | P0 |
| T-704 | `.../graph/nodes.go` | `RiskCheckNode/ExecutionNode` | P0 |
| T-705 | `.../graph/edges.go` | 条件路由函数 | P0 |
| T-706 | `.../graph/workflow.go` | `BuildTradingGraph()` | P0 |

#### 5.2 工具层

| Task ID | 文件 | 工具 | 优先级 |
|---------|------|------|--------|
| T-729 | `.../tools/market_tools.go` | `get_market_data` | P0 |
| T-730 | `.../tools/order_tools.go` | `place_order` | P0 |
| T-731 | `.../tools/order_tools.go` | `cancel_order` | P0 |
| T-732 | `.../tools/account_tools.go` | `get_account_info` | P0 |
| T-733 | `.../tools/account_tools.go` | `get_positions` | P0 |

#### 5.3 Human-in-the-loop

| Task ID | 文件 | 功能 | 优先级 |
|---------|------|------|--------|
| T-740 | `.../services/human_confirm.go` | 确认任务管理 | P1 |
| T-741 | `.../services/human_confirm.go` | `shouldAutoConfirm()` | P1 |
| T-742 | `.../services/human_confirm.go` | 超时处理 | P1 |

---

### Phase 6: 前端重构

**工期**: 3-4 周
**目标**: 重构为港股/A股交易界面

#### 6.1 组件结构

| Task ID | 组件路径 | 说明 | 优先级 |
|---------|---------|------|--------|
| T-801 | `.../MarketBoard/index.tsx` | 涨跌幅排行榜 | P0 |
| T-802 | `.../PositionPanel/index.tsx` | 持仓管理面板 | P0 |
| T-803 | `.../TradeDialog/index.tsx` | 交易确认对话框 | P0 |
| T-804 | `.../AccountOverview/index.tsx` | 资金账户概览 | P0 |
| T-805 | `.../RotationChart/index.tsx` | 轮动分析图表 | P1 |

#### 6.2 状态管理

| Task ID | 文件 | Store | 优先级 |
|---------|------|-------|--------|
| T-831 | `stores/hkAshareStore.ts` | 账户/持仓/订单 | P0 |
| T-832 | `stores/marketDataStore.ts` | 实时行情/WebSocket | P0 |

#### 6.3 类型定义

| Task ID | 文件 | 类型 | 优先级 |
|---------|------|------|--------|
| T-820 | `types/hk-ashare.ts` | `StockQuote` | P0 |
| T-821 | `types/hk-ashare.ts` | `Position` | P0 |
| T-822 | `types/hk-ashare.ts` | `OrderRequest` | P0 |

---

### Phase 7: 迁移与测试

**工期**: 2-3 周
**目标**: 确保系统稳定运行

| Task ID | 内容 | 优先级 |
|---------|------|--------|
| T-900 | 数据库迁移脚本 | P0 |
| T-901 | Trader 接口单元测试 | P0 |
| T-902 | T1Manager 单元测试 | P0 |
| T-903 | 券商 API 集成测试 | P0 |
| T-904 | LangGraph 节点测试 | P1 |
| T-905 | E2E Playwright 测试 | P1 |

---

## 三、数据源方案

### 3.1 港股数据源

| 数据源 | 用途 | 延迟 | 成本 | 推荐度 |
|--------|------|------|------|--------|
| 富途 API | 实时行情 + 交易 | ~100ms | 免费 | ⭐⭐⭐ |
| 老虎 API | 实时行情 | ~200ms | 免费 | ⭐⭐ |
| 长桥 API | 实时行情 + 交易 | ~100ms | 免费 | ⭐⭐⭐ |
| HKEX Orion | 备份行情 | ~500ms | 付费 | ⭐ |

**推荐组合**: 富途(主) + 长桥(备)

### 3.2 A股数据源

| 数据源 | 用途 | 延迟 | 成本 | 推荐度 |
|--------|------|------|------|--------|
| 东方财富 | 实时行情 | ~500ms | 免费 | ⭐⭐⭐ |
| 同花顺 iFinD | Level2 行情 | ~200ms | 付费 | ⭐⭐ |
| Tushare Pro | 历史数据 | - | 免费/付费 | ⭐⭐ |

**推荐组合**: 东方财富(主) + Tushare(历史)

---

## 四、关键设计决策 (ADR)

### ADR-001: 统一 Trader 接口

```go
type BrokerAdapter interface {
    GetMarketType() MarketType
    GetBalance() (*AccountBalance, error)
    GetPositions() ([]*Position, error)
    PlaceOrder(req *OrderRequest) (*OrderResult, error)
    CancelOrder(orderID string) error
    GetOrderStatus(orderID string) (*OrderResult, error)
    GetMarketData(symbol string) (*MarketData, error)
}
```

**决策**: 使用 `MarketType` 区分市场，统一 `BrokerAdapter` 接口
**理由**: 策略层代码统一，便于市场间策略迁移，适配器模式便于扩展新券商

### ADR-002: T+1 在应用层处理

```go
type T1Manager struct {
    db            *gorm.DB
    tradeStore    *CNTradeStore
    positionStore *CNPositionStore
    t1LockStore   *T1LockStore
}

func (m *T1Manager) AllowedSellQty(traderID, symbol string) (float64, error)
func (m *T1Manager) OnTradeExecute(trade *CNTrade) error
func (m *T1Manager) OnTradingDayEnd(traderID string) error
```

**决策**: `T1Manager` 在应用层追踪 T+1 限制
**理由**: 不依赖券商 API，跨券商统一管理，日终自动解冻

### ADR-003: LangGraph 状态化 Agent

```go
type TradingState struct {
    ThreadID    string
    UserID      string
    Intent      IntentType
    Account     *AccountContext
    Positions   []*Position
    CurrentTask *TaskContext
    RiskResult  *RiskResult
    HumanAction *HumanAction
}
```

**决策**: 迁移到 LangGraph StateGraph 架构
**理由**: 显式状态管理、Human-in-the-loop、细粒度工具支持

### ADR-004: 券商 API 为主行情源

**决策**: 港股以券商 API 为主，HKEX 为备份
**理由**: 低延迟(~100ms)、免费、含涡轮数据

---

## 五、里程碑定义

| 里程碑 | 完成条件 | 预估时间 |
|--------|----------|----------|
| **M1: 类型与数据层** | 所有 Go 类型定义完成，Schema 迁移完成 | Week 6 |
| **M2: 行情服务** | 港股/A股实时行情可用 | Week 9 |
| **M3: 交易通道** | 至少一个券商可完整交易 | Week 13 |
| **M4: LangGraph Agent** | Agent 可处理查询和交易 | Week 17 |
| **M5: 前端可用** | 港股/A股交易界面可用 | Week 21 |
| **M6: 测试完成** | 单元测试 >80%，E2E 通过 | Week 24 |

---

## 六、风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| 券商 API 不稳定 | 高 | 高 | 多券商备份、限流控制 |
| T+1 制度复杂性 | 中 | 中 | 独立 T1Manager、充分测试 |
| A股 Level2 数据成本 | 低 | 低 | 基础数据起步 |
| LangGraph 学习曲线 | 中 | 中 | 逐步迁移、保留旧实现 |
| 实时行情延迟 | 中 | 中 | 多源聚合、缓存优化 |
| 人员投入不足 | 高 | 中 | 敏捷迭代、优先级排序 |

---

## 七、预估工期汇总

| Phase | 内容 | Task 数 | 工期 | 累计 |
|-------|------|---------|------|------|
| Phase 1 | 类型定义与数据层 | 11 | 4-6 周 | 4-6 周 |
| Phase 2 | 市场数据层 | 6 | 2-3 周 | 6-9 周 |
| Phase 3 | 券商适配器 | 10 | 3-4 周 | 9-13 周 |
| Phase 4 | 策略引擎适配 | 6 | 2-3 周 | 11-16 周 |
| Phase 5 | LangGraph Agent | 10 | 3-4 周 | 14-20 周 |
| Phase 6 | 前端重构 | 8 | 3-4 周 | 17-24 周 |
| Phase 7 | 迁移与测试 | 6 | 2-3 周 | 19-27 周 |
| **总计** | | **57** | **19-27 周** | |

---

## 八、文档索引

| 文档 | 路径 | 说明 |
|------|------|------|
| 主方案 | `refactor/REFACTOR_PLAN.md` | 本文件 |
| 架构设计 | `refactor/ARCHITECTURE.md` | 详细系统架构 |
| 数据源 | `refactor/DATA_SOURCES.md` | 港A股数据源调研 |
| 前端方案 | `refactor/FRONTEND.md` | 前端重构设计 |
| LangGraph | `refactor/LANGGRAPH.md` | Agent 重构设计 |
| 执行计划 | `plan/00_EXECUTION_PLAN.md` | Task-by-Task 详细计划 |
