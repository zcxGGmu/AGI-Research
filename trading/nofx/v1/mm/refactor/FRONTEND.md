# 港A股交易前端重构设计方案

## 1. 现有架构分析

### 1.1 当前前端技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 框架 | React 18 + TypeScript | 主流前端框架 |
| 构建 | Vite | 快速开发构建 |
| 样式 | Tailwind CSS | 原子化CSS |
| 状态管理 | Zustand | 轻量级状态管理 |
| 图表 | TradingView Widget | 第三方图表 |
| HTTP | SWR + fetch | 数据获取与缓存 |
| UI组件 | Radix UI + 自定义 | 对话框、按钮等 |

### 1.2 现有目录结构

```
web/src/
├── components/
│   ├── charts/          # 图表组件 (TradingView, EquityChart)
│   ├── common/          # 通用组件 (Header, ConfirmDialog, Container)
│   ├── trader/          # 交易者组件 (DecisionCard, PositionHistory)
│   ├── strategy/        # 策略编辑组件
│   ├── auth/            # 认证组件
│   └── landing/         # 落地页组件
├── pages/               # 页面组件
│   ├── TraderDashboardPage.tsx
│   └── ...
├── stores/              # Zustand stores
│   ├── tradersConfigStore.ts
│   └── tradersModalStore.ts
├── types/               # TypeScript类型
│   ├── trading.ts       # 交易类型 (Position, DecisionAction)
│   └── config.ts
├── lib/api/             # API调用层
│   ├── traders.ts
│   └── strategies.ts
├── hooks/                # 自定义Hooks
└── contexts/            # React Contexts
```

### 1.3 现有类型定义问题

现有类型专为**合约/杠杆交易**设计，不适用于港A股：

```typescript
// 现有 Position 类型 (不适用)
interface Position {
  symbol: string
  side: string              // "long" | "short"
  leverage: number          // 杠杆倍数 (合约特有)
  liquidation_price: number  // 强平价格 (合约特有)
  margin_used: number       // 已用保证金 (合约特有)
}

// 现有 DecisionAction 类型 (不适用)
interface DecisionAction {
  action: string            // "open_long" | "open_short" | "close_long"
  leverage: number
  stop_loss?: number
  take_profit?: number
}
```

---

## 2. 港A股前端架构设计

### 2.1 新目录结构

```
web/src/
├── components/
│   ├── hk-ashare/         # 港A股特有组件 (新)
│   │   ├── MarketBoard/        # 涨跌幅排行榜
│   │   ├── RotationChart/      # 轮动分析图表
│   │   ├── PositionPanel/      # 持仓管理面板
│   │   ├── TradeDialog/        # 交易确认对话框
│   │   └── AccountOverview/    # 资金账户概览
│   ├── charts/             # 图表组件 (重构)
│   │   ├── TradingViewChart/   # 重构支持港股代码
│   │   └── StockChart/         # A股专用图表
│   ├── common/             # 通用组件 (保留)
│   ├── trading/            # 交易组件 (重构)
│   │   ├── OrderBook/          # 订单簿
│   │   ├── TradePanel/         # 交易面板
│   │   └── PositionList/       # 持仓列表
│   └── ui/                 # 基础UI (保留)
├── pages/
│   ├── HKTraderDashboard/  # 港股仪表板 (新)
│   ├── AShareDashboard/    # A股仪表板 (新)
│   └── TraderDashboardPage.tsx  # 保留加密货币
├── stores/
│   ├── hkAshareStore.ts     # 港A股状态 (新)
│   ├── marketDataStore.ts   # 行情数据状态 (新)
│   └── tradingStore.ts     # 交易状态 (新)
├── types/
│   ├── hk-ashare.ts        # 港A股类型 (新)
│   ├── trading.ts          # 重构
│   └── market.ts           # 行情类型 (新)
├── lib/api/
│   ├── hkAshare.ts         # 港A股 API (新)
│   ├── market.ts           # 行情 API (新)
│   └── websocket.ts        # WebSocket 管理 (新)
└── hooks/
    ├── useWebSocket.ts     # WebSocket Hook (新)
    └── useMarketData.ts     # 行情数据 Hook (新)
```

### 2.2 组件设计

#### 2.2.1 行情看板组件 (MarketBoard)

```typescript
// components/hk-ashare/MarketBoard/index.tsx
interface MarketBoardProps {
  market: 'HK' | 'SH' | 'SZ'  // 港股、上海、深圳
  category: 'gainers' | 'losers' | 'turnover' | 'volume'
  onStockClick?: (symbol: string) => void
}

// 伪代码
function MarketBoard({ market, category, onStockClick }: MarketBoardProps) {
  const { data, loading } = useHKAshareMarketData(market, category)

  return (
    <div className="market-board">
      <Header>
        <MarketTabs market={market} />
        <CategoryTabs category={category} />
      </Header>
      <Table>
        <thead>
          <tr>
            <th>排名</th>
            <th>股票代码</th>
            <th>股票名称</th>
            <th>最新价</th>
            <th>涨跌幅</th>
            <th>涨跌额</th>
            <th>成交量</th>
            <th>成交额</th>
            <th>换手率</th>
          </tr>
        </thead>
        <tbody>
          {data?.stocks.map((stock, index) => (
            <StockRow
              key={stock.symbol}
              rank={index + 1}
              stock={stock}
              onClick={() => onStockClick?.(stock.symbol)}
            />
          ))}
        </tbody>
      </Table>
    </div>
  )
}
```

#### 2.2.2 持仓管理面板 (PositionPanel)

```typescript
// components/hk-ashare/PositionPanel/index.tsx
interface PositionPanelProps {
  positions: AsharePosition[]
  onClosePosition: (order: CloseOrder) => void
  onAddMargin: (order: AddMarginOrder) => void
}

interface AsharePosition {
  symbol: string           // e.g., "00700" (腾讯) or "600519" (茅台)
  name: string             // 股票名称
  quantity: number         // 持股数量 (港股) / 股数 (A股)
  available: number         // 可卖数量
  cost: number             // 成本价
  currentPrice: number     // 当前价
  marketValue: number      // 市值
  unrealizedPnL: number    // 浮动盈亏
  unrealizedPnLPct: number // 盈亏比例
  // 融资融券特有
  marginUsed?: number      // 融资额
  marginRatio?: number     // 保证金比例
  // A股特有
  tradingDay?: string      // 交易日 (T+1)
  exchange?: 'SSE' | 'SZSE' | 'HKEX'
}
```

#### 2.2.3 交易确认对话框 (TradeDialog)

```typescript
// components/hk-ashare/TradeDialog/index.tsx
interface TradeDialogProps {
  open: boolean
  onClose: () => void
  onConfirm: (order: TradeOrder) => void
  order: TradeOrder | null
}

interface TradeOrder {
  symbol: string
  name: string
  side: 'buy' | 'sell'
  type: 'market' | 'limit' | 'stop'
  price?: number           // 限价/止损价格
  quantity: number         // 数量
  // 港股特有
  lotSize: number          // 每手股数
  boardLot: number         // 买卖盘单位
  // 费用
  estimatedFee: {
    commission: number     // 佣金
    stampDuty: number      // 印花税 (A股卖出时收取)
    exchangeFee: number    // 过户费/交易征费
  }
}
```

#### 2.2.4 资金账户概览 (AccountOverview)

```typescript
// components/hk-ashare/AccountOverview/index.tsx
interface AccountOverviewProps {
  account: AshareAccount
}

interface AshareAccount {
  totalAssets: number      // 总资产
  totalMarketValue: number // 持仓市值
  availableCash: number    // 可用资金
  frozenCash: number       // 冻结资金 (买入时冻结)
  todayProfit: number      // 今日盈亏
  todayProfitPct: number    // 今日收益率
  totalProfit: number      // 总盈亏
  totalProfitPct: number    // 总收益率
  // 融资融券
  marginUsed: number       // 已用融资额度
  marginAvailable: number  // 可用融资额度
  // A股特有
  sellableToday: number    // 今日可卖 (T+1)
}
```

#### 2.2.5 轮动分析图表 (RotationChart)

```typescript
// components/hk-ashare/RotationChart/index.tsx
interface RotationChartProps {
  symbols: string[]
  intervals: string[]
}

function RotationChart({ symbols, intervals }: RotationChartProps) {
  // 显示行业/板块轮动时序图
  // X轴: 时间
  // Y轴: 行业/板块
  // 颜色/大小: 涨跌幅或资金流向

  return (
    <div className="rotation-heatmap">
      <Chart纵轴="板块" 横轴="时间" 数据={rotationData} />
    </div>
  )
}
```

---

## 3. 类型定义重构

### 3.1 新增港A股类型

```typescript
// types/hk-ashare.ts

// === 市场数据 ===

export type Market = 'HK' | 'SH' | 'SZ'
export type Exchange = 'HKEX' | 'SSE' | 'SZSE'

export interface StockQuote {
  symbol: string
  name: string
  exchange: Exchange
  lastPrice: number
  prevClose: number
  change: number
  changePct: number
  volume: number
  turnover: number
  high: number
  low: number
  open: number
  timestamp: number
}

export interface MarketIndex {
  name: string
  code: string
  current: number
  change: number
  changePct: number
  volume: number
  turnover: number
}

// === 订单与持仓 ===

export type OrderSide = 'buy' | 'sell'
export type OrderType = 'market' | 'limit' | 'stop' | 'stop_limit'
export type OrderStatus = 'pending' | 'partial' | 'filled' | 'cancelled' | 'rejected'

export interface StockOrder {
  orderId: string
  symbol: string
  name: string
  side: OrderSide
  type: OrderType
  price: number
  quantity: number
  filledQty: number
  avgPrice: number
  status: OrderStatus
  createdAt: string
  updatedAt: string
  // 港股特有
  lotSize?: number
  // A股特有
  accountId?: string
}

export interface AsharePosition {
  symbol: string
  name: string
  exchange: Exchange
  quantity: number
  available: number
  cost: number
  currentPrice: number
  marketValue: number
  unrealizedPnL: number
  unrealizedPnLPct: number
  // 融资融券
  marginUsed?: number
  marginRatio?: number
  // A股 T+1
  sellableToday?: number
}

export interface AshareAccount {
  accountId: string
  exchange: Exchange
  totalAssets: number
  marketValue: number
  availableCash: number
  frozenCash: number
  todayProfit: number
  todayProfitPct: number
  totalProfit: number
  totalProfitPct: number
  // 融资
  marginUsed?: number
  marginAvailable?: number
}

// === 排行榜 ===

export type RankingCategory = 'gainers' | 'losers' | 'turnover' | 'volume' | 'mostActive'

export interface StockRanking {
  rank: number
  symbol: string
  name: string
  lastPrice: number
  change: number
  changePct: number
  volume: number
  turnover: number
}
```

### 3.2 原有类型重构

```typescript
// types/trading.ts (重构后)

// 保留加密货币交易类型，新增港A股兼容层
export interface Position {
  // 共用字段
  symbol: string
  side: 'long' | 'short'
  quantity: number
  entry_price: number
  mark_price: number
  unrealized_pnl: number
  unrealized_pnl_pct: number

  // 加密货币特有
  leverage?: number
  liquidation_price?: number
  margin_used?: number

  // 港A股特有
  exchange?: Exchange
  name?: string
  available?: number
  cost?: number
}
```

---

## 4. 状态管理重构

### 4.1 新增 Zustand Stores

```typescript
// stores/hkAshareStore.ts
import { create } from 'zustand'

interface HKAshareState {
  // 当前选中的市场
  currentMarket: Market
  currentExchange: Exchange

  // 账户数据
  accounts: Record<Exchange, AshareAccount | null>
  positions: Record<Exchange, AsharePosition[]>

  // 订单数据
  orders: StockOrder[]
  orderHistory: StockOrder[]

  // Actions
  setCurrentMarket: (market: Market) => void
  setCurrentExchange: (exchange: Exchange) => void
  updateAccount: (exchange: Exchange, account: AshareAccount) => void
  updatePositions: (exchange: Exchange, positions: AsharePosition[]) => void
  addOrder: (order: StockOrder) => void
  updateOrder: (orderId: string, updates: Partial<StockOrder>) => void
}

export const useHKAshareStore = create<HKAshareState>((set) => ({
  currentMarket: 'HK',
  currentExchange: 'HKEX',
  accounts: { HKEX: null, SSE: null, SZSE: null },
  positions: { HKEX: [], SSE: [], SZSE: [] },
  orders: [],
  orderHistory: [],

  setCurrentMarket: (market) => set({ currentMarket: market }),
  setCurrentExchange: (exchange) => set({ currentExchange: exchange }),

  updateAccount: (exchange, account) => set((state) => ({
    accounts: { ...state.accounts, [exchange]: account }
  })),

  updatePositions: (exchange, positions) => set((state) => ({
    positions: { ...state.positions, [exchange]: positions }
  })),

  addOrder: (order) => set((state) => ({
    orders: [...state.orders, order]
  })),

  updateOrder: (orderId, updates) => set((state) => ({
    orders: state.orders.map((o) =>
      o.orderId === orderId ? { ...o, ...updates } : o
    )
  })),
}))

// stores/marketDataStore.ts
interface MarketDataState {
  // 实时行情数据
  quotes: Record<string, StockQuote>

  // 指数数据
  indices: Record<string, MarketIndex>

  // 排行榜数据
  rankings: Record<Market, Record<RankingCategory, StockRanking[]>>

  // WebSocket 连接状态
  wsConnected: boolean

  // Actions
  updateQuote: (quote: StockQuote) => void
  updateIndex: (index: MarketIndex) => void
  setRankings: (market: Market, category: RankingCategory, data: StockRanking[]) => void
  setWsConnected: (connected: boolean) => void
}
```

---

## 5. API 层设计

### 5.1 REST API 接口

```typescript
// lib/api/hkAshare.ts

export const hkAshareApi = {
  // 账户
  getAccounts: () => api.get<{ accounts: AshareAccount[] }>('/api/hk-ashare/accounts'),

  // 持仓
  getPositions: (exchange: Exchange) =>
    api.get<{ positions: AsharePosition[] }>(`/api/hk-ashare/${exchange}/positions`),

  // 订单
  getOrders: (exchange: Exchange, status?: OrderStatus) =>
    api.get<{ orders: StockOrder[] }>(`/api/hk-ashare/${exchange}/orders`, { status }),

  // 下单
  placeOrder: (exchange: Exchange, order: Omit<StockOrder, 'orderId' | 'status'>) =>
    api.post<{ order: StockOrder }>(`/api/hk-ashare/${exchange}/orders`, order),

  // 撤单
  cancelOrder: (exchange: Exchange, orderId: string) =>
    api.delete<void>(`/api/hk-ashare/${exchange}/orders/${orderId}`),
}

// lib/api/market.ts
export const marketApi = {
  // 行情
  getQuote: (symbol: string, exchange: Exchange) =>
    api.get<{ quote: StockQuote }>(`/api/market/${exchange}/${symbol}`),

  // 排行榜
  getMarketBoard: (market: Market, category: RankingCategory) =>
    api.get<{ rankings: StockRanking[] }>(`/api/market/${market}/ranking`, { category }),

  // 指数
  getIndices: (market: Market) =>
    api.get<{ indices: MarketIndex[] }>(`/api/market/${market}/indices`),
}
```

### 5.2 WebSocket 实时行情

```typescript
// lib/websocket.ts

type WsMessage =
  | { type: 'quote'; data: StockQuote }
  | { type: 'index'; data: MarketIndex }
  | { type: 'order_update'; data: StockOrder }
  | { type: 'position_update'; data: AsharePosition }

class MarketWebSocket {
  private ws: WebSocket | null = null
  private subscribers: Map<string, Set<(msg: WsMessage) => void>> = new Map()
  private reconnectAttempts = 0
  private maxReconnectAttempts = 5

  connect(url: string) {
    this.ws = new WebSocket(url)

    this.ws.onmessage = (event) => {
      const message: WsMessage = JSON.parse(event.data)
      this.notifySubscribers(message.type, message)
    }

    this.ws.onclose = () => this.handleReconnect(url)
    this.ws.onerror = (error) => console.error('WebSocket error:', error)
  }

  subscribe(topic: string, callback: (msg: WsMessage) => void) {
    if (!this.subscribers.has(topic)) {
      this.subscribers.set(topic, new Set())
      this.ws?.send(JSON.stringify({ action: 'subscribe', topic }))
    }
    this.subscribers.get(topic)!.add(callback)
  }

  unsubscribe(topic: string, callback: (msg: WsMessage) => void) {
    this.subscribers.get(topic)?.delete(callback)
    if (this.subscribers.get(topic)?.size === 0) {
      this.subscribers.delete(topic)
      this.ws?.send(JSON.stringify({ action: 'unsubscribe', topic }))
    }
  }

  private notifySubscribers(type: string, message: WsMessage) {
    this.subscribers.get(type)?.forEach((cb) => cb(message))
  }

  private handleReconnect(url: string) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++
      setTimeout(() => this.connect(url), 1000 * this.reconnectAttempts)
    }
  }
}

export const marketWs = new MarketWebSocket()
```

---

## 6. TradingView 图表适配

### 6.1 港股代码映射

```typescript
// TradingViewChart 重构支持港A股
const MARKET_CONFIG = {
  HK: {
    prefix: 'HKEX:',
    suffix: '',
    example: 'HKEX:00700'  // 腾讯
  },
  SH: {
    prefix: 'SSE:',
    suffix: '',
    example: 'SSE:600519'  // 茅台
  },
  SZ: {
    prefix: 'SZSE:',
    suffix: '',
    example: 'SZSE:000858'  // 五粮液
  }
}

// 指数代码
const INDEX_CONFIG = {
  'HSI': 'HKEX:HSI',      // 恒生指数
  'HSTECH': 'HKEX:HSTECH', // 恒生科技
  'SSE:000001': 'SSE:000001', // 上证指数
  'SZSE:399001': 'SZSE:399001' // 深证成指
}
```

### 6.2 重构后的组件

```typescript
// components/charts/TradingViewChart/index.tsx (重构)

interface TradingViewChartProps {
  symbol: string
  market: 'HK' | 'SH' | 'SZ'
  height?: number
  interval?: string
  showToolbar?: boolean
  theme?: 'dark' | 'light'
}

function TradingViewChart({
  symbol,
  market,
  height = 400,
  interval = 'D',
  showToolbar = true,
  theme = 'dark'
}: TradingViewChartProps) {
  // 构建正确的 TradingView 符号格式
  const getFullSymbol = () => {
    const config = MARKET_CONFIG[market]
    // 港股代码处理 (去掉前缀)
    const normalizedSymbol = symbol.replace(/^(00700|00762|06039)/, match => match)
    return `${config.prefix}${normalizedSymbol}${config.suffix}`
  }

  // ... 原有 TradingView Widget 加载逻辑
}
```

---

## 7. 与现有组件的差异

| 组件 | 现有 | 重构后 | 差异说明 |
|------|------|--------|----------|
| TradingViewChart | 仅支持加密货币 (BINANCE:, BYBIT:) | 支持港A股代码 | 新增 HK/SH/SZ 市场配置 |
| Position | 合约持仓 (杠杆/保证金) | A股/港股持仓 (T+1/每手) | 移除杠杆字段，新增交易制度字段 |
| OrderBook | 加密货币订单簿 | 股票订单簿 | 买卖盘口显示逻辑不同 |
| DecisionCard | 交易决策展示 | 保留作为参考 | 港股交易无需 AI 决策 |
| Header | 通用 Header | 扩展语言切换 | 增加市场切换入口 |

---

## 8. 实施建议

### 8.1 分阶段实施

1. **Phase 1: 类型与API层**
   - 新增 `types/hk-ashare.ts`
   - 新增 `lib/api/hkAshare.ts`
   - 新增 `lib/websocket.ts`

2. **Phase 2: 状态管理**
   - 新增 `stores/hkAshareStore.ts`
   - 新增 `stores/marketDataStore.ts`

3. **Phase 3: 核心组件**
   - MarketBoard (涨跌幅排行榜)
   - AccountOverview (资金概览)
   - PositionPanel (持仓管理)

4. **Phase 4: 交易流程**
   - TradeDialog (交易确认)
   - OrderPanel (订单输入)

5. **Phase 5: 图表与高级功能**
   - TradingViewChart 港股适配
   - RotationChart (轮动分析)

### 8.2 复用策略

- **保留复用**: Header, ConfirmDialog, Container, 基本UI组件
- **重构复用**: TradingViewChart (扩展市场支持)
- **新增**: 港A股特有组件
