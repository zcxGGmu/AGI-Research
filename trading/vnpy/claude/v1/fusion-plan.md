# FinnewsHunter x VeighNa (vnpy) 融合方案

> **版本**: v2.0 | **日期**: 2026-03-21 | **状态**: 设计评审

---

## 一、项目概览

### FinnewsHunter — 新闻驱动的智能投研平台

| 维度 | 描述 |
|------|------|
| **定位** | 中国金融新闻采集 + 多智能体辩论式投研分析 |
| **技术栈** | Python FastAPI + React/TS + PostgreSQL + Redis + Milvus + Neo4j + Celery |
| **核心能力** | 10+ 新闻源爬虫、多 LLM 情感分析、Bull/Bear 辩论系统、Alpha 因子挖掘（DSL+RL）、知识图谱 |
| **数据流** | 新闻源 → 爬虫 → 存储(PG/Milvus/Neo4j) → LLM 分析 → 辩论决策 → SSE 实时推送前端 |
| **交易能力** | 仅产出投资建议（强烈推荐/推荐/中性/谨慎/回避），**无实际下单能力** |

### VeighNa (vnpy v4.3.0) — 全功能量化交易平台

| 维度 | 描述 |
|------|------|
| **定位** | 事件驱动的量化交易框架，支持实盘+回测 |
| **技术栈** | Python 3.10-3.13 + PySide6(Qt6) + ZeroMQ + Polars/Pandas + PyTorch |
| **核心能力** | 25+ 交易所网关、事件引擎、CTA/组合/价差策略、Alpha 研究(101/158 因子)、ML 模型(Lasso/LGB/MLP) |
| **数据流** | 交易所行情 → Gateway → EventEngine → Strategy → OrderRequest → Gateway → 交易所 |
| **新闻能力** | **完全没有**新闻/舆情/NLP 集成 |

### 互补性分析

```
FinnewsHunter                          VeighNa (vnpy)
┌──────────────────────┐              ┌──────────────────────┐
│  新闻采集 (10+ 源)    │              │  交易所网关 (25+)     │
│  LLM 情感分析         │              │  事件驱动引擎         │
│  多智能体辩论系统      │   ════════>  │  策略框架 (CTA/Alpha) │
│  知识图谱 (Neo4j)     │   互补融合    │  订单管理系统 (OMS)   │
│  Alpha 因子挖掘 (DSL)  │  <════════  │  风控系统             │
│  向量搜索 (Milvus)    │              │  回测引擎             │
│  Web UI (React)       │              │  桌面 GUI (Qt6)       │
│                       │              │  Alpha 研究 (ML)      │
│  ❌ 无交易执行能力     │              │  ❌ 无新闻/情感数据    │
└──────────────────────┘              └──────────────────────┘
```

**核心结论：两个系统形成完美的「感知-决策-执行」闭环补位。**

---

## 二、融合目标

> **将 FinnewsHunter 的新闻情感智能作为 vnpy 策略系统的"第六感"，使量化策略能够感知市场舆情并自动化执行交易。**

具体目标：

1. **新闻情感因子注入 vnpy 策略** — 将新闻情感分数、辩论评级作为可用信号源
2. **事件驱动的新闻触发交易** — 重大新闻自动触发策略调仓
3. **Alpha 因子体系统一** — 合并两侧的因子挖掘能力
4. **辩论系统 → 交易信号** — 辩论结论直接生成可执行的交易指令
5. **统一回测** — 在 vnpy 回测引擎中使用历史新闻数据

---

## 三、融合架构

### 3.1 总体架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Unified Trading Platform                          │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                    vnpy EventEngine (核心事件总线)                    │    │
│  │                                                                     │    │
│  │  原有事件:  EVENT_TICK. | EVENT_ORDER. | EVENT_TRADE. | EVENT_LOG   │    │
│  │  新增事件:  EVENT_NEWS. | EVENT_SENTIMENT. | EVENT_DEBATE_SIGNAL.   │    │
│  │                                                                     │    │
│  │  [单线程消费 Queue, 支持 type-specific + general handlers]           │    │
│  └─────────┬──────────────────────────────────────────┬──────────────┘    │
│            │                                          │                    │
│  ┌─────────▼──────────────┐     ┌─────────────────────▼────────────┐     │
│  │  Market Data Layer      │     │  News Intelligence Layer          │     │
│  │                         │     │  (BaseApp: vnpy_newsengine)       │     │
│  │  CTP Gateway            │     │                                    │     │
│  │  IB Gateway             │     │  NewsEngine (BaseEngine)           │     │
│  │  XTP Gateway            │     │    ├─ CrawlThread (爬虫线程池)    │     │
│  │  ... (25+ gateways)     │     │    ├─ LLMThread (LLM 调用线程池)  │     │
│  │                         │     │    ├─ NewsCacheManager (内存缓存)  │     │
│  │  [BaseGateway 子类]     │     │    └─ NewsDatabase (SQLite/PG)    │     │
│  └─────────┬───────────────┘     │                                    │     │
│            │                     │  DebateEngine (BaseEngine, 可选)   │     │
│            │                     │    ├─ Bull/Bear/Manager Agents     │     │
│            │                     │    └─ SignalConverter              │     │
│            │                     └──────────────────┬────────────────┘     │
│            │                                        │                      │
│  ┌─────────▼────────────────────────────────────────▼──────────────┐     │
│  │                    Strategy Layer (策略层)                         │     │
│  │                                                                    │     │
│  │  CTA Strategies (价量)        — CtaTemplate                       │     │
│  │  Alpha Strategies (ML 因子)   — AlphaStrategy                     │     │
│  │  News Strategies (新闻驱动)   — NewsStrategyTemplate  ← 新增      │     │
│  │  Hybrid Strategies (混合)     — 组合 BarGenerator + SentimentMgr  │     │
│  └──────────────────────────┬────────────────────────────────────────┘     │
│                             │                                              │
│  ┌──────────────────────────▼────────────────────────────────────────┐     │
│  │                    Execution Layer (执行层)                         │     │
│  │  OmsEngine → OffsetConverter → BaseGateway → Exchange              │     │
│  │  RiskManager (新闻情感风控增强)                                     │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                    Storage Layer (存储层)                           │     │
│  │  SQLite/PG (新闻+情感) | Parquet (行情+因子) | [可选] Neo4j (图谱)  │     │
│  └───────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键架构修正：使用 BaseEngine + BaseApp 而非 BaseGateway

> **重要决策：** vnpy 的 `BaseGateway` 是为交易所连接设计的，它强制要求实现 `send_order()`、`cancel_order()`、`query_account()`、`query_position()` 等 6 个抽象方法。新闻引擎无需交易接口，因此**不应**继承 `BaseGateway`。
>
> 正确做法：新闻引擎作为 `BaseEngine` 子类注册，通过 `BaseApp` 描述符挂载到 `MainEngine`。这与 vnpy 官方扩展（如 `vnpy_ctastrategy`、`vnpy_riskmanager`）的模式一致。

```python
# 正确的挂载方式：
main_engine = MainEngine(event_engine)
main_engine.add_gateway(CtpGateway)       # 交易网关 — BaseGateway
main_engine.add_app(NewsEngineApp)        # 新闻引擎 — BaseApp + BaseEngine
main_engine.add_app(DebateEngineApp)      # 辩论引擎 — BaseApp + BaseEngine (可选)
main_engine.add_app(NewsStrategyApp)      # 新闻策略 — BaseApp + BaseEngine
```

---

## 四、详细技术规格

### 4.1 事件定义

遵循 vnpy 事件命名规范：**带尾部 `.` 的事件支持 per-symbol 订阅**。

```python
# vnpy_newsengine/event.py

EVENT_NEWS = "eNews."                    # 新闻事件，支持 EVENT_NEWS + vt_symbol 精确订阅
EVENT_SENTIMENT = "eSentiment."          # 情感分析事件
EVENT_DEBATE_SIGNAL = "eDebateSignal."   # 辩论信号事件
EVENT_NEWS_LOG = "eNewsLog"              # 新闻引擎日志（无尾部点号 = 仅全局）
```

### 4.2 数据对象定义

严格遵循 vnpy `BaseData` 模式（`@dataclass`，`gateway_name` 必填，`__post_init__` 计算衍生字段）：

```python
# vnpy_newsengine/object.py
from dataclasses import dataclass, field
from datetime import datetime
from vnpy.trader.object import BaseData

@dataclass
class NewsData(BaseData):
    """新闻数据 — 对标 FinnewsHunter 的 NewsItem"""

    news_id: str = ""
    title: str = ""
    content: str = ""
    source: str = ""                       # sina / eastmoney / tencent / ...
    publish_time: datetime = None
    stock_codes: list[str] = field(default_factory=list)
    url: str = ""

    # 衍生字段
    vt_symbols: list[str] = field(default=None, init=False)

    def __post_init__(self) -> None:
        """将 stock_codes (如 '600000') 转为 vt_symbols (如 '600000.SSE')"""
        self.vt_symbols = [
            self._to_vt_symbol(code) for code in self.stock_codes
        ]

    @staticmethod
    def _to_vt_symbol(code: str) -> str:
        """A 股代码 → vt_symbol 转换"""
        if code.startswith(("6", "9")):
            return f"{code}.SSE"
        elif code.startswith(("0", "2", "3")):
            return f"{code}.SZSE"
        elif code.startswith(("4", "8")):
            return f"{code}.BSE"
        return f"{code}.SSE"


@dataclass
class SentimentData(BaseData):
    """情感分析数据 — 对标 FinnewsHunter NewsAnalystAgent 输出"""

    news_id: str = ""
    vt_symbol: str = ""                    # 关联标的
    sentiment_score: float = 0.0           # -1.0 (极度看空) ~ +1.0 (极度看多)
    confidence: float = 0.0                # 0.0 ~ 1.0
    impact_level: str = "low"              # high / medium / low
    recommendation: str = "neutral"        # strong_buy / buy / neutral / cautious / avoid
    analysis_summary: str = ""
    publish_time: datetime = None          # 原始新闻发布时间


@dataclass
class DebateSignalData(BaseData):
    """辩论系统产出的交易信号"""

    vt_symbol: str = ""
    rating: str = ""                       # 强烈推荐 / 推荐 / 中性 / 谨慎 / 回避
    rating_score: float = 0.0              # -1.0 ~ +1.0 (量化评级)
    bull_score: float = 0.0                # 多方论证评分 1-10
    bear_score: float = 0.0                # 空方论证评分 1-10
    data_sufficiency: float = 0.0          # 数据充分度 0-1
    key_reasons: list[str] = field(default_factory=list)
    debate_rounds: int = 0
    was_interrupted: bool = False          # 经理是否提前中断辩论
    execution_time: float = 0.0            # 辩论耗时（秒）
    signal_time: datetime = None
```

### 4.3 NewsEngine — 核心引擎

```python
# vnpy_newsengine/engine.py

class NewsEngine(BaseEngine):
    """
    新闻智能引擎 — 管理爬虫、情感分析、数据缓存。

    线程模型：
    - crawl_pool: ThreadPoolExecutor(max_workers=3) 爬虫线程池
    - llm_pool:   ThreadPoolExecutor(max_workers=2) LLM 调用线程池
    - 两个线程池独立运行，通过 Queue 解耦
    - 最终通过 EventEngine.put() 推送事件（线程安全）
    """

    engine_name = "news"

    def __init__(self, main_engine: MainEngine, event_engine: EventEngine) -> None:
        super().__init__(main_engine, event_engine, self.engine_name)

        # --- 内存缓存 ---
        self.news_cache: dict[str, NewsData] = {}                    # news_id -> NewsData
        self.sentiment_cache: dict[str, list[SentimentData]] = {}    # vt_symbol -> [SentimentData]
        self.seen_urls: set[str] = set()                             # 去重

        # --- 线程池 ---
        self.crawl_pool: ThreadPoolExecutor = None
        self.llm_pool: ThreadPoolExecutor = None
        self.crawl_queue: Queue = Queue()       # NewsData 待分析队列
        self.active: bool = False

        # --- 配置 ---
        self.setting: dict = {}
        self.crawlers: dict[str, BaseCrawler] = {}
        self.llm_client: LLMClient = None
        self.news_db: NewsDatabase = None

        # --- 注册事件处理 ---
        self.event_engine.register(EVENT_TIMER, self._on_timer)

    def start(self, setting: dict) -> None:
        """启动新闻引擎"""
        ...

    def stop(self) -> None:
        """停止新闻引擎，清理线程池"""
        ...

    def close(self) -> None:
        """BaseEngine 生命周期回调"""
        self.stop()

    # --- 主动 API ---
    def get_news(self, news_id: str) -> NewsData | None: ...
    def get_latest_news(self, vt_symbol: str, limit: int = 20) -> list[NewsData]: ...
    def get_sentiment(self, vt_symbol: str) -> SentimentData | None: ...
    def get_sentiment_history(self, vt_symbol: str, limit: int = 100) -> list[SentimentData]: ...
    def trigger_crawl(self, sources: list[str] = None) -> None: ...
    def trigger_debate(self, vt_symbol: str) -> None: ...

    # --- 事件推送（线程安全，可从任何线程调用）---
    def _push_news_event(self, news: NewsData) -> None:
        """推送新闻事件 — 双事件模式：全局 + per-symbol"""
        self.event_engine.put(Event(EVENT_NEWS, news))
        for vt_symbol in news.vt_symbols:
            self.event_engine.put(Event(EVENT_NEWS + vt_symbol, news))

    def _push_sentiment_event(self, sentiment: SentimentData) -> None:
        self.event_engine.put(Event(EVENT_SENTIMENT, sentiment))
        self.event_engine.put(Event(EVENT_SENTIMENT + sentiment.vt_symbol, sentiment))

    # --- 定时器回调（在 EventEngine 线程上执行）---
    def _on_timer(self, event: Event) -> None:
        """每 N 秒触发一次爬虫检查"""
        ...
```

### 4.4 线程模型与并发设计

```
                        ┌─────────────────────────────────┐
                        │       EventEngine Thread         │
                        │  (单线程顺序处理所有事件)          │
                        │                                   │
                        │  _on_timer() ──→ 触发爬虫调度     │
                        │  Strategy.on_sentiment() 回调     │
                        │  OmsEngine.process_xxx() 回调     │
                        └────────────┬──────────────────────┘
                                     │ put(Event)
                                     │ (线程安全)
    ┌────────────────────────────────┼────────────────────────────────┐
    │                                │                                │
    ▼                                ▼                                ▼
┌───────────────┐          ┌─────────────────┐          ┌──────────────────┐
│ CrawlThread-1  │          │ CrawlThread-2    │          │ LLM Thread-1      │
│ (sina 爬虫)    │          │ (eastmoney 爬虫) │          │ (情感分析)         │
│                │          │                   │          │                    │
│ requests.get() │          │ requests.get()    │          │ litellm.complete() │
│ BeautifulSoup  │          │ BeautifulSoup     │          │ → SentimentData    │
│ → NewsData     │          │ → NewsData        │          │ → put(EVENT_SENT.) │
│ → put(EVENT_N.)│          │ → put(EVENT_N.)   │          │                    │
└───────────────┘          └─────────────────┘          └──────────────────┘

关键约束：
1. EventEngine 是单线程消费者 — handler 必须快速返回（< 10ms）
2. LLM 调用（1-10s）绝不能在 EventEngine 线程上执行
3. 爬虫 HTTP 请求（0.5-5s）绝不能在 EventEngine 线程上执行
4. 所有长耗时操作提交到 ThreadPoolExecutor
5. 唯一的跨线程通信方式：EventEngine.put(Event)
```

### 4.5 AgenticX 剥离方案

FinnewsHunter 的核心逻辑被 AgenticX 框架包裹，但实际有效代码很少依赖框架能力。以下是精确的逐组件剥离策略：

| 原始组件 | AgenticX 依赖 | 替换策略 | 实际改动量 |
|----------|---------------|----------|------------|
| `BaseCrawler` | 继承 `BaseTool`，使用 `ToolMetadata` | 改为普通 Python 类，删除 `execute()/aexecute()` 方法 | ~20 行 |
| `NewsAnalystAgent` | 继承 `Agent(BaseModel)`，用 `AgentExecutor` | 改为普通类。关键发现：**`AgentExecutor` 被初始化但完全没用到**，实际 LLM 调用走的是 `_llm_provider.invoke()` 直接调用 | ~30 行 |
| `Bull/Bear/Manager Agents` | 继承 `Agent(BaseModel)`，用 `object.__setattr__` hack 绕过 Pydantic 限制 | 改为普通类，直接 `self.llm_client = llm_client` | ~15 行/Agent |
| `DebateOrchestrator` | 无直接 AgenticX 依赖，但通过 Agent 间接依赖 | 替换 Agent 实例化方式即可 | ~10 行 |
| `LLMService` | 使用 `LiteLLMProvider` / `BailianProvider` | 替换为 `litellm.acompletion()` 直接调用或 `openai` SDK | ~50 行 |

**LLM 抽象层替代方案：**

```python
# vnpy_newsengine/llm/client.py

from dataclasses import dataclass
import litellm   # pip install litellm — 支持 100+ LLM 提供商的统一接口

@dataclass(frozen=True)
class LLMConfig:
    provider: str = "deepseek"
    model: str = "deepseek-chat"
    api_key: str = ""
    base_url: str = ""
    temperature: float = 0.7
    max_tokens: int = 4096
    timeout: int = 30

class LLMClient:
    """统一 LLM 客户端 — 替代 AgenticX 的 LiteLLMProvider"""

    def __init__(self, config: LLMConfig) -> None:
        self.config = config
        # litellm 支持通过 model 前缀自动路由：
        # "deepseek/deepseek-chat", "openai/gpt-4o", "anthropic/claude-3-5-sonnet"
        self.model_id = f"{config.provider}/{config.model}"

    def invoke(self, messages: list[dict]) -> str:
        """同步调用（用于线程池）"""
        response = litellm.completion(
            model=self.model_id,
            messages=messages,
            api_key=self.config.api_key,
            base_url=self.config.base_url or None,
            temperature=self.config.temperature,
            max_tokens=self.config.max_tokens,
            timeout=self.config.timeout,
        )
        return response.choices[0].message.content

    async def ainvoke(self, messages: list[dict]) -> str:
        """异步调用（用于辩论引擎的 asyncio.gather）"""
        response = await litellm.acompletion(
            model=self.model_id,
            messages=messages,
            api_key=self.config.api_key,
            base_url=self.config.base_url or None,
            temperature=self.config.temperature,
            max_tokens=self.config.max_tokens,
            timeout=self.config.timeout,
        )
        return response.choices[0].message.content
```

### 4.6 配置映射

vnpy 的配置通过 `.vntrader/vt_setting.json` 管理。新增字段：

```python
# 在 SETTINGS dict 中新增（或在插件自己的 JSON 配置中）

NEWS_SETTINGS = {
    # --- LLM ---
    "news.llm.provider": "deepseek",         # deepseek / openai / qwen / moonshot / zhipu / anthropic
    "news.llm.model": "deepseek-chat",
    "news.llm.api_key": "",
    "news.llm.base_url": "",
    "news.llm.temperature": 0.7,
    "news.llm.max_tokens": 4096,
    "news.llm.timeout": 30,

    # --- 爬虫 ---
    "news.crawl.sources": "sina,eastmoney,tencent",   # 逗号分隔
    "news.crawl.interval": 60,                         # 秒
    "news.crawl.timeout": 30,
    "news.crawl.max_retries": 3,
    "news.crawl.user_agent": "Mozilla/5.0 ...",

    # --- 情感分析 ---
    "news.sentiment.enabled": True,
    "news.sentiment.min_confidence": 0.5,    # 低于此置信度不推送
    "news.sentiment.cache_ttl": 3600,        # 秒

    # --- 存储 ---
    "news.storage.type": "sqlite",           # sqlite / postgresql
    "news.storage.path": "news.db",          # SQLite 路径（相对于 .vntrader）

    # --- 辩论（可选）---
    "news.debate.enabled": False,
    "news.debate.mode": "parallel_then_summarize",
    "news.debate.max_rounds": 3,
    "news.debate.auto_trigger": False,       # 是否新闻自动触发辩论
}
```

---

## 五、数据流设计

### 5.1 实时新闻 → 交易执行（详细流程）

```
新闻源 (Sina, EastMoney, Tencent, Yicai, NBD, ...)
     │
     ▼
[CrawlThread Pool]  ←── EVENT_TIMER 每 N 秒触发调度
     │
     ├── 1. requests.get() + 编码检测 (gb2312/gbk/utf-8)
     ├── 2. BeautifulSoup CSS 选择器内容提取
     ├── 3. URL 去重检查 (seen_urls: set)
     ├── 4. 股票代码识别 (URL 关键词 + 标题关键词)
     │
     ▼
NewsData 创建
     │
     ├──→ EventEngine.put(EVENT_NEWS, news)           # 全局新闻事件
     ├──→ EventEngine.put(EVENT_NEWS + vt_symbol, news) # Per-symbol 新闻事件（可多个）
     ├──→ news_cache[news_id] = news                  # 内存缓存
     ├──→ news_db.save(news)                          # 持久化
     │
     └──→ crawl_queue.put(news)   ──→   [LLM Thread Pool]
                                              │
                                              ├── 构建情感分析 Prompt（复用 FinnewsHunter 模板）
                                              ├── litellm.completion() → 结构化输出
                                              ├── 正则提取 sentiment_score / confidence
                                              │
                                              ▼
                                        SentimentData 创建
                                              │
                                              ├──→ EventEngine.put(EVENT_SENTIMENT + vt_symbol)
                                              ├──→ sentiment_cache[vt_symbol].append(data)
                                              │
                                              └──→ [可选] 高影响事件触发辩论
                                                        │
                                                        ▼
                                                   DebateEngine（异步执行）
                                                        │
                                                        ├── DataCollector 聚合新闻+行情
                                                        ├── Bull asyncio.gather Bear
                                                        ├── [可选] 多轮辩论
                                                        ├── Manager 裁决
                                                        │
                                                        ▼
                                                   DebateSignalData
                                                        │
                                                        └──→ EventEngine.put(EVENT_DEBATE_SIGNAL)

═══════════════════════════════════════════════════════════════════════════

EventEngine 单线程消费:

EVENT_SENTIMENT + "600000.SSE"
     │
     ▼
NewsStrategyEngine 分发到对应策略
     │
     ▼
Strategy.on_sentiment(sentiment)
     │
     ├── SentimentManager.update()
     ├── 极端事件检查 → on_breaking_news()
     │
     │  [下一个 EVENT_TICK / EVENT_BAR 到来时]
     ▼
Strategy.on_bar(bar)
     │
     ├── ArrayManager 价量指标
     ├── SentimentManager 情感指标
     ├── 混合决策逻辑
     │
     ▼
self.buy() / self.sell()
     │
     ▼
MainEngine.send_order(req, "CTP")
     │
     ▼
CTP Gateway → 交易所
```

### 5.2 回测数据流（Alpha 因子路径）

```
Historical News + Sentiment (SQLite/PG)
     │
     ▼
sentiment_to_dataproxy() 转换工具
     │
     ├── 按 [datetime, vt_symbol] 聚合日频情感均值
     ├── 输出 pl.DataFrame[datetime, vt_symbol, data]   ← DataProxy 标准 schema
     │
     ▼
AlphaDataset.add_feature("sentiment_avg", result=sentiment_df)     ← Approach B: precomputed
AlphaDataset.add_feature("news_count",    result=news_count_df)
AlphaDataset.add_feature("sent_momentum", "ts_delta(sentiment_avg, 5)")  ← Approach A: expression
     │
     ▼
dataset.prepare_data()     # 多进程并行计算
     │
     ▼
AlphaModel.fit(dataset)   # LightGBM / MLP 训练
     │
     ▼
signal_df = model.predict(dataset, Segment.TEST)   # [datetime, vt_symbol, signal]
     │
     ▼
BacktestingEngine.add_strategy(SentimentAlphaStrategy, signal_df=signal_df)
BacktestingEngine.run_backtesting()
     │
     ▼
calculate_statistics()  →  Sharpe, MaxDD, IC, Rank IC, ...
```

### 5.3 CTA 回测路径（事件驱动）

```
Historical BarData (vnpy database)       Historical SentimentData (news database)
         │                                          │
         ▼                                          ▼
  按时间排序的 BarData 序列              按时间排序的 SentimentData 序列
         │                                          │
         └──────────────┬───────────────────────────┘
                        │
                        ▼
              NewsBacktestingEngine
                        │
   for dt in sorted(all_timestamps):
                        │
         ├── if dt has bars:    strategy.on_bar(bar)
         ├── if dt has sents:   strategy.on_sentiment(sentiment)
         └── cross_order()      # 订单撮合
                        │
                        ▼
              calculate_statistics()
```

---

## 六、Alpha 因子体系统一

### 6.1 现状对比

| 维度 | FinnewsHunter | vnpy Alpha |
|------|---------------|------------|
| **因子类型** | 情感因子 (SENTIMENT, NEWS_COUNT) + 自定义 DSL | 价量因子 (Alpha 101/158) |
| **计算引擎** | FactorVM (PyTorch 栈式虚拟机) | Polars 表达式引擎 (DataProxy + eval) |
| **表达式系统** | Token 序列 → FactorVM 执行 | Python 字符串 → eval() 执行 |
| **ML 模型** | Transformer + RL (因子发现) | Lasso / LightGBM / MLP (因子组合) |
| **评估指标** | Sortino, Sharpe, IC, Rank IC, MaxDD, Turnover | Sharpe, Drawdown, Returns |
| **数据格式** | PyTorch Tensor `[batch, time_steps]` | Polars DataFrame `[datetime, vt_symbol, data]` |

### 6.2 统一方案：四种集成路径

#### 路径 A：预计算结果注入（推荐，零侵入）

最低摩擦方案，利用 `AlphaDataset.add_feature(name, result=df)` 已有机制：

```python
# 无需修改 vnpy 核心代码

# 1. 从新闻数据库提取情感时序
sentiment_df = load_sentiment_timeseries(
    start="2024-01-01", end="2025-12-31",
    vt_symbols=["600000.SSE", "000001.SZSE", ...],
)
# sentiment_df schema: [datetime, vt_symbol, data]  (data = 日均情感分数)

# 2. 注入 AlphaDataset
dataset = AlphaDataset(bar_df, train_period, valid_period, test_period)
dataset.add_feature("sentiment_avg", result=sentiment_df)
dataset.add_feature("news_count", result=news_count_df)

# 3. 可以在表达式中引用已注入的列
# （在 prepare_data 后，sentiment_avg 成为 DataFrame 的列，可被后续 expression 引用）
```

#### 路径 B：表达式函数扩展

新增 `sent_function.py`，注册到 `calculate_by_expression()` 的 eval scope：

```python
# vnpy_newsengine/alpha/sent_function.py

from vnpy.alpha.dataset.utility import DataProxy

def sent_score(feature: DataProxy) -> DataProxy:
    """横截面情感分数排名"""
    ...

def sent_momentum(feature: DataProxy, window: int = 10) -> DataProxy:
    """情感动量 = ts_delta(sentiment, window)"""
    ...

def sent_surprise(feature: DataProxy, window: int = 20) -> DataProxy:
    """情感惊奇 = (current - ts_mean(window)) / ts_std(window)"""
    ...
```

需要修改 `vnpy/alpha/dataset/utility.py` 中的 `calculate_by_expression()` 函数，增加一行 import。

#### 路径 C：Polars Expr 原生路径

```python
dataset.add_feature(
    "sent_ma10",
    pl.col("sentiment_avg").rolling_mean(10).over("vt_symbol")
)
```

#### 路径 D：FactorVM DSL 移植（高级，可选）

将 FinnewsHunter 的 PyTorch FactorVM 因子表达式翻译为 vnpy DataProxy 表达式。`vocab.py` 中已有 `SENTIMENT` 和 `NEWS_COUNT` 特征，可映射为 DataProxy 列。

### 6.3 推荐策略

**Phase 1 用路径 A**（预计算注入），因为完全不需要改 vnpy 核心代码。
**Phase 4 补充路径 B**（函数扩展），使情感因子可以在表达式 DSL 中与价量因子自由组合。

---

## 七、新闻驱动策略详细设计

### 7.1 SentimentManager — 情感数据管理器

类比 vnpy 的 `ArrayManager`（100 根 K 线滚动窗口 + 30+ TA 指标），提供情感数据的滚动窗口聚合：

```python
# vnpy_newsengine/sentiment/manager.py

from collections import deque
from dataclasses import dataclass, field
import numpy as np

class SentimentManager:
    """
    情感数据滚动窗口管理器

    设计对标 vnpy ArrayManager：
    - ArrayManager: 管理 OHLCV numpy 数组，提供 SMA/EMA/RSI/MACD 等指标
    - SentimentManager: 管理情感分数序列，提供均值/趋势/密度/极端事件等指标
    """

    def __init__(self, size: int = 100) -> None:
        self.size = size
        self.count = 0
        self.inited = False

        # 核心数据（numpy 数组，对齐 ArrayManager 风格）
        self.score_array = np.zeros(size)        # 情感分数
        self.confidence_array = np.zeros(size)   # 置信度
        self.timestamp_array = np.zeros(size)    # Unix 时间戳
        self.news_count_array = np.zeros(size)   # 新闻计数（每个 slot 可能有多条）

    def update(self, sentiment: SentimentData) -> None:
        """更新情感数据"""
        self.count += 1
        if not self.inited and self.count >= self.size:
            self.inited = True

        # 环形缓冲
        self.score_array[:-1] = self.score_array[1:]
        self.score_array[-1] = sentiment.sentiment_score

        self.confidence_array[:-1] = self.confidence_array[1:]
        self.confidence_array[-1] = sentiment.confidence

    # --- 指标方法（返回标量）---

    def avg_score(self, window: int = 0) -> float:
        """加权平均情感分数（置信度加权）"""
        w = window or self.size
        scores = self.score_array[-w:]
        confs = self.confidence_array[-w:]
        total_conf = confs.sum()
        if total_conf == 0:
            return 0.0
        return float(np.dot(scores, confs) / total_conf)

    def score_std(self, window: int = 20) -> float:
        """情感波动率"""
        return float(np.std(self.score_array[-window:]))

    def score_trend(self, window: int = 20) -> float:
        """情感趋势（线性回归斜率）"""
        y = self.score_array[-window:]
        x = np.arange(window)
        if np.std(x) == 0:
            return 0.0
        slope = np.polyfit(x, y, 1)[0]
        return float(slope)

    def momentum(self, fast: int = 5, slow: int = 20) -> float:
        """情感动量 = 短期均值 - 长期均值"""
        return self.avg_score(fast) - self.avg_score(slow)

    def has_extreme_event(self, threshold: float = 0.8) -> bool:
        """最近一条情感是否为极端值"""
        return abs(self.score_array[-1]) >= threshold

    def surprise(self, window: int = 20) -> float:
        """情感惊奇度 = (最新值 - 均值) / 标准差"""
        std = self.score_std(window)
        if std < 1e-8:
            return 0.0
        return (self.score_array[-1] - np.mean(self.score_array[-window:])) / std
```

### 7.2 策略模板

```python
# vnpy_newsstrategy/template.py

class NewsStrategyTemplate:
    """
    新闻驱动策略基类

    提供三种回调入口：
    1. on_bar()           — 常规 K 线驱动（价量信号）
    2. on_sentiment()     — 情感数据驱动（新闻信号）
    3. on_debate_signal() — 辩论系统信号驱动

    子类覆写感兴趣的回调即可。
    """

    # 策略参数（子类覆写）
    author: str = ""
    parameters: list[str] = []
    variables: list[str] = []

    def __init__(
        self,
        strategy_engine,
        strategy_name: str,
        vt_symbol: str,
        setting: dict,
    ) -> None:
        self.strategy_engine = strategy_engine
        self.strategy_name = strategy_name
        self.vt_symbol = vt_symbol

        self.inited: bool = False
        self.trading: bool = False
        self.pos: float = 0

        # 内置管理器
        self.sentiment_mgr = SentimentManager(size=200)
        self.bar_generator = BarGenerator(self.on_bar)
        self.array_manager = ArrayManager(size=100)

    # --- 生命周期回调 ---
    def on_init(self) -> None: ...
    def on_start(self) -> None: ...
    def on_stop(self) -> None: ...

    # --- 数据回调 ---
    def on_tick(self, tick: TickData) -> None:
        self.bar_generator.update_tick(tick)

    def on_bar(self, bar: BarData) -> None:
        """价量信号入口 — 子类覆写"""
        pass

    def on_sentiment(self, sentiment: SentimentData) -> None:
        """情感信号入口 — 子类覆写"""
        self.sentiment_mgr.update(sentiment)

    def on_debate_signal(self, signal: DebateSignalData) -> None:
        """辩论信号入口 — 子类覆写"""
        pass

    def on_order(self, order: OrderData) -> None: ...
    def on_trade(self, trade: TradeData) -> None: ...

    # --- 交易接口（委托给 strategy_engine）---
    def buy(self, price: float, volume: float, ...) -> list[str]: ...
    def sell(self, price: float, volume: float, ...) -> list[str]: ...
    def short(self, price: float, volume: float, ...) -> list[str]: ...
    def cover(self, price: float, volume: float, ...) -> list[str]: ...
    def cancel_all(self) -> None: ...
```

### 7.3 示例策略：情感动量

```python
# vnpy_newsstrategy/strategies/sentiment_momentum.py

class SentimentMomentumStrategy(NewsStrategyTemplate):
    """
    情感动量策略

    核心逻辑：
    - 价格趋势（双均线）确认方向
    - 情感动量确认市场共识
    - 情感惊奇度触发加速入场
    - 极端事件触发紧急止损

    仅做多（A 股限制做空）。
    """

    # 参数
    fast_window: int = 10
    slow_window: int = 30
    sentiment_threshold: float = 0.2
    surprise_threshold: float = 2.0
    stop_loss_pct: float = 0.05
    fixed_size: float = 1.0

    # 变量
    fast_ma: float = 0.0
    slow_ma: float = 0.0
    sent_momentum: float = 0.0
    sent_surprise: float = 0.0

    parameters = ["fast_window", "slow_window", "sentiment_threshold",
                   "surprise_threshold", "stop_loss_pct", "fixed_size"]
    variables = ["fast_ma", "slow_ma", "sent_momentum", "sent_surprise"]

    def on_bar(self, bar: BarData) -> None:
        self.array_manager.update_bar(bar)
        am = self.array_manager
        sm = self.sentiment_mgr

        if not am.inited or not sm.inited:
            return

        self.fast_ma = am.sma(self.fast_window)
        self.slow_ma = am.sma(self.slow_window)
        self.sent_momentum = sm.momentum(5, 20)
        self.sent_surprise = sm.surprise(20)

        price_bullish = self.fast_ma > self.slow_ma
        sentiment_bullish = self.sent_momentum > self.sentiment_threshold

        if self.pos == 0:
            # 入场：价格趋势 + 情感共振
            if price_bullish and sentiment_bullish:
                self.buy(bar.close_price, self.fixed_size)

            # 加速入场：情感惊奇（利好突发）
            elif self.sent_surprise > self.surprise_threshold:
                self.buy(bar.close_price, self.fixed_size)

        elif self.pos > 0:
            # 止损：价格反转 + 情感恶化
            if not price_bullish and not sentiment_bullish:
                self.sell(bar.close_price, abs(self.pos))

    def on_sentiment(self, sentiment: SentimentData) -> None:
        """极端负面事件紧急响应"""
        super().on_sentiment(sentiment)

        if self.pos > 0 and sentiment.impact_level == "high" and sentiment.sentiment_score < -0.7:
            # 紧急平仓 — 不等下一根 K 线
            self.sell(0, abs(self.pos))  # price=0 表示市价单
```

---

## 八、错误处理与优雅降级

### 8.1 降级策略

```
┌─────────────────────────────────────────────────────┐
│                    降级矩阵                           │
├──────────────────┬──────────────────────────────────┤
│ 故障场景          │ 降级行为                          │
├──────────────────┼──────────────────────────────────┤
│ LLM 调用超时      │ 跳过情感分析，仅推送原始 NewsData │
│                  │ 策略退化为纯价量模式               │
│                  │ 记录 warning 日志                 │
├──────────────────┼──────────────────────────────────┤
│ LLM API Key 无效  │ 引擎启动时检测，降级为"仅爬虫"模式│
│                  │ 策略 sentiment_mgr.inited = False │
├──────────────────┼──────────────────────────────────┤
│ 所有爬虫源失败    │ 停止爬虫调度，进入冷却期(5min)    │
│                  │ 策略继续运行（使用缓存情感数据）    │
├──────────────────┼──────────────────────────────────┤
│ 单个爬虫源被封禁  │ 该源进入指数退避(30s→60s→120s→...)│
│                  │ 其他源继续正常采集（多源冗余）     │
├──────────────────┼──────────────────────────────────┤
│ 新闻数据库满/损坏 │ 切换到纯内存模式(deque缓存最近N条)│
│                  │ 告警通知                          │
├──────────────────┼──────────────────────────────────┤
│ 辩论引擎崩溃      │ 不影响新闻/情感事件正常推送       │
│                  │ 辩论策略收不到信号，自动不交易     │
├──────────────────┼──────────────────────────────────┤
│ EventEngine 积压  │ 新闻事件启用批量聚合(100ms窗口)   │
│                  │ 超过阈值丢弃低优先级新闻          │
└──────────────────┴──────────────────────────────────┘
```

### 8.2 关键不变式

```
1. 新闻引擎故障 绝不能 导致交易网关断连
2. LLM 调用 绝不能 阻塞 EventEngine 线程
3. 情感信号 绝不能 单独触发大额交易（必须有价量信号确认）
4. 爬虫异常 绝不能 抛出到 EventEngine 线程（try/except 在线程池内捕获）
5. 所有线程池必须有 timeout 和 max_queue_size 限制
```

---

## 九、测试策略

### 9.1 单元测试

| 测试模块 | 测试内容 | 框架 |
|----------|----------|------|
| `test_object.py` | NewsData/SentimentData 数据类创建、vt_symbol 转换 | pytest |
| `test_sentiment_manager.py` | 滚动窗口更新、均值/趋势/动量/惊奇度计算 | pytest |
| `test_llm_client.py` | LLMClient mock 测试、超时处理、重试 | pytest + unittest.mock |
| `test_crawlers.py` | 各源爬虫 HTML 解析（离线 fixture）、去重逻辑 | pytest |
| `test_signal_converter.py` | 辩论评级 → DebateSignalData 转换 | pytest |

### 9.2 集成测试

| 测试场景 | 验证内容 |
|----------|----------|
| 爬虫 → EventEngine | 爬虫线程产出 NewsData，在 EventEngine 上收到 EVENT_NEWS |
| 爬虫 → LLM → EventEngine | 新闻经过情感分析后，收到 EVENT_SENTIMENT |
| 新闻 → 策略 → 订单 | 策略收到 SentimentData 后触发 buy/sell |
| 完整辩论流程 | DataCollector → Bull/Bear → Manager → DebateSignalData |
| Alpha 因子注入 | 情感因子在 AlphaDataset.prepare_data() 后正确合并 |

### 9.3 回测验证

| 验证项 | 方法 |
|--------|------|
| 因子有效性 | IC / Rank IC 分析（目标 > 0.03） |
| 策略 Sharpe | 对比纯价量策略 vs 价量+情感混合策略 |
| 过拟合检查 | Train/Valid/Test 三段 Sharpe 一致性 |
| 时间敏感性 | 新闻延迟 0s/30s/60s/120s 对策略收益的影响 |
| 情感衰减 | 情感信号有效窗口分析（1h/4h/1d/3d） |

### 9.4 压力测试

| 场景 | 目标 |
|------|------|
| 新闻突增（模拟重大事件） | 10x 正常新闻量，EventEngine 延迟 < 100ms |
| LLM 全部超时 | 系统降级为纯价量模式，无崩溃 |
| 连续运行 72h | 内存无泄漏（news_cache 有 maxsize 限制） |

---

## 十、模块拆分与目录结构

### 10.1 新增模块清单

```
vnpy_newsengine/                         # Phase 1+3: 新闻引擎（独立 pip 包）
├── pyproject.toml
├── vnpy_newsengine/
│   ├── __init__.py                      # NewsEngineApp(BaseApp) 定义
│   ├── engine.py                        # NewsEngine(BaseEngine) — 核心引擎
│   ├── object.py                        # NewsData, SentimentData, DebateSignalData
│   ├── event.py                         # EVENT_NEWS, EVENT_SENTIMENT, EVENT_DEBATE_SIGNAL
│   │
│   ├── llm/                             # LLM 抽象层（替代 AgenticX）
│   │   ├── __init__.py
│   │   ├── client.py                    # LLMClient (litellm 封装)
│   │   └── prompts.py                   # 情感分析 / 辩论 Prompt 模板
│   │
│   ├── crawlers/                        # 爬虫（从 FinnewsHunter 移植）
│   │   ├── __init__.py
│   │   ├── base.py                      # BaseCrawler (去除 AgenticX 依赖)
│   │   ├── sina.py
│   │   ├── eastmoney.py
│   │   ├── tencent.py
│   │   ├── yicai.py
│   │   ├── nbd.py
│   │   └── netease.py
│   │
│   ├── sentiment/                       # 情感分析
│   │   ├── __init__.py
│   │   ├── manager.py                   # SentimentManager (滚动窗口)
│   │   └── analyzer.py                  # NewsAnalyzer (移植自 NewsAnalystAgent)
│   │
│   ├── debate/                          # 辩论引擎（Phase 3）
│   │   ├── __init__.py
│   │   ├── orchestrator.py              # DebateOrchestrator (移植+重构)
│   │   ├── agents.py                    # BullAgent, BearAgent, ManagerAgent
│   │   ├── signal_converter.py          # 评级 → DebateSignalData
│   │   └── config.py                    # 辩论模式配置
│   │
│   ├── storage/                         # 存储层
│   │   ├── __init__.py
│   │   ├── database.py                  # NewsDatabase ABC
│   │   ├── sqlite_db.py                 # SQLite 实现
│   │   └── pg_db.py                     # PostgreSQL 实现（可选）
│   │
│   ├── alpha/                           # Alpha 因子桥接（Phase 4）
│   │   ├── __init__.py
│   │   ├── sent_function.py             # DataProxy 情感因子函数
│   │   └── data_loader.py              # 情感数据 → DataProxy 转换
│   │
│   └── ui/                              # Qt6 UI（Phase 5）
│       ├── __init__.py
│       ├── news_widget.py               # 新闻面板
│       └── sentiment_widget.py          # 情感仪表盘
│
└── tests/
    ├── test_object.py
    ├── test_engine.py
    ├── test_crawlers.py
    ├── test_sentiment_manager.py
    ├── test_analyzer.py
    ├── test_debate.py
    └── fixtures/                        # 离线 HTML / JSON 测试数据
        ├── sina_sample.html
        ├── eastmoney_sample.html
        └── llm_response_mock.json


vnpy_newsstrategy/                       # Phase 2: 新闻策略应用（独立 pip 包）
├── pyproject.toml
├── vnpy_newsstrategy/
│   ├── __init__.py                      # NewsStrategyApp(BaseApp) 定义
│   ├── engine.py                        # NewsStrategyEngine(BaseEngine)
│   ├── template.py                      # NewsStrategyTemplate 基类
│   ├── backtesting.py                   # NewsBacktestingEngine
│   └── strategies/
│       ├── __init__.py
│       ├── sentiment_momentum.py        # 情感动量策略
│       ├── news_breakout.py             # 突发新闻策略
│       └── debate_follow.py             # 辩论跟随策略
│
└── tests/
    ├── test_template.py
    ├── test_backtesting.py
    └── test_strategies.py


examples/                                # 示例代码
├── news_trading/
│   ├── run.py                           # 实盘启动脚本
│   └── setting.json                     # 配置示例
├── news_backtesting/
│   ├── backtest_sentiment_momentum.py
│   └── backtest_debate_follow.py
└── alpha_sentiment/
    ├── 01_data_preparation.ipynb        # 情感数据准备
    ├── 02_factor_analysis.ipynb         # IC / Rank IC 分析
    └── 03_model_training.ipynb          # LightGBM + 情感因子
```

### 10.2 对 vnpy 核心的修改（零修改方案）

经过深入分析 vnpy 的插件机制，**可以做到完全不修改 vnpy 核心代码**：

| 原方案修改点 | 零修改替代方案 |
|-------------|---------------|
| 在 `vnpy/trader/event.py` 添加事件常量 | 在 `vnpy_newsengine/event.py` 中定义，import 后直接使用（事件类型只是字符串） |
| 在 `vnpy/trader/object.py` 添加数据类 | 在 `vnpy_newsengine/object.py` 中定义（`BaseData` 可被外部继承） |
| 在 `vnpy/alpha/dataset/utility.py` 注入 import | 使用路径 A（预计算结果注入），无需修改表达式引擎 |

> **这意味着整个融合项目是 vnpy 的纯外部插件，可以 `pip install` 即用，不影响 vnpy 升级。**

---

## 十一、技术决策与权衡

### 11.1 架构模式选择

| 决策点 | 选项 A | 选项 B | **决定** | **理由** |
|--------|--------|--------|----------|----------|
| 集成方式 | 微服务（gRPC 通信） | 单体插件（BaseApp） | **B** | 降低运维复杂度，vnpy 插件机制成熟 |
| 新闻组件挂载 | BaseGateway 子类 | BaseEngine + BaseApp | **B** | Gateway 强制实现 send_order 等 6 个交易方法，不适用 |
| 新闻采集 | 保留 Celery | ThreadPoolExecutor | **B** | 去掉 Celery+Redis 依赖，爬虫负载轻量，线程池足够 |
| LLM 调用 | 同步阻塞 | 线程池 + litellm | **B** | EventEngine 单线程不可阻塞 |
| LLM 框架 | 保留 AgenticX | litellm 直接调用 | **B** | AgenticX 闭源且被 hack 使用，litellm 支持 100+ 模型 |
| 新闻存储 | PostgreSQL + Milvus | SQLite（可选 PG） | **B** | Phase 1 零外部依赖；向量搜索场景少可后加 |
| 前端 | React Web UI | Qt6 桌面 | **B** | vnpy 用户群体习惯 Qt6，可选 RPC 暴露 Web 接口 |
| Alpha 集成 | 修改 eval scope | 预计算结果注入 | **B** | 零侵入，不修改 vnpy 核心 |
| 辩论 asyncio | 嵌入 EventEngine 线程 | 独立 asyncio 事件循环线程 | **B** | 辩论 30s+ 绝不能阻塞事件总线 |

### 11.2 复用与重写策略

| FinnewsHunter 组件 | 代码量 | 复用策略 | AgenticX 依赖 | 改动量 |
|---------------------|--------|----------|---------------|--------|
| `BaseCrawler` + 10 源爬虫 | ~2000 行 | **直接移植** | `BaseTool` 基类 | 删 ~20 行/源 |
| `NewsAnalystAgent.analyze_news()` | ~300 行 | **直接移植** | `Agent` 基类（实际未用 `AgentExecutor`） | 删 ~30 行 |
| `_repair_markdown_table()` | ~50 行 | **原样复用** | 无 | 0 行 |
| `_extract_structured_info()` | ~80 行 | **原样复用** | 无 | 0 行 |
| `BullResearcherAgent` | ~200 行 | **重构移植** | `Agent(BaseModel)` + `__setattr__` hack | 删 ~15 行 |
| `BearResearcherAgent` | ~200 行 | **重构移植** | 同上 | 删 ~15 行 |
| `InvestmentManagerAgent` | ~250 行 | **重构移植** | 同上 | 删 ~15 行 |
| `DebateOrchestrator` | ~600 行 | **重构移植** | 无直接依赖，但用 Agent 实例 | 替换 ~10 行 |
| `LLMService` | ~200 行 | **替换重写** | `LiteLLMProvider` / `BailianProvider` | 用 `litellm` 重写 ~50 行 |
| `FactorVM / DSL ops` | ~300 行 | **可选移植** | PyTorch（无 AgenticX） | 0 行 |
| `ProviderRegistry` (金融数据) | ~400 行 | **不复用** | — | vnpy 有 DataFeed 体系 |
| React 前端 | ~5000 行 | **不复用** | — | Qt6 替代 |

### 11.3 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| LLM 调用延迟(1-10s)阻塞交易 | 高 | 高 | 独立线程池 + timeout 熔断 + 降级为纯价量模式 |
| LLM 情感分析准确率不足 | 中 | 高 | 情感信号仅做过滤/增强（不单独触发）+ 置信度阈值过滤 |
| 新闻爬虫被反爬封禁 | 高 | 中 | 10+ 源冗余 + 指数退避 + User-Agent 轮换 + 请求限速 |
| EventEngine 单线程积压 | 低 | 高 | 新闻事件量远小于 tick（<1%）+ 批量聚合 + 背压机制 |
| 情感因子过拟合 | 中 | 中 | Train/Valid/Test 三段检验 + 滚动窗口外样本测试 |
| litellm 版本兼容性 | 低 | 低 | 锁定版本 + 封装抽象层，仅暴露 `invoke()` |
| 新闻历史数据缺失 | 中 | 中 | 回测使用 FinnewsHunter PG 已有数据 + 冷启动爬取工具 |

---

## 十二、实施路线图

### Phase 1：新闻引擎核心（3 周）

**目标：** 新闻数据能流入 vnpy 事件引擎，情感分析可运行。

```
Week 1: 基础框架
  ├─ 项目脚手架 (pyproject.toml, __init__.py, BaseApp 注册)
  ├─ NewsData / SentimentData / DebateSignalData 数据类
  ├─ EVENT_NEWS / EVENT_SENTIMENT 事件定义
  ├─ LLMClient (litellm 封装 + 超时 + 重试)
  └─ 单元测试

Week 2: 爬虫移植
  ├─ BaseCrawler 剥离 AgenticX (删除 BaseTool 继承)
  ├─ 移植 3 个核心爬虫: sina, eastmoney, tencent
  ├─ NewsEngine 框架 (CrawlThread Pool + dedup + cache)
  ├─ SQLite 新闻存储
  └─ 集成测试: 爬虫 → EventEngine

Week 3: 情感分析
  ├─ NewsAnalyzer 移植 (剥离 AgenticX Agent 基类)
  ├─ LLM Thread Pool 集成
  ├─ 情感分析 Prompt 调优
  ├─ 集成测试: 爬虫 → LLM → EVENT_SENTIMENT
  └─ 降级测试: LLM 不可用时系统行为
```

**验收标准：**
- `main_engine.add_app(NewsEngineApp)` 成功挂载
- 自动爬取新闻并在 EventEngine 中收到 `EVENT_NEWS` 和 `EVENT_SENTIMENT`
- LLM 超时时系统不崩溃，降级为仅推送原始新闻

### Phase 2：策略框架（3 周）

**目标：** 策略可以消费新闻/情感信号，回测引擎可运行。

```
Week 4: 策略基础
  ├─ SentimentManager 实现 + 单元测试
  ├─ NewsStrategyTemplate 基类
  ├─ NewsStrategyEngine (事件订阅 + 策略分发)
  └─ 策略参数化框架

Week 5: 回测引擎
  ├─ NewsBacktestingEngine (时间对齐: bar + sentiment)
  ├─ 历史情感数据加载器
  ├─ 回测统计扩展 (情感 Attribution)
  └─ 集成测试

Week 6: 示例策略
  ├─ SentimentMomentum 策略实现
  ├─ NewsBreakout 策略实现
  ├─ 回测验证 (vs 纯价量基准)
  └─ 参数优化支持 (复用 vnpy optimize.py)
```

**验收标准：**
- 历史新闻+行情数据回测通过
- SentimentMomentum Sharpe > 纯双均线策略
- 回测报告包含情感因子贡献分析

### Phase 3：辩论引擎（2 周）

**目标：** 多智能体辩论产出可执行交易信号。

```
Week 7: 辩论移植
  ├─ DebateOrchestrator 重构 (SSE → Event 推送)
  ├─ BullAgent / BearAgent / ManagerAgent 剥离 AgenticX
  ├─ SignalConverter (评级 → DebateSignalData)
  └─ 独立 asyncio 线程集成

Week 8: 策略集成
  ├─ DebateFollowStrategy 实现
  ├─ 辩论触发机制 (手动 + 高影响新闻自动)
  ├─ 集成测试: 股票代码 → 辩论 → 信号 → 策略
  └─ 辩论结果缓存 (避免重复辩论)
```

**验收标准：** 输入股票代码 → 辩论完成 < 30s → DebateSignalData 推送到策略。

### Phase 4：Alpha 因子统一（2 周）

**目标：** 情感因子可在 vnpy Alpha 研究体系中使用。

```
Week 9: 数据桥接
  ├─ sentiment_to_dataproxy() 转换工具
  ├─ 新闻历史数据导入 (FinnewsHunter PG → 本地)
  ├─ AlphaDataset.add_feature(result=sentiment_df) 验证
  └─ sent_function.py (DataProxy 情感因子函数)

Week 10: 研究 Notebook
  ├─ 01_data_preparation.ipynb
  ├─ 02_factor_analysis.ipynb (IC / Rank IC 分析)
  ├─ 03_model_training.ipynb (LightGBM + 情感因子)
  └─ A/B 对比: Alpha158 vs Alpha158+Sentiment
```

**验收标准：** IC 分析完成，混合模型 vs 纯价量模型有量化对比结论。

### Phase 5：生产化与 UI（2 周）

```
Week 11: Qt6 UI + 监控
  ├─ 新闻面板 (实时新闻流 + 情感分数着色)
  ├─ 情感仪表盘 (多标的趋势图)
  ├─ 引擎状态监控 (爬虫成功率、LLM 延迟)
  └─ 告警集成 (极端事件 → EmailEngine)

Week 12: 稳定性 + 文档
  ├─ 72h 连续运行压力测试
  ├─ 内存泄漏检查 (cache eviction 验证)
  ├─ 策略开发指南
  └─ 部署手册 (Docker Compose 可选)
```

---

## 十三、依赖管理

### vnpy_newsengine 依赖

```toml
[project]
name = "vnpy_newsengine"
requires-python = ">=3.10"
dependencies = [
    "vnpy>=4.3.0",

    # LLM
    "litellm>=1.40.0",

    # 爬虫
    "requests>=2.31.0",
    "beautifulsoup4>=4.12.0",
    "lxml>=5.0.0",

    # NLP
    "jieba>=0.42.1",

    # 存储
    # (SQLite 内置于 Python stdlib)
]

[project.optional-dependencies]
postgresql = ["asyncpg>=0.29.0", "psycopg2-binary>=2.9.0"]
neo4j = ["neo4j>=5.0.0"]
vector = ["pymilvus>=2.3.0"]
debate = []  # 辩论引擎无额外依赖（使用同一个 litellm）
alpha = ["polars>=0.20.0", "torch>=2.0.0"]  # Alpha 因子桥接
all = ["vnpy_newsengine[postgresql,neo4j,vector,alpha]"]
```

### 与原系统依赖对比

| 原 FinnewsHunter 依赖 | 融合方案是否需要 | 说明 |
|------------------------|-----------------|------|
| FastAPI + Uvicorn | **不需要** | 无 Web 服务器 |
| Celery + Redis | **不需要** | 用 ThreadPoolExecutor 替代 |
| PostgreSQL | **可选** | Phase 1 用 SQLite，后续可升级 |
| Milvus + etcd + MinIO | **可选** | 向量搜索需求不大可省略 |
| Neo4j | **可选** | 知识图谱功能保留为 extra |
| AgenticX | **不需要** | 完全剥离，用 litellm 替代 |
| React + Vite + Tailwind | **不需要** | Qt6 替代 |
| SQLAlchemy + Alembic | **不需要** | 轻量 SQLite 直接用 sqlite3 |

**结果：必须依赖从 ~30 个缩减到 5 个**（vnpy + litellm + requests + beautifulsoup4 + lxml）。

---

## 十四、监控与可观测性

### 14.1 关键指标

```python
# 通过 EVENT_NEWS_LOG 推送到 vnpy LogEngine

@dataclass
class NewsEngineMetrics:
    # 爬虫指标
    crawl_total: int = 0              # 总爬取次数
    crawl_success: int = 0            # 成功次数
    crawl_fail: int = 0               # 失败次数
    crawl_dedup: int = 0              # 去重拦截次数
    crawl_latency_ms: float = 0.0     # 平均爬取延迟

    # LLM 指标
    llm_total: int = 0
    llm_success: int = 0
    llm_fail: int = 0
    llm_timeout: int = 0
    llm_latency_ms: float = 0.0       # 平均 LLM 延迟
    llm_tokens_used: int = 0          # Token 消耗

    # 辩论指标
    debate_total: int = 0
    debate_avg_time_s: float = 0.0
    debate_avg_rounds: float = 0.0

    # 事件指标
    events_pushed: int = 0             # 推送到 EventEngine 的事件总数
    queue_depth: int = 0               # 当前队列深度
```

### 14.2 日志级别

| 级别 | 事件 |
|------|------|
| DEBUG | 每条新闻的爬取详情、LLM 请求/响应 |
| INFO | 新闻入库、情感分析完成、辩论完成 |
| WARNING | 爬虫源临时失败(将重试)、LLM 超时(降级)、队列积压 |
| ERROR | 爬虫源持续失败、LLM API 认证失败、数据库错误 |
| CRITICAL | 所有爬虫源失败、EventEngine 积压超阈值 |

---

## 十五、成功指标

| 指标 | 目标 | 测量方法 |
|------|------|----------|
| **新闻采集延迟** | 发布后 < 60s 入库 | 对比新闻 publish_time 与入库时间 |
| **情感分析延迟** | 入库后 < 10s | LLM 调用耗时统计 |
| **辩论系统延迟** | 单次辩论 < 30s | orchestrator.execution_time |
| **情感因子 IC** | > 0.03 (日频) | AlphaLens IC 分析 |
| **混合策略 Sharpe** | > 纯价量 10%+ | BacktestingEngine 统计对比 |
| **回测数据覆盖** | > 1 年历史 | 数据库记录时间跨度 |
| **系统稳定性** | 72h 无崩溃 | 连续运行压力测试 |
| **事件引擎影响** | 新闻事件 < 1% tick | 事件计数比对 |
| **必须依赖数** | ≤ 5 个核心包 | pyproject.toml |
| **vnpy 核心修改** | 0 行 | git diff vnpy/ |
| **AgenticX 依赖** | 完全移除 | import 检查 |
| **测试覆盖率** | > 80% | pytest --cov |

---

## 附录 A：关键接口映射

### FinnewsHunter → vnpy 接口对应

| FinnewsHunter 接口 | vnpy 对应 | 映射方式 |
|---------------------|-----------|----------|
| `BaseTool.execute()` | `NewsEngine._crawl_source()` | 线程池调用 |
| `Agent.invoke([messages])` | `LLMClient.invoke(messages)` | litellm 替代 |
| `SSE callback on_event` | `EventEngine.put(Event)` | 事件推送 |
| `FastAPI POST /crawl` | `NewsEngine.trigger_crawl()` | 方法调用 |
| `FastAPI GET /news` | `NewsEngine.get_latest_news()` | 方法调用 |
| `Celery task` | `ThreadPoolExecutor.submit()` | 线程池替代 |
| `Redis dedup` | `seen_urls: set` | 内存去重 |
| `PostgreSQL News model` | `SQLite news table` | 简化存储 |
| `Milvus vector search` | `(可选, Phase 5+)` | 降级为关键词搜索 |
| `Neo4j knowledge graph` | `(可选, Phase 5+)` | 降级为无知识图谱 |

### vnpy 事件生命周期对应

| vnpy 原有模式 | 新闻引擎对应 |
|---------------|-------------|
| `Gateway.on_tick() → EVENT_TICK` | `NewsEngine._push_news_event() → EVENT_NEWS` |
| `OmsEngine 缓存 tick` | `NewsEngine 缓存 news/sentiment` |
| `Strategy.on_tick() → BarGenerator` | `Strategy.on_sentiment() → SentimentManager` |
| `ArrayManager.sma(10)` | `SentimentManager.avg_score(10)` |
| `BarData [symbol, exchange, OHLCV]` | `SentimentData [vt_symbol, score, confidence]` |
| `BacktestingEngine 遍历 BarData` | `NewsBacktestingEngine 同时遍历 BarData + SentimentData` |
| `AlphaDataset.add_feature("alpha1", expr)` | `AlphaDataset.add_feature("sentiment", result=df)` |

---

## 附录 B：关键数据 Schema 映射

### B.1 新闻数据 (FinnewsHunter NewsItem → vnpy NewsData)

| FinnewsHunter `NewsItem` 字段 | vnpy `NewsData` 字段 | 转换规则 |
|-------------------------------|----------------------|----------|
| `title: str` | `title: str` | 直接映射 |
| `content: str` | `content: str` | 直接映射 |
| `url: str` | `url: str` | 直接映射 |
| `source: str` | `source: str` | 直接映射 |
| `publish_time: str` | `publish_time: datetime` | `datetime.fromisoformat()` |
| `stock_codes: List[str]` | `stock_codes: list[str]` | 直接映射 |
| — | `vt_symbols: list[str]` | `__post_init__` 自动转换 |
| — | `gateway_name: str` | 固定 `"NEWS"` |
| — | `news_id: str` | `hashlib.md5(url)` |

### B.2 情感数据 (FinnewsHunter structured_data → vnpy SentimentData)

| FinnewsHunter `_extract_structured_info()` 输出 | vnpy `SentimentData` 字段 | 转换规则 |
|--------------------------------------------------|---------------------------|----------|
| `sentiment: str` (正面/负面/中性) | `sentiment_score: float` | 正面→+0.7, 负面→-0.7, 中性→0.0（再根据分值微调） |
| `sentiment_score: int` (1-10) | `sentiment_score: float` | `(score - 5.5) / 4.5` 归一化到 [-1, +1] |
| `confidence: str` (高/中/低) | `confidence: float` | 高→0.9, 中→0.6, 低→0.3 |
| — | `impact_level: str` | 基于 sentiment_score 绝对值: >0.7 high, >0.4 medium, low |
| — | `recommendation: str` | 从 analysis_result 正则提取投资建议 |

### B.3 Alpha 因子数据 (SentimentData → AlphaDataset DataProxy)

| 源字段 | 目标 Schema | 聚合规则 |
|--------|-------------|----------|
| `SentimentData.sentiment_score` | `[datetime, vt_symbol, data]` | 日频均值：同一天多条新闻取置信度加权平均 |
| `SentimentData` 计数 | `[datetime, vt_symbol, data]` | 日频计数：同一天新闻数量 |
| `DebateSignalData.rating_score` | `[datetime, vt_symbol, data]` | 日频最新值：取当天最后一次辩论评级 |
