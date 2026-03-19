# FastAPI + LangGraph Production System Research

## 1. Project Structure

```
nofx/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── .env
├── .env.example
└── src/
    └── nofx/
        ├── __init__.py
        ├── main.py                  # FastAPI app entry
        ├── config.py                # pydantic-settings
        ├── deps.py                  # FastAPI dependencies
        ├── db/
        │   ├── __init__.py
        │   ├── engine.py            # async SQLAlchemy engine
        │   ├── session.py           # async session factory
        │   └── models/
        │       ├── __init__.py
        │       ├── base.py          # DeclarativeBase
        │       └── user.py
        ├── auth/
        │   ├── __init__.py
        │   ├── router.py            # login/register/totp endpoints
        │   ├── jwt.py               # JWT create/verify
        │   ├── password.py          # bcrypt hashing
        │   ├── totp.py              # pyotp TOTP
        │   └── deps.py              # get_current_user dependency
        ├── cache/
        │   ├── __init__.py
        │   ├── redis.py             # redis client lifecycle
        │   └── decorators.py        # cache decorator
        ├── graph/
        │   ├── __init__.py
        │   ├── checkpointer.py      # AsyncPostgresSaver setup
        │   ├── trading_graph.py     # LangGraph graph definition
        │   └── nodes/
        │       ├── __init__.py
        │       ├── market_data.py
        │       ├── analysis.py
        │       └── execution.py
        ├── api/
        │   ├── __init__.py
        │   ├── graph_routes.py      # SSE + WebSocket endpoints
        │   └── trading_routes.py
        └── scheduler/
            ├── __init__.py
            └── jobs.py              # APScheduler periodic tasks
```

---

## 2. pyproject.toml

```toml
[project]
name = "nofx"
version = "0.1.0"
description = "Trading system with LangGraph agents"
requires-python = ">=3.12"
dependencies = [
    # Core
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    # LangGraph
    "langgraph>=0.4.0",
    "langgraph-checkpoint-postgres>=3.0.0",
    "langchain-openai>=0.3.0",
    # Database
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    # Auth
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "pyotp>=2.9.0",
    "python-multipart>=0.0.12",
    # Cache
    "redis>=5.2.0",
    # Scheduler
    "apscheduler>=3.10.4",
    # Config
    "pydantic-settings>=2.6.0",
    # SSE
    "sse-starlette>=2.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.28.0",
    "ruff>=0.8.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/nofx"]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

---

## 3. pydantic-settings Configuration

```python
# src/nofx/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # App
    app_name: str = "nofx"
    debug: bool = False

    # Postgres
    postgres_host: str = "localhost"
    postgres_port: int = 5432
    postgres_user: str = "nofx"
    postgres_password: str = ""
    postgres_db: str = "nofx"

    @property
    def database_url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.postgres_user}:{self.postgres_password}"
            f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
        )

    @property
    def database_url_sync(self) -> str:
        """For Alembic migrations (sync driver)."""
        return (
            f"postgresql://{self.postgres_user}:{self.postgres_password}"
            f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
        )

    @property
    def checkpoint_conn_string(self) -> str:
        """For langgraph-checkpoint-postgres (raw asyncpg, no SQLAlchemy prefix)."""
        return (
            f"postgresql://{self.postgres_user}:{self.postgres_password}"
            f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
        )

    # Redis
    redis_url: str = "redis://localhost:6379/0"
    cache_ttl: int = 300  # seconds

    # JWT
    jwt_secret: str = ""
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 30

    # OpenAI
    openai_api_key: str = ""


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## 4. FastAPI App Entry with Lifespan

```python
# src/nofx/main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI

from nofx.config import get_settings
from nofx.db.engine import dispose_engine
from nofx.cache.redis import close_redis, init_redis
from nofx.graph.checkpointer import close_checkpointer, init_checkpointer
from nofx.scheduler.jobs import start_scheduler, stop_scheduler
from nofx.auth.router import router as auth_router
from nofx.api.graph_routes import router as graph_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    # Startup
    await init_redis(settings.redis_url)
    await init_checkpointer(settings.checkpoint_conn_string)
    start_scheduler()
    yield
    # Shutdown
    stop_scheduler()
    await close_checkpointer()
    await close_redis()
    await dispose_engine()


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(
        title=settings.app_name,
        lifespan=lifespan,
    )
    app.include_router(auth_router, prefix="/auth", tags=["auth"])
    app.include_router(graph_router, prefix="/graph", tags=["graph"])
    return app


app = create_app()
```

---

## 5. SQLAlchemy 2.0 Async Setup

### Engine & Session

```python
# src/nofx/db/engine.py
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine

from nofx.config import get_settings

_engine: AsyncEngine | None = None


def get_engine() -> AsyncEngine:
    global _engine
    if _engine is None:
        settings = get_settings()
        _engine = create_async_engine(
            settings.database_url,
            pool_size=20,
            max_overflow=10,
            pool_pre_ping=True,
            echo=settings.debug,
        )
    return _engine


async def dispose_engine() -> None:
    global _engine
    if _engine is not None:
        await _engine.dispose()
        _engine = None
```

```python
# src/nofx/db/session.py
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from nofx.db.engine import get_engine

_session_factory: async_sessionmaker[AsyncSession] | None = None


def get_session_factory() -> async_sessionmaker[AsyncSession]:
    global _session_factory
    if _session_factory is None:
        _session_factory = async_sessionmaker(
            get_engine(),
            class_=AsyncSession,
            expire_on_commit=False,
        )
    return _session_factory


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI dependency that yields an async session."""
    factory = get_session_factory()
    async with factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Model Definition

```python
# src/nofx/db/models/base.py
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(),
        onupdate=func.now(),
    )
```

```python
# src/nofx/db/models/user.py
import uuid

from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from nofx.db.models.base import Base, TimestampMixin


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid4,
    )
    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True,
    )
    hashed_password: Mapped[str] = mapped_column(String(255))
    totp_secret: Mapped[str | None] = mapped_column(String(64), nullable=True)
    is_active: Mapped[bool] = mapped_column(default=True)
```

### Alembic Async Migration Setup

```ini
# alembic.ini (relevant section)
[alembic]
script_location = alembic
sqlalchemy.url = driver://user:pass@localhost/dbname  # overridden in env.py
```

```python
# alembic/env.py
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

from nofx.config import get_settings
from nofx.db.models.base import Base
# Import all models so Base.metadata is populated
from nofx.db.models import user  # noqa: F401

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    settings = get_settings()
    context.configure(
        url=settings.database_url_sync,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    settings = get_settings()
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = settings.database_url
    connectable = async_engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

## 6. LangGraph + PostgreSQL Checkpointer

```python
# src/nofx/graph/checkpointer.py
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

_checkpointer: AsyncPostgresSaver | None = None


async def init_checkpointer(conn_string: str) -> None:
    """
    Call during app startup (lifespan).
    from_conn_string creates an asyncpg connection pool internally.
    setup() creates the checkpoint tables if they don't exist.
    """
    global _checkpointer
    _checkpointer = AsyncPostgresSaver.from_conn_string(conn_string)
    # __aenter__ initializes the pool
    await _checkpointer.__aenter__()
    await _checkpointer.setup()


async def close_checkpointer() -> None:
    global _checkpointer
    if _checkpointer is not None:
        await _checkpointer.__aexit__(None, None, None)
        _checkpointer = None


def get_checkpointer() -> AsyncPostgresSaver:
    if _checkpointer is None:
        raise RuntimeError("Checkpointer not initialized")
    return _checkpointer
```

### Passing a Pre-configured Pool (Advanced)

If you need control over pool size, pass your own asyncpg pool:

```python
# Alternative: bring your own pool
import asyncpg
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver


async def init_checkpointer_with_pool(conn_string: str) -> AsyncPostgresSaver:
    pool = await asyncpg.create_pool(
        conn_string,
        min_size=5,
        max_size=20,
        command_timeout=60,
    )
    checkpointer = AsyncPostgresSaver(pool)
    await checkpointer.setup()
    return checkpointer
```

### Graph Definition

```python
# src/nofx/graph/trading_graph.py
from typing import Annotated, TypedDict

from langchain_openai import ChatOpenAI
from langgraph.graph import END, StateGraph
from langgraph.graph.message import add_messages

from nofx.graph.checkpointer import get_checkpointer


class TradingState(TypedDict):
    messages: Annotated[list, add_messages]
    market_data: dict | None
    analysis: str | None
    decision: str | None


async def fetch_market_data(state: TradingState) -> dict:
    # Node: fetch latest market data
    return {"market_data": {"BTC": 67000, "ETH": 3500}}


async def analyze(state: TradingState) -> dict:
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = await llm.ainvoke(
        f"Analyze this market data: {state['market_data']}"
    )
    return {"analysis": result.content}


async def decide(state: TradingState) -> dict:
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = await llm.ainvoke(
        f"Based on analysis: {state['analysis']}, recommend: BUY/SELL/HOLD"
    )
    return {"decision": result.content}


def build_trading_graph() -> StateGraph:
    builder = StateGraph(TradingState)
    builder.add_node("fetch_market_data", fetch_market_data)
    builder.add_node("analyze", analyze)
    builder.add_node("decide", decide)

    builder.set_entry_point("fetch_market_data")
    builder.add_edge("fetch_market_data", "analyze")
    builder.add_edge("analyze", "decide")
    builder.add_edge("decide", END)

    return builder


def get_compiled_graph():
    """Returns a compiled graph with Postgres checkpointing."""
    builder = build_trading_graph()
    return builder.compile(checkpointer=get_checkpointer())
```

---

## 7. FastAPI + LangGraph Integration: SSE & WebSocket

### SSE Streaming Endpoint

```python
# src/nofx/api/graph_routes.py
import json
import uuid
from typing import AsyncGenerator

from fastapi import APIRouter, Depends, WebSocket, WebSocketDisconnect
from sse_starlette.sse import EventSourceResponse

from nofx.auth.deps import get_current_user
from nofx.graph.trading_graph import TradingState, get_compiled_graph

router = APIRouter()


async def _stream_graph(
    thread_id: str,
    input_state: TradingState,
) -> AsyncGenerator[str, None]:
    """Generator that yields SSE-formatted events from the graph."""
    graph = get_compiled_graph()
    config = {"configurable": {"thread_id": thread_id}}

    async for event in graph.astream_events(
        input_state,
        config=config,
        version="v2",
    ):
        kind = event["event"]
        if kind == "on_chain_start":
            yield json.dumps({
                "event": "node_start",
                "node": event.get("name", ""),
            })
        elif kind == "on_chain_end":
            yield json.dumps({
                "event": "node_end",
                "node": event.get("name", ""),
                "data": str(event.get("data", {}).get("output", "")),
            })
        elif kind == "on_chat_model_stream":
            content = event["data"]["chunk"].content
            if content:
                yield json.dumps({
                    "event": "token",
                    "data": content,
                })

    yield json.dumps({"event": "done"})


@router.post("/run/stream")
async def run_graph_stream(
    user=Depends(get_current_user),
):
    """SSE endpoint: streams graph execution events to the client."""
    thread_id = str(uuid.uuid4())
    input_state: TradingState = {
        "messages": [],
        "market_data": None,
        "analysis": None,
        "decision": None,
    }
    return EventSourceResponse(
        _stream_graph(thread_id, input_state),
        media_type="text/event-stream",
    )


@router.post("/run/invoke")
async def run_graph_invoke(
    user=Depends(get_current_user),
):
    """Non-streaming: runs graph to completion and returns final state."""
    graph = get_compiled_graph()
    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}
    input_state: TradingState = {
        "messages": [],
        "market_data": None,
        "analysis": None,
        "decision": None,
    }
    result = await graph.ainvoke(input_state, config=config)
    return {"thread_id": thread_id, "result": result}
```

### WebSocket Endpoint

```python
# Append to src/nofx/api/graph_routes.py

@router.websocket("/ws/{thread_id}")
async def websocket_graph(websocket: WebSocket, thread_id: str):
    """
    WebSocket endpoint for real-time bidirectional communication.
    Client sends messages, server streams graph events back.
    """
    await websocket.accept()
    graph = get_compiled_graph()
    config = {"configurable": {"thread_id": thread_id}}

    try:
        while True:
            # Wait for client message
            data = await websocket.receive_json()
            user_message = data.get("message", "")

            input_state: TradingState = {
                "messages": [("human", user_message)],
                "market_data": None,
                "analysis": None,
                "decision": None,
            }

            # Stream graph events back over WebSocket
            async for event in graph.astream_events(
                input_state,
                config=config,
                version="v2",
            ):
                kind = event["event"]
                if kind == "on_chat_model_stream":
                    content = event["data"]["chunk"].content
                    if content:
                        await websocket.send_json({
                            "type": "token",
                            "data": content,
                        })
                elif kind == "on_chain_end":
                    await websocket.send_json({
                        "type": "node_complete",
                        "node": event.get("name", ""),
                    })

            await websocket.send_json({"type": "done"})

    except WebSocketDisconnect:
        pass
```

---

## 8. Authentication (JWT + OAuth2 + TOTP)

### Password Hashing

```python
# src/nofx/auth/password.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

### JWT Token Management

```python
# src/nofx/auth/jwt.py
from datetime import datetime, timedelta, timezone

from jose import JWTError, jwt

from nofx.config import get_settings


def create_access_token(
    subject: str,
    extra_claims: dict | None = None,
) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(
        minutes=settings.jwt_expire_minutes
    )
    payload = {
        "sub": subject,
        "exp": expire,
        **(extra_claims or {}),
    }
    return jwt.encode(
        payload,
        settings.jwt_secret,
        algorithm=settings.jwt_algorithm,
    )


def decode_access_token(token: str) -> dict:
    settings = get_settings()
    try:
        return jwt.decode(
            token,
            settings.jwt_secret,
            algorithms=[settings.jwt_algorithm],
        )
    except JWTError as e:
        raise ValueError(f"Invalid token: {e}") from e
```

### TOTP (Two-Factor Authentication)

```python
# src/nofx/auth/totp.py
import pyotp


def generate_totp_secret() -> str:
    """Generate a new TOTP secret for a user."""
    return pyotp.random_base32()


def get_provisioning_uri(
    secret: str,
    email: str,
    issuer: str = "nofx",
) -> str:
    """Returns a URI for QR code generation (Google Authenticator, etc.)."""
    totp = pyotp.TOTP(secret)
    return totp.provisioning_uri(name=email, issuer_name=issuer)


def verify_totp(secret: str, code: str) -> bool:
    """Verify a TOTP code. valid_window=1 allows 30s clock skew."""
    totp = pyotp.TOTP(secret)
    return totp.verify(code, valid_window=1)
```

### Auth Dependencies

```python
# src/nofx/auth/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from nofx.auth.jwt import decode_access_token
from nofx.db.models.user import User
from nofx.db.session import get_db

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        payload = decode_access_token(token)
        user_id: str = payload.get("sub")
        if user_id is None:
            raise ValueError("Missing sub claim")
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )

    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if user is None or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found or inactive",
        )
    return user
```

### Auth Router

```python
# src/nofx/auth/router.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from nofx.auth.jwt import create_access_token
from nofx.auth.password import hash_password, verify_password
from nofx.auth.totp import generate_totp_secret, get_provisioning_uri, verify_totp
from nofx.db.models.user import User
from nofx.db.session import get_db
from nofx.auth.deps import get_current_user

router = APIRouter()


class RegisterRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


class TOTPSetupResponse(BaseModel):
    secret: str
    provisioning_uri: str


class TOTPVerifyRequest(BaseModel):
    code: str


@router.post("/register", status_code=status.HTTP_201_CREATED)
async def register(
    body: RegisterRequest,
    db: AsyncSession = Depends(get_db),
):
    existing = await db.execute(
        select(User).where(User.email == body.email)
    )
    if existing.scalar_one_or_none():
        raise HTTPException(status_code=409, detail="Email already registered")

    user = User(
        email=body.email,
        hashed_password=hash_password(body.password),
    )
    db.add(user)
    await db.flush()
    return {"id": str(user.id), "email": user.email}


@router.post("/login", response_model=TokenResponse)
async def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(User).where(User.email == form.username)
    )
    user = result.scalar_one_or_none()
    if not user or not verify_password(form.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Bad credentials")

    # If TOTP is enabled, require it via a separate step
    if user.totp_secret:
        return TokenResponse(
            access_token=create_access_token(
                str(user.id),
                extra_claims={"totp_verified": False},
            ),
        )

    return TokenResponse(
        access_token=create_access_token(str(user.id)),
    )


@router.post("/totp/setup", response_model=TOTPSetupResponse)
async def setup_totp(
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    secret = generate_totp_secret()
    user.totp_secret = secret
    db.add(user)
    await db.flush()
    return TOTPSetupResponse(
        secret=secret,
        provisioning_uri=get_provisioning_uri(secret, user.email),
    )


@router.post("/totp/verify", response_model=TokenResponse)
async def verify_totp_endpoint(
    body: TOTPVerifyRequest,
    user: User = Depends(get_current_user),
):
    if not user.totp_secret:
        raise HTTPException(status_code=400, detail="TOTP not configured")
    if not verify_totp(user.totp_secret, body.code):
        raise HTTPException(status_code=401, detail="Invalid TOTP code")

    return TokenResponse(
        access_token=create_access_token(
            str(user.id),
            extra_claims={"totp_verified": True},
        ),
    )
```

---

## 9. Redis Caching Patterns

### Redis Client Lifecycle

```python
# src/nofx/cache/redis.py
from redis.asyncio import Redis

_redis: Redis | None = None


async def init_redis(url: str) -> None:
    global _redis
    _redis = Redis.from_url(
        url,
        decode_responses=True,
        max_connections=20,
    )
    # Verify connection
    await _redis.ping()


async def close_redis() -> None:
    global _redis
    if _redis is not None:
        await _redis.aclose()
        _redis = None


def get_redis() -> Redis:
    if _redis is None:
        raise RuntimeError("Redis not initialized")
    return _redis
```

### Cache Decorator with TTL

```python
# src/nofx/cache/decorators.py
import functools
import json
from typing import Callable

from nofx.cache.redis import get_redis
from nofx.config import get_settings


def cached(
    key_prefix: str,
    ttl: int | None = None,
    key_builder: Callable[..., str] | None = None,
):
    """
    Async cache decorator backed by Redis.

    Usage:
        @cached("market_data", ttl=60)
        async def get_btc_price(symbol: str) -> dict:
            ...

        @cached("analysis", key_builder=lambda s, tf: f"{s}:{tf}")
        async def get_analysis(symbol: str, timeframe: str) -> dict:
            ...
    """
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            redis = get_redis()
            settings = get_settings()
            effective_ttl = ttl or settings.cache_ttl

            # Build cache key
            if key_builder:
                suffix = key_builder(*args, **kwargs)
            else:
                parts = [str(a) for a in args]
                parts += [f"{k}={v}" for k, v in sorted(kwargs.items())]
                suffix = ":".join(parts) if parts else "default"

            cache_key = f"{key_prefix}:{suffix}"

            # Try cache hit
            cached_value = await redis.get(cache_key)
            if cached_value is not None:
                return json.loads(cached_value)

            # Cache miss: call function
            result = await func(*args, **kwargs)

            # Store in cache
            await redis.set(
                cache_key,
                json.dumps(result, default=str),
                ex=effective_ttl,
            )
            return result

        # Expose cache invalidation
        async def invalidate(*args, **kwargs):
            redis = get_redis()
            if key_builder:
                suffix = key_builder(*args, **kwargs)
            else:
                parts = [str(a) for a in args]
                parts += [f"{k}={v}" for k, v in sorted(kwargs.items())]
                suffix = ":".join(parts) if parts else "default"
            await redis.delete(f"{key_prefix}:{suffix}")

        wrapper.invalidate = invalidate
        return wrapper
    return decorator
```

### Usage Example: Market Data Caching

```python
# src/nofx/graph/nodes/market_data.py
from nofx.cache.decorators import cached


@cached("market:price", ttl=30)
async def get_price(symbol: str) -> dict:
    """Cached for 30s — market data doesn't need sub-second freshness."""
    # ... call exchange API
    return {"symbol": symbol, "price": 67000.0, "ts": "2026-03-19T12:00:00Z"}


@cached("market:orderbook", ttl=5)
async def get_orderbook(symbol: str, depth: int = 20) -> dict:
    """Orderbook cached for 5s — more volatile."""
    # ... call exchange API
    return {"symbol": symbol, "bids": [], "asks": [], "depth": depth}
```

---

## 10. APScheduler: Background Task Scheduling

```python
# src/nofx/scheduler/jobs.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger

_scheduler: AsyncIOScheduler | None = None


def get_scheduler() -> AsyncIOScheduler:
    global _scheduler
    if _scheduler is None:
        _scheduler = AsyncIOScheduler()
    return _scheduler


def start_scheduler() -> None:
    scheduler = get_scheduler()

    # Run trading cycle every 5 minutes during market hours
    scheduler.add_job(
        run_trading_cycle,
        trigger=CronTrigger(
            day_of_week="mon-fri",
            hour="9-16",
            minute="*/5",
        ),
        id="trading_cycle",
        replace_existing=True,
    )

    # Refresh market data cache every 30 seconds
    scheduler.add_job(
        refresh_market_cache,
        trigger=IntervalTrigger(seconds=30),
        id="market_cache_refresh",
        replace_existing=True,
    )

    # Daily portfolio rebalance check at 9:00 AM
    scheduler.add_job(
        daily_rebalance_check,
        trigger=CronTrigger(hour=9, minute=0),
        id="daily_rebalance",
        replace_existing=True,
    )

    scheduler.start()


def stop_scheduler() -> None:
    global _scheduler
    if _scheduler is not None:
        _scheduler.shutdown(wait=False)
        _scheduler = None


async def run_trading_cycle() -> None:
    """Periodic trading cycle: runs the LangGraph agent."""
    from nofx.graph.trading_graph import get_compiled_graph
    import uuid

    graph = get_compiled_graph()
    thread_id = f"scheduled-{uuid.uuid4()}"
    config = {"configurable": {"thread_id": thread_id}}

    await graph.ainvoke(
        {
            "messages": [],
            "market_data": None,
            "analysis": None,
            "decision": None,
        },
        config=config,
    )


async def refresh_market_cache() -> None:
    """Proactively warm the market data cache."""
    from nofx.graph.nodes.market_data import get_price
    symbols = ["BTC", "ETH", "SOL"]
    for symbol in symbols:
        await get_price.invalidate(symbol)
        await get_price(symbol)


async def daily_rebalance_check() -> None:
    """Placeholder for daily portfolio rebalance logic."""
    pass
```

---

## 11. Docker Multi-Stage Build with uv

```dockerfile
# Dockerfile
# Stage 1: Build dependencies
FROM python:3.12-slim AS builder

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Copy dependency files first (cache layer)
COPY pyproject.toml uv.lock ./

# Install dependencies into a virtual environment
RUN uv sync --frozen --no-dev --no-install-project

# Copy source code
COPY src/ src/
COPY alembic.ini alembic/
COPY alembic/ alembic/

# Install the project itself
RUN uv sync --frozen --no-dev


# Stage 2: Runtime
FROM python:3.12-slim AS runtime

# Create non-root user
RUN groupadd -r nofx && useradd -r -g nofx nofx

WORKDIR /app

# Copy the virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy alembic config for migrations
COPY --from=builder /app/alembic.ini /app/alembic.ini
COPY --from=builder /app/alembic /app/alembic

# Put venv on PATH
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONUNBUFFERED=1

USER nofx

EXPOSE 8000

CMD ["uvicorn", "nofx.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: >
      sh -c "alembic upgrade head &&
             uvicorn nofx.main:app --host 0.0.0.0 --port 8000"

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: nofx
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: nofx
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nofx"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### .env.example

```bash
# .env.example
# Postgres
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=nofx
POSTGRES_PASSWORD=changeme
POSTGRES_DB=nofx

# Redis
REDIS_URL=redis://localhost:6379/0
CACHE_TTL=300

# JWT
JWT_SECRET=your-secret-key-change-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30

# OpenAI
OPENAI_API_KEY=sk-...

# App
DEBUG=false
```

---

## 12. LangGraph Server vs Custom FastAPI Wrapper

| Aspect | LangGraph Server | Custom FastAPI Wrapper |
|--------|-----------------|----------------------|
| Setup complexity | Low (built-in) | Medium (you own the HTTP layer) |
| Auth/middleware | Limited, opinionated | Full control (JWT, TOTP, rate limiting) |
| Streaming | Built-in SSE | Manual via sse-starlette / WebSocket |
| Background tasks | Not built-in | APScheduler, Celery, etc. |
| Scaling | LangGraph Cloud or Docker | Any ASGI deployment (k8s, ECS, etc.) |
| Checkpointing | Automatic | Manual setup (AsyncPostgresSaver) |
| Custom endpoints | Limited | Unlimited |
| Observability | LangSmith integration | Custom (OpenTelemetry, Prometheus) |
| Existing API integration | Difficult | Natural (just add routers) |

**Recommendation for a trading system**: Custom FastAPI wrapper. You need custom auth, background scheduling for trading cycles, Redis caching for market data, and full control over the API surface. LangGraph Server is better suited for simpler chatbot-style deployments.

---

## 13. Key Decisions & Gotchas

### AsyncPostgresSaver Lifecycle

The biggest gotcha: `AsyncPostgresSaver.from_conn_string()` returns an async context manager. In a FastAPI app, you cannot use `async with` in the module scope. Instead, manually call `__aenter__` / `__aexit__` in the lifespan handler (as shown in section 6). Alternatively, pass a pre-created `asyncpg.Pool` directly to `AsyncPostgresSaver(pool)`.

### aioredis is Dead

`aioredis` was merged into `redis-py` starting from v4.2. Use `redis.asyncio` (from the `redis` package) directly. Do not install `aioredis` separately.

### APScheduler 3.x vs 4.x

APScheduler 4.x is a complete rewrite with a different API. As of early 2026, 3.x is still the stable, widely-used version. The examples above use 3.x (`AsyncIOScheduler`). If you want 4.x, the API changes significantly (data stores, event brokers, etc.).

### FastAPI Lifespan vs @app.on_event

`@app.on_event("startup")` and `@app.on_event("shutdown")` are deprecated. Use the `lifespan` async context manager pattern (shown in section 4).

### SQLAlchemy expire_on_commit

Set `expire_on_commit=False` in the session factory. Without this, accessing attributes on a model after `commit()` triggers a lazy load, which fails in async context.

---

## Sources

### FastAPI + LangGraph Integration
- [Production-Ready Template (GitHub)](https://github.com/luwhano/fastapi-langgraph-agent-production-ready-template)
- [Streaming AI Agent with FastAPI & LangGraph (2025-26 Guide)](https://dev.to/kasi_viswanath/streaming-ai-agent-with-fastapi-langgraph-2025-26-guide-1nkn)
- [SSE Streaming - langgraph-fullstack-python (DeepWiki)](https://deepwiki.com/langchain-ai/langgraph-fullstack-python/2.3-sse-streaming)
- [SSE with FastAPI, React, and LangGraph](https://www.softgrade.org/sse-with-fastapi-react-langgraph/)
- [Agent Service Toolkit (GitHub)](https://github.com/JoshuaC215/agent-service-toolkit)
- [Real-Time AI Agent Streaming with WebSockets (Vstorm)](https://oss.vstorm.co/blog/websocket-streaming-ai-agents/)
- [FastAPI SSE Official Docs](https://fastapi.tiangolo.com/tutorial/server-sent-events/)

### LangGraph Checkpointing
- [langgraph-checkpoint-postgres (PyPI)](https://pypi.org/project/langgraph-checkpoint-postgres/)
- [AsyncPostgresSaver API Reference](https://reference.langchain.com/python/langgraph.checkpoint.postgres/aio/AsyncPostgresSaver)
- [Postgres Checkpointer Internals (Blog)](https://blog.lordpatil.com/posts/langgraph-postgres-checkpointer/)
- [PR #1452: Allow Passing Pool](https://github.com/langchain-ai/langgraph/pull/1452)
- [Production Connection Pooling Discussion](https://forum.langchain.com/t/langgraph-production-connection-pooling-inquiry/1730)

### Authentication
- [FastAPI OAuth2 + JWT Official Docs](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [Securing FastAPI with JWT (TestDriven.io)](https://testdriven.io/blog/fastapi-jwt-auth/)
- [OTP FastAPI Guide](https://monkdeskapps.com/blog/otp-fastapi-a-comprehensive-guide)
- [two-fast-auth (GitHub)](https://github.com/rennf93/two-fast-auth)

### SQLAlchemy 2.0 Async
- [Async SQLAlchemy 2 Step-by-Step Guide](https://dev.to/amverum/asynchronous-sqlalchemy-2-a-simple-step-by-step-guide-to-configuration-models-relationships-and-3ob3)
- [10 SQLAlchemy 2.0 Patterns for Clean Async Postgres](https://medium.com/@ThinkingLoop/10-sqlalchemy-2-0-patterns-for-clean-async-postgres-af8c4bcd86fe)
- [SQLAlchemy Async I/O Official Docs](http://docs.sqlalchemy.org/en/latest/orm/extensions/asyncio.html)
- [Alembic Async Migrations Discussion](https://github.com/sqlalchemy/alembic/discussions/1229)

### Redis Caching
- [Async Caching with aiocache and Redis](https://medium.com/neural-engineer/async-caching-in-python-with-aiocache-and-redis-8ca985614a9a)
- [Caching with Redis in FastAPI](https://python.plainenglish.io/caching-with-redis-boost-your-fastapi-applications-with-speed-and-scalability-af45626a57f3)
- [redis-func-cache (PyPI)](https://pypi.org/project/redis-func-cache/)

### Project Structure & Docker
- [Optimal Dockerfile for Python with uv (Depot)](https://depot.dev/docs/container-builds/optimal-dockerfiles/python-uv-dockerfile)
- [Best Practices for Python & uv in Docker](https://ashishb.net/programming/using-python-uv-inside-docker/)
- [Multi-Stage Docker Builds with uv](https://dev.to/kummerer94/multi-stage-docker-builds-for-pyton-projects-using-uv-223g)
- [Pydantic Settings Official Docs](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [pyproject.toml Guide (Better Stack)](https://betterstack.com/community/guides/scaling-python/pyproject-explained/)

### APScheduler
- [FastAPI + Scheduler Background Tasks](https://digon.io/en/blog/2025_06_05_async_job_scheduling_with_fastapi)
- [PyCron: FastAPI + APScheduler (GitHub)](https://github.com/ankur-katiyar/pycron)
- [Lifespan Events Migration (StackOverflow)](https://stackoverflow.com/questions/78042466/)
