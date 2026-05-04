# Feature Specification: Real-Time KB Ingestion Pipeline

**Feature Branch**: `001-kb-ingest-pipeline`
**Created**: 2026-05-04
**Status**: Draft
**PRD Reference**: `PRD §6.1 Data Ingestion`, `PRD §6.3 Regime Classification`, `PRD §9 Bi-Temporal Tracking`, `PRD §10 ATR & Regime State Ingestion`, `PRD §11.1 Performance`
**Input**: User description: "Build the real-time data ingestion pipeline for Layer 1 KB — ingest OHLCV price data, ATR metrics, and macro events into the bi-temporal graph store with regime classification"

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Live Price Data Reaches the Knowledge Graph (Priority: P1)

A market data event (OHLCV candlestick close) occurs on a connected exchange. Within 500
milliseconds, that event is recorded as a node in the knowledge graph — tagged with when it
actually happened in the market (valid time), when the system recorded it (transaction time), and
the currently active market regime for that asset.

**Why this priority**: This is the foundational data flow. All regime classification, ATR
computation, and upper-layer inference depend on live price data being in the graph promptly and
correctly labeled. Without this, the entire system is blind.

**Independent Test**: Can be tested end-to-end by publishing a synthetic OHLCV event to the
ingestion entry point, then querying the knowledge graph and confirming the node exists with
correct temporal metadata and regime tag — entirely independently of ATR or macro event flows.

**Acceptance Scenarios**:

1. **Given** a live OHLCV event is published for asset BTC-USD with a known market timestamp,
   **When** the ingestion pipeline processes it,
   **Then** a price node appears in the graph within 500ms, carrying `valid_time` equal to the
   market timestamp, `transaction_time` equal to the system receipt time, and `regime_state`
   reflecting the current classified regime for BTC-USD.

2. **Given** two OHLCV events arrive from different exchange sources for the same asset and
   overlapping time window,
   **When** both are ingested,
   **Then** both are stored as separate nodes with their respective source identifiers and
   transaction times, without either being silently dropped.

3. **Given** an OHLCV event arrives with a missing or malformed market timestamp,
   **When** the ingestion pipeline attempts to process it,
   **Then** the event is rejected, logged as an ingestion error with the asset id and failure
   reason, and does not enter the graph in a corrupted state.

---

### User Story 2 — ATR Computation Drives Regime Classification (Priority: P1)

As new OHLCV data enters the graph, the pipeline continuously computes the Average True Range
for each tracked asset and compares it against historical percentile thresholds. When the ATR
crosses a configured boundary, the system records a regime transition event and immediately
notifies all subscribed consumers (Layer 2 Attention Graph, Layer 3 Social Graph, Layer 4
Causal DAG) within 100 milliseconds.

**Why this priority**: Regime state is the control signal for the entire multi-layer system.
If regime transitions are missed, stale, or incorrect, every downstream layer operates under
wrong assumptions. A delayed notification (> 100ms) can cause upper layers to compute decisions
using the wrong regime context during a critical market event.

**Independent Test**: Can be tested by feeding a synthetic OHLCV series that is engineered to
cross an ATR percentile threshold at a known data point, then verifying: (a) the ATR is computed
correctly, (b) a regime transition event is recorded in the graph, and (c) a notification is
emitted within 100ms of the threshold crossing — all without requiring macro event or news feeds.

**Acceptance Scenarios**:

1. **Given** a stream of OHLCV data whose 14-period ATR rises above the configured Breakout
   threshold (default: 65th percentile of 252-day rolling window),
   **When** the regime classifier evaluates the update,
   **Then** the asset's regime state transitions from `CONTRACTION` or `EXPANSION` to `BREAKOUT`,
   a `RegimeTransitionEdge` is recorded in the graph with the triggering ATR value and timestamp,
   and a `REGIME_CHANGE` notification is emitted within 100ms.

2. **Given** an ATR value oscillates repeatedly above and below a threshold within consecutive
   bars,
   **When** the regime classifier processes these bars,
   **Then** no regime transition is recorded until the ATR remains on one side of the threshold
   for at least 2 consecutive bars (hysteresis rule), preventing spurious transition events.

3. **Given** a regime transition occurs for asset BTC-USD,
   **When** the transition notification is emitted,
   **Then** the notification payload includes: `asset_id`, `from_regime`, `to_regime`,
   `valid_time`, `atr_value`, `atr_percentile`, and `confirmation_bars` — and all subscribed
   consumers receive it within 100ms.

4. **Given** the ATR threshold configuration is updated for a specific asset class,
   **When** subsequent OHLCV data is processed,
   **Then** the regime classifier uses the new thresholds without requiring a system restart.

---

### User Story 3 — Macro Events Are Graph-Encoded with Bi-Temporal Metadata (Priority: P2)

A scheduled macro event (such as an FOMC rate decision or NFP jobs report) occurs in the real
world. The pipeline receives this event from the macro calendar data source, normalizes it into
a structured event record, and stores it as an event node in the knowledge graph — tagged with
when the announcement actually occurred (valid time), when the system recorded it (transaction
time), and the market regime active at that moment.

**Why this priority**: Macro events are exogenous shocks that rewire causal structure between
assets. They must be in the graph for Layer 4 (Causal DAG) to detect structural breaks. However,
macro event data delivery is inherently less time-critical than price data, justifying P2.

**Independent Test**: Can be tested by publishing a synthetic macro event record to the ingestion
entry point, then confirming a graph node of type `MACRO` exists with the correct event type,
valid_time, transaction_time, and regime_state — independently of price or ATR flows.

**Acceptance Scenarios**:

1. **Given** a macro event record (type: `FOMC`, scheduled time known) is received from the
   calendar data source,
   **When** the pipeline processes it,
   **Then** a node of type `MacroEventNode` is created in the graph carrying the event type,
   scheduled time as `valid_time`, system receipt time as `transaction_time`, and the regime
   state active at the time of ingestion.

2. **Given** the same macro event is delivered by the data source twice (duplicate delivery),
   **When** the pipeline processes the second delivery,
   **Then** no duplicate node is created; the existing node is confirmed idempotently, and the
   duplicate delivery is logged.

3. **Given** a macro event arrives with a transaction_time that is 30+ seconds after its
   valid_time (late delivery),
   **When** the pipeline stores it,
   **Then** the ingestion lag (transaction_time − valid_time) is recorded as a metric and the
   node is flagged with `late_delivery: true` to alert downstream consumers that this fact
   arrived with delay.

---

### User Story 4 — Temporal Replay Reconstructs Past Graph State (Priority: P3)

A backtesting engineer needs to know exactly what the knowledge graph contained at a specific
historical timestamp — for example, "what data did the system have at 09:45:22 UTC on a given
date, using only information the system knew at that moment?" The pipeline's event log enables
exact reconstruction of the graph state at any past point in time.

**Why this priority**: Temporal replay is essential for backtesting strategy performance, but it
relies on the ingestion pipeline having stored immutable events correctly from the start. It is
not needed for the live pipeline to be operational, making it P3.

**Independent Test**: Can be tested by ingesting a known sequence of events, querying the graph
at a specific past timestamp, and confirming that: (a) only events with `transaction_time ≤ T`
appear, and (b) the result is identical to what the live graph showed at time T — using a
pre-recorded live session as the ground truth.

**Acceptance Scenarios**:

1. **Given** a series of ingestion events spanning a known time window has been processed,
   **When** a point-in-time query is executed for timestamp T within that window,
   **Then** the result contains exactly the nodes and edges that existed at time T — no nodes
   with `transaction_time > T` are returned, and no nodes that existed at T are missing.

2. **Given** a temporal replay is initiated from a specific historical timestamp,
   **When** the replay completes,
   **Then** the reconstructed graph state matches the original live state at that timestamp
   with 100% accuracy (zero divergence).

---

### Edge Cases

- What happens when the exchange data feed disconnects mid-session? The pipeline must detect
  the gap, log the outage period, and resume from the last known good event on reconnection
  without creating phantom nodes for the missing period.
- What happens when two data sources report conflicting valid_times for the same macro event?
  Both are stored with their source identifiers; the most recent source-declared timestamp is
  used as the primary valid_time, and the conflict is logged for review.
- What happens when the ATR percentile ranking window contains insufficient history (< 252 days
  of data)? The regime classifier operates on available history, marks the result as
  `low_confidence: true`, and does not emit a regime transition until sufficient history exists.
- What happens when the event log storage reaches capacity? The pipeline enters a backpressure
  state, pausing new ingestion and alerting operators before any data is dropped.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The pipeline MUST accept and process OHLCV price data from at least 2 independent
  exchange data sources with an end-to-end ingestion latency below 500ms (p99).

- **FR-002**: The pipeline MUST assign a `valid_time` (source-declared event timestamp) and a
  `transaction_time` (system receipt timestamp) to every ingested event before graph storage.

- **FR-003**: The pipeline MUST reject events with missing `valid_time` or unparseable timestamps,
  log the rejection with asset id and failure reason, and never store a corrupted record.

- **FR-004**: The pipeline MUST compute a 14-period ATR (configurable) for each tracked asset
  from incoming OHLCV data and update the asset's ATR property in the graph with each new bar.

- **FR-005**: The pipeline MUST classify the current market regime for each asset into exactly
  one of four states — `EXPANSION`, `CONTRACTION`, `BREAKOUT`, `CRISIS` — based on ATR
  percentile thresholds against a rolling 252-day window.

- **FR-006**: The pipeline MUST apply a 2-bar confirmation (hysteresis) before recording a
  regime transition, to prevent spurious transitions from threshold oscillation.

- **FR-007**: Upon confirming a regime transition, the pipeline MUST record a
  `RegimeTransitionEdge` in the graph and emit a `REGIME_CHANGE` notification to all subscribed
  consumers within 100ms.

- **FR-008**: The `REGIME_CHANGE` notification MUST include: `asset_id`, `from_regime`,
  `to_regime`, `valid_time`, `transaction_time`, `atr_value`, `atr_percentile`,
  `confirmation_bars`, and a `downstream_directive` map for consumer-specific actions.

- **FR-009**: ATR classification thresholds MUST be configurable per asset class without
  requiring a system restart.

- **FR-010**: The pipeline MUST ingest structured macro events of types `FOMC`, `NFP`, `CPI`,
  and `EARNINGS` from a calendar data source and store each as a `MacroEventNode` in the graph.

- **FR-011**: The pipeline MUST tag every graph node and edge mutation with the `regime_state`
  active at the time of creation or mutation.

- **FR-012**: The pipeline MUST handle duplicate event delivery idempotently — processing the
  same event twice MUST NOT create duplicate nodes.

- **FR-013**: The pipeline MUST record the ingestion lag (`transaction_time − valid_time`) as a
  tracked metric for every event, with alerts when p99 lag exceeds 200ms.

- **FR-014**: The pipeline MUST support point-in-time graph queries: given a timestamp T, return
  only nodes and edges with `transaction_time ≤ T`.

- **FR-015**: The pipeline MUST emit structured logs for every ingestion event containing:
  `asset_id`, `regime_state`, `valid_time`, `tx_time`, and `pipeline_stage`.

### Key Entities

- **Price Event**: A single OHLCV candlestick record for one asset, representing open/high/
  low/close prices and volume over a time interval. Carries source identifier, exchange origin,
  valid time, and transaction time.

- **ATR Metric**: A computed volatility measurement derived from a rolling window of price
  events for one asset. Carries the current ATR value, its percentile rank within the
  historical window, and the timestamp of computation.

- **Regime State**: A classified market condition for one asset, valid over a time period.
  Carries the regime type, triggering ATR value and percentile, start time, and — once closed —
  an end time and the transition trigger reason.

- **Macro Event**: A structured record of a scheduled or unscheduled macroeconomic announcement.
  Carries the event type (FOMC/NFP/CPI/EARNINGS), the scheduled or actual occurrence time
  (valid time), system receipt time (transaction time), and the regime active at ingestion.

- **Regime Transition**: A recorded state change from one regime to another for a specific asset.
  Carries the before/after regime types, the ATR value at transition, the triggering event
  identifier (if any), and both temporal timestamps.

- **Ingestion Error**: A record of a rejected or failed ingestion attempt. Carries the source
  event identifier (if available), failure reason, asset id, and timestamp.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All OHLCV price events appear in the knowledge graph within 500ms of their
  market-reported timestamp (p99 across a sustained run of 1,000+ sequential events).

- **SC-002**: All regime transition notifications reach subscribed consumers within 100ms of
  the threshold-crossing confirmation (p99 measured end-to-end from ATR computation to
  consumer receipt acknowledgement).

- **SC-003**: 100% of stored graph nodes and edges carry non-null `valid_time` and
  `transaction_time` values, verified by a post-ingestion integrity scan.

- **SC-004**: 100% of stored graph mutations carry a non-null `regime_state` tag, verified by
  a post-ingestion integrity scan.

- **SC-005**: The pipeline sustains a minimum throughput of 10,000 events per second under
  continuous load, with 50,000 events per second as the target ceiling.

- **SC-006**: Historical graph state reconstructed via temporal replay matches the original
  live state at the same timestamp with zero divergence, validated against a pre-recorded
  reference session.

- **SC-007**: ATR-based regime classification achieves greater than 85% accuracy when validated
  against a set of known historical regime transition ground-truth labels.

- **SC-008**: The pipeline recovers from a complete in-memory store failure and resumes correct
  operation within 5 minutes by replaying the event log, with zero data loss.

- **SC-009**: Duplicate event delivery results in zero duplicate nodes in the graph, validated
  by delivering the same 100 events twice and scanning for duplicates.

- **SC-010**: Every ingestion rejection generates a logged error record; no event is silently
  dropped, verified by counting input events vs. (stored nodes + logged rejections).

---

## Assumptions

- The ingestion pipeline connects to pre-provisioned exchange data feeds; connection
  authentication and credential management are outside this feature's scope.
- ATR is computed using Wilder's smoothing method; this is the standard for financial ATR.
- Regime classification operates per-asset (not global macro); a global regime signal may
  be a future extension.
- The initial asset scope is crypto (BTC-USD, ETH-USD as primary); the design must support
  expansion to equities and FX without architectural changes.
- The 252-day rolling window for ATR percentile ranking is derived from one calendar year of
  trading days; this is the standard financial lookback for volatility regimes.
- News and social signal ingestion (PRD FR-INS-04, FR-INS-05) are out of scope for this
  feature; they will be specified and built as a separate ingestion sub-pipeline.
- On-chain data ingestion (PRD FR-INS-06, P2) is out of scope for this feature.
- The graph store infrastructure (hot store, warm store, event log) is assumed to be
  provisioned and accessible; infrastructure setup is not part of this feature.
- Macro events arrive from a structured calendar API, not from unstructured news text.
- The regime classifier uses configurable thresholds: Crisis > 90th ATR percentile,
  Breakout = 65th–90th percentile (with directional confirmation), Contraction < 25th
  percentile, Expansion = remainder. These are defaults that operators can override.
