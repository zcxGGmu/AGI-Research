# NoFX 项目改造方案：从加密货币到 A 股/港股交易系统

## 目录

1. [项目背景与目标](#1-项目背景与目标)
2. [核心差异分析](#2-核心差异分析)
3. [架构改造方案](#3-架构改造方案)
4. [详细实施计划](#4-详细实施计划)
5. [技术选型建议](#5-技术选型建议)
6. [风险与挑战](#6-风险与挑战)
7. [附录](#7-附录)

---

## 1. 项目背景与目标

### 1.1 当前 NoFX 架构概述

NoFX 是一个基于 AI 驱动的加密货币自动交易系统，核心组件包括：

```
┌─────────────────────────────────────────────────────────────┐
│                      NoFX 当前架构                            │
├─────────────────────────────────────────────────────────────┤
│  前端: React + TypeScript + Tailwind CSS                     │
│  后端: Go + Gin + SQLite                                     │
│  AI: 多模型支持 (DeepSeek, Qwen, OpenAI, Claude, etc.)       │
├─────────────────────────────────────────────────────────────┤
│  核心模块:                                                    │
│  - market/:      加密货币市场数据 (Binance API)              │
│  - provider/:   交易所提供商 (仅 Binance 期货)               │
│  - trader/:      自动交易执行器                               │
│  - backtest/:   历史回测引擎                                 │
│  - decision/:    AI 决策引擎                                 │
│  - debate/:      AI 多智能体辩论系统                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 改造目标

| 目标 | 描述 | 优先级 |
|------|------|--------|
| **多市场支持** | 支持 A 股和港股交易 | P0 |
| **券商接入** | 集成主流券商交易接口 | P0 |
| **行情数据** | 接入股票市场实时/历史数据 | P0 |
| **交易规则适配** | 适配股票市场交易规则 | P0 |
| **T+1 交易** | 支持 A 股 T+1 交割制度 | P1 |
| **融资融券** | 支持融资融券业务 | P1 |
| **港股通** | 支持港股通机制 | P2 |

---

## 2. 核心差异分析

### 2.1 交易机制对比

| 特性 | 加密货币 (Binance 期货) | A 股 | 港股 |
|------|------------------------|------|------|
| **交易时间** | 7×24 小时 | 周一至周五 9:30-15:00 | 周一至周五 9:30-16:00 |
| **交割制度** | T+0 (即时) | T+1 (次日交割) | T+0 (即时) |
| **交易方向** | 双向 (多/空) | 单向 (仅做多) | 双向 (多/空+衍生品) |
| **杠杆** | 1x - 125x | 融资融券 (约 1x-2x) | 融资融券/衍生品 |
| **最小单位** | 精确到小数点 | 100 股为 1 手 | 1 股或 100 股/手 |
| **涨跌幅限制** | 通常无限制 | 主板 10%, 创业板/科创板 20% | 无限制 (个别有波动调节) |
| **手续费** | 挂单 0.02%, 市单 0.04% | 佣金 (约 0.01%-0.03%) + 印花税 0.1% | 佣金 (约 0.01%-0.025%) |
| **强平机制** | 有 (保证金率) | 无 (需手动补充保证金) | 有 (融资融券) |

### 2.2 数据模型差异

#### 2.2.1 交易对 vs 股票代码

| 类型 | 格式 | 示例 |
|------|------|------|
| **加密货币** | BASEUSDT | BTCUSDT, ETHUSDT |
| **A 股** | 市场+代码 | 600519.SH (贵州茅台), 000001.SZ (平安银行) |
| **港股** | 代码.HK | 0700.HK (腾讯), 9988.HK (阿里巴巴) |

#### 2.2.2 持仓模型差异

```go
// 当前加密货币持仓模型
type Position struct {
    Symbol           string  // "BTCUSDT"
    Side             string  // "long" or "short"
    Quantity         float64 // 0.00123456 (任意精度)
    EntryPrice       float64
    Leverage         int     // 3x, 5x, 10x
    LiquidationPrice float64 // 有强平价
    Margin           float64 // 保证金
}

// 改造后股票持仓模型
type StockPosition struct {
    Symbol           string  // "600519.SH"
    Side             string  // "long" (A 股仅做多)
    Quantity         int     // 100, 200, 300 (必须是 100 的整数倍)
    AvailableQty     int     // 可卖数量 (T+1 限制)
    EntryPrice       float64
    Cost             float64 // 含手续费
    MarketValue      float64 // 市值
    UnrealizedPnL    float64
    CanSellToday     bool    // A 股今日买入不可卖
}
```

### 2.3 API 接口差异

#### 2.3.1 当前 Binance 期货 API

```go
// Binance 期货特有的概念
- Open Interest (持仓量)
- Funding Rate (资金费率)
- Mark Price (标记价格)
- Liquidation Price (强平价格)
- Index Price (指数价格)
```

#### 2.3.2 股票 API 特有概念

```go
// A 股/港股特有概念
- 限价单 / 市价单
- 买卖盘口 (五档行情)
- 成交量 / 成交额
- 换手率
- 市盈率 (PE), 市净率 (PB)
- 分红送配
- 股东人数
- 资金流向
- 北向资金 / 南向资金
```

---

## 3. 架构改造方案

### 3.1 整体架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端 (React + TS)                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ 仪表盘   │ │ 策略配置 │ │ 回测界面 │ │ AI辩论   │ │ 交易监控 │  │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘ └─────┬────┘ └─────┬────┘  │
└────────┼────────────┼────────────┼────────────┼────────────┼─────────┘
         │            │            │            │            │
         ▼            ▼            ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        REST API + WebSocket                          │
│                     (api/server.go + handlers)                       │
└─────────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Manager    │    │   Backtest   │    │    Debate    │
│  交易管理器   │    │   回测引擎    │    │  AI辩论引擎   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Trader     │    │   Decision   │    │     MCP      │
│  交易执行器   │    │  决策引擎     │    │  AI模型客户端 │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └─────────┬─────────┴─────────┬─────────┘
                 ▼                   ▼
        ┌────────────────┐  ┌────────────────┐
        │   新增 Provider │  │  改造 Market   │
        │  ┌───────────┐  │  │ ┌────────────┐│
        │  │ A股券商   │  │  │ │ A股行情源  ││
        │  │ 港股券商   │  │  │ │ 港股行情源  ││
        │  └───────────┘  │  │ └────────────┘│
        └────────────────┘  └────────────────┘
                 │                   │
                 ▼                   ▼
        ┌─────────────────────────────────┐
        │          Store (SQLite)         │
        │  - 股票代码列表                  │
        │  - 交易记录 (适配 T+1)           │
        │  - 持仓信息 (手数管理)           │
        │  - 回测数据                     │
        └─────────────────────────────────┘
```

### 3.2 模块改造清单

| 模块 | 改造类型 | 工作量 | 说明 |
|------|---------|--------|------|
| `market/` | 重度改造 | 大 | 替换数据源、调整时间周期、增加股票指标 |
| `provider/` | 新增 | 大 | 新建 A 股/港股券商适配层 |
| `trader/` | 重度改造 | 中 | 适配 T+1、手数管理、涨跌停限制 |
| `backtest/` | 中度改造 | 中 | 适配股票交易规则、复权处理 |
| `decision/` | 轻度改造 | 小 | 调整提示词、移除期货特有概念 |
| `store/` | 中度改造 | 中 | 数据表结构调整 |
| `api/` | 轻度改造 | 小 | 接口兼容、新增字段 |
| `web/` | 轻度改造 | 小 | UI 展示调整 |

---

## 4. 详细实施计划

### 4.1 Phase 1: 基础架构改造 (2-3 周)

#### 4.1.1 创建 Provider 抽象层

```go
// provider/broker/interface.go
package broker

import "time"

// MarketType 市场类型
type MarketType string

const (
    MarketCN    MarketType = "CN"    // A 股
    MarketHK    MarketType = "HK"    // 港股
    MarketUS    MarketType = "US"    // 美股 (预留)
)

// OrderType 订单类型
type OrderType string

const (
    OrderLimit   OrderType = "limit"   // 限价单
    OrderMarket  OrderType = "market"  // 市价单
    OrderStop    OrderType = "stop"    // 止损单
)

// OrderSide 订单方向
type OrderSide string

const (
    OrderBuy  OrderSide = "buy"  // 买入
    OrderSell OrderSide = "sell" // 卖出
)

// OrderStatus 订单状态
type OrderStatus string

const (
    StatusPending   OrderStatus = "pending"    // 待报
    StatusSubmitted OrderStatus = "submitted"  // 已报
    StatusPartial   OrderStatus = "partial"    // 部成
    StatusFilled    OrderStatus = "filled"     // 已成
    StatusCanceled  OrderStatus = "canceled"   // 已撤
    StatusRejected  OrderStatus = "rejected"   // 废单
)

// Order 订单
type Order struct {
    OrderID       string      `json:"order_id"`
    Symbol        string      `json:"symbol"`        // "600519.SH"
    Side          OrderSide   `json:"side"`
    Type          OrderType   `json:"type"`
    Quantity      int         `json:"quantity"`      // 股数 (A 股需是 100 的倍数)
    Price         float64     `json:"price"`         // 委托价格
    FilledQty     int         `json:"filled_qty"`    // 成交数量
    AvgPrice      float64     `json:"avg_price"`     // 成交均价
    Status        OrderStatus `json:"status"`
    CreateTime    time.Time   `json:"create_time"`
    UpdateTime    time.Time   `json:"update_time"`
    Message       string      `json:"message"`       // 错误信息
}

// Position 持仓
type Position struct {
    Symbol        string    `json:"symbol"`         // "600519.SH"
    Quantity      int       `json:"quantity"`       // 持仓数量
    AvailableQty  int       `json:"available_qty"`  // 可卖数量
    CostPrice     float64   `json:"cost_price"`     // 成本价
    MarketPrice   float64   `json:"market_price"`   // 市场价
    MarketValue   float64   `json:"market_value"`   // 市值
    UnrealizedPnL float64   `json:"unrealized_pnl"` // 浮动盈亏
    CanSellToday  bool      `json:"can_sell_today"` // 今日可卖
}

// Account 账户信息
type Account struct {
    TotalAssets    float64 `json:"total_assets"`    // 总资产
    Cash           float64 `json:"cash"`            // 现金
    MarketValue    float64 `json:"market_value"`    // 证券市值
    BuyingPower    float64 `json:"buying_power"`    // 可用资金
    TotalPnL       float64 `json:"total_pnl"`       // 总盈亏
    FrozenCash     float64 `json:"frozen_cash"`     // 冻结资金
}

// Broker 券商接口
type Broker interface {
    // 基础信息
    Name() string
    MarketType() MarketType

    // 账户
    GetAccount() (*Account, error)
    GetPositions() ([]Position, error)
    GetOpenOrders(symbol string) ([]Order, error)
    GetOrderHistory(symbol string, limit int) ([]Order, error)

    // 交易
    PlaceOrder(order *Order) (*Order, error)
    CancelOrder(orderID string) error

    // 实时数据 (WebSocket)
    SubscribeQuotes(symbols []string, handler func(*Quote))
    SubscribeOrders(handler func(*OrderEvent))

    // 关闭连接
    Close()
}

// Quote 实时行情
type Quote struct {
    Symbol       string    `json:"symbol"`        // "600519.SH"
    Name         string    `json:"name"`          // "贵州茅台"
    Price        float64   `json:"price"`         // 最新价
    Open         float64   `json:"open"`          // 今开
    High         float64   `json:"high"`          // 最高
    Low          float64   `json:"low"`           // 最低
    PreClose     float64   `json:"pre_close"`     // 昨收
    Volume       int64     `json:"volume"`        // 成交量
    Amount       float64   `json:"amount"`        // 成交额
    BidPrice     [5]float64 `json:"bid_price"`    // 买一到买五
    BidVolume    [5]int64   `json:"bid_volume"`   // 买一到买五量
    AskPrice     [5]float64 `json:"ask_price"`    // 卖一到卖五
    AskVolume    [5]int64   `json:"ask_volume"`   // 卖一到卖五量
    Timestamp    int64     `json:"timestamp"`     // 时间戳
}

// OrderEvent 订单事件
type OrderEvent struct {
    Order  *Order  `json:"order"`
    Reason string  `json:"reason"` // 触发原因
}

// SecurityInfo 证券信息
type SecurityInfo struct {
    Symbol       string  `json:"symbol"`       // "600519.SH"
    Name         string  `json:"name"`         // "贵州茅台"
    Market       string  `json:"market"`       // "SH", "SZ", "HK"
    ListDate     string  `json:"list_date"`    // 上市日期
    TotalShares  int64   `json:"total_shares"` // 总股本
    FloatShares  int64   `json:"float_shares"` // 流通股本
    Industry     string  `json:"industry"`     // 行业
    Sector       string  `json:"sector"`       // 板块
    IsActive     bool    `json:"is_active"`    // 是否活跃
}
```

#### 4.1.2 A 股券商适配器示例

```go
// provider/broker/cn/simulated.go
package cn

import (
    "sync"
    "time"
)

// SimulatedCNBroker 模拟 A 股券商 (用于测试)
type SimulatedCNBroker struct {
    mu        sync.RWMutex
    account   *Account
    positions map[string]*Position
    orders    map[string]*Order
    orderSeq  int64
}

func NewSimulatedCNBroker(initialCash float64) *SimulatedCNBroker {
    return &SimulatedCNBroker{
        account: &Account{
            Cash:        initialCash,
            BuyingPower: initialCash,
        },
        positions: make(map[string]*Position),
        orders:    make(map[string]*Order),
    }
}

func (b *SimulatedCNBroker) Name() string { return "Simulated A-Share" }
func (b *SimulatedCNBroker) MarketType() MarketType { return MarketCN }

func (b *SimulatedCNBroker) PlaceOrder(order *Order) (*Order, error) {
    b.mu.Lock()
    defer b.mu.Unlock()

    // A 股交易规则验证
    if err := b.validateOrder(order); err != nil {
        return nil, err
    }

    // 生成订单 ID
    b.orderSeq++
    order.OrderID = fmt.Sprintf("CN%d", b.orderSeq)
    order.CreateTime = time.Now()
    order.Status = StatusSubmitted

    b.orders[order.OrderID] = order

    // 模拟成交
    go b.simulateFill(order)

    return order, nil
}

// validateOrder 验证 A 股交易规则
func (b *SimulatedCNBroker) validateOrder(order *Order) error {
    // 1. 数量必须是 100 的整数倍
    if order.Quantity%100 != 0 {
        return fmt.Errorf("A 股买入数量必须是 100 的整数倍")
    }

    // 2. 检查资金是否充足
    if order.Side == OrderBuy {
        required := float64(order.Quantity) * order.Price
        if required > b.account.BuyingPower {
            return fmt.Errorf("资金不足: 需要 %.2f, 可用 %.2f",
                required, b.account.BuyingPower)
        }
    }

    // 3. 检查持仓是否充足 (卖出)
    if order.Side == OrderSell {
        pos, ok := b.positions[order.Symbol]
        if !ok || pos.Quantity < order.Quantity {
            return fmt.Errorf("持仓不足")
        }

        // 4. T+1 检查
        if !pos.CanSellToday {
            return fmt.Errorf("A 股 T+1: 今日买入不可卖")
        }
    }

    return nil
}

// GetPositions 获取持仓 (实现 T+1 逻辑)
func (b *SimulatedCNBroker) GetPositions() ([]Position, error) {
    b.mu.RLock()
    defer b.mu.RUnlock()

    positions := make([]Position, 0, len(b.positions))
    for _, pos := range b.positions {
        // 更新市值
        price, _ := b.getMarketPrice(pos.Symbol)
        pos.MarketPrice = price
        pos.MarketValue = float64(pos.Quantity) * price
        pos.UnrealizedPnL = (price - pos.CostPrice) * float64(pos.Quantity)

        positions = append(positions, *pos)
    }
    return positions, nil
}
```

#### 4.1.3 港股券商适配器示例

```go
// provider/broker/hk/simulated.go
package hk

import (
    "fmt"
    "sync"
    "time"
)

// SimulatedHKBroker 模拟港股券商
type SimulatedHKBroker struct {
    mu        sync.RWMutex
    account   *Account
    positions map[string]*Position
    orders    map[string]*Order
    orderSeq  int64
}

func NewSimulatedHKBroker(initialCash float64) *SimulatedHKBroker {
    return &SimulatedHKBroker{
        account: &Account{
            Cash:        initialCash,
            BuyingPower: initialCash,
        },
        positions: make(map[string]*Position),
        orders:    make(map[string]*Order),
    }
}

func (b *SimulatedHKBroker) Name() string { return "Simulated HK" }
func (b *SimulatedHKBroker) MarketType() MarketType { return MarketHK }

func (b *SimulatedHKBroker) PlaceOrder(order *Order) (*Order, error) {
    b.mu.Lock()
    defer b.mu.Unlock()

    // 港股交易规则
    if err := b.validateOrder(order); err != nil {
        return nil, err
    }

    b.orderSeq++
    order.OrderID = fmt.Sprintf("HK%d", b.orderSeq)
    order.CreateTime = time.Now()
    order.Status = StatusSubmitted

    b.orders[order.OrderID] = order

    // 港股 T+0, 立即成交
    go b.simulateFill(order)

    return order, nil
}

// validateOrder 验证港股交易规则
func (b *SimulatedHKBroker) validateOrder(order *Order) error {
    // 港股可以买入任意股数 (非 100 的倍数)
    // 检查资金是否充足
    if order.Side == OrderBuy {
        required := float64(order.Quantity) * order.Price
        if required > b.account.BuyingPower {
            return fmt.Errorf("资金不足")
        }
    }

    // 港股 T+0, 检查持仓即可
    if order.Side == OrderSell {
        pos, ok := b.positions[order.Symbol]
        if !ok || pos.Quantity < order.Quantity {
            return fmt.Errorf("持仓不足")
        }
    }

    return nil
}
```

### 4.2 Phase 2: 市场数据层改造 (2-3 周)

#### 4.2.1 行情数据源抽象

```go
// market/quote/provider.go
package quote

import "time"

// QuoteProvider 行情数据源接口
type QuoteProvider interface {
    // 实时行情
    GetQuote(symbol string) (*Quote, error)
    GetQuotes(symbols []string) (map[string]*Quote, error)

    // K 线数据
    GetKlines(symbol string, period KlinePeriod, limit int) ([]*Kline, error)

    // 订阅实时行情
    SubscribeQuotes(symbols []string) (<-chan *Quote, error)

    // 证券列表
    GetSecurities(market string) ([]*Security, error)
}

// KlinePeriod K 线周期
type KlinePeriod string

const (
    Period1Min  KlinePeriod = "1m"
    Period5Min  KlinePeriod = "5m"
    Period15Min KlinePeriod = "15m"
    Period30Min KlinePeriod = "30m"
    Period1Hour KlinePeriod = "1h"
    Period1Day  KlinePeriod = "1d"
    Period1Week KlinePeriod = "1w"
    Period1Mon  KlinePeriod = "1M"
)

// Quote 实时行情
type Quote struct {
    Symbol      string    `json:"symbol"`
    Name        string    `json:"name"`
    Price       float64   `json:"price"`
    Change      float64   `json:"change"`
    ChangePct   float64   `json:"change_pct"`
    Volume      int64     `json:"volume"`
    Amount      float64   `json:"amount"`
    BidPrice    float64   `json:"bid_price"`
    AskPrice    float64   `json:"ask_price"`
    Open        float64   `json:"open"`
    High        float64   `json:"high"`
    Low         float64   `json:"low"`
    PreClose    float64   `json:"pre_close"`
    LimitUp     float64   `json:"limit_up"`   // 涨停价
    LimitDown   float64   `json:"limit_down"` // 跌停价
    Timestamp   int64     `json:"timestamp"`
}

// Kline K 线数据
type Kline struct {
    Symbol    string    `json:"symbol"`
    Timestamp int64     `json:"timestamp"`
    Open      float64   `json:"open"`
    High      float64   `json:"high"`
    Low       float64   `json:"low"`
    Close     float64   `json:"close"`
    Volume    int64     `json:"volume"`
    Amount    float64   `json:"amount"`
}

// Security 证券信息
type Security struct {
    Symbol      string  `json:"symbol"`      // "600519.SH"
    Name        string  `json:"name"`        // "贵州茅台"
    Market      string  `json:"market"`      // "SH", "SZ", "HK"
    Type        string  `json:"type"`        // "stock", "index", "etf"
    ListDate    string  `json:"list_date"`   // 上市日期
    DelistDate  string  `json:"delist_date"` // 退市日期
    IsActive    bool    `json:"is_active"`
}

// A股行情提供商实现
type CNQuoteProvider struct {
    dataSource string // "sina", "tencent", "eastmoney"
}

func (p *CNQuoteProvider) GetQuote(symbol string) (*Quote, error) {
    // 从新浪财经获取实时行情
    // http://hq.sinajs.cn/list=sh600519
    url := fmt.Sprintf("http://hq.sinajs.cn/list=%s",
        strings.ToLower(strings.Replace(symbol, ".", "", 1)))

    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    // 解析响应: var hq_str_sh600519="贵州茅台,1689.00,1680.00,..."
    // ... 解析逻辑

    return &Quote{
        Symbol:  symbol,
        Name:    "贵州茅台",
        Price:   1689.00,
        Change:  9.00,
        ChangePct: 0.54,
        // ...
    }, nil
}

// 港股行情提供商实现
type HKQuoteProvider struct {
    dataSource string // "sina", "eastmoney"
}

func (p *HKQuoteProvider) GetQuote(symbol string) (*Quote, error) {
    // 港股行情获取
    // http://hq.sinajs.cn/list=hk0700
    url := fmt.Sprintf("http://hq.sinajs.cn/list=hk%s",
        strings.TrimPrefix(symbol, "0"))

    // ... 实现逻辑

    return &Quote{
        Symbol:  symbol,
        Name:    "腾讯控股",
        Price:   320.50,
        // ...
    }, nil
}
```

#### 4.2.2 股票特有指标

```go
// market/indicators/stock_indicators.go
package indicators

import "github.com/mmcdole/gofeed"

// StockIndicators 股票特有指标
type StockIndicators struct {
    // 估值指标
    PE      float64 // 市盈率
    PEB     float64 // 市盈率(动)
    PEMR    float64 // 市盈率(TTM)
    PB      float64 // 市净率
    PS      float64 // 市销率
    PCF     float64 // 市现率

    // 盈利能力
    ROE     float64 // 净资产收益率
    ROA     float64 // 总资产报酬率
    GrossMargin float64 // 毛利率
    NetMargin  float64 // 净利率

    // 成长能力
    RevenueGrowth  float64 // 营收增长率
    ProfitGrowth   float64 // 利润增长率
    EPSGrowth      float64 // 每股收益增长率

    // 股息指标
    DividendYield float64 // 股息率
    DividendRatio float64 // 分红比例

    // 技术指标
    TurnoverRate   float64 // 换手率
    Amplitude      float64 // 振幅
    MainInflow     float64 // 主力净流入
    NorthInflow    float64 // 北向资金净流入

    // 市场情绪
    Peil          float64 // 市盈率位置 (历史百分位)
    PbPosition    float64 // 市净率位置
}

// FundamentalData 基本面数据
type FundamentalData struct {
    Symbol          string  `json:"symbol"`
    ReportDate      string  `json:"report_date"`      // 报告期
    TotalRevenue    float64 `json:"total_revenue"`    // 营业总收入
    NetProfit       float64 `json:"net_profit"`       // 净利润
    EPS             float64 `json:"eps"`              // 每股收益
    BVPS            float64 `json:"bvps"`             // 每股净资产
    TotalAssets     float64 `json:"total_assets"`     // 总资产
    TotalLiability  float64 `json:"total_liability"`  // 总负债
    TotalEquity     float64 `json:"total_equity"`     // 股东权益
}
```

### 4.3 Phase 3: 交易执行器改造 (2 周)

```go
// trader/stock_trader.go
package trader

import (
    "nofx/provider/broker"
    "time"
)

// StockTrader 股票交易器
type StockTrader struct {
    cfg       *Config
    broker    broker.Broker
    decision  *DecisionEngine

    mu             sync.RWMutex
    status         Status
    lastDecisionAt time.Time

    // 控制通道
    stopCh   chan struct{}
    doneCh   chan struct{}
}

type Config struct {
    Market     broker.MarketType `json:"market"`
    Symbols    []string          `json:"symbols"`
    Strategy   string            `json:"strategy"`
    MaxPos     int               `json:"max_positions"` // 最大持仓数
    MaxPosPct  float64           `json:"max_position_pct"` // 单只股票最大仓位
    StopLoss   float64           `json:"stop_loss"`     // 止损比例
    TakeProfit float64           `json:"take_profit"`   // 止盈比例
}

// Run 运行交易器
func (t *StockTrader) Run(ctx context.Context) error {
    t.mu.Lock()
    t.status = StatusRunning
    t.mu.Unlock()

    // 根据市场类型设置定时器
    var ticker *time.Ticker
    if t.cfg.Market == broker.MarketCN {
        // A 股: 交易时间内每 5 分钟决策一次
        ticker = time.NewTicker(5 * time.Minute)
    } else {
        // 港股: 每 5 分钟决策一次
        ticker = time.NewTicker(5 * time.Minute)
    }
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            t.handleStop()
            return nil
        case <-t.stopCh:
            t.handleStop()
            return nil
        case <-ticker.C:
            // 检查是否在交易时间内
            if !t.isTradingTime() {
                continue
            }

            // 执行决策
            if err := t.executeDecision(); err != nil {
                logger.Errorf("决策执行失败: %v", err)
            }
        }
    }
}

// isTradingTime 检查是否在交易时间内
func (t *StockTrader) isTradingTime() bool {
    now := time.Now()
    weekday := now.Weekday()

    // 周末不交易
    if weekday == time.Saturday || weekday == time.Sunday {
        return false
    }

    hour, min, _ := now.Clock()
    currentMinutes := hour*60 + min

    if t.cfg.Market == broker.MarketCN {
        // A 股交易时间: 9:30-11:30, 13:00-15:00
        if (currentMinutes >= 570 && currentMinutes <= 690) || // 9:30-11:30
           (currentMinutes >= 780 && currentMinutes <= 900) {  // 13:00-15:00
            return true
        }
    } else if t.cfg.Market == broker.MarketHK {
        // 港股交易时间: 9:30-12:00, 13:00-16:00
        if (currentMinutes >= 570 && currentMinutes <= 720) || // 9:30-12:00
           (currentMinutes >= 780 && currentMinutes <= 960) {  // 13:00-16:00
            return true
        }
    }

    return false
}

// executeDecision 执行交易决策
func (t *StockTrader) executeDecision() error {
    // 1. 获取账户信息
    account, err := t.broker.GetAccount()
    if err != nil {
        return err
    }

    // 2. 获取持仓
    positions, err := t.broker.GetPositions()
    if err != nil {
        return err
    }

    // 3. 调用 AI 决策引擎
    decisions, err := t.decision.MakeDecision(&Context{
        Account:    account,
        Positions:  positions,
        Symbols:    t.cfg.Symbols,
        MarketType: t.cfg.Market,
    })
    if err != nil {
        return err
    }

    // 4. 执行决策
    for _, dec := range decisions {
        if err := t.executeSingleDecision(dec); err != nil {
            logger.Errorf("执行决策失败 [%s]: %v", dec.Symbol, err)
        }
    }

    t.lastDecisionAt = time.Now()
    return nil
}

// executeSingleDecision 执行单个决策
func (t *StockTrader) executeSingleDecision(dec *Decision) error {
    switch dec.Action {
    case "buy":
        return t.executeBuy(dec)
    case "sell":
        return t.executeSell(dec)
    case "hold":
        // 持有不动
        return nil
    default:
        return fmt.Errorf("未知操作: %s", dec.Action)
    }
}

// executeBuy 执行买入
func (t *StockTrader) executeBuy(dec *Decision) error {
    // 计算买入数量 (必须是 100 的整数倍)
    account, _ := t.broker.GetAccount()

    // 单只股票最大仓位限制
    maxAmount := account.TotalAssets * t.cfg.MaxPosPct
    maxQty := int(maxAmount / dec.Price)

    // 调整为 100 的整数倍
    qty := (maxQty / 100) * 100
    if qty < 100 {
        return fmt.Errorf("资金不足，无法买入 1 手")
    }

    // 创建订单
    order := &broker.Order{
        Symbol:   dec.Symbol,
        Side:     broker.OrderBuy,
        Type:     broker.OrderLimit,
        Quantity: qty,
        Price:    dec.Price,
    }

    // 下单
    result, err := t.broker.PlaceOrder(order)
    if err != nil {
        return err
    }

    logger.Infof("买入成功: %s %d股 @ %.2f, 订单ID: %s",
        dec.Symbol, qty, dec.Price, result.OrderID)

    return nil
}

// executeSell 执行卖出
func (t *StockTrader) executeSell(dec *Decision) error {
    positions, _ := t.broker.GetPositions()

    var targetPos *broker.Position
    for _, pos := range positions {
        if pos.Symbol == dec.Symbol {
            targetPos = &pos
            break
        }
    }

    if targetPos == nil || targetPos.Quantity == 0 {
        return fmt.Errorf("没有持仓")
    }

    // A 股 T+1 检查
    if t.cfg.Market == broker.MarketCN && !targetPos.CanSellToday {
        return fmt.Errorf("A 股 T+1: 今日买入不可卖")
    }

    // 创建订单
    qty := targetPos.AvailableQty
    order := &broker.Order{
        Symbol:   dec.Symbol,
        Side:     broker.OrderSell,
        Type:     broker.OrderLimit,
        Quantity: qty,
        Price:    dec.Price,
    }

    // 下单
    result, err := t.broker.PlaceOrder(order)
    if err != nil {
        return err
    }

    logger.Infof("卖出成功: %s %d股 @ %.2f, 订单ID: %s",
        dec.Symbol, qty, dec.Price, result.OrderID)

    return nil
}
```

### 4.4 Phase 4: 回测引擎适配 (2 周)

```go
// backtest/stock_backtest.go
package backtest

import (
    "context"
    "time"
)

// StockBacktestConfig 股票回测配置
type StockBacktestConfig struct {
    RunID        string              `json:"run_id"`
    Market       string              `json:"market"`       // "CN", "HK"
    StartDate    string              `json:"start_date"`   // "2023-01-01"
    EndDate      string              `json:"end_date"`     // "2024-01-01"
    InitialCash  float64             `json:"initial_cash"` // 1000000
    Symbols      []string            `json:"symbols"`
    Strategy     *StrategyConfig     `json:"strategy"`

    // 股票特有配置
    Commission   float64             `json:"commission"`   // 佣金率 (默认 0.0003)
    StampTax     float64             `json:"stamp_tax"`    // 印花税 (仅卖出)
    MinCommission float64            `json:"min_commission"` // 最低佣金 (5 元)
}

// StockBacktestRunner 股票回测引擎
type StockBacktestRunner struct {
    cfg            *StockBacktestConfig
    feed           *StockDataFeed
    account        *StockAccount
    strategyEngine *decision.StrategyEngine

    // 状态
    state          *BacktestState
    status         RunState
}

// StockDataFeed 股票数据源
type StockDataFeed struct {
    symbols   []string
    startDate time.Time
    endDate   time.Time

    // K 线数据 (按日期索引)
    klines   map[string]map[string][]*market.KlineBar
}

// StockAccount 股票账户 (模拟 A 股 T+1)
type StockAccount struct {
    initialCash float64
    cash        float64
    positions   map[string]*StockBacktestPosition
    t0Positions map[string]*StockBacktestPosition // 当日买入 (不可卖)
    date        time.Time // 当前日期
}

type StockBacktestPosition struct {
    Symbol       string
    Quantity     int     // 持仓数量
    AvailableQty int     // 可卖数量
    AvgPrice     float64 // 成本价
    BuyDate      time.Time // 买入日期
}

// NewStockAccount 创建股票账户
func NewStockAccount(initialCash float64) *StockAccount {
    return &StockAccount{
        initialCash: initialCash,
        cash:        initialCash,
        positions:   make(map[string]*StockBacktestPosition),
        t0Positions: make(map[string]*StockBacktestPosition),
    }
}

// Buy 买入
func (a *StockAccount) Buy(symbol string, price float64, quantity int, date time.Time) error {
    // 调整为 100 的整数倍
    if quantity%100 != 0 {
        quantity = (quantity / 100) * 100
    }

    if quantity < 100 {
        return fmt.Errorf("买入数量必须 >= 100")
    }

    amount := float64(quantity) * price
    commission := a.calcCommission(amount)

    // 检查资金
    totalCost := amount + commission
    if a.cash < totalCost {
        return fmt.Errorf("资金不足: 需要 %.2f, 可用 %.2f", totalCost, a.cash)
    }

    // 扣除资金
    a.cash -= totalCost

    // 记录持仓 (A 股 T+1, 当日买入不可卖)
    a.t0Positions[symbol] = &StockBacktestPosition{
        Symbol:       symbol,
        Quantity:     quantity,
        AvailableQty: 0, // 当日不可卖
        AvgPrice:     price + commission/float64(quantity),
        BuyDate:      date,
    }

    return nil
}

// Sell 卖出
func (a *StockAccount) Sell(symbol string, price float64, quantity int, date time.Time) error {
    pos, ok := a.positions[symbol]
    if !ok {
        return fmt.Errorf("没有持仓")
    }

    if pos.AvailableQty < quantity {
        return fmt.Errorf("可卖数量不足")
    }

    amount := float64(quantity) * price
    commission := a.calcCommission(amount)
    stampTax := amount * 0.001 // 印花税 0.1%

    // 更新持仓
    pos.Quantity -= quantity
    pos.AvailableQty -= quantity

    // 增加资金
    a.cash += amount - commission - stampTax

    return nil
}

// calcCommission 计算佣金
func (a *StockAccount) calcCommission(amount float64) float64 {
    commission := amount * 0.0003 // 万分之三
    if commission < 5 {
        commission = 5 // 最低 5 元
    }
    return commission
}

// SettleT0 结算 T+0 持仓
func (a *StockAccount) SettleT0() {
    // 将 T+0 持仓转为可卖持仓
    for symbol, pos := range a.t0Positions {
        pos.AvailableQty = pos.Quantity // 变为可卖
        if existing, ok := a.positions[symbol]; ok {
            // 合并持仓
            totalQty := existing.Quantity + pos.Quantity
            totalCost := existing.AvgPrice*float64(existing.Quantity) +
                         pos.AvgPrice*float64(pos.Quantity)
            existing.Quantity = totalQty
            existing.AvailableQty += pos.AvailableQty
            existing.AvgPrice = totalCost / float64(totalQty)
        } else {
            a.positions[symbol] = pos
        }
    }
    a.t0Positions = make(map[string]*StockBacktestPosition)
}

// RunBacktest 运行股票回测
func (r *StockBacktestRunner) RunBacktest(ctx context.Context) error {
    r.status = RunStateRunning

    // 按日期迭代
    currentDate := r.feed.startDate
    for !currentDate.After(r.feed.endDate) {
        // 检查是否为交易日
        if !r.isTradingDay(currentDate) {
            currentDate = currentDate.AddDate(0, 0, 1)
            continue
        }

        // 1. 结算 T+0 持仓
        r.account.SettleT0()

        // 2. 执行交易决策
        if err := r.executeDecision(currentDate); err != nil {
            logger.Errorf("决策执行失败 [%s]: %v", currentDate, err)
        }

        // 3. 更新权益
        r.updateEquity(currentDate)

        currentDate = currentDate.AddDate(0, 0, 1)
    }

    r.status = RunStateCompleted
    return nil
}

// executeDecision 执行决策
func (r *StockBacktestRunner) executeDecision(date time.Time) error {
    // 获取当日市场数据
    marketData := make(map[string]*market.Data)
    for _, symbol := range r.cfg.Symbols {
        data, err := r.feed.GetDataOnDate(symbol, date)
        if err != nil {
            continue
        }
        marketData[symbol] = data
    }

    // 构建决策上下文
    ctx := &decision.Context{
        CurrentTime:   date.Format("2006-01-02"),
        Account:       r.getAccountInfo(),
        Positions:     r.getPositionsInfo(),
        MarketDataMap: marketData,
        MarketType:    r.cfg.Market,
    }

    // 调用 AI 决策
    decisions, err := r.strategyEngine.Execute(ctx)
    if err != nil {
        return err
    }

    // 执行交易
    for _, dec := range decisions {
        r.executeTrade(dec, date)
    }

    return nil
}

// isTradingDay 检查是否为交易日
func (r *StockBacktestRunner) isTradingDay(date time.Time) bool {
    // 简单判断: 排除周末
    weekday := date.Weekday()
    if weekday == time.Saturday || weekday == time.Sunday {
        return false
    }
    // TODO: 需要配合交易日历排除节假日
    return true
}
```

### 4.5 Phase 5: 决策引擎适配 (1 周)

```go
// decision/stock_prompt_builder.go
package decision

import (
    "fmt"
    "strings"
)

// StockPromptBuilder 股票提示词构建器
type StockPromptBuilder struct {
    lang     Language
    strategy *StrategyConfig
}

// BuildSystemPrompt 构建系统提示词 (股票版本)
func (b *StockPromptBuilder) BuildSystemPrompt(equity float64, variant string) string {
    var basePrompt strings.Builder

    // 移除期货特有概念
    basePrompt.WriteString("# 🎯 股票交易决策系统\n\n")
    basePrompt.WriteString("你是一个专业的股票交易 AI 助手，负责分析市场并做出交易决策。\n\n")

    // 添加市场特性
    if b.strategy.Market == "CN" {
        basePrompt.WriteString("## 📊 A 股交易规则\n\n")
        basePrompt.WriteString("- **交易时间**: 周一至周五 9:30-11:30, 13:00-15:00\n")
        basePrompt.WriteString("- **交割制度**: T+1 (今日买入明日可卖)\n")
        basePrompt.WriteString("- **交易单位**: 100 股为 1 手\n")
        basePrompt.WriteString("- **涨跌停限制**: 主板 10%, 创业板/科创板 20%\n")
        basePrompt.WriteString("- **交易方向**: 仅做多 (不能做空)\n")
        basePrompt.WriteString("- **手续费**: 佣金万分之三 + 印花税千分之一 (仅卖出)\n\n")
    } else if b.strategy.Market == "HK" {
        basePrompt.WriteString("## 📊 港股交易规则\n\n")
        basePrompt.WriteString("- **交易时间**: 周一至周五 9:30-12:00, 13:00-16:00\n")
        basePrompt.WriteString("- **交割制度**: T+0 (当日可买卖)\n")
        basePrompt.WriteString("- **交易单位**: 部分股票 100 股/手，部分 1000 股/手\n")
        basePrompt.WriteString("- **涨跌停限制**: 无限制 (个别股票有波动调节)\n")
        basePrompt.WriteString("- **交易方向**: 可做多做空 (通过衍生品)\n\n")
    }

    // 添加数据字典 (移除期货特有字段)
    basePrompt.WriteString(GetStockSchemaPrompt(b.lang))

    // 添加交易规则
    basePrompt.WriteString(b.buildTradingRules())

    return basePrompt.String()
}

// buildTradingRules 构建交易规则
func (b *StockPromptBuilder) buildTradingRules() string {
    var sb strings.Builder

    sb.WriteString("## ⚖️ 交易规则\n\n")

    // 风险管理
    sb.WriteString("### 风险管理\n")
    sb.WriteString(fmt.Sprintf("- **最大持仓数**: %d 只股票\n", b.strategy.MaxPositions))
    sb.WriteString(fmt.Sprintf("- **单股最大仓位**: %.1f%%\n", b.strategy.MaxPositionPct*100))
    sb.WriteString(fmt.Sprintf("- **止损比例**: %.1f%%\n", b.strategy.StopLoss*100))
    sb.WriteString(fmt.Sprintf("- **止盈比例**: %.1f%%\n", b.strategy.TakeProfit*100))

    // A 股特有规则
    if b.strategy.Market == "CN" {
        sb.WriteString("\n### A 股特有规则\n")
        sb.WriteString("- **T+1 限制**: 今日买入的股票今日不可卖\n")
        sb.WriteString("- **涨停限制**: 涨停板可能无法买入\n")
        sb.WriteString("- **跌停限制**: 跌停板可能无法卖出\n")
    }

    return sb.String()
}

// GetStockSchemaPrompt 获取股票数据字典
func GetStockSchemaPrompt(lang Language) string {
    if lang == LangChinese {
        return getStockSchemaPromptZH()
    }
    return getStockSchemaPromptEN()
}

func getStockSchemaPromptZH() string {
    return `
## 📖 数据字典

### 账户指标
- **Equity** (总权益): 现金 + 持仓市值
- **Cash** (可用资金): 可用于买入股票的资金
- **MarketValue** (证券市值): 所有持仓的市值之和
- **TotalPnL** (总盈亏): (当前总权益 - 初始资金) / 初始资金

### 持仓指标
- **Symbol** (股票代码): 如 600519.SH (贵州茅台)
- **Quantity** (持仓数量): 股数 (必须是 100 的整数倍)
- **AvailableQty** (可卖数量): 可卖出数量 (A 股受 T+1 限制)
- **CostPrice** (成本价): 买入均价 (含手续费)
- **MarketPrice** (市价): 当前市场价格
- **UnrealizedPnL** (浮动盈亏): (市价 - 成本价) × 持仓数量
- **UnrealizedPnLPct** (浮动盈亏百分比): 浮动盈亏 / 成本 × 100%

### 市场数据
- **Price** (最新价): 当前成交价
- **Change** (涨跌额): 当前价 - 昨收价
- **ChangePct** (涨跌幅): 涨跌额 / 昨收价 × 100%
- **Volume** (成交量): 成交股数
- **Amount** (成交额): 成交金额
- **TurnoverRate** (换手率): 成交量 / 流通股本 × 100%
- **Amplitude** (振幅): (最高 - 最低) / 昨收 × 100%

### 股票特有指标
- **PE** (市盈率): 股价 / 每股收益
- **PB** (市净率): 股价 / 每股净资产
- **DividendYield** (股息率): 每股股息 / 股价 × 100%
- **MainInflow** (主力净流入): 大单净买入额

### 决策动作
- **buy**: 买入股票
- **sell**: 卖出股票
- **hold**: 持仓不动
- **wait**: 空仓等待
`
}
```

---

## 5. 技术选型建议

### 5.1 行情数据源

| 数据源 | A 股 | 港股 | 成本 | 推荐度 |
|--------|------|------|------|--------|
| **新浪财经** | ✅ | ✅ | 免费 | ⭐⭐⭐⭐ |
| **腾讯财经** | ✅ | ✅ | 免费 | ⭐⭐⭐⭐ |
| **东方财富** | ✅ | ✅ | 免费 | ⭐⭐⭐⭐⭐ |
| **网易财经** | ✅ | ✅ | 免费 | ⭐⭐⭐ |
| **Tushare Pro** | ✅ | ✅ | 付费 | ⭐⭐⭐⭐⭐ |
| **AKShare** | ✅ | ✅ | 开源 | ⭐⭐⭐⭐⭐ |
| **Wind** | ✅ | ✅ | 高价 | ⭐⭐⭐ |

**推荐方案**:
- 开发/测试阶段: 新浪财经 + 东方财富 (免费)
- 生产环境: Tushare Pro / AKShare (稳定、数据全)

### 5.2 券商接口

| 券商 | 接口类型 | A 股 | 港股 | 接入难度 |
|------|---------|------|------|----------|
| **富途牛牛** | OpenAPI | ❌ | ✅ | 中 |
| **老虎证券** | OpenAPI | ✅ | ✅ | 中 |
| **华盛通** | API | ✅ | ✅ | 中 |
| **雪球** | API | ✅ | ✅ | 低 |
| **同花顺** | 模拟交易 | ✅ | ❌ | 低 |
| **国金证券** | 量化接口 | ✅ | ❌ | 高 |

**推荐方案**:
- A 股: 同花顺模拟交易 (开发) → 券商实盘接口 (生产)
- 港股: 富途 OpenAPI / 老虎证券

### 5.3 历史数据存储

```go
// 股票历史数据存储结构
type StockKline struct {
    Symbol    string    `json:"symbol"`     // 600519.SH
    Timestamp int64     `json:"timestamp"`  // 时间戳
    Open      float64   `json:"open"`       // 开盘价
    High      float64   `json:"high"`       // 最高价
    Low       float64   `json:"low"`        // 最低价
    Close     float64   `json:"close"`      // 收盘价
    Volume    int64     `json:"volume"`     // 成交量 (股)
    Amount    float64   `json:"amount"`     // 成交额 (元)
    Turnover  float64   `json:"turnover"`   // 换手率

    // 复权因子
    AdjFactor float64   `json:"adj_factor"` // 复权因子
}

// 复权类型
type AdjType string

const (
    AdjNone    AdjType = "none"    // 不复权
    AdjForward AdjType = "forward" // 前复权
    AdjBack    AdjType = "back"    // 后复权
)
```

---

## 6. 风险与挑战

### 6.1 技术风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 数据源不稳定 | 回测/交易失败 | 多数据源备份、本地缓存 |
| API 限制 | 频率受限 | 请求合并、增量更新 |
| 时序对齐 | 跨市场数据不一致 | 统一时间戳、时区处理 |
| 精度问题 | 计算误差 | 使用 Decimal 类型 |

### 6.2 业务风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 涨跌停无法成交 | 策略失效 | 预判涨停、限价挂单 |
| T+1 限制无法平仓 | 资金占用 | 预留现金、仓位控制 |
| 流动性不足 | 滑点过大 | 选择活跃股票、分批交易 |
| 停牌风险 | 无法交易 | 实时监控、提前处理 |

### 6.3 合规风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 量化交易监管 | 策略受限 | 关注监管动态、合规设计 |
| 账户关联风险 | 账户冻结 | 分散券商、资金隔离 |
| 异常交易检测 | 账户限制 | 避免异常模式、人工监督 |

---

## 7. 附录

### 7.1 A 股股票代码规范

```
格式: 代码.市场

示例:
- 600519.SH (上海主板 - 贵州茅台)
- 000001.SZ (深圳主板 - 平安银行)
- 300001.SZ (深圳创业板 - 特锐德)
- 688001.SH (上海科创板 - 时代电气

市场代码:
- SH: 上海证券交易所
- SZ: 深圳证券交易所

板块代码:
- 6xxxx: 上海主板
- 0xxxx: 深圳主板
- 3xxxx: 深圳创业板
- 6xxxx: 上海科创板
```

### 7.2 港股股票代码规范

```
格式: 代码.HK

示例:
- 0700.HK (腾讯控股)
- 9988.HK (阿里巴巴)
- 0941.HK (中国移动)

代码规则:
- 4位数字: 主板
- 5位数字: 创业板 (可选)
```

### 7.3 交易时间对照表

| 市场 | 时区 | 交易时间 (周一至周五) |
|------|------|---------------------|
| A 股 | UTC+8 | 9:30-11:30, 13:00-15:00 |
| 港股 | UTC+8 | 9:30-12:00, 13:00-16:00 |
| 加密货币 | UTC+0 | 7×24 小时 |

### 7.4 A 股手续费明细

| 项目 | 费率 | 备注 |
|------|------|------|
| 佣金 | 万分之三 | 最低 5 元 |
| 印花税 | 千分之一 | 仅卖出收取 |
| 过户费 | 万分之0.2 | 上海收取 |
| 规费 | 万分之0.687 | 经手费+证管费 |

**总费率**:
- 买入: 约 0.003087% (佣金 + 规费)
- 卖出: 约 0.103087% (佣金 + 印花税 + 规费)

### 7.5 代码结构建议

```
nofx/
├── provider/
│   └── broker/              # 新增: 券商接口层
│       ├── interface.go      # Broker 接口定义
│       ├── cn/               # A 股券商
│       │   ├── simulated.go  # 模拟券商
│       │   ├── tonghuashun/  # 同花顺实盘
│       │   └── ...
│       └── hk/               # 港股券商
│           ├── simulated.go  # 模拟券商
│           ├── futu/         # 富途牛牛
│           └── ...
├── market/
│   ├── quote/                # 新增: 行情数据层
│   │   ├── provider.go       # QuoteProvider 接口
│   │   ├── cn/               # A 股行情
│   │   │   ├── sina.go       # 新浪财经
│   │   │   ├── tushare.go    # Tushare Pro
│   │   │   └── akshare.go    # AKShare (通过 Python)
│   │   └── hk/               # 港股行情
│   │       ├── sina.go       # 新浪财经
│   │       └── eastmoney.go  # 东方财富
│   └── indicators/           # 新增: 股票指标
│       └── stock_indicators.go
├── trader/
│   ├── stock_trader.go       # 改造: 股票交易器
│   └── crypto_trader.go      # 保留: 加密货币交易器
├── backtest/
│   ├── stock_backtest.go     # 改造: 股票回测
│   └── crypto_backtest.go    # 保留: 加密货币回测
├── decision/
│   ├── stock_prompt.go       # 新增: 股票提示词
│   └── crypto_prompt.go      # 保留: 加密货币提示词
└── store/
    ├── stock_schema.sql      # 新增: 股票数据表
    └── crypto_schema.sql     # 保留: 加密货币数据表
```

---

**文档版本**: 1.0
**创建日期**: 2025-12-28
**作者**: Claude Code
**适用版本**: NoFX v1.x

---

## 实施路线图

```
Week 1-2:  基础架构搭建
├── Provider 抽象层设计
├── 市场数据源接入 (A 股/港股)
└── 配置系统改造

Week 3-4:  交易功能实现
├── 券商接口适配 (模拟)
├── 订单管理系统
└── 持仓管理 (T+1 逻辑)

Week 5-6:  回测系统适配
├── 股票数据源
├── 回测引擎改造
└── 性能指标计算

Week 7-8:  AI 决策适配
├── 股票提示词设计
├── 决策引擎调整
└── 策略配置更新

Week 9-10: 前端适配与联调
├── UI 调整 (股票代码格式)
├── 实时行情展示
├── 订单界面更新
└── 整体联调测试

Week 11-12: 测试与优化
├── 单元测试
├── 集成测试
├── 性能优化
└── 文档完善
```
