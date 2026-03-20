# vnpy + FinnewsHunter 融合方案

> 文档版本: v1.2
> 生成日期: 2026-03-20
> 更新日期: 2026-03-20 (完善 11-16 章节: 部署、监控、测试、安全、容错、性能)
> 融合目标: 构建**舆情驱动量化交易系统 (Sentiment-Alpha Trader)**

---

## 1. 项目分析与定位

### 1.1 两个项目的核心价值

| 维度 | vnpy (VeighNa) | FinnewsHunter |
|------|----------------|---------------|
| **定位** | 量化交易执行框架 | 金融新闻智能分析系统 |
| **核心能力** | 交易引擎、订单管理、风控、Alpha回测 | 新闻采集、舆情分析、多智能体辩论 |
| **数据模型** | TickData, BarData, OrderData, TradeData | News, Analysis, DebateResult |
| **智能体** | 无 | Bull/Bear/Manager 多智能体辩论 |
| **技术栈** | Python, EventEngine, PySide6 | FastAPI, AgenticX, PostgreSQL, Milvus, Neo4j |
| **优势** | 完善的风控、持仓管理、多交易所接口 | LLM驱动的舆情分析、因子DSL、知识图谱 |

### 1.2 融合价值

**当前痛点**:
- vnpy 有完整的交易执行能力，但缺乏**新闻/舆情驱动的Alpha信号生成**
- FinnewsHunter 有强大的舆情分析能力，但缺乏**信号到交易的闭环**

**融合后系统能力**:
```
新闻事件 → 舆情分析 → Alpha信号 → 量化策略 → 交易执行 → 持仓风控
     ↑                                                              ↓
     ←———————————————————————— 反馈优化 —————————————————————————←
```

---

## 2. 融合架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           舆情驱动量化交易系统                                │
│                    (Sentiment-Alpha Trading System)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      数据源层 (Data Sources)                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│  │  │  财经   │  │  新闻   │  │  K线    │  │  资金   │  │  宏观   │   │   │
│  │  │  新闻   │  │  社交   │  │  行情   │  │  流向   │  │  数据   │   │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘   │   │
│  └───────┼────────────┼────────────┼────────────┼────────────┼────────┘   │
│          │            │            │            │            │              │
│          ▼            ▼            ▼            ▼            ▼              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      事件中枢 (EventBridge)                            │   │
│  │  ┌────────────────────────────────────────────────────────────────┐  │   │
│  │  │                    vnpy EventEngine                               │  │   │
│  │  │   EVENT_TICK │ EVENT_TRADE │ EVENT_ORDER │ EVENT_POSITION      │  │   │
│  │  │   EVENT_ACCOUNT │ EVENT_SENTIMENT │ EVENT_ALPHA_SIGNAL       │  │   │
│  │  └────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│          ┌─────────────────────────┼─────────────────────────┐           │
│          ▼                         ▼                         ▼           │
│  ┌───────────────┐        ┌───────────────┐        ┌───────────────┐      │
│  │   新闻分析域   │        │   交易执行域   │        │   Alpha挖掘域  │      │
│  │ (FinnewsHunter)│        │    (vnpy)     │        │  (融合新增)   │      │
│  ├───────────────┤        ├───────────────┤        ├───────────────┤      │
│  │ NewsCollector │        │ MainEngine    │        │AlphaSignal   │      │
│  │ NewsAnalyst   │        │ OmsEngine     │        │  Engine      │      │
│  │ BullAgent     │        │ OffsetConvert │        │Sentiment     │      │
│  │ BearAgent     │        │ RiskManager   │        │  Calculator  │      │
│  │ ManagerAgent  │        │ Gateways      │        │FactorVM      │      │
│  │ EmbeddingSvc  │        │               │        │              │      │
│  │ Neo4jGraph    │        │               │        │              │      │
│  └───────────────┘        └───────────────┘        └───────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块职责划分

#### **保留模块 (各自独立运行)**

| 归属 | 模块 | 职责 |
|------|------|------|
| vnpy | `event/engine.py` | 事件驱动核心引擎 |
| vnpy | `trader/engine.py` | 主引擎、Gateway管理 |
| vnpy | `trader/gateway.py` | 交易接口抽象 |
| vnpy | `alpha/*` | Alpha投研模块 |
| FinnewsHunter | `agents/*` | 多智能体辩论系统 |
| FinnewsHunter | `services/llm_service.py` | LLM多Provider封装 |
| FinnewsHunter | `tools/crawler_base.py` | 爬虫框架 |
| FinnewsHunter | `financial/registry.py` | 数据源Provider注册 |
| FinnewsHunter | `knowledge/graph_service.py` | 知识图谱服务 |

#### **融合新增模块**

| 模块 | 路径 | 职责 |
|------|------|------|
| `SentimentEngine` | `src/alpha/sentiment_engine.py` | 舆情信号计算 |
| `EventBridge` | `src/bridge/event_bridge.py` | 双系统事件互通 |
| `AlphaSignalApp` | `src/alpha/app.py` | Alpha信号App (vnpy风格) |
| `SentimentGateway` | `src/gateways/sentiment_gateway.py` | 舆情数据Gateway |
| `SignalStrategy` | `src/strategies/signal_strategy.py` | 信号驱动策略模板 |

### 2.3 核心交互流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         舆情 → 信号 → 交易 流程                          │
└──────────────────────────────────────────────────────────────────────────┘

[阶段1: 新闻采集与分析]
┌─────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  新闻源  │───▶│NewsCollector │───▶│NewsAnalyst   │───▶│Sentiment     │
│         │    │              │    │  Agent       │    │Calculator    │
└─────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                                 │
                                                                 ▼
[阶段2: Alpha信号生成]                                    ┌──────────────┐
┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │ EVENT_       │
│FactorDataset │◀───│AlphaSignal   │◀───│Sentiment     │   │ SENTIMENT   │
│(vnpy alpha)  │    │Engine        │    │Score         │   └──────┬───────┘
└──────┬───────┘    └──────────────┘    └──────────────┘          │
       │                                                             │
       ▼                                                             ▼
[阶段3: 信号触发]                      ┌──────────────┐    ┌──────────────┐
┌──────────────┐    ┌──────────────┐   │SignalStrategy│◀───│ EventBridge │
│FactorVM      │───▶│AlphaSignal   │───▶│              │    │             │
│(vnpy alpha)  │    │Engine        │   │              │    │             │
└──────────────┘    └──────┬───────┘   └──────────────┘    └──────┬───────┘
                           │                                        │
                           ▼                                        ▼
[阶段4: 交易执行]           ┌──────────────────────────────────────────────┐
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐
│OrderRequest  │───▶│MainEngine    │───▶│Gateway       │───▶│  Exchange │
│              │    │(vnpy)        │    │              │    │           │
└──────────────┘    └──────────────┘    └──────────────┘    └────────────┘
```

---

## 3. 技术融合方案

### 3.1 事件桥接设计 (EventBridge)

EventBridge 是融合系统的核心枢纽，负责连接 vnpy 的同步事件引擎与 FinnewsHunter 的异步智能体系统。

#### 3.1.1 事件类型扩展

```python
# event_bridge/types.py

class EventSource(Enum):
    """事件来源枚举"""
    VNPY = "vnpy"           # 来自 vnpy 交易引擎
    FINNEWS = "finnews"     # 来自 FinnewsHunter
    EXTERNAL = "external"   # 外部系统
    BRIDGE = "bridge"       # 桥接层内部生成

class EventPriority(Enum):
    """事件优先级"""
    LOW = 0
    NORMAL = 1
    HIGH = 2
    CRITICAL = 3

# vnpy 原生事件类型 (保持兼容)
EVENT_TICK = "eTick."
EVENT_TRADE = "eTrade."
EVENT_ORDER = "eOrder."
EVENT_POSITION = "ePosition."
EVENT_ACCOUNT = "eAccount."

# 扩展事件类型 - AI/量化分析相关
EVENT_SENTIMENT = "eSentiment"           # 情感分析结果
EVENT_SENTIMENT_REQUEST = "eSentimentRequest"  # 情感分析请求
EVENT_ALPHA_SIGNAL = "eAlphaSignal"     # Alpha 信号
EVENT_NEWS = "eNews"                    # 新闻数据
EVENT_DEBATE_START = "eDebateStart"    # 辩论开始
EVENT_DEBATE_COMPLETE = "eDebateComplete"  # 辩论完成
EVENT_DECISION = "eDecision"            # 投资决策
EVENT_RISK_ALERT = "eRiskAlert"         # 风险告警

@dataclass
class BridgeEvent:
    """桥接层统一事件格式"""
    event_id: str                    # 唯一ID (UUID)
    event_type: str                  # 事件类型
    source: EventSource               # 事件来源
    data: Any                        # 事件数据
    priority: EventPriority = EventPriority.NORMAL
    timestamp: datetime = field(default_factory=datetime.now)
    origin_event: Optional[VnpyEvent] = None  # 原始 vnpy 事件引用
    hop_count: int = 0               # 跨越系统次数
    ttl: int = 10                   # 最大传播 hops
    processed: bool = False
    context: Dict[str, Any] = field(default_factory=dict)
```

#### 3.1.2 核心类实现

```python
# event_bridge/core.py

class EventBridge:
    """
    EventBridge 核心类
    实现 vnpy EventEngine 与 FastAPI/AgenticX 之间的双向事件桥接
    """

    def __init__(
        self,
        name: str = "EventBridge",
        vnpy_engine: Optional[EventEngine] = None,
        max_queue_size: int = 1000,
        enable_metrics: bool = True
    ):
        self.name = name
        self._vnpy_engine = vnpy_engine
        self._owns_engine = vnpy_engine is None

        # 线程安全
        self._lock = Lock()
        self._handlers_lock = Lock()

        # vnpy 端
        self._vnpy_engine_running = False
        self._vnpy_to_bridge_queue: Queue = Queue(maxsize=max_queue_size)

        # FastAPI/异步端
        self._async_loop: Optional[AsyncEventLoop] = None

        # 处理器注册
        self._vnpy_handlers: Dict[str, List[HandlerRegistration]] = defaultdict(list)
        self._fastapi_handlers: Dict[str, List[HandlerRegistration]] = defaultdict(list)
        self._bidirectional_mappings: Dict[str, str] = {}

        # 过滤器与限流
        self._event_filter = EventFilter()
        self._rate_limiter: Optional[RateLimiter] = None
        self._event_coalescer = EventCoalescer(window_seconds=0.5, max_batch_size=5)

        # 指标
        self._metrics = EventMetrics() if enable_metrics else None

        # LLM 服务 (懒加载)
        self._llm_service_getter: Optional[Callable[[], Any]] = None

        # 运行状态
        self._running = False

    def start(self):
        """启动 EventBridge"""
        self._running = True

        # 启动异步事件循环
        self._async_loop = AsyncEventLoop(
            name=f"{self.name}-AsyncLoop",
            max_queue_size=1000
        )
        self._async_loop.start()

        # 启动桥接转发线程
        self._start_bridge_thread()

        logger.info(f"{self.name} started")

    def register_vnpy_handler(
        self,
        event_type: str,
        handler: Callable[[VnpyEvent], None],
        priority: int = 0,
        bidirectional: bool = False,
        target_fastapi_type: Optional[str] = None
    ):
        """注册 vnpy 端事件处理器"""
        with self._handlers_lock:
            reg = HandlerRegistration(
                event_type=event_type,
                handler=handler,
                direction=BridgeDirection.VNPY_TO_FASTAPI,
                priority=priority
            )
            self._vnpy_handlers[event_type].append(reg)

            if self._vnpy_engine:
                self._vnpy_engine.register(event_type, handler)

            if bidirectional and target_fastapi_type:
                self._bidirectional_mappings[event_type] = target_fastapi_type

    def register_fastapi_handler(
        self,
        event_type: str,
        handler: Callable[[BridgeEvent], Any],
        priority: int = 0,
        is_async: bool = False
    ):
        """注册 FastAPI 端事件处理器"""
        with self._handlers_lock:
            reg = HandlerRegistration(
                event_type=event_type,
                handler=handler,
                direction=BridgeDirection.FASTAPI_TO_VNPY,
                priority=priority
            )
            self._fastapi_handlers[event_type].append(reg)

            if self._async_loop:
                if is_async:
                    async_handler = AsyncEventHandler(
                        name=f"Async_{event_type}",
                        async_handler=handler
                    )
                else:
                    async_handler = SyncToAsyncBridge(
                        name=f"Sync_{event_type}",
                        sync_handler=handler
                    )
                self._async_loop.register_handler(event_type, async_handler)

    def emit_to_fastapi(
        self,
        event_type: str,
        data: Any,
        priority: EventPriority = EventPriority.NORMAL,
        **context
    ) -> BridgeEvent:
        """发送事件到 FastAPI/FinnewsHunter 端"""
        event = BridgeEvent(
            event_id=str(uuid.uuid4()),
            event_type=event_type,
            source=EventSource.VNPY,
            data=data,
            priority=priority,
            context=context
        )

        if self._async_loop:
            self._async_loop.put(event)

        if self._metrics:
            self._metrics.inc("events_emitted_to_fastapi")

        return event

    def emit_to_vnpy(
        self,
        event_type: str,
        data: Any,
        priority: EventPriority = EventPriority.NORMAL
    ):
        """发送事件到 vnpy 端"""
        vnpy_event = VnpyEvent(type=event_type, data=data)

        if self._vnpy_engine:
            self._vnpy_engine.put(vnpy_event)

        return vnpy_event
```

#### 3.1.3 事件过滤与限流

```python
# event_bridge/filter.py

class RateLimiter:
    """基于令牌桶的事件限流器"""

    def __init__(self, config: RateLimitConfig):
        self._config = config
        self._buckets: Dict[str, deque] = defaultdict(lambda: deque(maxlen=config.max_events))
        self._lock = Lock()

    def is_allowed(self, event: BridgeEvent) -> bool:
        """检查事件是否允许通过"""
        key = self._get_event_key(event)
        now = time.time()

        with self._lock:
            bucket = self._buckets[key]
            priority_multiplier = self._get_priority_multiplier(event)
            effective_max = int(self._config.max_events * priority_multiplier)

            if len(bucket) >= effective_max:
                if bucket and now - bucket[0] < self._config.window_seconds:
                    return False

            bucket.append(now)
            return True

class EventFilter:
    """事件过滤器 - 支持白名单/黑名单、去重"""

    def __init__(self):
        self._whitelist: Set[str] = set()
        self._blacklist: Set[str] = set()
        self._custom_filters: List[Callable[[BridgeEvent], bool]] = []
        self._seen_events: Dict[str, datetime] = {}
        self._dedup_window: float = 1.0

    def should_process(self, event: BridgeEvent) -> bool:
        """判断事件是否应该被处理"""
        # 黑名单检查
        if event.event_type in self._blacklist:
            return False

        # 白名单检查
        if self._whitelist and event.event_type not in self._whitelist:
            return False

        # 自定义过滤器
        for filter_func in self._custom_filters:
            if not filter_func(event):
                return False

        # 去重检查
        if self._is_duplicate(event):
            return False

        return True
```

#### 3.1.4 线程安全设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          线程安全架构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐         ┌─────────────────┐                       │
│  │  vnpy EventLoop │         │  AsyncEventLoop │                       │
│  │    (主线程)      │         │    (独立线程)    │                       │
│  └────────┬────────┘         └────────┬────────┘                       │
│           │                             │                                │
│           │ Queue (线程安全)              │ asyncio.Queue (协程安全)       │
│           │                             │                                │
│           ▼                             ▼                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    BridgeEventQueue                               │   │
│  │                    (Python Queue, 线程安全)                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      EventBridge                                  │   │
│  │  - _lock: Lock (保护共享状态)                                     │   │
│  │  - _handlers_lock: Lock (保护处理器注册)                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

| 场景 | 解决方案 | 说明 |
|------|----------|------|
| vnpy -> FastAPI | `queue.Queue` | Python 内置线程安全队列 |
| FastAPI 内部 | `asyncio.Queue` | 协程安全的异步队列 |
| 共享状态修改 | `threading.Lock` | 保护 `_handlers` 等共享数据 |
| 事件对象传递 | **不可变复制** | `BridgeEvent.with_context()` 返回新实例 |
| 跨线程调用 | `run_in_executor` | 将同步调用放到线程池执行 |

### 3.2 舆情信号计算 (SentimentEngine)

SentimentEngine 负责将 FinnewsHunter 的新闻舆情数据转换为 vnpy Alpha 模块可用的交易信号。

#### 3.2.1 核心数据结构

```python
# vnpy/alpha/sentiment/data.py

class SentimentType(Enum):
    """情感倾向枚举"""
    BULLISH = "bullish"    # 看多
    BEARISH = "bearish"    # 看空
    NEUTRAL = "neutral"    # 中性

@dataclass(slots=True)
class SentimentData:
    """
    标准化舆情数据容器

    Attributes:
        datetime: 数据时间戳
        vt_symbol: 合约代码 (如 "600519.SS")
        sentiment_score: 情感评分 (-1 到 1, 负=悲观, 正=乐观)
        confidence: 置信度 (0 到 1)
        news_count: 当日关联新闻数量
        source: 新闻来源 (sina, jrj, cnstock 等)
        source_weight: 来源权重 (用于加权平均)
    """
    datetime: datetime
    vt_symbol: str
    sentiment_score: float          # -1.0 到 1.0
    confidence: float = 1.0         # 0.0 到 1.0
    news_count: int = 1
    source: str = "unknown"
    source_weight: float = 1.0
    keywords: list[str] = field(default_factory=list)
    news_id: Optional[int] = None

    def weighted_score(self) -> float:
        """计算加权情感评分: score * confidence * source_weight * sqrt(news_count)"""
        import math
        return (
            self.sentiment_score
            * self.confidence
            * self.source_weight
            * math.sqrt(self.news_count)
        )

@dataclass(slots=True)
class AlphaSignal:
    """
    Alpha 交易信号数据结构

    Attributes:
        datetime: 信号生成时间
        vt_symbol: 合约代码
        direction: 信号方向 (LONG/SHORT/EXIT/HOLD)
        signal: 原始信号值 (-1 到 1)
        strength: 信号强度 (0 到 1)
        confidence: 信号置信度 (0 到 1)
    """
    datetime: datetime
    vt_symbol: str
    direction: SignalDirection
    signal: float = 0.0
    strength: float = 0.0
    confidence: float = 0.0
    sentiment_type: SentimentType = SentimentType.NEUTRAL
    sentiment_score: float = 0.0
    news_count: int = 0
    factors: Optional[dict] = None
```

#### 3.2.2 舆情信号计算器

```python
# vnpy/alpha/sentiment/calculator.py

@dataclass
class WindowConfig:
    """滚动窗口配置"""
    window_size: int = 5           # 窗口大小 (天数)
    min_periods: int = 1          # 最小有效数据量
    decay_factor: float = 0.9    # 指数衰减因子

class RollingWindowManager:
    """滚动窗口管理器 - 负责维护不同时间窗口的情感数据"""

    def __init__(self, config: Optional[WindowConfig] = None) -> None:
        self.config = config or WindowConfig()
        self._data: dict[str, list[SentimentData]] = {}

    def add(self, data: SentimentData) -> None:
        """添加单条舆情数据"""
        if data.vt_symbol not in self._data:
            self._data[data.vt_symbol] = []
        self._data[data.vt_symbol].append(data)

    def get_window(
        self,
        vt_symbol: str,
        end_dt: datetime,
        window_size: Optional[int] = None
    ) -> list[SentimentData]:
        """获取指定时间窗口内的数据"""
        window_size = window_size or self.config.window_size
        start_dt = end_dt - timedelta(days=window_size)

        if vt_symbol not in self._data:
            return []

        return [
            d for d in self._data[vt_symbol]
            if start_dt <= d.datetime <= end_dt
        ]

class WeightedScoreCalculator:
    """
    加权评分计算器

    核心算法:
    1. 新闻重要性加权: source_weight * confidence
    2. 时间衰减: decay_factor^(距离天数)
    3. 数量放大: sqrt(news_count)

    最终评分 = Σ(score_i * weight_i * decay_i) / Σ(weight_i * decay_i)
    """

    def calculate(
        self,
        data_list: list[SentimentData],
        reference_dt: datetime
    ) -> tuple[float, float, float]:
        """计算加权情感评分"""
        if not data_list:
            return 0.0, 0.0, 0

        weights: list[float] = []
        scores: list[float] = []
        confidences: list[float] = []

        for data in data_list:
            # 基础权重 = 来源权重 * 置信度 * sqrt(新闻数量)
            base_weight = (
                self.config.source_weights.get(data.source, self.config.source_weights["default"])
                * data.confidence
                * math.sqrt(data.news_count)
            )

            # 时间衰减 = decay_factor^(距离天数)
            days_diff = max(0, (reference_dt - data.datetime).days)
            decay = math.pow(self.config.decay_factor, days_diff)

            weight = base_weight * decay
            weights.append(weight)
            scores.append(data.sentiment_score)
            confidences.append(data.confidence)

        total_weight = sum(weights)
        if total_weight <= 0:
            return 0.0, 0.0, len(data_list)

        weighted_score = sum(s * w for s, w in zip(scores, weights)) / total_weight
        avg_confidence = sum(c * w for c, w in zip(confidences, weights)) / total_weight

        return weighted_score, avg_confidence, len(data_list)

class SignalStrengthEvaluator:
    """
    信号强度评估器

    评估维度:
    1. 一致性 (Consensus): 新闻情感方向的一致程度
    2. 集中度 (Concentration): 情感评分的集中程度
    3. 显著性 (Significance): 与中性值(0)的偏离程度
    4. 数量支撑 (Support): 新闻数量是否足够
    """

    def evaluate(
        self,
        data_list: list[SentimentData],
        weighted_score: float
    ) -> tuple[float, dict]:
        """评估信号强度"""
        if not data_list:
            return 0.0, {"reason": "no_data"}

        n = len(data_list)
        scores = [d.sentiment_score for d in data_list]

        # 1. 一致性
        bullish_count = sum(1 for s in scores if s > 0.1)
        bearish_count = sum(1 for s in scores if s < -0.1)
        consensus = max(bullish_count, bearish_count) / n

        # 2. 集中度 (1 - 标准化标准差)
        std = np.std(scores)
        concentration = 1.0 - min(std / 1.0, 1.0)

        # 3. 显著性
        significance = min(abs(weighted_score) / 1.0, 1.0)

        # 4. 数量支撑
        support = min(math.log(n + 1) / math.log(20), 1.0)

        # 加权组合
        strength = (
            self.w_consensus * consensus
            + self.w_concentration * concentration
            + self.w_significance * significance
            + self.w_support * support
        )

        return strength, {
            "consensus": consensus,
            "concentration": concentration,
            "significance": significance,
            "support": support,
            "n": n,
        }
```

#### 3.2.3 因子 DSL 表达式

```
# 可用函数
sentinel_sentiment(n)     - n天滚动情感评分
sentinel_news_count(n)     - n天滚动新闻数量
sentinel_confidence(n)    - n天滚动置信度
sentinel_strength(n)       - n天滚动信号强度

# 运算符
+ - * / > < >= <= == & |

# 示例表达式
"sentinel_sentiment(5) > 0.2"                           # 5日情感评分 > 0.2
"sentinel_sentiment(5) * sentinel_strength(5) > 0.1"    # 加权信号
"ts_rank(sentinel_sentiment(10), 5) > 0.8"              # 10日情感排名
"ts_delta(sentinel_sentiment(5), 1) > 0.1"             # 情感突变
```

#### 3.2.4 数学模型总结

**加权情感评分**:
```
S_weighted = Σ(s_i * w_i * d_i) / Σ(w_i * d_i)

其中:
- s_i: 第 i 条新闻的情感评分 (-1 到 1)
- w_i: 第 i 条新闻的权重 = source_weight * confidence * sqrt(news_count)
- d_i: 第 i 条新闻的时间衰减 = decay_factor^(days_diff)
```

**信号强度**:
```
Strength = w1 * Consensus + w2 * Concentration + w3 * Significance + w4 * Support

其中:
- Consensus = max(bullish_count, bearish_count) / n
- Concentration = 1 - min(std(scores) / 1.0, 1.0)
- Significance = min(|S_weighted| / 1.0, 1.0)
- Support = min(log(n + 1) / log(20), 1.0)

默认权重: w1=0.3, w2=0.2, w3=0.3, w4=0.2
```

### 3.3 SignalStrategy 信号驱动策略

SignalStrategy 负责将 Alpha 信号转化为具体的交易指令。

#### 3.3.1 仓位管理算法

```python
# vnpy/alpha/strategy/signal_strategy.py

class PositionManagement:
    """
    高级仓位管理 - 支持多种 sizing 算法
    """

    def __init__(
        self,
        strategy: "SignalStrategy",
        base_method: str = "fixed",
        base_volume: float = 100,
        max_position_pct: float = 0.1,
        max_total_exposure: float = 1.0
    ) -> None:
        self.strategy = strategy
        self.base_method = base_method
        self.base_volume = base_volume
        self.max_position_pct = max_position_pct
        self.max_total_exposure = max_total_exposure

        self.entry_prices: dict[str, float] = {}
        self.stop_losses: dict[str, float] = {}
        self.position_costs: dict[str, float] = defaultdict(float)

    def calculate_size(
        self,
        vt_symbol: str,
        signal: Signal,
        current_price: float
    ) -> PositionSize:
        """根据信号和风险参数计算仓位"""
        portfolio_value = self.strategy.get_portfolio_value()

        if portfolio_value <= 0:
            return PositionSize(vt_symbol=vt_symbol, volume=0, ...)

        # 根据方法计算基础仓位
        if self.base_method == "fixed":
            volume = self.base_volume
        elif self.base_method == "kelly":
            volume = self._kelly_size(signal, portfolio_value)
        elif self.base_method == "risk_parity":
            volume = self._risk_parity_size(vt_symbol, signal, portfolio_value)
        elif self.base_method == "volatility":
            volume = self._volatility_size(vt_symbol, signal, current_price, portfolio_value)
        else:
            volume = self.base_volume

        # 应用约束
        max_volume = portfolio_value * self.max_position_pct / current_price
        volume = min(volume, max_volume)

        # 计算止损和止盈
        risk_pct = getattr(self.strategy, 'risk_per_trade', 0.02)
        stop_distance = current_price * risk_pct
        stop_loss = current_price - stop_distance if signal.direction == Direction.LONG \
                    else current_price + stop_distance
        take_profit = current_price + stop_distance * 2

        return PositionSize(
            vt_symbol=vt_symbol, volume=volume, entry_price=current_price,
            stop_loss=stop_loss, take_profit=take_profit,
            risk_amount=stop_distance * volume, sizing_method=self.base_method
        )

    def _kelly_size(self, signal: Signal, portfolio_value: float) -> float:
        """Kelly Criterion 仓位管理"""
        win_rate = 0.55
        loss_rate = 1 - win_rate
        win_loss_ratio = 1.5

        kelly_fraction = (win_rate * win_loss_ratio - loss_rate) / win_loss_ratio
        kelly_fraction = max(0, min(kelly_fraction, 0.25))
        adjusted_fraction = kelly_fraction * signal.confidence

        return (portfolio_value * adjusted_fraction) / signal.strength

    def _risk_parity_size(self, vt_symbol: str, signal: Signal, portfolio_value: float) -> float:
        """风险均衡仓位管理"""
        target_risk = portfolio_value * 0.02
        risk_contribution = target_risk / len(self.strategy.vt_symbols)
        atr = self.atr_values.get(vt_symbol, 0.02)
        return (risk_contribution / atr) * signal.confidence

    def _volatility_size(self, vt_symbol: str, signal: Signal, current_price: float, portfolio_value: float) -> float:
        """波动率仓位管理"""
        atr = self.atr_values.get(vt_symbol, current_price * 0.02)
        historical_vol = self.historical_volatility.get(vt_symbol, 0.02)
        target_vol = portfolio_value * 0.01
        vol_adjusted_size = target_vol / (atr * historical_vol)
        return vol_adjusted_size * signal.strength * signal.confidence
```

#### 3.3.2 风险管理器

```python
# vnpy/alpha/strategy/risk_manager.py

class RiskManager:
    """
    舆情驱动的动态风险管理器

    Features:
    - 单笔仓位限制
    - 日亏损限额
    - 舆情异常检测
    - 回撤控制
    """

    DEFAULT_RULES: dict[str, RiskRule] = {
        "max_position_pct": RiskRule(
            name="max_position_pct",
            params={"value": 0.1}  # 10% of portfolio per position
        ),
        "daily_loss_limit": RiskRule(
            name="daily_loss_limit",
            params={"value": 0.05, "period": "daily"}  # 5% daily loss limit
        ),
        "max_drawdown": RiskRule(
            name="max_drawdown",
            params={"value": 0.15}  # 15% max drawdown
        ),
        "sentiment_anomaly": RiskRule(
            name="sentiment_anomaly",
            params={"threshold": 0.3, "lookback": 10}  # 30% change threshold
        ),
        "sentiment_cooldown": RiskRule(
            name="sentiment_cooldown",
            params={"minutes": 60}  # 60 minute cooldown after anomaly
        )
    }

    def check_sentiment_anomaly(self, vt_symbol: str) -> bool:
        """检测舆情异常"""
        rule = self.rules["sentiment_anomaly"]
        if not rule.enabled:
            return False

        # 检查冷却期
        if vt_symbol in self.sentiment_cooldowns:
            cooldown = self.sentiment_cooldowns[vt_symbol]
            cooldown_minutes = rule.params.get("minutes", 60)
            if datetime.now() < cooldown:
                return True

        history = self.sentiment_history.get(vt_symbol, [])
        if len(history) < 2:
            return False

        # 计算变化
        threshold = rule.params.get("threshold", 0.3)
        change = abs(history[-1] - history[0]) / (abs(history[0]) + 1e-6)

        if change > threshold:
            cooldown_minutes = rule.params.get("minutes", 60)
            self.sentiment_cooldowns[vt_symbol] = datetime.now()
            return True

        return False

    def get_risk_report(self) -> dict:
        """生成综合风控报告"""
        return {
            "daily_pnl": self.status.daily_pnl,
            "daily_loss": self.status.daily_loss,
            "max_drawdown": self.status.max_drawdown,
            "blocked_symbols": list(self.status.blocked_symbols),
            "sentiment_levels": {
                vt_symbol: self.get_sentiment_level(vt_symbol).value
                for vt_symbol in self.status.sentiment_scores
            }
        }
```

#### 3.3.3 舆情优先策略实现

```python
# vnpy/alpha/strategy/sentiment_strategy.py

class SentimentStrategy(SignalStrategy):
    """
    舆情优先交易策略

    Features:
    - 舆情信号作为主要驱动
    - Alpha 信号作为确认
    - 基于舆情的动态仓位调整
    - 实时舆情监控与调整
    """

    def __init__(
        self,
        strategy_engine: "BacktestingEngine",
        strategy_name: str,
        vt_symbols: list[str],
        setting: dict
    ) -> None:
        super().__init__(strategy_engine, strategy_name, vt_symbols, setting)

        # 策略参数
        self.sentiment_weight: float = setting.get("sentiment_weight", 0.6)
        self.alpha_weight: float = setting.get("alpha_weight", 0.4)
        self.sentiment_threshold: float = setting.get("sentiment_threshold", 0.55)
        self.min_holding_period: int = setting.get("min_holding_period", 3)

        # 舆情跟踪
        self.current_sentiment: dict[str, float] = defaultdict(float)
        self.target_sentiment: dict[str, float] = defaultdict(float)

        # 风险管理器
        self.risk_manager = RiskManager(self)

    def on_bars(self, bars: dict[str, BarData]) -> None:
        """K线切片回调"""
        alpha_signals_df: pl.DataFrame = self.get_signal()

        if alpha_signals_df.is_empty():
            return

        self.process_alpha_signal(alpha_signals_df, bars)

    def on_signals(self, signals: list[Signal], bars: dict[str, BarData]) -> None:
        """处理生成信号 (舆情优先)"""
        # 更新舆情
        for signal in signals:
            self._update_sentiment(signal)

        # 舆情加权聚合
        aggregated = self._aggregate_with_sentiment(signals)

        # 计算目标仓位
        targets = self.calculate_target_positions(aggregated, bars)

        # 应用舆情调整
        targets = self._apply_sentiment_adjustments(targets, bars)

        # 应用风控
        targets = self.apply_risk_controls(targets, bars)

        # 应用持仓期过滤
        targets = self._apply_holding_period_filter(targets)

        # 执行信号
        self.execute_signals(targets, bars)

    def _aggregate_with_sentiment(self, signals: list[Signal]) -> dict[str, Signal]:
        """舆情加权信号聚合"""
        alpha_signals = {s.vt_symbol: s for s in signals if s.signal_type == SignalType.ALPHA}
        sentiment_signals = self._generate_sentiment_signals(signals)

        combined: dict[str, list[Signal]] = defaultdict(list)
        for s in alpha_signals.values():
            combined[s.vt_symbol].append(s)
        for s in sentiment_signals.values():
            combined[s.vt_symbol].append(s)

        result: dict[str, Signal] = {}
        for vt_symbol, symbol_signals in combined.items():
            alpha_sig = alpha_signals.get(vt_symbol)
            sentiment_sig = sentiment_signals.get(vt_symbol)

            if alpha_sig and sentiment_sig:
                # 舆情加权组合
                weighted_direction = (
                    alpha_sig.strength * self.alpha_weight * (1 if alpha_sig.direction == Direction.LONG else -1) +
                    sentiment_sig.strength * self.sentiment_weight * (1 if sentiment_sig.direction == Direction.LONG else -1)
                ) / (self.alpha_weight + self.sentiment_weight)

                result[vt_symbol] = Signal(
                    vt_symbol=vt_symbol,
                    signal_type=SignalType.MIXED,
                    direction=Direction.LONG if weighted_direction > 0 else Direction.SHORT,
                    strength=abs(weighted_direction),
                    confidence=(alpha_sig.confidence * self.alpha_weight + sentiment_sig.confidence * self.sentiment_weight) / (self.alpha_weight + self.sentiment_weight),
                    timestamp=max(alpha_sig.timestamp, sentiment_sig.timestamp),
                    metadata={"alpha_signal": alpha_sig, "sentiment_signal": sentiment_sig}
                )
            elif alpha_sig:
                result[vt_symbol] = alpha_sig
            elif sentiment_sig:
                result[vt_symbol] = sentiment_sig

        return result

    def _apply_sentiment_adjustments(
        self,
        targets: dict[str, float],
        bars: dict[str, BarData]
    ) -> dict[str, float]:
        """应用舆情仓位调整"""
        adjusted: dict[str, float] = {}

        for vt_symbol, target in targets.items():
            sentiment = self.current_sentiment.get(vt_symbol, 0.5)

            if sentiment < 0.3 and target > 0:
                # 舆情危急，减少多头仓位
                adjusted[vt_symbol] = target * 0.5
                self.write_log(f"Reducing {vt_symbol} due to critical sentiment: {sentiment:.2f}")
            elif sentiment > 0.7 and target > 0:
                # 舆情强劲，维持或增加
                adjusted[vt_symbol] = target * 1.2
            else:
                adjusted[vt_symbol] = target

        return adjusted
```

#### 3.3.4 回测/实盘切换机制

```python
# vnpy/alpha/strategy/live_trading.py

class TradingModeManager:
    """
    回测/实盘切换管理器

    Usage:
        # 回测模式
        manager = TradingModeManager(BacktestingAdapter(engine))
        manager.execute_signals(targets, bars)

        # 实盘模式
        manager = TradingModeManager(LiveTradingAdapter())
        manager.execute_signals(targets, bars)
    """

    def __init__(self, adapter: TradingAdapter) -> None:
        self.adapter = adapter
        self.is_backtesting = isinstance(adapter, BacktestingAdapter)

    @property
    def trading_mode(self) -> str:
        return "backtesting" if self.is_backtesting else "live"

    def execute_signals(
        self,
        targets: dict[str, float],
        bars: dict[str, BarData],
        price_add: float = 0.05
    ) -> None:
        """在当前模式下执行交易信号"""
        for vt_symbol, target in targets.items():
            self._adjust_position(vt_symbol, target, bars.get(vt_symbol), price_add)

    def _adjust_position(
        self,
        vt_symbol: str,
        target: float,
        bar: BarData | None,
        price_add: float
    ) -> None:
        """调整仓位到目标"""
        if bar is None:
            return

        current = self.adapter.get_pos(vt_symbol)
        diff = target - current

        if abs(diff) < 1e-6:
            return

        if diff > 0:
            price = bar.close_price * (1 + price_add)
            self.adapter.send_order(vt_symbol, Direction.LONG, Offset.OPEN, price, diff)
        else:
            price = bar.close_price * (1 - price_add)
            volume = abs(diff)
            self.adapter.send_order(vt_symbol, Direction.SHORT, Offset.CLOSE, price, volume)

---

## 4. 目录结构设计

### 4.1 融合项目目录

```
sentiment_alpha_trader/
├── README.md
├── pyproject.toml
├── requirements.txt
├── .env.example
│
├── src/                          # 融合代码
│   ├── __init__.py
│   ├── main.py                   # 融合系统入口
│   │
│   ├── bridge/                   # 事件桥接模块
│   │   ├── __init__.py
│   │   ├── event_bridge.py       # 核心事件桥接
│   │   ├── models.py             # 融合数据模型
│   │   └── adapter.py            # 数据格式适配器
│   │
│   ├── alpha/                    # Alpha信号模块
│   │   ├── __init__.py
│   │   ├── sentiment_engine.py   # 舆情信号引擎
│   │   ├── signal_generator.py   # 信号生成器
│   │   ├── factor_cache.py       # 因子缓存
│   │   └── app.py                # vnpy风格App
│   │
│   ├── strategies/               # 策略模块
│   │   ├── __init__.py
│   │   ├── signal_strategy.py    # 信号驱动策略
│   │   └── sentiment_strategy.py  # 舆情优先策略
│   │
│   ├── gateways/                 # 融合Gateway
│   │   ├── __init__.py
│   │   ├── sentiment_gateway.py   # 舆情数据Gateway
│   │   └── news_gateway.py       # 新闻数据Gateway
│   │
│   └── knowledge/                # 知识图谱同步
│       ├── __init__.py
│       └── sync_service.py        # 图谱同步服务
│
├── vnpy/                         # vnpy源码(作为依赖或 submodule)
│   ├── event/
│   ├── trader/
│   └── alpha/
│
├── FinnewsHunter/               # FinnewsHunter源码(作为依赖或 submodule)
│   └── backend/app/
│       ├── agents/
│       ├── services/
│       ├── tools/
│       └── knowledge/
│
└── tests/
    ├── test_event_bridge.py
    ├── test_sentiment_engine.py
    └── test_integration.py
```

### 4.2 依赖管理

```toml
# pyproject.toml

[project]
name = "sentiment-alpha-trader"
version = "0.1.0"
description = "Sentiment-driven quantitative trading system"
requires-python = ">=3.10"

dependencies = [
    # vnpy core
    "vnpy",

    # FinnewsHunter dependencies
    "fastapi>=0.110.0",
    "uvicorn>=0.27.0",
    "sqlalchemy[asyncio]>=2.0.25",
    "asyncpg>=0.29.0",
    "redis>=5.0.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
    "agenticx>=0.1.0",
    "akshare>=1.13.0",
    "beautifulsoup4>=4.12.0",
    "httpx>=0.27.0",

    # Optional
    "milvus-lite>=2.4.0",
    "neo4j>=5.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.2.0",
    "mypy>=1.8.0",
]
```

---

## 5. API 设计

### 5.1 融合系统 API

```python
# src/main.py

from fastapi import FastAPI, HTTPException
from contextlib import asynccontextmanager
from vnpy.trader.engine import MainEngine
from vnpy.trader.gateway import BaseGateway

from src.bridge.event_bridge import EventBridge
from src.alpha.sentiment_engine import SentimentCalculator
from FinnewsHunter.app.agents.orchestrator import DebateOrchestrator
from FinnewsHunter.app.core.config import Settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动
    settings = Settings()
    event_engine = EventEngine()
    main_engine = MainEngine(event_engine)

    # 初始化融合组件
    event_bridge = EventBridge(event_engine)
    sentiment_calculator = SentimentCalculator()
    debate_orchestrator = DebateOrchestrator()

    # 注册App (vnpy风格)
    # main_engine.add_app(AlphaSignalApp)

    app.state.event_engine = event_engine
    app.state.main_engine = main_engine
    app.state.event_bridge = event_bridge
    app.state.sentiment_calculator = sentiment_calculator
    app.state.debate_orchestrator = debate_orchestrator

    yield

    # 关闭
    main_engine.close()

app = FastAPI(title="Sentiment-Alpha Trader API", lifespan=lifespan)

# --- 舆情相关API ---

@app.post("/api/v1/sentiment/news")
async def submit_news(
    title: str,
    content: str,
    url: str,
    source: str,
    stock_codes: list[str],
    sentiment_score: float = 0.0,
    sentiment_label: str = "neutral"
):
    """提交新闻并触发舆情分析"""
    # 1. 保存新闻到数据库
    # 2. 触发舆情计算
    # 3. 通过EventBridge发送到vnpy
    pass

@app.get("/api/v1/sentiment/{vt_symbol}")
async def get_sentiment(vt_symbol: str, lookback_hours: int = 24):
    """获取标的舆情评分"""
    pass

@app.get("/api/v1/alpha_signal/{vt_symbol}")
async def get_alpha_signal(vt_symbol: str):
    """获取Alpha信号"""
    pass

# --- 辩论相关API (复用FinnewsHunter) ---

@app.post("/api/v1/agents/debate")
async def run_debate(stock_code: str, mode: str = "parallel"):
    """运行多智能体辩论"""
    pass

@app.get("/api/v1/agents/debate/{debate_id}/status")
async def get_debate_status(debate_id: str):
    """获取辩论状态"""
    pass

# --- 交易相关API (复用vnpy) ---

@app.post("/api/v1/orders/send")
async def send_order(vt_symbol: str, direction: str, volume: float, price: float = 0):
    """发送订单"""
    pass

@app.post("/api/v1/orders/cancel")
async def cancel_order(vt_symbol: str, order_id: str):
    """取消订单"""
    pass

@app.get("/api/v1/positions")
async def get_positions():
    """获取持仓"""
    pass

@app.get("/api/v1/account")
async def get_account():
    """获取账户信息"""
    pass
```

---

## 6. 数据流设计

### 6.1 完整数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据流图                                        │
└─────────────────────────────────────────────────────────────────────────────┘

[外部数据源]
     │
     │  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
     ├──▶│ 财经新闻爬虫  │     │  K线行情     │     │  社交媒体   │
     │  │ (FinnewsHunter)│   │  (AkShare)   │     │  (待扩展)   │
     │  └──────┬───────┘     └──────┬───────┘     └──────────────┘
     │         │
     │         ▼
     │  ┌──────────────────────────────────────────────────────────────┐
     │  │                     PostgreSQL 数据库                          │
     │  │  ┌─────────┐  ┌────────────┐  ┌───────────┐  ┌─────────────┐  │
     │  │  │  News   │  │ Analysis   │  │ Sentiment │  │AlphaSignal │  │
     │  │  │         │  │            │  │  Cache    │  │            │  │
     │  │  └─────────┘  └────────────┘  └───────────┘  └─────────────┘  │
     │  └──────────────────────────────────────────────────────────────┘
     │                                      │
     │    ┌─────────────────────────────────┼─────────────────────────┐
     │    │                                 │                         │
     │    ▼                                 ▼                         ▼
     │  ┌──────────────┐            ┌──────────────┐         ┌──────────────┐
     │  │ EventBridge  │            │SentimentCalc │         │向量数据库    │
     │  │              │───────────▶│              │────────▶│  (Milvus)    │
     │  └──────┬───────┘            └──────┬───────┘         └──────────────┘
     │         │                           │
     │         │ EVENT_SENTIMENT            │ 计算Alpha信号
     │         │ EVENT_ALPHA_SIGNAL         ▼
     │         │                    ┌──────────────┐
     │         │                    │AlphaSignal   │
     │         └───────────────────▶│Engine         │
     │                              └───────┬───────┘
     │                                      │ EVENT_ALPHA_SIGNAL
     │                                      ▼
     │                              ┌──────────────┐
     │                              │SignalStrategy│
     │                              │              │──▶ OrderRequest
     │                              └──────────────┘
     │                                      │
     │                                      ▼
     │                              ┌──────────────┐
     │                              │ MainEngine   │
     │                              │   (vnpy)     │
     │                              └──────┬───────┘
     │                                     │
     │         ┌──────────────────────────┼──────────────────────────┐
     │         ▼                          ▼                          ▼
     │  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
     │  │  CTP Gateway │           │  HTTP Gateway│           │  WebSocket  │
     │  │  (期货)      │           │   (股票)    │           │  Gateway    │
     │  └─────────────┘           └─────────────┘           └─────────────┘
     │         │                          │                          │
     │         ▼                          ▼                          ▼
     │  ┌─────────────────────────────────────────────────────────────┐
     │  │                        交易所                                 │
     │  │   中金所 │ 上交所 │ 深交所 │ 上期所 │ 大商所 │ 郑商所 │ ...     │
     │  └─────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           知识图谱 (Neo4j)                                   │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐            │
│  │Company  │◀───▶│Stock    │◀───▶│News     │◀───▶│Analysis │            │
│  │         │     │         │     │         │     │         │            │
│  └─────────┘     └────┬────┘     └─────────┘     └─────────┘            │
│                       │                                                   │
│                       └──────────────────────────────────────────────────▶  │
│                              Trade │ Signal │ Sentiment                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 实施计划

### 7.1 阶段划分

| 阶段 | 周期 | 目标 | 关键交付物 |
|------|------|------|-----------|
| **Phase 1: 基础融合** | 2周 | 实现事件桥接 | EventBridge, SentimentGateway |
| **Phase 2: 信号引擎** | 2周 | 完成舆情Alpha | SentimentEngine, SignalGenerator |
| **Phase 3: 策略集成** | 1周 | 打通交易闭环 | SignalStrategy, 完整数据流 |
| **Phase 4: 知识图谱** | 1周 | 图谱同步 | TradeGraphSync, 关系可视化 |
| **Phase 5: 优化完善** | 1周 | 性能与稳定 | 测试、监控、文档 |

### 7.2 Phase 1 详细任务

```
Week 1-2: 基础融合
├── [1.1] 搭建项目骨架
│   ├── 创建目录结构
│   ├── 配置依赖 (pyproject.toml)
│   └── 初始化 git submodule (vnpy + FinnewsHunter)
│
├── [1.2] 实现 EventBridge
│   ├── 继承 vnpy EventEngine
│   ├── 定义融合事件类型
│   └── 实现双向事件路由
│
├── [1.3] 实现 SentimentGateway
│   ├── 实现 BaseGateway 接口
│   ├── on_sentiment() 回调
│   └── query_sentiment() 接口
│
└── [1.4] 编写单元测试
    ├── test_event_bridge.py
    └── test_sentiment_gateway.py
```

### 7.3 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| vnpy 版本升级导致 API 不兼容 | 高 | 使用版本锁定 + 抽象层隔离 |
| FinnewsHunter AgenticX 框架耦合 | 中 | 抽象出接口，降低依赖 |
| 事件风暴导致性能问题 | 中 | 添加事件过滤 + 限流 |
| 数据一致性 (新闻 vs 交易) | 低 | 使用事务 + 最终一致性 |

---

## 8. 关键文件清单

### 8.1 需新建的文件

| 文件路径 | 描述 |
|----------|------|
| `src/bridge/event_bridge.py` | 事件桥接核心 |
| `src/bridge/models.py` | 融合数据模型 |
| `src/alpha/sentiment_engine.py` | 舆情信号引擎 |
| `src/alpha/app.py` | vnpy风格App |
| `src/strategies/signal_strategy.py` | 信号驱动策略 |
| `src/gateways/sentiment_gateway.py` | 舆情Gateway |
| `src/main.py` | FastAPI入口 |
| `pyproject.toml` | 项目配置 |
| `tests/test_event_bridge.py` | 事件桥接测试 |
| `tests/test_integration.py` | 集成测试 |

### 8.2 需修改的文件

| 文件路径 | 修改内容 |
|----------|----------|
| `FinnewsHunter/backend/app/agents/orchestrator.py` | 添加事件桥接回调 |
| `FinnewsHunter/backend/app/services/llm_service.py` | 支持异步调用 |
| `vnpy/vnpy/event/engine.py` | (如需扩展事件类型) |
| `vnpy/vnpy/alpha/dataset/template.py` | 支持舆情因子 |

---

## 9. 总结

### 9.1 融合核心价值

1. **舆情量化**: 将 FinnewsHunter 的新闻舆情分析能力与 vnpy 的交易执行能力结合
2. **信号闭环**: 实现从"新闻 → 舆情 → Alpha信号 → 交易 → 反馈"的完整闭环
3. **知识增强**: 通过 Neo4j 图谱记录交易与舆情的关联，支持回溯分析
4. **多智能体协作**: 复用辩论机制进行复杂投资决策

### 9.2 技术亮点

- **EventBridge**: 创新性地连接两个事件系统
- **SentimentGateway**: 将舆情数据标准化为 vnpy Gateway 接口
- **AlphaSignalApp**: 以 vnpy App 风格封装舆情Alpha功能

### 9.3 下一步行动

1. 评审本方案并提出修改意见
2. 确定 Phase 1 优先级和人员分工
3. 搭建开发环境，验证 EventBridge 原型

---

## 10. 详细模块设计索引

本方案包含以下详细设计模块：

| 章节 | 模块 | 核心类/函数 |
|------|------|-------------|
| 3.1 | EventBridge 事件桥接 | `EventBridge`, `BridgeEvent`, `RateLimiter`, `EventFilter` |
| 3.2 | SentimentEngine 舆情计算 | `SentimentData`, `AlphaSignal`, `SentimentCalculator`, `WeightedScoreCalculator`, `SignalStrengthEvaluator` |
| 3.3 | SignalStrategy 信号策略 | `PositionManagement`, `RiskManager`, `SentimentStrategy`, `TradingModeManager` |
| 11 | 部署架构 | `Dockerfile`, `docker-compose.yml`, `Kustomize`, `Kubernetes Deployment` |
| 12 | 监控运维 | `Prometheus`, `Grafana`, `AlertManager`, `backup/restore.sh` |
| 13 | 测试策略 | `pytest`, `testcontainers`, `Playwright E2E`, `GitHub Actions CI/CD` |
| 14 | 安全设计 | `JWTHandler`, `RBAC`, `RateLimiter`, `EncryptedField` |
| 15 | 容错恢复 | `CircuitBreaker`, `RetryConfig`, `GracefulDegradation`, `Saga` |
| 16 | 性能优化 | `ConnectionPool`, `MultiLevelCache`, `BatchProcessor`, `AsyncBatcher` |

### 核心设计亮点

1. **双队列异步桥接架构**
   - vnpy 端使用 `queue.Queue` (线程安全)
   - FastAPI 端使用 `asyncio.Queue` (协程安全)
   - 中间使用桥接线程转发，零阻塞

2. **舆情信号计算数学模型**
   - 加权情感评分: `S_weighted = Σ(s_i * w_i * d_i) / Σ(w_i * d_i)`
   - 信号强度: 多维度加权评估 (一致性、集中度、显著性、支撑度)

3. **仓位管理算法**
   - Fixed: 固定仓位
   - Kelly Criterion: 最优增长
   - Risk Parity: 风险均衡
   - Volatility-based: 波动率自适应

4. **舆情驱动风控**
   - 舆情异常检测 (30% 变化阈值)
   - 动态仓位调整 (舆情危急时自动降仓)
   - 冷却期机制 (60分钟)

### 文件结构总览

```
sentiment_alpha_trader/
├── event_bridge/                      # 事件桥接 (新增)
│   ├── __init__.py
│   ├── core.py                       # EventBridge 核心
│   ├── types.py                     # 事件类型定义
│   ├── filter.py                    # 过滤与限流
│   ├── processor.py                  # 异步处理器
│   └── exceptions.py                 # 异常定义
│
├── vnpy/trader/                     # vnpy 扩展 (修改)
│   ├── sentiment_gateway.py          # 舆情 Gateway (新增)
│   ├── sentiment_cache.py           # Redis 缓存 (新增)
│   ├── vector_store.py              # Milvus 向量 (新增)
│   └── sync_service.py             # 数据同步 (新增)
│
├── vnpy/alpha/sentiment/            # 舆情 Alpha (新增)
│   ├── __init__.py
│   ├── data.py                     # SentimentData
│   ├── signal.py                  # AlphaSignal
│   ├── calculator.py               # 核心计算器
│   ├── factor_expression.py        # DSL 表达式
│   ├── dataset.py                 # AlphaDataset 扩展
│   ├── strategy.py                 # 策略实现
│   └── backtesting.py             # 回测支持
│
└── src/                            # 融合层
    ├── main.py                    # FastAPI 入口
    ├── bridge/                    # 桥接模块
    └── strategies/                # 策略模块
```

---

## 11. 部署架构

### 11.1 Docker 多阶段构建

```dockerfile
# deployment/docker/Dockerfile

# Stage 1: Builder
FROM python:3.11-slim as builder
WORKDIR /build
RUN apt-get update && apt-get install -y build-essential curl git && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2-4: Base → Dependencies → Application
FROM python:3.11-slim as base
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl libgomp1 libgl1 libglib2.0-0 && rm -rf /var/lib/apt/lists/* && useradd --create-home --shell /bin/bash appuser

# Stage 5: Development
FROM application as development
RUN pip install --no-cache-dir pytest pytest-asyncio pytest-cov ruff mypy
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# Stage 6: Production (默认)
FROM application as production
RUN pip install --no-cache-dir gunicorn uvicorn
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 CMD curl -f http://localhost:8000/health || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "src.main:app"]

# Stage 7: vnpy Runner
FROM base as vnpy-runner
WORKDIR /app
COPY --chown=appuser:appuser vnpy/ /app/vnpy/
COPY --chown=appuser:appuser src/ /app/src/
USER appuser
CMD ["python", "-m", "src.alpha.app"]
```

### 11.2 docker-compose 架构

```yaml
# deployment/docker/docker-compose.yml (核心服务)

services:
  api:
    build:
      context: ../..
      dockerfile: deployment/docker/Dockerfile
      target: production
    container_name: sentiment-alpha-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - ../config/${ENV:-prod}.env
    volumes:
      - ../../logs:/app/logs
      - ../../data:/app/data
    depends_on:
      - postgres
      - redis
      - milvus
      - neo4j
    networks:
      - sentiment-network

  vnpy-engine:
    build:
      context: ../..
      dockerfile: deployment/docker/Dockerfile
      target: vnpy-runner
    container_name: sentiment-alpha-vnpy
    restart: unless-stopped
    privileged: true  # 需要交易权限
    volumes:
      - ../../logs:/app/logs
      - ../../vnpy_data:/app/vnpy_data

  postgres:
    image: postgres:16-alpine
    container_name: sentiment-alpha-postgres
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data

  milvus:
    image: milvusdb/milvus:v2.3.3
    depends_on:
      - etcd
      - minio

  neo4j:
    image: neo4j:5.14-community
    volumes:
      - neo4j_data:/data

  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3030:3000"

networks:
  sentiment-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  milvus_data:
  neo4j_data:
  prometheus_data:
  grafana_data:
```

### 11.3 Kubernetes 部署

```yaml
# deployment/kubernetes/base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sentiment-alpha-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sentiment-alpha
      component: api
  template:
    metadata:
      labels:
        app: sentiment-alpha
        component: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
    spec:
      serviceAccountName: sentiment-alpha
      containers:
        - name: api
          image: sentiment-alpha:latest
          ports:
            - name: http
              containerPort: 8000
          envFrom:
            - configMapRef:
                name: sentiment-alpha-config
            - secretRef:
                name: sentiment-alpha-secret
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
```

### 11.4 Kustomize 多环境管理

```yaml
# deployment/kubernetes/overlays/prod/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: prod-
namespace: sentiment-alpha-prod

bases:
  - ../../base

patchesStrategicMerge:
  - deployment-patch.yaml
  - hpa-patch.yaml

configMapGenerator:
  - name: sentiment-alpha-config
    behavior: merge
    literals:
      - APP_ENV=production
      - LOG_LEVEL=WARNING
      - API_WORKERS=4

replicas:
  - name: sentiment-alpha-api
    count: 3
```

### 11.5 配置管理

```yaml
# deployment/config/settings.yaml

app:
  name: "sentiment-alpha-trader"
  version: "0.1.0"
  environment: "${ENV}"

server:
  host: "0.0.0.0"
  port: 8000
  workers: 4
  timeout: 300

logging:
  level: "${LOG_LEVEL}"
  format: "json"
  output: "/app/logs"

database:
  postgres:
    host: "${POSTGRES_HOST}"
    port: 5432
    database: "${POSTGRES_DB}"
    pool_size: 20
    max_overflow: 10
  redis:
    host: "${REDIS_HOST}"
    port: 6379
    max_connections: 50
  milvus:
    host: "${MILVUS_HOST}"
    port: 19530
  neo4j:
    host: "${NEO4J_HOST}"
    bolt_port: 7687

sentiment:
  window:
    size: 5
    min_periods: 1
  weights:
    source:
      sina: 1.0
      jrj: 0.9
      cnstock: 0.9
      eastmoney: 0.8
    decay_factor: 0.9
  thresholds:
    bullish: 0.55
    bearish: 0.45
    min_confidence: 0.6
    min_news_count: 3

alpha:
  strength_weights:
    consensus: 0.3
    concentration: 0.2
    significance: 0.3
    support: 0.2
  risk:
    max_position_pct: 0.1
    max_total_exposure: 1.0
    daily_loss_limit: 0.05
    max_drawdown: 0.15

monitoring:
  prometheus:
    enabled: true
    port: 9090
  health:
    enabled: true
    port: 8001
  tracing:
    enabled: true
    sample_rate: 0.1
```

---

## 12. 监控运维

### 12.1 Prometheus 指标配置

```yaml
# deployment/monitoring/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'sentiment-alpha-api'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - sentiment-alpha-prod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_component]
        action: keep
        regex: api
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: keep
        regex: "8000|8001"

  - job_name: 'sentiment-alpha-vnpy'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_component]
        action: keep
        regex: vnpy-engine

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']
```

### 12.2 核心监控指标

| 指标类别 | 指标名称 | 描述 |
|----------|----------|------|
| **系统** | `http_requests_total` | HTTP 请求总数 |
| | `http_request_duration_seconds` | 请求延迟分布 |
| | `container_memory_usage_bytes` | 内存使用量 |
| | `container_cpu_usage_seconds_total` | CPU 使用时间 |
| **交易** | `trading_daily_pnl` | 日盈亏 |
| | `trading_positions_total` | 持仓数量 |
| | `trading_capital_utilization` | 资金利用率 |
| | `trading_max_drawdown` | 最大回撤 |
| **舆情** | `sentiment_score` | 舆情评分 |
| | `sentiment_news_total` | 新闻总数 |
| | `sentiment_score_change` | 舆情变化率 |
| **Alpha** | `alpha_signal_strength` | 信号强度 |
| | `alpha_signal_confidence` | 信号置信度 |
| | `alpha_signal_direction` | 信号方向分布 |

### 12.3 告警规则

```yaml
# deployment/monitoring/alerts/trading-alerts.yaml

groups:
  - name: trading_alerts
    rules:
      - alert: HighDailyLoss
        expr: trading_daily_pnl < -0.05
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "日亏损超过 5%"
          description: "当前日亏损为 {{ $value | humanizePercentage }}"

      - alert: MaxDrawdownBreached
        expr: trading_max_drawdown > 0.15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "最大回撤超过 15%"

      - alert: SentimentAnomalyDetected
        expr: sentiment_score_change > 0.3
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "舆情异常波动"

      - alert: LowSignalConfidence
        expr: avg(alpha_signal_confidence) < 0.6
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Alpha 信号置信度过低"

      - alert: PositionConcentration
        expr: max(position_value / trading_portfolio_value) > 0.3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "仓位过于集中"

  - name: infrastructure
    rules:
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.85
        for: 5m
        labels:
          severity: warning

      - alert: DatabaseConnectionPoolExhausted
        expr: postgres_connections_available / postgres_connections_max < 0.1
        for: 5m
        labels:
          severity: warning

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
```

### 12.4 备份与恢复

```bash
# deployment/backup/backup.sh

backup_postgres() {
    local backup_file="${BACKUP_DIR}/postgres_${DATE}.sql.gz"
    pg_dump -U "${POSTGRES_USER}" \
            -h "${POSTGRES_HOST}" \
            -d "${POSTGRES_DB}" \
    | gzip > "${backup_file}"
    echo "${backup_file}"
}

backup_redis() {
    local backup_file="${BACKUP_DIR}/redis_${DATE}.rdb.gz"
    redis-cli -h "${REDIS_HOST}" -p "${REDIS_PORT}" -a "${REDIS_PASSWORD}" --rdb /tmp/redis_backup.rdb
    gzip -c /tmp/redis_backup.rdb > "${backup_file}"
    echo "${backup_file}"
}

backup_milvus() {
    milvus-backup create -n "backup_${DATE}" || true
}

backup_neo4j() {
    cypher-shell -u "${NEO4J_USER}" -p "${NEO4J_PASSWORD}" \
        "CALL apoc.export.cypher.all('${BACKUP_DIR}/neo4j_${DATE}.cypher', {})"
}
```

---

## 13. 测试策略

### 13.1 测试金字塔

```
                        ┌─────────────────┐
                        │   E2E Tests     │  10%
                        │  (Playwright)   │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │ Integration     │  20%
                        │ Tests          │
                        └────────┬────────┘
                                 │
┌────────────────────────────────┴────────────────────────────────┐
│                        Unit Tests                               │  70%
│  EventBridge | SentimentEngine | SignalStrategy | RiskManager   │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 单元测试示例

```python
# tests/unit/test_event_bridge.py

class TestEventBridge:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.vnpy_engine = MagicMock()
        self.bridge = EventBridge(
            name="TestBridge",
            vnpy_engine=self.vnpy_engine,
            max_queue_size=100,
            enable_metrics=False
        )
        yield
        self.bridge.stop()

    def test_emit_to_vnpy_creates_correct_event(self):
        """测试 emit_to_vnpy 创建正确的事件"""
        result = self.bridge.emit_to_vnpy("eSentiment", {"vt_symbol": "600519.SSE", "sentiment_score": 0.75})
        assert isinstance(result, VnpyEvent)
        assert result.type == "eSentiment"
        self.vnpy_engine.put.assert_called_once()

    def test_concurrent_event_emission(self):
        """测试并发事件发送的线程安全"""
        def emit_events():
            for i in range(100):
                self.bridge.emit_to_vnpy("eSentiment", {"index": i})
        threads = [threading.Thread(target=emit_events) for _ in range(5)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        assert events_emitted.qsize() == 500


class TestWeightedScoreCalculator:
    def test_calculate_with_empty_data(self):
        """测试空数据返回零值"""
        score, confidence, count = self.calculator.calculate([], datetime.now())
        assert score == 0.0
        assert confidence == 0.0

    def test_weighted_score_property(self):
        """测试 SentimentData.weighted_score() 方法"""
        data = SentimentData(datetime.now(), "600519.SSE", 0.75, 0.85, 4, "sina", 1.0)
        weighted = data.weighted_score()
        expected = 0.75 * 0.85 * 1.0 * math.sqrt(4)
        assert abs(weighted - expected) < 0.001


class TestPositionManagement:
    def test_calculate_size_fixed_method(self):
        """测试固定仓位计算"""
        signal = MagicMock()
        signal.direction = SignalDirection.LONG
        signal.strength = 0.8
        signal.confidence = 0.9
        size = self.pm.calculate_size("600519.SSE", signal, 1000.0)
        assert size.volume == 100

    def test_calculate_size_respects_max_position_pct(self):
        """测试仓位计算遵守最大仓位限制"""
        self.pm.base_method = "fixed"
        self.pm.base_volume = 10000
        size = self.pm.calculate_size("600519.SSE", signal, 100.0)
        max_volume = 1_000_000 * 0.1 / 100
        assert size.volume <= max_volume
```

### 13.3 集成测试配置

```python
# tests/integration/conftest.py

@pytest.fixture(scope="session")
def test_env() -> IntegrationTestEnvironment:
    """创建测试环境配置"""
    postgres_mgr = PostgreSQLContainerManager()
    redis_mgr = RedisContainerManager()
    neo4j_mgr = Neo4jContainerManager()

    env = IntegrationTestEnvironment(
        postgres_dsn=postgres_mgr.get_dsn(),
        redis_url=redis_mgr.get_url(),
        neo4j_uri=neo4j_mgr.get_uri(),
        neo4j_user=neo4j_mgr.user,
        neo4j_password=neo4j_mgr.password
    )
    yield env
    postgres_mgr.stop()
    redis_mgr.stop()
    neo4j_mgr.stop()

@pytest.fixture
async def postgres_pool(test_env) -> AsyncGenerator[asyncpg.Pool, None]:
    pool = await asyncpg.create_pool(test_env.postgres_dsn, min_size=2, max_size=10)
    yield pool
    await pool.close()
```

### 13.4 E2E 关键旅程

```typescript
// tests/e2e/journeys/news-to-trade.spec.ts

test('Happy Path: News enters system and generates trade', async ({ page }) => {
    // Step 1: Submit news event via API
    const newsResponse = await page.request.post('/api/v1/sentiment/news', {
        data: {
            title: '某上市公司发布超预期财报',
            source: 'sina',
            stock_codes: ['600519.SSE'],
            urgency: 'high'
        }
    });
    expect(newsResponse.ok()).toBeTruthy();
    const newsResult = await newsResponse.json();

    // Step 2: Poll for sentiment result
    let sentimentResult = null;
    for (let i = 0; i < 10; i++) {
        const sentimentResponse = await page.request.get(`/api/v1/sentiment/${newsResult.vt_symbol}`);
        if (sentimentResponse.ok()) {
            const data = await sentimentResponse.json();
            if (data.sentiment_score !== undefined) {
                sentimentResult = data;
                break;
            }
        }
        await page.waitForTimeout(2000);
    }
    expect(sentimentResult.sentiment_score).toBeGreaterThan(0);

    // Step 3: Verify alpha signal is generated
    const signalResponse = await page.request.get(`/api/v1/alpha_signal/${newsResult.vt_symbol}`);
    expect(signalResponse.ok()).toBeTruthy();
    const signalData = await signalResponse.json();
    expect(signalData.direction).toMatch(/LONG|SHORT/);

    // Step 4: Execute trade (simulated)
    const executeButton = page.locator('[data-testid="execute-trade-btn"]');
    await executeButton.click();
    await page.click('[data-testid="confirm-trade-btn"]');
    await page.waitForSelector('[data-testid="order-created-toast"]');
});
```

### 13.5 CI/CD 工作流

```yaml
# .github/workflows/test.yml

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Run unit tests
        run: |
          pytest tests/unit/ \
            --cov=src \
            --cov-report=xml \
            --cov-fail-under=80 \
            --timeout=60

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - name: Run integration tests
        env:
          POSTGRES_DSN: postgresql://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
        run: pytest tests/integration/ --timeout=120

  e2e-tests:
    name: E2E Tests (Playwright)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Run E2E tests
        working-directory: tests/e2e
        run: npx playwright test
```

---

## 14. 安全设计

### 14.1 JWT/OAuth2 认证

```python
# src/core/security.py

from datetime import datetime, timedelta
from typing import Optional
import jwt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class JWTHandler:
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.access_token_expire_minutes = 30

    def create_access_token(self, data: dict, expires_delta: Optional[timedelta] = None) -> str:
        to_encode = data.copy()
        if expires_delta:
            expire = datetime.utcnow() + expires_delta
        else:
            expire = datetime.utcnow() + timedelta(minutes=self.access_token_expire_minutes)
        to_encode.update({"exp": expire})
        encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
        return encoded_jwt

    def verify_token(self, token: str) -> Optional[dict]:
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.PyJWTError:
            return None

# FastAPI 依赖注入
async def get_current_user(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    payload = jwt_handler.verify_token(token)
    if payload is None:
        raise HTTPException(status_code=401, detail="Invalid token")
    return payload.get("sub")
```

### 14.2 RBAC 权限控制

```python
# src/core/rbac.py

from enum import Enum
from typing import Set

class Role(str, Enum):
    ADMIN = "admin"
    TRADER = "trader"
    ANALYST = "analyst"
    VIEWER = "viewer"

class Permission(str, Enum):
    # 交易权限
    TRADE_EXECUTE = "trade:execute"
    TRADE_VIEW = "trade:view"
    # 信号权限
    SIGNAL_CREATE = "signal:create"
    SIGNAL_MODIFY = "signal:modify"
    SIGNAL_DELETE = "signal:delete"
    # 系统权限
    SYSTEM_CONFIG = "system:config"
    USER_MANAGE = "user:manage"

ROLE_PERMISSIONS: dict[Role, Set[Permission]] = {
    Role.ADMIN: set(Permission),
    Role.TRADER: {
        Permission.TRADE_EXECUTE, Permission.TRADE_VIEW,
        Permission.SIGNAL_CREATE, Permission.SIGNAL_VIEW
    },
    Role.ANALYST: {
        Permission.SIGNAL_CREATE, Permission.SIGNAL_MODIFY,
        Permission.SIGNAL_VIEW
    },
    Role.VIEWER: {Permission.SIGNAL_VIEW},
}

def check_permission(role: Role, permission: Permission) -> bool:
    return permission in ROLE_PERMISSIONS.get(role, set())

# FastAPI 依赖
def require_permission(permission: Permission):
    async def dependency(current_user: dict = Depends(get_current_user)):
        role = Role(current_user.get("role", "viewer"))
        if not check_permission(role, permission):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return dependency
```

### 14.3 API 限流

```python
# src/core/rate_limiter.py

from datetime import datetime, timedelta
from collections import defaultdict
import asyncio

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests: dict[str, list[datetime]] = defaultdict(list)
        self._lock = asyncio.Lock()

    async def is_allowed(self, key: str) -> bool:
        async with self._lock:
            now = datetime.utcnow()
            cutoff = now - timedelta(seconds=self.window_seconds)

            # 清理过期请求
            self.requests[key] = [
                req_time for req_time in self.requests[key]
                if req_time > cutoff
            ]

            if len(self.requests[key]) >= self.max_requests:
                return False

            self.requests[key].append(now)
            return True

# 速率限制配置
RATE_LIMITS = {
    "/api/v1/auth/login": (5, 60),       # 5次/分钟
    "/api/v1/trade/execute": (10, 60),    # 10次/分钟
    "/api/v1/sentiment/news": (30, 60),   # 30次/分钟
    "/api/v1/query": (100, 60),           # 100次/分钟
}
```

### 14.4 字段级加密

```python
# src/core/encryption.py

from cryptography.fernet import Fernet
from sqlalchemy import Column, String
from sqlalchemy.dialects.postgresql import BYTEA

class EncryptedField:
    """字段级加密装饰器"""

    def __init__(self, key: bytes):
        self.cipher = Fernet(key)

    def encrypt(self, value: str) -> bytes:
        if value is None:
            return None
        return self.cipher.encrypt(value.encode())

    def decrypt(self, encrypted_value: bytes) -> str:
        if encrypted_value is None:
            return None
        return self.cipher.decrypt(encrypted_value).decode()

# 使用示例
# API Keys, 交易密码等敏感字段应加密存储
SENSITIVE_FIELDS = [
    "api_key",
    "api_secret",
    "trade_password",
    "2fa_secret"
]
```

### 14.5 安全检查清单

| 检查项 | 描述 | 状态 |
|--------|------|------|
| 凭据管理 | 无硬编码凭据，使用环境变量或 K8s Secret | ✓ |
| 输入校验 | 所有用户输入使用 Pydantic validation | ✓ |
| SQL 注入 | 使用 ORM 参数化查询 | ✓ |
| XSS 防护 | API 返回数据 JSON 序列化，无 HTML 拼接 | ✓ |
| CSRF 保护 | 仅提供 API，无表单渲染 | ✓ |
| 限流 | 所有端点配置速率限制 | ✓ |
| 加密 | 敏感字段 Fernet 对称加密 | ✓ |
| 日志脱敏 | 禁止记录敏感信息（密码、API Key） | ✓ |

---

## 15. 容错恢复

### 15.1 熔断器模式

```python
# src/core/circuit_breaker.py

from datetime import datetime, timedelta
from enum import Enum
from typing import Callable, TypeVar, Any
import asyncio

class CircuitState(str, Enum):
    CLOSED = "closed"      # 正常
    OPEN = "open"          # 熔断
    HALF_OPEN = "half_open"  # 半开

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 60.0,
        expected_exception: type = Exception
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        self.failure_count = 0
        self.last_failure_time: datetime | None = None
        self.state = CircuitState.CLOSED
        self._lock = asyncio.Lock()

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        async with self._lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise CircuitBreakerOpen("Circuit breaker is OPEN")

            try:
                result = await func(*args, **kwargs)
                self._on_success()
                return result
            except self.expected_exception as e:
                self._on_failure()
                raise

    def _should_attempt_reset(self) -> bool:
        if self.last_failure_time is None:
            return True
        return (datetime.utcnow() - self.last_failure_time).total_seconds() >= self.recovery_timeout

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

class CircuitBreakerOpen(Exception):
    """熔断器打开异常"""
    pass
```

### 15.2 指数退避重试

```python
# src/core/retry.py

from datetime import datetime, timedelta
from typing import Callable, TypeVar, Any
import asyncio
import random

T = TypeVar('T')

class RetryConfig:
    def __init__(
        self,
        max_attempts: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: bool = True
    ):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter

async def with_retry(
    func: Callable[..., Any],
    config: RetryConfig,
    *args,
    **kwargs
) -> Any:
    last_exception = None

    for attempt in range(config.max_attempts):
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            last_exception = e

            if attempt < config.max_attempts - 1:
                delay = min(
                    config.base_delay * (config.exponential_base ** attempt),
                    config.max_delay
                )
                if config.jitter:
                    delay = delay * (0.5 + random.random())  # 0.5-1.5 倍抖动

                await asyncio.sleep(delay)

    raise last_exception

# 使用示例
retry_config = RetryConfig(
    max_attempts=3,
    base_delay=1.0,
    max_delay=30.0,
    jitter=True
)

result = await with_retry(
    llm_service.analyze,
    retry_config,
    news_content
)
```

### 15.3 优雅降级

```python
# src/core/degradation.py

from enum import Enum
from typing import Any, Optional
from dataclasses import dataclass

class DegradationLevel(str, Enum):
    FULL = "full"           # 完全正常
    DEGRADED = "degraded"    # 降级运行
    FALLBACK = "fallback"    # 使用后备方案
    OFFLINE = "offline"      # 完全离线

@dataclass
class DegradationResult:
    level: DegradationLevel
    data: Any
    source: str  # "primary", "cache", "static", "none"
    message: Optional[str] = None

class GracefulDegradation:
    def __init__(self):
        self.cache = RedisCache()  # 降级缓存
        self.fallback_data = {}    # 静态后备数据

    async def execute(
        self,
        primary_func: Callable,
        fallback_func: Optional[Callable] = None,
        cache_key: Optional[str] = None,
        cache_ttl: int = 300
    ) -> DegradationResult:
        # Step 1: 尝试从缓存获取
        if cache_key:
            cached = await self.cache.get(cache_key)
            if cached:
                return DegradationResult(
                    level=DegradationLevel.DEGRADED,
                    data=cached,
                    source="cache"
                )

        # Step 2: 尝试主函数
        try:
            result = await primary_func()

            # 写入缓存
            if cache_key:
                await self.cache.set(cache_key, result, ttl=cache_ttl)

            return DegradationResult(
                level=DegradationLevel.FULL,
                data=result,
                source="primary"
            )

        except Exception as e:
            # Step 3: 尝试后备函数
            if fallback_func:
                try:
                    result = await fallback_func()
                    return DegradationResult(
                        level=DegradationLevel.FALLBACK,
                        data=result,
                        source="fallback",
                        message=f"Primary failed: {str(e)}"
                    )
                except Exception:
                    pass

            # Step 4: 使用静态后备数据
            if cache_key and cache_key in self.fallback_data:
                return DegradationResult(
                    level=DegradationLevel.OFFLINE,
                    data=self.fallback_data[cache_key],
                    source="static",
                    message=f"All sources failed: {str(e)}"
                )

            raise

# 降级策略配置
DEGRADATION_STRATEGIES = {
    "sentiment_analysis": {
        "cache_ttl": 300,
        "fallback_enabled": True,
        "static_fallback": {"sentiment_score": 0.0, "confidence": 0.0}
    },
    "llm_analysis": {
        "cache_ttl": 600,
        "fallback_enabled": True,
        "static_fallback": {"summary": "Analysis unavailable", "keywords": []}
    }
}
```

### 15.4 事务补偿

```python
# src/core/saga.py

from typing import Callable, List, Any
from dataclasses import dataclass
from enum import Enum

class StepStatus(str, Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    COMPENSATED = "compensated"
    FAILED = "failed"

@dataclass
class SagaStep:
    name: str
    forward: Callable
    backward: Callable
    status: StepStatus = StepStatus.PENDING

class Saga:
    def __init__(self, name: str):
        self.name = name
        self.steps: List[SagaStep] = []
        self.completed_steps: List[SagaStep] = []

    def add_step(self, name: str, forward: Callable, backward: Callable):
        self.steps.append(SagaStep(name=name, forward=forward, backward=backward))
        return self

    async def execute(self, *args, **kwargs) -> Any:
        for step in self.steps:
            try:
                result = await step.forward(*args, **kwargs)
                step.status = StepStatus.COMPLETED
                self.completed_steps.append(step)
            except Exception as e:
                # 补偿已完成的步骤
                await self.compensate()
                raise SagaExecutionError(f"Saga {self.name} failed at {step.name}: {e}")

        return result

    async def compensate(self):
        """反向补偿已完成的步骤"""
        for step in reversed(self.completed_steps):
            try:
                await step.backward()
                step.status = StepStatus.COMPENSATED
            except Exception as e:
                # 记录补偿失败，需要人工介入
                step.status = StepStatus.FAILED
                print(f"Compensation failed for {step.name}: {e}")

class SagaExecutionError(Exception):
    pass

# 使用示例: 创建订单 Saga
order_saga = Saga("create_order")
order_saga.add_step(
    name="reserve_capital",
    forward=lambda: reserve_capital(amount),
    backward=lambda: release_capital(amount)
)
order_saga.add_step(
    name="send_order",
    forward=lambda: trading_gateway.send_order(order),
    backward=lambda: trading_gateway.cancel_order(order_id)
)
order_saga.add_step(
    name="update_position",
    forward=lambda: position_manager.add_position(order),
    backward=lambda: position_manager.remove_position(order_id)
)
```

### 15.5 健康检查与自愈

```bash
# deployment/scripts/healthcheck.sh

check_http_endpoint() {
    local url="$1"
    local name="$2"
    if curl -sf --max-time 10 "${url}" > /dev/null; then
        log "${name}: OK"
        return 0
    else
        error "${name}: FAILED"
        return 1
    fi
}

check_api_health() {
    check_http_endpoint "${API_URL}/health" "API Health"
    check_http_endpoint "${API_URL}/ready" "API Readiness"
    check_http_endpoint "${API_URL}/metrics" "API Metrics"
}

check_vnpy_health() {
    check_http_endpoint "${VNPY_URL}/health" "vnpy Health"
}

check_database_health() {
    check_tcp_port "${POSTGRES_HOST}" "${POSTGRES_PORT}" "PostgreSQL"
    check_tcp_port "${REDIS_HOST}" "${REDIS_PORT}" "Redis"
}
```

---

## 16. 性能优化

### 16.1 连接池配置

```python
# src/core/connection_pools.py

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
import asyncpg
import redis.asyncio as redis

# PostgreSQL 连接池
async def create_postgres_pool(
    dsn: str,
    pool_size: int = 20,
    max_overflow: int = 10,
    pool_timeout: int = 30
):
    engine = create_async_engine(
        dsn,
        pool_size=pool_size,
        max_overflow=max_overflow,
        pool_timeout=pool_timeout,
        pool_pre_ping=True,  # 连接前 ping
        pool_recycle=3600,   # 1小时回收连接
        echo=False
    )

    async_session = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False
    )

    return engine, async_session

# Redis 连接池
async def create_redis_pool(
    url: str,
    max_connections: int = 50,
    socket_timeout: float = 5.0,
    socket_connect_timeout: float = 5.0
):
    pool = redis.ConnectionPool.from_url(
        url,
        max_connections=max_connections,
        socket_timeout=socket_timeout,
        socket_connect_timeout=socket_connect_timeout,
        decode_responses=True,
        health_check_interval=30
    )
    return redis.Redis(connection_pool=pool)

# Milvus 连接池
from pymilvus import connections, Collection

class MilvusConnectionPool:
    def __init__(self, host: str, port: int, pool_size: int = 10):
        self.host = host
        self.port = port
        self.pool_size = pool_size
        self._connections = []
        self._available = []
        self._lock = asyncio.Lock()

    async def initialize(self):
        for i in range(self.pool_size):
            conn = connections.add_connection(
                f"conn_{i}",
                host=self.host,
                port=self.port
            )
            self._connections.append(conn)
            self._available.append(conn)

    async def acquire(self):
        async with self._lock:
            if not self._available:
                raise RuntimeError("No available connections")
            conn = self._available.pop(0)
            return conn

    async def release(self, conn):
        async with self._lock:
            self._available.append(conn)
```

### 16.2 多级缓存策略

```python
# src/core/cache.py

from typing import Any, Optional, Callable
from datetime import datetime, timedelta
import asyncio
import hashlib
import json

class CacheLevel:
    L1 = "l1"  # 进程内内存
    L2 = "l2"  # Redis 分布式缓存
    L3 = "l3"  # 数据库持久化

class MultiLevelCache:
    def __init__(self, redis_client, default_ttl: int = 300):
        self.redis = redis_client
        self.default_ttl = default_ttl
        self.l1_cache: dict = {}
        self.l1_max_size = 1000
        self._lock = asyncio.Lock()

    def _make_key(self, prefix: str, key: str) -> str:
        return f"cache:{prefix}:{key}"

    def _make_l1_key(self, prefix: str, key: str) -> str:
        return f"{prefix}:{key}"

    async def get(self, prefix: str, key: str) -> Optional[Any]:
        # L1: 进程内缓存 (最快)
        l1_key = self._make_l1_key(prefix, key)
        if l1_key in self.l1_cache:
            return self.l1_cache[l1_key]

        # L2: Redis 缓存
        l2_key = self._make_key(prefix, key)
        value = await self.redis.get(l2_key)

        if value:
            # 回填 L1
            await self._set_l1(l1_key, json.loads(value))
            return json.loads(value)

        return None

    async def set(
        self,
        prefix: str,
        key: str,
        value: Any,
        ttl: Optional[int] = None
    ):
        ttl = ttl or self.default_ttl

        # 设置 L1
        await self._set_l1(self._make_l1_key(prefix, key), value)

        # 设置 L2
        l2_key = self._make_key(prefix, key)
        await self.redis.setex(l2_key, ttl, json.dumps(value))

    async def _set_l1(self, key: str, value: Any):
        async with self._lock:
            if len(self.l1_cache) >= self.l1_max_size:
                # LRU 淘汰
                oldest_key = next(iter(self.l1_cache))
                del self.l1_cache[oldest_key]
            self.l1_cache[key] = value

    async def delete(self, prefix: str, key: str):
        l1_key = self._make_l1_key(prefix, key)
        l2_key = self._make_key(prefix, key)

        async with self._lock:
            self.l1_cache.pop(l1_key, None)

        await self.redis.delete(l2_key)

    async def invalidate_prefix(self, prefix: str):
        """使缓前缀下所有缓存失效"""
        pattern = f"cache:{prefix}:*"
        cursor = 0
        while True:
            cursor, keys = await self.redis.scan(cursor, match=pattern, count=100)
            if keys:
                await self.redis.delete(*keys)
            if cursor == 0:
                break

        async with self._lock:
            self.l1_cache = {
                k: v for k, v in self.l1_cache.items()
                if not k.startswith(f"{prefix}:")
            }

# 缓存键命名规范
CACHE_KEYS = {
    "sentiment": "sentiment:{vt_symbol}:latest",
    "signal": "signal:{signal_id}",
    "portfolio": "portfolio:{account_id}:positions",
    "news": "news:{news_id}",
}
```

### 16.3 批量处理优化

```python
# src/core/batch_processor.py

from typing import Callable, List, Any
from datetime import datetime
import asyncio
from dataclasses import dataclass

@dataclass
class BatchConfig:
    max_size: int = 100
    max_wait_ms: int = 100
    max_workers: int = 4

class BatchProcessor:
    def __init__(
        self,
        process_func: Callable,
        config: BatchConfig
    ):
        self.process_func = process_func
        self.config = config
        self.buffer: List[Any] = []
        self.last_process_time = datetime.utcnow()
        self._lock = asyncio.Lock()
        self._triggered = asyncio.Event()

    async def add(self, item: Any) -> None:
        async with self._lock:
            self.buffer.append(item)

            should_process = (
                len(self.buffer) >= self.config.max_size or
                self._should_flush()
            )

            if should_process:
                await self._process_buffer()

    def _should_flush(self) -> bool:
        elapsed = (datetime.utcnow() - self.last_process_time).total_seconds() * 1000
        return elapsed >= self.config.max_wait_ms

    async def _process_buffer(self):
        if not self.buffer:
            return

        batch = self.buffer[:self.config.max_size]
        self.buffer = self.buffer[self.config.max_size:]
        self.last_process_time = datetime.utcnow()

        # 并发处理
        chunk_size = len(batch) // self.config.max_workers + 1
        chunks = [
            batch[i:i + chunk_size]
            for i in range(0, len(batch), chunk_size)
        ]

        await asyncio.gather(
            *[self.process_func(chunk) for chunk in chunks]
        )

    async def flush(self):
        """手动刷新缓冲区"""
        async with self._lock:
            while self.buffer:
                await self._process_buffer()

# 使用示例: 批量发送订单
async def process_order_batch(orders: List[Order]):
    # 合并同一标的的订单
    grouped = group_by_symbol(orders)
    for symbol, symbol_orders in grouped.items():
        # 批量下单
        await gateway.send_batch_orders(symbol_orders)

order_processor = BatchProcessor(
    process_func=process_order_batch,
    config=BatchConfig(max_size=50, max_wait_ms=100, max_workers=4)
)
```

### 16.4 异步 I/O 优化

```python
# src/core/async_utils.py

import asyncio
from typing import List, Any, Callable
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_connection(pool):
    """异步上下文管理器管理连接"""
    conn = await pool.acquire()
    try:
        yield conn
    finally:
        await pool.release(conn)

async def gather_with_concurrency(
    n: int,
    *tasks,
    return_exceptions: bool = False
) -> List[Any]:
    """限制并发数的 gather"""
    semaphore = asyncio.Semaphore(n)

    async def sem_task(task):
        async with semaphore:
            return await task

    return await asyncio.gather(
        *(sem_task(task) for task in tasks),
        return_exceptions=return_exceptions
    )

class AsyncBatcher:
    """异步批量处理器，支持动态批处理"""

    def __init__(
        self,
        batch_size: int = 100,
        max_wait_ms: int = 50,
        process_fn: Callable = None
    ):
        self.batch_size = batch_size
        self.max_wait_ms = max_wait_ms
        self.process_fn = process_fn
        self._queue: asyncio.Queue = asyncio.Queue()
        self._task: asyncio.Task = None

    async def start(self):
        self._task = asyncio.create_task(self._process_loop())

    async def stop(self):
        await self._queue.join()
        self._task.cancel()

    async def add(self, item):
        await self._queue.put(item)

    async def _process_loop(self):
        while True:
            batch = []
            try:
                # 带超时的获取
                item = await asyncio.wait_for(
                    self._queue.get(),
                    timeout=self.max_wait_ms / 1000
                )
                batch.append(item)

                # 尽量填满 batch
                while len(batch) < self.batch_size and not self._queue.empty():
                    try:
                        batch.append(self._queue.get_nowait())
                    except asyncio.QueueEmpty:
                        break

                if self.process_fn:
                    await self.process_fn(batch)

                for _ in batch:
                    self._queue.task_done()

            except asyncio.TimeoutError:
                if batch and self.process_fn:
                    await self.process_fn(batch)
                    for _ in batch:
                        self._queue.task_done()
            except asyncio.CancelledError:
                break
```

### 16.5 性能指标

| 优化项 | 指标目标 | 实现方式 |
|--------|----------|----------|
| API P99 延迟 | < 200ms | 连接池、缓存、异步 I/O |
| 数据库查询 | < 50ms | 索引优化、查询缓存 |
| Redis 操作 | < 5ms | L1/L2 多级缓存 |
| 事件处理吞吐 | > 10000 events/s | 批量处理、并发控制 |
| 内存使用 | < 4GB | 对象池化、及时释放 |
| CPU 利用率 | 60-80% | 协程池、工作负载均衡 |

---
