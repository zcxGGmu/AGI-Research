# NOFX 港A股 LangGraph 重构方案

> **版本**: v2.0
> **日期**: 2026-03-19
> **状态**: 待审核

---

## 目录

1. [项目概述](#1-项目概述)
2. [现有架构分析](#2-现有架构分析)
3. [重构目标与范围](#3-重构目标与范围)
4. [技术选型与理由](#4-技术选型与理由)
5. [LangGraph 架构设计](#5-langgraph-架构设计)
6. [港A股适配方案](#6-港a股适配方案)
7. [模块迁移计划](#7-模块迁移计划)
8. [前端重构方案](#8-前端重构方案)
9. [实施路线图](#9-实施路线图)
10. [风险评估与缓解](#10-风险评估与缓解)
11. [测试与部署策略](#11-测试与部署策略)
12. [附录](#附录)

---

## 1. 项目概述

### 1.1 背景

**NOFX** 是一个开源的 AI 驱动加密货币自动交易系统，具备以下特性：

- 支持 10+ 加密货币交易所（CEX/DEX）
- 支持 7+ AI 模型提供商
- 可视化策略工作室
- Telegram 智能体交互
- 实时仪表板

**项目地址**: `/home/zq/work-space/repo/ai-projs/trade/nofx`

### 1.2 重构动机

| 动机 | 说明 |
|------|------|
| **市场转换** | 从加密货币转向港A股市场 |
| **架构升级** | 使用 LangGraph 实现更灵活的智能体工作流 |
| **可维护性** | Go 后端 → Python 后端，降低学习曲线 |
| **AI 能力增强** | 利用 LangGraph 的持久化、人机协作、多智能体协作 |

### 1.3 重构目标

```
┌─────────────────────────────────────────────────────────────────┐
│                        重构目标                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. 完全移除加密货币支持，替换为港A股交易                         │
│  2. 使用 LangGraph 重构核心决策流程                              │
│  3. 保持前端界面风格，仅修改市场相关组件                          │
│  4. 实现可扩展的券商接口架构                                     │
│  5. 集成港A股数据源                                              │
└─────────────────────────────────────────────────────────────────┘
```

#### 重构全景图

```mermaid
mindmap
  root((NOFX 重构))
    技术栈迁移
      Go → Python
      Gin → FastAPI
      GORM → SQLAlchemy
    市场转换
      加密货币 → 港A股
      Binance/OKX → 富途/老虎
      CoinAnk → AkShare
    AI 编排升级
      自研 MCP → LangGraph
      单线程 → 多智能体协作
      无状态 → Checkpointer 持久化
    新增能力
      人机协作审批
      工作流持久化
      断点恢复
```

---

## 2. 现有架构分析

### 2.1 技术栈

| 层级 | 现有技术 | 重构后技术 |
|------|----------|------------|
| 后端 | Go 1.25 + Gin | Python 3.11+ + FastAPI |
| 前端 | React 18 + TypeScript | 保持不变 |
| 数据库 | SQLite / PostgreSQL | PostgreSQL |
| AI 编排 | 自研 MCP 层 | LangGraph |
| 交易所 | Binance, Bybit, OKX 等 | 富途, 老虎, A股券商 |

### 2.2 核心模块职责

```
现有架构 (Go):
├── main.go              # 应用入口
├── api/                 # HTTP API 层 (Gin)
├── auth/                # JWT 认证
├── config/              # 配置管理
├── crypto/              # 加密服务 (API Key 加密)
├── kernel/              # 策略引擎
│   ├── engine.go        # 决策引擎
│   ├── grid_engine.go   # 网格交易引擎
│   └── prompts.go       # AI Prompt 模板
├── manager/             # Trader 生命周期管理
├── market/              # 市场数据获取
│   ├── data.go          # K线、指标
│   └── providers/       # 数据源适配
├── mcp/                 # AI 客户端层
│   ├── client.go        # 统一接口
│   └── provider/        # 各厂商实现
├── provider/            # 外部数据提供商
├── store/               # 数据存储层 (GORM)
├── telegram/            # Telegram Bot
├── trader/              # 交易所连接器
│   ├── types/interface.go  # 统一接口定义
│   ├── binance/
│   ├── bybit/
│   ├── okx/
│   └── hyperliquid/
└── web/                 # React 前端
```

### 2.3 关键接口定义

**交易所接口** (`trader/types/interface.go`):

```go
type Trader interface {
    GetBalance() (map[string]interface{}, error)
    GetPositions() ([]map[string]interface{}, error)
    OpenLong(symbol string, quantity float64, leverage int) (map[string]interface{}, error)
    OpenShort(symbol string, quantity float64, leverage int) (map[string]interface{}, error)
    CloseLong(symbol string, quantity float64) (map[string]interface{}, error)
    CloseShort(symbol string, quantity float64) (map[string]interface{}, error)
    SetLeverage(symbol string, leverage int) error
    GetMarketPrice(symbol string) (float64, error)
    SetStopLoss(symbol string, positionSide string, quantity, stopPrice float64) error
    SetTakeProfit(symbol string, positionSide string, quantity, takeProfitPrice float64) error
    CancelAllOrders(symbol string) error
    GetOpenOrders(symbol string) ([]OpenOrder, error)
}
```

**AI 客户端接口** (`mcp/client.go`):

```go
type AIClient interface {
    SetAPIKey(apiKey string, customURL string, customModel string)
    CallWithMessages(systemPrompt, userPrompt string) (string, error)
    CallWithRequestStream(req *Request, onChunk func(string)) (string, error)
}
```

### 2.4 现有系统架构图

```mermaid
graph TB
    subgraph Frontend["前端 (React)"]
        Dashboard["Dashboard"]
        Strategy["Strategy Studio"]
        Competition["Competition"]
    end

    subgraph Backend["后端 (Go + Gin)"]
        API["API Layer"]
        Auth["JWT Auth"]
        Manager["Trader Manager"]
    end

    subgraph Core["核心引擎"]
        Kernel["决策引擎"]
        MCP["MCP AI 层"]
        Market["市场数据"]
    end

    subgraph Exchanges["加密货币交易所"]
        Binance["Binance"]
        Bybit["Bybit"]
        OKX["OKX"]
        Others["其他 DEX/CEX"]
    end

    subgraph Data["数据存储"]
        SQLite["SQLite/PG"]
    end

    subgraph External["外部服务"]
        Telegram["Telegram Bot"]
        AI["AI 模型"]
    end

    Dashboard --> API
    Strategy --> API
    Competition --> API

    API --> Auth
    API --> Manager
    Manager --> Kernel
    Kernel --> MCP
    Kernel --> Market

    Market --> Exchanges
    Manager --> Exchanges

    MCP --> AI
    API --> Telegram
    Manager --> SQLite
```

### 2.5 模块依赖关系图

```mermaid
graph LR
    subgraph Layer1["入口层"]
        main["main.go"]
    end

    subgraph Layer2["API 层"]
        api["api/"]
        auth["auth/"]
    end

    subgraph Layer3["业务层"]
        manager["manager/"]
        kernel["kernel/"]
        telegram["telegram/"]
    end

    subgraph Layer4["集成层"]
        trader["trader/"]
        mcp["mcp/"]
        market["market/"]
        provider["provider/"]
    end

    subgraph Layer5["基础设施层"]
        store["store/"]
        config["config/"]
        crypto["crypto/"]
    end

    main --> api
    api --> auth
    api --> manager
    manager --> kernel
    manager --> trader
    kernel --> mcp
    kernel --> market
    market --> provider
    manager --> store
    auth --> crypto
    telegram --> api
```

### 2.6 加密货币相关代码清单

| 目录/文件 | 功能 | 处理方式 |
|-----------|------|----------|
| `trader/binance/` | Binance 连接器 | 删除 |
| `trader/bybit/` | Bybit 连接器 | 删除 |
| `trader/okx/` | OKX 连接器 | 删除 |
| `trader/bitget/` | Bitget 连接器 | 删除 |
| `trader/gate/` | Gate.io 连接器 | 删除 |
| `trader/kucoin/` | KuCoin 连接器 | 删除 |
| `trader/hyperliquid/` | Hyperliquid DEX 连接器 | 删除 |
| `trader/aster/` | Aster DEX 连接器 | 删除 |
| `trader/lighter/` | Lighter DEX 连接器 | 删除 |
| `provider/coinank/` | CoinAnk 数据源 | 删除 |
| `provider/hyperliquid/` | Hyperliquid 数据 | 删除 |
| `market/data.go` | 加密货币行情 | 重构适配股票 |
| `kernel/prompts.go` | AI Prompt 模板 | 重构适配股票分析 |
| `crypto/crypto.go` | API Key 加密 | 保留（通用） |

---

## 3. 重构目标与范围

### 3.1 功能范围

#### 保留功能

- [x] AI 多模型支持（DeepSeek, GPT, Claude, Qwen 等）
- [x] 策略可视化配置
- [x] 实时仪表板
- [x] 交易记录与历史
- [x] 风险控制参数
- [x] Telegram Bot 交互
- [x] 用户认证与权限

#### 新增功能

- [ ] 港股交易支持（富途 API）
- [ ] A股交易支持（模拟盘优先）
- [ ] 港A股行情数据集成
- [ ] 涨跌停检测
- [ ] 交易时段限制
- [ ] 融资融券支持（可选）
- [ ] LangGraph 工作流持久化
- [ ] 人机协作审批流程

#### 移除功能

- [ ] 所有加密货币交易所连接器
- [ ] 加密货币特定功能（资金费率、永续合约、杠杆调整）
- [ ] x402 支付集成（可选保留）
- [ ] DEX 相关功能

### 3.2 非功能目标

| 目标 | 指标 |
|------|------|
| 性能 | API 响应 < 500ms (P99) |
| 可用性 | 99.9% 运行时间 |
| 可扩展性 | 支持新增券商无需修改核心代码 |
| 安全性 | API Key 加密存储，通信 TLS |
| 可观测性 | 完整日志、指标、追踪 |

---

## 4. 技术选型与理由

### 4.1 后端框架：Python + FastAPI

**选择理由**:

| 维度 | Go | Python | 优势方 |
|------|-----|--------|--------|
| AI 生态 | 需自行集成 | LangChain, LangGraph 原生支持 | Python ✓ |
| 学习曲线 | 较陡 | 平缓，社区资源丰富 | Python ✓ |
| 开发效率 | 编译型，迭代慢 | 解释型，快速迭代 | Python ✓ |
| 量化库 | 较少 | AkShare, Tushare, pandas | Python ✓ |
| 性能 | 高 | 中等（足够） | Go ✓ |
| 并发 | 原生 goroutine | asyncio | Go ✓ |

**结论**: Python 在 AI、量化领域的生态优势显著，性能对于交易决策场景足够。

### 4.2 智能体编排：LangGraph

**选择理由**:

1. **状态管理**: 内置 Checkpointer 实现执行状态持久化
2. **人机协作**: `interrupt()` 机制支持交易审批
3. **多智能体**: Supervisor 模式支持研究、风控、执行智能体协作
4. **可视化**: LangGraph Studio 支持工作流可视化调试
5. **容错性**: 支持断点恢复、重试机制
6. **工具集成**: 与 LangChain 工具生态无缝集成

### 4.3 券商 API：富途 OpenD

**选择理由**:

| 维度 | 富途 | 老虎 | IBKR |
|------|------|------|------|
| 港股支持 | ✓ | ✓ | ✓ |
| A股支持 | ✓ | ✓ | ✓ |
| Python SDK | 官方支持 | 官方支持 | ib_insync |
| 文档质量 | 优秀 | 良好 | 一般 |
| 模拟盘 | ✓ | ✓ | ✓ |
| 费用 | API 免费 | API 免费 | API 免费 |
| 本地部署 | OpenD 网关 | 云端 | TWS/Gateway |

**结论**: 富途 OpenD API 文档完善，支持港股+A股，推荐作为主要交易通道。

### 4.4 数据源：AkShare + Tushare

| 数据源 | 用途 | 理由 |
|--------|------|------|
| AkShare | 研究/回测数据 | 完全免费，覆盖广 |
| Tushare Pro | 补充数据 | 因子库、宏观数据 |
| 富途 API | 实时行情 | 交易同步 |

---

## 5. LangGraph 架构设计

### 5.0 LangGraph 核心概念

```mermaid
graph TB
    subgraph Core["LangGraph 核心组件"]
        SG["StateGraph<br/>状态图"]
        State["State<br/>状态定义"]
        Nodes["Nodes<br/>节点函数"]
        Edges["Edges<br/>边/路由"]
        Checkpointer["Checkpointer<br/>持久化"]
    end

    subgraph Features["关键特性"]
        HITL["Human-in-the-loop<br/>人机协作"]
        Memory["Memory<br/>记忆系统"]
        Tools["Tools<br/>工具集成"]
        Multi["Multi-Agent<br/>多智能体"]
        Interrupt["interrupt()<br/>中断机制"]
    end

    SG --> State
    SG --> Nodes
    SG --> Edges
    SG --> Checkpointer

    Checkpointer --> HITL
    HITL --> Interrupt
    State --> Memory
    Nodes --> Tools
    SG --> Multi
```

### 5.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Frontend (React)                              │
│  Dashboard │ Strategy Studio │ Competition │ Settings                   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ WebSocket / REST
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API Gateway (FastAPI)                            │
│  /api/traders │ /api/strategies │ /api/positions │ /api/auth            │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
┌───────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ TraderManager │          │ LangGraph       │          │  PostgreSQL     │
│ (Python)      │          │ Orchestrator    │          │  (持久化)       │
└───────┬───────┘          └────────┬────────┘          └─────────────────┘
        │                           │
        ▼                           ▼
┌───────────────┐          ┌─────────────────────────────────────────────┐
│ Broker        │          │              LangGraph Workflow             │
│ Adapters      │          │  ┌─────────┐ ┌─────────┐ ┌─────────────┐   │
│               │          │  │Research │→│ Risk    │→│ Execution   │   │
│ ┌───────────┐ │          │  │ Agent   │ │ Agent   │ │ Agent       │   │
│ │ Futu      │ │          │  └─────────┘ └─────────┘ └─────────────┘   │
│ │ Tiger     │ │          │       ↓           ↓             ↓          │
│ │ Paper     │ │          │  ┌─────────────────────────────────────┐   │
│ └───────────┘ │          │  │         Supervisor Node             │   │
└───────────────┘          │  └─────────────────────────────────────┘   │
                            │  ┌─────────────────────────────────────┐   │
                            │  │         Human-in-the-loop           │   │
                            │  │         (交易审批)                   │   │
                            │  └─────────────────────────────────────┘   │
                            └─────────────────────────────────────────────┘
```

### 5.2 核心状态定义

#### 状态流转图

```mermaid
stateDiagram-v2
    [*] --> MarketData: 启动交易

    MarketData --> Research: 获取行情
    note right of MarketData
        symbol, market_data
        technical_indicators
    end note

    Research --> Risk: 分析完成
    note right of Research
        research_analysis
        news_sentiment
    end note

    Risk --> Decision: 风险评估
    note right of Risk
        risk_level
        risk_score
        requires_approval
    end note

    Decision --> Approval: 需要审批?
    Decision --> Execute: 自动执行

    Approval --> Approved: 用户批准
    Approval --> Rejected: 用户拒绝

    Approved --> Execute
    Rejected --> [*]: 取消

    Execute --> Success: 执行成功
    Execute --> Failed: 执行失败
    Execute --> Pending: 非交易时段

    Success --> [*]
    Failed --> [*]
    Pending --> [*]: 等待开盘
```

#### 数据流时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as API Gateway
    participant Graph as LangGraph
    participant Market as Market Data
    participant Research as Research Agent
    participant Risk as Risk Agent
    participant Broker as Broker Adapter

    User->>API: 发起交易请求
    API->>Graph: 创建交易会话

    Graph->>Market: 获取市场数据
    Market-->>Graph: 行情数据

    Graph->>Research: 执行研究分析
    Research->>Research: AI 分析
    Research-->>Graph: 分析报告

    Graph->>Risk: 风险评估
    Risk-->>Graph: 风险结果

    alt 需要审批
        Graph-->>User: 请求审批
        User->>Graph: 确认/拒绝
    end

    Graph->>Broker: 执行交易
    Broker-->>Graph: 订单结果
    Graph-->>API: 执行结果
    API-->>User: 返回结果
```

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages
from datetime import datetime

class TradingState(TypedDict):
    """交易工作流状态"""

    # 会话信息
    messages: Annotated[list, add_messages]
    thread_id: str
    created_at: datetime

    # 用户配置
    user_id: str
    trader_id: str
    strategy_id: str

    # 市场数据
    symbol: str                    # 股票代码，如 "HK.00700"
    market: str                    # "HK" | "SH" | "SZ"
    market_data: MarketData        # 行情数据
    technical_indicators: dict     # 技术指标
    fundamental_data: dict         # 基本面数据
    news_sentiment: dict           # 新闻情绪

    # 分析结果
    research_analysis: str | None
    risk_assessment: RiskAssessment | None

    # 决策
    decision: str | None           # "buy" | "sell" | "hold"
    position_size: float | None    # 仓位大小
    price_limit: float | None      # 限价

    # 执行状态
    order_id: str | None
    execution_status: str          # "pending" | "executed" | "failed" | "cancelled"
    execution_result: dict | None

    # 审批
    requires_approval: bool
    approval_status: str | None    # "pending" | "approved" | "rejected"

    # 错误处理
    errors: list[str]
    retry_count: int


class MarketData(TypedDict):
    """行情数据"""
    symbol: str
    name: str
    price: float
    open: float
    high: float
    low: float
    volume: int
    turnover: float
    change_pct: float
    timestamp: datetime


class RiskAssessment(TypedDict):
    """风险评估"""
    risk_level: str           # "low" | "medium" | "high"
    risk_score: float         # 0-100
    max_position_size: float
    stop_loss_price: float | None
    take_profit_price: float | None
    warnings: list[str]
```

### 5.3 节点设计

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.postgres import PostgresSaver

# ============== 节点函数 ==============

async def market_data_node(state: TradingState) -> dict:
    """市场数据获取节点"""
    from adapters.futu import FutuAdapter

    adapter = FutuAdapter()
    data = await adapter.get_market_data(state["symbol"])

    return {
        "market_data": data,
        "technical_indicators": calculate_indicators(data)
    }


async def research_node(state: TradingState) -> dict:
    """研究分析节点 - AI 分析"""
    from langchain_openai import ChatOpenAI

    model = ChatOpenAI(model="deepseek-chat")

    prompt = f"""
    分析股票 {state['symbol']} ({state['market_data']['name']}):

    当前价格: {state['market_data']['price']}
    涨跌幅: {state['market_data']['change_pct']}%
    成交量: {state['market_data']['volume']}

    技术指标:
    - MA5: {state['technical_indicators']['ma5']}
    - MA20: {state['technical_indicators']['ma20']}
    - RSI: {state['technical_indicators']['rsi']}
    - MACD: {state['technical_indicators']['macd']}

    请给出投资建议，包括:
    1. 市场分析
    2. 技术面分析
    3. 操作建议 (buy/sell/hold)
    4. 建议仓位
    5. 风险提示
    """

    response = await model.ainvoke(prompt)

    return {"research_analysis": response.content}


async def risk_node(state: TradingState) -> dict:
    """风险评估节点"""

    # 计算风险分数
    risk_score = calculate_risk_score(
        state["market_data"],
        state["technical_indicators"],
        state.get("research_analysis", "")
    )

    # 确定风险等级
    if risk_score > 70:
        risk_level = "high"
        requires_approval = True
    elif risk_score > 40:
        risk_level = "medium"
        requires_approval = True
    else:
        risk_level = "low"
        requires_approval = False

    return {
        "risk_assessment": {
            "risk_level": risk_level,
            "risk_score": risk_score,
            "max_position_size": calculate_max_position(state, risk_score),
            "warnings": get_risk_warnings(state)
        },
        "requires_approval": requires_approval
    }


async def decision_node(state: TradingState) -> dict:
    """决策节点"""

    if state["requires_approval"]:
        # 人机协作：请求审批
        decision = interrupt({
            "type": "trade_approval",
            "symbol": state["symbol"],
            "market_data": state["market_data"],
            "research_analysis": state["research_analysis"],
            "risk_assessment": state["risk_assessment"],
            "message": "高风险交易需要确认，是否继续？"
        })

        if decision.get("action") != "approve":
            return {
                "decision": "cancelled",
                "execution_status": "cancelled"
            }

    # 解析研究分析，提取决策
    decision = parse_decision(state["research_analysis"])
    position_size = min(
        decision["position_size"],
        state["risk_assessment"]["max_position_size"]
    )

    return {
        "decision": decision["action"],
        "position_size": position_size,
        "price_limit": decision.get("price_limit")
    }


async def execution_node(state: TradingState) -> dict:
    """交易执行节点"""

    if state["decision"] == "hold":
        return {"execution_status": "skipped"}

    # 检查交易时段
    if not is_trading_hours(state["market"]):
        return {
            "execution_status": "pending",
            "errors": ["非交易时段，订单将在开盘后执行"]
        }

    # 检查涨跌停
    if is_limit_up_or_down(state["symbol"], state["market_data"]):
        return {
            "execution_status": "failed",
            "errors": ["股票涨跌停，无法交易"]
        }

    # 执行交易
    try:
        from adapters.futu import FutuAdapter
        adapter = FutuAdapter()

        if state["decision"] == "buy":
            result = await adapter.place_order(
                symbol=state["symbol"],
                side="buy",
                quantity=state["position_size"],
                price=state.get("price_limit")
            )
        else:  # sell
            result = await adapter.place_order(
                symbol=state["symbol"],
                side="sell",
                quantity=state["position_size"],
                price=state.get("price_limit")
            )

        return {
            "order_id": result["order_id"],
            "execution_status": "executed",
            "execution_result": result
        }

    except Exception as e:
        return {
            "execution_status": "failed",
            "errors": [str(e)]
        }


# ============== 路由函数 ==============

def route_by_decision(state: TradingState) -> str:
    """根据决策路由"""
    if state.get("decision") == "cancelled":
        return END
    if state.get("decision") == "hold":
        return END
    return "execute"


# ============== 构建图 ==============

def build_trading_graph():
    graph = StateGraph(TradingState)

    # 添加节点
    graph.add_node("market_data", market_data_node)
    graph.add_node("research", research_node)
    graph.add_node("risk", risk_node)
    graph.add_node("decide", decision_node)
    graph.add_node("execute", execution_node)

    # 定义边
    graph.add_edge(START, "market_data")
    graph.add_edge("market_data", "research")
    graph.add_edge("research", "risk")
    graph.add_edge("risk", "decide")
    graph.add_conditional_edges("decide", route_by_decision)
    graph.add_edge("execute", END)

    return graph


# ============== 编译并配置持久化 ==============

async def create_app():
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/nofx")
    checkpointer = PostgresSaver(engine)

    graph = build_trading_graph()
    app = graph.compile(
        checkpointer=checkpointer,
        durability_mode="sync"
    )

    return app
```

### 5.4 多智能体协作架构

#### Supervisor 模式架构图

```mermaid
graph TB
    subgraph Main["主工作流"]
        START((START))
        Supervisor["Supervisor<br/>调度节点"]
        END((END))
    end

    subgraph ResearchAgent["研究智能体"]
        R1["获取新闻"]
        R2["情绪分析"]
        R3["技术分析"]
        R4["生成报告"]
    end

    subgraph RiskAgent["风控智能体"]
        K1["仓位检查"]
        K2["风险指标"]
        K3["最终检查"]
    end

    subgraph ExecAgent["执行智能体"]
        E1["订单构建"]
        E2["交易执行"]
        E3["状态确认"]
    end

    START --> Supervisor

    Supervisor -->|"research"| R1
    R1 --> R2
    R1 --> R3
    R2 --> R4
    R3 --> R4
    R4 --> Supervisor

    Supervisor -->|"risk"| K1
    K1 --> K3
    K2 --> K3
    Supervisor -->|"risk"| K2
    K3 --> Supervisor

    Supervisor -->|"execute"| E1
    E1 --> E2
    E2 --> E3
    E3 --> Supervisor

    Supervisor -->|"done"| END
```

#### 节点协作序列图

```mermaid
sequenceDiagram
    participant S as Supervisor
    participant R as Research Agent
    participant K as Risk Agent
    participant E as Execute Agent
    participant DB as Checkpointer

    S->>DB: 保存状态
    S->>R: 调度研究任务

    R->>R: 获取新闻 (并行)
    R->>R: 技术分析 (并行)
    R->>DB: 保存中间状态
    R-->>S: 返回分析结果

    S->>DB: 更新状态
    S->>K: 调度风控任务

    K->>K: 仓位检查 (并行)
    K->>K: 风险指标 (并行)
    K->>DB: 保存检查结果
    K-->>S: 返回风控结果

    alt 风控通过
        S->>E: 调度执行任务
        E->>E: 下单执行
        E->>DB: 保存执行结果
        E-->>S: 返回订单信息
    else 风控拒绝
        S-->>S: 结束流程
    end
```

```python
# ============== 子图：研究智能体 ==============

def create_research_agent():
    """研究智能体子图"""

    class ResearchState(TypedDict):
        symbol: str
        market_data: dict
        news: list
        analysis: str

    graph = StateGraph(ResearchState)

    async def fetch_news(state):
        # 获取新闻
        return {"news": await get_stock_news(state["symbol"])}

    async def analyze_news(state):
        # 分析新闻情绪
        return {"sentiment": analyze_sentiment(state["news"])}

    async def technical_analysis(state):
        # 技术分析
        return {"technical": analyze_technicals(state["market_data"])}

    async def generate_report(state):
        # 生成研究报告
        return {"analysis": generate_research_report(state)}

    graph.add_node("fetch_news", fetch_news)
    graph.add_node("analyze_news", analyze_news)
    graph.add_node("technical", technical_analysis)
    graph.add_node("report", generate_report)

    graph.add_edge(START, "fetch_news")
    graph.add_edge("fetch_news", "analyze_news")
    graph.add_edge(START, "technical")  # 并行
    graph.add_edge("analyze_news", "report")
    graph.add_edge("technical", "report")
    graph.add_edge("report", END)

    return graph.compile()


# ============== 子图：风控智能体 ==============

def create_risk_agent():
    """风控智能体子图"""

    class RiskState(TypedDict):
        symbol: str
        position_size: float
        account_balance: float
        risk_metrics: dict
        passed: bool

    graph = StateGraph(RiskState)

    async def check_position_limit(state):
        # 检查仓位限制
        passed = state["position_size"] <= state["account_balance"] * 0.3
        return {"position_check": passed}

    async def check_risk_metrics(state):
        # 检查风险指标
        metrics = calculate_risk_metrics(state["symbol"])
        return {"risk_metrics": metrics}

    async def final_check(state):
        # 最终检查
        passed = (
            state.get("position_check", True) and
            state["risk_metrics"]["var"] < 0.05
        )
        return {"passed": passed}

    graph.add_node("position_check", check_position_limit)
    graph.add_node("risk_metrics", check_risk_metrics)
    graph.add_node("final_check", final_check)

    graph.add_edge(START, "position_check")
    graph.add_edge(START, "risk_metrics")  # 并行
    graph.add_edge("position_check", "final_check")
    graph.add_edge("risk_metrics", "final_check")
    graph.add_edge("final_check", END)

    return graph.compile()


# ============== 主图：Supervisor 模式 ==============

def create_main_graph():
    """主图：Supervisor 协调多个智能体"""

    class MainState(TypedDict):
        symbol: str
        market_data: dict
        research_result: dict
        risk_result: dict
        final_decision: str
        next_agent: str

    async def supervisor(state: MainState) -> Command:
        """Supervisor：决定下一个执行的智能体"""

        if not state.get("research_done"):
            return Command(goto="research")

        if not state.get("risk_done"):
            return Command(goto="risk")

        # 根据结果做最终决策
        if state["risk_result"]["passed"]:
            return Command(goto="execute")
        else:
            return Command(goto=END)

    graph = StateGraph(MainState)

    graph.add_node("supervisor", supervisor)
    graph.add_node("research", create_research_agent())
    graph.add_node("risk", create_risk_agent())
    graph.add_node("execute", execution_node)

    graph.add_edge(START, "supervisor")
    for agent in ["research", "risk", "execute"]:
        graph.add_edge(agent, "supervisor")

    return graph.compile(checkpointer=PostgresSaver(...))
```

---

## 6. 港A股适配方案

### 6.0 市场概览

```mermaid
graph LR
    subgraph HK["港股市场"]
        HK1["恒生指数"]
        HK2["国企指数"]
        HK3["科技指数"]
    end

    subgraph SH["沪市A股"]
        SH1["上证指数"]
        SH2["科创板"]
    end

    subgraph SZ["深市A股"]
        SZ1["深证成指"]
        SZ2["创业板"]
    end

    subgraph Connect["互联互通"]
        HSI["港股通"]
        LZ["陆股通"]
    end

    HK --> HSI
    SH --> LZ
    SZ --> LZ
    HSI --> SH
    HSI --> SZ
```

### 6.1 市场规则适配

#### 交易时间对比图

```mermaid
gantt
    title 交易时段对比 (北京时间)
    dateFormat HH:mm
    axisFormat %H:%M

    section 港股
    开盘集合竞价    :crit, hk1, 09:20, 10m
    上午连续竞价    :active, hk2, 09:30, 2h30m
    午间休市        :hk3, 12:00, 1h
    下午连续竞价    :active, hk4, 13:00, 3h
    收市集合竞价    :crit, hk5, 16:00, 10m

    section A股
    开盘集合竞价    :crit, a1, 09:15, 10m
    上午连续竞价    :active, a2, 09:30, 2h
    午间休市        :a3, 11:30, 1h30m
    下午连续竞价    :active, a4, 13:00, 2h
```

#### 交易时间

| 市场 | 交易时段 | 午间休市 | 集合竞价 |
|------|----------|----------|----------|
| 港股 | 09:30-12:00, 13:00-16:00 | 12:00-13:00 | 16:00-16:10 (收市竞价) |
| A股 | 09:30-11:30, 13:00-15:00 | 11:30-13:00 | 09:15-09:25 (开盘) |

```python
def is_trading_hours(market: str, dt: datetime = None) -> bool:
    """检查是否在交易时段"""
    dt = dt or datetime.now()
    time = dt.time()

    if market == "HK":
        morning = time >= time(9, 30) and time <= time(12, 0)
        afternoon = time >= time(13, 0) and time <= time(16, 10)
        return morning or afternoon

    elif market in ("SH", "SZ"):
        morning = time >= time(9, 30) and time <= time(11, 30)
        afternoon = time >= time(13, 0) and time <= time(15, 0)
        return morning or afternoon

    return False
```

#### 涨跌停规则

```mermaid
graph TB
    subgraph HK_Rules["港股规则"]
        HK1["无涨跌停限制"]
        HK2["市场波动调节机制 VCM"]
        HK3["适用: 恒生指数成分股"]
        HK4["触发: ±10% 暂停5分钟"]
    end

    subgraph A_Main["A股主板"]
        M1["涨跌幅: ±10%"]
        M2["ST股票: ±5%"]
        M3["新股首日: ±44%"]
    end

    subgraph A_Star["科创板/创业板"]
        S1["涨跌幅: ±20%"]
        S2["新股前5日: 无限制"]
        S3["盘中停牌: ±30%/±60%"]
    end

    HK_Rules --> Check{"涨跌停检查"}
    A_Main --> Check
    A_Star --> Check

    Check -->|"涨停"| Reject["拒绝买入"]
    Check -->|"跌停"| Reject2["拒绝卖出"]
    Check -->|"正常"| Allow["允许交易"]
```

| 市场 | 涨跌幅限制 | 特殊情况 |
|------|------------|----------|
| 港股 | 无限制 | 有市场波动调节机制(VCM) |
| A股主板 | 10% | ST 股票 5% |
| A股科创板 | 20% | 首日不设涨跌幅 |
| A股创业板 | 20% | 首日不设涨跌幅 |

```python
def is_limit_up_or_down(symbol: str, market_data: dict) -> bool:
    """检查是否涨跌停"""
    market = symbol.split(".")[0]
    change_pct = market_data["change_pct"]

    if market == "HK":
        # 港股无涨跌停，检查 VCM
        return False

    # A股涨跌停检查
    limit = get_limit_for_symbol(symbol)

    # 涨停
    if change_pct >= limit - 0.01:
        return True
    # 跌停
    if change_pct <= -limit + 0.01:
        return True

    return False
```

### 6.2 数据模型适配

#### 股票标识

```python
# 加密货币: BTCUSDT, ETHUSDT
# 港股: HK.00700 (腾讯), HK.00941 (中国移动)
# A股: SH.600519 (贵州茅台), SZ.000001 (平安银行)

class StockSymbol:
    """股票代码标准化"""

    @staticmethod
    def parse(symbol: str) -> tuple[str, str]:
        """解析股票代码，返回 (市场, 代码)"""
        if "." in symbol:
            market, code = symbol.split(".")
            return market, code

        # 纯代码推断市场
        if symbol.startswith("6"):
            return "SH", symbol
        elif symbol.startswith(("0", "3")):
            return "SZ", symbol
        else:
            raise ValueError(f"Unknown symbol format: {symbol}")

    @staticmethod
    def format(market: str, code: str) -> str:
        """格式化为标准代码"""
        return f"{market}.{code}"
```

#### 订单模型

```python
from pydantic import BaseModel
from enum import Enum
from datetime import datetime

class OrderSide(str, Enum):
    BUY = "buy"
    SELL = "sell"

class OrderType(str, Enum):
    MARKET = "market"      # 市价单
    LIMIT = "limit"        # 限价单
    STOP = "stop"          # 止损单
    STOP_LIMIT = "stop_limit"  # 止损限价单

class OrderStatus(str, Enum):
    PENDING = "pending"
    SUBMITTED = "submitted"
    FILLED = "filled"
    CANCELLED = "cancelled"
    REJECTED = "rejected"

class StockOrder(BaseModel):
    """股票订单模型"""

    # 基本信息
    order_id: str | None = None
    symbol: str
    name: str | None = None

    # 订单参数
    side: OrderSide
    order_type: OrderType
    quantity: int              # 股数（必须是 100 的倍数）
    price: float | None = None # 限价单价格

    # 状态
    status: OrderStatus = OrderStatus.PENDING
    filled_quantity: int = 0
    filled_price: float | None = None

    # 时间
    created_at: datetime
    updated_at: datetime | None = None

    # 费用
    commission: float | None = None
    stamp_duty: float | None = None  # 印花税（仅卖出）

    class Config:
        use_enum_values = True
```

### 6.3 券商接口适配

#### 适配器模式架构

```mermaid
classDiagram
    class BrokerAdapter {
        <<interface>>
        +connect() None
        +disconnect() None
        +get_market_data(symbol) MarketData
        +get_positions() List~Position~
        +get_balance() AccountBalance
        +place_order(order) OrderResult
        +cancel_order(order_id) bool
        +get_klines(symbol, start, end) List~Kline~
    }

    class FutuAdapter {
        -host: str
        -port: int
        -quote_ctx: OpenQuoteContext
        -trade_ctx: OpenHKTradeContext
        +connect() None
        +get_market_data(symbol) MarketData
        +place_order(order) OrderResult
    }

    class TigerAdapter {
        -client: TigerClient
        +connect() None
        +get_market_data(symbol) MarketData
        +place_order(order) OrderResult
    }

    class PaperAdapter {
        -positions: Dict
        -balance: float
        +connect() None
        +get_market_data(symbol) MarketData
        +place_order(order) OrderResult
    }

    BrokerAdapter <|.. FutuAdapter
    BrokerAdapter <|.. TigerAdapter
    BrokerAdapter <|.. PaperAdapter
```

#### 券商能力对比

```mermaid
graph LR
    subgraph Futu["富途 OpenD"]
        F1["✓ 港股交易"]
        F2["✓ A股交易"]
        F3["✓ 美股交易"]
        F4["✓ 实时行情"]
        F5["✓ 模拟盘"]
    end

    subgraph Tiger["老虎证券"]
        T1["✓ 港股交易"]
        T2["✓ 美股交易"]
        T3["✓ 实时行情"]
        T4["✓ 模拟盘"]
        T5["✗ A股交易"]
    end

    subgraph Paper["模拟券商"]
        P1["✓ 模拟交易"]
        P2["✓ 回测支持"]
        P3["✗ 实盘交易"]
    end

    Futu --> Select["券商选择"]
    Tiger --> Select
    Paper --> Select
```

#### 统一接口定义

```python
from abc import ABC, abstractmethod
from typing import Protocol

class BrokerAdapter(Protocol):
    """券商适配器协议"""

    # 连接管理
    async def connect(self) -> None: ...
    async def disconnect(self) -> None: ...
    async def is_connected(self) -> bool: ...

    # 行情接口
    async def get_market_data(self, symbol: str) -> MarketData: ...
    async def subscribe_quotes(self, symbols: list[str], callback) -> None: ...

    # 交易接口
    async def get_positions(self) -> list[Position]: ...
    async def get_balance(self) -> AccountBalance: ...
    async def place_order(self, order: StockOrder) -> OrderResult: ...
    async def cancel_order(self, order_id: str) -> bool: ...
    async def get_order_status(self, order_id: str) -> OrderStatus: ...

    # 历史数据
    async def get_klines(
        self,
        symbol: str,
        start: datetime,
        end: datetime,
        interval: str = "1d"
    ) -> list[Kline]: ...
```

#### 富途适配器实现

```python
from futu import OpenQuoteContext, OpenHKTradeContext, RET_OK

class FutuAdapter:
    """富途 OpenD 适配器"""

    def __init__(self, host: str = "127.0.0.1", port: int = 11111):
        self.host = host
        self.port = port
        self.quote_ctx = None
        self.trade_ctx = None

    async def connect(self) -> None:
        """连接 OpenD"""
        self.quote_ctx = OpenQuoteContext(self.host, self.port)
        self.trade_ctx = OpenHKTradeContext(self.host, self.port)

    async def get_market_data(self, symbol: str) -> MarketData:
        """获取实时行情"""
        ret, data = self.quote_ctx.get_market_snapshot([symbol])

        if ret != RET_OK:
            raise Exception(f"获取行情失败: {data}")

        row = data.iloc[0]

        return MarketData(
            symbol=symbol,
            name=row["stock_name"],
            price=float(row["last_price"]),
            open=float(row["open_price"]),
            high=float(row["high_price"]),
            low=float(row["low_price"]),
            volume=int(row["volume"]),
            turnover=float(row["turnover"]),
            change_pct=float(row["change_rate"]),
            timestamp=datetime.now()
        )

    async def place_order(self, order: StockOrder) -> OrderResult:
        """下单"""
        ret, data = self.trade_ctx.place_order(
            price=order.price or 0,  # 0 表示市价单
            qty=order.quantity,
            code=order.symbol,
            trd_side=TrdSide.BUY if order.side == OrderSide.BUY else TrdSide.SELL,
            order_type=OrderType.MARKET if order.order_type == "market" else OrderType.NORMAL
        )

        if ret != RET_OK:
            raise Exception(f"下单失败: {data}")

        return OrderResult(
            order_id=str(data.iloc[0]["order_id"]),
            status=OrderStatus.SUBMITTED
        )
```

### 6.4 数据源集成

```python
import akshare as ak
import tushare as ts

class MarketDataProvider:
    """市场数据提供者"""

    async def get_stock_list(self, market: str) -> list[str]:
        """获取股票列表"""

        if market == "HK":
            df = ak.stock_hk_spot()
            return df["代码"].tolist()

        elif market in ("SH", "SZ"):
            df = ak.stock_zh_a_spot_em()
            return df["代码"].tolist()

    async def get_history_klines(
        self,
        symbol: str,
        start: str,
        end: str,
        adjust: str = "qfq"  # 前复权
    ) -> pd.DataFrame:
        """获取历史 K 线"""

        market, code = StockSymbol.parse(symbol)

        if market == "HK":
            df = ak.stock_hk_daily(symbol=code, adjust=adjust)
        else:
            df = ak.stock_zh_a_hist(
                symbol=code,
                period="daily",
                start_date=start,
                end_date=end,
                adjust=adjust
            )

        return df

    async def get_financial_data(self, symbol: str) -> dict:
        """获取财务数据"""

        market, code = StockSymbol.parse(symbol)

        if market in ("SH", "SZ"):
            # A股财务数据
            df = ak.stock_financial_analysis_indicator(symbol=code)
            return df.to_dict("records")

        return {}
```

---

## 7. 模块迁移计划

### 7.1 迁移映射表

| Go 模块 | Python 模块 | 迁移策略 |
|---------|-------------|----------|
| `api/` | `api/` | 重写，FastAPI |
| `auth/` | `auth/` | 重写，JWT 保持 |
| `config/` | `config/` | 重写，pydantic-settings |
| `crypto/` | `security/` | 重写，加密逻辑保留 |
| `kernel/` | `agents/` | 使用 LangGraph 重构 |
| `manager/` | `manager/` | 重写，异步管理 |
| `market/` | `data/` | 重写，AkShare 集成 |
| `mcp/` | `llm/` | 重写，LangChain 集成 |
| `provider/` | `providers/` | 删除加密货币，添加股票数据源 |
| `store/` | `models/` | 重写，SQLAlchemy |
| `telegram/` | `telegram/` | 重写，python-telegram-bot |
| `trader/` | `brokers/` | 删除加密货币，添加富途/老虎 |
| `web/` | 保持 | 仅修改市场相关组件 |

### 7.2 新目录结构

```
nofx-v2/
├── pyproject.toml
├── poetry.lock
├── .env.example
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── src/
│   └── nofx/
│       ├── __init__.py
│       ├── main.py                 # FastAPI 入口
│       ├── api/
│       │   ├── __init__.py
│       │   ├── router.py           # API 路由
│       │   ├── traders.py          # Trader API
│       │   ├── strategies.py       # Strategy API
│       │   └── auth.py             # 认证 API
│       ├── agents/
│       │   ├── __init__.py
│       │   ├── state.py            # LangGraph 状态定义
│       │   ├── nodes/
│       │   │   ├── market_data.py
│       │   │   ├── research.py
│       │   │   ├── risk.py
│       │   │   └── execution.py
│       │   ├── graphs/
│       │   │   ├── trading.py      # 交易工作流
│       │   │   ├── research.py     # 研究工作流
│       │   │   └── competition.py  # 竞赛工作流
│       │   └── tools/
│       │       ├── market.py       # 市场数据工具
│       │       └── trading.py      # 交易工具
│       ├── brokers/
│       │   ├── __init__.py
│       │   ├── base.py             # 基类定义
│       │   ├── futu.py             # 富途适配器
│       │   ├── tiger.py            # 老虎适配器
│       │   └── paper.py            # 模拟券商
│       ├── data/
│       │   ├── __init__.py
│       │   ├── providers/
│       │   │   ├── akshare.py
│       │   │   └── tushare.py
│       │   └── cache.py            # 数据缓存
│       ├── llm/
│       │   ├── __init__.py
│       │   ├── providers/
│       │   │   ├── openai.py
│       │   │   ├── deepseek.py
│       │   │   ├── claude.py
│       │   │   └── qwen.py
│       │   └── prompts/
│       │       ├── research.py
│       │       └── trading.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── user.py
│       │   ├── trader.py
│       │   ├── strategy.py
│       │   ├── position.py
│       │   └── order.py
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── jwt.py
│       │   └── dependencies.py
│       ├── security/
│       │   ├── __init__.py
│       │   └── encryption.py
│       ├── telegram/
│       │   ├── __init__.py
│       │   ├── bot.py
│       │   └── handlers.py
│       ├── config/
│       │   ├── __init__.py
│       │   └── settings.py
│       └── utils/
│           ├── __init__.py
│           ├── trading_hours.py
│           └── risk.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── web/                            # React 前端（保持）
```

### 7.3 数据库迁移

```sql
-- 新增市场字段
ALTER TABLE traders ADD COLUMN market VARCHAR(10) DEFAULT 'HK';
ALTER TABLE positions ADD COLUMN market VARCHAR(10);

-- 修改符号格式
-- 旧: BTCUSDT
-- 新: HK.00700

-- 新增订单表
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    trader_id VARCHAR(36) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    market VARCHAR(10) NOT NULL,
    side VARCHAR(10) NOT NULL,
    order_type VARCHAR(20) NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(18, 4),
    status VARCHAR(20) NOT NULL,
    order_id VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    FOREIGN KEY (trader_id) REFERENCES traders(id)
);

-- 新增风险配置
CREATE TABLE risk_configs (
    id SERIAL PRIMARY KEY,
    trader_id VARCHAR(36) NOT NULL,
    max_position_pct DECIMAL(5, 2) DEFAULT 30.00,
    max_single_trade_pct DECIMAL(5, 2) DEFAULT 10.00,
    stop_loss_pct DECIMAL(5, 2) DEFAULT 5.00,
    take_profit_pct DECIMAL(5, 2) DEFAULT 15.00,
    requires_approval BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (trader_id) REFERENCES traders(id)
);
```

---

## 8. 前端重构方案

### 8.1 需要修改的组件

| 组件路径 | 修改内容 |
|----------|----------|
| `web/src/pages/AITradersPage.tsx` | 市场选择（港股/A股） |
| `web/src/components/trader/` | 交易对选择 → 股票代码选择 |
| `web/src/components/charts/` | K线图适配股票数据 |
| `web/src/components/strategy/` | 策略参数（移除杠杆、资金费率） |
| `web/src/types/` | 类型定义适配 |

### 8.2 新增组件

```
web/src/components/
├── stock/
│   ├── StockSearch.tsx       # 股票搜索
│   ├── StockQuote.tsx        # 股票报价卡片
│   ├── MarketSelector.tsx    # 市场选择器
│   └── TradingHoursBadge.tsx # 交易时段徽章
```

### 8.3 API 适配

```typescript
// web/src/lib/api/trading.ts

export interface StockSymbol {
  symbol: string;      // "HK.00700"
  name: string;        // "腾讯控股"
  market: "HK" | "SH" | "SZ";
  price: number;
  changePct: number;
}

export interface StockOrder {
  symbol: string;
  side: "buy" | "sell";
  quantity: number;    // 股数，100 的倍数
  price?: number;      // 限价
  orderType: "market" | "limit";
}

// 获取股票列表
export async function getStockList(market: string): Promise<StockSymbol[]>

// 搜索股票
export async function searchStocks(query: string): Promise<StockSymbol[]>

// 获取股票行情
export async function getStockQuote(symbol: string): Promise<MarketData>

// 下单
export async function placeOrder(order: StockOrder): Promise<OrderResult>
```

---

## 9. 实施路线图

### 9.0 项目总览

```mermaid
timeline
    title NOFX 重构项目时间线
    section Phase 0
        准备阶段 : 环境搭建
        : LangGraph PoC
        : 架构设计确认
    section Phase 1
        核心重构 : FastAPI 骨架
        : LangGraph 工作流
        : 券商集成
        : 数据源集成
    section Phase 2
        前端适配 : 组件修改
        : API 联调
    section Phase 3
        测试优化 : 单元测试
        : 集成测试
        : 性能测试
    section Phase 4
        部署上线 : 生产部署
        : 监控告警
        : 文档完善
```

### 9.1 阶段规划

```mermaid
gantt
    title NOFX 重构甘特图
    dateFormat  YYYY-MM-DD

    section Phase 0: 准备
    环境搭建           :p0a, 2026-03-20, 7d
    LangGraph PoC      :p0b, after p0a, 7d
    架构设计确认       :milestone, p0m, after p0b, 0d

    section Phase 1: 核心
    FastAPI 项目骨架   :p1a, after p0b, 5d
    数据库模型         :p1b, after p1a, 3d
    认证系统           :p1c, after p1b, 2d
    LangGraph 状态定义 :p1d, after p1c, 3d
    核心节点实现       :p1e, after p1d, 5d
    富途适配器         :p1f, after p1e, 5d
    数据源集成         :p1g, after p1f, 5d
    核心功能完成       :milestone, p1m, after p1g, 0d

    section Phase 2: 前端
    组件修改           :p2a, after p1m, 5d
    API 联调           :p2b, after p2a, 5d
    前端适配完成       :milestone, p2m, after p2b, 0d

    section Phase 3: 测试
    单元测试           :p3a, after p2m, 5d
    集成测试           :p3b, after p3a, 3d
    性能测试           :p3c, after p3b, 2d
    测试通过           :milestone, p3m, after p3c, 0d

    section Phase 4: 部署
    生产环境部署       :p4a, after p3m, 5d
    监控告警           :p4b, after p4a, 3d
    文档完善           :p4c, after p4b, 2d
    生产上线           :milestone, p4m, after p4c, 0d
```

#### 阶段规划

```
Phase 0: 准备阶段 (Week 1-2)
├── 环境搭建
│   ├── Python 开发环境
│   ├── PostgreSQL 数据库
│   └── 富途 OpenD 部署
├── LangGraph 学习验证
│   └── PoC: 单股票分析流程
└── 架构设计确认
    └── 技术评审会议

Phase 1: 核心重构 (Week 3-6)
├── Week 3: 基础框架
│   ├── FastAPI 项目骨架
│   ├── 数据库模型
│   └── 认证系统
├── Week 4: LangGraph 工作流
│   ├── 状态定义
│   ├── 核心节点实现
│   └── 工作流编译
├── Week 5: 券商集成
│   ├── 富途适配器
│   ├── 模拟券商
│   └── 交易接口
└── Week 6: 数据源集成
    ├── AkShare 集成
    ├── K线数据
    └── 技术指标

Phase 2: 前端适配 (Week 7-8)
├── Week 7: 组件修改
│   ├── 市场选择
│   ├── 股票搜索
│   └── 订单表单
└── Week 8: 测试集成
    ├── API 联调
    └── 端到端测试

Phase 3: 测试与优化 (Week 9-10)
├── 单元测试
├── 集成测试
├── 性能测试
└── Bug 修复

Phase 4: 部署上线 (Week 11-12)
├── 生产环境部署
├── 监控告警
├── 文档完善
└── 用户培训
```

### 9.2 里程碑

#### 里程碑依赖图

```mermaid
graph LR
    M0["M0: 项目启动"]

    M1["M1: 架构设计完成<br/>Week 2"]
    M1_Deliver["交付物:<br/>- 架构设计文档<br/>- PoC 演示"]

    M2["M2: 核心功能完成<br/>Week 6"]
    M2_Deliver["交付物:<br/>- Python 后端<br/>- LangGraph 工作流<br/>- 富途集成"]

    M3["M3: 前端适配完成<br/>Week 8"]
    M3_Deliver["交付物:<br/>- 前后端集成<br/>- UI 组件更新"]

    M4["M4: 测试通过<br/>Week 10"]
    M4_Deliver["交付物:<br/>- 测试报告<br/>- 性能报告"]

    M5["M5: 生产上线<br/>Week 12"]
    M5_Deliver["交付物:<br/>- 部署文档<br/>- 运维手册"]

    M0 --> M1
    M1 --> M1_Deliver
    M1 --> M2
    M2 --> M2_Deliver
    M2 --> M3
    M3 --> M3_Deliver
    M3 --> M4
    M4 --> M4_Deliver
    M4 --> M5
    M5 --> M5_Deliver
```

| 里程碑 | 日期 | 交付物 |
|--------|------|--------|
| M1: 架构设计完成 | Week 2 | 架构设计文档、PoC 演示 |
| M2: 核心功能完成 | Week 6 | 可运行的 Python 后端 |
| M3: 前端适配完成 | Week 8 | 完整的前后端集成 |
| M4: 测试通过 | Week 10 | 测试报告、性能报告 |
| M5: 生产上线 | Week 12 | 部署文档、运维手册 |

### 9.3 风险缓解措施

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| 富途 API 不稳定 | 高 | 低 | 备选老虎 API，模拟券商测试 |
| LangGraph 学习曲线 | 中 | 中 | 提前 PoC 验证，内部培训 |
| 数据源限制 | 中 | 中 | 多数据源备份，缓存策略 |
| 性能不达标 | 高 | 低 | 异步处理，缓存优化 |

---

## 10. 风险评估与缓解

### 10.0 风险矩阵

```mermaid
quadrantChart
    title 风险评估矩阵
    x-axis 低影响 --> 高影响
    y-axis 低概率 --> 高概率
    quadrant-1 高优先级
    quadrant-2 中优先级
    quadrant-3 低优先级
    quadrant-4 监控区

    "合规风险": [0.9, 0.3]
    "资金安全": [0.95, 0.25]
    "富途API限制": [0.6, 0.45]
    "LangGraph稳定性": [0.5, 0.4]
    "数据一致性": [0.7, 0.6]
    "需求变更": [0.4, 0.55]
    "Python性能": [0.3, 0.3]
    "人员风险": [0.35, 0.35]
    "进度延期": [0.45, 0.5]
```

### 10.1 风险响应流程

```mermaid
flowchart TD
    Risk["风险识别"] --> Assess["风险评估"]
    Assess --> Level{"风险等级?"}

    Level -->|"高"| Critical["立即处理"]
    Level -->|"中"| Medium["计划处理"]
    Level -->|"低"| Low["记录监控"]

    Critical --> Plan["制定缓解计划"]
    Plan --> Execute["执行缓解措施"]
    Execute --> Monitor["持续监控"]

    Medium --> Schedule["排入迭代"]
    Schedule --> Execute

    Low --> Log["记录风险日志"]
    Log --> Review["定期回顾"]

    Monitor --> Reassess["重新评估"]
    Review --> Reassess
    Reassess --> Assess
```

### 10.1 技术风险

| 风险项 | 等级 | 说明 | 缓解措施 |
|--------|------|------|----------|
| LangGraph 稳定性 | 中 | 新框架，可能有 bug | 充分测试，保留降级方案 |
| 富途 API 限制 | 中 | 频率限制，需要 OpenD | 使用模拟环境测试，合理限流 |
| 数据一致性 | 高 | 多数据源可能不一致 | 数据校验，异常告警 |
| Python 性能 | 低 | 解释型语言 | 异步 IO，缓存优化 |

### 10.2 业务风险

| 风险项 | 等级 | 说明 | 缓解措施 |
|--------|------|------|----------|
| 合规风险 | 高 | 港A股交易有监管要求 | 模拟盘优先，用户协议，风险提示 |
| 资金安全 | 高 | 程序化交易风险 | 人机协作审批，限额控制 |
| 市场风险 | 高 | 交易策略可能亏损 | 风险提示，小额测试 |

### 10.3 项目风险

| 风险项 | 等级 | 说明 | 缓解措施 |
|--------|------|------|----------|
| 需求变更 | 中 | 重构过程中需求可能变化 | 迭代开发，快速响应 |
| 人员风险 | 中 | 关键人员可能离职 | 文档完善，知识共享 |
| 进度延期 | 中 | 复杂度评估不足 | 预留缓冲时间，MVP 优先 |

---

## 11. 测试与部署策略

### 11.0 测试金字塔

```mermaid
graph TB
    subgraph E2E["端到端测试 (10%)"]
        E1["用户交易流程"]
        E2["审批流程"]
        E3["报告生成"]
    end

    subgraph Integration["集成测试 (30%)"]
        I1["API 端点测试"]
        I2["数据库操作"]
        I3["券商接口测试"]
        I4["LangGraph 工作流"]
    end

    subgraph Unit["单元测试 (60%)"]
        U1["工具函数"]
        U2["状态转换"]
        U3["节点逻辑"]
        U4["数据验证"]
    end

    Unit --> Integration
    Integration --> E2E
```

### 11.1 测试策略

```mermaid
flowchart LR
    subgraph TestTypes["测试类型"]
        Unit["单元测试<br/>pytest"]
        Integration["集成测试<br/>pytest-asyncio"]
        E2E["端到端测试<br/>Playwright"]
    end

    subgraph Coverage["覆盖率目标"]
        C1["代码覆盖率: 80%+"]
        C2["分支覆盖率: 70%+"]
        C3["关键路径: 100%"]
    end

    subgraph Tools["测试工具"]
        T1["pytest"]
        T2["pytest-asyncio"]
        T3["pytest-cov"]
        T4["factory_boy"]
        T5["Playwright"]
    end

    TestTypes --> Coverage
    Tools --> TestTypes
```

### 11.2 CI/CD 流水线

```mermaid
flowchart TD
    subgraph Trigger["触发条件"]
        Push["Push to main"]
        PR["Pull Request"]
        Schedule["定时构建"]
    end

    subgraph Build["构建阶段"]
        Checkout["代码检出"]
        Install["依赖安装"]
        Lint["代码检查<br/>ruff/mypy"]
        Build["构建"]
    end

    subgraph Test["测试阶段"]
        Unit["单元测试"]
        Integration["集成测试"]
        Coverage["覆盖率检查"]
        Security["安全扫描"]
    end

    subgraph Deploy["部署阶段"]
        Docker["Docker 构建"]
        Staging["部署到 Staging"]
        E2E_Test["E2E 测试"]
        Production["部署到生产"]
    end

    Trigger --> Build
    Build --> Test
    Test --> Deploy
```

### 11.3 GitHub Actions 配置

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install ruff mypy
      - run: ruff check src/
      - run: mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: nofx_test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install poetry
      - run: poetry install
      - run: poetry run pytest --cov=src --cov-report=xml
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/nofx_test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t nofx:${{ github.sha }} .
      - name: Push to Registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USER }} --password-stdin
          docker push nofx:${{ github.sha }}
```

### 11.4 部署架构

```mermaid
graph TB
    subgraph Cloud["云基础设施"]
        subgraph K8s["Kubernetes 集群"]
            Ingress["Ingress Controller"]
            API["FastAPI Pods<br/>replicas: 3"]
            Worker["LangGraph Workers<br/>replicas: 2"]
        end

        subgraph Data["数据层"]
            PG["PostgreSQL<br/>Primary + Replica"]
            Redis["Redis<br/>缓存/队列"]
        end

        subgraph Monitoring["监控层"]
            Prometheus["Prometheus"]
            Grafana["Grafana"]
            Loki["Loki<br/>日志"]
        end
    end

    subgraph External["外部服务"]
        Futu["富途 OpenD"]
        Telegram["Telegram API"]
        AI["AI 模型 API"]
    end

    Ingress --> API
    API --> Worker
    API --> PG
    API --> Redis
    Worker --> Futu
    Worker --> AI
    API --> Telegram

    Prometheus --> API
    Prometheus --> Worker
    Prometheus --> PG
    Grafana --> Prometheus
```

### 11.5 监控告警架构

```mermaid
flowchart LR
    subgraph App["应用层"]
        Metrics["指标采集<br/>Prometheus Client"]
        Logs["日志采集<br/>Structured Logging"]
        Traces["链路追踪<br/>OpenTelemetry"]
    end

    subgraph Collect["采集层"]
        Prom["Prometheus"]
        Lok["Loki"]
        Temp["Tempo"]
    end

    subgraph Alert["告警层"]
        AlertManager["AlertManager"]
        Rules["告警规则"]
        Routes["告警路由"]
    end

    subgraph Notify["通知渠道"]
        Email["邮件"]
        Slack["Slack"]
        PagerDuty["PagerDuty"]
    end

    Metrics --> Prom
    Logs --> Lok
    Traces --> Temp

    Prom --> AlertManager
    AlertManager --> Rules
    Rules --> Routes
    Routes --> Email
    Routes --> Slack
    Routes --> PagerDuty
```

#### 关键告警规则

| 告警名称 | 条件 | 级别 | 通知渠道 |
|---------|------|------|---------|
| API 响应超时 | P99 > 1s | Warning | Slack |
| API 错误率 | > 5% | Critical | PagerDuty |
| 数据库连接池耗尽 | 连接数 > 90% | Critical | PagerDuty |
| LangGraph 工作流失败 | 失败率 > 1% | Warning | Slack |
| 券商连接断开 | 连接状态 = false | Critical | PagerDuty |
| 磁盘使用率 | > 80% | Warning | Email |

---

## 附录

### A. 技术栈清单

```toml
# pyproject.toml

[tool.poetry]
name = "nofx"
version = "2.0.0"
description = "AI Trading Assistant for HK/A-Share Markets"

[tool.poetry.dependencies]
python = "^3.11"

# Web Framework
fastapi = "^0.115.0"
uvicorn = "^0.30.0"

# AI / LangGraph
langgraph = "^0.2.0"
langchain-openai = "^0.2.0"
langchain-anthropic = "^0.2.0"

# Database
sqlalchemy = "^2.0"
asyncpg = "^0.29"
alembic = "^1.13"

# Broker APIs
futu-api = "^6.5.0"

# Data Sources
akshare = "^1.14"
tushare = "^1.4"

# Auth
python-jose = "^3.3"
passlib = "^1.7"

# Utils
pydantic = "^2.0"
pydantic-settings = "^2.0"
httpx = "^0.27"

# Telegram
python-telegram-bot = "^21.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
pytest-asyncio = "^0.24"
pytest-cov = "^5.0"
ruff = "^0.6"
mypy = "^1.11"
```

### B. 环境变量配置

```bash
# .env.example

# Database
DATABASE_URL=postgresql+asyncpg://nofx:nofx@localhost:5432/nofx

# JWT
JWT_SECRET=your-secret-key
JWT_ALGORITHM=HS256
JWT_EXPIRE_HOURS=24

# AI Providers
OPENAI_API_KEY=sk-xxx
DEEPSEEK_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-xxx

# Broker
FUTU_HOST=127.0.0.1
FUTU_PORT=11111

# Data Sources
TUSHARE_TOKEN=xxx

# Telegram
TELEGRAM_BOT_TOKEN=xxx

# App
APP_ENV=development
DEBUG=true
```

### C. 参考资料

1. [LangGraph 官方文档](https://docs.langchain.com/oss/python/langgraph/)
2. [富途 OpenAPI 文档](https://openapi.futunn.com/futu-api-doc/)
3. [AkShare 文档](https://akshare.akfamily.xyz/)
4. [FastAPI 文档](https://fastapi.tiangolo.com/)
5. [原 NOFX 项目](https://github.com/NoFxAiOS/nofx)

---

> **文档版本**: v2.0
> **最后更新**: 2026-03-19
> **作者**: Claude Code Team

---

## 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0 | 2026-03-19 | 初始版本 |
| v2.0 | 2026-03-19 | 增加 Mermaid 图表、完善测试部署章节、优化可读性 |

### 文档增强摘要

v2.0 版本新增以下图表和内容：

**架构设计**
- LangGraph 核心概念图
- 状态流转图 (State Diagram)
- 数据流时序图
- Supervisor 多智能体架构图
- 节点协作序列图

**港A股适配**
- 市场概览图
- 交易时段甘特图
- 涨跌停规则决策图
- 券商适配器类图
- 券商能力对比图

**实施路线图**
- 项目时间线图
- 甘特图 (12周规划)
- 里程碑依赖图

**风险管理**
- 风险矩阵图 (Quadrant Chart)
- 风险响应流程图

**测试与部署** (新章节)
- 测试金字塔图
- CI/CD 流水线图
- 部署架构图
- 监控告警架构图
- GitHub Actions 配置示例