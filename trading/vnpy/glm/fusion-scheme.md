# FinnewsHunter × VeighNa 融合方案

> 基于金融新闻多智能体系统与量化交易框架的深度融合设计

---

## 一、项目概述

### 1.1 项目背景

本项目旨在将两个独立的金融科技项目进行深度融合：

| 项目 | 定位 | 核心能力 |
|------|------|----------|
| **FinnewsHunter** | 金融新闻多智能体分析平台 | 新闻爬取、情感分析、多智能体辩论、知识图谱 |
| **VeighNa** | 量化交易开发框架 | 策略引擎、交易接口、回测系统、实盘交易 |

### 1.2 融合目标

构建一个**AI驱动的量化投资决策系统**，实现：

```
新闻舆情 → 智能分析 → 策略生成 → 回测验证 → 实盘交易
```

**核心价值主张**：
- 将非结构化新闻数据转化为可执行的交易信号
- 多智能体辩论机制提供决策置信度评估
- 从研究到实盘的完整闭环

---

## 二、项目深度分析

### 2.1 FinnewsHunter 项目分析

#### 2.1.1 项目定位

FinnewsHunter 是一个企业级金融新闻分析系统，基于 AgenticX 框架构建，集成实时新闻流、深度量化分析和多智能体辩论机制。

#### 2.1.2 技术栈

| 层级 | 技术选型 |
|------|----------|
| **后端框架** | FastAPI + Uvicorn |
| **数据库** | PostgreSQL + SQLAlchemy |
| **向量数据库** | Milvus（向量检索） |
| **图数据库** | Neo4j（知识图谱） |
| **缓存/队列** | Redis + Celery |
| **前端** | React + TypeScript + Shadcn UI + Vite |
| **AI框架** | AgenticX（多智能体编排） |
| **LLM** | 多供应商支持（百炼、OpenAI、DeepSeek、Kimi、智谱） |

#### 2.1.3 核心模块

```
backend/app/
├── agents/                    # 智能体模块
│   ├── news_analyst.py        # 新闻分析师
│   ├── debate_agents.py       # 多空辩论智能体
│   │   ├── BullResearcherAgent   # 多方研究员
│   │   ├── BearResearcherAgent   # 空方研究员
│   │   └── InvestmentManagerAgent # 投资经理
│   ├── data_collector_v2.py   # 数据采集智能体
│   ├── search_analyst.py      # 搜索分析师
│   ├── orchestrator.py        # 辩论编排器
│   └── quantitative_agent.py  # 量化分析智能体
├── tools/                     # 工具模块
│   └── crawlers/              # 多源新闻爬虫
├── services/                  # 业务服务
│   ├── llm_service.py         # LLM服务
│   ├── embedding_service.py   # 向量化服务
│   └── analysis_service.py    # 分析服务
├── storage/                   # 存储层
│   └── vector_storage.py      # Milvus向量存储
├── knowledge/                 # 知识图谱
└── alpha_mining/              # Alpha挖掘
```

#### 2.1.4 核心能力

1. **多源新闻爬取**：10+金融新闻源（新浪、腾讯、财经网等）
2. **智能情感分析**：基于LLM的情感评分与市场影响评估
3. **多空辩论机制**：多智能体对抗式分析，提升决策质量
4. **动态数据检索**：AkShare实时行情、博查AI新闻搜索
5. **知识图谱**：Neo4j存储股票关联关系
6. **向量检索**：Milvus语义相似度搜索

#### 2.1.5 数据流

```
新闻源 → 爬虫服务 → PostgreSQL → 情感分析 → Milvus向量化
                                        ↓
                              多空辩论 → 投资建议
```

---

### 2.2 VeighNa 项目分析

#### 2.2.1 项目定位

VeighNa 是一套基于 Python 的开源量化交易系统开发框架，提供从策略开发、回测到实盘交易的完整解决方案。

#### 2.2.2 技术栈

| 层级 | 技术选型 |
|------|----------|
| **核心框架** | Python 3.10+ |
| **GUI** | PySide6 + PyQtGraph |
| **数据分析** | Pandas + NumPy + Polars |
| **机器学习** | scikit-learn + LightGBM + PyTorch |
| **技术分析** | TA-Lib |
| **事件引擎** | 自研事件驱动引擎 |
| **RPC** | ZeroMQ |

#### 2.2.3 核心模块

```
vnpy/
├── trader/                    # 核心交易模块
│   ├── engine.py              # 主引擎
│   ├── gateway.py             # 交易接口基类
│   ├── event.py               # 事件定义
│   ├── object.py              # 数据对象
│   ├── database.py            # 数据库接口
│   ├── datafeed.py            # 数据服务接口
│   └── ui/                    # 图形界面
├── alpha/                     # AI量化模块
│   ├── lab.py                 # 研究实验室
│   ├── dataset/               # 因子数据集
│   │   └── alpha_158.py       # Alpha158因子
│   ├── model/                 # ML模型
│   │   ├── lasso_model.py     # Lasso回归
│   │   ├── lgb_model.py       # LightGBM
│   │   └── mlp_model.py       # MLP神经网络
│   └── strategy/              # 策略模块
├── event/                     # 事件引擎
├── chart/                     # K线图表
└── rpc/                       # RPC服务
```

#### 2.2.4 核心能力

1. **多市场交易接口**：
   - 国内：CTP期货、XTP证券、ETF期权等
   - 海外：IB、TAP等

2. **策略类型**：
   - CTA策略引擎
   - 组合策略引擎
   - 价差交易
   - 期权交易

3. **AI量化能力**（vnpy.alpha）：
   - Alpha158因子特征工程
   - LightGBM/MLP预测模型
   - 截面/时序策略支持

4. **数据库支持**：
   - SQLite/MySQL/PostgreSQL
   - MongoDB/DolphinDB/TDengine

#### 2.2.5 数据流

```
数据源 → Gateway → 主引擎 → 策略引擎 → 订单执行
                ↓
              事件引擎 → 各模块响应
```

---

## 三、融合架构设计

### 3.1 融合层次架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层 (Application Layer)                │
├─────────────────────────────────────────────────────────────────┤
│  Web前端 (React)    │   VeighNa GUI (PySide6)  │   API接口      │
└─────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────┐
│                        服务层 (Service Layer)                    │
├─────────────────────────────────────────────────────────────────┤
│  新闻分析服务    │   辩论引擎    │   策略管理    │   交易引擎   │
│  (FinnewsHunter) │   (融合)      │   (VeighNa)   │   (VeighNa)  │
└─────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────┐
│                        智能体层 (Agent Layer)                    │
├─────────────────────────────────────────────────────────────────┤
│  新闻分析师      │   多空辩论智能体   │   量化策略智能体         │
│  搜索分析师      │   投资经理智能体   │   风控智能体             │
└─────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────┐
│                        数据层 (Data Layer)                       │
├─────────────────────────────────────────────────────────────────┤
│  PostgreSQL      │   Milvus      │   Neo4j      │   时序数据库  │
│  (结构化数据)    │   (向量检索)  │   (知识图谱)  │   (行情数据)  │
└─────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────┐
│                        基础设施层 (Infrastructure)               │
├─────────────────────────────────────────────────────────────────┤
│  Redis缓存    │   Celery任务队列    │   Docker部署              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 融合模式

采用**微内核 + 插件**架构：

```
                    ┌──────────────────┐
                    │   融合核心引擎   │
                    │  (Fusion Core)   │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ 新闻分析插件  │   │  策略执行插件 │   │  数据服务插件 │
│(FinnewsHunter)│   │   (VeighNa)   │   │   (融合)      │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 3.3 核心融合模块

#### 3.3.1 新闻信号转换器

将新闻分析结果转换为VeighNa策略信号：

```python
class NewsSignalConverter:
    """新闻信号转换器：将分析结果转为交易信号"""

    def convert(self, analysis_result: AnalysisResult) -> SignalData:
        """
        将新闻分析结果转换为交易信号

        输入：
            - sentiment_score: 情感得分 (-1 to 1)
            - confidence: 置信度 (0 to 1)
            - affected_stocks: 关联股票列表
            - debate_result: 多空辩论结论

        输出：
            - direction: 方向 (LONG/SHORT/NEUTRAL)
            - strength: 信号强度 (0-100)
            - target_symbols: 目标标的
            - valid_until: 信号有效期
        """
        pass
```

#### 3.3.2 智能策略适配器

将AI生成的策略注入VeighNa策略引擎：

```python
class AIStrategyAdapter:
    """AI策略适配器：桥接新闻分析与VeighNa策略"""

    def __init__(self, main_engine: MainEngine, event_engine: EventEngine):
        self.main_engine = main_engine
        self.event_engine = event_engine
        self.news_signal_queue = Queue()

    def on_news_signal(self, signal: SignalData):
        """接收新闻信号并触发策略响应"""
        # 1. 验证信号有效性
        # 2. 查询当前持仓
        # 3. 调用VeighNa策略执行
        pass
```

#### 3.3.3 事件总线融合

统一事件驱动架构：

```python
class FusionEventBus:
    """融合事件总线"""

    # VeighNa事件类型
    EVENT_TICK = "EVENT_TICK"
    EVENT_BAR = "EVENT_BAR"
    EVENT_ORDER = "EVENT_ORDER"

    # FinnewsHunter事件类型
    EVENT_NEWS = "EVENT_NEWS"
    EVENT_ANALYSIS = "EVENT_ANALYSIS"
    EVENT_DEBATE = "EVENT_DEBATE"

    # 融合事件类型
    EVENT_SIGNAL = "EVENT_SIGNAL"      # 交易信号
    EVENT_RISK = "EVENT_RISK"          # 风控预警

    def emit(self, event_type: str, data: Any):
        """统一事件发布"""
        pass
```

---

## 四、核心功能设计

### 4.1 新闻驱动策略

#### 4.1.1 策略类型

| 策略类型 | 触发条件 | 执行逻辑 |
|----------|----------|----------|
| **快速响应策略** | 重大新闻发布 | 毫秒级信号生成，自动执行 |
| **辩论决策策略** | 多空辩论完成 | 等待辩论结论后执行 |
| **情感过滤策略** | 情感得分超阈值 | 作为其他策略的过滤器 |
| **组合轮动策略** | 行业新闻聚合 | 行业轮动信号生成 |

#### 4.1.2 信号生成流程

```
新闻入库 → 情感分析 → 相关性匹配 → 多空辩论 → 信号生成 → 风控检查 → 执行
   │           │            │            │           │           │
   5s         3s           2s          30s         1s         1s
                                                        ↓
                                                   总延迟: ~42s
```

### 4.2 多智能体辩论与策略验证

```
┌─────────────────────────────────────────────────────────────────┐
│                      辩论驱动决策流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│   │ 多方研究员  │ ←→  │  辩论编排器  │ ←→  │ 空方研究员  │      │
│   │(BullAgent)  │     │(Orchestrator)│     │(BearAgent)  │      │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘      │
│          │                   │                   │              │
│          └───────────────────┼───────────────────┘              │
│                              ↓                                  │
│                    ┌─────────────────┐                          │
│                    │   投资经理      │                          │
│                    │(InvestmentMgr)  │                          │
│                    └────────┬────────┘                          │
│                             │                                   │
│                             ↓                                   │
│                    ┌─────────────────┐                          │
│                    │   决策结果      │                          │
│                    │ • 方向建议      │                          │
│                    │ • 置信度        │                          │
│                    │ • 风险评估      │                          │
│                    └────────┬────────┘                          │
│                             │                                   │
│                             ↓                                   │
│                    ┌─────────────────┐                          │
│                    │ VeighNa回测验证 │                          │
│                    │ (历史情景回放)  │                          │
│                    └─────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 知识图谱增强

利用Neo4j构建股票关系网络：

```
                    ┌──────────────┐
                    │   贵州茅台   │
                    │  (SH600519)  │
                    └──────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │   白酒行业   │ │   消费板块   │ │   上证50    │
    │  (行业关联)  │ │  (板块关联)  │ │ (指数成分)  │
    └─────────────┘ └─────────────┘ └─────────────┘
           │
           ▼
    ┌─────────────┐
    │   五粮液    │
    │(同行业关联) │
    └─────────────┘
```

**应用场景**：
1. **新闻传导分析**：某股票新闻影响关联股票
2. **行业轮动信号**：行业新闻聚合生成轮动信号
3. **风险传染评估**：负面新闻风险传导路径分析

---

## 五、技术实现方案

### 5.1 项目结构

```
fusion-trading/
├── core/                           # 融合核心
│   ├── fusion_engine.py            # 融合引擎
│   ├── event_bus.py                # 事件总线
│   └── signal_converter.py         # 信号转换器
├── agents/                         # 智能体模块（来自FinnewsHunter）
│   ├── news_analyst.py
│   ├── debate_agents.py
│   ├── search_analyst.py
│   └── trading_agent.py            # 新增：交易智能体
├── strategies/                     # 策略模块（基于VeighNa）
│   ├── news_driven_strategy.py     # 新闻驱动策略
│   ├── sentiment_filter_strategy.py # 情感过滤策略
│   └── debate_signal_strategy.py   # 辩论信号策略
├── gateways/                       # 交易接口（来自VeighNa）
│   └── ...
├── data/                           # 数据模块
│   ├── news_storage.py             # 新闻存储
│   ├── vector_store.py             # 向量存储
│   └── knowledge_graph.py          # 知识图谱
├── api/                            # API服务
│   ├── rest_api.py                 # REST接口
│   └── websocket.py                # WebSocket推送
├── ui/                             # 前端界面
│   ├── web/                        # Web界面（React）
│   └── desktop/                    # 桌面端（PySide6）
└── config/                         # 配置文件
    ├── settings.py
    └── docker-compose.yml
```

### 5.2 核心接口设计

#### 5.2.1 融合引擎接口

```python
from abc import ABC, abstractmethod
from typing import Optional
from dataclasses import dataclass
from datetime import datetime

@dataclass
class TradingSignal:
    """交易信号数据结构"""
    symbol: str                    # 标的代码
    direction: str                 # 方向: LONG/SHORT/NEUTRAL
    strength: float                # 强度: 0-100
    confidence: float              # 置信度: 0-1
    source: str                    # 信号来源: NEWS/DEBATE/ML
    valid_until: datetime          # 有效期
    metadata: dict                 # 元数据

class FusionEngine:
    """融合引擎核心类"""

    def __init__(self):
        self.vnpy_engine: MainEngine = None
        self.news_engine: NewsEngine = None
        self.event_bus: FusionEventBus = None

    def start(self):
        """启动融合引擎"""
        # 1. 初始化VeighNa主引擎
        # 2. 启动新闻分析服务
        # 3. 注册事件监听
        # 4. 启动策略
        pass

    def on_news_event(self, news: NewsData):
        """新闻事件处理"""
        pass

    def on_signal_event(self, signal: TradingSignal):
        """信号事件处理"""
        pass

    def execute_trade(self, signal: TradingSignal):
        """执行交易"""
        pass
```

#### 5.2.2 策略基类

```python
from vnpy.trader.object import BarData, TickData
from vnpy.trader.constant import Direction, Offset

class NewsDrivenStrategy(CtaTemplate):
    """新闻驱动策略基类"""

    # 策略参数
    sentiment_threshold: float = 0.6    # 情感阈值
    confidence_threshold: float = 0.7   # 置信度阈值
    signal_valid_seconds: int = 300     # 信号有效期(秒)

    # 信号缓存
    active_signals: dict[str, TradingSignal] = {}

    def on_init(self):
        """策略初始化"""
        self.load_knowledge_graph()

    def on_news_signal(self, signal: TradingSignal):
        """接收新闻信号"""
        # 1. 验证信号有效性
        # 2. 检查是否与当前持仓冲突
        # 3. 生成交易委托
        pass

    def on_bar(self, bar: BarData):
        """K线数据更新"""
        # 检查是否有待执行的信号
        # 执行风控检查
        # 发送委托
        pass
```

### 5.3 部署架构

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 核心服务
  fusion-engine:
    build: ./core
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
      - milvus
      - neo4j

  # 新闻爬虫服务
  news-crawler:
    build: ./agents
    depends_on:
      - fusion-engine
      - celery-worker

  # Celery异步任务
  celery-worker:
    build: ./core
    command: celery -A tasks worker -l info

  celery-beat:
    build: ./core
    command: celery -A tasks beat -l info

  # 数据库
  postgres:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  milvus:
    image: milvusdb/milvus:latest

  neo4j:
    image: neo4j:5
    ports:
      - "7474:7474"
      - "7687:7687"

  # Web前端
  web-ui:
    build: ./ui/web
    ports:
      - "3000:80"

volumes:
  pgdata:
  milvus-data:
  neo4j-data:
```

---

## 六、数据流设计

### 6.1 实时数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                         实时数据流                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  新闻源 ──→ 爬虫服务 ──→ PostgreSQL ──→ LLM分析 ──→ Milvus     │
│                                  │                              │
│                                  ↓                              │
│                           情感/影响评估                          │
│                                  │                              │
│                                  ↓                              │
│                          多空辩论引擎                            │
│                                  │                              │
│                                  ↓                              │
│                           交易信号生成                           │
│                                  │                              │
│                                  ↓                              │
│  VeighNa主引擎 ←── 事件总线 ←── 信号队列                        │
│       │                                                         │
│       ↓                                                         │
│  策略引擎执行                                                    │
│       │                                                         │
│       ↓                                                         │
│  交易网关 ──→ 券商/期货公司                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 历史数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                         历史数据流                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  新闻历史数据 ──→ 向量化 ──→ Milvus                             │
│       │                                                         │
│       ↓                                                         │
│  情感标注/关联标注                                               │
│       │                                                         │
│       ↓                                                         │
│  训练数据集构建                                                  │
│       │                                                         │
│       ↓                                                         │
│  ML模型训练 (vnpy.alpha)                                        │
│       │                                                         │
│       ↓                                                         │
│  策略回测验证                                                    │
│       │                                                         │
│       ↓                                                         │
│  策略优化迭代                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、实施路线图

### 7.1 阶段划分

```
Phase 1: 基础融合 (4周)
├── Week 1-2: 项目结构整合
│   ├── 创建融合项目骨架
│   ├── 迁移FinnewsHunter核心模块
│   └── 迁移VeighNa核心模块
│
├── Week 3: 数据层融合
│   ├── 统一数据库设计
│   ├── 配置Milvus/Neo4j
│   └── 数据迁移脚本
│
└── Week 4: 事件总线对接
    ├── 设计统一事件格式
    ├── 实现事件转换器
    └── 单元测试

Phase 2: 核心功能开发 (6周)
├── Week 5-6: 信号转换器
│   ├── 新闻→信号映射逻辑
│   ├── 置信度计算算法
│   └── 信号有效期管理
│
├── Week 7-8: 新闻驱动策略
│   ├── 策略基类开发
│   ├── 信号接收/处理逻辑
│   └── 回测框架适配
│
├── Week 9-10: 多智能体融合
│   ├── 辩论结果→交易决策
│   ├── 风险评估智能体
│   └── 实时决策推送
│
└── Week 11-12: 前端融合
    ├── Web界面整合
    ├── 信号监控面板
    └── 实时辩论可视化

Phase 3: 高级功能 (4周)
├── Week 13-14: 知识图谱应用
│   ├── 股票关系网络构建
│   ├── 新闻传导分析
│   └── 风险传染评估
│
├── Week 15-16: AI策略增强
│   ├── 新闻因子集成到vnpy.alpha
│   ├── 模型训练流程
│   └── 自动化调参
│
└── Week 17-18: 实盘对接
    ├── 交易接口对接
    ├── 风控系统
    └── 监控告警

Phase 4: 优化与部署 (2周)
├── Week 19: 性能优化
│   ├── 延迟优化
│   ├── 并发处理
│   └── 内存管理
│
└── Week 20: 生产部署
    ├── Docker编排优化
    ├── 日志/监控
    └── 文档完善
```

### 7.2 里程碑

| 里程碑 | 时间节点 | 交付物 |
|--------|----------|--------|
| M1: 基础框架 | 第4周 | 可运行的项目骨架，数据层打通 |
| M2: 信号闭环 | 第12周 | 新闻→信号→策略完整链路 |
| M3: 功能完整 | 第16周 | 所有核心功能可用 |
| M4: 生产就绪 | 第20周 | 可实盘部署的系统 |

---

## 八、风险评估与缓解措施

### 8.1 技术风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| LLM响应延迟 | 高 | 中 | 异步处理+信号缓存，设置超时降级 |
| 多智能体协调复杂 | 中 | 高 | 清晰的角色定义，完善的日志追踪 |
| 新闻爬虫反爬 | 高 | 中 | 多IP轮换、模拟人类行为、备用数据源 |
| 实盘交易风险 | 中 | 高 | 严格的风控策略、模拟盘先行 |

### 8.2 业务风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 新闻信号质量不稳定 | 高 | 高 | 多信号交叉验证、人工审核机制 |
| 过度拟合历史数据 | 中 | 高 | 样本外测试、滚动回测 |
| 市场极端情况 | 中 | 高 | 止损机制、最大回撤控制 |

### 8.3 运维风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 服务宕机 | 中 | 高 | 高可用部署、自动故障转移 |
| 数据丢失 | 低 | 高 | 多副本备份、异地灾备 |
| API限流 | 中 | 中 | 请求队列、多账号轮换 |

---

## 九、关键代码示例

### 9.1 融合引擎启动

```python
from vnpy.event import EventEngine
from vnpy.trader.engine import MainEngine
from vnpy_ctp import CtpGateway

from fusion.core.event_bus import FusionEventBus
from fusion.core.signal_converter import NewsSignalConverter
from fusion.agents import DebateOrchestrator
from fusion.strategies import NewsDrivenStrategy

class FusionTradingEngine:
    """融合交易引擎"""

    def __init__(self):
        # 初始化VeighNa核心
        self.event_engine = EventEngine()
        self.main_engine = MainEngine(self.event_engine)

        # 添加交易接口
        self.main_engine.add_gateway(CtpGateway)

        # 初始化融合组件
        self.fusion_bus = FusionEventBus(self.event_engine)
        self.signal_converter = NewsSignalConverter()
        self.debate_orchestrator = DebateOrchestrator()

        # 注册事件处理
        self._register_handlers()

    def _register_handlers(self):
        """注册事件处理器"""
        # VeighNa事件
        self.event_engine.register(EVENT_TICK, self._on_tick)

        # 融合事件
        self.fusion_bus.subscribe("NEWS_ANALYZED", self._on_news_analyzed)
        self.fusion_bus.subscribe("DEBATE_COMPLETED", self._on_debate_completed)

    def _on_news_analyzed(self, event):
        """新闻分析完成事件"""
        analysis = event.data

        # 触发辩论
        self.debate_orchestrator.start_debate(
            news_id=analysis.news_id,
            stock_codes=analysis.stock_codes
        )

    def _on_debate_completed(self, event):
        """辩论完成事件"""
        debate_result = event.data

        # 转换为交易信号
        signal = self.signal_converter.convert(debate_result)

        # 发布信号事件
        self.fusion_bus.publish("TRADING_SIGNAL", signal)

    def start(self):
        """启动引擎"""
        self.main_engine.connect(gateway_name="CTP", setting={})
        print("融合交易引擎启动成功")
```

### 9.2 新闻驱动策略

```python
from vnpy.trader.object import BarData, TickData, OrderData, TradeData
from vnpy.trader.constant import Direction, Offset
from vnpy_ctastrategy import CtaTemplate

class NewsMomentumStrategy(CtaTemplate):
    """新闻动量策略"""

    author = "Fusion Trading Team"

    # 策略参数
    sentiment_threshold = 0.7
    confidence_threshold = 0.8
    signal_expire_seconds = 300
    stop_loss_pct = 0.02
    take_profit_pct = 0.05

    # 策略变量
    current_signal = None
    signal_time = None

    parameters = [
        "sentiment_threshold",
        "confidence_threshold",
        "signal_expire_seconds"
    ]

    variables = [
        "current_signal",
        "signal_time"
    ]

    def on_init(self):
        """初始化"""
        self.write_log("策略初始化")

    def on_start(self):
        """启动"""
        self.write_log("策略启动")

    def on_signal(self, signal: TradingSignal):
        """接收交易信号"""
        # 验证信号
        if signal.confidence < self.confidence_threshold:
            return

        if signal.sentiment < self.sentiment_threshold:
            return

        # 检查信号是否过期
        if signal.is_expired():
            return

        # 缓存信号
        self.current_signal = signal
        self.signal_time = signal.timestamp

        # 执行交易
        self._execute_signal(signal)

    def _execute_signal(self, signal: TradingSignal):
        """执行交易信号"""
        if signal.direction == "LONG":
            self.buy(signal.price, self.pos_size)
        elif signal.direction == "SHORT":
            self.short(signal.price, self.pos_size)

    def on_bar(self, bar: BarData):
        """K线更新"""
        # 检查信号是否过期
        if self.current_signal:
            elapsed = (bar.datetime - self.signal_time).total_seconds()
            if elapsed > self.signal_expire_seconds:
                self.current_signal = None

    def on_trade(self, trade: TradeData):
        """成交回报"""
        pass
```

---

## 十、数据对象映射关系

### 10.1 核心数据结构映射

FinnewsHunter 与 VeighNa 的数据对象需要进行双向映射：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         数据对象映射关系                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FinnewsHunter                    │  VeighNa                               │
│  ─────────────────────────────────┼─────────────────────────────────────── │
│                                   │                                        │
│  News (新闻模型)                   │  NewsData (新增)                       │
│  ├─ id: int                       │  ├─ news_id: int                       │
│  ├─ title: str                    │  ├─ title: str                         │
│  ├─ content: str                  │  ├─ content: str                       │
│  ├─ source: str                   │  ├─ source: str                        │
│  ├─ stock_codes: List[str]        │  ├─ related_symbols: List[str]         │
│  ├─ sentiment_score: float        │  ├─ sentiment: float                   │
│  └─ publish_time: datetime        │  └─ datetime: datetime                 │
│                                   │                                        │
│  Analysis (分析结果)               │  AnalysisData (新增)                   │
│  ├─ news_id: int                  │  ├─ news_id: int                       │
│  ├─ agent_name: str               │  ├─ agent_name: str                    │
│  ├─ sentiment: str                │  ├─ sentiment: str                     │
│  ├─ sentiment_score: float        │  ├─ sentiment_score: float             │
│  ├─ confidence: float             │  ├─ confidence: float                  │
│  └─ structured_data: JSON         │  └─ structured_data: dict              │
│                                   │                                        │
│  DebateResult (辩论结果)           │  TradingSignal (新增)                  │
│  ├─ stock_code: str               │  ├─ symbol: str                        │
│  ├─ bull_score: float             │  ├─ direction: Direction               │
│  ├─ bear_score: float             │  ├─ strength: float                    │
│  ├─ final_decision: str           │  ├─ confidence: float                  │
│  ├─ confidence: float             │  ├─ valid_until: datetime              │
│  └─ investment_rating: str        │  └─ source: str                        │
│                                   │                                        │
│                                   │  VeighNa 内置对象                       │
│                                   │  ├─ TickData (Tick行情)                │
│                                   │  ├─ BarData (K线数据)                  │
│                                   │  ├─ OrderData (订单)                   │
│                                   │  ├─ TradeData (成交)                   │
│                                   │  ├─ PositionData (持仓)                │
│                                   │  └─ AccountData (账户)                 │
│                                   │                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 统一数据对象定义

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Dict, Any, Optional
from enum import Enum

from vnpy.trader.object import BaseData
from vnpy.trader.constant import Direction, Exchange

class SignalSource(Enum):
    """信号来源"""
    NEWS = "news"               # 新闻情感信号
    DEBATE = "debate"           # 多空辩论信号
    ML = "ml"                   # 机器学习信号
    HYBRID = "hybrid"           # 混合信号

class Sentiment(Enum):
    """情感类型"""
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

@dataclass
class NewsData(BaseData):
    """新闻数据对象（融合定义）"""
    news_id: int
    title: str
    content: str
    source: str                        # 新闻来源
    url: str = ""
    related_symbols: List[str] = field(default_factory=list)
    sentiment: Sentiment = Sentiment.NEUTRAL
    sentiment_score: float = 0.0
    confidence: float = 0.0
    datetime: datetime = field(default_factory=datetime.now)

    def __post_init__(self):
        self.vt_symbol = f"NEWS.{self.news_id}"

@dataclass
class AnalysisData(BaseData):
    """分析结果数据对象"""
    news_id: int
    agent_name: str
    agent_role: str = ""
    sentiment: Sentiment = Sentiment.NEUTRAL
    sentiment_score: float = 0.0
    confidence: float = 0.0
    analysis_result: str = ""
    structured_data: Dict[str, Any] = field(default_factory=dict)
    datetime: datetime = field(default_factory=datetime.now)

@dataclass
class TradingSignal(BaseData):
    """交易信号数据对象"""
    symbol: str                        # 标的代码 (如 SH600519)
    exchange: Exchange = Exchange.SSE  # 交易所
    direction: Direction = Direction.NET  # 方向
    strength: float = 0.0              # 信号强度 (0-100)
    confidence: float = 0.0            # 置信度 (0-1)
    source: SignalSource = SignalSource.NEWS
    valid_until: datetime = field(default_factory=datetime.now)
    price: float = 0.0                 # 建议价格
    metadata: Dict[str, Any] = field(default_factory=dict)

    def __post_init__(self):
        self.vt_symbol = f"{self.symbol}.{self.exchange.value}"

    def is_expired(self) -> bool:
        """检查信号是否过期"""
        return datetime.now() > self.valid_until

    def is_valid(self) -> bool:
        """检查信号是否有效"""
        return not self.is_expired() and self.confidence > 0

@dataclass
class DebateResult(BaseData):
    """辩论结果数据对象"""
    debate_id: str
    stock_code: str
    stock_name: str = ""

    # 多空评分
    bull_score: float = 0.0            # 多方得分 (0-100)
    bear_score: float = 0.0            # 空方得分 (0-100)

    # 最终决策
    final_decision: str = "neutral"    # bull / bear / neutral
    confidence: float = 0.0

    # 投资建议
    investment_rating: str = ""        # strong_buy / buy / hold / sell / strong_sell
    target_price: Optional[float] = None

    # 详细内容
    bull_arguments: List[str] = field(default_factory=list)
    bear_arguments: List[str] = field(default_factory=list)
    risk_warnings: List[str] = field(default_factory=list)

    # 元数据
    debate_rounds: int = 0
    total_tokens: int = 0
    duration_seconds: float = 0.0
    datetime: datetime = field(default_factory=datetime.now)
```

### 10.3 数据转换器实现

```python
from typing import Optional

class DataConverter:
    """数据转换器：FinnewsHunter ↔ VeighNa"""

    @staticmethod
    def news_to_newsdata(news: "News") -> NewsData:
        """将 FinnewsHunter News 转换为 NewsData"""
        return NewsData(
            gateway_name="finnews",
            news_id=news.id,
            title=news.title,
            content=news.content,
            source=news.source,
            url=news.url,
            related_symbols=news.stock_codes or [],
            sentiment=Sentiment(news.sentiment) if news.sentiment else Sentiment.NEUTRAL,
            sentiment_score=news.sentiment_score or 0.0,
            confidence=news.confidence or 0.0,
            datetime=news.publish_time or datetime.now()
        )

    @staticmethod
    def analysis_to_analysisdata(analysis: "Analysis") -> AnalysisData:
        """将 Analysis 转换为 AnalysisData"""
        return AnalysisData(
            gateway_name="finnews",
            news_id=analysis.news_id,
            agent_name=analysis.agent_name,
            agent_role=analysis.agent_role or "",
            sentiment=Sentiment(analysis.sentiment) if analysis.sentiment else Sentiment.NEUTRAL,
            sentiment_score=analysis.sentiment_score or 0.0,
            confidence=analysis.confidence or 0.0,
            analysis_result=analysis.analysis_result,
            structured_data=analysis.structured_data or {},
            datetime=analysis.created_at or datetime.now()
        )

    @staticmethod
    def debate_to_signal(result: DebateResult) -> TradingSignal:
        """将辩论结果转换为交易信号"""
        # 确定方向
        if result.bull_score > result.bear_score + 20:
            direction = Direction.LONG
            strength = result.bull_score
        elif result.bear_score > result.bull_score + 20:
            direction = Direction.SHORT
            strength = result.bear_score
        else:
            direction = Direction.NET
            strength = 0

        # 解析交易所
        exchange = Exchange.SSE if result.stock_code.startswith("SH") else Exchange.SZSE

        return TradingSignal(
            gateway_name="debate",
            symbol=result.stock_code,
            exchange=exchange,
            direction=direction,
            strength=strength,
            confidence=result.confidence,
            source=SignalSource.DEBATE,
            valid_until=datetime.now() + timedelta(hours=24),  # 信号有效期24小时
            metadata={
                "debate_id": result.debate_id,
                "bull_score": result.bull_score,
                "bear_score": result.bear_score,
                "investment_rating": result.investment_rating,
                "target_price": result.target_price
            }
        )

    @staticmethod
    def signal_to_order_request(
        signal: TradingSignal,
        volume: float,
        price: float = 0.0
    ) -> "OrderRequest":
        """将信号转换为订单请求"""
        from vnpy.trader.object import OrderRequest
        from vnpy.trader.constant import OrderType

        return OrderRequest(
            symbol=signal.symbol,
            exchange=signal.exchange,
            direction=signal.direction,
            type=OrderType.MARKET if price == 0 else OrderType.LIMIT,
            volume=volume,
            price=price,
            reference=f"signal_{signal.source.value}"
        )
```

---

## 十一、事件系统对接方案

### 11.1 事件类型统一定义

```python
from enum import Enum

class FusionEventType(Enum):
    """融合事件类型"""

    # === VeighNa 原生事件 ===
    EVENT_TICK = "EVENT_TICK"               # Tick行情推送
    EVENT_BAR = "EVENT_BAR"                 # K线数据推送
    EVENT_ORDER = "EVENT_ORDER"             # 订单状态更新
    EVENT_TRADE = "EVENT_TRADE"             # 成交推送
    EVENT_POSITION = "EVENT_POSITION"       # 持仓更新
    EVENT_ACCOUNT = "EVENT_ACCOUNT"         # 账户资金更新
    EVENT_CONTRACT = "EVENT_CONTRACT"       # 合约信息
    EVENT_TIMER = "EVENT_TIMER"             # 定时器事件
    EVENT_LOG = "EVENT_LOG"                 # 日志事件

    # === FinnewsHunter 事件 ===
    EVENT_NEWS = "EVENT_NEWS"               # 新新闻入库
    EVENT_NEWS_ANALYZED = "EVENT_NEWS_ANALYZED"  # 新闻分析完成
    EVENT_DEBATE_START = "EVENT_DEBATE_START"    # 辩论开始
    EVENT_DEBATE_ROUND = "EVENT_DEBATE_ROUND"    # 辩论回合
    EVENT_DEBATE_COMPLETE = "EVENT_DEBATE_COMPLETE"  # 辩论完成

    # === 融合新增事件 ===
    EVENT_SIGNAL = "EVENT_SIGNAL"           # 交易信号生成
    SIGNAL_EXPIRED = "SIGNAL_EXPIRED"       # 信号过期
    EVENT_RISK_ALERT = "EVENT_RISK_ALERT"   # 风控预警
    EVENT_TRADE_DECISION = "EVENT_TRADE_DECISION"  # 交易决策
    EVENT_BACKTEST_START = "EVENT_BACKTEST_START"  # 回测开始
    EVENT_BACKTEST_COMPLETE = "EVENT_BACKTEST_COMPLETE"  # 回测完成
```

### 11.2 融合事件总线实现

```python
import asyncio
from queue import Queue
from threading import Thread
from collections import defaultdict
from typing import Callable, Any, Dict, List
from datetime import datetime
import logging

from vnpy.event import EventEngine, Event

logger = logging.getLogger(__name__)

class FusionEventBus:
    """
    融合事件总线

    统一 VeighNa 事件引擎与 FinnewsHunter 事件系统
    支持同步和异步两种模式
    """

    def __init__(self, vnpy_event_engine: EventEngine = None):
        # VeighNa 事件引擎
        self._vnpy_engine = vnpy_event_engine or EventEngine()

        # 扩展事件处理器
        self._handlers: Dict[str, List[Callable]] = defaultdict(list)

        # 事件队列（用于异步处理）
        self._async_queue: asyncio.Queue = asyncio.Queue()

        # 信号缓存
        self._signal_cache: Dict[str, TradingSignal] = {}

        # 启动事件处理
        self._register_bridge()

        logger.info("FusionEventBus 初始化完成")

    def _register_bridge(self):
        """注册 VeighNa 事件桥接"""
        # 桥接 VeighNa 事件到融合事件
        self._vnpy_engine.register_general(self._on_vnpy_event)

    def _on_vnpy_event(self, event: Event):
        """处理 VeighNa 事件"""
        event_type = event.type
        data = event.data

        # 触发扩展处理器
        handlers = self._handlers.get(event_type, [])
        for handler in handlers:
            try:
                handler(data)
            except Exception as e:
                logger.error(f"事件处理器错误: {event_type}, {e}")

    def register(self, event_type: str, handler: Callable):
        """注册事件处理器"""
        # 如果是 VeighNa 原生事件，注册到 VeighNa 引擎
        if event_type.startswith("EVENT_"):
            self._vnpy_engine.register(event_type, handler)
        else:
            # 自定义事件注册到扩展处理器
            self._handlers[event_type].append(handler)

        logger.debug(f"注册事件处理器: {event_type}")

    def unregister(self, event_type: str, handler: Callable):
        """注销事件处理器"""
        if event_type.startswith("EVENT_"):
            self._vnpy_engine.unregister(event_type, handler)
        else:
            if handler in self._handlers.get(event_type, []):
                self._handlers[event_type].remove(handler)

    def emit(self, event_type: str, data: Any):
        """发送事件"""
        if event_type.startswith("EVENT_"):
            # VeighNa 事件
            event = Event(event_type, data)
            self._vnpy_engine.put(event)
        else:
            # 自定义事件
            for handler in self._handlers.get(event_type, []):
                try:
                    handler(data)
                except Exception as e:
                    logger.error(f"事件处理错误: {event_type}, {e}")

    def emit_signal(self, signal: TradingSignal):
        """发送交易信号"""
        # 缓存信号
        self._signal_cache[signal.vt_symbol] = signal

        # 发送信号事件
        self.emit(FusionEventType.EVENT_SIGNAL.value, signal)

        logger.info(f"交易信号: {signal.vt_symbol} {signal.direction.value} "
                   f"强度:{signal.strength:.1f} 置信度:{signal.confidence:.2f}")

    def get_cached_signal(self, vt_symbol: str) -> Optional[TradingSignal]:
        """获取缓存的信号"""
        signal = self._signal_cache.get(vt_symbol)
        if signal and not signal.is_expired():
            return signal
        return None

    def start(self):
        """启动事件引擎"""
        self._vnpy_engine.start()
        logger.info("融合事件总线启动")

    def stop(self):
        """停止事件引擎"""
        self._vnpy_engine.stop()
        logger.info("融合事件总线停止")
```

### 11.3 事件驱动流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         事件驱动流程                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐                                                           │
│   │ 新闻入库    │ ──→ EVENT_NEWS                                            │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 新闻分析    │ ──→ EVENT_NEWS_ANALYZED                                   │
│   │ (LLM调用)  │                                                           │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 辩论触发    │ ──→ EVENT_DEBATE_START                                    │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 多空辩论    │ ──→ EVENT_DEBATE_ROUND (多次)                             │
│   │ (智能体协作)│                                                           │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 辩论完成    │ ──→ EVENT_DEBATE_COMPLETE                                 │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 信号转换    │ ──→ EVENT_SIGNAL                                          │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │ 风控检查    │ ──→ │ 通过/拦截   │                                       │
│   └──────┬──────┘     └──────┬──────┘                                       │
│          │                   │                                              │
│          │  (通过)           │  (拦截)                                      │
│          ▼                   ▼                                              │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │ 策略执行    │     │ EVENT_RISK_ALERT                                   │
│   │             │     └─────────────┘                                       │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 订单执行    │ ──→ EVENT_ORDER                                           │
│   └──────┬──────┘                                                           │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────┐                                                           │
│   │ 成交回报    │ ──→ EVENT_TRADE                                           │
│   └─────────────┘                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 十二、回测系统融合

### 12.1 新闻历史回测引擎

```python
from datetime import datetime, timedelta
from typing import List, Dict, Any, Optional
import pandas as pd

from vnpy.trader.object import BarData, TickData
from vnpy.trader.constant import Exchange, Interval

class NewsBacktestEngine:
    """
    新闻历史回测引擎

    将历史新闻数据与行情数据对齐，验证新闻策略效果
    """

    def __init__(
        self,
        news_data: pd.DataFrame,
        price_data: pd.DataFrame,
        initial_capital: float = 1_000_000
    ):
        """
        Args:
            news_data: 历史新闻数据，包含 ['datetime', 'stock_code', 'sentiment', 'confidence']
            price_data: 历史行情数据，包含 ['datetime', 'symbol', 'open', 'high', 'low', 'close', 'volume']
            initial_capital: 初始资金
        """
        self.news_data = news_data
        self.price_data = price_data
        self.initial_capital = initial_capital

        # 回测结果
        self.trades: List[Dict] = []
        self.signals: List[Dict] = []
        self.equity_curve: List[Dict] = []

        # 统计指标
        self.total_return = 0.0
        self.max_drawdown = 0.0
        self.sharpe_ratio = 0.0
        self.win_rate = 0.0

    def run(
        self,
        start: datetime,
        end: datetime,
        signal_threshold: float = 0.6,
        holding_period: int = 5  # 持仓天数
    ) -> Dict[str, Any]:
        """
        运行回测

        Args:
            start: 回测开始时间
            end: 回测结束时间
            signal_threshold: 信号阈值
            holding_period: 默认持仓周期（天）
        """
        # 筛选回测区间数据
        news_subset = self.news_data[
            (self.news_data['datetime'] >= start) &
            (self.news_data['datetime'] <= end)
        ].copy()

        # 按时间排序
        news_subset = news_subset.sort_values('datetime')

        # 遍历新闻事件
        for idx, news in news_subset.iterrows():
            # 生成信号
            signal = self._generate_signal(news, signal_threshold)
            if not signal:
                continue

            # 获取入场价格
            entry_price = self._get_entry_price(
                news['stock_code'],
                news['datetime'],
                signal['direction']
            )
            if not entry_price:
                continue

            # 获取出场价格
            exit_datetime = news['datetime'] + timedelta(days=holding_period)
            exit_price = self._get_exit_price(
                news['stock_code'],
                exit_datetime,
                signal['direction']
            )
            if not exit_price:
                continue

            # 计算收益
            if signal['direction'] == 'LONG':
                pnl_pct = (exit_price - entry_price) / entry_price
            else:
                pnl_pct = (entry_price - exit_price) / entry_price

            # 记录交易
            trade = {
                'news_id': news.get('id'),
                'stock_code': news['stock_code'],
                'signal_time': news['datetime'],
                'direction': signal['direction'],
                'entry_price': entry_price,
                'exit_price': exit_price,
                'exit_time': exit_datetime,
                'pnl_pct': pnl_pct,
                'sentiment': news['sentiment'],
                'confidence': news['confidence']
            }
            self.trades.append(trade)

        # 计算统计指标
        self._calculate_metrics()

        return {
            'total_trades': len(self.trades),
            'total_return': self.total_return,
            'max_drawdown': self.max_drawdown,
            'sharpe_ratio': self.sharpe_ratio,
            'win_rate': self.win_rate,
            'trades': self.trades
        }

    def _generate_signal(self, news: pd.Series, threshold: float) -> Optional[Dict]:
        """根据新闻生成信号"""
        sentiment = news.get('sentiment_score', 0)
        confidence = news.get('confidence', 0)

        # 阈值过滤
        if confidence < threshold:
            return None

        # 方向判断
        if sentiment > 0.3:
            direction = 'LONG'
        elif sentiment < -0.3:
            direction = 'SHORT'
        else:
            return None

        return {
            'direction': direction,
            'strength': abs(sentiment) * 100,
            'confidence': confidence
        }

    def _get_entry_price(
        self,
        stock_code: str,
        signal_time: datetime,
        direction: str
    ) -> Optional[float]:
        """获取入场价格（次日开盘价）"""
        next_day = signal_time + timedelta(days=1)

        price = self.price_data[
            (self.price_data['symbol'] == stock_code) &
            (self.price_data['datetime'].dt.date == next_day.date())
        ]

        if len(price) > 0:
            return price.iloc[0]['open']
        return None

    def _get_exit_price(
        self,
        stock_code: str,
        exit_time: datetime,
        direction: str
    ) -> Optional[float]:
        """获取出场价格"""
        price = self.price_data[
            (self.price_data['symbol'] == stock_code) &
            (self.price_data['datetime'].dt.date == exit_time.date())
        ]

        if len(price) > 0:
            return price.iloc[0]['close']
        return None

    def _calculate_metrics(self):
        """计算回测指标"""
        if not self.trades:
            return

        # 计算总收益
        returns = [t['pnl_pct'] for t in self.trades]
        self.total_return = sum(returns)

        # 计算胜率
        winning = [r for r in returns if r > 0]
        self.win_rate = len(winning) / len(returns) if returns else 0

        # 计算最大回撤
        cumulative = pd.Series(returns).cumsum()
        rolling_max = cumulative.expanding().max()
        drawdowns = cumulative - rolling_max
        self.max_drawdown = abs(drawdowns.min()) if len(drawdowns) > 0 else 0

        # 计算夏普比率（简化）
        if len(returns) > 1:
            mean_return = pd.Series(returns).mean()
            std_return = pd.Series(returns).std()
            self.sharpe_ratio = mean_return / std_return * (252 ** 0.5) if std_return > 0 else 0
```

### 12.2 辩论历史验证

```python
class DebateBacktester:
    """
    辩论结果历史验证器

    对比辩论结论与后续走势，评估辩论质量
    """

    def __init__(
        self,
        debate_results: List[DebateResult],
        price_data: pd.DataFrame
    ):
        self.debate_results = debate_results
        self.price_data = price_data

        # 评估结果
        self.evaluations: List[Dict] = []
        self.accuracy_metrics: Dict[str, float] = {}

    def evaluate(
        self,
        holding_days: int = 20,
        min_confidence: float = 0.6
    ) -> Dict[str, Any]:
        """
        评估辩论预测准确率

        Args:
            holding_days: 评估周期（天）
            min_confidence: 最低置信度要求
        """
        valid_debates = [
            d for d in self.debate_results
            if d.confidence >= min_confidence
        ]

        for debate in valid_debates:
            # 获取辩论后N天价格
            start_price = self._get_price_after(
                debate.stock_code,
                debate.datetime,
                days=1
            )
            end_price = self._get_price_after(
                debate.stock_code,
                debate.datetime,
                days=holding_days
            )

            if start_price is None or end_price is None:
                continue

            # 计算实际涨跌
            actual_return = (end_price - start_price) / start_price

            # 判断预测是否正确
            predicted_bull = debate.bull_score > debate.bear_score
            actual_bull = actual_return > 0

            is_correct = predicted_bull == actual_bull

            self.evaluations.append({
                'debate_id': debate.debate_id,
                'stock_code': debate.stock_code,
                'datetime': debate.datetime,
                'predicted': 'BULL' if predicted_bull else 'BEAR',
                'actual_return': actual_return,
                'is_correct': is_correct,
                'confidence': debate.confidence,
                'bull_score': debate.bull_score,
                'bear_score': debate.bear_score
            })

        # 计算准确率
        self._calculate_accuracy()

        return self.accuracy_metrics

    def _get_price_after(
        self,
        stock_code: str,
        start_time: datetime,
        days: int
    ) -> Optional[float]:
        """获取指定天数后的收盘价"""
        target_date = start_time + timedelta(days=days)

        price = self.price_data[
            (self.price_data['symbol'] == stock_code) &
            (self.price_data['datetime'].dt.date == target_date.date())
        ]

        if len(price) > 0:
            return price.iloc[0]['close']
        return None

    def _calculate_accuracy(self):
        """计算预测准确率指标"""
        if not self.evaluations:
            return

        total = len(self.evaluations)
        correct = sum(1 for e in self.evaluations if e['is_correct'])

        # 总体准确率
        self.accuracy_metrics['overall_accuracy'] = correct / total

        # 按置信度分层
        high_conf = [e for e in self.evaluations if e['confidence'] >= 0.8]
        if high_conf:
            self.accuracy_metrics['high_confidence_accuracy'] = (
                sum(1 for e in high_conf if e['is_correct']) / len(high_conf)
            )

        # 多头预测准确率
        bull_preds = [e for e in self.evaluations if e['predicted'] == 'BULL']
        if bull_preds:
            self.accuracy_metrics['bull_accuracy'] = (
                sum(1 for e in bull_preds if e['is_correct']) / len(bull_preds)
            )

        # 空头预测准确率
        bear_preds = [e for e in self.evaluations if e['predicted'] == 'BEAR']
        if bear_preds:
            self.accuracy_metrics['bear_accuracy'] = (
                sum(1 for e in bear_preds if e['is_correct']) / len(bear_preds)
            )
```

---

## 十三、风险管理模块

### 13.1 风控智能体

```python
from dataclasses import dataclass
from typing import List, Dict, Any, Optional
from datetime import datetime, timedelta
from enum import Enum

class RiskLevel(Enum):
    """风险等级"""
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

@dataclass
class RiskAlert:
    """风险预警"""
    alert_id: str
    risk_level: RiskLevel
    risk_type: str              # position/exposure/loss/news
    message: str
    details: Dict[str, Any]
    timestamp: datetime
    suggested_action: str = ""  # 建议操作

class RiskManagerAgent:
    """
    风险管理智能体

    实时监控交易风险，包括：
    - 仓位风险
    - 敞口风险
    - 亏损风险
    - 新闻风险
    """

    def __init__(
        self,
        max_position_pct: float = 0.2,      # 单标的最大仓位
        max_sector_pct: float = 0.4,        # 单行业最大仓位
        max_daily_loss: float = 0.03,       # 日最大亏损
        max_drawdown: float = 0.1,          # 最大回撤
        news_risk_threshold: float = 0.7    # 新闻风险阈值
    ):
        self.max_position_pct = max_position_pct
        self.max_sector_pct = max_sector_pct
        self.max_daily_loss = max_daily_loss
        self.max_drawdown = max_drawdown
        self.news_risk_threshold = news_risk_threshold

        # 风险记录
        self.alerts: List[RiskAlert] = []
        self.daily_pnl: float = 0.0
        self.peak_equity: float = 0.0

    def check_signal_risk(
        self,
        signal: TradingSignal,
        current_positions: Dict[str, Any],
        account: "AccountData"
    ) -> Optional[RiskAlert]:
        """检查信号风险"""

        # 1. 检查仓位集中度
        position_risk = self._check_position_concentration(
            signal, current_positions, account
        )
        if position_risk:
            return position_risk

        # 2. 检查行业敞口
        sector_risk = self._check_sector_exposure(
            signal, current_positions
        )
        if sector_risk:
            return sector_risk

        # 3. 检查日亏损限制
        loss_risk = self._check_daily_loss_limit(account)
        if loss_risk:
            return loss_risk

        # 4. 检查新闻风险
        news_risk = self._check_news_risk(signal)
        if news_risk:
            return news_risk

        return None  # 通过风控检查

    def _check_position_concentration(
        self,
        signal: TradingSignal,
        positions: Dict[str, Any],
        account: "AccountData"
    ) -> Optional[RiskAlert]:
        """检查仓位集中度"""
        symbol = signal.symbol
        current_pos = positions.get(symbol, {})

        # 计算当前仓位比例
        current_pct = current_pos.get('market_value', 0) / account.balance

        # 预计新增仓位
        estimated_new_pct = 0.05  # 假设单笔交易5%
        total_pct = current_pct + estimated_new_pct

        if total_pct > self.max_position_pct:
            return RiskAlert(
                alert_id=f"pos_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                risk_level=RiskLevel.HIGH,
                risk_type="position",
                message=f"单标的仓位超限: {symbol} 预计{total_pct:.1%} > 上限{self.max_position_pct:.1%}",
                details={
                    "symbol": symbol,
                    "current_pct": current_pct,
                    "estimated_new_pct": estimated_new_pct,
                    "limit": self.max_position_pct
                },
                timestamp=datetime.now(),
                suggested_action="REJECT"
            )

        return None

    def _check_sector_exposure(
        self,
        signal: TradingSignal,
        positions: Dict[str, Any]
    ) -> Optional[RiskAlert]:
        """检查行业敞口"""
        # 需要行业分类数据
        # 简化示例：假设已有行业映射
        sector = self._get_sector(signal.symbol)

        # 计算行业总仓位
        sector_positions = [
            pos for sym, pos in positions.items()
            if self._get_sector(sym) == sector
        ]
        sector_pct = sum(pos.get('pct', 0) for pos in sector_positions)

        if sector_pct > self.max_sector_pct:
            return RiskAlert(
                alert_id=f"sector_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                risk_level=RiskLevel.MEDIUM,
                risk_type="exposure",
                message=f"行业敞口超限: {sector} 当前{sector_pct:.1%} > 上限{self.max_sector_pct:.1%}",
                details={
                    "sector": sector,
                    "current_pct": sector_pct,
                    "limit": self.max_sector_pct
                },
                timestamp=datetime.now(),
                suggested_action="REDUCE"
            )

        return None

    def _check_daily_loss_limit(
        self,
        account: "AccountData"
    ) -> Optional[RiskAlert]:
        """检查日亏损限制"""
        daily_pnl_pct = self.daily_pnl / account.balance

        if daily_pnl_pct < -self.max_daily_loss:
            return RiskAlert(
                alert_id=f"loss_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                risk_level=RiskLevel.CRITICAL,
                risk_type="loss",
                message=f"日亏损超限: {daily_pnl_pct:.2%} < 下限-{self.max_daily_loss:.2%}",
                details={
                    "daily_pnl": self.daily_pnl,
                    "daily_pnl_pct": daily_pnl_pct,
                    "limit": self.max_daily_loss
                },
                timestamp=datetime.now(),
                suggested_action="STOP_TRADING"
            )

        return None

    def _check_news_risk(
        self,
        signal: TradingSignal
    ) -> Optional[RiskAlert]:
        """检查新闻风险"""
        # 获取相关负面新闻
        negative_news = self._get_negative_news(signal.symbol)

        if negative_news and negative_news.get('severity', 0) > self.news_risk_threshold:
            return RiskAlert(
                alert_id=f"news_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                risk_level=RiskLevel.HIGH,
                risk_type="news",
                message=f"检测到重大负面新闻: {signal.symbol}",
                details={
                    "symbol": signal.symbol,
                    "news_count": negative_news.get('count', 0),
                    "severity": negative_news.get('severity', 0),
                    "headlines": negative_news.get('headlines', [])
                },
                timestamp=datetime.now(),
                suggested_action="CAUTION"
            )

        return None

    def _get_sector(self, symbol: str) -> str:
        """获取股票所属行业（简化实现）"""
        # 实际实现需要行业分类数据
        return "UNKNOWN"

    def _get_negative_news(self, symbol: str) -> Optional[Dict]:
        """获取负面新闻（需要连接新闻服务）"""
        # 实际实现需要查询新闻数据库
        return None
```

### 13.2 风控策略配置

```python
@dataclass
class RiskConfig:
    """风控配置"""

    # 仓位限制
    max_single_position: float = 0.2     # 单标的最大仓位 20%
    max_sector_position: float = 0.4     # 单行业最大仓位 40%
    max_total_position: float = 0.95     # 最大总仓位 95%

    # 亏损限制
    max_daily_loss: float = 0.03         # 日最大亏损 3%
    max_weekly_loss: float = 0.05        # 周最大亏损 5%
    max_drawdown: float = 0.10           # 最大回撤 10%

    # 交易限制
    max_daily_trades: int = 20           # 日最大交易次数
    max_order_size: float = 0.1          # 单笔最大委托比例

    # 信号过滤
    min_signal_confidence: float = 0.6   # 最小信号置信度
    min_signal_strength: float = 30.0    # 最小信号强度

    # 新闻风控
    news_blacklist_hours: int = 24       # 重大负面新闻后禁止交易时长
    sentiment_threshold: float = -0.5    # 情感阈值

class RiskController:
    """风控控制器"""

    def __init__(self, config: RiskConfig):
        self.config = config
        self.agent = RiskManagerAgent(
            max_position_pct=config.max_single_position,
            max_sector_pct=config.max_sector_position,
            max_daily_loss=config.max_daily_loss,
            max_drawdown=config.max_drawdown
        )

        # 交易统计
        self.daily_trade_count = 0
        self.daily_pnl = 0.0

    def should_execute(self, signal: TradingSignal) -> tuple[bool, Optional[str]]:
        """
        判断是否应该执行信号

        Returns:
            (是否执行, 拒绝原因)
        """
        # 检查置信度
        if signal.confidence < self.config.min_signal_confidence:
            return False, f"置信度不足: {signal.confidence:.2f} < {self.config.min_signal_confidence}"

        # 检查信号强度
        if signal.strength < self.config.min_signal_strength:
            return False, f"信号强度不足: {signal.strength:.1f} < {self.config.min_signal_strength}"

        # 检查交易次数
        if self.daily_trade_count >= self.config.max_daily_trades:
            return False, f"已达日交易上限: {self.daily_trade_count}"

        # 检查信号是否过期
        if signal.is_expired():
            return False, "信号已过期"

        return True, None

    def on_trade_executed(self, trade: "TradeData"):
        """交易执行后更新统计"""
        self.daily_trade_count += 1

    def on_daily_reset(self):
        """每日重置"""
        self.daily_trade_count = 0
        self.daily_pnl = 0.0
```

---

## 十四、性能优化方案

### 14.1 延迟优化

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         延迟优化方案                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   模块                    │  当前延迟    │  优化目标   │  优化措施           │
│   ─────────────────────────┼──────────────┼──────────────┼─────────────────   │
│   新闻爬取                 │  5-10s      │  3-5s       │  并行爬虫           │
│   LLM情感分析              │  3-5s       │  2-3s       │  模型量化/缓存      │
│   向量化存储               │  1-2s       │  0.5s       │  异步批处理         │
│   多空辩论                 │  20-40s     │  10-20s     │  并行发言/缓存      │
│   信号转换                 │  0.1s       │  0.05s      │  内存缓存           │
│   风控检查                 │  0.1s       │  0.05s      │  预计算             │
│   订单执行                 │  0.1s       │  0.1s       │  -                  │
│   ─────────────────────────┼──────────────┼──────────────┼─────────────────   │
│   总延迟                   │  ~30-60s    │  ~15-30s    │                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 关键优化代码

```python
import asyncio
from functools import lru_cache
from typing import List, Dict, Any
import redis
import json

class PerformanceOptimizer:
    """性能优化器"""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self._embedding_cache: Dict[str, List[float]] = {}

    # === LLM 响应缓存 ===

    async def get_cached_analysis(self, news_id: int) -> Optional[Dict]:
        """获取缓存的分析结果"""
        key = f"analysis:{news_id}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    async def cache_analysis(self, news_id: int, result: Dict, ttl: int = 3600):
        """缓存分析结果"""
        key = f"analysis:{news_id}"
        self.redis.setex(key, ttl, json.dumps(result))

    # === 批量处理 ===

    async def batch_analyze_news(self, news_list: List[Dict]) -> List[Dict]:
        """批量分析新闻（并行）"""
        tasks = [
            self._analyze_single(news)
            for news in news_list
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results

    async def _analyze_single(self, news: Dict) -> Dict:
        """分析单条新闻"""
        # 检查缓存
        cached = await self.get_cached_analysis(news['id'])
        if cached:
            return cached

        # 调用LLM
        result = await self._call_llm(news)

        # 缓存结果
        await self.cache_analysis(news['id'], result)

        return result

    # === 预计算 ===

    @lru_cache(maxsize=1000)
    def get_stock_sector(self, symbol: str) -> str:
        """获取股票行业（带缓存）"""
        # 查询数据库或API
        pass

    @lru_cache(maxsize=10000)
    def get_stock_name(self, symbol: str) -> str:
        """获取股票名称（带缓存）"""
        pass

    # === 连接池 ===

    def get_db_connection_pool(self):
        """获取数据库连接池"""
        from sqlalchemy import create_engine
        from sqlalchemy.pool import QueuePool

        engine = create_engine(
            "postgresql://...",
            poolclass=QueuePool,
            pool_size=10,
            max_overflow=20,
            pool_pre_ping=True
        )
        return engine

# === 异步辩论优化 ===

class AsyncDebateOrchestrator:
    """异步辩论编排器（并行发言）"""

    async def run_parallel_debate(
        self,
        stock_code: str,
        news_list: List[Dict]
    ) -> DebateResult:
        """
        并行辩论模式

        多方和空方同时生成论点，减少总耗时
        """
        # 并行调用多空双方
        bull_task = asyncio.create_task(
            self._get_bull_analysis(stock_code, news_list)
        )
        bear_task = asyncio.create_task(
            self._get_bear_analysis(stock_code, news_list)
        )

        # 等待双方完成
        bull_analysis, bear_analysis = await asyncio.gather(
            bull_task, bear_task
        )

        # 投资经理综合决策
        result = await self._manager_decision(
            bull_analysis, bear_analysis
        )

        return result
```

### 14.3 数据库优化

```sql
-- 新闻表索引优化
CREATE INDEX idx_news_stock_codes ON news USING GIN (stock_codes);
CREATE INDEX idx_news_sentiment ON news (sentiment_score) WHERE sentiment_score IS NOT NULL;
CREATE INDEX idx_news_created_at ON news (created_at DESC);

-- 分析结果表索引
CREATE INDEX idx_analysis_news_id ON analyses (news_id);
CREATE INDEX idx_analysis_sentiment ON analyses (sentiment, confidence);

-- 分区表（按时间分区）
CREATE TABLE news_2026_q1 PARTITION OF news
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');

CREATE TABLE news_2026_q2 PARTITION OF news
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
```

---

## 十五、测试策略

### 15.1 测试金字塔

```
                    ┌─────────────┐
                    │   E2E测试   │  (10%)
                    │  完整流程   │
                    └─────────────┘
                  ┌───────────────────┐
                  │    集成测试       │  (30%)
                  │  模块间交互       │
                  └───────────────────┘
            ┌───────────────────────────────┐
            │          单元测试              │  (60%)
            │    函数/类/方法级别测试        │
            └───────────────────────────────┘
```

### 15.2 单元测试示例

```python
import pytest
from datetime import datetime, timedelta
from fusion.core.signal_converter import NewsSignalConverter, TradingSignal
from fusion.core.data_objects import Sentiment, SignalSource

class TestSignalConverter:
    """信号转换器测试"""

    def test_sentiment_to_direction_positive(self):
        """测试正面情感转多头信号"""
        converter = NewsSignalConverter()

        analysis = {
            'sentiment_score': 0.7,
            'confidence': 0.8,
            'stock_codes': ['SH600519']
        }

        signal = converter.convert(analysis)

        assert signal.direction == Direction.LONG
        assert signal.confidence == 0.8

    def test_sentiment_to_direction_negative(self):
        """测试负面情感转空头信号"""
        converter = NewsSignalConverter()

        analysis = {
            'sentiment_score': -0.6,
            'confidence': 0.75,
            'stock_codes': ['SZ000858']
        }

        signal = converter.convert(analysis)

        assert signal.direction == Direction.SHORT
        assert signal.confidence == 0.75

    def test_neutral_sentiment_no_signal(self):
        """测试中性情感不生成信号"""
        converter = NewsSignalConverter()

        analysis = {
            'sentiment_score': 0.1,
            'confidence': 0.5,
            'stock_codes': ['SH600519']
        }

        signal = converter.convert(analysis)

        assert signal.direction == Direction.NET
        assert signal.strength == 0

    def test_signal_expiration(self):
        """测试信号过期检测"""
        signal = TradingSignal(
            gateway_name="test",
            symbol="SH600519",
            exchange=Exchange.SSE,
            direction=Direction.LONG,
            valid_until=datetime.now() - timedelta(hours=1)
        )

        assert signal.is_expired() == True

    def test_low_confidence_filter(self):
        """测试低置信度过滤"""
        converter = NewsSignalConverter(min_confidence=0.6)

        analysis = {
            'sentiment_score': 0.8,
            'confidence': 0.4,
            'stock_codes': ['SH600519']
        }

        signal = converter.convert(analysis)

        assert signal is None  # 低置信度不生成信号

class TestRiskManager:
    """风险管理测试"""

    def test_position_limit_check(self):
        """测试仓位限制检查"""
        manager = RiskManagerAgent(max_position_pct=0.2)

        signal = TradingSignal(
            gateway_name="test",
            symbol="SH600519",
            direction=Direction.LONG
        )

        positions = {
            "SH600519": {"market_value": 150000, "pct": 0.15}
        }
        account = type('Account', (), {'balance': 1000000})()

        alert = manager.check_signal_risk(signal, positions, account)

        # 预计仓位会超过限制
        assert alert is not None
        assert alert.risk_type == "position"
```

### 15.3 集成测试示例

```python
import pytest
import asyncio
from fusion.core.fusion_engine import FusionEngine
from fusion.core.event_bus import FusionEventBus, FusionEventType

class TestFusionIntegration:
    """融合系统集成测试"""

    @pytest.fixture
    async def fusion_engine(self):
        """创建融合引擎实例"""
        engine = FusionEngine()
        yield engine
        engine.stop()

    async def test_news_to_signal_flow(self, fusion_engine):
        """测试新闻到信号的完整流程"""
        # 准备测试新闻
        news_data = {
            'id': 1,
            'title': '贵州茅台发布超预期财报',
            'content': '茅台Q4净利润同比增长25%，超出市场预期...',
            'source': 'sina',
            'stock_codes': ['SH600519']
        }

        # 监听信号事件
        received_signal = None

        def on_signal(signal):
            nonlocal received_signal
            received_signal = signal

        fusion_engine.event_bus.register(
            FusionEventType.EVENT_SIGNAL.value,
            on_signal
        )

        # 触发新闻事件
        fusion_engine.event_bus.emit(
            FusionEventType.EVENT_NEWS.value,
            news_data
        )

        # 等待信号生成
        await asyncio.sleep(5)

        # 验证信号
        assert received_signal is not None
        assert received_signal.symbol == 'SH600519'
        assert received_signal.direction in [Direction.LONG, Direction.NET]

class TestBacktestIntegration:
    """回测系统集成测试"""

    def test_news_backtest_accuracy(self):
        """测试新闻回测准确率"""
        # 准备历史数据
        news_data = pd.DataFrame([
            {'datetime': '2025-01-01', 'stock_code': 'SH600519',
             'sentiment_score': 0.6, 'confidence': 0.8}
        ])

        price_data = pd.DataFrame([
            {'datetime': '2025-01-02', 'symbol': 'SH600519',
             'open': 1800, 'close': 1850}
        ])

        # 运行回测
        engine = NewsBacktestEngine(news_data, price_data)
        result = engine.run(
            start=datetime(2025, 1, 1),
            end=datetime(2025, 1, 10)
        )

        # 验证结果
        assert result['total_trades'] == 1
        assert result['win_rate'] == 1.0
```

---

## 十六、监控与运维

### 16.1 监控指标

```yaml
# Prometheus 监控指标

# 新闻采集
- news_crawl_total{source="sina"}           # 爬取新闻总数
- news_crawl_errors{source="sina"}          # 爬取错误数
- news_crawl_latency_seconds{source="sina"} # 爬取延迟

# 分析服务
- analysis_total{agent="news_analyst"}      # 分析总数
- analysis_latency_seconds{agent="news_analyst"}  # 分析延迟
- analysis_llm_tokens{model="qwen-plus"}    # Token消耗

# 辩论服务
- debate_total{result="bull"}               # 辩论总数（按结果分类）
- debate_rounds_histogram                   # 辩论轮次分布
- debate_latency_seconds                    # 辩论耗时

# 交易信号
- signal_total{source="debate"}             # 信号总数
- signal_executed_total{direction="long"}   # 执行信号数
- signal_rejected_total{reason="risk"}      # 拒绝信号数

# 风控系统
- risk_alert_total{level="high"}            # 风险预警数
- position_utilization{symbol="SH600519"}   # 仓位利用率

# 系统健康
- service_health{service="fusion_engine"}   # 服务健康状态
- database_connections_active               # 活跃数据库连接
- redis_memory_used_bytes                   # Redis内存使用
```

### 16.2 告警规则

```yaml
# AlertManager 告警规则

groups:
  - name: fusion_trading_alerts
    rules:
      # 新闻爬虫告警
      - alert: NewsCrawlFailureRate
        expr: rate(news_crawl_errors[5m]) / rate(news_crawl_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "新闻爬取失败率超过10%"
          description: "数据源 {{ $labels.source }} 爬取失败率 {{ $value | humanizePercentage }}"

      # 分析延迟告警
      - alert: AnalysisLatencyHigh
        expr: histogram_quantile(0.95, analysis_latency_seconds) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "分析延迟过高"
          description: "P95延迟 {{ $value }}秒"

      # 辩论超时告警
      - alert: DebateTimeout
        expr: debate_latency_seconds > 120
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "辩论超时"
          description: "辩论耗时 {{ $value }}秒，可能存在问题"

      # 风控告警
      - alert: HighRiskAlert
        expr: increase(risk_alert_total{level="critical"}[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "严重风险预警"
          description: "检测到严重风险，请立即处理"

      # 服务健康告警
      - alert: ServiceDown
        expr: service_health == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务不可用"
          description: "{{ $labels.service }} 服务已停止"
```

### 16.3 日志规范

```python
import logging
import json
from datetime import datetime

class StructuredLogger:
    """结构化日志记录器"""

    def __init__(self, name: str):
        self.logger = logging.getLogger(name)

    def log_event(self, event_type: str, data: dict, level: str = "INFO"):
        """记录结构化事件日志"""
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": event_type,
            "data": data,
            "level": level
        }

        message = json.dumps(log_entry, ensure_ascii=False)

        if level == "ERROR":
            self.logger.error(message)
        elif level == "WARNING":
            self.logger.warning(message)
        else:
            self.logger.info(message)

    def log_news_event(self, news_id: int, action: str, **kwargs):
        """记录新闻事件"""
        self.log_event("NEWS", {
            "news_id": news_id,
            "action": action,
            **kwargs
        })

    def log_debate_event(self, debate_id: str, phase: str, **kwargs):
        """记录辩论事件"""
        self.log_event("DEBATE", {
            "debate_id": debate_id,
            "phase": phase,
            **kwargs
        })

    def log_signal_event(self, signal: TradingSignal, action: str, **kwargs):
        """记录信号事件"""
        self.log_event("SIGNAL", {
            "symbol": signal.symbol,
            "direction": signal.direction.value,
            "strength": signal.strength,
            "confidence": signal.confidence,
            "action": action,
            **kwargs
        })

    def log_trade_event(self, order_id: str, action: str, **kwargs):
        """记录交易事件"""
        self.log_event("TRADE", {
            "order_id": order_id,
            "action": action,
            **kwargs
        })

# 使用示例
logger = StructuredLogger("fusion_trading")

# 记录新闻入库
logger.log_news_event(
    news_id=12345,
    action="CRAWLED",
    source="sina",
    title="茅台股价创新高"
)

# 记录辩论
logger.log_debate_event(
    debate_id="debate_001",
    phase="BULL_ANALYSIS",
    stock_code="SH600519",
    round=1
)

# 记录信号
logger.log_signal_event(
    signal=trading_signal,
    action="GENERATED",
    source="debate"
)
```

### 16.4 运维脚本

```bash
#!/bin/bash
# fusion_ops.sh - 融合系统运维脚本

# 服务管理
start_all() {
    echo "启动所有服务..."
    docker-compose -f docker-compose.yml up -d
    docker-compose -f docker-compose.yml logs -f
}

stop_all() {
    echo "停止所有服务..."
    docker-compose -f docker-compose.yml down
}

restart_service() {
    local service=$1
    echo "重启服务: $service"
    docker-compose -f docker-compose.yml restart $service
}

# 健康检查
health_check() {
    echo "执行健康检查..."

    # 检查 PostgreSQL
    docker exec fusion_postgres pg_isready || echo "PostgreSQL 不可用"

    # 检查 Redis
    docker exec fusion_redis redis-cli ping || echo "Redis 不可用"

    # 检查 Milvus
    curl -s http://localhost:19530/healthz || echo "Milvus 不可用"

    # 检查 API
    curl -s http://localhost:8000/health || echo "API 服务不可用"
}

# 数据备份
backup_data() {
    local backup_dir="/backup/$(date +%Y%m%d)"
    mkdir -p $backup_dir

    echo "备份数据到 $backup_dir..."

    # 备份 PostgreSQL
    docker exec fusion_postgres pg_dump -U finnews finnews_db > $backup_dir/postgres_backup.sql

    # 备份 Redis
    docker exec fusion_redis redis-cli BGSAVE
    docker cp fusion_redis:/data/dump.rdb $backup_dir/redis_backup.rdb

    echo "备份完成"
}

# 日志查看
view_logs() {
    local service=$1
    local tail=${2:-100}
    docker-compose -f docker-compose.yml logs --tail=$tail $service
}

# 重置系统
reset_system() {
    echo "警告：将清除所有数据！"
    read -p "确认重置？(yes/no): " confirm

    if [ "$confirm" == "yes" ]; then
        docker-compose -f docker-compose.yml down -v
        docker-compose -f docker-compose.yml up -d
        echo "系统已重置"
    fi
}

# 主菜单
case "$1" in
    start) start_all ;;
    stop) stop_all ;;
    restart) restart_service $2 ;;
    health) health_check ;;
    backup) backup_data ;;
    logs) view_logs $2 $3 ;;
    reset) reset_system ;;
    *) echo "用法: $0 {start|stop|restart|health|backup|logs|reset}" ;;
esac
```

---

## 十七、总结与展望

### 17.1 核心价值

本融合方案将两个优秀的开源项目深度融合，创造独特价值：

1. **信息优势**：将非结构化新闻信息转化为结构化交易信号
2. **决策质量**：多智能体辩论机制提供更全面的视角
3. **执行能力**：VeighNa成熟的交易执行框架保障落地
4. **闭环迭代**：从研究到实盘的完整闭环

### 17.2 差异化优势

| 维度 | 传统量化 | 本方案 |
|------|----------|--------|
| 数据源 | 结构化数据为主 | 新闻+结构化数据融合 |
| 决策机制 | 单一模型/规则 | 多智能体辩论 |
| 适应性 | 静态规则 | 动态学习 |
| 可解释性 | 黑盒 | 辩论过程可追溯 |

### 17.3 未来方向

1. **强化学习集成**：让智能体从历史交易中学习
2. **多市场扩展**：支持更多海外市场
3. **个性化配置**：用户可自定义智能体角色
4. **社区生态**：开放插件市场，吸引开发者贡献

---

**文档版本**: v2.0
**创建日期**: 2026-03-21
**更新日期**: 2026-03-21
**作者**: AI Multi-Agent Team