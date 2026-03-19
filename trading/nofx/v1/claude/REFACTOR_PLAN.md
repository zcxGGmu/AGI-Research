# NoFx 重构方案：加密货币 → 港A股 + LangGraph 多智能体架构

> 版本: v2.0 | 日期: 2026-03-19 | 状态: 草案

---

## 目录

1. [项目概述与重构目标](#1-项目概述与重构目标)
2. [现有架构分析](#2-现有架构分析)
3. [技术选型](#3-技术选型)
4. [新架构设计](#4-新架构设计)
5. [LangGraph 多智能体系统设计](#5-langgraph-多智能体系统设计)
6. [数据源替换方案](#6-数据源替换方案)
7. [交易接口适配层](#7-交易接口适配层)
8. [前端改造方案](#8-前端改造方案)
9. [数据库 Schema 迁移](#9-数据库-schema-迁移)
10. [分阶段实施计划](#10-分阶段实施计划)
11. [风险评估与缓解](#11-风险评估与缓解)
12. [测试策略](#12-测试策略)

---

## 1. 项目概述与重构目标

### 1.1 现有系统

NoFx 是一个 AI 驱动的加密货币自动交易系统，采用 Go 后端 + React 前端架构：
- 后端：Go (Gin) 单体应用，支持 5 个加密货币交易所（Binance、Bybit、Hyperliquid、Aster、LIGHTER）
- AI 决策：单次 LLM 调用（DeepSeek/Qwen），无状态，无多智能体协作
- 前端：React 18 + TypeScript + Tailwind + Recharts

### 1.2 重构目标

| 维度 | 现状 | 目标 |
|------|------|------|
| 交易市场 | 加密货币（5个交易所） | 港股 + A股 |
| AI 架构 | 单次 LLM 调用 | LangGraph 多智能体协作 |
| 后端语言 | Go | Python（LangGraph 原生支持） |
| 决策流程 | 单步：数据→LLM→决策 | 多步：研究→分析→风控→决策→执行 |
| 状态管理 | 无状态 | 有状态（LangGraph Checkpointer） |
| 人工干预 | 无 | Human-in-the-Loop（大额交易审批） |

### 1.3 核心原则

1. **渐进式重构**：分阶段实施，每阶段可独立验证
2. **前端复用**：最大化复用现有 React 前端，仅替换数据层
3. **架构解耦**：LangGraph 智能体系统与交易执行层分离
4. **风控优先**：港A股有 T+1/涨跌停限制，风控逻辑需完全重写

---

## 2. 现有架构分析

### 2.1 需要删除的模块（加密货币相关）

```
trader/
├── binance_futures.go      ❌ 删除（Binance 合约交易）
├── bybit_trader.go         ❌ 删除（Bybit 交易）
├── hyperliquid_trader.go   ❌ 删除（Hyperliquid DEX）
├── aster_trader.go         ❌ 删除（Aster DEX）
├── lighter_trader.go       ❌ 删除（LIGHTER DEX V1）
├── lighter_trader_v2*.go   ❌ 删除（LIGHTER DEX V2）
├── interface.go            🔄 重写（Trader 接口适配港A股）
├── auto_trader.go          🔄 重写（用 LangGraph 替代）
└── helpers.go              🔄 重写

pool/
├── coin_pool.go            ❌ 删除（AI500 + OI Top 币种池）

market/
├── api_client.go           ❌ 删除（Binance REST API）
├── websocket_client.go     ❌ 删除（Binance WebSocket）
├── combined_streams.go     ❌ 删除
├── monitor.go              ❌ 删除
├── data.go                 🔄 重写（港A股数据源）
├── types.go                🔄 重写（港A股数据结构）

mcp/                        ❌ 删除整个目录（用 LangGraph 替代）

decision/
├── engine.go               ❌ 删除（用 LangGraph 智能体替代）
├── prompt_manager.go       🔄 重构为 LangGraph prompt 模板

prompts/                    🔄 重写所有提示词模板
```

### 2.2 可复用的模块

```
web/                        ✅ 大部分复用（替换数据类型和 API 调用）
api/server.go               🔄 部分复用（路由结构、认证中间件）
auth/                       ✅ 完全复用（JWT + OTP）
crypto/                     ✅ 完全复用（RSA + AES 加密）
config/                     🔄 部分复用（配置结构需修改）
logger/                     ✅ 完全复用（日志 + Telegram 通知）
bootstrap/                  ✅ 完全复用（初始化框架）
```

---

## 3. 技术选型

### 3.1 后端技术栈

| 组件 | 选型 | 理由 |
|------|------|------|
| AI 编排 | LangGraph (Python) | 原生多智能体、状态管理、HITL |
| Web 框架 | FastAPI | 异步、WebSocket、OpenAPI 文档 |
| 数据库 | PostgreSQL + SQLAlchemy | 生产级持久化，LangGraph PostgresSaver |
| 缓存 | Redis | 行情缓存、会话管理 |
| 任务队列 | Celery / APScheduler | 定时扫描、异步执行 |
| 行情数据 | AKShare / Tushare / 东方财富 API | 港A股实时/历史数据 |
| 交易接口 | 券商 API（富途/长桥/盈透） | 港A股下单执行 |

### 3.2 LangGraph 依赖

```toml
[project]
name = "nofx-v2"
requires-python = ">=3.11"

[project.dependencies]
langgraph = ">=1.1"
langgraph-supervisor = ">=0.1"
langgraph-checkpoint-postgres = ">=0.1"
langchain-openai = ">=0.3"
langchain-community = ">=0.3"
fastapi = ">=0.115"
uvicorn = ">=0.34"
sqlalchemy = ">=2.0"
asyncpg = ">=0.30"
redis = ">=5.0"
akshare = ">=1.14"
pydantic = ">=2.0"
```

### 3.3 港A股数据源对比

| 数据源 | A股 | 港股 | 实时行情 | 历史K线 | 免费 | 推荐 |
|--------|-----|------|----------|---------|------|------|
| AKShare | ✅ | ✅ | ✅(延迟) | ✅ | ✅ | ⭐⭐⭐ 首选 |
| Tushare | ✅ | ✅ | ✅ | ✅ | 部分 | ⭐⭐ |
| 东方财富 | ✅ | ✅ | ✅ | ✅ | ✅ | ⭐⭐ |
| 富途 OpenAPI | ✅ | ✅ | ✅(实时) | ✅ | 付费 | ⭐⭐⭐ 交易首选 |
| 长桥 OpenAPI | ✅ | ✅ | ✅(实时) | ✅ | 付费 | ⭐⭐⭐ |

推荐组合：**AKShare（免费行情+分析）+ 富途/长桥 OpenAPI（交易执行）**

---

## 4. 新架构设计

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Frontend                           │
│  (复用现有 UI，替换数据类型和 API 调用)                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │ REST + WebSocket
┌──────────────────────────▼──────────────────────────────────────┐
│                     FastAPI Gateway                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Auth API │  │Trader API│  │Market API│  │ WebSocket API│    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              LangGraph Multi-Agent System                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Supervisor Agent                       │    │
│  │            (任务路由 + 全局状态管理)                       │    │
│  └────────┬──────────┬──────────┬──────────┬───────────────┘    │
│           │          │          │          │                     │
│  ┌────────▼───┐ ┌────▼────┐ ┌──▼──────┐ ┌▼──────────┐         │
│  │ Research   │ │Technical│ │Fundament│ │ Sentiment │         │
│  │ Agent      │ │ Analyst │ │ Analyst │ │ Analyst   │         │
│  │(市场研究)  │ │(技术分析)│ │(基本面) │ │(舆情分析) │         │
│  └────────┬───┘ └────┬────┘ └──┬──────┘ └┬──────────┘         │
│           └──────────┼─────────┼─────────┘                     │
│                      │                                          │
│              ┌───────▼────────┐                                 │
│              │  Synthesizer   │                                 │
│              │  (信号综合)     │                                 │
│              └───────┬────────┘                                 │
│                      │                                          │
│              ┌───────▼────────┐                                 │
│              │ Risk Manager   │                                 │
│              │ (风控审核)      │                                 │
│              └───────┬────────┘                                 │
│                      │                                          │
│              ┌───────▼────────┐     ┌──────────────┐           │
│              │ HITL Gate      │────▶│ Human Review │           │
│              │ (人工审批门控)  │◀────│ (大额审批)   │           │
│              └───────┬────────┘     └──────────────┘           │
│                      │                                          │
│              ┌───────▼────────┐                                 │
│              │ Execution Agent│                                 │
│              │ (交易执行)      │                                 │
│              └────────────────┘                                 │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
│   PostgreSQL    │ │     Redis       │ │  Broker API     │
│ (状态+检查点)   │ │ (行情缓存)      │ │ (富途/长桥)     │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 4.2 目录结构

```
nofx-v2/
├── pyproject.toml                    # Python 项目配置
├── langgraph.json                    # LangGraph Server 配置
├── docker-compose.yml                # 容器编排
├── .env.example                      # 环境变量模板
│
├── src/
│   ├── __init__.py
│   │
│   ├── agents/                       # LangGraph 智能体
│   │   ├── __init__.py
│   │   ├── state.py                  # 全局状态定义
│   │   ├── supervisor.py             # Supervisor 智能体
│   │   ├── research_agent.py         # 市场研究智能体
│   │   ├── technical_analyst.py      # 技术分析智能体
│   │   ├── fundamental_analyst.py    # 基本面分析智能体
│   │   ├── sentiment_analyst.py      # 舆情分析智能体
│   │   ├── synthesizer.py            # 信号综合智能体
│   │   ├── risk_manager.py           # 风控智能体
│   │   ├── execution_agent.py        # 交易执行智能体
│   │   └── graph.py                  # 主图构建
│   │
│   ├── tools/                        # LangGraph 工具
│   │   ├── __init__.py
│   │   ├── market_data.py            # 行情数据工具
│   │   ├── technical_indicators.py   # 技术指标计算
│   │   ├── fundamental_data.py       # 基本面数据工具
│   │   ├── news_sentiment.py         # 新闻舆情工具
│   │   ├── broker_api.py             # 券商交易工具
│   │   └── risk_calculator.py        # 风险计算工具
│   │
│   ├── market/                       # 行情数据层
│   │   ├── __init__.py
│   │   ├── provider.py               # 数据源抽象接口
│   │   ├── akshare_provider.py       # AKShare 实现
│   │   ├── futu_provider.py          # 富途实时行情
│   │   ├── types.py                  # 港A股数据类型
│   │   └── cache.py                  # Redis 行情缓存
│   │
│   ├── broker/                       # 券商交易层
│   │   ├── __init__.py
│   │   ├── interface.py              # 交易接口抽象
│   │   ├── futu_broker.py            # 富途交易实现
│   │   ├── longbridge_broker.py      # 长桥交易实现
│   │   ├── simulator.py              # 模拟交易（回测）
│   │   └── types.py                  # 交易类型定义
│   │
│   ├── api/                          # FastAPI 路由
│   │   ├── __init__.py
│   │   ├── app.py                    # FastAPI 应用
│   │   ├── auth.py                   # 认证路由
│   │   ├── traders.py                # Trader 管理路由
│   │   ├── market.py                 # 行情路由
│   │   ├── websocket.py              # WebSocket 推送
│   │   └── middleware.py             # 中间件
│   │
│   ├── models/                       # 数据库模型
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── trader.py
│   │   ├── exchange.py               # → broker_config.py
│   │   └── decision_log.py
│   │
│   ├── config/                       # 配置
│   │   ├── __init__.py
│   │   ├── settings.py               # Pydantic Settings
│   │   └── constants.py              # 港A股常量
│   │
│   └── utils/                        # 工具
│       ├── __init__.py
│       ├── crypto.py                 # 加密（复用）
│       ├── logger.py                 # 日志
│       └── telegram.py               # Telegram 通知
│
├── prompts/                          # 提示词模板
│   ├── supervisor.md
│   ├── technical_analyst.md
│   ├── fundamental_analyst.md
│   ├── sentiment_analyst.md
│   ├── risk_manager.md
│   └── synthesizer.md
│
├── tests/                            # 测试
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
└── web/                              # 前端（从原项目复制并修改）
    └── ...
```

---

## 5. LangGraph 多智能体系统设计

### 5.1 全局状态定义

```python
from typing import Annotated, TypedDict, Literal
from langgraph.graph import add_messages
import operator

class MarketRegime(TypedDict):
    """市场环境判断"""
    trend: Literal["bull", "bear", "sideways"]
    volatility: Literal["low", "medium", "high"]
    sector_rotation: str  # 当前热点板块

class StockAnalysis(TypedDict):
    """单只股票分析结果"""
    symbol: str
    market: Literal["A", "HK"]
    technical_score: float       # -100 ~ +100
    fundamental_score: float     # -100 ~ +100
    sentiment_score: float       # -100 ~ +100
    composite_score: float       # 综合评分
    signal: Literal["strong_buy", "buy", "hold", "sell", "strong_sell"]
    reasoning: str

class RiskAssessment(TypedDict):
    """风控评估"""
    approved: bool
    risk_level: Literal["low", "medium", "high", "extreme"]
    max_position_pct: float      # 最大仓位占比
    stop_loss_pct: float         # 建议止损比例
    warnings: list[str]
    adjustments: dict            # 风控调整建议

class TradeOrder(TypedDict):
    """交易指令"""
    symbol: str
    market: Literal["A", "HK"]
    action: Literal["buy", "sell", "hold"]
    quantity: int                 # 股数（A股100整数倍，港股按手）
    price_type: Literal["market", "limit"]
    limit_price: float | None
    stop_loss: float
    take_profit: float
    reasoning: str

class TradingState(TypedDict):
    """LangGraph 全局状态"""
    # 消息历史
    messages: Annotated[list, add_messages]

    # 输入
    trader_id: str
    watchlist: list[str]         # 关注股票列表
    portfolio: dict              # 当前持仓

    # 中间结果（各智能体产出）
    market_regime: MarketRegime | None
    stock_analyses: Annotated[list[StockAnalysis], operator.add]
    risk_assessment: RiskAssessment | None

    # 输出
    trade_orders: list[TradeOrder]
    execution_results: list[dict]

    # 控制流
    current_phase: str
    requires_human_approval: bool
    human_decision: str | None
```

### 5.2 智能体职责定义

| 智能体 | 职责 | 输入 | 输出 | 工具 |
|--------|------|------|------|------|
| Supervisor | 任务路由、全局协调 | 用户请求 | 路由决策 | 无（纯路由） |
| Research Agent | 市场环境研究 | watchlist | market_regime | 新闻API、宏观数据 |
| Technical Analyst | 技术指标分析 | 股票代码 | technical_score | K线数据、指标计算 |
| Fundamental Analyst | 基本面分析 | 股票代码 | fundamental_score | 财报、估值数据 |
| Sentiment Analyst | 舆情分析 | 股票代码 | sentiment_score | 新闻、社交媒体 |
| Synthesizer | 信号综合 | 三维评分 | trade_orders | 无（纯推理） |
| Risk Manager | 风控审核 | trade_orders | risk_assessment | 风险计算器 |
| Execution Agent | 交易执行 | approved_orders | execution_results | 券商API |

### 5.3 Graph 构建（核心代码）

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, interrupt, Command
from langgraph.checkpoint.postgres import PostgresSaver

def build_trading_graph() -> StateGraph:
    builder = StateGraph(TradingState)

    # 注册节点
    builder.add_node("supervisor", supervisor_node)
    builder.add_node("research", research_agent)
    builder.add_node("technical_analyst", technical_analyst)
    builder.add_node("fundamental_analyst", fundamental_analyst)
    builder.add_node("sentiment_analyst", sentiment_analyst)
    builder.add_node("synthesizer", synthesizer_node)
    builder.add_node("risk_manager", risk_manager_node)
    builder.add_node("human_review", human_review_node)
    builder.add_node("execution", execution_agent)

    # 入口 → Supervisor
    builder.add_edge(START, "supervisor")

    # Supervisor → Research（市场环境分析）
    builder.add_edge("supervisor", "research")

    # Research → 并行分析（使用 Send API fan-out）
    builder.add_conditional_edges("research", fan_out_analysts)

    # 三个分析师 → Synthesizer（fan-in 等待全部完成）
    builder.add_edge(
        ["technical_analyst", "fundamental_analyst", "sentiment_analyst"],
        "synthesizer"
    )

    # Synthesizer → Risk Manager
    builder.add_edge("synthesizer", "risk_manager")

    # Risk Manager → 条件路由（是否需要人工审批）
    builder.add_conditional_edges("risk_manager", route_after_risk, {
        "human_review": "human_review",
        "execution": "execution",
        "end": END,
    })

    # Human Review → 条件路由
    builder.add_conditional_edges("human_review", route_after_human, {
        "execution": "execution",
        "end": END,
    })

    # Execution → END
    builder.add_edge("execution", END)

    return builder


def fan_out_analysts(state: TradingState):
    """并行分发给三个分析师（Send API）"""
    sends = []
    for symbol in state["watchlist"]:
        sends.extend([
            Send("technical_analyst", {**state, "current_symbol": symbol}),
            Send("fundamental_analyst", {**state, "current_symbol": symbol}),
            Send("sentiment_analyst", {**state, "current_symbol": symbol}),
        ])
    return sends


def route_after_risk(state: TradingState) -> str:
    """风控后路由：大额交易需人工审批"""
    if not state.get("trade_orders"):
        return "end"
    if state.get("requires_human_approval"):
        return "human_review"
    return "execution"


def route_after_human(state: TradingState) -> str:
    """人工审批后路由"""
    if state.get("human_decision") == "approved":
        return "execution"
    return "end"


def human_review_node(state: TradingState):
    """Human-in-the-Loop 审批节点"""
    orders_summary = format_orders_for_review(state["trade_orders"])
    decision = interrupt(f"请审批以下交易:\n{orders_summary}")
    return {
        "human_decision": decision,
        "requires_human_approval": False,
    }
```

### 5.4 关键智能体实现示例

#### Technical Analyst（技术分析智能体）

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def get_kline_data(symbol: str, period: str = "daily", count: int = 120) -> dict:
    """获取K线数据（支持A股和港股）"""
    import akshare as ak
    if symbol.startswith(("6", "0", "3")):  # A股
        df = ak.stock_zh_a_hist(symbol=symbol, period=period, adjust="qfq")
    else:  # 港股
        df = ak.stock_hk_hist(symbol=symbol, period=period, adjust="qfq")
    return df.tail(count).to_dict(orient="records")

@tool
def calculate_indicators(kline_data: list[dict]) -> dict:
    """计算技术指标：MACD、RSI、EMA、布林带、ATR"""
    import pandas as pd
    import talib
    df = pd.DataFrame(kline_data)
    close = df["收盘"].values
    return {
        "macd": talib.MACD(close)[0][-1],
        "rsi_14": talib.RSI(close, 14)[-1],
        "ema_20": talib.EMA(close, 20)[-1],
        "ema_50": talib.EMA(close, 50)[-1],
        "bb_upper": talib.BBANDS(close)[0][-1],
        "bb_lower": talib.BBANDS(close)[2][-1],
        "atr_14": talib.ATR(df["最高"].values, df["最低"].values, close, 14)[-1],
    }

technical_analyst = create_react_agent(
    model="openai:gpt-4o",
    tools=[get_kline_data, calculate_indicators],
    name="technical_analyst",
    prompt="""你是专业的技术分析师，专注于港股和A股市场。
    分析给定股票的技术面，输出 -100 到 +100 的评分。
    考虑：趋势（EMA交叉）、动量（MACD/RSI）、波动率（ATR/布林带）、成交量。
    A股注意：T+1限制、涨跌停板（±10%/±20%创业板）。
    港股注意：无涨跌停、可做空、T+0。""",
)
```

#### Risk Manager（风控智能体）

```python
def risk_manager_node(state: TradingState) -> dict:
    """风控审核节点 - 港A股特殊规则"""
    orders = state.get("trade_orders", [])
    portfolio = state.get("portfolio", {})
    warnings = []
    approved_orders = []

    for order in orders:
        risk_checks = []

        # 1. A股 T+1 检查
        if order["market"] == "A" and order["action"] == "sell":
            if is_bought_today(order["symbol"], portfolio):
                warnings.append(f"{order['symbol']}: A股T+1限制，今日买入不可卖出")
                continue

        # 2. 涨跌停检查
        if order["market"] == "A":
            limit = get_price_limit(order["symbol"])  # 10% 或 20%
            if is_near_limit(order["symbol"], limit):
                warnings.append(f"{order['symbol']}: 接近涨跌停板，风险较高")

        # 3. 单只股票仓位上限（不超过总资产20%）
        position_pct = calculate_position_pct(order, portfolio)
        if position_pct > 0.20:
            order["quantity"] = adjust_quantity(order, 0.20, portfolio)
            warnings.append(f"{order['symbol']}: 仓位调整至20%上限")

        # 4. 总仓位检查（不超过80%）
        total_position_pct = calculate_total_position_pct(portfolio)
        if total_position_pct > 0.80:
            warnings.append("总仓位超过80%，暂停新开仓")
            if order["action"] == "buy":
                continue

        # 5. 大额交易审批（单笔超过总资产5%）
        if position_pct > 0.05:
            return {
                "requires_human_approval": True,
                "trade_orders": [order],
                "risk_assessment": {
                    "approved": True,
                    "risk_level": "high",
                    "warnings": warnings,
                },
            }

        approved_orders.append(order)

    return {
        "trade_orders": approved_orders,
        "risk_assessment": {
            "approved": len(approved_orders) > 0,
            "risk_level": "medium" if warnings else "low",
            "warnings": warnings,
        },
    }
```

### 5.5 LangGraph 编译与运行

```python
from langgraph.checkpoint.postgres import PostgresSaver

# 构建图
builder = build_trading_graph()

# 编译（带持久化 + HITL）
checkpointer = PostgresSaver.from_conn_string(DATABASE_URL)
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_review"],  # 人工审批前暂停
)

# 运行一次分析周期
config = {"configurable": {"thread_id": f"trader-{trader_id}-{cycle_id}"}}
result = graph.invoke(
    {
        "trader_id": trader_id,
        "watchlist": ["600519", "00700", "000858", "09988"],
        "portfolio": current_portfolio,
    },
    config=config,
)
```

---

## 6. 数据源替换方案

### 6.1 港A股数据类型定义

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

class Market(Enum):
    A_SHARE = "A"       # A股（上海/深圳）
    HK_STOCK = "HK"     # 港股

class StockBoard(Enum):
    """A股板块分类"""
    MAIN = "main"           # 主板（±10%）
    GEM = "gem"             # 创业板（±20%）
    STAR = "star"           # 科创板（±20%）
    BSE = "bse"             # 北交所（±30%）

@dataclass
class StockQuote:
    """股票实时行情"""
    symbol: str
    name: str
    market: Market
    board: StockBoard | None     # 仅A股
    current_price: float
    open_price: float
    high_price: float
    low_price: float
    prev_close: float
    change_pct: float            # 涨跌幅%
    volume: int                  # 成交量（股）
    turnover: float              # 成交额（元）
    turnover_rate: float         # 换手率%
    pe_ratio: float              # 市盈率
    pb_ratio: float              # 市净率
    market_cap: float            # 总市值
    free_float_cap: float        # 流通市值
    limit_up: float | None       # 涨停价（仅A股）
    limit_down: float | None     # 跌停价（仅A股）
    is_suspended: bool           # 是否停牌
    timestamp: datetime

@dataclass
class StockKline:
    """K线数据"""
    symbol: str
    open: float
    high: float
    low: float
    close: float
    volume: int
    turnover: float
    timestamp: datetime

@dataclass
class FundamentalData:
    """基本面数据"""
    symbol: str
    name: str
    pe_ttm: float               # 滚动市盈率
    pb: float                   # 市净率
    ps_ttm: float               # 滚动市销率
    roe: float                  # 净资产收益率
    revenue_growth: float       # 营收增长率
    profit_growth: float        # 净利润增长率
    debt_ratio: float           # 资产负债率
    dividend_yield: float       # 股息率
    free_cash_flow: float       # 自由现金流
    sector: str                 # 所属行业
    concept: list[str]          # 概念板块
```

### 6.2 数据源适配器（AKShare 实现）

```python
import akshare as ak
import pandas as pd
from abc import ABC, abstractmethod

class MarketDataProvider(ABC):
    """行情数据提供者抽象接口"""

    @abstractmethod
    async def get_quote(self, symbol: str) -> StockQuote: ...

    @abstractmethod
    async def get_kline(self, symbol: str, period: str, count: int) -> list[StockKline]: ...

    @abstractmethod
    async def get_fundamental(self, symbol: str) -> FundamentalData: ...

    @abstractmethod
    async def get_sector_flow(self) -> dict: ...


class AKShareProvider(MarketDataProvider):
    """AKShare 数据源实现"""

    async def get_quote(self, symbol: str) -> StockQuote:
        if self._is_a_share(symbol):
            df = ak.stock_zh_a_spot_em()
            row = df[df["代码"] == symbol].iloc[0]
            return StockQuote(
                symbol=symbol,
                name=row["名称"],
                market=Market.A_SHARE,
                board=self._detect_board(symbol),
                current_price=row["最新价"],
                change_pct=row["涨跌幅"],
                volume=int(row["成交量"]),
                turnover=row["成交额"],
                pe_ratio=row.get("市盈率-动态", 0),
                # ... 其他字段
            )
        else:
            df = ak.stock_hk_spot_em()
            # ... 港股处理

    async def get_kline(self, symbol: str, period: str = "daily", count: int = 120) -> list[StockKline]:
        if self._is_a_share(symbol):
            df = ak.stock_zh_a_hist(symbol=symbol, period=period, adjust="qfq")
        else:
            df = ak.stock_hk_hist(symbol=symbol, period=period, adjust="qfq")
        return [StockKline(**row) for row in df.tail(count).to_dict("records")]

    def _is_a_share(self, symbol: str) -> bool:
        return symbol[:1] in ("6", "0", "3", "8", "4")

    def _detect_board(self, symbol: str) -> StockBoard:
        if symbol.startswith("3"):
            return StockBoard.GEM
        if symbol.startswith("68"):
            return StockBoard.STAR
        if symbol.startswith(("8", "4")):
            return StockBoard.BSE
        return StockBoard.MAIN
```

### 6.3 原系统数据源映射

| 原系统（加密货币） | 新系统（港A股） | 说明 |
|---------------------|-----------------|------|
| `market.Get(symbol)` | `provider.get_quote(symbol)` | 实时行情 |
| `market.GetKlines()` | `provider.get_kline()` | K线数据 |
| `pool.GetCoinPool()` | 选股策略（板块轮动/热点追踪） | 候选标的来源 |
| `pool.GetOITopPositions()` | 北向资金/融资融券数据 | 资金流向信号 |
| `market.OpenInterest` | 融资融券余额 | 市场情绪指标 |
| `market.FundingRate` | 无对应（删除） | 加密货币特有 |

---

## 7. 交易接口适配层

### 7.1 港A股交易接口定义

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum

class OrderSide(Enum):
    BUY = "buy"
    SELL = "sell"

class OrderType(Enum):
    MARKET = "market"       # 市价单
    LIMIT = "limit"         # 限价单
    LIMIT_IF_TOUCHED = "lit"  # 触价单（港股）

class OrderStatus(Enum):
    PENDING = "pending"
    PARTIAL = "partial"
    FILLED = "filled"
    CANCELLED = "cancelled"
    REJECTED = "rejected"

@dataclass
class OrderRequest:
    symbol: str
    market: Market
    side: OrderSide
    quantity: int            # A股：100整数倍；港股：按手
    order_type: OrderType
    limit_price: float | None = None

@dataclass
class OrderResult:
    order_id: str
    symbol: str
    status: OrderStatus
    filled_quantity: int
    filled_price: float
    commission: float
    timestamp: datetime

@dataclass
class PortfolioPosition:
    symbol: str
    name: str
    market: Market
    quantity: int            # 持仓数量
    available_quantity: int  # 可卖数量（A股T+1）
    cost_price: float        # 成本价
    current_price: float     # 现价
    market_value: float      # 市值
    unrealized_pnl: float    # 浮动盈亏
    unrealized_pnl_pct: float

class BrokerInterface(ABC):
    """券商交易接口抽象"""

    @abstractmethod
    async def get_balance(self) -> dict: ...

    @abstractmethod
    async def get_positions(self) -> list[PortfolioPosition]: ...

    @abstractmethod
    async def place_order(self, order: OrderRequest) -> OrderResult: ...

    @abstractmethod
    async def cancel_order(self, order_id: str) -> bool: ...

    @abstractmethod
    async def get_order_status(self, order_id: str) -> OrderResult: ...

    @abstractmethod
    async def get_today_orders(self) -> list[OrderResult]: ...

    @abstractmethod
    async def get_today_trades(self) -> list[dict]: ...
```

### 7.2 富途 OpenAPI 实现

```python
from futu import OpenSecTradeContext, TrdSide, OrderType as FutuOrderType

class FutuBroker(BrokerInterface):
    def __init__(self, host: str, port: int, trd_env: str = "SIMULATE"):
        self.ctx = OpenSecTradeContext(host=host, port=port)
        self.trd_env = trd_env

    async def get_positions(self) -> list[PortfolioPosition]:
        ret, data = self.ctx.position_list_query()
        if ret != 0:
            raise BrokerError(f"获取持仓失败: {data}")
        return [
            PortfolioPosition(
                symbol=row["code"],
                name=row["stock_name"],
                market=self._detect_market(row["code"]),
                quantity=int(row["qty"]),
                available_quantity=int(row["can_sell_qty"]),
                cost_price=row["cost_price"],
                current_price=row["nominal_price"],
                market_value=row["market_val"],
                unrealized_pnl=row["pl_val"],
                unrealized_pnl_pct=row["pl_ratio"] * 100,
            )
            for _, row in data.iterrows()
        ]

    async def place_order(self, order: OrderRequest) -> OrderResult:
        # A股数量必须是100的整数倍
        if order.market == Market.A_SHARE:
            order.quantity = (order.quantity // 100) * 100
            if order.quantity < 100:
                raise BrokerError("A股最小交易单位为100股")

        side = TrdSide.BUY if order.side == OrderSide.BUY else TrdSide.SELL
        ret, data = self.ctx.place_order(
            price=order.limit_price or 0,
            qty=order.quantity,
            code=order.symbol,
            trd_side=side,
            order_type=self._map_order_type(order.order_type),
        )
        if ret != 0:
            raise BrokerError(f"下单失败: {data}")
        return OrderResult(order_id=str(data["order_id"][0]), ...)
```

### 7.3 模拟交易器（回测/开发用）

```python
class SimulatorBroker(BrokerInterface):
    """模拟交易器 - 用于回测和开发环境"""

    def __init__(self, initial_cash: float = 1_000_000):
        self.cash = initial_cash
        self.positions: dict[str, PortfolioPosition] = {}
        self.orders: list[OrderResult] = []
        self.today_buys: set[str] = set()  # T+1 追踪

    async def place_order(self, order: OrderRequest) -> OrderResult:
        # A股 T+1 检查
        if (order.market == Market.A_SHARE
            and order.side == OrderSide.SELL
            and order.symbol in self.today_buys):
            raise BrokerError(f"{order.symbol}: A股T+1限制")

        # 模拟成交
        price = await self._get_current_price(order.symbol)
        commission = price * order.quantity * 0.0003  # 万三佣金
        # ... 更新持仓和现金
```

### 7.4 原系统交易接口映射

| 原接口（Trader interface） | 新接口（BrokerInterface） | 差异说明 |
|---------------------------|--------------------------|----------|
| `GetBalance()` | `get_balance()` | 返回结构不同 |
| `GetPositions()` | `get_positions()` | 增加 available_quantity（T+1） |
| `OpenLong()` | `place_order(BUY)` | 无杠杆，A股100股整数倍 |
| `OpenShort()` | 仅港股支持做空 | A股无法做空（融券除外） |
| `CloseLong()` | `place_order(SELL)` | A股 T+1 限制 |
| `CloseShort()` | `place_order(BUY)` 平空 | 仅港股 |
| `SetLeverage()` | ❌ 删除 | 港A股无杠杆（融资除外） |
| `SetMarginMode()` | ❌ 删除 | 无保证金模式 |
| `SetStopLoss()` | 条件单 / 软件层面实现 | 非交易所原生支持 |
| `SetTakeProfit()` | 条件单 / 软件层面实现 | 非交易所原生支持 |
| `FormatQuantity()` | 内置于 place_order | A股100整数倍，港股按手 |

---

## 8. 前端改造方案

### 8.1 改造策略

前端采用**最小改动**策略，复用现有 React 组件结构，仅替换：
1. 数据类型定义（`types.ts`）
2. API 调用层（`lib/api.ts`）
3. 加密货币特有的 UI 文案和图标
4. 交易所配置改为券商配置

### 8.2 类型定义变更

```typescript
// types.ts - 港A股版本

export type MarketType = 'A' | 'HK'

export interface StockPosition {
  symbol: string
  name: string
  market: MarketType
  quantity: number
  available_quantity: number    // 可卖数量（A股T+1）
  cost_price: number
  current_price: number
  market_value: number
  unrealized_pnl: number
  unrealized_pnl_pct: number
  // 删除: leverage, liquidation_price, margin_used
}

export interface AccountInfo {
  total_assets: number          // 总资产
  cash_balance: number          // 现金余额
  market_value: number          // 持仓市值
  total_pnl: number
  total_pnl_pct: number
  position_count: number
  // 删除: margin_used, margin_used_pct, wallet_balance
}

export interface TraderInfo {
  trader_id: string
  trader_name: string
  ai_model: string
  broker_id?: string            // 原 exchange_id
  is_running?: boolean
  custom_prompt?: string
  watchlist?: string[]          // 原 trading_symbols (逗号分隔) → 数组
  market_filter?: MarketType[]  // 新增：市场筛选
}

export interface BrokerConfig {                // 原 Exchange
  id: string
  name: string                                 // "futu" | "longbridge" | "simulator"
  enabled: boolean
  api_key?: string
  api_secret?: string
  host?: string
  port?: number
  trade_env?: 'REAL' | 'SIMULATE'
  // 删除: 所有加密货币交易所特有字段
}

export interface CreateTraderRequest {
  name: string
  ai_model_id: string
  broker_id: string                            // 原 exchange_id
  initial_balance?: number
  scan_interval_minutes?: number
  watchlist?: string                           // 关注股票列表
  market_filter?: string                       // "A" | "HK" | "A,HK"
  custom_prompt?: string
  // 删除: leverage, is_cross_margin, use_coin_pool, use_oi_top
}
```

### 8.3 组件改造清单

| 组件 | 改动级别 | 具体变更 |
|------|----------|----------|
| `TraderDashboard` | 🔄 中等 | 删除杠杆/保证金显示，增加可卖数量、涨跌停提示 |
| `EquityChart` | ✅ 复用 | 仅改数据源 |
| `ComparisonChart` | ✅ 复用 | 仅改数据源 |
| `TraderConfigModal` | 🔄 中等 | 删除杠杆/保证金模式，改为关注列表+市场筛选 |
| `ExchangeConfigModal` | 🔄 重写 | → `BrokerConfigModal`（富途/长桥配置） |
| `ModelConfigModal` | ✅ 复用 | AI 模型配置不变 |
| `SignalSourceModal` | ❌ 删除 | 币种池概念不再适用 |
| `CompetitionPage` | ✅ 复用 | 仅改数据类型 |
| `Landing` 页面 | 🔄 中等 | 文案从"加密货币"改为"港A股" |
| `i18n/translations.ts` | 🔄 中等 | 更新所有加密货币相关翻译 |

### 8.4 新增前端组件

```
components/
├── stock/
│   ├── StockSearch.tsx          # 股票搜索（支持代码/名称/拼音）
│   ├── WatchlistEditor.tsx      # 关注列表编辑器
│   ├── StockQuoteCard.tsx       # 股票行情卡片
│   ├── MarketOverview.tsx       # 市场概览（A股/港股大盘）
│   ├── SectorHeatmap.tsx        # 板块热力图
│   └── T1Indicator.tsx          # T+1 可卖提示组件
├── broker/
│   └── BrokerConfigModal.tsx    # 券商配置弹窗
└── approval/
    └── TradeApprovalPanel.tsx   # 人工审批面板（HITL）
```

---

## 9. 数据库 Schema 迁移

### 9.1 PostgreSQL Schema（替代 SQLite）

```sql
-- 用户表（复用）
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    otp_secret VARCHAR(255),
    otp_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI 模型配置（复用）
CREATE TABLE ai_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    provider VARCHAR(50) NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    api_key_encrypted TEXT,          -- AES-256-GCM 加密
    custom_api_url VARCHAR(500),
    custom_model_name VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 券商配置（替代 exchanges 表）
CREATE TABLE brokers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,       -- "futu", "longbridge", "simulator"
    enabled BOOLEAN DEFAULT TRUE,
    api_key_encrypted TEXT,
    api_secret_encrypted TEXT,
    host VARCHAR(255) DEFAULT 'localhost',
    port INTEGER DEFAULT 11111,
    trade_env VARCHAR(20) DEFAULT 'SIMULATE',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, name)
);

-- Trader 配置（重构）
CREATE TABLE traders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    ai_model_id UUID REFERENCES ai_models(id),
    broker_id UUID REFERENCES brokers(id),
    initial_balance DECIMAL(15,2) DEFAULT 0,
    scan_interval_minutes INTEGER DEFAULT 5,
    is_running BOOLEAN DEFAULT FALSE,
    -- 港A股特有
    watchlist TEXT[],                 -- 关注股票列表
    market_filter VARCHAR(10) DEFAULT 'A,HK',
    max_position_pct DECIMAL(5,2) DEFAULT 0.20,  -- 单只最大仓位
    max_total_position_pct DECIMAL(5,2) DEFAULT 0.80,
    -- 提示词
    custom_prompt TEXT,
    override_base_prompt BOOLEAN DEFAULT FALSE,
    system_prompt_template VARCHAR(50) DEFAULT 'default',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 决策日志（新增，替代 JSON 文件）
CREATE TABLE decision_logs (
    id BIGSERIAL PRIMARY KEY,
    trader_id UUID REFERENCES traders(id) ON DELETE CASCADE,
    cycle_number INTEGER NOT NULL,
    system_prompt TEXT,
    user_prompt TEXT,
    cot_trace TEXT,
    decisions JSONB,
    execution_results JSONB,
    account_snapshot JSONB,
    positions_snapshot JSONB,
    success BOOLEAN DEFAULT TRUE,
    error_message TEXT,
    ai_request_duration_ms INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_decision_logs_trader ON decision_logs(trader_id, created_at DESC);

-- LangGraph 检查点（由 PostgresSaver 自动管理）
-- langgraph 会自动创建 checkpoints 和 checkpoint_writes 表
```

### 9.2 数据迁移策略

| 原表 | 新表 | 迁移方式 |
|------|------|----------|
| `users` | `users` | 直接迁移（结构兼容） |
| `ai_models` | `ai_models` | 直接迁移 |
| `exchanges` | `brokers` | 结构重写，数据不迁移 |
| `traders` | `traders` | 结构重写，删除杠杆/保证金字段 |
| `user_signal_sources` | ❌ 删除 | 币种池不再适用 |
| `system_config` | `system_config` | 直接迁移 |
| `beta_codes` | `beta_codes` | 直接迁移 |
| JSON 决策日志 | `decision_logs` | 迁移到 PostgreSQL |

---

## 10. 分阶段实施计划

### Phase 0: 基础设施搭建（1周）

```
目标：搭建 Python 项目骨架 + 开发环境
├── [ ] 初始化 Python 项目（pyproject.toml, uv/poetry）
├── [ ] 配置 PostgreSQL + Redis
├── [ ] 搭建 FastAPI 基础框架
├── [ ] 移植认证模块（JWT + OTP）
├── [ ] 移植加密模块（RSA + AES）
├── [ ] 配置 LangGraph + Checkpointer
├── [ ] Docker Compose 编排
└── [ ] CI/CD 基础配置
```

### Phase 1: 数据层（1-2周）

```
目标：港A股行情数据接入 + 缓存
├── [ ] 实现 MarketDataProvider 抽象接口
├── [ ] AKShare Provider（A股 + 港股行情）
├── [ ] 技术指标计算模块（TA-Lib）
├── [ ] Redis 行情缓存层
├── [ ] 基本面数据接入
├── [ ] 新闻/舆情数据接入
├── [ ] 单元测试（覆盖率 ≥ 80%）
└── [ ] 数据源健康检查 + 降级策略
```

### Phase 2: LangGraph 智能体系统（2-3周）

```
目标：核心多智能体系统
├── [ ] 定义 TradingState 全局状态
├── [ ] 实现 Supervisor Agent
├── [ ] 实现 Research Agent（市场环境分析）
├── [ ] 实现 Technical Analyst（技术分析）
├── [ ] 实现 Fundamental Analyst（基本面分析）
├── [ ] 实现 Sentiment Analyst（舆情分析）
├── [ ] 实现 Synthesizer（信号综合）
├── [ ] 实现 Risk Manager（风控，含港A股规则）
├── [ ] 实现 Human-in-the-Loop 审批流
├── [ ] 构建主 Graph + 编译
├── [ ] 集成测试（完整决策流程）
└── [ ] 提示词模板编写与调优
```

### Phase 3: 交易执行层（1-2周）

```
目标：券商交易接口
├── [ ] 实现 BrokerInterface 抽象
├── [ ] SimulatorBroker（模拟交易）
├── [ ] FutuBroker（富途 OpenAPI）
├── [ ] LongbridgeBroker（长桥 OpenAPI）
├── [ ] Execution Agent 集成
├── [ ] 条件单（止损/止盈）软件层实现
├── [ ] A股 T+1 / 涨跌停 / 100股整数倍 规则
├── [ ] 港股手数规则
└── [ ] 交易执行集成测试
```

### Phase 4: API 层 + 前端改造（1-2周）

```
目标：FastAPI 路由 + 前端适配
├── [ ] FastAPI 路由（Trader CRUD, Market, Auth）
├── [ ] WebSocket 实时推送
├── [ ] 前端类型定义更新（types.ts）
├── [ ] API 调用层更新（api.ts）
├── [ ] 组件改造（Dashboard, Config, etc.）
├── [ ] 新增组件（StockSearch, WatchlistEditor, etc.）
├── [ ] 人工审批面板（TradeApprovalPanel）
├── [ ] i18n 文案更新
└── [ ] E2E 测试
```

### Phase 5: 集成测试 + 上线（1周）

```
目标：全链路验证 + 生产部署
├── [ ] 全链路集成测试（数据→分析→决策→执行）
├── [ ] 模拟交易回测（至少1个月历史数据）
├── [ ] 性能测试（并发分析、延迟）
├── [ ] 安全审计
├── [ ] 生产环境部署
├── [ ] 监控 + 告警配置
└── [ ] 文档更新
```

### 总工期估算：6-10 周

---

## 11. 风险评估与缓解

### 11.1 技术风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Go → Python 性能下降 | 中 | 高 | 异步IO + Redis缓存 + 关键路径优化 |
| LangGraph 学习曲线 | 中 | 中 | 先用 create_react_agent 快速原型 |
| AKShare 数据延迟/不稳定 | 高 | 中 | 多数据源降级 + 缓存兜底 |
| 券商 API 限流 | 中 | 中 | 请求队列 + 速率限制 |
| LLM 幻觉导致错误决策 | 高 | 中 | 多智能体交叉验证 + 风控兜底 |

### 11.2 业务风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| A股 T+1 导致策略受限 | 中 | 高 | 调整策略为中长线，减少日内交易 |
| 涨跌停板导致无法止损 | 高 | 低 | 分散持仓 + 提前止损 + 风控预警 |
| 港A股交易时间限制 | 低 | 确定 | 定时任务仅在交易时段运行 |
| 监管合规风险 | 高 | 低 | 模拟交易优先，实盘需合规审查 |

### 11.3 港A股 vs 加密货币关键差异

| 特性 | 加密货币 | A股 | 港股 |
|------|----------|-----|------|
| 交易时间 | 24/7 | 9:30-15:00 | 9:30-16:00 |
| T+N | T+0 | T+1 | T+0 |
| 涨跌停 | 无 | ±10%/±20% | 无 |
| 做空 | 支持 | 融券（门槛高） | 支持 |
| 杠杆 | 1-125x | 融资（~1x） | 融资（~2x） |
| 最小单位 | 0.001 | 100股 | 1手（股数不等） |
| 结算 | 即时 | T+1 | T+2 |
| 手续费 | 0.02-0.1% | ~0.03%+印花税 | ~0.03%+印花税 |

---

## 12. 测试策略

### 12.1 测试金字塔

```
        ┌─────────┐
        │  E2E    │  Playwright（关键用户流程）
        │  Tests  │  - 创建Trader → 运行 → 查看决策
       ─┼─────────┼─
       │Integration│  pytest（API + LangGraph 集成）
       │  Tests    │  - 完整决策流程
       │           │  - 数据源 → 分析 → 风控 → 执行
      ─┼───────────┼─
      │  Unit Tests │  pytest（各模块独立测试）
      │             │  - 技术指标计算
      │             │  - 风控规则验证
      │             │  - A股T+1/涨跌停逻辑
      └─────────────┘
```

### 12.2 关键测试场景

```python
# 1. A股 T+1 限制测试
def test_a_share_t1_restriction():
    """今日买入的A股不能当日卖出"""
    broker = SimulatorBroker()
    await broker.place_order(OrderRequest("600519", Market.A_SHARE, OrderSide.BUY, 100, ...))
    with pytest.raises(BrokerError, match="T\\+1"):
        await broker.place_order(OrderRequest("600519", Market.A_SHARE, OrderSide.SELL, 100, ...))

# 2. A股涨跌停检查
def test_a_share_price_limit():
    """接近涨跌停时风控应发出警告"""
    state = build_test_state(symbol="000858", change_pct=9.5)
    result = risk_manager_node(state)
    assert "涨跌停" in result["risk_assessment"]["warnings"][0]

# 3. 港股做空测试
def test_hk_short_selling():
    """港股支持做空"""
    broker = SimulatorBroker()
    result = await broker.place_order(
        OrderRequest("00700", Market.HK_STOCK, OrderSide.SELL, 100, ...)
    )
    assert result.status == OrderStatus.FILLED

# 4. LangGraph 完整流程测试
def test_full_trading_cycle():
    """完整决策流程：研究→分析→风控→执行"""
    graph = build_trading_graph().compile(checkpointer=InMemorySaver())
    result = graph.invoke({
        "trader_id": "test-1",
        "watchlist": ["600519", "00700"],
        "portfolio": {},
    })
    assert result["trade_orders"] is not None
    assert result["risk_assessment"]["approved"] is True

# 5. Human-in-the-Loop 测试
def test_hitl_large_trade_approval():
    """大额交易需要人工审批"""
    graph = build_trading_graph().compile(
        checkpointer=InMemorySaver(),
        interrupt_before=["human_review"],
    )
    config = {"configurable": {"thread_id": "test-hitl"}}
    result = graph.invoke(large_trade_state, config)
    assert "__interrupt__" in str(result)
    # 模拟审批
    result = graph.invoke(Command(resume="approved"), config)
    assert result["execution_results"]
```

### 12.3 覆盖率要求

| 模块 | 最低覆盖率 | 重点 |
|------|-----------|------|
| `agents/` | 80% | 状态转换、路由逻辑 |
| `tools/` | 90% | 数据获取、指标计算 |
| `broker/` | 95% | 交易执行、T+1、涨跌停 |
| `market/` | 85% | 数据解析、缓存 |
| `api/` | 80% | 路由、认证 |

---

## 附录 A: 港A股交易时间表

```
A股交易时间（北京时间）:
  集合竞价: 09:15 - 09:25
  连续竞价: 09:30 - 11:30, 13:00 - 15:00
  盘后固定价格: 15:05 - 15:30（仅科创板/创业板）

港股交易时间（北京时间）:
  开市前时段: 09:00 - 09:30
  持续交易: 09:30 - 12:00, 13:00 - 16:00
  收市竞价: 16:00 - 16:10
```

## 附录 B: LangGraph 配置文件

```json
// langgraph.json
{
  "dependencies": ["."],
  "graphs": {
    "trading": "./src/agents/graph.py:build_trading_graph"
  },
  "env": ".env",
  "python_version": "3.11",
  "pip_config_file": "pip.conf",
  "dockerfile_lines": []
}
```

## 附录 C: 环境变量模板

```bash
# .env.example

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/nofx_v2
REDIS_URL=redis://localhost:6379/0

# LLM
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=sk-...

# Broker
FUTU_HOST=localhost
FUTU_PORT=11111
FUTU_TRADE_ENV=SIMULATE

LONGBRIDGE_APP_KEY=...
LONGBRIDGE_APP_SECRET=...
LONGBRIDGE_ACCESS_TOKEN=...

# Auth
JWT_SECRET=...
```
```
