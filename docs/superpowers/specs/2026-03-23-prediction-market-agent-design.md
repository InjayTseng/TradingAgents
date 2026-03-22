# Prediction Market Agent Design Spec

## Overview

Adapt the TradingAgents multi-agent framework to support prediction market analysis (Polymarket) as a parallel module. The system will output structured trading signals (BUY_YES / BUY_NO / PASS with position sizing) without executing actual orders.

## Architecture Decision

**Approach:** Parallel module at `tradingagents/prediction_market/`

- Shares: `llm_clients/`, `default_config.py` (extended), memory system concepts
- Does not modify: existing `agents/`, `dataflows/`, `graph/` code
- New module is self-contained with its own agents, dataflows, graph, and CLI entry point

## Directory Structure

```
tradingagents/
├── agents/              # Existing stock agents (untouched)
├── dataflows/           # Existing data layer (untouched)
├── graph/               # Existing graph (untouched)
├── llm_clients/         # Shared LLM client infrastructure
├── default_config.py    # Extended with PM config keys
├── prediction_market/   # NEW - all prediction market code
│   ├── __init__.py
│   ├── pm_config.py     # PM-specific default config
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── analysts/
│   │   │   ├── __init__.py
│   │   │   ├── event_analyst.py
│   │   │   ├── odds_analyst.py
│   │   │   ├── information_analyst.py
│   │   │   └── sentiment_analyst.py
│   │   ├── researchers/
│   │   │   ├── __init__.py
│   │   │   ├── yes_researcher.py
│   │   │   └── no_researcher.py
│   │   ├── risk_mgmt/
│   │   │   ├── __init__.py
│   │   │   ├── aggressive_debator.py
│   │   │   ├── conservative_debator.py
│   │   │   └── neutral_debator.py
│   │   ├── trader/
│   │   │   ├── __init__.py
│   │   │   └── pm_trader.py
│   │   ├── managers/
│   │   │   ├── __init__.py
│   │   │   ├── research_manager.py
│   │   │   └── risk_manager.py
│   │   └── utils/
│   │       ├── __init__.py
│   │       ├── pm_agent_states.py
│   │       ├── pm_tools.py
│   │       └── pm_agent_utils.py
│   ├── dataflows/
│   │   ├── __init__.py
│   │   ├── polymarket.py
│   │   └── polymarket_news.py
│   └── graph/
│       ├── __init__.py
│       ├── pm_trading_graph.py
│       ├── setup.py
│       ├── conditional_logic.py
│       ├── propagation.py
│       ├── reflection.py
│       └── signal_processing.py
```

## State Design

### PMAgentState

```python
class PMAgentState(MessagesState):
    # Inputs
    market_id: str              # Polymarket condition_id
    market_question: str        # Full question text
    trade_date: str             # Analysis date

    # Analyst reports
    event_report: str           # Event Analyst output
    odds_report: str            # Odds Analyst output
    information_report: str     # Information Analyst output
    sentiment_report: str       # Sentiment Analyst output

    # Debate
    investment_debate_state: PMInvestDebateState
    investment_plan: str

    # Trading
    trader_investment_plan: str
    risk_debate_state: PMRiskDebateState
    final_trade_decision: str

    # Routing
    sender: str
```

### PMInvestDebateState

```python
class PMInvestDebateState(TypedDict):
    yes_history: str            # YES researcher's history
    no_history: str             # NO researcher's history
    history: str                # Full debate transcript
    current_response: str       # Latest argument
    judge_decision: str         # Research manager's synthesis
    count: int                  # Turn counter
```

### PMRiskDebateState

Identical structure to existing `RiskDebateState` — the risk debate framework is domain-agnostic.

## Agent Roles

### Analyst Team (4 agents, all use `quick_thinking_llm`)

#### 1. Event Analyst

**Purpose:** Understand the event itself — what is being predicted, how it resolves, timeline.

**Tools:** `get_market_info`, `get_resolution_criteria`, `get_event_context`

**Prompt core:** Analyze the prediction market event. Parse the resolution criteria for clarity and objectivity. Assess the event timeline. Identify key dates/triggers that could cause resolution. Rate resolution ambiguity (clear/moderate/ambiguous). Output a structured event analysis report.

**Output:** `event_report` containing:
- Event description and resolution criteria
- Key dates and triggers
- Resolution ambiguity score
- Related markets within the same event

#### 2. Odds Analyst

**Purpose:** Assess market microstructure — is the current price efficient? Where is the edge?

**Tools:** `get_market_info`, `get_market_price_history`, `get_order_book`

**Prompt core:** Analyze the prediction market's pricing, liquidity, and historical probability movement. Assess bid-ask spread, order book depth, volume trends. Identify price patterns (trending, mean-reverting, volatile). Calculate market efficiency indicators. Determine if current price represents fair probability.

**Output:** `odds_report` containing:
- Current price/implied probability
- Spread and liquidity assessment
- Price trend analysis (probability velocity/acceleration)
- Market efficiency score
- Lifecycle stage (early/mid/late)

#### 3. Information Analyst

**Purpose:** Find information relevant to the event outcome that the market may not have priced in.

**Tools:** `get_news`, `get_global_news`, `get_related_markets`

**Prompt core:** Search for news, data, and developments related to the prediction market event. Identify information the market may not have fully incorporated. Assess how each piece of information affects the probability of the event occurring. Provide evidence-based analysis of factors for and against the event.

**Output:** `information_report` containing:
- Key news and developments
- Impact assessment on event probability
- Information freshness (how recently known)
- Unpriced information signals

#### 4. Sentiment Analyst

**Purpose:** Gauge public opinion and social media sentiment about the event.

**Tools:** `get_news` (reused from existing), `search_markets`

**Prompt core:** Analyze social media discussions, public polls, expert opinions, and crowd sentiment related to the prediction market event. Assess whether public sentiment diverges from market pricing. Identify viral narratives or emerging consensus that could shift probabilities.

**Output:** `sentiment_report` containing:
- Social media sentiment summary
- Public opinion trends
- Expert vs. crowd divergence
- Narrative momentum assessment

### Researcher Team (2 agents, `quick_thinking_llm`)

#### YES Researcher

**Purpose:** Build the case that the event WILL occur (probability should be higher).

**Prompt core:** Using all analyst reports, construct an evidence-based argument for why the event will occur and the market is underpricing YES. Cite specific evidence from the reports. Counter the NO researcher's arguments directly. Incorporate past memory reflections to avoid past mistakes.

**Memory:** `yes_memory` (BM25-based `FinancialSituationMemory`)

#### NO Researcher

**Purpose:** Build the case that the event will NOT occur (probability should be lower).

**Prompt core:** Using all analyst reports, construct an evidence-based argument for why the event will not occur and the market is overpricing YES. Cite specific evidence from the reports. Counter the YES researcher's arguments directly. Incorporate past memory reflections.

**Memory:** `no_memory` (BM25-based `FinancialSituationMemory`)

### Research Manager (`deep_thinking_llm`)

**Purpose:** Judge the YES/NO debate and synthesize an investment thesis.

**Prompt core:** Evaluate the debate. Estimate the true probability of the event occurring. Compare to market price to determine edge. Output a clear recommendation with probability estimate, confidence level, and reasoning.

**Memory:** `invest_judge_memory`

### PM Trader (`quick_thinking_llm`)

**Purpose:** Convert the investment thesis into a concrete position recommendation.

**Prompt core:** Based on the research manager's analysis, calculate:
1. Estimated true probability vs. market price → edge
2. Kelly Criterion position size (using 0.25x fractional Kelly)
3. Final recommendation: BUY_YES / BUY_NO / PASS

**Constraints:**
- Minimum edge threshold: 5% (below this → PASS)
- Maximum position: configurable (default 5% of bankroll)
- Must state: action, position_size_usd, estimated_probability, edge, confidence

**Memory:** `trader_memory`

### Risk Debate Team (3 agents, `quick_thinking_llm`)

Same structure as existing. Prompt adjustments:

- **Aggressive Debator:** Focus on edge magnitude, information advantage, favorable odds
- **Conservative Debator:** Focus on resolution ambiguity, liquidity risk, correlation exposure, model uncertainty
- **Neutral Debator:** Balance edge vs. risk, advocate fractional Kelly, assess time-to-resolution impact

### Risk Manager (`deep_thinking_llm`)

**Purpose:** Final judge — approve, modify, or reject the trade recommendation.

**Output modifications:** Must explicitly assess resolution risk, liquidity risk, correlation risk.

**Memory:** `risk_manager_memory`

## Data Layer

### Polymarket API Integration

**File:** `prediction_market/dataflows/polymarket.py`

Uses Polymarket's public REST APIs (no auth needed for read-only):

| Function | API | Endpoint |
|----------|-----|----------|
| `get_market_info(market_id)` | Gamma | `GET /markets/{id}` |
| `get_market_price_history(market_id, start_ts, end_ts)` | CLOB | `GET /prices-history` |
| `get_order_book(market_id)` | CLOB | `GET /book` |
| `get_resolution_criteria(market_id)` | Gamma | `GET /markets/{id}` (parsed) |
| `get_event_context(event_id)` | Gamma | `GET /events/{id}` |
| `get_related_markets(query, limit)` | Gamma | `GET /events?order=volume_24hr` |
| `search_markets(query, limit)` | Gamma | `GET /public-search` |

**Dependencies:** `requests` (already in requirements.txt)

**Caching:** Local file cache in `dataflows/data_cache/polymarket/` following existing pattern.

**News:** Reuse existing `get_news` and `get_global_news` tools from `tradingagents.dataflows` — news about real-world events is highly relevant to prediction markets.

### Tool Definitions

**File:** `prediction_market/agents/utils/pm_tools.py`

Each tool is a `@tool`-decorated function following existing conventions:

```python
@tool
def get_market_info(market_id: str, curr_date: str) -> str:
    """Get prediction market info including question, prices, volume, and resolution criteria."""
    ...

@tool
def get_market_price_history(market_id: str, start_date: str, end_date: str) -> str:
    """Get historical probability time series for a prediction market."""
    ...

@tool
def get_order_book(market_id: str) -> str:
    """Get current order book depth for a prediction market."""
    ...

@tool
def get_resolution_criteria(market_id: str) -> str:
    """Get detailed resolution criteria and source for a prediction market."""
    ...

@tool
def get_event_context(event_id: str, curr_date: str) -> str:
    """Get all markets grouped under a prediction market event."""
    ...

@tool
def get_related_markets(query: str, limit: int = 5) -> str:
    """Search for prediction markets related to a topic."""
    ...

@tool
def search_markets(query: str, status: str = "active", limit: int = 10) -> str:
    """Search Polymarket for markets matching a query."""
    ...
```

## Graph Execution Flow

```
START
  │
  ▼
[Event Analyst] ←──────────────┐
  │ tool_calls?                 │
  ├─YES→ [tools_event] ─────────┘
  └─NO→ [Msg Clear Event]
              │
              ▼
         [Odds Analyst] ←──────────────┐
              │ tool_calls?             │
              ├─YES→ [tools_odds] ──────┘
              └─NO→ [Msg Clear Odds]
                          │
                          ▼
                     [Information Analyst] ←──────────────┐
                          │ tool_calls?                    │
                          ├─YES→ [tools_information] ──────┘
                          └─NO→ [Msg Clear Information]
                                      │
                                      ▼
                               [Sentiment Analyst] ←──────────────┐
                                      │ tool_calls?                │
                                      ├─YES→ [tools_sentiment] ────┘
                                      └─NO→ [Msg Clear Sentiment]
                                                    │
                                                    ▼
                                            [YES Researcher] ←─────────┐
                                                    │                   │
                                            [NO Researcher] ───────────┘
                                                    │ (count >= 2*max)
                                                    ▼
                                         [Research Manager]
                                                    │
                                                    ▼
                                              [PM Trader]
                                                    │
                                                    ▼
                                        [Aggressive Analyst] ←─────────────┐
                                                    │                       │
                                       [Conservative Analyst]              │
                                                    │                       │
                                           [Neutral Analyst] ──────────────┘
                                                    │ (count >= 3*max)
                                                    ▼
                                              [Risk Judge]
                                                    │
                                                    ▼
                                                  END
```

## Signal Processing

The `final_trade_decision` is parsed into a structured JSON signal:

```json
{
  "action": "BUY_YES | BUY_NO | PASS",
  "market_id": "0x...",
  "market_question": "Will X happen by Y?",
  "estimated_probability": 0.75,
  "market_price": 0.60,
  "edge": 0.15,
  "position_size_usd": 940,
  "kelly_fraction": 0.094,
  "confidence": "high | medium | low",
  "resolution_risk": "low | medium | high",
  "liquidity_score": "good | fair | poor",
  "reasoning": "...",
  "timestamp": "2026-03-23T12:00:00Z"
}
```

## Reflection & Memory

Post-analysis, the caller can invoke `reflect_and_remember(resolution_outcome)` to update memories:

- `yes_memory` — Did the YES researcher's arguments prove correct?
- `no_memory` — Did the NO researcher's arguments prove correct?
- `trader_memory` — Was the position sizing appropriate? Was the edge estimate accurate?
- `invest_judge_memory` — Was the probability estimate calibrated?
- `risk_manager_memory` — Were the identified risks realized?

Reflection metrics shift from returns% to:
- Calibration (was estimated probability close to actual outcome rate?)
- Edge accuracy (did the identified edge materialize?)
- Risk assessment quality (did flagged risks materialize?)

## Configuration

New keys added to config:

```python
PM_DEFAULT_CONFIG = {
    # Inherit LLM settings from parent config
    "llm_provider": "openai",
    "deep_think_llm": "gpt-5.2",
    "quick_think_llm": "gpt-5-mini",

    # Prediction market specific
    "pm_mode": True,
    "polymarket_base_url": "https://gamma-api.polymarket.com",
    "polymarket_clob_url": "https://clob.polymarket.com",

    # Trading parameters
    "kelly_fraction": 0.25,          # Fractional Kelly multiplier
    "min_edge_threshold": 0.05,      # Minimum edge to take a position
    "max_position_pct": 0.05,        # Max % of bankroll per position
    "max_cluster_exposure_pct": 0.15, # Max % per correlated cluster
    "bankroll": 10000,               # Default bankroll for sizing

    # Debate settings (inherited)
    "max_debate_rounds": 1,
    "max_risk_discuss_rounds": 1,
    "max_recur_limit": 100,
}
```

## Entry Point

### Python API

```python
from tradingagents.prediction_market import PMTradingAgentsGraph

graph = PMTradingAgentsGraph(
    selected_analysts=["event", "odds", "information", "sentiment"],
    config={"bankroll": 10000, "kelly_fraction": 0.25}
)

final_state, signal = graph.propagate(
    market_id="0x...",
    trade_date="2026-03-23"
)

print(signal)  # Structured JSON signal
```

### CLI (future)

```bash
tradingagents pm analyze --market-id "0x..." --date "2026-03-23"
tradingagents pm search --query "US election"
```

## Out of Scope (v1)

- Actual order placement via py-clob-client
- WebSocket real-time data streaming
- Backtesting engine
- Multi-market portfolio optimization
- Cross-platform arbitrage (Kalshi, Manifold)
- Categorical/scalar market support (binary only for v1)

## Dependencies

No new pip dependencies required for v1:
- `requests` — already in requirements.txt
- All Polymarket API calls are simple REST GET requests

Future: `py-clob-client` for order placement (v2)
