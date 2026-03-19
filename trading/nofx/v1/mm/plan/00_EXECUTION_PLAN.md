# NOFX 港A股重构 - Task-by-Task 详细实施计划

## 概述

本文档是 NOFX 项目从加密货币交易系统重构为港股/A股交易系统的详细 Task-by-Task 实施计划。
每个 Task 都被拆解为最小的可执行单元，包含文件路径、函数名、交付物和验收标准。

**项目路径**: `/home/zq/work-space/repo/ai-projs/trade/nofx`
**本文档路径**: `plan/00_EXECUTION_PLAN.md`

---

## Task 依赖关系图

```
Phase 1 (类型定义)
├── T-101 MarketType
├── T-102 AccountBalance ─┐
├── T-103 Position ────────┼── T-601 T1Manager
├── T-104 OrderRequest ────┤
├── T-107 CNPosition ──────┘
└── T-501~505 Schema

Phase 2 (行情服务)
├── T-401 HKMarketDataService
└── T-407 CNMarketDataService

Phase 3 (券商适配器) ← Phase 1, Phase 2
├── T-201 BrokerAdapter
├── T-202~206 FutuAdapter
└── T-301~302 EastMoneyAdapter

Phase 4 (策略引擎) ← Phase 3
├── T-613 engine_hk.go
└── T-619 engine_cn.go

Phase 5 (LangGraph) ← Phase 1, Phase 3
├── T-701~702 State
├── T-703~704 Nodes
├── T-705 Edges
└── T-729~733 Tools

Phase 6 (前端) ← Phase 5
├── T-801~805 Components
└── T-831~832 Stores

Phase 7 (测试) ← Phase 1~6
└── T-901~905 Tests
```

---

## Phase 1: 类型定义与数据层

### T-101: MarketType 枚举

**文件**: `trader/types/market_type.go`

```go
package types

// MarketType 市场类型枚举
type MarketType string

const (
    MarketTypeCrypto MarketType = "crypto"  // 加密货币 (遗留)
    MarketTypeHK    MarketType = "hk"      // 港股
    MarketTypeCN    MarketType = "cn"      // A股
)

// IsValid 检查市场类型是否有效
func (m MarketType) IsValid() bool {
    switch m {
    case MarketTypeCrypto, MarketTypeHK, MarketTypeCN:
        return true
    }
    return false
}
```

**验收标准**:
- [ ] MarketType 枚举定义正确 (crypto/hk/cn)
- [ ] IsValid() 方法正确
- [ ] 单元测试覆盖

---

### T-102: AccountBalance 结构

**文件**: `trader/types/account.go`

```go
package types

import "time"

// AccountBalance 账户余额
type AccountBalance struct {
    MarketType     MarketType `json:"market_type"`      // 市场类型
    TotalEquity    float64   `json:"total_equity"`     // 总权益
    AvailableCash  float64   `json:"available_cash"`    // 可用资金
    FrozenCash     float64   `json:"frozen_cash"`       // 冻结资金
    MarketValue    float64   `json:"market_value"`      // 持仓市值
    TotalAsset     float64   `json:"total_asset"`       // 总资产
    Currency       string    `json:"currency"`          // 币种: HKD/CNY/USD
    UpdatedAt      time.Time `json:"updated_at"`        // 更新时间
}

// AvailableBuyPower 计算可用买入资金
func (b *AccountBalance) AvailableBuyPower() float64 {
    return b.AvailableCash
}
```

**验收标准**:
- [ ] AccountBalance 包含所有字段
- [ ] JSON tag 正确
- [ ] AvailableBuyPower() 计算正确

---

### T-103: Position 结构

**文件**: `trader/types/position.go`

```go
package types

import "time"

// Position 持仓信息 (通用)
type Position struct {
    Symbol           string    `json:"symbol"`            // 股票代码
    Name             string    `json:"name"`              // 股票名称
    Quantity         float64   `json:"quantity"`          // 持股数量
    AvailableQty     float64   `json:"available_qty"`    // 可用数量 (T+1限制)
    FrozenQty        float64   `json:"frozen_qty"`        // 冻结数量 (当日买入)
    AvgCost          float64   `json:"avg_cost"`          // 平均成本
    CurrentPrice     float64   `json:"current_price"`    // 当前价格
    MarketValue      float64   `json:"market_value"`     // 市值
    UnrealizedPnL    float64   `json:"unrealized_pnl"`   // 浮动盈亏
    UnrealizedPnLPct float64   `json:"unrealized_pnl_pct"` // 浮动盈亏百分比
    Side             string    `json:"side"`              // 方向: "long"
    Status           string    `json:"status"`           // 状态: "OPEN", "FROZEN"
    Exchange         string    `json:"exchange"`         // 交易所: "SEHK", "SSE", "SZSE"
    UpdatedAt        time.Time `json:"updated_at"`       // 更新时间
}

// CalculateMarketValue 计算市值
func (p *Position) CalculateMarketValue() float64 {
    return p.Quantity * p.CurrentPrice
}

// CalculateUnrealizedPnL 计算浮动盈亏
func (p *Position) CalculateUnrealizedPnL() float64 {
    return (p.CurrentPrice - p.AvgCost) * p.Quantity
}
```

**验收标准**:
- [ ] Position 包含所有字段
- [ ] T+1 相关字段正确 (AvailableQty, FrozenQty)
- [ ] 计算方法正确

---

### T-104: OrderRequest/OrderResult

**文件**: `trader/types/order.go`

```go
package types

import "time"

// OrderSide 订单方向
type OrderSide string

const (
    OrderSideBuy  OrderSide = "BUY"
    OrderSideSell OrderSide = "SELL"
)

// OrderType 订单类型
type OrderType string

const (
    OrderTypeMarket OrderType = "MARKET"  // 市价单
    OrderTypeLimit OrderType = "LIMIT"   // 限价单
)

// OrderStatus 订单状态
type OrderStatus string

const (
    OrderStatusPending  OrderStatus = "PENDING"   // 待成交
    OrderStatusNew      OrderStatus = "NEW"       // 已下单
    OrderStatusPartial  OrderStatus = "PARTIAL"   // 部分成交
    OrderStatusFilled   OrderStatus = "FILLED"    // 全部成交
    OrderStatusCanceled OrderStatus = "CANCELED"  // 已取消
    OrderStatusRejected OrderStatus = "REJECTED"  // 已拒绝
)

// OrderRequest 订单请求
type OrderRequest struct {
    TraderID    string     `json:"trader_id"`     // Trader ID
    Symbol      string     `json:"symbol"`        // 股票代码
    Side        OrderSide  `json:"side"`          // 买入/卖出
    Quantity    float64    `json:"quantity"`      // 数量
    OrderType   OrderType  `json:"order_type"`    // 订单类型
    LimitPrice  float64    `json:"limit_price"`   // 限价
    MarketType  MarketType `json:"market_type"`  // 市场类型
}

// OrderResult 订单结果
type OrderResult struct {
    OrderID   string      `json:"order_id"`      // 订单ID
    Symbol    string      `json:"symbol"`        // 股票代码
    Side      OrderSide   `json:"side"`          // 方向
    Quantity  float64     `json:"quantity"`      // 委托数量
    FilledQty float64     `json:"filled_qty"`    // 已成交数量
    AvgPrice  float64     `json:"avg_price"`     // 成交均价
    Status    OrderStatus `json:"status"`        // 订单状态
    OrderType OrderType   `json:"order_type"`    // 订单类型
    CreatedAt time.Time   `json:"created_at"`    // 创建时间
    UpdatedAt time.Time   `json:"updated_at"`    // 更新时间
    Message   string      `json:"message"`        // 错误信息
}
```

**验收标准**:
- [ ] 枚举类型完整 (OrderSide/OrderType/OrderStatus)
- [ ] OrderRequest/OrderResult 字段完整
- [ ] JSON tag 正确

---

### T-107: CNPosition/CNTrade/T1Lock

**文件**: `trader/types/cn_position.go`

```go
package types

// CNPosition A股持仓扩展
type CNPosition struct {
    Position
    Exchange     string  `json:"exchange"`     // 交易所: "SSE", "SZSE", "BSE"
    Board        string  `json:"board"`        // 板块: "Main", "STAR", "BJSE"
    FrozenQty    float64 `json:"frozen_qty"`   // 冻结数量 (T+1)
    TradingLimit float64 `json:"trading_limit"` // 涨跌停限制
}

// CNTrade A股成交记录
type CNTrade struct {
    ID               int64   `json:"id"`
    OrderID         int64   `json:"order_id"`
    ExchangeTradeID string  `json:"exchange_trade_id"` // 交易所成交编号
    Symbol          string  `json:"symbol"`
    Side            string  `json:"side"`          // "BUY", "SELL"
    Price           float64 `json:"price"`
    Quantity        float64 `json:"quantity"`
    Amount          float64 `json:"amount"`       // 成交金额
    Commission      float64 `json:"commission"`   // 佣金
    StampDuty       float64 `json:"stamp_duty"`   // 印花税(卖出收取)
    SettlementFee   float64 `json:"settlement_fee"` // 过户费
    TradeDate       int64   `json:"trade_date"`   // 成交日期 (YYYYMMDD)
    SettlementDate  int64   `json:"settlement_date"` // 结算日期 (T+1)
}

// T1Lock T+1锁定记录
type T1Lock struct {
    ID        int64     `json:"id"`
    TraderID  string    `json:"trader_id"`
    Symbol    string    `json:"symbol"`
    Quantity  float64   `json:"quantity"`
    TradeDate int64     `json:"trade_date"`   // 买入日期
    LockedAt  int64     `json:"locked_at"`    // 锁定时间
    UnlockAt  int64     `json:"unlock_at"`    // 解锁时间 (次日0点)
    Status    string    `json:"status"`       // "LOCKED", "UNLOCKED"
}
```

**验收标准**:
- [ ] CNPosition 包含A股特有字段
- [ ] CNTrade 包含所有费用字段 (佣金/印花税/过户费)
- [ ] T1Lock 结构正确

---

### T-501: hk_stock_positions 表

**文件**: `store/hk_position.go`

```go
package store

import "time"

// HKStockPosition 港股持仓表
type HKStockPosition struct {
    ID                  int64     `gorm:"primaryKey;autoIncrement"`
    TraderID            string    `gorm:"index;size:36;not null"`
    ExchangeID          string    `gorm:"size:36;not null"`
    Symbol              string    `gorm:"size:20;not null;index"`
    Name                string    `gorm:"size:100"`
    Quantity            float64   `gorm:"type:decimal(20,4);not null;default:0"`
    AvailableQty        float64   `gorm:"type:decimal(20,4);not null;default:0"`
    FrozenQty           float64   `gorm:"type:decimal(20,4);not null;default:0"`
    AvgCost             float64   `gorm:"type:decimal(20,4);not null;default:0"`
    CurrentPrice        float64   `gorm:"type:decimal(20,4);not null;default:0"`
    ExchangePositionID  string    `gorm:"size:100"`
    Currency            string    `gorm:"size:10;default:'HKD'"`
    LotSize             int       `gorm:"not null;default:100"`
    CreatedAt           time.Time
    UpdatedAt           time.Time
}

func (HKStockPosition) TableName() string {
    return "hk_stock_positions"
}
```

**验收标准**:
- [ ] 表名正确 `hk_stock_positions`
- [ ] 所有字段类型和约束正确
- [ ] `trader_id` 和 `symbol` 索引正确

---

### T-503: cn_stock_positions 表

**文件**: `store/cn_position.go`

```go
package store

import "time"

// CNStockPosition A股持仓表
type CNStockPosition struct {
    ID                int64     `gorm:"primaryKey;autoIncrement"`
    TraderID          string    `gorm:"index;size:36;not null"`
    ExchangeID        string    `gorm:"size:36;not null"`
    Symbol            string    `gorm:"size:20;not null;index"`
    Name              string    `gorm:"size:100"`
    Exchange          string    `gorm:"size:10;not null"`    // SSE/SZSE/BSE
    Quantity          float64   `gorm:"type:decimal(20,4);not null;default:0"`
    AvailableQty      float64   `gorm:"type:decimal(20,4);not null;default:0"` // 可卖
    FrozenQty         float64   `gorm:"type:decimal(20,4);not null;default:0"` // 冻结
    AvgCost           float64   `gorm:"type:decimal(20,4);not null;default:0"`
    CurrentPrice      float64   `gorm:"type:decimal(20,4);not null;default:0"`
    TradingLimit      float64   `gorm:"type:decimal(20,4);not null;default:0.1"` // 涨跌停10%
    AccountID         string    `gorm:"size:36"`
    CreatedAt         time.Time
    UpdatedAt         time.Time
}

func (CNStockPosition) TableName() string {
    return "cn_stock_positions"
}
```

---

### T-505: cn_t1_locks 表

**文件**: `store/cn_t1_lock.go`

```go
package store

import "time"

// T1Lock T+1锁定记录表
type T1Lock struct {
    ID        int64     `gorm:"primaryKey;autoIncrement"`
    TraderID  string    `gorm:"index;size:36;not null"`
    Symbol    string    `gorm:"size:20;not null;index"`
    Quantity  float64   `gorm:"type:decimal(20,4);not null"`
    TradeDate int64     `gorm:"not null;index"`      // 买入日期 YYYYMMDD
    LockedAt  time.Time `gorm:"not null"`
    UnlockAt  time.Time `gorm:"not null"`            // 解锁时间 (次日 00:00:00)
    Status    string    `gorm:"size:20;not null;index"` // LOCKED/UNLOCKED
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (T1Lock) TableName() string {
    return "cn_t1_locks"
}
```

---

## Phase 3: T1Manager 实现

### T-601: T1Manager 结构

**文件**: `kernel/t1_manager.go`

```go
package kernel

import (
    "nofx/store"
    "gorm.io/gorm"
)

// T1Manager T+1交易管理器
type T1Manager struct {
    db            *gorm.DB
    tradeStore    *CNTradeStore
    positionStore *CNPositionStore
    t1LockStore   *T1LockStore
}

// NewT1Manager 创建T1Manager
func NewT1Manager(db *gorm.DB) *T1Manager {
    return &T1Manager{
        db:            db,
        tradeStore:    NewCNTradeStore(db),
        positionStore: NewCNPositionStore(db),
        t1LockStore:   NewT1LockStore(db),
    }
}
```

**依赖**: T-503 (CNPosition), T-504 (CNTrade), T-505 (T1Lock)

---

### T-602: AllowedSellQty 实现

**文件**: `kernel/t1_manager.go`

```go
// AllowedSellQty 计算指定股票允许卖出的数量
// 业务规则: 今日买入不可卖, 昨日及之前买入可卖
func (m *T1Manager) AllowedSellQty(traderID, symbol string) (float64, error) {
    // Step 1: 获取该股票所有持仓
    positions, err := m.positionStore.GetByTraderAndSymbol(traderID, symbol)
    if err != nil {
        return 0, fmt.Errorf("查询持仓失败: %w", err)
    }

    // Step 2: 获取今日买入数量
    todayBuys, err := m.tradeStore.GetTodayBuys(traderID, symbol)
    if err != nil {
        return 0, fmt.Errorf("查询今日买入失败: %w", err)
    }

    // Step 3: 计算总持仓和今日买入
    var totalQty, todayBuyQty float64
    for _, p := range positions {
        totalQty += p.Quantity
    }
    for _, t := range todayBuys {
        todayBuyQty += t.Quantity
    }

    // Step 4: 计算可卖数量 = 总持仓 - 今日买入
    allowedQty := totalQty - todayBuyQty
    if allowedQty < 0 {
        allowedQty = 0
    }

    return allowedQty, nil
}
```

**验收标准**:
- [ ] 正确计算总持仓
- [ ] 正确计算今日买入
- [ ] 可卖 = 总持仓 - 今日买入
- [ ] 边界情况正确处理

---

### T-603: OnTradeExecute 实现

**文件**: `kernel/t1_manager.go`

```go
// OnTradeExecute 处理成交后T+1逻辑
func (m *T1Manager) OnTradeExecute(trade *store.CNTrade) error {
    return m.db.Transaction(func(tx *gorm.DB) error {
        if trade.Side == "BUY" {
            return m.freezeForT1(tx, trade)
        } else {
            return m.unfreezeForT1(tx, trade)
        }
    })
}

// freezeForT1 买入时冻结 (T+1当日不可卖)
func (m *T1Manager) freezeForT1(tx *gorm.DB, trade *store.CNTrade) error {
    // 计算解锁时间 (次日 00:00:00)
    tradeDate := time.Unix(trade.TradeDate, 0)
    unlockAt := time.Date(tradeDate.Year(), tradeDate.Month(), tradeDate.Day()+1,
        0, 0, 0, 0, tradeDate.Location())

    // 创建 T1Lock 记录
    t1Lock := &store.T1Lock{
        TraderID:  trade.TraderID,
        Symbol:    trade.Symbol,
        Quantity:  trade.Quantity,
        TradeDate: trade.TradeDate,
        LockedAt:  time.Now(),
        UnlockAt:  unlockAt,
        Status:    "LOCKED",
    }

    if err := m.t1LockStore.Create(tx, t1Lock); err != nil {
        return fmt.Errorf("创建T1Lock失败: %w", err)
    }

    // 更新持仓表的冻结数量
    if err := m.positionStore.IncrementFrozen(tx, trade.TraderID, trade.Symbol, trade.Quantity); err != nil {
        return fmt.Errorf("更新冻结数量失败: %w", err)
    }

    return nil
}
```

**验收标准**:
- [ ] BUY 时调用 freezeForT1
- [ ] SELL 时调用 unfreezeForT1
- [ ] 使用数据库事务
- [ ] T1Lock 记录正确创建

---

### T-604: OnTradingDayEnd 实现

**文件**: `kernel/t1_manager.go`

```go
// OnTradingDayEnd 日终解冻处理 (每个交易日 15:30 后调用)
func (m *T1Manager) OnTradingDayEnd(traderID string) error {
    now := time.Now()

    // 查找所有已到解锁时间的 T1Lock
    locks, err := m.t1LockStore.GetExpiredLocks(traderID, now)
    if err != nil {
        return fmt.Errorf("查询过期T1Lock失败: %w", err)
    }

    for _, lock := range locks {
        if err := m.unlockExpiredLock(lock); err != nil {
            logger.Errorf("解锁过期T1Lock失败: lock_id=%d, err=%v", lock.ID, err)
            continue
        }
    }

    return nil
}

// unlockExpiredLock 解锁单个过期记录
func (m *T1Manager) unlockExpiredLock(lock *store.T1Lock) error {
    return m.db.Transaction(func(tx *gorm.DB) error {
        // 更新 T1Lock 状态
        if err := m.t1LockStore.MarkUnlocked(tx, lock.ID); err != nil {
            return err
        }

        // 减少持仓冻结数量，增加可用数量
        if err := m.positionStore.DecrementFrozen(tx, lock.TraderID, lock.Symbol, lock.Quantity); err != nil {
            return err
        }
        if err := m.positionStore.IncrementAvailable(tx, lock.TraderID, lock.Symbol, lock.Quantity); err != nil {
            return err
        }

        return nil
    })
}
```

**验收标准**:
- [ ] 正确查找已过期的 T1Lock
- [ ] 解锁后更新持仓的 FrozenQty 和 AvailableQty
- [ ] 使用数据库事务

---

## Phase 3: BrokerAdapter 接口

### T-201: BrokerAdapter 接口定义

**文件**: `adapter/broker.go`

```go
package adapter

import "nofx/trader/types"

// BrokerAdapter 券商适配器接口
type BrokerAdapter interface {
    // GetMarketType 返回市场类型
    GetMarketType() types.MarketType

    // GetName 返回券商名称
    GetName() string

    // GetBalance 获取账户余额
    GetBalance() (*types.AccountBalance, error)

    // GetPositions 获取持仓列表
    GetPositions() ([]*types.Position, error)

    // PlaceOrder 下单
    PlaceOrder(req *types.OrderRequest) (*types.OrderResult, error)

    // CancelOrder 撤单
    CancelOrder(orderID string) error

    // GetOrderStatus 查询订单状态
    GetOrderStatus(orderID string) (*types.OrderResult, error)

    // GetMarketData 获取市场数据
    GetMarketData(symbol string) (*types.MarketData, error)

    // A股特有
    CheckT1Status(symbol string, quantity float64) (float64, error)
}
```

**依赖**: T-102 (AccountBalance), T-103 (Position), T-104 (OrderRequest)

---

## Phase 5: LangGraph 工作流

### T-701: TradingState 定义

**文件**: `telegram/agent/langgraph/graph/state.go`

```go
package graph

import "time"

// IntentType 意图类型
type IntentType string

const (
    IntentQuery    IntentType = "query"     // 查询类
    IntentTrade   IntentType = "trade"    // 交易类
    IntentAnalysis IntentType = "analysis" // 分析类
    IntentConfig  IntentType = "config"   // 配置类
    IntentUnknown IntentType = "unknown"  // 未知
)

// WorkflowStep 工作流步骤
type WorkflowStep string

const (
    StepStart         WorkflowStep = "start"
    StepIntentRouting WorkflowStep = "intent_routing"
    StepQueryProcess  WorkflowStep = "query_process"
    StepTradeAnalyze  WorkflowStep = "trade_analyze"
    StepRiskCheck     WorkflowStep = "risk_check"
    StepHumanConfirm  WorkflowStep = "human_confirm"
    StepOrderExecute  WorkflowStep = "order_execute"
    StepVerification  WorkflowStep = "verification"
    StepFormatResult  WorkflowStep = "format_result"
    StepEnd           WorkflowStep = "end"
    StepError         WorkflowStep = "error"
)

// TradingState LangGraph 核心状态
type TradingState struct {
    ThreadID    string        `json:"thread_id"`
    UserID      string        `json:"user_id"`
    UserMessage string        `json:"user_message"`
    Intent      IntentType    `json:"intent"`
    Messages    []Message     `json:"messages"`
    Account     *AccountContext `json:"account"`
    Positions   []*Position   `json:"positions"`
    CurrentTask *TaskContext  `json:"current_task"`
    WorkflowStep WorkflowStep `json:"workflow_step"`
    RiskResult  *RiskResult   `json:"risk_result"`
    HumanAction *HumanAction  `json:"human_action"`
    Error       *StateError   `json:"error"`
}

// Message 消息结构
type Message struct {
    Role    string    `json:"role"`
    Content string    `json:"content"`
    Time    time.Time `json:"time"`
}

// AccountContext 账户上下文
type AccountContext struct {
    UserEmail     string  `json:"user_email"`
    UserID        string  `json:"user_id"`
    TotalAsset    float64 `json:"total_asset"`
    AvailableCash float64 `json:"available_cash"`
    MarketValue   float64 `json:"market_value"`
}

// TaskContext 任务上下文
type TaskContext struct {
    Intent      IntentType         `json:"intent"`
    Entities    map[string]string  `json:"entities"`
    QueryResult any                `json:"query_result"`
    OrderRequest *OrderRequest     `json:"order_request"`
    CreatedAt   time.Time          `json:"created_at"`
}

// RiskResult 风控结果
type RiskResult struct {
    Passed      bool     `json:"passed"`
    RiskLevel   string   `json:"risk_level"` // LOW/MEDIUM/HIGH/CRITICAL
    Reasons     []string `json:"reasons"`
    Suggestions []string `json:"suggestions"`
}

// HumanAction 人工确认状态
type HumanAction struct {
    ActionType   string    `json:"action_type"`
    Details      string    `json:"details"`
    CreatedAt    time.Time `json:"created_at"`
    ExpiresAt    time.Time `json:"expires_at"`
    Confirmed    *bool     `json:"confirmed"`  // nil=pending, true=confirmed, false=rejected
    ConfirmToken string    `json:"confirm_token"`
}
```

**验收标准**:
- [ ] TradingState 包含所有必要字段
- [ ] IntentType 枚举正确
- [ ] WorkflowStep 枚举正确
- [ ] HumanAction 支持超时

---

### T-703: Nodes 定义

**文件**: `telegram/agent/langgraph/graph/nodes.go`

```go
package graph

import "fmt"

// NodeFunc 节点函数类型
type NodeFunc func(state *TradingState) (*TradingState, error)

// InputNode 输入节点 - 解析用户消息
func InputNode(state *TradingState) (*TradingState, error) {
    state.WorkflowStep = StepIntentRouting
    state.CurrentTask = &TaskContext{
        Entities: make(map[string]string),
        CreatedAt: now(),
    }
    return state, nil
}

// IntentNode 意图识别节点
func IntentNode(state *TradingState) (*TradingState, error) {
    prompt := fmt.Sprintf(`分析用户消息，识别意图类型。

用户消息: %s

可选意图:
- query: 查询账户、持仓、行情、订单等
- trade: 买入、卖出、止损等交易操作
- analysis: 市场分析、策略分析、性能分析
- config: 策略配置、模型配置等
- unknown: 无法识别

返回 JSON: {"intent": "意图类型", "entities": {"symbol": "股票代码", "quantity": "数量"}}`, state.UserMessage)

    response, err := llm.Call(prompt)
    if err != nil {
        return nil, fmt.Errorf("意图识别失败: %w", err)
    }

    var result IntentResult
    if err := json.Unmarshal([]byte(response), &result); err != nil {
        return nil, fmt.Errorf("解析意图失败: %w", err)
    }

    state.Intent = IntentType(result.Intent)
    state.CurrentTask.Intent = state.Intent
    state.CurrentTask.Entities = result.Entities
    state.WorkflowStep = StepQueryProcess

    return state, nil
}

// RiskCheckNode 风控检查节点
func RiskCheckNode(state *TradingState) (*TradingState, error) {
    order := state.CurrentTask.OrderRequest
    account := state.Account

    riskResult := &RiskResult{
        Passed:    true,
        RiskLevel: "LOW",
        Reasons:   []string{},
        Suggestions: []string{},
    }

    // 1. 资金检查
    requiredCash := order.Quantity * order.LimitPrice
    if order.Side == "BUY" && requiredCash > account.AvailableCash {
        riskResult.Passed = false
        riskResult.RiskLevel = "HIGH"
        riskResult.Reasons = append(riskResult.Reasons, fmt.Sprintf("资金不足: 需要 %.2f，可用 %.2f", requiredCash, account.AvailableCash))
    }

    // 2. T+1 检查 (A股)
    if order.MarketType == "cn" && order.Side == "SELL" {
        allowedQty, err := t1Manager.AllowedSellQty(state.UserID, order.Symbol)
        if err != nil {
            riskResult.Passed = false
            riskResult.Reasons = append(riskResult.Reasons, fmt.Sprintf("T+1检查失败: %v", err))
        } else if order.Quantity > allowedQty {
            riskResult.Passed = false
            riskResult.RiskLevel = "MEDIUM"
            riskResult.Reasons = append(riskResult.Reasons, fmt.Sprintf("T+1限制: 允许卖出 %.2f", allowedQty))
            riskResult.Suggestions = append(riskResult.Suggestions, "请等待持仓解冻后再次尝试")
        }
    }

    state.RiskResult = riskResult
    return state, nil
}
```

**验收标准**:
- [ ] InputNode 正确初始化状态
- [ ] IntentNode 正确识别意图
- [ ] RiskCheckNode 包含资金检查和 T+1 检查

---

### T-705: Edges 定义

**文件**: `telegram/agent/langgraph/graph/edges.go`

```go
package graph

// RouteByIntent 按意图路由
func RouteByIntent(state *TradingState) string {
    switch state.Intent {
    case IntentQuery:
        return "query"
    case IntentTrade:
        return "trade"
    case IntentAnalysis:
        return "analysis"
    case IntentConfig:
        return "config"
    default:
        return "unknown"
    }
}

// RouteByRiskCheck 按风控结果路由
func RouteByRiskCheck(state *TradingState) string {
    if !state.RiskResult.Passed {
        return "reject"
    }
    switch state.RiskResult.RiskLevel {
    case "HIGH", "CRITICAL":
        return "human_confirm"
    default:
        return "auto_execute"
    }
}

// RouteByHumanAction 按人工确认结果路由
func RouteByHumanAction(state *TradingState) string {
    if state.HumanAction == nil {
        return "execute"
    }
    switch *state.HumanAction.Confirmed {
    case true:
        return "execute"
    case false:
        return "cancel"
    }
    return "await"
}
```

**验收标准**:
- [ ] RouteByIntent 正确路由到 query/trade/analysis/config
- [ ] RouteByRiskCheck 正确判断 reject/human_confirm/auto_execute
- [ ] RouteByHumanAction 正确处理 await 状态

---

## Phase 6: 前端组件

### T-801: MarketBoard 组件

**文件**: `web/src/components/hk-ashare/MarketBoard/types.ts`

```typescript
// StockQuote 股票行情
export interface StockQuote {
  symbol: string;        // 股票代码
  name: string;          // 股票名称
  currentPrice: number;  // 当前价格
  changePercent: number;  // 涨跌幅百分比
  volume: number;        // 成交量
  turnover: number;      // 成交额
  market: 'hk' | 'cn';  // 市场类型
  updatedAt: string;      // 更新时间
}

// MarketBoardProps
export interface MarketBoardProps {
  title?: string;
  market: 'hk' | 'cn' | 'all';
  onStockClick: (symbol: string, market: 'hk' | 'cn') => void;
  refreshInterval?: number; // 刷新间隔(ms)
}
```

**文件**: `web/src/components/hk-ashare/MarketBoard/index.tsx`

```typescript
import React, { useEffect, useState } from 'react';
import { StockQuote, MarketBoardProps } from './types';
import { hkAshareAPI } from '@/services/api/hkAshare';

export const MarketBoard: React.FC<MarketBoardProps> = ({
  title = '涨跌幅榜',
  market,
  onStockClick,
  refreshInterval = 5000,
}) => {
  const [quotes, setQuotes] = useState<StockQuote[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchQuotes = async () => {
      try {
        const data = await hkAshareAPI.getQuotes(market);
        setQuotes(data);
      } catch (error) {
        console.error('获取行情失败:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchQuotes();
    const interval = setInterval(fetchQuotes, refreshInterval);
    return () => clearInterval(interval);
  }, [market, refreshInterval]);

  if (loading) return <div className="p-4">加载中...</div>;

  return (
    <div className="bg-white rounded-lg shadow p-4">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <div className="space-y-2">
        {quotes.map((quote, index) => (
          <div
            key={quote.symbol}
            className="flex items-center justify-between p-2 hover:bg-gray-50 cursor-pointer"
            onClick={() => onStockClick(quote.symbol, quote.market)}
          >
            <span className="text-gray-500 w-8">{index + 1}</span>
            <span className="flex-1">{quote.name}</span>
            <span className="text-right font-mono">{quote.currentPrice}</span>
            <span className={`text-right w-20 ${quote.changePercent >= 0 ? 'text-red-500' : 'text-green-500'}`}>
              {quote.changePercent >= 0 ? '+' : ''}{quote.changePercent.toFixed(2)}%
            </span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default MarketBoard;
```

**验收标准**:
- [ ] 正确获取和显示行情数据
- [ ] 自动刷新功能
- [ ] 点击股票调用 onStockClick

---

### T-831: Zustand Store

**文件**: `web/src/stores/hkAshareStore.ts`

```typescript
import { create } from 'zustand';
import { Position, OrderRequest } from '@/components/hk-ashare/types';

interface HKAshareState {
  account: {
    totalAsset: number;
    availableCash: number;
    marketValue: number;
  } | null;

  positions: Position[];
  orders: OrderRequest[];
  t1Locks: { symbol: string; quantity: number; unlockAt: string }[];
  wsConnected: boolean;

  fetchAccount: () => Promise<void>;
  fetchPositions: () => Promise<void>;
  placeOrder: (order: OrderRequest) => Promise<void>;
  cancelOrder: (orderId: string) => Promise<void>;
  checkT1Status: (symbol: string) => Promise<number>;
}

export const useHKAshareStore = create<HKAshareState>((set, get) => ({
  account: null,
  positions: [],
  orders: [],
  t1Locks: [],
  wsConnected: false,

  fetchAccount: async () => {
    const account = await hkAshareAPI.getAccount();
    set({ account });
  },

  fetchPositions: async () => {
    const positions = await hkAshareAPI.getPositions();
    set({ positions });
  },

  placeOrder: async (order: OrderRequest) => {
    const result = await hkAshareAPI.placeOrder(order);
    if (result.status === 'NEW' || result.status === 'FILLED') {
      get().fetchPositions();
      get().fetchAccount();
    }
    return result;
  },

  cancelOrder: async (orderId: string) => {
    await hkAshareAPI.cancelOrder(orderId);
    get().fetchPositions();
  },

  checkT1Status: async (symbol: string) => {
    const locks = await hkAshareAPI.getT1Status(symbol);
    const lockedQty = locks.reduce((sum, lock) => sum + lock.quantity, 0);
    set((state) => ({
      t1Locks: [...state.t1Locks.filter((l) => l.symbol !== symbol), ...locks],
    }));
    return lockedQty;
  },
}));
```

**验收标准**:
- [ ] 账户状态管理
- [ ] 持仓状态管理
- [ ] T+1 锁定状态管理
- [ ] placeOrder 后自动刷新

---

## 完整 Task 清单

| Task ID | Phase | 文件路径 | 内容 | 优先级 | 状态 |
|---------|-------|----------|------|--------|------|
| T-101 | 1 | trader/types/market_type.go | MarketType 枚举 | P0 | ⬜ |
| T-102 | 1 | trader/types/account.go | AccountBalance | P0 | ⬜ |
| T-103 | 1 | trader/types/position.go | Position | P0 | ⬜ |
| T-104 | 1 | trader/types/order.go | OrderRequest/Result | P0 | ⬜ |
| T-107 | 1 | trader/types/cn_position.go | CNPosition/CNTrade/T1Lock | P0 | ⬜ |
| T-501 | 1 | store/hk_position.go | hk_stock_positions | P0 | ⬜ |
| T-503 | 1 | store/cn_position.go | cn_stock_positions | P0 | ⬜ |
| T-504 | 1 | store/cn_trade.go | cn_trades | P0 | ⬜ |
| T-505 | 1 | store/cn_t1_lock.go | cn_t1_locks | P0 | ⬜ |
| T-601 | 3 | kernel/t1_manager.go | T1Manager 结构 | P0 | ⬜ |
| T-602 | 3 | kernel/t1_manager.go | AllowedSellQty() | P0 | ⬜ |
| T-603 | 3 | kernel/t1_manager.go | OnTradeExecute() | P0 | ⬜ |
| T-604 | 3 | kernel/t1_manager.go | OnTradingDayEnd() | P0 | ⬜ |
| T-201 | 3 | adapter/broker.go | BrokerAdapter 接口 | P0 | ⬜ |
| T-202 | 3 | adapter/hk/futu/adapter.go | FutuAdapter | P0 | ⬜ |
| T-301 | 3 | adapter/cn/eastmoney/adapter.go | EastMoneyAdapter | P0 | ⬜ |
| T-401 | 2 | market/hk_market.go | HKMarketDataService | P0 | ⬜ |
| T-407 | 2 | market/cn_market.go | CNMarketDataService | P0 | ⬜ |
| T-701 | 5 | .../graph/state.go | TradingState | P0 | ⬜ |
| T-703 | 5 | .../graph/nodes.go | Nodes | P0 | ⬜ |
| T-705 | 5 | .../graph/edges.go | Edges | P0 | ⬜ |
| T-801 | 6 | .../MarketBoard/index.tsx | MarketBoard | P0 | ⬜ |
| T-831 | 6 | stores/hkAshareStore.ts | Zustand Store | P0 | ⬜ |
| ... | ... | ... | ... | ... | ... |

---

## 预估工期

| Phase | 内容 | Task 数 | 工期 | 累计 |
|-------|------|---------|------|------|
| Phase 1 | 类型定义与数据层 | 9 | 4-6 周 | 4-6 周 |
| Phase 2 | 市场数据层 | 6 | 2-3 周 | 6-9 周 |
| Phase 3 | 券商适配器 | 10 | 3-4 周 | 9-13 周 |
| Phase 4 | 策略引擎适配 | 6 | 2-3 周 | 11-16 周 |
| Phase 5 | LangGraph Agent | 10 | 3-4 周 | 14-20 周 |
| Phase 6 | 前端重构 | 8 | 3-4 周 | 17-24 周 |
| Phase 7 | 迁移与测试 | 6 | 2-3 周 | 19-27 周 |
| **总计** | | **55+** | **19-27 周** | |
