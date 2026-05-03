# PRD: Real-Time Knowledge Base (KB) Layer
### Layer 1 of the Regime-Adaptive Active Inference Architecture
**Document Type:** Product Requirements Document  
**Version:** 1.0.0  
**Classification:** Quant Systems Design  
**Methodology:** First Principles Decomposition  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [First Principles Decomposition](#2-first-principles-decomposition)
3. [Problem Statement](#3-problem-statement)
4. [Goals & Non-Goals](#4-goals--non-goals)
5. [System Context & Positioning](#5-system-context--positioning)
6. [Functional Requirements](#6-functional-requirements)
7. [Data Architecture](#7-data-architecture)
8. [Node & Edge Schema Design](#8-node--edge-schema-design)
9. [Bi-Temporal Tracking Specification](#9-bi-temporal-tracking-specification)
10. [ATR & Regime State Ingestion](#10-atr--regime-state-ingestion)
11. [Non-Functional Requirements](#11-non-functional-requirements)
12. [API & Interface Contracts](#12-api--interface-contracts)
13. [Integration Points with Upper Layers](#13-integration-points-with-upper-layers)
14. [Technology Stack Recommendations](#14-technology-stack-recommendations)
15. [Risks & Mitigations](#15-risks--mitigations)
16. [Success Metrics](#16-success-metrics)
17. [Open Questions](#17-open-questions)

---

## 1. Executive Summary

This PRD defines the requirements for **Layer 1 — the Real-Time Knowledge Base (KB)**, the foundational substrate of a four-layer multi-hybrid graph architecture designed for advanced AI-driven financial agents, with specific adaptation to **regime-change detection** and **ATR-driven volatility classification**.

The KB Layer is not simply a database. It is a **living generative model** of the world — a time-aware, probabilistic, graph-structured memory that encodes hidden state variables (real-world events, market conditions, regime signals) as nodes and their conditional dependencies as edges. Every upstream layer — Attention, Social Graph, and Causal DAG — is epistemically blind without a correctly operating KB Layer.

---

## 2. First Principles Decomposition

Before specifying requirements, we decompose the problem to its atomic truths using First Principles reasoning.

### 2.1 What is the fundamental problem an AI agent faces in markets?

**Atomic truth:** Markets are non-stationary stochastic processes. Any model that assumes a fixed statistical regime will eventually fail. Therefore, the agent's memory system must not store facts — it must store **facts conditioned on regime context**.

### 2.2 What is the minimal unit of knowledge in a financial domain?

A knowledge unit is a **triple**:

```
(Entity, Relationship, Entity) @ (ValidTime, TransactionTime, RegimeState)
```

A flat key-value database collapses the temporal and regime dimensions. A Knowledge Graph preserves them. Therefore, a Knowledge Graph is the minimal viable structure for this problem — not because it is fashionable, but because it is the only structure that can represent conditional, time-varying, regime-dependent facts without information loss.

### 2.3 Why does the "when" matter as much as the "what"?

**Atomic truth:** In financial markets, the *lag* between when a fact becomes objectively true and when it is *priced in* is where alpha lives. A flat database collapses these two timestamps into one. A bi-temporal graph separates them — enabling the agent to reason: *"This event was true at T1 but was not reflected in prices until T2. Therefore, a window existed between T1 and T2 where arbitrage was available."*

### 2.4 What makes a standard Knowledge Graph insufficient?

Standard KGs assume static ontologies and static edge weights. Financial reality violates this:

- Causal relationships between assets **invert** during regime transitions.
- A node's reliability as a signal changes with ATR state.
- New event types emerge continuously (e.g., a novel regulatory category).

Therefore, the KB Layer requires: **(a) dynamic edge weight mutation, (b) schema evolution, and (c) regime-conditional node activation**.

### 2.5 What is the minimal set of inputs the KB Layer must ingest?

From first principles, a financial KB needs to track:

| Input Class | Why It Cannot Be Omitted |
|---|---|
| Price data (OHLCV) | The primary signal medium of market consensus |
| ATR / Volatility metrics | The control variable for regime classification |
| Macroeconomic events | Exogenous shocks that rewire causal structure |
| News & sentiment signals | Off-chain social information with lead-lag properties |
| Order flow & liquidity data | Reveals institutional intent before price moves |
| Prediction market contract states | The direct target of agent action |

---

## 3. Problem Statement

Existing AI trading systems operate with **memoryless, stateless inference engines** layered on top of **flat time-series databases**. This architecture creates four critical failure modes:

1. **Regime Blindness:** The system cannot distinguish between a trending, mean-reverting, or crisis market regime. A signal that is alpha in one regime is noise (or anti-alpha) in another.

2. **Temporal Conflation:** Storing only transaction time collapses the gap between *when events happened* and *when they were priced in*, destroying the most exploitable information in financial markets.

3. **Causal Amnesia:** Without a structured memory of *which events preceded which outcomes* under *which conditions*, the agent cannot build causal models. It is forever confined to correlation.

4. **Context Fragmentation:** Disparate data streams (price, news, sentiment, on-chain data) are stored in separate silos with no relational structure. The system cannot represent cross-domain dependencies.

The Real-Time KB Layer is the direct architectural solution to all four failure modes.

---

## 4. Goals & Non-Goals

### 4.1 Goals

- **G1:** Provide a unified, queryable, graph-structured memory of all market-relevant entities and their relationships.
- **G2:** Implement bi-temporal tracking (valid time + transaction time) for every node and edge mutation.
- **G3:** Classify and tag all stored facts with a **regime state** (Expansion, Contraction, Crisis, Recovery) derived from ATR signal processing.
- **G4:** Ingest, normalize, and graph-encode real-time off-chain events (news, macro announcements, social signals) with sub-second latency targets.
- **G5:** Expose a stable, versioned graph query API to Layer 2 (Attention Graph), Layer 3 (Social Graph), and Layer 4 (Causal DAG).
- **G6:** Support **schema evolution** — new node types and edge types must be registrable at runtime without full system restarts.
- **G7:** Enable **temporal replay** — reconstruct the exact state of the knowledge graph at any historical moment for backtesting purposes.

### 4.2 Non-Goals

- **NG1:** The KB Layer does not perform attention routing (that is Layer 2's responsibility).
- **NG2:** The KB Layer does not perform causal inference (that is Layer 4's responsibility).
- **NG3:** The KB Layer does not execute trades or interface with exchange APIs directly.
- **NG4:** The KB Layer is not a general-purpose relational database or OLAP warehouse.
- **NG5:** The KB Layer does not rank the credibility of social sources (that is Layer 3's responsibility).

---

## 5. System Context & Positioning

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA SOURCES                            │
│  [Market Feeds] [News APIs] [Social Streams] [Macro Events]        │
│  [On-Chain Data] [Options Flow] [ATR Signal Pipeline]              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ Ingest (Streaming + Batch)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│            LAYER 1: REAL-TIME KNOWLEDGE BASE (KB)                  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │ Event Parser │  │ Regime       │  │  Bi-Temporal Graph Store  │ │
│  │ & Normalizer │→ │ Classifier   │→ │  (Nodes + Edges + Time)   │ │
│  └──────────────┘  └──────────────┘  └───────────────────────────┘ │
│                                               │                     │
│                                    ┌──────────▼──────────┐         │
│                                    │  Graph Query Engine  │         │
│                                    │  (Versioned API)     │         │
│                                    └──────────┬──────────┘         │
└───────────────────────────────────────────────┼────────────────────┘
                           ┌───────────┬─────────┴───────────┐
                           ▼           ▼                      ▼
                     [Layer 2:    [Layer 3:           [Layer 4:
                     Attention    Social Graph]        Causal DAG]
                     Graph]
```

---

## 6. Functional Requirements

### 6.1 Data Ingestion

| ID | Requirement | Priority |
|---|---|---|
| FR-INS-01 | The system MUST ingest real-time OHLCV price data from at least 2 exchange connectors with <500ms latency. | P0 |
| FR-INS-02 | The system MUST ingest ATR values computed on configurable rolling windows (14-period default). | P0 |
| FR-INS-03 | The system MUST ingest structured macro event data (FOMC, NFP, CPI, earnings) from a calendar API. | P0 |
| FR-INS-04 | The system MUST ingest unstructured news signals from at least 1 NLP-processed news feed, tagged with sentiment and entity mentions. | P1 |
| FR-INS-05 | The system MUST ingest social signal summaries from Layer 3's preprocessed credibility-ranked feed. | P1 |
| FR-INS-06 | The system SHOULD ingest on-chain transaction volume and large wallet movement events. | P2 |
| FR-INS-07 | All ingested events MUST be assigned a `valid_time` (when it occurred in reality) and a `transaction_time` (when the system recorded it). | P0 |

### 6.2 Graph Construction & Mutation

| ID | Requirement | Priority |
|---|---|---|
| FR-GC-01 | The system MUST represent all market entities (assets, contracts, events, agents) as **nodes** in a property graph. | P0 |
| FR-GC-02 | The system MUST represent conditional dependencies between nodes as **typed, weighted, directional edges**. | P0 |
| FR-GC-03 | Edge weights MUST be mutable and MUST store a mutation history with timestamps. | P0 |
| FR-GC-04 | The system MUST support runtime registration of new node types and edge types via a schema registry. | P1 |
| FR-GC-05 | Every node state mutation MUST be logged as an immutable event — the current state must be derivable from the event log. | P0 |
| FR-GC-06 | The system MUST tag every node and edge with the **active regime state** at the time of creation or mutation. | P0 |

### 6.3 Regime Classification

| ID | Requirement | Priority |
|---|---|---|
| FR-RC-01 | The system MUST classify the current market regime from ATR inputs into at least 4 states: `EXPANSION`, `CONTRACTION`, `BREAKOUT`, `CRISIS`. | P0 |
| FR-RC-02 | Regime state transitions MUST be recorded as first-class events in the graph with a dedicated `RegimeTransitionEdge`. | P0 |
| FR-RC-03 | The regime classifier MUST support user-configurable ATR thresholds per asset class. | P1 |
| FR-RC-04 | The system MUST emit a `REGIME_CHANGE` event to all subscribing upper layers within 100ms of a confirmed regime transition. | P0 |

### 6.4 Temporal Query & Replay

| ID | Requirement | Priority |
|---|---|---|
| FR-TQ-01 | The graph query engine MUST support point-in-time queries: "What was the state of the graph at time T?" | P0 |
| FR-TQ-02 | The system MUST support range queries: "What nodes changed between T1 and T2?" | P0 |
| FR-TQ-03 | The system MUST support regime-conditional queries: "What edges existed during CONTRACTION regimes?" | P1 |
| FR-TQ-04 | The system MUST support full temporal replay for backtesting, reconstructing the graph state at any historical moment. | P0 |

---

## 7. Data Architecture

### 7.1 Storage Layers

The KB Layer uses a **three-tier storage architecture** to balance speed, durability, and historical depth:

```
┌──────────────────────────────────────────────────────────┐
│  TIER 1: Hot Store (In-Memory Graph)                     │
│  Technology: RedisGraph / Memgraph                       │
│  Retention: Last 72 hours of live graph state            │
│  Purpose: Sub-millisecond reads for Layer 2/3/4          │
│  Access: Direct graph query (Cypher / GQL)               │
└──────────────────────────────┬───────────────────────────┘
                               │ Async flush
┌──────────────────────────────▼───────────────────────────┐
│  TIER 2: Warm Store (Persistent Graph DB)                │
│  Technology: Neo4j / Kuzu / Neptune                      │
│  Retention: Full operational history (rolling 2 years)   │
│  Purpose: Complex traversal queries, backtesting         │
│  Access: Cypher / GQL queries                            │
└──────────────────────────────┬───────────────────────────┘
                               │ Async archival
┌──────────────────────────────▼───────────────────────────┐
│  TIER 3: Cold Store (Event Log / Object Storage)         │
│  Technology: Apache Kafka (event log) + S3/GCS           │
│  Retention: Unlimited (immutable append-only log)        │
│  Purpose: Full audit trail, regulatory compliance,       │
│           temporal replay from any point in history      │
│  Access: Stream replay / batch materialization           │
└──────────────────────────────────────────────────────────┘
```

### 7.2 Event Sourcing Pattern

The KB Layer follows an **event-sourced architecture** — the persistent truth is the **stream of mutation events**, not the current graph state. The current state is a materialized view derived from replaying the event log. This ensures:

- **Auditability:** Every state can be explained by the sequence of events that produced it.
- **Temporal replay:** Backtesting is achieved by replaying the event log up to a target timestamp.
- **Fault tolerance:** The graph can be reconstructed from scratch if the in-memory store is lost.

---

## 8. Node & Edge Schema Design

### 8.1 Core Node Types

```yaml
# === ASSET NODE ===
AssetNode:
  id: UUID
  symbol: string           # e.g., "BTC-USD"
  asset_class: enum        # CRYPTO | EQUITY | FX | COMMODITY | RATE
  exchange: string
  current_price: float
  current_atr_14: float
  regime_state: RegimeEnum
  valid_from: timestamp
  valid_to: timestamp | null
  tx_time: timestamp

# === EVENT NODE ===
EventNode:
  id: UUID
  event_type: enum         # MACRO | NEWS | SOCIAL | ON_CHAIN | EARNINGS | REGULATORY
  description: string
  sentiment_score: float   # [-1.0, 1.0]
  confidence: float        # [0.0, 1.0]
  source_id: UUID          # Reference to SourceNode
  regime_at_occurrence: RegimeEnum
  valid_time: timestamp
  tx_time: timestamp

# === REGIME STATE NODE ===
RegimeNode:
  id: UUID
  regime_type: enum        # EXPANSION | CONTRACTION | BREAKOUT | CRISIS
  atr_value: float
  atr_percentile: float    # Relative ATR level (0-100)
  asset_id: UUID
  valid_from: timestamp
  valid_to: timestamp | null
  transition_trigger: string  # What caused the transition

# === MARKET CONTRACT NODE ===
ContractNode:
  id: UUID
  contract_type: enum      # PREDICTION | OPTIONS | FUTURES | SPOT
  underlying_asset_id: UUID
  expiry: timestamp | null
  current_probability: float  # For prediction markets
  liquidity_score: float
  manipulation_risk: float    # Fed by Layer 3
  regime_at_open: RegimeEnum
  valid_from: timestamp
  tx_time: timestamp

# === SOURCE NODE ===
SourceNode:
  id: UUID
  source_type: enum        # NEWS_API | SOCIAL | EXCHANGE | ON_CHAIN | MACRO_CALENDAR
  name: string
  base_credibility: float  # Initial credibility score (refined by Layer 3)
  latency_ms: float        # Historical average delivery latency
  last_active: timestamp
```

### 8.2 Core Edge Types

```yaml
# === CONDITIONAL DEPENDENCY EDGE ===
ConditionalDependencyEdge:
  id: UUID
  from_node: UUID
  to_node: UUID
  weight: float            # Precision weight (0.0 - 1.0)
  regime_condition: RegimeEnum | null  # null = regime-agnostic
  lag_ms: integer          # Typical lag between cause and effect
  confidence: float
  valid_from: timestamp
  valid_to: timestamp | null

# === CAUSAL LEAD EDGE ===
# (Populated primarily by Layer 4, but seeded here)
CausalLeadEdge:
  id: UUID
  cause_node: UUID
  effect_node: UUID
  direction_confidence: float
  regime_condition: RegimeEnum
  average_lag_ms: integer
  valid_from: timestamp

# === REGIME TRANSITION EDGE ===
RegimeTransitionEdge:
  id: UUID
  from_regime: RegimeEnum
  to_regime: RegimeEnum
  asset_id: UUID
  trigger_event_id: UUID | null
  atr_at_transition: float
  valid_time: timestamp
  tx_time: timestamp
```

---

## 9. Bi-Temporal Tracking Specification

### 9.1 The Two Time Axes

Every entity in the KB graph is tracked across two independent time axes:

| Axis | Name | Definition | Use Case |
|---|---|---|---|
| **Axis 1** | Valid Time (`VT`) | When the fact was true in reality | "This news event occurred at 14:32:05 UTC" |
| **Axis 2** | Transaction Time (`TT`) | When the system recorded the fact | "We processed and stored this event at 14:32:07 UTC" |

The gap between `VT` and `TT` is the **ingestion lag** — a critical signal in itself. If an event has a large VT-TT gap, it may mean our data pipeline has latency issues, OR that the event was discovered late from a secondary source (retroactive information).

### 9.2 Bi-Temporal Query Semantics

```sql
-- Standard queries the system must support:

-- "What did the graph know as of NOW about events that happened before T1?"
SELECT * FROM nodes WHERE valid_time <= T1 AND tx_time <= NOW()

-- "What was the graph's view of the world at system time T_sys,
--  showing only facts it considered current at that moment?"
SELECT * FROM nodes WHERE valid_from <= T_sys AND (valid_to IS NULL OR valid_to > T_sys)
                      AND tx_time <= T_sys

-- "Show me facts that were retroactively discovered (late-ingested)
--  — high alpha indicator"
SELECT * FROM nodes WHERE (tx_time - valid_time) > THRESHOLD
```

### 9.3 Regime-Temporal Cube

The most powerful query capability is the **Regime-Temporal Cube** — querying the intersection of:

1. **What was true in reality** (valid time)
2. **What the system knew** (transaction time)
3. **What market regime was active** (regime state)

This cube is the fundamental structure that enables the agent to answer: *"Given that we were in a BREAKOUT regime, and we knew X at T1, which facts were we acting on?"*

---

## 10. ATR & Regime State Ingestion

### 10.1 ATR Pipeline

The Average True Range (ATR) is not merely a data field — it is the **primary control signal** that governs how the entire graph is interpreted by upper layers. The KB Layer must treat ATR as a first-class citizen.

```
Raw OHLCV  ──►  ATR Calculator  ──►  ATR Percentile Ranker  ──►  Regime Classifier
                (Wilder's, 14p)       (rolling 252-day window)     (Threshold Engine)
                                                                         │
                                                              ┌──────────▼───────────────┐
                                                              │  RegimeNode Created/      │
                                                              │  Updated in Graph         │
                                                              │  + RegimeTransitionEdge   │
                                                              │  emitted if state changed │
                                                              └──────────────────────────┘
```

### 10.2 Regime Classification Logic

```python
# Pseudo-code for regime classification engine
def classify_regime(atr_value: float, atr_percentile: float,
                    price_series: Series, config: RegimeConfig) -> RegimeEnum:

    if atr_percentile >= config.crisis_threshold:        # e.g., > 90th percentile
        return RegimeEnum.CRISIS

    elif atr_percentile >= config.breakout_threshold:    # e.g., 65th - 90th percentile
        # Additional confirmation: directional breakout check
        if is_directional_breakout(price_series):
            return RegimeEnum.BREAKOUT
        else:
            return RegimeEnum.EXPANSION

    elif atr_percentile <= config.contraction_threshold: # e.g., < 25th percentile
        return RegimeEnum.CONTRACTION

    else:
        return RegimeEnum.EXPANSION
```

### 10.3 Regime Transition Event Schema

When a regime transition is detected, the following event is emitted synchronously to all layer subscriptions before any further ingestion occurs. This is a **hard synchronization barrier** — all layers must acknowledge the regime change before the KB continues processing new facts under the new regime context.

```json
{
  "event_type": "REGIME_CHANGE",
  "asset_id": "uuid-btc-usd",
  "from_regime": "CONTRACTION",
  "to_regime": "BREAKOUT",
  "valid_time": "2025-10-14T09:45:22.341Z",
  "tx_time": "2025-10-14T09:45:22.398Z",
  "atr_value": 1842.50,
  "atr_percentile": 71.3,
  "trigger": "ATR_THRESHOLD_BREACH",
  "confirmation_bars": 2,
  "downstream_directive": {
    "layer2_attention": "REROUTE_TO_MOMENTUM",
    "layer4_causal": "REWIRE_EDGES_FOR_NEW_REGIME"
  }
}
```

---

## 11. Non-Functional Requirements

### 11.1 Performance

| Metric | Target | Hard Limit |
|---|---|---|
| Ingestion throughput | 50,000 events/sec | 10,000 events/sec minimum |
| Graph write latency (p99) | < 10ms | < 50ms |
| Graph point query latency (p99) | < 5ms (hot store) | < 100ms (warm store) |
| Regime transition notification latency | < 100ms end-to-end | < 500ms |
| Temporal replay speed | 100x realtime | 10x realtime minimum |

### 11.2 Reliability

| Requirement | Specification |
|---|---|
| Availability | 99.9% uptime during market hours |
| Event log durability | Zero data loss (Kafka replication factor ≥ 3) |
| Recovery Time Objective (RTO) | < 5 minutes (hot store loss, cold store replay) |
| Recovery Point Objective (RPO) | < 1 second (event log lag) |

### 11.3 Scalability

- **Horizontal scaling:** Event ingestion pipeline must scale to N parallel consumers without ordering violations within a single asset stream.
- **Graph partitioning:** The warm store must support asset-class-based graph partitioning to isolate high-cardinality asset namespaces.
- **Schema evolution:** New node and edge types must be registrable with zero downtime using the schema registry.

### 11.4 Observability

- All ingestion pipelines must emit **structured logs** with `asset_id`, `regime_state`, `valid_time`, `tx_time`, and `pipeline_stage`.
- A **graph health dashboard** must expose: node count by type, edge count by type, current regime state per asset, ingestion lag distribution, and event log consumer offset lag.
- **Regime transition events** must be independently tracked in a dedicated metrics stream with alerting thresholds.

---

## 12. API & Interface Contracts

### 12.1 Graph Query API (Internal)

The KB Layer exposes a versioned gRPC + REST API consumed by upper layers.

```protobuf
// gRPC Service Definition (abbreviated)
service KnowledgeBaseService {

  // Point-in-time graph snapshot
  rpc GetGraphSnapshot(SnapshotRequest) returns (GraphSnapshot);

  // Streaming subscription to node/edge mutations
  rpc SubscribeMutations(SubscriptionFilter) returns (stream MutationEvent);

  // Regime state query
  rpc GetCurrentRegime(RegimeRequest) returns (RegimeState);

  // Bi-temporal range query
  rpc QueryBiTemporal(BiTemporalQuery) returns (NodeList);

  // Subscribe to regime change events
  rpc SubscribeRegimeChanges(RegimeSubscription) returns (stream RegimeChangeEvent);
}
```

### 12.2 Schema Registry API

```http
# Register a new node type
POST /schema/node-types
Content-Type: application/json
{
  "type_name": "DerivativesFlowNode",
  "properties": ["notional_usd", "put_call_ratio", "open_interest"],
  "required_properties": ["notional_usd"],
  "valid_from": "2025-10-14T00:00:00Z"
}

# List all registered types (at a given transaction time)
GET /schema/node-types?as_of=2025-10-14T09:00:00Z

# Register a new edge type with regime conditions
POST /schema/edge-types
```

### 12.3 Event Publication Contract

The KB Layer publishes to a **shared event bus** (Kafka topics) that upper layers consume:

| Topic | Publisher | Consumers | Event Types |
|---|---|---|---|
| `kb.mutations` | KB Layer | Layer 2, 4 | Node/edge create, update, delete |
| `kb.regime_changes` | KB Layer | Layer 2, 3, 4 | `REGIME_CHANGE` events |
| `kb.ingestion_errors` | KB Layer | Ops/Alerting | Pipeline failures, schema violations |
| `kb.temporal_replay` | KB Layer | All layers | Replay events for backtesting |

---

## 13. Integration Points with Upper Layers

### 13.1 Layer 2 (Attention Graph) Integration

The Attention Graph consumes two streams from the KB:

1. **`kb.mutations` stream** — used to update attention weights as new nodes become active.
2. **`kb.regime_changes` stream** — the primary trigger for attention rerouting. Upon receiving a `REGIME_CHANGE` event, Layer 2 immediately re-runs its precision-weighting algorithm.

**Data contract the KB must fulfill for Layer 2:**
- Every node must carry a `signal_volatility_correlation` property: how correlated this node type's signals are with ATR volatility. This allows Layer 2 to calculate precision weights without additional lookups.

### 13.2 Layer 3 (Social Graph) Integration

Layer 3 both consumes from and writes back to the KB:

- **Consumes:** `EventNode` records of type `NEWS` and `SOCIAL` to rank their source credibility.
- **Writes back:** Updates the `base_credibility` field on `SourceNode` records after computing calibration scores.

**Critical requirement:** Layer 3 write-backs must be treated as **second-class mutations** — they are tagged with `source: LAYER3` and do not trigger `kb.mutations` downstream events to Layer 4 (to prevent feedback loops).

### 13.3 Layer 4 (Causal DAG) Integration

Layer 4 is the primary consumer of the KB's structural data:

- **Consumes:** Full graph snapshots at regime transition points to re-run causal inference.
- **Consumes:** `RegimeTransitionEdge` events to determine which causal relationships need to be rewired.
- **Writes back:** `CausalLeadEdge` records that encode the inferred causal direction between nodes under specific regime conditions.

---

## 14. Technology Stack Recommendations

| Component | Primary Recommendation | Alternative | Rationale |
|---|---|---|---|
| Hot Graph Store | **Memgraph** | RedisGraph | In-memory Cypher-compatible graph with streaming support |
| Warm Graph Store | **Neo4j** (Community/Enterprise) | Kuzu, TigerGraph | Mature, battle-tested, Cypher query language, temporal plugin support |
| Event Log / Message Bus | **Apache Kafka** | Redpanda | Durable, partitioned, replayable event stream; de-facto standard |
| Time-series (ATR pipeline) | **TimescaleDB** or **QuestDB** | InfluxDB | High-throughput time-series optimized for financial OHLCV data |
| Schema Registry | **Confluent Schema Registry** | AWS Glue | Enforces schema evolution rules on Kafka topics |
| Graph Query Middleware | **Graphiti** (or custom) | MAGMA | Time-aware relational memory layer purpose-built for AI agents |
| Stream Processor | **Apache Flink** | Kafka Streams | Stateful stream processing for ATR calculation and regime classification |
| Service Communication | **gRPC + Protocol Buffers** | REST/JSON | Low-latency inter-layer communication |
| Observability | **Prometheus + Grafana** | Datadog | Graph health, ingestion lag, and regime event dashboards |

---

## 15. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Clock skew between data sources** causes valid_time errors | Medium | High | Implement NTP-synchronized timestamps; use source-declared timestamps as valid_time, not system clock |
| **Regime classification oscillation** around ATR thresholds | High | Medium | Add hysteresis band (e.g., require 2 consecutive bars above threshold for confirmation) |
| **Graph write bottleneck** during high-volatility events (which is exactly when throughput peaks) | Medium | Critical | Pre-allocate node IDs; use batch write operations; implement back-pressure signaling to ingestion layer |
| **Schema evolution breaks existing queries** | Low | High | Enforce backward-compatible schema changes via registry; version all node types; never delete fields, only deprecate |
| **Layer 3 write-backs creating feedback loops** via KB mutations | Medium | Medium | Tag and filter Layer 3 writes; do not re-emit as standard `kb.mutations` events |
| **Cold store replay performance** makes backtesting unviably slow | Medium | Medium | Pre-compute materialized graph snapshots at daily intervals; use snapshots as replay starting points |
| **ATR itself being a lagging indicator** causes delayed regime detection | High | Medium | Supplement ATR with leading indicators (e.g., options implied volatility, order book imbalance) as secondary regime signals |

---

## 16. Success Metrics

### 16.1 System Health KPIs

| KPI | Target | Measurement Method |
|---|---|---|
| Ingestion lag (p99) | < 200ms from source to graph | `valid_time` vs `tx_time` delta distribution |
| Regime transition notification latency | < 100ms | Time from ATR threshold breach to Layer 2 acknowledgment |
| Graph query availability | > 99.9% | Uptime monitoring on hot store query endpoint |
| Temporal replay accuracy | 100% match to live run | Deterministic replay vs. live graph state comparison on known historical segments |
| Schema evolution zero-downtime | 100% of schema changes | Deployment rollback rate on schema registry updates |

### 16.2 Alpha Generation KPIs

The ultimate measure of Layer 1 quality is its contribution to agent decision quality:

| KPI | Target | How It Traces to Layer 1 |
|---|---|---|
| Regime detection accuracy | > 85% (confirmed ex-post) | Correct ATR-based regime classification in KB |
| Information lead time | > 0ms average (agent acts before crowd) | Low ingestion lag + bi-temporal gap analysis |
| Causal edge stability by regime | < 10% edge inversion rate within a regime | Correct regime tagging on all KB facts |
| Backtest fidelity | < 1% divergence from live run | Temporal replay accuracy |

---

## 17. Open Questions

| ID | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-01 | Should ATR be calculated on raw price or adjusted-price series for regime classification? Using adjusted prices avoids distortions from splits/dividends but introduces look-ahead bias in live systems. | Quant Team | Sprint 1 |
| OQ-02 | What is the correct granularity for regime state — per-asset, per-asset-class, or global macro? A global regime might suppress important asset-specific signals. | Architecture | Sprint 1 |
| OQ-03 | How should the KB handle conflicting valid_times from different data sources for the same event? (e.g., two news APIs reporting the same announcement with different timestamps) | Data Engineering | Sprint 2 |
| OQ-04 | Should the KB Layer implement its own access control layer, or delegate to the infrastructure layer (e.g., Kafka ACLs + graph DB role-based access)? | Security / Infra | Sprint 2 |
| OQ-05 | What is the retention policy for cold store data? Regulatory requirements may mandate 7+ years of audit trail retention. | Legal / Compliance | Sprint 3 |
| OQ-06 | Should `RegimeNode` be a snapshot node (new node per regime period) or a mutable node with edge history? Snapshot approach simplifies temporal queries but increases node cardinality significantly. | Architecture | Sprint 1 |

---

*Document authored using First Principles decomposition. Every requirement traces back to an atomic truth about the problem domain. Version-controlled. Subject to revision as Layer 2-4 interface contracts are finalized.*

---
**End of PRD — Layer 1: Real-Time Knowledge Base**
