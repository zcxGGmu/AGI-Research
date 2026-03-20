# VeighNa + FinnewsHunter 融合方案

> **日期**: 2026-03-20
> **状态**: 提案阶段
> **目标**: 将 VeighNa（量化交易执行框架）与 FinnewsHunter（AI 新闻情报平台）融合为 **IntelliTrader** — 一个新闻驱动的智能量化交易系统

---

## 一、项目分析

### 1.1 VeighNa（vnpy）v4.3.0

| 维度 | 详情 |
|------|------|
| **定位** | Python 开源量化交易框架，事件驱动架构 |
| **核心引擎** | `EventEngine`（线程安全队列 pub-sub）→ `MainEngine` → `Gateway` → 交易所 |
| **数据对象** | `TickData`, `BarData`, `OrderData`, `TradeData`, `PositionData`, `AccountData` |
| **插件体系** | `BaseGateway`（35+ 交易所适配器）, `BaseApp`（交易应用）, `BaseDatabase`, `BaseDatafeed` |
| **交易应用** | CTA 策略、组合策略、算法交易、价差交易、期权交易 |
| **Alpha 模块** | `AlphaLab`（Parquet）, `AlphaDataset`（特征工程, Alpha101/158）, `AlphaModel`（Lasso/LightGBM/MLP）, `AlphaStrategy`, `BacktestingEngine` |
| **工具类** | `BarGenerator`（tick→bar 聚合）, `ArrayManager`（30+ TA-Lib 指标） |
| **分布式** | ZeroMQ RPC |
| **GUI** | PySide6 |
| **存储** | SQLite / MySQL / PostgreSQL / MongoDB / TDengine / DolphinDB |

**核心优势**: 成熟的交易执行基础设施、丰富的交易所连接、完善的回测与风控。

### 1.2 FinnewsHunter

| 维度 | 详情 |
|------|------|
| **定位** | 多智能体金融新闻分析与投资决策平台 |
| **后端** | FastAPI + Celery + Redis，基于 AgenticX 框架 |
| **智能体** | `NewsAnalyst`, `BullResearcher`, `BearResearcher`, `InvestmentManager`, `SearchAnalyst` |
| **辩论编排** | `DebateOrchestrator`：`parallel`（并行）, `realtime_debate`（实时辩论）, `quick_analysis`（快速分析） |
| **新闻爬虫** | 10 个中国财经新闻源（新浪、腾讯、网易、东方财富、每经、第一财经、中新经纬、经济观察、财经网、21 世纪） |
| **数据架构** | Provider-Fetcher（OpenBB 风格 TET 管道） |
| **知识图谱** | Neo4j（公司、行业、产品、概念、关键词） |
| **Alpha 挖掘** | Transformer + RL 因子表达式生成，DSL/VM 因子求值 |
| **LLM 支持** | 6 家供应商：百炼（阿里云）, OpenAI, DeepSeek, Kimi, 智谱, Anthropic |
| **存储** | PostgreSQL, Redis, Milvus（向量）, Neo4j（图） |
| **前端** | React + TypeScript + Vite |

**核心优势**: 实时新闻情报获取、多智能体辩论决策、知识图谱、情感因子生成。

### 1.3 互补关系

```
FinnewsHunter                          VeighNa
┌──────────────────┐                   ┌──────────────────┐
│  新闻采集         │                   │  行情接收         │
│  情感分析         │  ──── 融合 ────→  │  策略执行         │
│  多智能体辩论     │                   │  订单管理         │
│  Alpha 因子挖掘   │                   │  风险控制         │
│  知识图谱         │                   │  回测引擎         │
└──────────────────┘                   └──────────────────┘
   情报层（Intelligence）                执行层（Execution）
```

---

## 二、系统架构设计

### 2.1 总体架构

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         IntelliTrader 融合系统                              │
│                                                                            │
│  ┌─────────────────────────────┐    ┌─────────────────────────────────┐   │
│  │   情报层（FinnewsHunter）     │    │    执行层（VeighNa）              │   │
│  │                             │    │                                 │   │
│  │  ┌─────────┐ ┌───────────┐ │    │  ┌──────────┐ ┌─────────────┐ │   │
│  │  │新闻爬虫  │ │知识图谱    │ │    │  │MainEngine│ │EventEngine  │ │   │
│  │  │(10源)   │ │(Neo4j)    │ │    │  │          │ │(pub-sub)    │ │   │
│  │  └────┬────┘ └─────┬─────┘ │    │  └─────┬────┘ └──────┬──────┘ │   │
│  │       │             │       │    │        │             │        │   │
│  │       ▼             ▼       │    │        ▼             │        │   │
│  │  ┌──────────────────────┐   │    │  ┌──────────┐        │        │   │
│  │  │  多智能体辩论         │   │    │  │Gateways  │        │        │   │
│  │  │  Bull ←→ Bear        │   │    │  │(35+交易所)│        │        │   │
│  │  │       ↓              │   │    │  └─────┬────┘        │        │   │
│  │  │  InvestmentManager   │   │    │        │             │        │   │
│  │  └──────────┬───────────┘   │    │        ▼             │        │   │
│  │             │               │    │  ┌──────────┐        │        │   │
│  │  ┌──────────▼───────────┐   │    │  │策略引擎   │        │        │   │
│  │  │  Alpha 挖掘引擎       │   │    │  │CTA/组合/  │        │        │   │
│  │  │  Transformer+RL      │   │    │  │Alpha/算法 │        │        │   │
│  │  │  情感因子             │   │    │  └─────┬────┘        │        │   │
│  │  └──────────┬───────────┘   │    │        │             │        │   │
│  └─────────────┼───────────────┘    └────────┼─────────────┼────────┘   │
│                │                             │             │            │
│  ══════════════╪═════════════════════════════╪═════════════╪════════    │
│                │          桥接层（Bridge）     │             │            │
│                ▼                             ▼             ▼            │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  ┌───────────────┐  ┌──────────────┐  ┌───────────────────────┐ │   │
│  │  │ SignalBridge   │  │ FactorBridge │  │ DataBridge            │ │   │
│  │  │               │  │              │  │                       │ │   │
│  │  │ 辩论结果       │  │ 情感因子      │  │ NewsData→EVENT_NEWS  │ │   │
│  │  │ →EVENT_SIGNAL │  │ →AlphaDataset│  │ MarketData→辩论上下文 │ │   │
│  │  │ →SignalData   │  │              │  │                       │ │   │
│  │  │ →CTA策略      │  │              │  │                       │ │   │
│  │  └───────┬───────┘  └──────┬───────┘  └───────────┬───────────┘ │   │
│  │          │                 │                      │             │   │
│  │          ▼                 ▼                      ▼             │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │               EventEngine（vnpy 事件总线）                │   │   │
│  │  │  EVENT_TICK  EVENT_SIGNAL  EVENT_NEWS  EVENT_FACTOR      │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  │                                                                  │   │
│  │  ┌───────────────┐  ┌──────────────┐  ┌──────────────────────┐ │   │
│  │  │ RiskGate      │  │ SymbolResolver│  │ AuditTrail          │ │   │
│  │  │ 新闻风控叠加   │  │ 公司名→代码   │  │ 全链路审计日志       │ │   │
│  │  └───────────────┘  └──────────────┘  └──────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  存储层                                                          │   │
│  │  PostgreSQL（订单、交易、信号、审计）| Redis（缓存、Celery）        │   │
│  │  Neo4j（知识图谱）| Milvus（向量嵌入）| Parquet（因子数据）        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  展示层                                                          │   │
│  │  React 前端 + PySide6 GUI（可选）+ WebSocket 实时推送              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 架构核心决策

| # | 决策 | 理由 |
|---|------|------|
| 1 | **保留两套运行时**（vnpy 进程内 + FinnewsHunter 分布式） | 最小化对两个系统的改动，各自保留优势 |
| 2 | **Redis 作为跨系统消息总线** | 两个系统都已使用 Redis，低延迟，原生 pub/sub |
| 3 | **桥接层运行在 vnpy 进程内**（Phase 1-2） | 最简部署，直接访问 EventEngine |
| 4 | **双速因子管道**（实时信号 + 批量因子） | 匹配新闻的天然属性：突发→实时，研究→批量 |
| 5 | **PostgreSQL 作为审计存储** | FinnewsHunter 已使用 PG，审计需要关系查询 |
| 6 | **NewsCtaStrategy 作为标准 CtaTemplate 子类** | 充分利用 vnpy 的回测、风控、GUI 生态 |

---

## 三、核心集成点

### 3.1 五大连接缝

| # | 集成点 | FinnewsHunter 侧 | VeighNa 侧 | 桥接机制 |
|---|--------|-------------------|------------|----------|
| 1 | **信号生成** | InvestmentManager 辩论输出 | CtaStrategy / PortfolioStrategy 信号输入 | `SignalBridge` |
| 2 | **因子管道** | 情感因子 + Alpha 挖掘因子 | AlphaDataset 特征工程 + AlphaModel | `FactorBridge` |
| 3 | **行情数据** | akshare 实时数据（辩论上下文） | Gateway tick/bar 数据 | `DataBridge` 双向适配 |
| 4 | **知识增强** | Neo4j 图谱（公司、行业、概念） | 策略参数调优、股票池筛选 | 只读查询 API |
| 5 | **事件总线** | Celery 异步任务 + Redis pub/sub | EventEngine 队列 pub/sub | `EventBridge` 翻译层 |

### 3.2 运行时不对称性（关键挑战）

两个系统的运行模型根本不同：

- **VeighNa**: 同步、进程内、单线程事件循环，所有组件共享一个 Python 进程，通过线程安全队列通信
- **FinnewsHunter**: 分布式、多进程、异步，FastAPI + Celery + Redis，组件是独立服务

**解决方案**: `SignalBridge` 在 vnpy 进程内运行一个专用 Redis 订阅线程。收到 FinnewsHunter 消息时，构造 vnpy `Event` 对象并调用 `event_engine.put(event)` 安全入队。反方向通过 Redis publish。

---

## 四、端到端数据流

### 4.1 实时路径（延迟目标: < 30 秒）

```
新闻采集 → 快速分类 → 信号桥接 → 事件注入 → 策略决策 → 风控门 → 订单执行 → 审计
```

详细流程：

```
Step 1: 新闻采集（FinnewsHunter）
│  爬虫在新浪/东方财富等检测到突发新闻
│  → 发布到 Redis channel: "news:raw"
│  → Celery 任务: news_analysis.classify
│
Step 2: 快速分类（FinnewsHunter）
│  quick_analysis 模式（无完整辩论）
│  NewsAnalyst 提取: {company, sentiment, urgency, impact_score}
│  若 impact_score > 阈值 → 触发完整辩论（异步）
│  若 urgency == "critical" → 快速路径信号
│
Step 3: 信号桥接（Bridge Layer）
│  从 Redis 接收已分类新闻事件
│  通过知识图谱将公司名 → 股票代码（如 "贵州茅台" → "600519.SSE"）
│  构造 NewsSignalData
│
Step 4: 事件注入（Bridge → VeighNa）
│  SignalBridge.emit(Event(EVENT_NEWS_SIGNAL, data=NewsSignalData))
│  → EventEngine 分发给所有已注册处理器
│
Step 5: 策略决策（VeighNa）
│  NewsCtaStrategy.on_news_signal(signal):
│    - 检查当前持仓
│    - 结合技术指标（ArrayManager）
│    - 应用置信度阈值
│    - 满足条件则生成 OrderRequest
│
Step 6: 风控门（VeighNa 增强）
│  NewsAwareRiskManager.check_order(order):
│    - 持仓限制检查
│    - 回撤保护
│    - 相关性检查（避免单一板块 all-in）
│    - 新闻时效性检查（>5 分钟拒绝）
│
Step 7: 订单执行（VeighNa）
│  MainEngine.send_order() → Gateway → 交易所
│  ORDER → TRADE → POSITION 回流 EventEngine
│
Step 8: 审计（Bridge Layer）
│  AuditTrail 捕获完整链路:
│  news_id → signal_id → order_id → trade_id
│  全部持久化到 PostgreSQL
```

### 4.2 批量路径（隔夜周期）

```
Step 1: 每夜新闻聚合
│  对当天所有新闻进行完整辩论（parallel 模式）
│
Step 2: 情感因子生成
│  FactorBridge 提取每只股票的日度情感分数
│  → 作为特征写入 AlphaDataset（Parquet）
│
Step 3: Alpha 模型重训练（定期）
│  AlphaModel（LightGBM/MLP）以情感特征重训练
│  → 更新模型权重
│
Step 4: 次日信号生成
│  AlphaStrategy.on_bar():
│    - 加载更新后的模型
│    - 生成持仓目标
│    - 组合再平衡
```

---

## 五、桥接层数据模型

### 5.1 新增事件类型

```python
# bridge/event_types.py
# 遵循 vnpy EVENT_ 前缀命名规范

EVENT_NEWS_RAW = "eNewsRaw"           # 原始新闻到达
EVENT_NEWS_CLASSIFIED = "eNewsClass"   # 新闻已分类（含情感）
EVENT_NEWS_SIGNAL = "eNewsSignal"      # 来自新闻分析的交易信号
EVENT_DEBATE_RESULT = "eDebateResult"  # 完整辩论结果
EVENT_FACTOR_UPDATE = "eFactorUpdate"  # 新因子值可用
EVENT_RISK_ALERT = "eRiskAlert"        # 风控门触发
EVENT_AUDIT_LOG = "eAuditLog"          # 审计日志条目
```

### 5.2 核心数据对象

```python
# bridge/data_objects.py
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
from enum import Enum


class SignalDirection(Enum):
    LONG = "long"
    SHORT = "short"
    NEUTRAL = "neutral"
    EXIT_LONG = "exit_long"
    EXIT_SHORT = "exit_short"


class NewsUrgency(Enum):
    CRITICAL = "critical"     # 市场级别，立即行动
    HIGH = "high"             # 重要，数分钟内行动
    MEDIUM = "medium"         # 值得关注，批量处理可接受
    LOW = "low"               # 背景信息


@dataclass
class NewsEventData:
    """流入桥接层的原始新闻事件"""
    news_id: str
    title: str
    content: str
    source: str               # "sina", "eastmoney", etc.
    publish_time: datetime
    symbols: list[str]        # 提取的股票代码
    industries: list[str]     # 知识图谱关联的行业
    urgency: NewsUrgency
    raw_sentiment: float      # -1.0 到 1.0
    gateway_name: str = ""


@dataclass
class DebateResultData:
    """FinnewsHunter 多智能体辩论输出"""
    debate_id: str
    news_id: str
    mode: str                 # "parallel", "realtime_debate", "quick_analysis"
    bull_score: float         # 0-100
    bear_score: float         # 0-100
    consensus_direction: SignalDirection
    confidence: float         # 0-100
    key_arguments: dict       # {"bull": [...], "bear": [...]}
    investment_decision: dict  # InvestmentManager 结构化输出
    duration_ms: int
    llm_provider: str
    timestamp: datetime


@dataclass
class NewsSignalData:
    """
    新闻驱动的交易信号 — 流入 vnpy EventEngine 的主要对象
    设计为可被 CtaStrategy / PortfolioStrategy 消费
    """
    signal_id: str
    symbol: str               # vnpy 格式: "600519.SSE"
    exchange: str             # "SSE", "SZSE"
    direction: SignalDirection
    confidence: float         # 0-100

    # 交易建议
    target_price: Optional[float] = None
    stop_loss: Optional[float] = None
    take_profit: Optional[float] = None
    position_pct: Optional[float] = None  # 建议仓位占权益百分比

    # 溯源链
    news_id: str = ""
    debate_id: str = ""
    source_type: str = ""     # "quick_analysis" 或 "full_debate"
    news_summary: str = ""

    # 因子数据
    sentiment_score: float = 0.0
    impact_score: float = 0.0

    # 时间
    news_time: Optional[datetime] = None
    signal_time: datetime = field(default_factory=datetime.now)

    @property
    def age_seconds(self) -> float:
        """信号时效性检查"""
        if self.news_time:
            return (self.signal_time - self.news_time).total_seconds()
        return 0.0


@dataclass
class FactorUpdateData:
    """情感/新闻因子值，用于 Alpha 管道集成"""
    symbol: str
    date: str                 # "2026-03-20"
    factors: dict             # {"sentiment_score": 0.75, "news_count_24h": 12, ...}
    timestamp: datetime = field(default_factory=datetime.now)


@dataclass
class AuditEntry:
    """全链路审计: 新闻 → 信号 → 订单 → 成交"""
    entry_id: str
    news_id: Optional[str]
    debate_id: Optional[str]
    signal_id: Optional[str]
    order_id: Optional[str]
    trade_id: Optional[str]
    event_type: str           # "news_received", "signal_generated", "order_sent" 等
    details: dict
    timestamp: datetime = field(default_factory=datetime.now)
```

### 5.3 配置模型

```python
# bridge/config.py
from dataclasses import dataclass


@dataclass
class BridgeConfig:
    """桥接层统一配置"""

    # FinnewsHunter 连接
    finnews_redis_url: str = "redis://localhost:6379/0"
    finnews_api_url: str = "http://localhost:8000"
    celery_broker_url: str = "redis://localhost:6379/1"

    # VeighNa 连接
    vnpy_rpc_host: str = "localhost"
    vnpy_rpc_port: int = 2014
    vnpy_in_process: bool = True       # True=共享 EventEngine

    # 信号生成
    quick_analysis_urgency_threshold: float = 0.7
    full_debate_impact_threshold: float = 0.5
    signal_confidence_min: float = 60.0
    signal_staleness_max_seconds: float = 300.0  # 5 分钟

    # 因子管道
    sentiment_factor_weight: float = 0.3
    factor_update_interval_minutes: int = 30

    # 风控叠加
    max_news_driven_position_pct: float = 5.0    # 单信号最大 5% 权益
    max_correlated_news_exposure: float = 15.0   # 同板块最大 15%
    news_cooldown_seconds: float = 60.0          # 同标的最小间隔

    # 知识图谱
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = ""

    # 审计
    audit_db_url: str = "postgresql://localhost:5432/intellitrader"

    # 存储
    parquet_base_path: str = "./data/factors"
```

---

## 六、关键适配器设计

### 6.1 SignalBridge（核心适配器）

```python
class SignalBridge:
    """
    将 FinnewsHunter 辩论结果翻译为 vnpy EventEngine 事件。

    职责:
    1. 监听 Redis channel 接收 FinnewsHunter 输出
    2. 通过知识图谱查找 公司名 → 股票代码
    3. 构造 NewsSignalData 对象
    4. 注入 vnpy EventEngine 的 EVENT_NEWS_SIGNAL
    5. 记录全链路审计日志
    """
    def __init__(self, event_engine, config: BridgeConfig): ...
    def start(self) -> None: ...
    def stop(self) -> None: ...

    def on_debate_result(self, result: DebateResultData) -> None: ...
    def on_quick_analysis(self, news: NewsEventData) -> None: ...
    def resolve_symbols(self, companies: list[str]) -> list[str]: ...
    def emit_signal(self, signal: NewsSignalData) -> None: ...
```

### 6.2 FactorBridge

```python
class FactorBridge:
    """
    将情感和新闻因子注入 vnpy AlphaDataset。
    批量模式运行：定期聚合情感分数并写入 Parquet。
    """
    def compute_sentiment_factors(self, symbol: str, date: str) -> dict: ...
    def compute_news_flow_factors(self, symbol: str, date: str) -> dict: ...
    def update_alpha_dataset(self, factors: list[FactorUpdateData]) -> None: ...
    def run_batch_update(self) -> None: ...
```

### 6.3 DataBridge

```python
class DataBridge:
    """
    双向数据归一化:
    - vnpy TickData/BarData → FinnewsHunter 辩论上下文
    - FinnewsHunter akshare 数据 → vnpy DataFeed 格式
    """
    def get_market_context_for_debate(self, symbols: list[str]) -> dict: ...
    def query_bar_data(self, symbol, exchange, interval, start, end) -> list: ...
```

### 6.4 NewsCtaStrategy（新增 vnpy 策略）

```python
from vnpy_ctastrategy import CtaTemplate

class NewsCtaStrategy(CtaTemplate):
    """
    消费 tick 数据与新闻信号的 CTA 策略。
    结合技术指标与情感驱动入场。
    """
    signal_confidence_threshold: float = 70.0
    technical_confirmation_required: bool = True
    position_pct_per_signal: float = 3.0
    max_concurrent_news_trades: int = 3
    signal_staleness_max: float = 300.0

    def on_news_signal(self, signal: NewsSignalData) -> None:
        """
        收到 EVENT_NEWS_SIGNAL 时的决策逻辑:
        1. 检查信号时效性
        2. 检查置信度阈值
        3. 若需技术确认，检查 TA 指标
        4. 计算仓位大小
        5. 通过风控门发送订单
        """
        ...
```

### 6.5 NewsAwareRiskManager

```python
class NewsAwareRiskManager:
    """
    扩展 vnpy 内置风控，增加新闻特定检查:
    - 信号时效性拒绝
    - 相关新闻敞口限制
    - 新闻驱动仓位冷却
    - 基于知识图谱的板块集中度限制
    """
    def check_news_signal(self, signal: NewsSignalData) -> tuple[bool, str]: ...
    def check_sector_exposure(self, symbol: str, direction: str) -> tuple[bool, str]: ...
    def check_cooldown(self, symbol: str) -> tuple[bool, str]: ...
```

---

## 七、部署拓扑

### Phase 1-2: 单机部署

```
┌──────────────────────────────────────────────────────────┐
│                      单主机                               │
│                                                          │
│  ┌───────────────┐       ┌───────────────┐              │
│  │ 进程 1:        │       │ 进程 2:        │              │
│  │ VeighNa Main  │       │ FinnewsHunter │              │
│  │               │       │ FastAPI        │              │
│  │ EventEngine   │◄─ZMQ─►│ + Celery       │              │
│  │ + Bridge      │       │ + 新闻爬虫     │              │
│  │ + Gateways    │       │ + 智能体辩论   │              │
│  │ + Strategies  │       │               │              │
│  └───────┬───────┘       └───────┬───────┘              │
│          │                       │                      │
│  ┌───────▼───────────────────────▼───────┐              │
│  │ PostgreSQL | Redis | Neo4j | Milvus   │              │
│  └───────────────────────────────────────┘              │
│                                                          │
│  ┌───────────────┐                                      │
│  │ React Frontend│                                      │
│  └───────────────┘                                      │
└──────────────────────────────────────────────────────────┘
```

### Phase 3+: 分布式部署

```
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│ Node A      │  │ Node B      │  │ Node C      │  │ Node D      │
│ VeighNa    │  │ FinnewsH   │  │ 数据库集群   │  │ 前端 + API  │
│ 执行+Bridge │  │ 情报+爬虫   │  │ PG+Redis   │  │ nginx      │
│             │  │ +辩论       │  │ +Neo4j     │  │            │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       └────────────────┴────────────────┴────────────────┘
                     ZeroMQ TCP / 内网
```

---

## 八、风险分析

| 风险 | 严重度 | 可能性 | 缓解措施 |
|------|--------|--------|----------|
| **EventEngine 线程 bug** 导致信号丢失或死锁 | 严重 | 中 | Bridge 线程隔离; `queue.put()` 线程安全; 全面集成测试 |
| **新闻信号基于过时信息**导致不利交易 | 严重 | 高 | Bridge 和策略双重时效检查; 可配置最大延迟; 交易时段过滤 |
| **股票代码解析错误**导致交易错误标的 | 严重 | 中 | 必须对交易所上市列表做代码验证; 低置信度时人工介入 |
| **LLM 幻觉**在辩论中产生虚假信号 | 高 | 中 | 置信度阈值过滤; 要求技术确认; 新闻信号仓位限制 |
| **Celery/Redis 故障**中断信号管道 | 高 | 低 | 健康监控; 降级轮询 FinnewsHunter API; 死信队列恢复 |
| **A 股 T+1 违规** | 高 | 低 | 风控门集成 T1Manager; 下单前二次校验 |
| **回测过拟合**历史情感数据 | 中 | 高 | 滚动窗口验证; 样本外测试; 因子衰减分析 |
| **知识图谱查询**影响热路径性能 | 中 | 中 | Redis 缓存 KG 查询（1 小时 TTL）; 预计算代码映射表 |

---

## 九、分阶段实施计划

### Phase 1: 基础搭建（第 1-3 周）

**目标**: 建立桥接层骨架，完成最小可行集成。

- [ ] 定义所有桥接数据对象（`NewsSignalData`, `DebateResultData` 等）
- [ ] 定义新增 `EVENT_*` 常量，兼容 vnpy 事件系统
- [ ] 实现 `SignalBridge`：Redis 订阅 → EventEngine 注入
- [ ] 实现 `SymbolResolver`：静态映射表（知识图谱集成延后）
- [ ] 编写最小化 `NewsCtaStrategy`：接收并日志记录信号，不交易
- [ ] 创建统一配置 schema（YAML）
- [ ] 搭建 PostgreSQL 审计表
- [ ] **集成测试**: FinnewsHunter 发布到 Redis → Bridge → vnpy EventEngine → Strategy 收到信号

**交付物**: 端到端信号流打通，无实际下单。

### Phase 2: 交易集成（第 4-6 周）

**目标**: 新闻信号通过 vnpy Gateway 触发真实订单。

- [ ] 实现 `NewsCtaStrategy` 完整下单逻辑
- [ ] 实现 `NewsAwareRiskManager`（时效、冷却、板块敞口）
- [ ] 实现 `DataBridge`：vnpy 行情数据注入辩论上下文
- [ ] 增加 A 股 T+1 感知到风控门
- [ ] 接通审计链路（新闻 → 信号 → 订单 → 成交）
- [ ] 前端增加 WebSocket 信号/订单实时推送
- [ ] **模拟交易验证**

**交付物**: 模拟交易运行，完整审计链路。

### Phase 3: 因子管道（第 7-9 周）

**目标**: 情感因子集成到 vnpy Alpha 研究管道。

- [ ] 实现 `FactorBridge`：情感因子计算
- [ ] 输出兼容 vnpy `AlphaLab` 格式的 Parquet
- [ ] 在 `AlphaDataset` 中增加情感特征
- [ ] 使用情感因子重训练 `AlphaModel`
- [ ] 实现结合技术+情感模型的 `AlphaStrategy`
- [ ] 回测基础设施：历史信号回放
- [ ] **对比**: 有/无新闻因子的策略表现

**交付物**: 情感因子 Alpha 的量化证据。

### Phase 4: 知识图谱增强（第 10-11 周）

**目标**: 深度集成 Neo4j，实现更智能的代码解析与板块分析。

- [ ] `SymbolResolver` 接入 Neo4j 实体解析
- [ ] 基于知识图谱的板块/行业敞口追踪
- [ ] 概念驱动的信号聚类（如一条"锂电"新闻 → 多只标的信号）
- [ ] 图谱关系丰富辩论上下文

**交付物**: 单条新闻事件生成多标的信号。

### Phase 5: 生产加固（第 12-14 周）

**目标**: 可靠性、可观测性、分布式部署。

- [ ] 所有桥接通信路径增加熔断器
- [ ] 健康检查与死信队列
- [ ] Prometheus 指标（信号延迟、辩论耗时、成交率）
- [ ] Grafana 仪表盘
- [ ] ZeroMQ RPC 分布式部署
- [ ] 压力测试：模拟 100+ 新闻事件/小时
- [ ] 故障演练：Redis 宕机、Neo4j 宕机、网络分区

**交付物**: 具备可观测性的生产就绪系统。

---

## 十、项目结构建议

```
intellitrader/
├── bridge/                      # 桥接层（新增核心）
│   ├── __init__.py
│   ├── config.py                # BridgeConfig
│   ├── event_types.py           # EVENT_NEWS_SIGNAL 等
│   ├── data_objects.py          # NewsSignalData, DebateResultData 等
│   ├── signal_bridge.py         # 辩论结果 → vnpy 事件
│   ├── factor_bridge.py         # 情感因子 → AlphaDataset
│   ├── data_bridge.py           # 双向数据归一化
│   ├── symbol_resolver.py       # 公司名/新闻 → 股票代码
│   ├── risk_gate.py             # NewsAwareRiskManager
│   └── audit_trail.py           # 全链路审计
├── strategies/                  # vnpy 策略（新增）
│   ├── news_cta_strategy.py     # 新闻驱动 CTA 策略
│   └── news_alpha_strategy.py   # 情感+技术 Alpha 策略
├── finnewshunter/               # FinnewsHunter 源码（git submodule 或 symlink）
│   └── backend/...
├── vnpy/                        # VeighNa 源码（git submodule 或 symlink）
│   └── ...
├── deploy/
│   ├── docker-compose.yml       # 全部基础设施
│   └── Dockerfile.*
├── tests/
│   ├── test_signal_bridge.py
│   ├── test_factor_bridge.py
│   ├── test_symbol_resolver.py
│   └── test_integration.py
├── config/
│   └── intellitrader.yaml       # 统一配置
└── README.md
```

---

## 十一、总结

本方案通过 **桥接层（Bridge Layer）** 将 FinnewsHunter 的新闻情报能力与 VeighNa 的交易执行能力融合，形成完整的 **新闻 → 分析 → 信号 → 交易 → 审计** 闭环。

关键设计原则：
1. **最小侵入**: 不修改两个原系统的核心代码，通过桥接层适配
2. **双速管道**: 突发新闻走实时路径（<30s），研究分析走批量路径（隔夜）
3. **渐进式交付**: 5 个阶段，每个阶段独立交付价值
4. **安全优先**: 多层风控（时效性、板块集中度、仓位限制、T+1 感知）
5. **可追溯**: 全链路审计，每笔交易可溯源到触发它的新闻
