# LangGraph Advanced Patterns for Financial Trading Multi-Agent System

## 1. Persistence & Checkpointing

### Core Concepts

LangGraph persistence enables conversation memory, time-travel debugging, and fault tolerance -- all critical for a trading system where state must survive crashes and decisions need audit trails.

### Checkpointer Setup

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from langgraph.graph import StateGraph, MessagesState

# Development: in-memory
memory = MemorySaver()

# Production: PostgreSQL (recommended for trading systems)
async with AsyncPostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/trading_db"
) as checkpointer:
    await checkpointer.setup()
    graph = workflow.compile(checkpointer=checkpointer)
```

### Thread-Based Conversation Memory

```python
# Each trading session gets a unique thread_id
config = {"configurable": {"thread_id": "trading-session-2026-03-19-001"}}

# First invocation
result = await graph.ainvoke(
    {"messages": [HumanMessage(content="Analyze AAPL sentiment")]},
    config=config
)

# Subsequent invocation on same thread -- has full history
result = await graph.ainvoke(
    {"messages": [HumanMessage(content="Now compare with MSFT")]},
    config=config
)
```

### State Snapshots & Time Travel

```python
# Get all checkpoints for a thread (audit trail)
all_states = []
async for state in graph.aget_state_history(config):
    all_states.append(state)

# Replay from a specific checkpoint
past_config = all_states[2].config  # go back 2 steps
result = await graph.ainvoke(
    {"messages": [HumanMessage(content="Re-evaluate with updated data")]},
    past_config
)
```

### Trading Application: Position State Persistence

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph
import operator

class TradingState(TypedDict):
    messages: Annotated[list, operator.add]
    portfolio: dict              # {symbol: {qty, avg_price, ...}}
    pending_orders: list         # orders awaiting execution
    risk_metrics: dict           # VaR, drawdown, exposure
    market_regime: str           # bull/bear/sideways/volatile
    analysis_results: Annotated[list, operator.add]  # accumulate analyses
    trade_signals: Annotated[list, operator.add]
    session_id: str
```

### Custom Checkpointer for Trading Audit

```python
from langgraph.checkpoint.base import BaseCheckpointSaver

class TradingAuditCheckpointer(BaseCheckpointSaver):
    """Persists every state transition for regulatory compliance."""

    async def aput(self, config, checkpoint, metadata, new_versions):
        # Store checkpoint with timestamp and decision rationale
        await self.storage.save(
            thread_id=config["configurable"]["thread_id"],
            checkpoint=checkpoint,
            metadata={
                **metadata,
                "timestamp": datetime.utcnow().isoformat(),
                "portfolio_snapshot": checkpoint["channel_values"].get("portfolio"),
            }
        )
```

---

## 2. Human-in-the-Loop (HITL)

### The `interrupt()` Pattern (Recommended, v0.2.58+)

The modern approach uses `interrupt()` inside node functions, replacing the older `interrupt_before`/`interrupt_after` compile options.

```python
from langgraph.types import interrupt, Command

def trade_execution_node(state: TradingState):
    """Pause for human approval before executing trades."""
    order = state["pending_orders"][-1]

    # This pauses the graph and returns the value to the caller
    human_decision = interrupt({
        "action": "approve_trade",
        "order": order,
        "risk_assessment": state["risk_metrics"],
        "message": f"Approve {order['side']} {order['qty']} {order['symbol']} @ {order['price']}?"
    })

    if human_decision["approved"]:
        return {"pending_orders": [], "messages": [f"Trade executed: {order}"]}
    else:
        return {"pending_orders": [], "messages": [f"Trade rejected: {human_decision.get('reason')}"]}
```

### Resuming After Interrupt

```python
# Graph pauses at interrupt() -- caller gets the interrupt value
# Resume with Command:
result = await graph.ainvoke(
    Command(resume={"approved": True}),
    config=config
)

# Or reject:
result = await graph.ainvoke(
    Command(resume={"approved": False, "reason": "Risk too high"}),
    config=config
)
```

### Multi-Level Approval for Trading

```python
def risk_gate_node(state: TradingState):
    """Multi-tier approval based on trade size."""
    order = state["pending_orders"][-1]
    notional = order["qty"] * order["price"]

    if notional < 10_000:
        return {"messages": ["Auto-approved: small trade"]}

    elif notional < 100_000:
        decision = interrupt({
            "level": "trader",
            "message": f"Trader approval needed: ${notional:,.0f} trade"
        })
        if decision["approved"]:
            return {"messages": ["Trader approved"]}

    else:
        decision = interrupt({
            "level": "risk_manager",
            "message": f"Risk manager approval needed: ${notional:,.0f} trade"
        })
        if not decision["approved"]:
            return {"messages": ["Blocked by risk manager"]}

        decision2 = interrupt({
            "level": "portfolio_manager",
            "message": f"PM final sign-off: ${notional:,.0f} trade"
        })
        if decision2["approved"]:
            return {"messages": ["PM approved"]}

    return {"messages": ["Trade rejected"], "pending_orders": []}
```

### State Editing (Manual Override)

```python
# Get current state
current = await graph.aget_state(config)

# Override portfolio positions (e.g., reconciliation)
await graph.aupdate_state(
    config,
    {"portfolio": {"AAPL": {"qty": 100, "avg_price": 178.50}}},
    as_node="portfolio_manager"  # attribute the edit to a node
)
```

---

## 3. Streaming

### Five Streaming Modes

```python
# 1. values -- full state after each node
async for state in graph.astream(inputs, config, stream_mode="values"):
    print(state)  # complete state dict

# 2. updates -- only the delta from each node
async for update in graph.astream(inputs, config, stream_mode="updates"):
    node_name = list(update.keys())[0]
    node_output = update[node_name]

# 3. messages -- LLM token streaming (most useful for UIs)
async for msg, metadata in graph.astream(inputs, config, stream_mode="messages"):
    if isinstance(msg, AIMessageChunk):
        print(msg.content, end="", flush=True)

# 4. events -- granular LangChain callback events
async for event in graph.astream_events(inputs, config, version="v2"):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")

# 5. custom -- emit your own data from nodes
from langgraph.types import StreamWriter

def analysis_node(state: TradingState, writer: StreamWriter):
    writer({"progress": "Fetching market data..."})
    data = fetch_market_data(state)
    writer({"progress": "Running technical analysis..."})
    signals = run_ta(data)
    writer({"progress": "Complete", "signals": signals})
    return {"trade_signals": signals}
```

### Multiple Stream Modes Simultaneously

```python
async for chunk in graph.astream(
    inputs, config,
    stream_mode=["updates", "messages", "custom"]
):
    mode, data = chunk
    if mode == "messages":
        # token-level LLM output
        pass
    elif mode == "updates":
        # node completion
        pass
    elif mode == "custom":
        # progress updates
        pass
```

### Trading Dashboard Streaming Pattern

```python
async def stream_trading_analysis(graph, query, config):
    """Stream analysis progress to a trading dashboard."""
    async for chunk in graph.astream(
        {"messages": [HumanMessage(content=query)]},
        config,
        stream_mode=["updates", "custom"]
    ):
        mode, data = chunk
        if mode == "custom":
            # Real-time progress: "Analyzing AAPL...", "Checking RSI..."
            await websocket.send_json({"type": "progress", "data": data})
        elif mode == "updates":
            node = list(data.keys())[0]
            await websocket.send_json({
                "type": "node_complete",
                "node": node,
                "data": data[node]
            })
```

---

## 4. Tool Calling & ToolNode

### Defining Trading Tools

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field

class MarketDataInput(BaseModel):
    symbol: str = Field(description="Ticker symbol, e.g. AAPL")
    timeframe: str = Field(default="1d", description="Candle timeframe: 1m, 5m, 1h, 1d")
    lookback: int = Field(default=30, description="Number of periods")

@tool(args_schema=MarketDataInput)
def get_market_data(symbol: str, timeframe: str = "1d", lookback: int = 30) -> dict:
    """Fetch OHLCV market data for a given symbol and timeframe."""
    # Implementation
    return {"symbol": symbol, "candles": [...], "volume": [...]}

@tool
def calculate_technical_indicators(symbol: str, indicators: list[str]) -> dict:
    """Calculate technical indicators (RSI, MACD, BB, EMA) for a symbol."""
    return {"symbol": symbol, "rsi": 45.2, "macd": {"signal": -0.5}}

@tool
def get_news_sentiment(symbol: str, hours: int = 24) -> dict:
    """Get recent news sentiment analysis for a symbol."""
    return {"symbol": symbol, "sentiment_score": 0.65, "articles": 12}

@tool
def place_order(symbol: str, side: str, qty: int, order_type: str = "market") -> dict:
    """Place a trading order. Requires human approval for large orders."""
    return {"order_id": "ORD-001", "status": "pending"}

@tool
def get_portfolio_positions() -> dict:
    """Get current portfolio positions and P&L."""
    return {"positions": [...], "total_pnl": 1250.00}
```

### ToolNode with Conditional Routing

```python
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.graph import StateGraph, MessagesState, START, END

tools = [
    get_market_data,
    calculate_technical_indicators,
    get_news_sentiment,
    place_order,
    get_portfolio_positions,
]

# Bind tools to LLM
llm = ChatOpenAI(model="gpt-4o").bind_tools(tools)

tool_node = ToolNode(tools)

def agent_node(state: MessagesState):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# Build graph
graph = StateGraph(MessagesState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges(
    "agent",
    tools_condition,  # routes to "tools" if last message has tool_calls, else END
)
graph.add_edge("tools", "agent")  # loop back after tool execution

app = graph.compile()
```

### Custom Tool Routing (Separate Dangerous Tools)

```python
def route_tools(state: MessagesState):
    """Route tool calls: order execution needs approval, others auto-execute."""
    last_msg = state["messages"][-1]
    if not hasattr(last_msg, "tool_calls") or not last_msg.tool_calls:
        return END

    for tc in last_msg.tool_calls:
        if tc["name"] == "place_order":
            return "approval_gate"  # human-in-the-loop
    return "tools"  # auto-execute safe tools

graph.add_conditional_edges("agent", route_tools, {
    "tools": "tools",
    "approval_gate": "approval_gate",
    END: END,
})
```

---

## 5. Map-Reduce / Parallel Execution

### The `Send()` API for Dynamic Fan-Out

```python
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    symbols: list[str]
    analysis_results: Annotated[list[dict], operator.add]
    final_report: str

class SymbolAnalysisState(TypedDict):
    symbol: str
    analysis_results: Annotated[list[dict], operator.add]

def fan_out_symbols(state: OverallState):
    """Dynamically create one analysis task per symbol."""
    return [
        Send("analyze_symbol", {"symbol": s})
        for s in state["symbols"]
    ]

def analyze_symbol(state: SymbolAnalysisState) -> dict:
    """Analyze a single symbol -- runs in parallel for each Send()."""
    symbol = state["symbol"]
    # Run technical analysis, sentiment, fundamentals...
    result = {
        "symbol": symbol,
        "signal": "BUY",
        "confidence": 0.78,
        "analysis": {
            "rsi": 35.2,
            "macd_signal": "bullish_crossover",
            "sentiment": 0.72,
        }
    }
    return {"analysis_results": [result]}

def synthesize_report(state: OverallState) -> dict:
    """Combine all parallel analyses into a final report."""
    results = state["analysis_results"]
    buy_signals = [r for r in results if r["signal"] == "BUY"]
    report = f"Analyzed {len(results)} symbols. {len(buy_signals)} BUY signals."
    return {"final_report": report}

# Build the map-reduce graph
graph = StateGraph(OverallState)
graph.add_node("analyze_symbol", analyze_symbol)
graph.add_node("synthesize", synthesize_report)

graph.add_conditional_edges(START, fan_out_symbols)  # dynamic fan-out
graph.add_edge("analyze_symbol", "synthesize")       # fan-in
graph.add_edge("synthesize", END)

app = graph.compile()

# Invoke
result = await app.ainvoke({
    "symbols": ["AAPL", "MSFT", "GOOGL", "AMZN", "NVDA"],
    "analysis_results": [],
})
```

### Multi-Dimensional Parallel Analysis

```python
def fan_out_analysis_dimensions(state: OverallState):
    """For each symbol, run multiple analysis types in parallel."""
    sends = []
    for symbol in state["symbols"]:
        for analysis_type in ["technical", "fundamental", "sentiment", "flow"]:
            sends.append(Send("run_analysis", {
                "symbol": symbol,
                "analysis_type": analysis_type,
            }))
    return sends
```

### Reducer Pattern for Aggregation

```python
from typing import Annotated
import operator

class TradingState(TypedDict):
    # operator.add merges lists from parallel nodes
    analysis_results: Annotated[list[dict], operator.add]
    trade_signals: Annotated[list[dict], operator.add]
    risk_alerts: Annotated[list[str], operator.add]

    # Custom reducer for portfolio (merge dicts)
    portfolio: Annotated[dict, merge_portfolios]

def merge_portfolios(existing: dict, update: dict) -> dict:
    """Custom reducer: merge portfolio updates without overwriting."""
    merged = {**existing}
    for symbol, position in update.items():
        if symbol in merged:
            merged[symbol] = {
                **merged[symbol],
                **position,
                "qty": merged[symbol].get("qty", 0) + position.get("qty_delta", 0)
            }
        else:
            merged[symbol] = position
    return merged
```

---

## 6. Adaptive RAG Patterns

### State Definition for RAG-Based Market Research

```python
from typing import TypedDict, Annotated, Literal
import operator

class ResearchState(TypedDict):
    question: str
    documents: Annotated[list, operator.add]
    generation: str
    web_search_needed: bool
    route: Literal["vectorstore", "web_search", "direct"]
    retry_count: int
```

### Query Router

```python
from pydantic import BaseModel, Field

class RouteQuery(BaseModel):
    """Route a user query to the most relevant data source."""
    datasource: Literal["vectorstore", "web_search", "direct"] = Field(
        description=(
            "vectorstore: for questions about historical patterns, SEC filings, earnings. "
            "web_search: for real-time market news, current prices, breaking events. "
            "direct: for general financial knowledge questions."
        )
    )

structured_llm = llm.with_structured_output(RouteQuery)

def route_question(state: ResearchState) -> str:
    question = state["question"]
    result = structured_llm.invoke(
        f"Route this financial research question: {question}"
    )
    return result.datasource
```

### Document Grader

```python
class GradeDocuments(BaseModel):
    """Grade whether a retrieved document is relevant to the question."""
    binary_score: Literal["yes", "no"] = Field(
        description="Is the document relevant to the financial question?"
    )

doc_grader = llm.with_structured_output(GradeDocuments)

def grade_documents(state: ResearchState) -> dict:
    question = state["question"]
    documents = state["documents"]

    filtered = []
    web_search_needed = False

    for doc in documents:
        score = doc_grader.invoke(
            f"Question: {question}\nDocument: {doc.page_content}"
        )
        if score.binary_score == "yes":
            filtered.append(doc)
        else:
            web_search_needed = True

    return {
        "documents": filtered,
        "web_search_needed": web_search_needed,
    }
```

### Hallucination & Answer Grader

```python
class GradeHallucinations(BaseModel):
    binary_score: Literal["yes", "no"] = Field(
        description="Is the answer grounded in the provided documents?"
    )

class GradeAnswer(BaseModel):
    binary_score: Literal["yes", "no"] = Field(
        description="Does the answer address the question?"
    )

def check_generation(state: ResearchState) -> str:
    """Grade the generated answer for hallucination and relevance."""
    # Check hallucination
    hallucination_grade = hallucination_grader.invoke({
        "documents": state["documents"],
        "generation": state["generation"],
    })
    if hallucination_grade.binary_score == "no":
        return "regenerate"  # hallucinated, try again

    # Check answer relevance
    answer_grade = answer_grader.invoke({
        "question": state["question"],
        "generation": state["generation"],
    })
    if answer_grade.binary_score == "yes":
        return "useful"
    return "not_useful"  # triggers web search or query rewrite
```

### Full Adaptive RAG Graph

```python
graph = StateGraph(ResearchState)

graph.add_node("retrieve", retrieve_node)
graph.add_node("grade_documents", grade_documents)
graph.add_node("generate", generate_node)
graph.add_node("web_search", web_search_node)
graph.add_node("rewrite_query", rewrite_query_node)

# Routing
graph.add_conditional_edges(START, route_question, {
    "vectorstore": "retrieve",
    "web_search": "web_search",
    "direct": "generate",
})

graph.add_edge("retrieve", "grade_documents")

graph.add_conditional_edges("grade_documents", decide_to_generate, {
    "generate": "generate",
    "web_search": "web_search",
})

graph.add_conditional_edges("generate", check_generation, {
    "useful": END,
    "not_useful": "rewrite_query",
    "regenerate": "generate",
})

graph.add_edge("web_search", "generate")
graph.add_edge("rewrite_query", "retrieve")

app = graph.compile()
```

---

## 7. ReAct Agent Pattern

### Using `create_react_agent` (Prebuilt)

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Simple creation -- handles the Reason-Act-Observe loop internally
trading_agent = create_react_agent(
    model=llm,
    tools=[
        get_market_data,
        calculate_technical_indicators,
        get_news_sentiment,
        get_portfolio_positions,
    ],
    prompt="You are a senior trading analyst. Analyze markets methodically.",
    checkpointer=MemorySaver(),
)

result = await trading_agent.ainvoke(
    {"messages": [HumanMessage(content="Analyze NVDA for a swing trade")]},
    config={"configurable": {"thread_id": "analyst-session-1"}}
)
```

### Custom ReAct Agent (Full Control)

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode, tools_condition

class TradingAgentState(MessagesState):
    """Extended state for trading-specific context."""
    iteration_count: int
    max_iterations: int

def should_continue(state: TradingAgentState) -> str:
    """Check tool calls AND iteration limit."""
    if state["iteration_count"] >= state.get("max_iterations", 10):
        return END  # prevent infinite loops
    last_msg = state["messages"][-1]
    if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
        return "tools"
    return END

def agent_node(state: TradingAgentState):
    system_prompt = """You are a quantitative trading analyst.
    Follow this analysis framework:
    1. Fetch market data for the requested symbol
    2. Calculate key technical indicators (RSI, MACD, Bollinger Bands)
    3. Check news sentiment
    4. Synthesize findings into a trade recommendation
    Always show your reasoning before making a recommendation."""

    messages = [SystemMessage(content=system_prompt)] + state["messages"]
    response = llm.invoke(messages)
    return {
        "messages": [response],
        "iteration_count": state.get("iteration_count", 0) + 1,
    }

tools = [get_market_data, calculate_technical_indicators, get_news_sentiment]
tool_node = ToolNode(tools)

graph = StateGraph(TradingAgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {
    "tools": "tools",
    END: END,
})
graph.add_edge("tools", "agent")

trading_agent = graph.compile(checkpointer=MemorySaver())
```

### Multi-Agent ReAct with Supervisor

```python
from langgraph.types import Command

class SupervisorState(TypedDict):
    messages: Annotated[list, operator.add]
    next_agent: str
    analysis_results: Annotated[list[dict], operator.add]

def supervisor_node(state: SupervisorState):
    """Route to specialized agents based on the task."""
    response = supervisor_llm.invoke(state["messages"])

    # Structured output to pick next agent
    routing = route_llm.with_structured_output(RouteDecision).invoke(
        state["messages"]
    )

    return Command(
        goto=routing.next_agent,
        update={"next_agent": routing.next_agent}
    )

# Each specialist is a subgraph
graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor_node)
graph.add_node("technical_analyst", technical_analyst_subgraph)
graph.add_node("fundamental_analyst", fundamental_analyst_subgraph)
graph.add_node("risk_manager", risk_manager_subgraph)
graph.add_node("execution_agent", execution_agent_subgraph)

graph.add_edge(START, "supervisor")
for agent in ["technical_analyst", "fundamental_analyst", "risk_manager", "execution_agent"]:
    graph.add_edge(agent, "supervisor")

graph.add_conditional_edges("supervisor", lambda s: s["next_agent"], {
    "technical_analyst": "technical_analyst",
    "fundamental_analyst": "fundamental_analyst",
    "risk_manager": "risk_manager",
    "execution_agent": "execution_agent",
    "FINISH": END,
})
```

---

## 8. LangGraph Server / Platform

### Architecture Overview

LangGraph Platform provides a production deployment layer with:
- **Assistants API**: CRUD for agent configurations
- **Threads API**: Manage conversation threads with persistence
- **Runs API**: Execute graph invocations (sync, async, streaming)
- **Cron API**: Scheduled agent runs (e.g., daily portfolio rebalancing)
- **Store API**: Cross-thread persistent memory

### `langgraph.json` Configuration

```json
{
  "dependencies": ["."],
  "graphs": {
    "trading_agent": "./src/agent.py:graph",
    "research_agent": "./src/research.py:graph",
    "risk_monitor": "./src/risk.py:graph"
  },
  "env": ".env",
  "python_version": "3.11"
}
```

### LangGraph SDK Client Usage

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:8123")

# Create a thread (trading session)
thread = await client.threads.create()

# Create a run with streaming
async for chunk in client.runs.stream(
    thread["thread_id"],
    "trading_agent",
    input={"messages": [{"role": "user", "content": "Analyze AAPL"}]},
    config={"configurable": {"user_id": "trader-001"}},
    stream_mode=["updates", "messages"],
):
    print(chunk)

# Schedule daily portfolio review
cron = await client.crons.create(
    "trading_agent",
    schedule="0 9 * * 1-5",  # weekdays at 9am
    input={"messages": [{"role": "user", "content": "Daily portfolio review"}]},
)
```

### Double-Texting Handling

```python
# When a user sends a new message while a run is in progress:
# Option 1: Reject -- return error
# Option 2: Enqueue -- queue the new input
# Option 3: Interrupt -- cancel current run, start new one
# Option 4: Rollback -- revert to pre-run state, start new one

async for chunk in client.runs.stream(
    thread_id,
    "trading_agent",
    input=new_input,
    multitask_strategy="interrupt",  # cancel current, start new
):
    pass
```

### Store API for Cross-Thread Memory

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# Store user preferences across sessions
await store.aput(
    ("user_preferences", "trader-001"),
    "risk_profile",
    {"max_position_size": 10000, "max_drawdown": 0.05, "preferred_sectors": ["tech"]}
)

# Retrieve in any thread
profile = await store.aget(("user_preferences", "trader-001"), "risk_profile")

# Compile graph with store
graph = workflow.compile(checkpointer=checkpointer, store=store)
```

---

## 9. Configuration Passing to Tools

### RunnableConfig Injection

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool

@tool
def get_market_data_authenticated(
    symbol: str,
    config: RunnableConfig,  # automatically injected, invisible to LLM
) -> dict:
    """Fetch market data using the caller's API credentials."""
    user_id = config.get("configurable", {}).get("user_id")
    api_key = config.get("configurable", {}).get("market_data_api_key")
    broker = config.get("configurable", {}).get("broker", "alpaca")

    # Use injected config for authenticated API calls
    client = MarketDataClient(api_key=api_key, broker=broker)
    return client.get_ohlcv(symbol)

@tool
def place_order_with_context(
    symbol: str,
    side: str,
    qty: int,
    config: RunnableConfig,
) -> dict:
    """Place order using the trader's account context."""
    account_id = config["configurable"]["account_id"]
    risk_limit = config["configurable"].get("max_order_size", 10000)

    if qty * get_price(symbol) > risk_limit:
        return {"error": f"Order exceeds risk limit of ${risk_limit}"}

    return execute_order(account_id, symbol, side, qty)
```

### Invoking with Config

```python
config = {
    "configurable": {
        "thread_id": "session-001",
        "user_id": "trader-42",
        "account_id": "ACC-12345",
        "market_data_api_key": os.environ["MARKET_DATA_KEY"],
        "broker": "alpaca",
        "max_order_size": 50000,
        "risk_profile": "moderate",
    }
}

result = await graph.ainvoke(
    {"messages": [HumanMessage(content="Buy 100 shares of AAPL")]},
    config=config,
)
```

### InjectedState Pattern (Access Full Graph State in Tools)

```python
from langgraph.prebuilt import InjectedState
from typing import Annotated

@tool
def check_portfolio_risk(
    proposed_trade: dict,
    state: Annotated[TradingState, InjectedState],  # full graph state injected
) -> dict:
    """Check if a proposed trade violates portfolio risk constraints."""
    portfolio = state["portfolio"]
    risk_metrics = state["risk_metrics"]

    current_exposure = sum(
        pos["qty"] * pos["current_price"]
        for pos in portfolio.values()
    )
    new_exposure = current_exposure + proposed_trade["qty"] * proposed_trade["price"]

    return {
        "within_limits": new_exposure < risk_metrics.get("max_exposure", 1_000_000),
        "current_exposure": current_exposure,
        "projected_exposure": new_exposure,
    }
```

---

## 10. Complete Trading System Architecture

### Full Multi-Agent Graph

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send, Command, interrupt
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.prebuilt import ToolNode
from typing import TypedDict, Annotated, Literal
import operator

# ── State Definition ──

class TradingSystemState(TypedDict):
    messages: Annotated[list, operator.add]
    symbols: list[str]
    market_regime: str
    analysis_results: Annotated[list[dict], operator.add]
    trade_signals: Annotated[list[dict], operator.add]
    risk_assessment: dict
    approved_orders: list[dict]
    execution_results: Annotated[list[dict], operator.add]
    session_metadata: dict

# ── Nodes ──

def market_regime_detector(state: TradingSystemState) -> dict:
    """Classify current market regime: bull/bear/sideways/volatile."""
    # Uses VIX, breadth, trend indicators
    regime = classify_regime(state["symbols"])
    return {"market_regime": regime}

def fan_out_analysis(state: TradingSystemState):
    """Fan out: one analysis task per symbol."""
    return [
        Send("symbol_analyst", {"symbol": s, "market_regime": state["market_regime"]})
        for s in state["symbols"]
    ]

def symbol_analyst(state: dict) -> dict:
    """Deep analysis of a single symbol (runs in parallel)."""
    symbol = state["symbol"]
    regime = state["market_regime"]
    # Technical + fundamental + sentiment analysis
    return {"analysis_results": [{
        "symbol": symbol,
        "signal": "BUY",
        "confidence": 0.82,
        "regime_adjusted": True,
    }]}

def signal_generator(state: TradingSystemState) -> dict:
    """Generate trade signals from aggregated analyses."""
    signals = []
    for analysis in state["analysis_results"]:
        if analysis["confidence"] > 0.7:
            signals.append({
                "symbol": analysis["symbol"],
                "action": analysis["signal"],
                "size": calculate_position_size(analysis, state["risk_assessment"]),
            })
    return {"trade_signals": signals}

def risk_manager(state: TradingSystemState) -> dict:
    """Evaluate portfolio-level risk for proposed trades."""
    assessment = evaluate_risk(
        signals=state["trade_signals"],
        portfolio=state.get("portfolio", {}),
        regime=state["market_regime"],
    )
    return {"risk_assessment": assessment}

def human_approval(state: TradingSystemState) -> dict:
    """Interrupt for human approval on significant trades."""
    signals = state["trade_signals"]
    risk = state["risk_assessment"]

    if risk.get("risk_level") == "low":
        return {"approved_orders": signals}  # auto-approve low risk

    decision = interrupt({
        "signals": signals,
        "risk_assessment": risk,
        "message": f"Approve {len(signals)} trades? Risk level: {risk['risk_level']}"
    })

    if decision["approved"]:
        approved = [s for s in signals if s["symbol"] in decision.get("symbols", [s["symbol"] for s in signals])]
        return {"approved_orders": approved}
    return {"approved_orders": []}

def execution_engine(state: TradingSystemState) -> dict:
    """Execute approved orders."""
    results = []
    for order in state["approved_orders"]:
        result = execute_trade(order)
        results.append(result)
    return {"execution_results": results}

# ── Graph Construction ──

graph = StateGraph(TradingSystemState)

graph.add_node("regime_detector", market_regime_detector)
graph.add_node("symbol_analyst", symbol_analyst)
graph.add_node("signal_generator", signal_generator)
graph.add_node("risk_manager", risk_manager)
graph.add_node("human_approval", human_approval)
graph.add_node("execution", execution_engine)

# Flow
graph.add_edge(START, "regime_detector")
graph.add_conditional_edges("regime_detector", fan_out_analysis)  # parallel fan-out
graph.add_edge("symbol_analyst", "signal_generator")              # fan-in
graph.add_edge("signal_generator", "risk_manager")
graph.add_edge("risk_manager", "human_approval")

def route_after_approval(state: TradingSystemState) -> str:
    if state["approved_orders"]:
        return "execution"
    return END

graph.add_conditional_edges("human_approval", route_after_approval, {
    "execution": "execution",
    END: END,
})
graph.add_edge("execution", END)

# ── Compile with Production Checkpointer ──

async def create_trading_system():
    checkpointer = AsyncPostgresSaver.from_conn_string(
        "postgresql://user:pass@localhost/trading"
    )
    await checkpointer.setup()
    return graph.compile(checkpointer=checkpointer)
```

---

## 11. Key Best Practices for Trading Systems

### State Design
- Use `Annotated[list, operator.add]` for any field that accumulates results from parallel nodes
- Keep state flat where possible; deeply nested state complicates reducers
- Separate ephemeral state (current analysis) from persistent state (portfolio, P&L)

### Checkpointing
- Use PostgreSQL in production for durability and queryability
- Every trade decision should be checkpointed for audit trails
- Use `thread_id` per trading session, `checkpoint_ns` for sub-workflows

### Human-in-the-Loop
- Use `interrupt()` (not `interrupt_before`/`interrupt_after`) for cleaner code
- Implement tiered approval: auto-approve small trades, require human for large ones
- Always persist state before interrupts so the system survives restarts

### Streaming
- Use `stream_mode=["updates", "custom"]` for dashboards
- Use `stream_mode="messages"` for chat-based trading interfaces
- Emit custom progress events from long-running analysis nodes via `StreamWriter`

### Map-Reduce
- Use `Send()` for dynamic parallelism (variable number of symbols)
- Each parallel node should return data via `Annotated[list, operator.add]` reducers
- Keep parallel nodes stateless and idempotent

### Tool Design
- Use `RunnableConfig` injection for API keys, account IDs, risk limits
- Use `InjectedState` when tools need access to portfolio state
- Separate read-only tools (market data) from write tools (order execution)
- Route dangerous tools through approval gates

### Error Handling
- Implement retry logic in tool nodes for transient API failures
- Use `max_iterations` guards in ReAct loops to prevent runaway agents
- Log all state transitions for debugging and compliance

---

## Sources

### Persistence & Checkpointing
- [Persistence in LangGraph -- Deep Practical Guide](https://pub.towardsai.net/persistence-in-langgraph-deep-practical-guide-36dc4c452c3b)
- [Mastering LangGraph Checkpointing: Best Practices](https://sparkco.ai/blog/mastering-langgraph-checkpointing-best-practices-for-2025)
- [LangGraph Memory Deep Dive](https://medium.com/@sahin.samia/agentic-ai-series-12-langgraph-memory-deep-dive-from-short-term-checkpoints-to-self-updating-effafab052f7)

### Human-in-the-Loop
- [LangGraph HITL Concepts (GitHub source)](https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/human_in_the_loop.md)
- [HITL Patterns: Approve, Reject, Edit](https://medium.com/the-advanced-school-of-ai/human-in-the-loop-in-langgraph-approve-or-reject-pattern-fcf6ba0c5990)
- [Interrupts and Commands in LangGraph](https://dev.to/jamesbmour/interrupts-and-commands-in-langgraph-building-human-in-the-loop-workflows-4ngl)
- [Architecting HITL Agents](https://medium.com/data-science-collective/architecting-human-in-the-loop-agents-interrupts-persistence-and-state-management-in-langgraph-fa36c9663d6f)

### Streaming
- [LangGraph Streaming 101: 5 Modes](https://dev.to/sreeni5018/langgraph-streaming-101-5-modes-to-build-responsive-ai-applications-4p3f)
- [Mastering LangGraph Streaming](https://sparkco.ai/blog/mastering-langgraph-streaming-advanced-techniques-and-best-practices)

### Tool Calling
- [Introduction to ToolNode](https://medium.com/@vivekvjnk/introduction-to-tool-use-with-langgraphs-toolnode-0121f3c8c323)
- [Building Tool Calling Agents](https://sangeethasaravanan.medium.com/building-tool-calling-agents-with-langgraph-a-complete-guide-ebdcdea8f475)
- [How to access RunnableConfig from a tool](https://python.langchain.com/docs/how_to/tool_configure)

### Map-Reduce
- [LangGraph Send() API](https://medium.com/@vishy2k5/langgraph-send-api-7aaab56bc6b8)
- [LangGraph Parallel Fan-Out/Fan-In](https://markaicode.com/langgraph-parallel-fan-out-fan-in/)
- [Scaling with Map-Reduce](https://medium.com/@welcome2suparna/from-theory-to-code-mastering-agentic-workflows-in-langgraph-part-2-scaling-with-map-reduce-f8331238a84a)

### Adaptive RAG
- [Agentic RAG: Query Routing, Document Grading, Query Rewriting](https://sajalsharma.com/posts/comprehensive-agentic-rag/)
- [Build a custom RAG agent (Official)](https://docs.langchain.com/oss/python/langgraph/agentic-rag)
- [Guide to Adaptive RAG Systems](https://www.analyticsvidhya.com/blog/2025/03/adaptive-rag-systems-with-langgraph/)

### ReAct Agent
- [Build ReAct Agent from Scratch](https://pub.towardsai.net/agentic-ai-build-react-agent-using-langgraph-facac8ae6031)
- [ReAct Agents Explained: Step-by-Step](https://www.intelligentmachines.blog/post/react-agents-explained-a-step-by-step-implementation-using-langgraph)

### LangGraph Server/Platform
- [LangGraph Platform GA Announcement](https://blog.langchain.com/langgraph-platform-ga/)
- [Why LangGraph Platform](https://blog.langchain.com/why-langgraph-platform)
- [LangGraph Server Overview](https://langchain-ai.github.io/langgraph/concepts/langgraph_server/)

### Configuration Passing
- [How to access RunnableConfig from a tool](https://python.langchain.com/docs/how_to/tool_configure)
- [LangGraph context API discussion](https://github.com/langchain-ai/langgraph/issues/5023)
