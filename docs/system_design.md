# **System Design: Arcpoint Context Layer Prototype** 

## **1\. Purpose and Scope**

The Context Layer is responsible for providing **time-aware, decision-ready intelligence** to an LLM routing engine. Its goal is not merely to collect metrics, but to transform heterogeneous signals into **actionable context** that can support high-stakes routing, quality investigation, and capacity planning decisions.

This prototype focuses on:

- determining which models are viable _right now_ for a given request,
- explaining historical quality degradation using observed outcomes, and
- forecasting near-term traffic risk.

The system intentionally does **not** attempt to solve real-time streaming ingestion, persistent hot storage, or full operator dashboards. Instead, it demonstrates the **core reasoning logic and architectural boundaries** required for a production control plane.

&nbsp;

## **2\. High-Level Architecture**

The system is structured around three primary components:

- Context API  
    A single FastAPI endpoint that accepts natural-language queries from an operator, agent, or routing system.
- Context Engine  
    The source of truth for:
  - model viability evaluation,
  - routing candidate selection,
  - historical quality analysis,
  - traffic forecasting,
  - and evidence-bounded explanation.
- Synthetic Data Pipeline  
    An offline generator that produces realistic, multi-day request logs with quality drift, SLA pressure, and failures.

**Request Flow**
&nbsp;

User / Agent Query  
↓  
Context API (thin layer)  
↓  
Context Engine  
├─ Deterministic filtering & scoring  
├─ Time-windowed analysis (if needed)  
├─ Candidate selection or aggregation  
└─ LLM-based interpretation (bounded)  
↓  
Natural-language response

The API layer remains intentionally thin. All state interpretation and decision logic lives inside the Context Engine.

&nbsp;

## **3\. Routing Strategy: Stratified Top-N Selection**

A core design decision is the use of **Stratified Top-N** routing rather than global ranking.

**Why global ranking fails**

Ranking all models with a single scalar score systematically biases toward:

- fast,
- cheap,
- general-purpose models,

at the expense of specialized models that may be slower or more expensive but **better suited for specific tasks**.

This failure mode becomes acute in reasoning-heavy or long-context workloads.

&nbsp;

**Stratified Top-N approach**

The routing process follows these steps:

- **Hard viability gating**
  - **models that are down, excessively stale, SLA-violating, or rate-limited are removed immediately.**
- **Capability tagging**
  - **models are tagged based on supported tasks, context window size, and latency characteristics.**
- **Capability buckets**
  - **models are grouped into overlapping capability buckets (e.g., task-specific, long-context, low-latency).**
- **Heuristic scoring within buckets**
  - **latency, cost, error rate, degradation status, and quality trends are combined into a lightweight score.**
- **Quota-based selection**
  - **a limited number of top candidates are selected per bucket and merged into a final candidate set.**

Only this **curated candidate pool** is exposed to the LLM for interpretation.

**Strong opinion**

The LLM is never allowed to see or choose from the full fleet.

Routing must remain bounded by deterministic system constraints.

&nbsp;

## **4\. Time, Freshness, and Change**

Time is treated as a **first-class dimension** throughout the system.

**System time anchoring**

Rather than using wall-clock time, the Context Engine computes a **system "now"** based on the most recent observed request timestamp. This ensures consistency when analyzing historical windows or delayed quality signals.

&nbsp;

**Staleness handling**

Model metadata includes a last_updated timestamp. Staleness is computed as:
&nbsp;

staleness = system_now − last_updated

Models are excluded if their metadata exceeds a defined staleness threshold. This prevents routing decisions from relying on outdated or unreliable state.

&nbsp;

**Historical analysis**

For questions such as _"Why did quality drop two days ago?"_, the system:

- resolves an explicit day-level time window,
- computes aggregates within that window,
- compares results against a previous-day baseline,
- ranks contributors by volume and degradation.

Importantly, **current model state is not used** to explain past behavior. Historical queries rely strictly on historical evidence.

&nbsp;

## **5\. Deterministic Logic vs LLM Usage**

A deliberate boundary exists between deterministic logic and LLM usage.

**Deterministic logic handles:**

- viability filtering,
- heuristic scoring,
- aggregation and ranking,
- time window resolution,
- forecasting calculations.

**LLM usage is limited to:**

- intent classification (routing vs forecast vs quality vs fleet),
- natural-language explanation of **pre-computed evidence**.

**Guardrails**

The LLM is explicitly instructed to:

- use only provided evidence,
- avoid speculation,
- avoid introducing unseen models or incidents,
- describe trends qualitatively rather than inventing metrics.

This ensures that explanations remain **faithful to system state**, not persuasive fiction.

&nbsp;

## **6\. Trade-offs and Design Decisions**

Several conscious trade-offs were made:

- **Offline synthetic data** is used instead of live telemetry to focus on reasoning correctness rather than infrastructure complexity.
- **Forecasting** uses simple rolling statistics for interpretability and robustness, not black-box ML.
- **No persistent state stores** are used; the prototype prioritizes logic clarity over scalability.
- **Single unified API endpoint** simplifies agent integration and avoids premature surface-area expansion.

These choices reflect a bias toward **pragmatism and correctness first**.
