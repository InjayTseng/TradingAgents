# Implementation Plan: Prediction Market Agent

Based on spec: `2026-03-23-prediction-market-agent-design.md`

## Phase 1: Foundation (sequential — everything else depends on this)

### Task 1.1: Module scaffold + State definitions + Config
- Create `tradingagents/prediction_market/` directory structure with all `__init__.py` files
- Implement `pm_agent_states.py` with `PMAgentState`, `PMInvestDebateState`, `PMRiskDebateState`
- Implement `pm_config.py` with `PM_DEFAULT_CONFIG`
- Wire up `__init__.py` exports

### Task 1.2: Polymarket data layer
- Implement `prediction_market/dataflows/polymarket.py`:
  - `get_polymarket_market_info(market_id)`
  - `get_polymarket_price_history(market_id, start_ts, end_ts)`
  - `get_polymarket_order_book(market_id)`
  - `get_polymarket_resolution_criteria(market_id)`
  - `get_polymarket_event_context(event_id)`
  - `get_polymarket_related_markets(query, limit)`
  - `get_polymarket_search(query, status, limit)`
- Local file caching in `dataflows/data_cache/polymarket/`

### Task 1.3: Tool definitions
- Implement `prediction_market/agents/utils/pm_tools.py` with all `@tool`-decorated functions
- Implement `prediction_market/agents/utils/pm_agent_utils.py` with `create_msg_delete` and tool re-exports

## Phase 2: Agents (parallelizable — 4 independent tracks)

### Task 2.1: Analyst agents
- `event_analyst.py` — tools: get_market_info, get_resolution_criteria, get_event_context
- `odds_analyst.py` — tools: get_market_info, get_market_price_history, get_order_book
- `information_analyst.py` — tools: get_news (reused), get_global_news (reused), get_related_markets
- `sentiment_analyst.py` — tools: get_news (reused), search_markets

### Task 2.2: Researcher agents
- `yes_researcher.py` — YES case advocate with memory
- `no_researcher.py` — NO case advocate with memory
- `managers/research_manager.py` — debate judge, probability estimator

### Task 2.3: Trader + Risk agents
- `trader/pm_trader.py` — Kelly Criterion position sizing, edge calculation
- `risk_mgmt/aggressive_debator.py` — edge magnitude focus
- `risk_mgmt/conservative_debator.py` — resolution/liquidity/correlation risk focus
- `risk_mgmt/neutral_debator.py` — balanced risk/reward
- `managers/risk_manager.py` — final judge

### Task 2.4: Signal processing + Reflection
- `graph/signal_processing.py` — parse free text → structured JSON signal
- `graph/reflection.py` — calibration-based reflection for all 5 memory agents

## Phase 3: Graph Assembly (sequential — needs Phase 2 complete)

### Task 3.1: Graph wiring
- `graph/conditional_logic.py` — tool loop routing, debate termination
- `graph/propagation.py` — initial state construction from market_id
- `graph/setup.py` — LangGraph StateGraph node/edge wiring
- `graph/pm_trading_graph.py` — main `PMTradingAgentsGraph` class

## Phase 4: Integration + Testing

### Task 4.1: End-to-end smoke test
- Write `test_pm.py` that runs a full analysis on a real Polymarket market
- Verify all agents execute, signal is produced, no crashes

## Execution Order

```
Phase 1 (sequential):
  1.1 → 1.2 → 1.3

Phase 2 (parallel, after Phase 1):
  2.1 ─┐
  2.2 ─┤ (all in parallel)
  2.3 ─┤
  2.4 ─┘

Phase 3 (sequential, after Phase 2):
  3.1

Phase 4 (after Phase 3):
  4.1
```
