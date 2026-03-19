# NOFX 港股/A股 LangGraph 重构 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace NOFX’s crypto-market trading stack with a Hong Kong/A-share broker-based stack orchestrated by LangGraph, while preserving reusable Go control-plane and frontend assets.

**Architecture:** Keep Go as the control plane for auth, config, REST gateway, and web integration. Add a Python `orchestrator/` service for LangGraph workflows, market-data adapters, broker adapters, graph runs, approvals, checkpointing, and streaming. Migrate data model, API semantics, and frontend language from exchange/coin/futures to broker/instrument/universe/equity, then remove crypto modules after paper-trading validation.

**Tech Stack:** Go, Gin, SQLite, Python 3.12, LangGraph, React 18, TypeScript, Vite, SWR, Zustand, Docker Compose.

---

## Backlog Overview

| Epic | Priority | Depends On | Outcome |
| --- | --- | --- | --- |
| E0 | P0 | - | 架构与供应商决策冻结 |
| E1 | P0 | E0 | 新股票域模型与数据库落地 |
| E2 | P0 | E0 | Python LangGraph orchestrator 骨架可运行 |
| E3 | P0 | E1,E2 | 港/A 市场数据层替换完成 |
| E4 | P0 | E1,E2 | Broker 执行层替换完成 |
| E5 | P0 | E2,E3,E4 | Research/Risk/Execution 主图闭环 |
| E6 | P1 | E1,E5 | 前端迁移到 broker / strategy agent / graph run |
| E7 | P1 | E5,E6 | Streaming、审批、回放完整 |
| E8 | P1 | E3-E7 | Cutover 与 crypto 删除 |

## Epic E0: Freeze Architecture and Provider Selection

**Files:**
- Create: `docs/adr/ADR-001-go-control-plane-python-orchestrator.md`
- Create: `docs/adr/ADR-002-hk-ashare-provider-selection.md`
- Create: `docs/adr/ADR-003-domain-model-v1.md`
- Create: `orchestrator/README.md`
- Modify: `docker-compose.yml`
- Modify: `README.md`

**Step 1: Write the failing architecture review checklist**
- Create a checklist in `docs/adr/ADR-001-go-control-plane-python-orchestrator.md` covering service boundaries, failure domains, ownership, and deployment responsibilities.

**Step 2: Capture provider comparison**
- Compare at least one broker/trading channel and one research/market-data channel in `docs/adr/ADR-002-hk-ashare-provider-selection.md`.
- Include connectivity, market coverage, sandbox availability, order APIs, session/calendar support, and operational risk.

**Step 3: Define domain vocabulary**
- In `docs/adr/ADR-003-domain-model-v1.md`, freeze terminology: `BrokerAccount`, `Instrument`, `UniverseConfig`, `StrategyAgent`, `RiskPolicy`, `GraphRun`, `OrderIntent`.

**Step 4: Document service topology**
- Add `orchestrator/README.md` describing how Go and Python services communicate.
- Update `docker-compose.yml` with a placeholder `orchestrator` service.

**Step 5: Verify**
- Run: `rg -n "exchange|coin pool|futures leverage" docs/adr orchestrator/README.md README.md`
- Expected: ADRs explicitly mark these as legacy crypto terms or migration targets, not future-state names.

**Acceptance Criteria:**
- Three ADRs exist and point to the same target architecture.
- Provider decision criteria are explicit enough to start PoC work.
- README mentions the orchestrator service at least as planned architecture.

## Epic E1: Introduce Stock-Market Domain Models and Database Migration

**Files:**
- Modify: `config/database.go`
- Create: `migrations/003_broker_accounts.sql`
- Create: `migrations/004_strategy_agents.sql`
- Create: `migrations/005_risk_policies.sql`
- Create: `migrations/006_graph_runs.sql`
- Create: `migrations/007_order_intents.sql`
- Create: `config/database_broker_test.go`

**Step 1: Write failing migration tests**
- Add tests in `config/database_broker_test.go` for creating, reading, and listing `broker_accounts`, `strategy_agents`, and `risk_policies`.

**Step 2: Add new SQL migrations**
- Create normalized tables for broker accounts, strategy agents, risk policies, graph runs, and order intents.
- Keep old `exchanges` and `traders` readable during transition.

**Step 3: Extend database access layer**
- Add CRUD methods for new entities in `config/database.go`.
- Add compatibility helpers to map legacy trader config to new strategy-agent config during cutover.

**Step 4: Re-run targeted tests**
- Run: `go test ./config -run 'Broker|StrategyAgent|RiskPolicy' -v`
- Expected: PASS

**Step 5: Verify schema language**
- Run: `rg -n "hyperliquid|binance|bybit|lighter|aster|btc_eth_leverage|altcoin_leverage" config/database.go migrations`
- Expected: legacy fields remain only in compatibility sections, not in new tables.

**Acceptance Criteria:**
- New tables exist and are exercised by tests.
- New records can be created without any crypto-only field.
- Legacy tables are still readable during migration.

## Epic E2: Scaffold the Python LangGraph Orchestrator

**Files:**
- Create: `orchestrator/pyproject.toml`
- Create: `orchestrator/src/nofx_orchestrator/__init__.py`
- Create: `orchestrator/src/nofx_orchestrator/state.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/main.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/session_gate.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/universe_builder.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/market_context.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/research_decision.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/risk_policy.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/execution.py`
- Create: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/reconcile_archive.py`
- Create: `orchestrator/tests/test_state_schema.py`
- Create: `orchestrator/tests/test_main_graph.py`

**Step 1: Write failing orchestrator tests**
- `test_state_schema.py` should assert the presence of core state slices.
- `test_main_graph.py` should assert the compiled graph contains the required subgraph entry points.

**Step 2: Create package skeleton**
- Add the Python package, dependencies, and minimal state schema.
- Stub all subgraphs with deterministic placeholder nodes returning structured state deltas.

**Step 3: Build the main graph**
- Wire subgraphs in the order `session_gate -> universe -> market_context -> research -> risk -> execution -> reconcile`.
- Add placeholder conditional routing for market-closed and approval-required paths.

**Step 4: Re-run targeted tests**
- Run: `cd orchestrator && pytest tests/test_state_schema.py tests/test_main_graph.py -q`
- Expected: PASS

**Step 5: Verify source layout**
- Run: `find orchestrator/src/nofx_orchestrator -maxdepth 3 -type f | sort`
- Expected: Graph and subgraph files exist under stable paths.

**Acceptance Criteria:**
- A compiled graph exists.
- State schema is centralized.
- Subgraphs can be imported and unit-tested independently.

## Epic E3: Replace Crypto Market Data with HK/A Market Data Adapters

**Files:**
- Create: `orchestrator/src/nofx_orchestrator/adapters/marketdata/base.py`
- Create: `orchestrator/src/nofx_orchestrator/adapters/marketdata/futu_adapter.py`
- Create: `orchestrator/src/nofx_orchestrator/adapters/marketdata/research_adapter.py`
- Create: `orchestrator/src/nofx_orchestrator/domain/instruments.py`
- Create: `orchestrator/src/nofx_orchestrator/domain/sessions.py`
- Create: `orchestrator/tests/test_market_sessions.py`
- Create: `orchestrator/tests/test_market_snapshot.py`
- Modify: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/market_context.py`

**Step 1: Write failing adapter tests**
- Add tests for session detection, instrument metadata normalization, and market snapshot completeness.

**Step 2: Add stock-domain DTOs**
- Model `Instrument`, `SessionState`, `MarketSnapshot`, `CorporateActionView`, and `TradabilityStatus`.

**Step 3: Implement adapters**
- `futu_adapter.py` becomes the trading-grade quote/session source.
- `research_adapter.py` supplies optional research-only enrichment.

**Step 4: Wire market-context subgraph**
- Replace crypto-specific assumptions with exchange calendar, lot size, tick size, suspension, and permission checks.

**Step 5: Re-run tests**
- Run: `cd orchestrator && pytest tests/test_market_sessions.py tests/test_market_snapshot.py -q`
- Expected: PASS

**Acceptance Criteria:**
- Market context reflects HK/A trading sessions.
- No dependency on funding rate, OI, or 24/7 perpetual assumptions remains in new adapter code.

## Epic E4: Replace Exchange Traders with Broker Adapters

**Files:**
- Create: `orchestrator/src/nofx_orchestrator/adapters/brokers/base.py`
- Create: `orchestrator/src/nofx_orchestrator/adapters/brokers/paper_broker.py`
- Create: `orchestrator/src/nofx_orchestrator/adapters/brokers/futu_broker.py`
- Create: `orchestrator/src/nofx_orchestrator/domain/orders.py`
- Create: `orchestrator/tests/test_paper_broker.py`
- Create: `orchestrator/tests/test_order_intent_mapping.py`
- Modify: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/execution.py`

**Step 1: Write failing broker tests**
- Cover account snapshot, submit order, cancel order, partial fill, reject, and idempotency.

**Step 2: Define broker-neutral contracts**
- Introduce `OrderIntent`, `BrokerOrder`, `FillEvent`, `PositionSnapshot`, and `AccountSnapshot`.

**Step 3: Implement paper broker**
- Support deterministic fills for local E2E and replay.

**Step 4: Implement first real broker adapter**
- Add a broker adapter with explicit translation from `OrderIntent` to broker request DTOs.

**Step 5: Re-run tests**
- Run: `cd orchestrator && pytest tests/test_paper_broker.py tests/test_order_intent_mapping.py -q`
- Expected: PASS

**Acceptance Criteria:**
- Execution subgraph can use broker-neutral interfaces.
- Real and paper brokers share the same contract.

## Epic E5: Rebuild Research, Risk, and Execution Workflow

**Files:**
- Create: `orchestrator/src/nofx_orchestrator/nodes/research.py`
- Create: `orchestrator/src/nofx_orchestrator/nodes/risk.py`
- Create: `orchestrator/src/nofx_orchestrator/nodes/reconcile.py`
- Create: `orchestrator/src/nofx_orchestrator/prompts/research_system.txt`
- Create: `orchestrator/src/nofx_orchestrator/prompts/risk_review.txt`
- Create: `orchestrator/tests/test_risk_routing.py`
- Create: `orchestrator/tests/test_research_schema.py`
- Modify: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/research_decision.py`
- Modify: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/risk_policy.py`
- Modify: `orchestrator/src/nofx_orchestrator/graphs/subgraphs/reconcile_archive.py`

**Step 1: Write failing schema tests**
- `test_research_schema.py` should reject free-form text and require structured research output.
- `test_risk_routing.py` should cover pass, deny, and approval-required branches.

**Step 2: Add structured outputs**
- Force LLM output into research summary and order-intent schemas.
- Keep deterministic risk limits in code.

**Step 3: Implement routing and interrupt points**
- Add approval interrupt for large orders or rule overrides.
- Add archival branch for denied or deferred actions.

**Step 4: Re-run tests**
- Run: `cd orchestrator && pytest tests/test_research_schema.py tests/test_risk_routing.py -q`
- Expected: PASS

**Acceptance Criteria:**
- Research output is structured.
- Risk decisions are code-controlled.
- Approval path is explicit and test-covered.

## Epic E6: Add Go Control-Plane APIs for New Resources

**Files:**
- Create: `api/broker_accounts.go`
- Create: `api/strategy_agents.go`
- Create: `api/graph_runs.go`
- Create: `api/approvals.go`
- Modify: `api/server.go`
- Modify: `manager/trader_manager.go`
- Create: `api/broker_accounts_test.go`
- Create: `api/strategy_agents_test.go`

**Step 1: Write failing API tests**
- Cover CRUD for broker accounts and strategy agents.
- Cover list/detail endpoints for graph runs and approvals.

**Step 2: Split handlers out of `api/server.go`**
- Keep route registration in `api/server.go`.
- Move resource-specific logic into dedicated handler files.

**Step 3: Replace manager responsibility**
- Rework `manager/trader_manager.go` from in-process crypto trader launcher into graph-run coordinator / orchestrator client.

**Step 4: Re-run tests**
- Run: `go test ./api ./manager -run 'Broker|StrategyAgent|GraphRun|Approval' -v`
- Expected: PASS

**Acceptance Criteria:**
- Go API exposes broker-account and strategy-agent resources.
- Manager no longer creates exchange-specific crypto traders on the new path.

## Epic E7: Migrate Frontend to Broker / Strategy Agent / Graph Runs

**Files:**
- Create: `web/src/components/brokers/BrokerConfigModal.tsx`
- Create: `web/src/components/strategy/StrategyAgentConfigModal.tsx`
- Create: `web/src/pages/GraphRunsPage.tsx`
- Create: `web/src/pages/ApprovalCenterPage.tsx`
- Modify: `web/src/components/AITradersPage.tsx`
- Modify: `web/src/components/traders/ExchangeConfigModal.tsx`
- Modify: `web/src/components/TraderConfigModal.tsx`
- Modify: `web/src/lib/api.ts`
- Modify: `web/src/types.ts`
- Modify: `web/src/routes/index.tsx`
- Create: `web/src/components/brokers/BrokerConfigModal.test.tsx`
- Create: `web/src/components/strategy/StrategyAgentConfigModal.test.tsx`

**Step 1: Write failing component tests**
- Add tests ensuring the broker form does not render crypto-wallet-only labels.
- Add tests ensuring strategy-agent config uses universe/risk terminology.

**Step 2: Create replacement components**
- Build `BrokerConfigModal` and `StrategyAgentConfigModal` using existing modal/layout patterns.

**Step 3: Rewire page semantics**
- Update `AITradersPage.tsx` to manage broker accounts, strategy agents, and graph runs instead of exchange-centric workflows.

**Step 4: Re-run targeted tests**
- Run: `cd web && npm test -- BrokerConfigModal StrategyAgentConfigModal`
- Expected: PASS

**Step 5: Build UI**
- Run: `cd web && npm run build`
- Expected: PASS

**Acceptance Criteria:**
- Main UI no longer defaults to exchange/coin/futures wording.
- Users can configure broker accounts and strategy agents from the main path.

## Epic E8: Add Streaming, Approval Resolution, and Replay

**Files:**
- Create: `api/graph_stream.go`
- Create: `orchestrator/src/nofx_orchestrator/streaming/events.py`
- Create: `orchestrator/src/nofx_orchestrator/replay/service.py`
- Create: `orchestrator/tests/test_streaming_events.py`
- Create: `orchestrator/tests/test_replay_service.py`
- Modify: `api/server.go`
- Modify: `web/src/pages/GraphRunsPage.tsx`
- Modify: `web/src/pages/ApprovalCenterPage.tsx`

**Step 1: Write failing orchestrator tests**
- Cover event taxonomy, replay metadata, and approval lifecycle.

**Step 2: Implement event schema**
- Standardize `run.started`, `node.started`, `approval.required`, `order.submitted`, `run.failed`, and `run.completed`.

**Step 3: Add Go-side proxy or bridge**
- Expose graph-stream events to the frontend via REST + SSE/WebSocket bridge.

**Step 4: Re-run tests**
- Run: `cd orchestrator && pytest tests/test_streaming_events.py tests/test_replay_service.py -q`
- Expected: PASS

**Acceptance Criteria:**
- Frontend can observe graph progress.
- Operators can resolve approvals and replay past runs.

## Epic E9: Cut Over and Remove Crypto Support

**Files:**
- Modify: `main.go`
- Modify: `README.md`
- Modify: `docs/architecture/README.md`
- Modify: `docs/architecture/README.zh-CN.md`
- Delete: `trader/binance_futures.go`
- Delete: `trader/bybit_trader.go`
- Delete: `trader/hyperliquid_trader.go`
- Delete: `trader/aster_trader.go`
- Delete: `trader/lighter_trader.go`
- Delete: `trader/lighter_trader_v2.go`
- Delete: `pool/coin_pool.go`
- Delete or archive: crypto-market-specific docs under `docs/getting-started/`

**Step 1: Write a cutover checklist**
- Enumerate every crypto-only module, route, doc page, translation key, and test file that must be removed or archived.

**Step 2: Disable new-path fallbacks**
- Remove legacy creation flows that still point to exchange-specific traders.

**Step 3: Delete crypto main-path code**
- Delete trader adapters and coin-pool logic only after paper-trading validation passes.

**Step 4: Run repository verification**
- Run: `go test ./...`
- Run: `cd orchestrator && pytest -q`
- Run: `cd web && npm test && npm run build`
- Expected: PASS

**Step 5: Run terminology audit**
- Run: `rg -n "binance|bybit|hyperliquid|lighter|aster|coin pool|funding rate|open interest|BTCUSDT|ETHUSDT" .`
- Expected: matches only in archived migration notes or explicitly deprecated docs.

**Acceptance Criteria:**
- The active code path is stock-market only.
- Docs, API, frontend, and runtime no longer present crypto support as a primary feature.

## Cross-Cutting Checklist

- [ ] Every new API resource has request/response contract tests
- [ ] Every subgraph has at least one focused unit test
- [ ] Every broker call path has idempotency and failure mapping
- [ ] Every frontend rename removes crypto wording, not just field labels
- [ ] Every cutover step keeps paper-trading validation runnable

## Suggested Issue Titles

1. `adr: freeze go-control-plane and langgraph orchestrator split`
2. `feat: add broker-account and strategy-agent schema`
3. `feat: scaffold python langgraph orchestrator`
4. `feat: add hk-a market data adapters`
5. `feat: add broker-neutral execution contracts`
6. `feat: rebuild research risk execution graph`
7. `feat: expose broker-account and graph-run APIs`
8. `feat: migrate frontend to broker and strategy-agent semantics`
9. `feat: add graph streaming replay and approvals`
10. `refactor: remove crypto trading modules after cutover`
