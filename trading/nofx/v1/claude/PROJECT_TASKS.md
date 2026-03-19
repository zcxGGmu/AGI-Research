# NoFx v2 项目开发方案：Task-by-Task 实施计划

> 版本: 1.0 | 日期: 2026-03-19 | 基于 REFACTOR_PLAN.md 的细化执行方案

---

## 总览

本文档将 REFACTOR_PLAN.md 中的 5 个 Phase 拆解为 **可执行的原子任务**，每个任务包含：
- 明确的输入/输出文件
- 验收标准（Definition of Done）
- 依赖关系
- 预估工时

**总任务数：98 个** | **总工期：8-10 周** | **里程碑：6 个**

---

## 文件结构总览（最终交付物）

```
nofx-v2/
├── pyproject.toml
├── uv.lock
├── langgraph.json
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── .env.example
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       ├── 001_initial_schema.py
│       └── 002_langgraph_tables.py
│
├── src/
│   ├── __init__.py
│   ├── main.py                          # FastAPI 入口
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py                  # Pydantic Settings
│   │   └── constants.py                 # 港A股常量（交易时间、涨跌停等）
│   │
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt.py                       # JWT 签发/验证
│   │   ├── password.py                  # bcrypt 哈希
│   │   ├── otp.py                       # TOTP (pyotp)
│   │   └── dependencies.py             # FastAPI 依赖注入
│   │
│   ├── crypto/
│   │   ├── __init__.py
│   │   ├── rsa.py                       # RSA 密钥管理
│   │   └── encryption.py               # AES-256-GCM + RSA 混合加密
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py                      # SQLAlchemy Base
│   │   ├── user.py
│   │   ├── ai_model.py
│   │   ├── broker.py                    # 替代 exchange
│   │   ├── trader.py
│   │   ├── decision_log.py
│   │   └── system_config.py
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── session.py                   # async engine + session
│   │   └── crud.py                      # CRUD 操作
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── app.py                       # FastAPI app + lifespan
│   │   ├── middleware.py                # CORS, 错误处理
│   │   ├── deps.py                      # 公共依赖
│   │   └── routes/
│   │       ├── __init__.py
│   │       ├── auth.py                  # 认证路由 (8 endpoints)
│   │       ├── traders.py               # Trader CRUD (12 endpoints)
│   │       ├── models.py                # AI 模型配置 (4 endpoints)
│   │       ├── brokers.py               # 券商配置 (4 endpoints)
│   │       ├── market.py                # 行情数据 (4 endpoints)
│   │       ├── decisions.py             # 决策日志 (4 endpoints)
│   │       ├── competition.py           # 竞赛排行 (3 endpoints)
│   │       ├── crypto_keys.py           # RSA 加密 (2 endpoints)
│   │       └── system.py               # 系统配置 (2 endpoints)
│   │
│   ├── market/
│   │   ├── __init__.py
│   │   ├── provider.py                  # MarketDataProvider ABC
│   │   ├── akshare_provider.py          # AKShare 实现
│   │   ├── futu_provider.py             # 富途实时行情
│   │   ├── types.py                     # StockQuote, StockKline, etc.
│   │   ├── indicators.py               # TA-Lib 技术指标
│   │   └── cache.py                     # Redis 行情缓存
│   │
│   ├── broker/
│   │   ├── __init__.py
│   │   ├── interface.py                 # BrokerInterface ABC
│   │   ├── types.py                     # OrderRequest, OrderResult, etc.
│   │   ├── futu_broker.py               # 富途交易
│   │   ├── longbridge_broker.py         # 长桥交易
│   │   └── simulator.py                 # 模拟交易
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── state.py                     # TradingState 定义
│   │   ├── graph.py                     # 主图构建 + 编译
│   │   ├── supervisor.py                # Supervisor 节点
│   │   ├── research_agent.py            # 市场研究
│   │   ├── technical_analyst.py         # 技术分析
│   │   ├── fundamental_analyst.py       # 基本面分析
│   │   ├── sentiment_analyst.py         # 舆情分析
│   │   ├── synthesizer.py              # 信号综合
│   │   ├── risk_manager.py             # 风控审核
│   │   └── execution_agent.py          # 交易执行
│   │
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── market_data.py              # 行情数据工具
│   │   ├── technical_indicators.py     # 技术指标工具
│   │   ├── fundamental_data.py         # 基本面数据工具
│   │   ├── news_sentiment.py           # 新闻舆情工具
│   │   ├── broker_api.py               # 券商交易工具
│   │   └── risk_calculator.py          # 风险计算工具
│   │
│   ├── scheduler/
│   │   ├── __init__.py
│   │   └── trading_scheduler.py        # APScheduler 定时任务
│   │
│   └── utils/
│       ├── __init__.py
│       ├── logger.py                    # 结构化日志
│       └── telegram.py                  # Telegram 通知
│
├── prompts/
│   ├── supervisor.md
│   ├── technical_analyst.md
│   ├── fundamental_analyst.md
│   ├── sentiment_analyst.md
│   ├── risk_manager.md
│   └── synthesizer.md
│
├── tests/
│   ├── conftest.py                      # pytest fixtures
│   ├── unit/
│   │   ├── test_market_types.py
│   │   ├── test_indicators.py
│   │   ├── test_broker_simulator.py
│   │   ├── test_risk_manager.py
│   │   ├── test_state.py
│   │   ├── test_auth.py
│   │   └── test_constants.py
│   ├── integration/
│   │   ├── test_akshare_provider.py
│   │   ├── test_graph_flow.py
│   │   ├── test_api_auth.py
│   │   ├── test_api_traders.py
│   │   └── test_decision_logging.py
│   └── e2e/
│       └── (Phase 5)
│
└── web/                                 # 从原项目复制并修改
    ├── package.json
    ├── vite.config.ts                   # proxy target 改为 :8000
    └── src/
        ├── types.ts                     # 重写
        ├── lib/api.ts                   # 重写
        └── ...                          # 组件级改造
```

---

## Phase 0: 基础设施搭建

### 里程碑 M0: Python 项目骨架可运行，`uvicorn` 启动无报错

#### T-0.1: 初始化 Python 项目

**输入**: 无
**输出文件**: `pyproject.toml`, `.python-version`, `uv.lock`, `src/__init__.py`

```toml
# pyproject.toml 核心内容
[project]
name = "nofx-v2"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.14",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "redis>=5.0",
    "langgraph>=1.1",
    "langgraph-checkpoint-postgres>=2.0",
    "langchain-openai>=0.3",
    "langchain-community>=0.3",
    "akshare>=1.14",
    "ta-lib>=0.5",
    "passlib[bcrypt]>=1.7",
    "python-jose[cryptography]>=3.3",
    "pyotp>=2.9",
    "cryptography>=43.0",
    "httpx>=0.27",
    "apscheduler>=3.10",
    "python-dotenv>=1.0",
    "loguru>=0.7",
    "python-telegram-bot>=21.0",
    "sse-starlette>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-asyncio>=0.24", "pytest-cov>=5.0", "httpx>=0.27", "ruff>=0.8"]
futu = ["futu-api>=9.0"]
longbridge = ["longport>=1.0"]
```

**验收标准**:
- [ ] `uv sync` 成功安装所有依赖
- [ ] `python -c "import fastapi; import langgraph; print('OK')"` 通过
- [ ] `.gitignore` 包含 `__pycache__`, `.venv`, `.env`, `*.pyc`

**预估工时**: 0.5h | **依赖**: 无

---

#### T-0.2: 配置管理模块

**输入**: 原 `config/config.go` 的配置结构
**输出文件**: `src/config/settings.py`, `src/config/constants.py`, `.env.example`

```python
# src/config/settings.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Database
    database_url: str = "postgresql+asyncpg://user:pass@localhost:5432/nofx_v2"
    redis_url: str = "redis://localhost:6379/0"

    # Auth
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expire_hours: int = 24

    # LLM
    openai_api_key: str = ""
    deepseek_api_key: str = ""

    # Broker
    futu_host: str = "localhost"
    futu_port: int = 11111
    futu_trade_env: str = "SIMULATE"

    # App
    beta_mode: bool = True
    api_port: int = 8000
    log_level: str = "INFO"

    # Telegram
    telegram_bot_token: str = ""
    telegram_chat_id: int = 0

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

    @property
    def sync_database_url(self) -> str:
        return self.database_url.replace("+asyncpg", "")
```

```python
# src/config/constants.py
from enum import Enum
from datetime import time

class Market(str, Enum):
    A_SHARE = "A"
    HK_STOCK = "HK"

class StockBoard(str, Enum):
    MAIN = "main"       # 主板 ±10%
    GEM = "gem"          # 创业板 ±20%
    STAR = "star"        # 科创板 ±20%
    BSE = "bse"          # 北交所 ±30%

# 涨跌停幅度
PRICE_LIMITS = {
    StockBoard.MAIN: 0.10,
    StockBoard.GEM: 0.20,
    StockBoard.STAR: 0.20,
    StockBoard.BSE: 0.30,
}

# A股交易时间
A_SHARE_TRADING_HOURS = [
    (time(9, 30), time(11, 30)),
    (time(13, 0), time(15, 0)),
]

# 港股交易时间
HK_TRADING_HOURS = [
    (time(9, 30), time(12, 0)),
    (time(13, 0), time(16, 0)),
]

# A股最小交易单位
A_SHARE_LOT_SIZE = 100

# 手续费
COMMISSION_RATE = 0.0003  # 万三
STAMP_TAX_RATE = 0.001    # 千一（卖出时）
```

**验收标准**:
- [ ] `Settings()` 可从 `.env` 加载
- [ ] 所有常量有类型注解
- [ ] `pytest tests/unit/test_constants.py` 通过

**预估工时**: 1h | **依赖**: T-0.1

---

#### T-0.3: 数据库连接 + SQLAlchemy 基础

**输入**: 无
**输出文件**: `src/db/session.py`, `src/models/base.py`

```python
# src/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from src.config.settings import Settings

settings = Settings()
engine = create_async_engine(settings.database_url, echo=False, pool_size=10, max_overflow=20)
async_session = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with async_session() as session:
        yield session
```

```python
# src/models/base.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import func
from datetime import datetime
import uuid

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime] = mapped_column(default=func.now(), onupdate=func.now())

class UUIDMixin:
    id: Mapped[str] = mapped_column(primary_key=True, default=lambda: str(uuid.uuid4()))
```

**验收标准**:
- [ ] `engine.connect()` 可连接 PostgreSQL
- [ ] `Base.metadata` 可被 Alembic 发现

**预估工时**: 1h | **依赖**: T-0.1, T-0.2

---

#### T-0.4: 数据库模型定义（全部 6 张表）

**输入**: 原 `config/database.go` 的 schema + REFACTOR_PLAN.md 第 9 节
**输出文件**: `src/models/user.py`, `src/models/ai_model.py`, `src/models/broker.py`, `src/models/trader.py`, `src/models/decision_log.py`, `src/models/system_config.py`

每个模型文件对应一张表，字段严格按 REFACTOR_PLAN.md 第 9.1 节定义。

**验收标准**:
- [ ] 6 个模型文件，每个 < 80 行
- [ ] 外键关系正确（user → ai_model, broker, trader）
- [ ] `traders.watchlist` 使用 `ARRAY(String)` 类型
- [ ] 敏感字段标注注释（需加密存储）

**预估工时**: 2h | **依赖**: T-0.3

---

#### T-0.5: Alembic 迁移配置 + 初始迁移

**输入**: T-0.4 的模型定义
**输出文件**: `alembic.ini`, `alembic/env.py`, `alembic/versions/001_initial_schema.py`

**验收标准**:
- [ ] `alembic upgrade head` 在空数据库上成功创建所有表
- [ ] `alembic downgrade base` 可回滚
- [ ] `env.py` 使用 async engine

**预估工时**: 1.5h | **依赖**: T-0.4

---

#### T-0.6: Redis 连接 + 缓存工具

**输入**: 无
**输出文件**: `src/market/cache.py`

```python
# src/market/cache.py
import redis.asyncio as redis
import json
from functools import wraps
from src.config.settings import Settings

_redis: redis.Redis | None = None

async def get_redis() -> redis.Redis:
    global _redis
    if _redis is None:
        settings = Settings()
        _redis = redis.from_url(settings.redis_url, decode_responses=True)
    return _redis

def cached(prefix: str, ttl: int = 60):
    """Redis 缓存装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            r = await get_redis()
            key = f"{prefix}:{':'.join(str(a) for a in args)}"
            cached_val = await r.get(key)
            if cached_val:
                return json.loads(cached_val)
            result = await func(*args, **kwargs)
            await r.setex(key, ttl, json.dumps(result, default=str))
            return result
        return wrapper
    return decorator
```

**验收标准**:
- [ ] `get_redis()` 可连接 Redis
- [ ] `@cached` 装饰器正确缓存/过期
- [ ] 单元测试覆盖缓存命中/未命中/过期

**预估工时**: 1h | **依赖**: T-0.2

---

#### T-0.7: 认证模块移植（JWT + bcrypt + OTP）

**输入**: 原 `auth/auth.go`（JWT HS256, bcrypt, TOTP）
**输出文件**: `src/auth/jwt.py`, `src/auth/password.py`, `src/auth/otp.py`, `src/auth/dependencies.py`

```python
# src/auth/jwt.py - JWT 签发/验证
# src/auth/password.py - bcrypt hash/verify
# src/auth/otp.py - pyotp TOTP setup/verify
# src/auth/dependencies.py - get_current_user FastAPI Depends
```

**关键映射**:
| Go 函数 | Python 函数 | 库 |
|---------|------------|-----|
| `auth.GenerateJWT()` | `create_access_token()` | python-jose |
| `auth.ValidateJWT()` | `verify_token()` | python-jose |
| `auth.HashPassword()` | `hash_password()` | passlib[bcrypt] |
| `auth.CheckPassword()` | `verify_password()` | passlib[bcrypt] |
| `auth.GenerateOTPSecret()` | `generate_otp_secret()` | pyotp |
| `auth.ValidateOTP()` | `verify_otp()` | pyotp |
| `auth.IsTokenBlacklisted()` | Redis SET 实现 | redis |

**验收标准**:
- [ ] JWT 签发/验证/过期测试通过
- [ ] bcrypt 哈希/验证测试通过
- [ ] OTP 生成/验证测试通过
- [ ] Token 黑名单（Redis）测试通过
- [ ] `get_current_user` 依赖注入测试通过

**预估工时**: 3h | **依赖**: T-0.3, T-0.6

---

#### T-0.8: 加密模块移植（RSA + AES）

**输入**: 原 `crypto/` 目录
**输出文件**: `src/crypto/rsa.py`, `src/crypto/encryption.py`

**关键映射**:
| Go 函数 | Python 函数 | 库 |
|---------|------------|-----|
| `crypto.GenerateRSAKeyPair()` | `generate_rsa_keypair()` | cryptography |
| `crypto.EncryptWithPublicKey()` | `rsa_encrypt()` | cryptography |
| `crypto.DecryptWithPrivateKey()` | `rsa_decrypt()` | cryptography |
| `crypto.AESEncrypt()` | `aes_encrypt()` | cryptography |
| `crypto.AESDecrypt()` | `aes_decrypt()` | cryptography |

**验收标准**:
- [ ] RSA 密钥对生成/加密/解密测试通过
- [ ] AES-256-GCM 加密/解密测试通过
- [ ] 混合加密（RSA 包裹 AES 数据密钥）测试通过
- [ ] 与前端 `lib/crypto.ts` 的 WebCrypto 兼容

**预估工时**: 2h | **依赖**: T-0.1

---

#### T-0.9: 日志 + Telegram 通知模块

**输入**: 原 `logger/` 目录
**输出文件**: `src/utils/logger.py`, `src/utils/telegram.py`

```python
# src/utils/logger.py - loguru 替代 logrus
from loguru import logger
# 配置: 控制台 + 文件轮转 + 可选 Telegram sink

# src/utils/telegram.py - 异步 Telegram 通知
# 复用原有逻辑: 缓冲通道(asyncio.Queue) + 异步发送 + 3次重试
```

**验收标准**:
- [ ] `logger.info/warning/error` 输出到控制台
- [ ] Telegram 通知异步发送测试通过
- [ ] 消息格式与原系统一致（emoji + level + message）

**预估工时**: 2h | **依赖**: T-0.2

---

#### T-0.10: FastAPI 应用骨架 + 健康检查

**输入**: 原 `api/server.go` 路由结构
**输出文件**: `src/api/app.py`, `src/api/middleware.py`, `src/main.py`

```python
# src/api/app.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: DB engine, Redis, LangGraph checkpointer
    yield
    # shutdown: close connections

app = FastAPI(title="NoFx v2", lifespan=lifespan)
```

```python
# src/main.py
import uvicorn
from src.api.app import app

if __name__ == "__main__":
    uvicorn.run("src.main:app", host="0.0.0.0", port=8000, reload=True)
```

**验收标准**:
- [ ] `uvicorn src.main:app` 启动无报错
- [ ] `GET /api/health` 返回 `{"status": "ok"}`
- [ ] CORS 中间件配置正确
- [ ] 错误处理中间件捕获未处理异常

**预估工时**: 1.5h | **依赖**: T-0.3, T-0.6

---

#### T-0.11: Docker Compose 编排

**输入**: 无
**输出文件**: `docker-compose.yml`, `Dockerfile`

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["8000:8000"]
    depends_on: [postgres, redis]
    env_file: .env
  postgres:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: nofx_v2
      POSTGRES_USER: nofx
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**验收标准**:
- [ ] `docker compose up` 启动所有服务
- [ ] 应用可连接 PostgreSQL 和 Redis
- [ ] 健康检查通过

**预估工时**: 1h | **依赖**: T-0.10

---

#### T-0.12: CRUD 操作层

**输入**: T-0.4 的模型 + 原 `config/database.go` 的操作
**输出文件**: `src/db/crud.py`

**需实现的 CRUD 函数（1:1 映射原 Go 代码）**:

```python
# 用户
async def create_user(db, email, password_hash) -> User
async def get_user_by_email(db, email) -> User | None
async def get_user_by_id(db, user_id) -> User | None
async def update_user_otp(db, user_id, otp_secret, otp_verified) -> None

# AI 模型
async def get_ai_models(db, user_id) -> list[AIModel]
async def create_ai_model(db, user_id, name, provider, ...) -> AIModel
async def update_ai_model(db, model_id, **kwargs) -> AIModel

# 券商（替代 exchange）
async def get_brokers(db, user_id) -> list[Broker]
async def create_broker(db, user_id, name, ...) -> Broker
async def update_broker(db, broker_id, **kwargs) -> Broker

# Trader
async def get_traders(db, user_id) -> list[Trader]
async def create_trader(db, user_id, ...) -> Trader
async def update_trader(db, trader_id, **kwargs) -> Trader
async def delete_trader(db, trader_id) -> None
async def update_trader_status(db, trader_id, is_running) -> None

# 决策日志
async def log_decision(db, trader_id, ...) -> DecisionLog
async def get_latest_decisions(db, trader_id, limit) -> list[DecisionLog]

# 系统配置
async def get_system_config(db, key) -> str | None
async def set_system_config(db, key, value) -> None
```

**验收标准**:
- [ ] 所有 CRUD 函数有 async 实现
- [ ] 集成测试使用测试数据库
- [ ] 敏感字段自动加密/解密

**预估工时**: 4h | **依赖**: T-0.4, T-0.5, T-0.8

---

### Phase 0 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| Python 项目可 `uv sync` | CI |
| PostgreSQL + Redis 连接正常 | `docker compose up` |
| 6 张数据库表已创建 | `alembic upgrade head` |
| JWT/OTP/加密模块测试通过 | `pytest tests/unit/test_auth.py` |
| FastAPI 健康检查 | `curl localhost:8000/api/health` |
| CRUD 层完整 | `pytest tests/integration/` |

---

## Phase 1: 数据层（港A股行情接入）

### 里程碑 M1: 可获取港A股实时行情 + 技术指标，Redis 缓存生效

---

#### T-1.1: 港A股数据类型定义

**输入**: REFACTOR_PLAN.md 第 6.1 节 + 原 `market/types.go`
**输出文件**: `src/market/types.py`

```python
# 需定义的数据类（全部使用 @dataclass 或 Pydantic BaseModel）:
# - StockQuote (实时行情, 20+ 字段)
# - StockKline (K线数据, 8 字段)
# - FundamentalData (基本面, 12 字段)
# - SectorFlow (板块资金流向)
# - NorthboundFlow (北向资金)
# - MarketOverview (大盘概览)
```

**验收标准**:
- [ ] 所有数据类有完整类型注解
- [ ] `StockQuote` 包含 `limit_up/limit_down`（仅A股）
- [ ] `StockQuote` 包含 `is_suspended`（停牌标志）
- [ ] 序列化/反序列化测试通过

**预估工时**: 1.5h | **依赖**: T-0.2

---

#### T-1.2: MarketDataProvider 抽象接口

**输入**: 原 `market/` 的功能需求
**输出文件**: `src/market/provider.py`

```python
from abc import ABC, abstractmethod

class MarketDataProvider(ABC):
    @abstractmethod
    async def get_quote(self, symbol: str) -> StockQuote: ...

    @abstractmethod
    async def get_kline(self, symbol: str, period: str, count: int) -> list[StockKline]: ...

    @abstractmethod
    async def get_fundamental(self, symbol: str) -> FundamentalData: ...

    @abstractmethod
    async def get_sector_flow(self) -> list[SectorFlow]: ...

    @abstractmethod
    async def get_northbound_flow(self) -> NorthboundFlow: ...

    @abstractmethod
    async def search_stock(self, keyword: str) -> list[dict]: ...

    @abstractmethod
    async def is_trading_time(self, market: Market) -> bool: ...

    # 辅助方法
    def detect_market(self, symbol: str) -> Market: ...
    def detect_board(self, symbol: str) -> StockBoard | None: ...
    def get_price_limit(self, symbol: str, prev_close: float) -> tuple[float, float]: ...
```

**验收标准**:
- [ ] 接口定义完整，覆盖所有数据获取需求
- [ ] `detect_market()` 正确识别 A股/港股代码
- [ ] `detect_board()` 正确识别主板/创业板/科创板/北交所
- [ ] `get_price_limit()` 计算涨跌停价格正确

**预估工时**: 1.5h | **依赖**: T-1.1

---

#### T-1.3: AKShare Provider 实现 - 实时行情

**输入**: AKShare API 文档 + `library-research.md`
**输出文件**: `src/market/akshare_provider.py`（第一部分）

```python
class AKShareProvider(MarketDataProvider):
    async def get_quote(self, symbol: str) -> StockQuote:
        """
        A股: ak.stock_zh_a_spot_em() -> 筛选 symbol
        港股: ak.stock_hk_spot_em() -> 筛选 symbol
        """

    async def search_stock(self, keyword: str) -> list[dict]:
        """股票搜索（代码/名称/拼音首字母）"""
```

**验收标准**:
- [ ] A股实时行情获取测试通过（600519 贵州茅台）
- [ ] 港股实时行情获取测试通过（00700 腾讯）
- [ ] 停牌股票正确标记 `is_suspended=True`
- [ ] 涨跌停价格计算正确
- [ ] 异常处理：网络超时、数据为空

**预估工时**: 3h | **依赖**: T-1.2

---

#### T-1.4: AKShare Provider 实现 - K线数据

**输出文件**: `src/market/akshare_provider.py`（追加）

```python
    async def get_kline(self, symbol: str, period: str = "daily", count: int = 120) -> list[StockKline]:
        """
        A股: ak.stock_zh_a_hist(symbol, period, adjust="qfq")
        港股: ak.stock_hk_hist(symbol, period, adjust="qfq")
        period: "daily" | "weekly" | "monthly"
        """
```

**验收标准**:
- [ ] 日K/周K/月K 数据获取正确
- [ ] 前复权数据正确
- [ ] 返回数据按时间升序排列
- [ ] `count` 参数正确截取最近 N 条

**预估工时**: 2h | **依赖**: T-1.3

---

#### T-1.5: AKShare Provider 实现 - 基本面数据

**输出文件**: `src/market/akshare_provider.py`（追加）

```python
    async def get_fundamental(self, symbol: str) -> FundamentalData:
        """
        PE/PB/PS: ak.stock_zh_a_spot_em() 中包含
        ROE/营收增长/利润增长: ak.stock_financial_analysis_indicator()
        股息率: ak.stock_a_indicator_lg()
        行业/概念: ak.stock_board_industry_name_em()
        """
```

**验收标准**:
- [ ] PE/PB/ROE 等指标获取正确
- [ ] 行业分类获取正确
- [ ] 数据缺失时返回默认值（不抛异常）

**预估工时**: 2h | **依赖**: T-1.3

---

#### T-1.6: AKShare Provider 实现 - 资金流向

**输出文件**: `src/market/akshare_provider.py`（追加）

```python
    async def get_sector_flow(self) -> list[SectorFlow]:
        """板块资金流向: ak.stock_sector_fund_flow_rank()"""

    async def get_northbound_flow(self) -> NorthboundFlow:
        """北向资金: ak.stock_hsgt_north_net_flow_in_em()"""
```

**验收标准**:
- [ ] 板块资金流向排名正确
- [ ] 北向资金净流入数据正确
- [ ] 非交易时间返回最近交易日数据

**预估工时**: 1.5h | **依赖**: T-1.3

---

#### T-1.7: 技术指标计算模块

**输入**: 原 `market/data.go` 的指标计算 + TA-Lib API
**输出文件**: `src/market/indicators.py`

```python
import talib
import numpy as np

def calculate_indicators(klines: list[StockKline]) -> dict:
    """计算全套技术指标"""
    close = np.array([k.close for k in klines])
    high = np.array([k.high for k in klines])
    low = np.array([k.low for k in klines])
    volume = np.array([k.volume for k in klines], dtype=float)

    macd, signal, hist = talib.MACD(close)
    return {
        "ema_20": talib.EMA(close, 20)[-1],
        "ema_50": talib.EMA(close, 50)[-1],
        "macd": macd[-1],
        "macd_signal": signal[-1],
        "macd_hist": hist[-1],
        "rsi_7": talib.RSI(close, 7)[-1],
        "rsi_14": talib.RSI(close, 14)[-1],
        "bb_upper": talib.BBANDS(close)[0][-1],
        "bb_middle": talib.BBANDS(close)[1][-1],
        "bb_lower": talib.BBANDS(close)[2][-1],
        "atr_14": talib.ATR(high, low, close, 14)[-1],
        "volume_ma_20": talib.SMA(volume, 20)[-1],
        "volume_ratio": volume[-1] / talib.SMA(volume, 20)[-1] if talib.SMA(volume, 20)[-1] > 0 else 1.0,
        "obv": talib.OBV(close, volume)[-1],
    }
```

**验收标准**:
- [ ] 所有 14 个指标计算正确（与 TradingView 对比）
- [ ] 输入数据不足时优雅降级（返回 NaN 或 None）
- [ ] 性能：120 条 K 线计算 < 10ms
- [ ] 单元测试覆盖率 ≥ 90%

**预估工时**: 2h | **依赖**: T-1.1

---

#### T-1.8: Redis 行情缓存集成

**输入**: T-0.6 的缓存工具
**输出文件**: `src/market/cache.py`（扩展）

```python
# 缓存策略:
# - 实时行情: TTL 10s（交易时间）/ 300s（非交易时间）
# - K线数据: TTL 60s（日K）/ 300s（周K/月K）
# - 基本面: TTL 3600s（1小时）
# - 板块资金: TTL 60s
# - 北向资金: TTL 30s
```

**验收标准**:
- [ ] 缓存命中时不调用 AKShare
- [ ] TTL 根据交易时间动态调整
- [ ] 缓存失效后自动刷新
- [ ] Redis 不可用时降级为直接调用

**预估工时**: 1.5h | **依赖**: T-0.6, T-1.3

---

#### T-1.9: 数据源健康检查 + 降级策略

**输出文件**: `src/market/akshare_provider.py`（追加健康检查方法）

```python
    async def health_check(self) -> dict:
        """检查数据源可用性"""
        results = {}
        try:
            await self.get_quote("600519")
            results["a_share_quote"] = "ok"
        except Exception as e:
            results["a_share_quote"] = f"error: {e}"
        # ... 类似检查港股、K线等
        return results
```

**验收标准**:
- [ ] 健康检查覆盖所有数据源
- [ ] 单个数据源失败不影响其他
- [ ] 降级策略：AKShare 失败 → 返回缓存数据 → 返回空数据

**预估工时**: 1h | **依赖**: T-1.3, T-1.8

---

### Phase 1 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| A股实时行情获取 | `pytest tests/integration/test_akshare_provider.py::test_a_share_quote` |
| 港股实时行情获取 | `pytest tests/integration/test_akshare_provider.py::test_hk_quote` |
| K线数据获取 | `pytest tests/integration/test_akshare_provider.py::test_kline` |
| 14 个技术指标计算 | `pytest tests/unit/test_indicators.py` |
| Redis 缓存生效 | `pytest tests/integration/test_cache.py` |
| 基本面数据获取 | `pytest tests/integration/test_akshare_provider.py::test_fundamental` |
| 数据源健康检查 | `curl localhost:8000/api/market/health` |

---

## Phase 2: LangGraph 多智能体系统

### 里程碑 M2: 完整决策流程可运行（研究→分析→风控→审批），InMemorySaver 测试通过

---

#### T-2.1: TradingState 全局状态定义

**输入**: REFACTOR_PLAN.md 第 5.1 节
**输出文件**: `src/agents/state.py`

完整实现 `TradingState` TypedDict，包含所有中间状态类型：
- `MarketRegime`, `StockAnalysis`, `RiskAssessment`, `TradeOrder`
- 使用 `Annotated[list, operator.add]` 作为 `stock_analyses` 的 reducer
- 使用 `add_messages` 作为 `messages` 的 reducer

**验收标准**:
- [ ] 所有 TypedDict 有完整类型注解
- [ ] Reducer 正确配置（stock_analyses 追加，messages 合并）
- [ ] 可被 StateGraph 接受
- [ ] 单元测试验证 reducer 行为

**预估工时**: 1.5h | **依赖**: T-0.1

---

#### T-2.2: LangGraph 工具定义 - 行情数据工具

**输入**: T-1.3 的 AKShare Provider
**输出文件**: `src/tools/market_data.py`

```python
from langchain_core.tools import tool

@tool
def get_stock_quote(symbol: str) -> dict:
    """获取股票实时行情（支持A股和港股）"""

@tool
def get_stock_kline(symbol: str, period: str = "daily", count: int = 120) -> list[dict]:
    """获取K线历史数据"""

@tool
def get_market_overview() -> dict:
    """获取大盘概览（上证/深证/恒生指数）"""

@tool
def get_sector_heatmap() -> list[dict]:
    """获取板块资金流向热力图"""

@tool
def get_northbound_flow() -> dict:
    """获取北向资金净流入数据"""

@tool
def search_stocks(keyword: str) -> list[dict]:
    """搜索股票（代码/名称/拼音）"""
```

**验收标准**:
- [ ] 所有工具有 docstring（LLM 可读）
- [ ] 返回值为 JSON 可序列化的 dict/list
- [ ] 异常处理：返回错误信息而非抛异常

**预估工时**: 2h | **依赖**: T-1.3, T-2.1

---

#### T-2.3: LangGraph 工具定义 - 技术指标工具

**输出文件**: `src/tools/technical_indicators.py`

```python
@tool
def calculate_technical_indicators(symbol: str, period: str = "daily") -> dict:
    """计算技术指标：MACD, RSI, EMA, 布林带, ATR, OBV"""

@tool
def detect_chart_patterns(symbol: str) -> list[dict]:
    """检测图表形态：头肩顶/底、双顶/底、三角形等"""

@tool
def calculate_support_resistance(symbol: str) -> dict:
    """计算支撑位和阻力位"""
```

**验收标准**:
- [ ] 指标计算结果与 T-1.7 一致
- [ ] 图表形态检测至少支持 5 种
- [ ] 支撑/阻力位计算合理

**预估工时**: 2h | **依赖**: T-1.7

---

#### T-2.4: LangGraph 工具定义 - 基本面数据工具

**输出文件**: `src/tools/fundamental_data.py`

```python
@tool
def get_financial_summary(symbol: str) -> dict:
    """获取财务摘要：PE/PB/ROE/营收增长/利润增长"""

@tool
def get_valuation_comparison(symbol: str) -> dict:
    """获取估值对比：与行业均值、历史分位数对比"""

@tool
def get_shareholder_changes(symbol: str) -> dict:
    """获取股东变动：十大股东、机构持仓变化"""
```

**验收标准**:
- [ ] 财务数据获取正确
- [ ] 估值对比包含行业均值
- [ ] 数据缺失时返回 "N/A" 而非报错

**预估工时**: 2h | **依赖**: T-1.5

---

#### T-2.5: LangGraph 工具定义 - 舆情工具

**输出文件**: `src/tools/news_sentiment.py`

```python
@tool
def get_stock_news(symbol: str, limit: int = 10) -> list[dict]:
    """获取个股相关新闻"""

@tool
def get_market_sentiment() -> dict:
    """获取市场整体情绪指标"""
```

**验收标准**:
- [ ] 新闻获取包含标题、时间、来源
- [ ] 市场情绪包含恐贪指数等

**预估工时**: 1.5h | **依赖**: T-1.3

---

#### T-2.6: LangGraph 工具定义 - 风险计算工具

**输出文件**: `src/tools/risk_calculator.py`

```python
@tool
def calculate_position_risk(
    symbol: str, quantity: int, entry_price: float,
    stop_loss: float, portfolio_value: float
) -> dict:
    """计算仓位风险：风险金额、仓位占比、风险回报比"""

@tool
def check_portfolio_risk(positions: list[dict], cash: float) -> dict:
    """检查组合风险：总仓位、集中度、相关性"""

@tool
def check_trading_rules(symbol: str, action: str, quantity: int) -> dict:
    """检查交易规则：T+1、涨跌停、最小单位"""
```

**验收标准**:
- [ ] 仓位风险计算正确
- [ ] A股 T+1 规则检查正确
- [ ] 涨跌停检查正确
- [ ] A股 100 股整数倍检查正确

**预估工时**: 2h | **依赖**: T-0.2

---

#### T-2.7: 提示词模板编写（6 个智能体）

**输入**: 原 `prompts/` 目录的策略思路 + 港A股特性
**输出文件**: `prompts/supervisor.md`, `prompts/technical_analyst.md`, `prompts/fundamental_analyst.md`, `prompts/sentiment_analyst.md`, `prompts/risk_manager.md`, `prompts/synthesizer.md`

每个提示词需包含：
1. 角色定义
2. 港A股市场特性说明
3. 分析框架
4. 输出格式要求
5. 约束条件

**验收标准**:
- [ ] 6 个提示词文件，每个 200-500 字
- [ ] 明确提及港A股特殊规则（T+1、涨跌停等）
- [ ] 输出格式与 TradingState 字段对应

**预估工时**: 4h | **依赖**: T-2.1

---

#### T-2.8: Research Agent 实现

**输出文件**: `src/agents/research_agent.py`

```python
from langgraph.prebuilt import create_react_agent

research_agent = create_react_agent(
    model="openai:gpt-4o",
    tools=[get_market_overview, get_sector_heatmap, get_northbound_flow, get_market_sentiment],
    name="research_agent",
    prompt=load_prompt("supervisor"),  # 市场环境研究
)
```

**职责**: 分析当前市场环境，输出 `MarketRegime`（趋势、波动率、热点板块）

**验收标准**:
- [ ] 可独立运行并返回 MarketRegime
- [ ] 正确调用行情工具
- [ ] 输出格式符合 TradingState 定义

**预估工时**: 2h | **依赖**: T-2.2, T-2.7

---

#### T-2.9: Technical Analyst 实现

**输出文件**: `src/agents/technical_analyst.py`

**职责**: 对单只股票进行技术分析，输出 `StockAnalysis.technical_score`（-100 ~ +100）

**验收标准**:
- [ ] 正确调用 K 线 + 技术指标工具
- [ ] 评分范围 -100 ~ +100
- [ ] 包含详细 reasoning

**预估工时**: 2h | **依赖**: T-2.3, T-2.7

---

#### T-2.10: Fundamental Analyst 实现

**输出文件**: `src/agents/fundamental_analyst.py`

**职责**: 对单只股票进行基本面分析，输出 `StockAnalysis.fundamental_score`

**验收标准**:
- [ ] 正确调用基本面数据工具
- [ ] 评分考虑 PE/PB/ROE/增长率
- [ ] 包含估值对比分析

**预估工时**: 2h | **依赖**: T-2.4, T-2.7

---

#### T-2.11: Sentiment Analyst 实现

**输出文件**: `src/agents/sentiment_analyst.py`

**职责**: 对单只股票进行舆情分析，输出 `StockAnalysis.sentiment_score`

**验收标准**:
- [ ] 正确调用新闻/舆情工具
- [ ] 评分反映市场情绪
- [ ] 包含关键新闻摘要

**预估工时**: 1.5h | **依赖**: T-2.5, T-2.7

---

#### T-2.12: Synthesizer 节点实现

**输出文件**: `src/agents/synthesizer.py`

```python
def synthesizer_node(state: TradingState) -> dict:
    """综合三维评分，生成交易信号"""
    analyses = state["stock_analyses"]
    # 按 symbol 分组，计算加权综合评分
    # 技术:基本面:舆情 = 0.4:0.35:0.25
    # 生成 TradeOrder 列表
```

**职责**: 综合技术/基本面/舆情三维评分，生成初步交易指令

**验收标准**:
- [ ] 正确聚合三个分析师的评分
- [ ] 加权计算综合评分
- [ ] 生成合理的 TradeOrder（含 quantity、stop_loss、take_profit）
- [ ] A股 quantity 为 100 整数倍

**预估工时**: 3h | **依赖**: T-2.1

---

#### T-2.13: Risk Manager 节点实现

**输入**: REFACTOR_PLAN.md 第 5.4 节
**输出文件**: `src/agents/risk_manager.py`

**风控规则清单**:
1. A股 T+1 检查（今日买入不可卖出）
2. 涨跌停检查（接近涨跌停板警告）
3. 单只仓位上限（≤ 总资产 20%）
4. 总仓位上限（≤ 80%）
5. 大额交易审批（单笔 > 总资产 5% → HITL）
6. 停牌股票过滤
7. 交易时间检查
8. 最小交易金额检查

**验收标准**:
- [ ] 8 条风控规则全部实现
- [ ] T+1 限制测试通过
- [ ] 涨跌停检查测试通过
- [ ] 大额交易正确触发 `requires_human_approval`
- [ ] 单元测试覆盖率 ≥ 95%

**预估工时**: 4h | **依赖**: T-2.6

---

#### T-2.14: Human-in-the-Loop 审批节点

**输出文件**: `src/agents/graph.py`（HITL 部分）

```python
from langgraph.types import interrupt

def human_review_node(state: TradingState) -> dict:
    orders_summary = format_orders_for_review(state["trade_orders"])
    decision = interrupt(f"请审批以下交易:\n{orders_summary}")
    return {
        "human_decision": decision,
        "requires_human_approval": False,
    }
```

**验收标准**:
- [ ] `interrupt()` 正确暂停图执行
- [ ] `Command(resume="approved")` 正确恢复
- [ ] 拒绝后正确跳过执行
- [ ] 审批信息包含完整交易详情

**预估工时**: 2h | **依赖**: T-2.13

---

#### T-2.15: Supervisor 节点 + 路由逻辑

**输出文件**: `src/agents/supervisor.py`

```python
def supervisor_node(state: TradingState) -> dict:
    """初始化分析周期，设置 current_phase"""
    return {
        "current_phase": "research",
        "messages": [SystemMessage(content="开始新一轮分析周期")],
    }
```

**验收标准**:
- [ ] 正确初始化分析周期
- [ ] 设置 current_phase

**预估工时**: 1h | **依赖**: T-2.1

---

#### T-2.16: 主图构建 + 编译

**输入**: REFACTOR_PLAN.md 第 5.3 节
**输出文件**: `src/agents/graph.py`

```python
def build_trading_graph() -> StateGraph:
    builder = StateGraph(TradingState)
    # 注册所有节点
    # 配置边和条件路由
    # fan_out_analysts (Send API)
    # fan-in 等待
    # 风控路由
    # HITL 路由
    return builder

def compile_graph(checkpointer=None):
    builder = build_trading_graph()
    return builder.compile(
        checkpointer=checkpointer or InMemorySaver(),
        interrupt_before=["human_review"],
    )
```

**验收标准**:
- [ ] 图编译无报错
- [ ] `graph.get_graph().draw_mermaid()` 生成正确的流程图
- [ ] 节点连接关系正确：supervisor → research → [3 analysts] → synthesizer → risk_manager → [human_review | execution | end]

**预估工时**: 3h | **依赖**: T-2.8 ~ T-2.15

---

#### T-2.17: 集成测试 - 完整决策流程

**输出文件**: `tests/integration/test_graph_flow.py`

```python
def test_full_trading_cycle():
    """完整流程：supervisor → research → analysts → synthesizer → risk → execution"""

def test_hitl_large_trade():
    """大额交易触发人工审批"""

def test_empty_watchlist():
    """空关注列表应优雅处理"""

def test_all_stocks_suspended():
    """所有股票停牌时应返回空决策"""

def test_risk_manager_blocks_trade():
    """风控拒绝交易"""
```

**验收标准**:
- [ ] 5 个集成测试全部通过
- [ ] 使用 InMemorySaver
- [ ] Mock 外部 API 调用（AKShare、LLM）

**预估工时**: 4h | **依赖**: T-2.16

---

#### T-2.18: PostgresSaver 集成

**输出文件**: `src/agents/graph.py`（追加）

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async def get_production_graph(database_url: str):
    checkpointer = AsyncPostgresSaver.from_conn_string(database_url)
    await checkpointer.setup()  # 自动创建 checkpoints 表
    return compile_graph(checkpointer=checkpointer)
```

**验收标准**:
- [ ] PostgresSaver 正确创建 checkpoints 表
- [ ] 图状态可持久化和恢复
- [ ] thread_id 隔离正确

**预估工时**: 2h | **依赖**: T-2.16, T-0.3

---

### Phase 2 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| TradingState 定义 | `pytest tests/unit/test_state.py` |
| 6 个 LangGraph 工具 | `pytest tests/unit/test_tools.py` |
| 6 个提示词模板 | 人工审查 |
| 7 个智能体节点 | 各自单元测试 |
| 主图编译 | `graph.get_graph().draw_mermaid()` |
| 完整流程集成测试 | `pytest tests/integration/test_graph_flow.py` |
| PostgresSaver 持久化 | `pytest tests/integration/test_checkpointer.py` |

---

## Phase 3: 交易执行层

### 里程碑 M3: SimulatorBroker 可执行模拟交易，A股 T+1 规则正确

---

#### T-3.1: 交易类型定义

**输入**: REFACTOR_PLAN.md 第 7.1 节
**输出文件**: `src/broker/types.py`

```python
# 需定义:
# - OrderSide (BUY/SELL)
# - OrderType (MARKET/LIMIT/LIMIT_IF_TOUCHED)
# - OrderStatus (PENDING/PARTIAL/FILLED/CANCELLED/REJECTED)
# - OrderRequest (symbol, market, side, quantity, order_type, limit_price)
# - OrderResult (order_id, symbol, status, filled_quantity, filled_price, commission)
# - PortfolioPosition (symbol, name, market, quantity, available_quantity, cost_price, ...)
# - BrokerError (自定义异常)
```

**验收标准**:
- [ ] 所有枚举和数据类有完整类型注解
- [ ] `OrderRequest` 包含 `market` 字段
- [ ] `PortfolioPosition` 包含 `available_quantity`（T+1）

**预估工时**: 1h | **依赖**: T-0.2

---

#### T-3.2: BrokerInterface 抽象接口

**输入**: REFACTOR_PLAN.md 第 7.1 节 + 原 `trader/interface.go`
**输出文件**: `src/broker/interface.py`

```python
class BrokerInterface(ABC):
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

**验收标准**:
- [ ] 接口覆盖所有交易操作
- [ ] 与原 Go `Trader` 接口功能对等（去除杠杆/保证金相关）

**预估工时**: 1h | **依赖**: T-3.1

---

#### T-3.3: SimulatorBroker 实现

**输入**: REFACTOR_PLAN.md 第 7.3 节
**输出文件**: `src/broker/simulator.py`

```python
class SimulatorBroker(BrokerInterface):
    def __init__(self, initial_cash: float = 1_000_000):
        self.cash = initial_cash
        self.positions: dict[str, PortfolioPosition] = {}
        self.orders: list[OrderResult] = []
        self.today_buys: set[str] = set()  # T+1 追踪
        self._order_counter = 0

    async def place_order(self, order: OrderRequest) -> OrderResult:
        # 1. A股 T+1 检查
        # 2. A股 100 股整数倍检查
        # 3. 涨跌停检查
        # 4. 资金充足检查
        # 5. 模拟成交（使用实时价格）
        # 6. 计算佣金 + 印花税
        # 7. 更新持仓和现金
```

**验收标准**:
- [ ] A股 T+1 限制正确（今日买入不可卖出）
- [ ] A股 100 股整数倍检查正确
- [ ] 佣金计算正确（万三 + 印花税千一卖出）
- [ ] 资金不足时拒绝下单
- [ ] 持仓更新正确（买入增加、卖出减少）
- [ ] `available_quantity` 正确反映 T+1
- [ ] 次日 `reset_daily_state()` 后可卖出
- [ ] 单元测试覆盖率 ≥ 95%

**预估工时**: 5h | **依赖**: T-3.2, T-1.3

---

#### T-3.4: FutuBroker 实现

**输入**: `library-research.md` 富途 API 部分
**输出文件**: `src/broker/futu_broker.py`

```python
from futu import OpenSecTradeContext, TrdSide, OrderType as FutuOrderType

class FutuBroker(BrokerInterface):
    def __init__(self, host: str, port: int, trd_env: str = "SIMULATE"):
        self.ctx = OpenSecTradeContext(host=host, port=port)
        # ...
```

**验收标准**:
- [ ] 模拟环境下单测试通过
- [ ] 持仓查询正确
- [ ] 余额查询正确
- [ ] 订单状态查询正确
- [ ] 连接断开时优雅处理

**预估工时**: 4h | **依赖**: T-3.2

---

#### T-3.5: LongbridgeBroker 实现

**输入**: `library-research.md` 长桥 API 部分
**输出文件**: `src/broker/longbridge_broker.py`

**验收标准**:
- [ ] 模拟环境下单测试通过
- [ ] 与 FutuBroker 接口一致

**预估工时**: 3h | **依赖**: T-3.2

---

#### T-3.6: Execution Agent 实现

**输出文件**: `src/agents/execution_agent.py`

```python
def execution_agent(state: TradingState) -> dict:
    """执行交易指令"""
    orders = state["trade_orders"]
    broker = get_broker(state["trader_id"])
    results = []
    for order in orders:
        try:
            result = await broker.place_order(order_to_request(order))
            results.append({"symbol": order["symbol"], "status": "success", ...})
        except BrokerError as e:
            results.append({"symbol": order["symbol"], "status": "failed", "error": str(e)})
    return {"execution_results": results}
```

**验收标准**:
- [ ] 正确调用 BrokerInterface
- [ ] 单笔失败不影响其他
- [ ] 执行结果记录完整

**预估工时**: 2h | **依赖**: T-3.3, T-2.16

---

#### T-3.7: LangGraph 交易工具集成

**输出文件**: `src/tools/broker_api.py`

```python
@tool
def place_trade(symbol: str, action: str, quantity: int, price_type: str = "market") -> dict:
    """执行交易（仅在 Execution Agent 中使用）"""

@tool
def get_portfolio_positions() -> list[dict]:
    """获取当前持仓"""

@tool
def get_account_balance() -> dict:
    """获取账户余额"""
```

**验收标准**:
- [ ] 工具正确调用 BrokerInterface
- [ ] 配置注入（通过 RunnableConfig 传递 broker 实例）

**预估工时**: 1.5h | **依赖**: T-3.3

---

#### T-3.8: 条件单实现（软件层止损/止盈）

**输出文件**: `src/broker/conditional_orders.py`

```python
class ConditionalOrderManager:
    """软件层条件单管理器（港A股无原生条件单）"""

    def __init__(self, broker: BrokerInterface, check_interval: int = 30):
        self.broker = broker
        self.orders: list[ConditionalOrder] = []
        self._running = False

    async def add_stop_loss(self, symbol: str, trigger_price: float, quantity: int): ...
    async def add_take_profit(self, symbol: str, trigger_price: float, quantity: int): ...
    async def start_monitoring(self): ...
    async def stop_monitoring(self): ...
```

**验收标准**:
- [ ] 止损触发正确（价格跌破触发价）
- [ ] 止盈触发正确（价格涨破触发价）
- [ ] 监控间隔可配置
- [ ] 非交易时间暂停监控

**预估工时**: 3h | **依赖**: T-3.3, T-1.3

---

#### T-3.9: 定时调度器（APScheduler）

**输出文件**: `src/scheduler/trading_scheduler.py`

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

class TradingScheduler:
    def __init__(self):
        self.scheduler = AsyncIOScheduler()
        self.active_traders: dict[str, str] = {}  # trader_id -> job_id

    def start_trader(self, trader_id: str, interval_minutes: int, graph, config: dict):
        """启动 trader 定时任务"""
        job = self.scheduler.add_job(
            self._run_cycle,
            trigger="interval",
            minutes=interval_minutes,
            args=[trader_id, graph, config],
            id=f"trader_{trader_id}",
        )
        self.active_traders[trader_id] = job.id

    def stop_trader(self, trader_id: str):
        """停止 trader"""
        job_id = self.active_traders.pop(trader_id, None)
        if job_id:
            self.scheduler.remove_job(job_id)

    async def _run_cycle(self, trader_id: str, graph, config: dict):
        """执行一次分析周期"""
        # 1. 检查是否在交易时间
        # 2. 构建输入状态
        # 3. graph.ainvoke(input, config)
        # 4. 记录决策日志
```

**验收标准**:
- [ ] Trader 可按间隔定时运行
- [ ] 非交易时间自动跳过
- [ ] 启动/停止正确
- [ ] 多 Trader 并发运行无冲突

**预估工时**: 3h | **依赖**: T-2.16, T-3.6

---

### Phase 3 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| SimulatorBroker | `pytest tests/unit/test_broker_simulator.py` (≥95% 覆盖) |
| FutuBroker | `pytest tests/integration/test_futu_broker.py` (模拟环境) |
| A股 T+1 规则 | 专项测试用例 |
| 条件单管理器 | `pytest tests/unit/test_conditional_orders.py` |
| 定时调度器 | `pytest tests/unit/test_scheduler.py` |
| Execution Agent | `pytest tests/integration/test_execution.py` |

---

## Phase 4: API 层 + 前端改造

### 里程碑 M4: 前后端联调通过，可在浏览器中创建 Trader 并查看决策

---

#### T-4.1: 认证路由（8 endpoints）

**输入**: 原 `api/server.go` 认证相关路由
**输出文件**: `src/api/routes/auth.py`

```python
# POST /api/register          - 用户注册
# POST /api/login              - 登录（返回 requires_otp）
# POST /api/verify-otp         - OTP 验证，返回 JWT
# POST /api/complete-registration - 完成注册（设置 OTP）
# POST /api/logout             - 登出（JWT 黑名单）
# POST /api/reset-password     - 重置密码
# POST /api/admin-login        - 管理员登录
# GET  /api/config             - 系统配置（公开）
```

**验收标准**:
- [ ] 8 个 endpoint 全部实现
- [ ] 请求/响应格式与原 Go API 兼容
- [ ] JWT 签发/验证正确
- [ ] OTP 流程正确
- [ ] 集成测试通过

**预估工时**: 4h | **依赖**: T-0.7, T-0.12

---

#### T-4.2: Trader 管理路由（12 endpoints）

**输入**: 原 `api/server.go` trader 相关路由
**输出文件**: `src/api/routes/traders.py`

```python
# GET    /api/my-traders           - 用户的 trader 列表
# GET    /api/traders              - 公开 trader 列表（排行榜）
# POST   /api/traders              - 创建 trader
# PUT    /api/traders/{id}         - 更新 trader
# DELETE /api/traders/{id}         - 删除 trader
# POST   /api/traders/{id}/start   - 启动 trader
# POST   /api/traders/{id}/stop    - 停止 trader
# PUT    /api/traders/{id}/prompt  - 更新自定义提示词
# GET    /api/traders/{id}/config  - 获取 trader 完整配置
# GET    /api/traders/{id}/public-config - 公开配置
# GET    /api/status               - trader 状态
# GET    /api/account              - 账户信息
```

**验收标准**:
- [ ] 12 个 endpoint 全部实现
- [ ] 创建 trader 时查询券商余额作为 initial_balance
- [ ] 启动/停止正确调用 TradingScheduler
- [ ] 权限检查：只能操作自己的 trader

**预估工时**: 6h | **依赖**: T-0.12, T-3.9

---

#### T-4.3: AI 模型配置路由（4 endpoints）

**输出文件**: `src/api/routes/models.py`

```python
# GET  /api/models              - 用户的 AI 模型配置
# PUT  /api/models              - 更新模型配置（加密传输）
# GET  /api/supported-models    - 系统支持的模型列表
```

**验收标准**:
- [ ] API Key 加密传输/存储
- [ ] 返回时不暴露完整 API Key

**预估工时**: 2h | **依赖**: T-0.12, T-0.8

---

#### T-4.4: 券商配置路由（4 endpoints）

**输出文件**: `src/api/routes/brokers.py`

```python
# GET  /api/brokers              - 用户的券商配置（替代 /api/exchanges）
# PUT  /api/brokers              - 更新券商配置（加密传输）
# GET  /api/supported-brokers    - 系统支持的券商列表
```

**验收标准**:
- [ ] 支持 futu / longbridge / simulator 三种券商
- [ ] 敏感字段加密传输/存储

**预估工时**: 2h | **依赖**: T-0.12, T-0.8

---

#### T-4.5: 行情数据路由（4 endpoints）

**输出文件**: `src/api/routes/market.py`

```python
# GET  /api/market/quote/{symbol}    - 实时行情
# GET  /api/market/kline/{symbol}    - K线数据
# GET  /api/market/search            - 股票搜索
# GET  /api/market/health            - 数据源健康检查
```

**验收标准**:
- [ ] 行情数据正确返回
- [ ] 搜索支持代码/名称/拼音
- [ ] 健康检查覆盖所有数据源

**预估工时**: 2h | **依赖**: T-1.3

---

#### T-4.6: 决策日志路由（4 endpoints）

**输出文件**: `src/api/routes/decisions.py`

```python
# GET  /api/decisions                - 所有决策日志
# GET  /api/decisions/latest         - 最近 N 条决策
# GET  /api/positions                - 当前持仓
# GET  /api/statistics               - 统计数据
```

**验收标准**:
- [ ] 决策日志从 PostgreSQL 读取（替代原 JSON 文件）
- [ ] 分页支持
- [ ] 统计数据计算正确

**预估工时**: 2h | **依赖**: T-0.12

---

#### T-4.7: 竞赛排行路由（3 endpoints）

**输出文件**: `src/api/routes/competition.py`

```python
# GET  /api/competition              - 竞赛数据
# GET  /api/top-traders              - Top 5 traders
# GET  /api/equity-history           - 权益曲线
# POST /api/equity-history-batch     - 批量权益曲线
```

**验收标准**:
- [ ] 排行榜按 PnL% 排序
- [ ] 权益曲线数据正确

**预估工时**: 2h | **依赖**: T-0.12

---

#### T-4.8: RSA 加密 + 系统配置路由

**输出文件**: `src/api/routes/crypto_keys.py`, `src/api/routes/system.py`

```python
# GET  /api/crypto/public-key        - RSA 公钥
# POST /api/crypto/decrypt           - 解密数据
# GET  /api/server-ip                - 服务器 IP
```

**验收标准**:
- [ ] 公钥格式与前端 WebCrypto 兼容
- [ ] 解密功能正确

**预估工时**: 1h | **依赖**: T-0.8

---

#### T-4.9: WebSocket 实时推送

**输出文件**: `src/api/routes/websocket.py`（新增）

```python
@app.websocket("/ws/trader/{trader_id}")
async def trader_websocket(websocket: WebSocket, trader_id: str):
    """实时推送 trader 状态、决策、持仓变化"""
    await websocket.accept()
    # 订阅 trader 事件
    # 推送: account_update, position_update, decision_update, execution_update
```

**验收标准**:
- [ ] WebSocket 连接/断开正确
- [ ] 实时推送 trader 状态变化
- [ ] 认证检查（JWT in query param）

**预估工时**: 3h | **依赖**: T-4.2

---

#### T-4.10: SSE 流式推送（LangGraph 分析过程）

**输出文件**: `src/api/routes/traders.py`（追加）

```python
from sse_starlette.sse import EventSourceResponse

@router.get("/api/traders/{id}/stream")
async def stream_analysis(id: str):
    """SSE 流式推送分析过程"""
    async def event_generator():
        async for event in graph.astream_events(input, config):
            yield {"event": event["event"], "data": json.dumps(event["data"])}
    return EventSourceResponse(event_generator())
```

**验收标准**:
- [ ] SSE 连接正确
- [ ] 实时推送各节点执行状态
- [ ] 客户端可接收并解析事件

**预估工时**: 2h | **依赖**: T-2.16, T-4.2

---

#### T-4.11: API 集成测试

**输出文件**: `tests/integration/test_api_auth.py`, `tests/integration/test_api_traders.py`

**验收标准**:
- [ ] 认证流程完整测试（注册→登录→OTP→JWT）
- [ ] Trader CRUD 完整测试
- [ ] 权限隔离测试（用户 A 不能操作用户 B 的 trader）
- [ ] 加密传输测试

**预估工时**: 4h | **依赖**: T-4.1 ~ T-4.8

---

#### T-4.12: 前端 - 复制 web/ 目录 + 基础配置

**输入**: 原 `web/` 目录
**输出文件**: `web/` 目录（复制）, `web/vite.config.ts`（修改 proxy target）

```typescript
// vite.config.ts 修改
proxy: {
  '/api': {
    target: 'http://localhost:8000',  // 原 8080 → 8000
    changeOrigin: true,
  },
  '/ws': {
    target: 'ws://localhost:8000',
    ws: true,
  },
},
```

**验收标准**:
- [ ] `npm install` 成功
- [ ] `npm run dev` 启动无报错
- [ ] API 代理指向 FastAPI 后端

**预估工时**: 0.5h | **依赖**: 无

---

#### T-4.13: 前端 - types.ts 重写

**输入**: REFACTOR_PLAN.md 第 8.2 节 + 前端探索结果
**输出文件**: `web/src/types.ts`

**变更清单**:
- `Position` → `StockPosition`（删除 leverage/liquidation_price/margin_used，增加 available_quantity/name/market）
- `AccountInfo`（删除 margin_used/margin_used_pct/wallet_balance，增加 cash_balance/market_value）
- `Exchange` → `BrokerConfig`（删除所有加密货币字段，增加 host/port/trade_env）
- `CreateTraderRequest`（删除 leverage/margin/coin_pool/oi_top，增加 watchlist/market_filter）
- `TraderInfo`（exchange_id → broker_id，增加 watchlist/market_filter）
- 新增: `MarketType`, `StockBoard`

**验收标准**:
- [ ] TypeScript 编译无报错
- [ ] 所有加密货币相关类型已删除
- [ ] 新类型与后端 API 响应匹配

**预估工时**: 2h | **依赖**: T-4.12

---

#### T-4.14: 前端 - api.ts 重写

**输入**: 前端探索结果中的完整 API 列表
**输出文件**: `web/src/lib/api.ts`

**变更清单**:
- `getExchangeConfigs()` → `getBrokerConfigs()`
- `getSupportedExchanges()` → `getSupportedBrokers()`
- `updateExchangeConfigs()` → `updateBrokerConfigs()`
- 删除: `getUserSignalSource()`, `saveUserSignalSource()`
- 新增: `getMarketQuote()`, `searchStocks()`, `getMarketHealth()`
- 所有 endpoint 路径与 FastAPI 路由对齐

**验收标准**:
- [ ] TypeScript 编译无报错
- [ ] 所有 API 调用与后端路由匹配

**预估工时**: 2h | **依赖**: T-4.13

---

#### T-4.15: 前端 - Zustand Store 更新

**输出文件**: `web/src/stores/tradersConfigStore.ts`, `web/src/stores/tradersModalStore.ts`

**变更清单**:
- `allExchanges` → `allBrokers`
- `configuredExchanges` → `configuredBrokers`
- 删除 `userSignalSource` 相关状态
- 删除 aster/hyperliquid 特殊过滤逻辑
- `showExchangeModal` → `showBrokerModal`
- `editingExchange` → `editingBroker`

**验收标准**:
- [ ] Store 状态与新类型匹配
- [ ] 无加密货币相关逻辑残留

**预估工时**: 1.5h | **依赖**: T-4.13

---

#### T-4.16: 前端 - TraderDashboard 改造

**输入**: 前端探索结果（989 行，最复杂的页面）
**输出文件**: `web/src/pages/TraderDashboard.tsx`

**变更清单**:
- 所有 `USDT` → `CNY`/`HKD`（根据 market 动态显示）
- 删除: leverage 显示、liquidation_price 显示、margin_used 显示
- 增加: `available_quantity` 显示（T+1 可卖提示）
- 增加: 股票名称显示（原只显示 symbol）
- 删除: candidate coins 警告区域
- 修改: position 表格列（删除杠杆/强平价，增加可卖数量/市场）

**验收标准**:
- [ ] Dashboard 正确显示港A股持仓
- [ ] T+1 可卖数量正确显示
- [ ] 无 USDT/加密货币文案残留
- [ ] 图表正常渲染

**预估工时**: 3h | **依赖**: T-4.14

---

#### T-4.17: 前端 - ExchangeConfigModal → BrokerConfigModal

**输入**: 原 `ExchangeConfigModal.tsx`（1085 行）
**输出文件**: `web/src/components/traders/BrokerConfigModal.tsx`

**变更清单**:
- 删除: Binance/Bybit/Hyperliquid/Aster/LIGHTER 所有配置表单
- 新增: 富途配置（host, port, trade_env）
- 新增: 长桥配置（app_key, app_secret, access_token）
- 新增: 模拟器配置（initial_cash）
- 保留: RSA 加密传输逻辑

**验收标准**:
- [ ] 三种券商配置表单正确
- [ ] 敏感字段加密传输
- [ ] 表单验证正确

**预估工时**: 3h | **依赖**: T-4.13

---

#### T-4.18: 前端 - TraderConfigModal 改造

**输入**: 原 `TraderConfigModal.tsx`（815 行）
**输出文件**: `web/src/components/TraderConfigModal.tsx`

**变更清单**:
- 删除: btc_eth_leverage, altcoin_leverage 输入
- 删除: is_cross_margin 开关
- 删除: use_coin_pool, use_oi_top 开关
- 删除: trading_symbols（BTCUSDT 格式）
- 新增: watchlist 编辑器（股票搜索 + 添加/删除）
- 新增: market_filter 选择（A股/港股/全部）
- 新增: max_position_pct 输入（单只最大仓位）
- 保留: AI 模型选择、扫描间隔、自定义提示词

**验收标准**:
- [ ] 关注列表编辑器可搜索/添加/删除股票
- [ ] 市场筛选正确
- [ ] 无加密货币相关表单残留

**预估工时**: 4h | **依赖**: T-4.13, T-4.5

---

#### T-4.19: 前端 - 删除加密货币专属组件

**变更清单**:
- 删除: `SignalSourceModal.tsx`（138 行）
- 删除: `SignalSourceWarning.tsx`（54 行）
- 删除: `ExchangeIcons.tsx`（215 行）→ 替换为 `BrokerIcons.tsx`
- 删除: `CryptoFeatureCard.tsx`（136 行）
- 删除: `WebCryptoEnvironmentCheck.tsx`（138 行）
- 更新: `ExchangesSection.tsx` → `BrokersSection.tsx`

**验收标准**:
- [ ] 所有删除的组件无引用残留
- [ ] TypeScript 编译无报错

**预估工时**: 2h | **依赖**: T-4.13

---

#### T-4.20: 前端 - 新增港A股组件

**输出文件**:
- `web/src/components/stock/StockSearch.tsx`
- `web/src/components/stock/WatchlistEditor.tsx`
- `web/src/components/stock/T1Indicator.tsx`
- `web/src/components/approval/TradeApprovalPanel.tsx`

**验收标准**:
- [ ] StockSearch 支持代码/名称/拼音搜索
- [ ] WatchlistEditor 可添加/删除/排序
- [ ] T1Indicator 正确显示可卖数量
- [ ] TradeApprovalPanel 可审批/拒绝交易

**预估工时**: 5h | **依赖**: T-4.5

---

#### T-4.21: 前端 - i18n 翻译更新

**输入**: 前端探索结果中的加密货币翻译 key 列表
**输出文件**: `web/src/i18n/translations.ts`

**变更清单**（约 60+ 个 key）:
- 删除所有 `hyperliquid*`, `aster*`, `lighter*`, `binance*` 相关 key
- 修改 `tradingSymbols` → `watchlist` 相关 key
- 修改 `leverage*` → 删除
- 修改 `margin*` → 删除
- 修改 `coinPool*`, `oiTop*`, `signalSource*` → 删除
- 新增: `broker*`, `watchlist*`, `marketFilter*`, `t1*`, `priceLimit*` 相关 key
- 修改 Landing 页面文案（加密货币 → 港A股）
- 修改 FAQ 内容（加密货币 → 港A股）

**验收标准**:
- [ ] 中英文翻译完整
- [ ] 无加密货币文案残留
- [ ] FAQ 内容适配港A股

**预估工时**: 3h | **依赖**: T-4.13

---

#### T-4.22: 前端 - Landing 页面改造

**输出文件**: `web/src/components/landing/*.tsx`

**变更清单**:
- HeroSection: 文案从加密货币改为港A股
- FeaturesSection: 特性描述适配
- HowItWorksSection: 流程说明适配
- AboutSection: 项目介绍适配
- FooterSection: 链接更新

**验收标准**:
- [ ] 所有文案适配港A股
- [ ] 无加密货币品牌/logo 残留

**预估工时**: 2h | **依赖**: T-4.21

---

#### T-4.23: 前端 - useTraderActions Hook 更新

**输入**: 原 `useTraderActions.ts`（651 行）
**输出文件**: `web/src/hooks/useTraderActions.ts`

**变更清单**:
- `saveExchangeConfig()` → `saveBrokerConfig()`
- `deleteExchangeConfig()` → `deleteBrokerConfig()`
- 删除: `saveSignalSource()` 相关逻辑
- 删除: aster/hyperliquid 特殊处理
- 更新: createTrader/updateTrader 参数

**验收标准**:
- [ ] 所有 CRUD 操作正确调用新 API
- [ ] 无加密货币逻辑残留

**预估工时**: 2h | **依赖**: T-4.14, T-4.15

---

#### T-4.24: 前端 - FAQ 内容重写

**输出文件**: `web/src/data/faqData.ts`, `web/src/i18n/translations.ts`（FAQ 部分）

**变更清单**:
- 删除所有加密货币 FAQ（Binance API、合约交易、USDT 等）
- 新增港A股 FAQ：
  - T+1 交易规则
  - 涨跌停板说明
  - 券商配置指南（富途/长桥）
  - 交易时间说明
  - 手续费说明
  - 模拟交易使用

**验收标准**:
- [ ] FAQ 内容完整覆盖港A股常见问题
- [ ] 中英文翻译完整

**预估工时**: 2h | **依赖**: T-4.21

---

### Phase 4 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| 43 个 API endpoint | `pytest tests/integration/test_api_*.py` |
| WebSocket 推送 | 浏览器 DevTools 验证 |
| SSE 流式推送 | `curl -N /api/traders/{id}/stream` |
| 前端类型更新 | `npm run build` 无报错 |
| Dashboard 改造 | 浏览器手动验证 |
| 券商配置弹窗 | 浏览器手动验证 |
| 关注列表编辑器 | 浏览器手动验证 |
| i18n 更新 | 中英文切换验证 |

---

## Phase 5: 集成测试 + 上线

### 里程碑 M5: 全链路测试通过，模拟交易运行 ≥ 24h 无异常

---

#### T-5.1: 全链路集成测试

**输出文件**: `tests/integration/test_full_pipeline.py`

```python
async def test_end_to_end_flow():
    """完整流程: 注册→配置→创建Trader→启动→分析→决策→执行→查看结果"""

async def test_multi_trader_concurrent():
    """多 Trader 并发运行"""

async def test_trader_restart_recovery():
    """Trader 重启后恢复状态（PostgresSaver）"""

async def test_non_trading_hours_skip():
    """非交易时间自动跳过"""

async def test_error_recovery():
    """数据源/券商异常时的降级处理"""
```

**验收标准**:
- [ ] 5 个全链路测试通过
- [ ] 使用 SimulatorBroker + Mock LLM
- [ ] 覆盖正常流程 + 异常流程

**预估工时**: 6h | **依赖**: Phase 4 全部完成

---

#### T-5.2: 模拟交易回测

**输出文件**: `tests/backtest/run_backtest.py`

```python
async def run_backtest(
    watchlist: list[str],
    start_date: str,
    end_date: str,
    initial_cash: float = 1_000_000,
):
    """使用历史数据运行回测"""
    # 1. 加载历史 K 线数据
    # 2. 按时间顺序模拟每个交易日
    # 3. 调用 LangGraph 图（Mock LLM 或真实 LLM）
    # 4. SimulatorBroker 执行交易
    # 5. 记录每日权益曲线
    # 6. 计算 Sharpe ratio, 最大回撤, 胜率
```

**验收标准**:
- [ ] 可回测至少 1 个月历史数据
- [ ] 输出权益曲线 + 统计指标
- [ ] A股 T+1 规则在回测中正确执行

**预估工时**: 6h | **依赖**: T-3.3, T-2.16

---

#### T-5.3: 性能测试

**输出文件**: `tests/performance/test_perf.py`

```python
async def test_analysis_latency():
    """单次分析周期延迟 < 60s"""

async def test_concurrent_traders():
    """10 个 Trader 并发运行，CPU < 80%"""

async def test_market_data_throughput():
    """100 只股票行情获取 < 10s"""

async def test_redis_cache_hit_rate():
    """缓存命中率 > 80%"""
```

**验收标准**:
- [ ] 单次分析 < 60s（含 LLM 调用）
- [ ] 10 Trader 并发无性能瓶颈
- [ ] 行情获取延迟可接受

**预估工时**: 3h | **依赖**: T-5.1

---

#### T-5.4: 安全审计

**输出文件**: `docs/security_audit.md`

**检查清单**:
- [ ] 无硬编码凭据
- [ ] 所有用户输入已验证（Pydantic）
- [ ] SQL 注入防护（SQLAlchemy 参数化）
- [ ] XSS 防护（FastAPI 自动转义）
- [ ] JWT 过期 + 黑名单
- [ ] API Key 加密存储
- [ ] CORS 配置正确
- [ ] Rate limiting（可选）

**预估工时**: 2h | **依赖**: T-5.1

---

#### T-5.5: 生产环境 Docker 配置

**输出文件**: `Dockerfile`（多阶段构建）, `docker-compose.prod.yml`, `nginx/nginx.conf`

```dockerfile
# Dockerfile - 多阶段构建
FROM python:3.11-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.11-slim AS runtime
COPY --from=builder /app/.venv /app/.venv
COPY src/ /app/src/
COPY prompts/ /app/prompts/
ENV PATH="/app/.venv/bin:$PATH"
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**验收标准**:
- [ ] Docker 镜像 < 500MB
- [ ] `docker compose -f docker-compose.prod.yml up` 启动正常
- [ ] Nginx 反向代理配置正确
- [ ] 健康检查通过

**预估工时**: 2h | **依赖**: T-0.11

---

#### T-5.6: 监控 + 告警配置

**输出文件**: `src/utils/metrics.py`（可选）

**监控项**:
- 应用健康检查（/api/health）
- 数据源可用性
- Trader 运行状态
- 错误率
- Telegram 告警（复用 T-0.9）

**验收标准**:
- [ ] 关键错误自动 Telegram 通知
- [ ] 数据源不可用时告警

**预估工时**: 2h | **依赖**: T-0.9

---

#### T-5.7: 24h 模拟交易验证

**验收标准**:
- [ ] SimulatorBroker 模式运行 ≥ 24h
- [ ] 无崩溃、无内存泄漏
- [ ] 决策日志正确记录
- [ ] 权益曲线正常
- [ ] 非交易时间正确跳过
- [ ] T+1 规则在跨日时正确重置

**预估工时**: 24h（运行时间）+ 2h（分析）| **依赖**: T-5.1

---

#### T-5.8: 文档更新

**输出文件**: `README.md`, `docs/getting-started.md`, `docs/architecture.md`

**验收标准**:
- [ ] README 包含快速开始指南
- [ ] 架构文档包含系统图
- [ ] 配置说明完整

**预估工时**: 3h | **依赖**: T-5.7

---

### Phase 5 交付清单

| 交付物 | 验证方式 |
|--------|----------|
| 全链路测试 | `pytest tests/integration/test_full_pipeline.py` |
| 回测报告 | 权益曲线 + Sharpe ratio |
| 性能报告 | 延迟 + 吞吐量指标 |
| 安全审计 | 检查清单全部通过 |
| Docker 生产镜像 | `docker compose up` |
| 24h 模拟运行 | 无异常日志 |
| 文档 | README + 架构图 |

---

## 任务依赖关系总览

```
Phase 0 (基础设施)
T-0.1 ──┬── T-0.2 ──┬── T-0.3 ──── T-0.4 ──── T-0.5
         │           │              │
         │           ├── T-0.6 ─────┤
         │           │              │
         ├── T-0.8   ├── T-0.9      ├── T-0.7
         │           │              │
         │           └── T-0.10 ────┘
         │                          │
         └──────────────────────── T-0.12

Phase 1 (数据层)                    Phase 2 (智能体)
T-1.1 ── T-1.2 ── T-1.3 ──┐        T-2.1 ──┬── T-2.2 ── T-2.8
                    │       │                ├── T-2.3 ── T-2.9
                    T-1.4   │                ├── T-2.4 ── T-2.10
                    T-1.5   │                ├── T-2.5 ── T-2.11
                    T-1.6   │                ├── T-2.6 ── T-2.13
                    T-1.7 ──┤                ├── T-2.7
                    T-1.8   │                ├── T-2.12
                    T-1.9   │                ├── T-2.14
                            │                ├── T-2.15
                            │                └── T-2.16 ── T-2.17 ── T-2.18
                            │
Phase 3 (交易执行)           │
T-3.1 ── T-3.2 ──┬── T-3.3 ┤
                  ├── T-3.4  │
                  ├── T-3.5  │
                  └── T-3.6 ─┤
                     T-3.7   │
                     T-3.8   │
                     T-3.9 ──┘

Phase 4 (API + 前端)
T-4.1~T-4.11 (API) ──┐
T-4.12~T-4.24 (前端) ─┤
                       │
Phase 5 (集成 + 上线)  │
T-5.1 ── T-5.2~T-5.8 ─┘
```

---

## 工时汇总

| Phase | 任务数 | 预估工时 | 建议周期 |
|-------|--------|----------|----------|
| Phase 0: 基础设施 | 12 | 20h | 1 周 |
| Phase 1: 数据层 | 9 | 16.5h | 1-1.5 周 |
| Phase 2: 智能体 | 18 | 40h | 2-3 周 |
| Phase 3: 交易执行 | 9 | 25.5h | 1.5-2 周 |
| Phase 4: API + 前端 | 24 | 52.5h | 2-3 周 |
| Phase 5: 集成上线 | 8 | 26h+ | 1-1.5 周 |
| **总计** | **80** | **~181h** | **8-10 周** |

---

## 并行执行建议

以下任务组可并行开发（不同开发者）：

**并行组 A**（后端基础）: T-0.1 → T-0.12
**并行组 B**（数据层）: T-1.1 → T-1.9（Phase 0 完成后）
**并行组 C**（智能体）: T-2.1 → T-2.18（与 Phase 1 并行，Mock 数据）
**并行组 D**（前端）: T-4.12 → T-4.24（与 Phase 2/3 并行，Mock API）

最优并行方案（2 人）：
- 开发者 1: Phase 0 → Phase 1 → Phase 3
- 开发者 2: Phase 2（Mock 数据）→ Phase 4（Mock API）
- 合并: Phase 5

最优并行方案（3 人）：
- 开发者 1: Phase 0 → Phase 3
- 开发者 2: Phase 1 → Phase 2
- 开发者 3: Phase 4（前端）
- 合并: Phase 5
