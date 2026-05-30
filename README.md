<div align="center">

<br/>

# Proleptic Inference Engine - PIE <br/> Humanet Research

**Eliminating Read-Time Computation Through Causal Dependency Propagation**  
*A Write-Time Pre-computation Runtime for Distributed State Management*

<br/>

[![Status](https://img.shields.io/badge/Status-Research%20%2F%20Early%20Prototype-blueviolet?style=flat-square)](.)
[![Core Language](https://img.shields.io/badge/Runtime-Go-00ADD8?style=flat-square)](.)
[![License](https://img.shields.io/badge/License-Proprietary%20%2F%20All%20Rights%20Reserved-red?style=flat-square)](.)
[![IP Status](https://img.shields.io/badge/IP-Patent%20Pending-orange?style=flat-square)](.)
[![By](https://img.shields.io/badge/By-Humanet-informational?style=flat-square)](https://humanet.uk)

<br/>

> ⚠️ **This repository contains architectural and theoretical documentation only.**  
> Core algorithms, propagation engine source, and CDG runtime are maintained in a private repository.  
> Unauthorized reproduction of the concepts, architecture, or system design described herein may constitute intellectual property infringement.

<br/>

</div>

---

## Abstract

Every distributed system built today operates on the same reactive premise: computation happens when a read is requested. A user queries data; the system retrieves, computes, and responds. This model, inherited unchanged from the earliest database architectures, forces the most expensive operations to coincide precisely with the moment of highest user-facing impact — the read path.

**PIE (Proleptic Inference Engine)** inverts this premise entirely.

By constructing a runtime model of causal dependencies between data nodes — a *Causal Dependency Graph* — PIE deterministically identifies, at write time, every downstream computation that a state change will eventually necessitate. These computations are executed immediately, speculatively, and asynchronously upon write, so that by the time any read is requested, the result already exists in a pre-computed state store.

The result: read operations become O(1) memory lookups regardless of underlying computational complexity. Latency collapses not by making computation faster, but by moving it to where it causes the least harm.

This repository documents the theoretical foundations, formal model, and architectural specification of the PIE runtime. Implementation source is not published here.

---

## Table of Contents

- [Motivation — The Reactive Tax](#motivation--the-reactive-tax)
- [Core Insight](#core-insight)
- [Formal Model](#formal-model)
- [Architecture](#architecture)
- [The Propagation Algorithm](#the-propagation-algorithm)
- [Developer Interface](#developer-interface)
- [Performance Model](#performance-model)
- [Comparison with Existing Approaches](#comparison-with-existing-approaches)
- [Limitations & Boundary Conditions](#limitations--boundary-conditions)
- [Real-World Impact Estimates](#real-world-impact-estimates)
- [Research Foundations](#research-foundations)
- [IP & Legal Notice](#ip--legal-notice)
- [Status & Roadmap](#status--roadmap)

---

## Motivation — The Reactive Tax

### How compute is wasted today

Consider a mid-scale e-commerce platform. At any given second, hundreds of users are viewing product pages. Each view triggers the following chain:

```
HTTP Request arrives
        ↓
Route matched → handler invoked
        ↓
ORM query constructed → sent to database
        ↓
Database: index traversal (B-Tree, O(log N))
        ↓
Row fetch → deserialization
        ↓
Business logic: price × quantity, tax calculation, discount rules
        ↓
Serialization → HTTP response
        ↓
User sees the number
```

If the product price has not changed since the last request — and in the vast majority of cases, it has not — **every single step of this chain is pure redundant recomputation.** The same query, the same traversal, the same arithmetic, the same serialization, performed thousands of times per minute to produce an identical result.

### The scale of the problem

| Workload Type | Estimated Read/Write Ratio | Redundant Computation |
|---|---|---|
| E-commerce product pages | 10,000 : 1 | ~99.99% of reads compute stale-equivalent results |
| SaaS analytics dashboards | 500 : 1 | ~99.8% |
| Financial portfolio views | 1,000 : 1 | ~99.9% |
| Leaderboards / ranking systems | 50,000 : 1 | ~99.998% |

The system pays full computational cost on the read path — the worst possible moment — for work that could have been done once, at write time, and cached indefinitely until the next state change.

This is the **Reactive Tax**: the systemic cost imposed by computing results on demand rather than on cause.

---

## Core Insight

PIE is built on a single observation drawn from both compiler theory and CPU microarchitecture:

> *Computation is not inherently tied to the moment it is requested. It is tied to the moment its inputs become available.*

Modern CPUs have known this since the 1990s. **Speculative execution** and **out-of-order execution** allow processors to compute results before they are explicitly needed, discarding incorrect speculations and retaining correct ones. The performance gains are enormous — modern CPUs would be 3–5× slower without these mechanisms.

PIE applies the same principle at the distributed systems layer:

- **Inputs becoming available** = a write operation changing a data node
- **Computation** = any function whose output depends on that node
- **Speculation** = replaced by *determinism* — because dependency graphs are known, no speculation is needed

The key distinction from CPU speculative execution: PIE does not guess. It *knows* which computations are affected by a state change, because those dependencies are explicitly modeled in the Causal Dependency Graph.

```
CPU speculative execution:  PREDICT → execute → verify → discard if wrong
PIE causal propagation:     DETECT change → traverse graph → execute → store
```

No wasted work. No incorrect results. Pure deterministic pre-computation.

---

## Formal Model

### Definitions

Let a PIE-managed system be described by a directed acyclic graph:

$$G = (V, E)$$

Where:

- $V = \{v_1, v_2, \ldots, v_n\}$ is the set of **data nodes** — any piece of state that can change or be computed
- $E \subseteq V \times V$ is the set of **causal edges** — a directed edge $(v_i, v_j) \in E$ means: *"the value of $v_j$ depends on the value of $v_i$"*

### Node classification

$$V = V_{\text{source}} \cup V_{\text{derived}}$$

- $V_{\text{source}}$: Nodes with no incoming edges — raw data written directly by application logic (prices, inventory counts, user inputs)
- $V_{\text{derived}}$: Nodes with at least one incoming edge — values computed from other nodes (totals, aggregates, formatted outputs)

### The propagation function

When any source node $v_i \in V_{\text{source}}$ is updated, PIE computes the **affected set**:

$$\mathcal{A}(v_i) = \{ v_j \in V \mid \exists \text{ directed path from } v_i \text{ to } v_j \text{ in } G \}$$

PIE then executes the recomputation sequence in **topological order** over $\mathcal{A}(v_i)$, ensuring that no derived node is recomputed before all of its dependencies have been updated:

$$\sigma = \text{TopologicalSort}(\mathcal{A}(v_i))$$

$$\forall v_j \in \sigma: \quad \text{Recompute}(v_j) \rightarrow \text{Store}(v_j)$$

### Read complexity

After propagation completes, any read of any node — source or derived — is a pure memory lookup:

$$\text{Read}(v_j) = \text{Store.Get}(v_j) \quad \Rightarrow \quad O(1)$$

The computational complexity of the underlying function that produces $v_j$ is **entirely absorbed into the write path**, where it is amortized across the write frequency rather than paid in full on every read.

### Consistency guarantee

PIE provides **causal consistency**: a read of any derived node $v_j$ always reflects the most recent write to all of its ancestors in $G$, provided propagation has completed. The propagation is **synchronous with respect to the dependency graph** — all downstream values are updated before the write operation is acknowledged as complete.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Application Layer                             │
│              (Any language — Go, Node, Python, Ruby...)              │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  PIE SDK (thin client library)
┌───────────────────────────▼──────────────────────────────────────────┐
│                        PIE Runtime                                   │
│                                                                      │
│  ┌─────────────────────┐     ┌──────────────────────────────────┐   │
│  │   CDG Registry      │     │     Propagation Engine           │   │
│  │                     │     │                                  │   │
│  │  Stores the causal  │     │  On every write:                 │   │
│  │  dependency graph   │     │  1. Identify affected set A(vi)  │   │
│  │  G = (V, E)         │     │  2. Topological sort             │   │
│  │                     │     │  3. Recompute each node          │   │
│  │  Built at startup   │     │  4. Write to State Store         │   │
│  │  from declarations  │     │  5. Acknowledge write            │   │
│  └─────────────────────┘     └──────────────────────────────────┘   │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Computed State Store                        │  │
│  │                                                                │  │
│  │   Key-value store of all pre-computed node values.            │  │
│  │   Every read hits this layer only — no DB, no logic.          │  │
│  │   Backed by pluggable adapters: Redis, in-memory, RocksDB.    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CDG Learner (optional)                      │  │
│  │                                                                │  │
│  │   Observes application I/O patterns to suggest or             │  │
│  │   automatically infer dependency relationships.                │  │
│  │   Manual declarations always take precedence.                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────────┐
│                    Existing Data Layer (unchanged)                   │
│          PostgreSQL · MySQL · MongoDB · Redis · Any database         │
└──────────────────────────────────────────────────────────────────────┘
```

PIE sits as a **middleware runtime** between application logic and the data layer. It requires no changes to the underlying database, no schema migrations, and no modifications to existing infrastructure. The application interacts with PIE through a thin SDK; PIE manages all propagation and state storage transparently.

---

## The Propagation Algorithm

*The following describes the behavior and guarantees of the propagation engine. Internal implementation details are not disclosed.*

### Write path

```
Application calls: pie.Write("product_price", 99.00)

PIE Runtime:
  1. Write value to source node → State Store
  2. Query CDG Registry: affected_set = A("product_price")
     → ["cart_subtotal", "vat_amount", "cart_total",
        "invoice_line", "accounting_entry", "search_index_price"]
  3. Topological sort of affected_set
     → ["cart_subtotal", "vat_amount"] → ["cart_total"] 
        → ["invoice_line"] → ["accounting_entry", "search_index_price"]
  4. Execute each node's registered function in order
  5. Write each result to State Store
  6. Acknowledge write to application
```

### Read path

```
Application calls: pie.Read("cart_total")

PIE Runtime:
  1. State Store lookup → return pre-computed value
  2. Done.

No database query. No computation. No network round-trip.
```

### Cycle detection

The CDG is validated as a DAG at registration time. Any circular dependency is detected and rejected before the runtime starts, preventing infinite propagation loops.

---

## Developer Interface

The following illustrates the **conceptual interface** of the PIE SDK. This is the surface-level API design; internal implementation is not shown.

```go
// ── Dependency Declaration ──────────────────────────────────────────

pie.Source("product_price")
pie.Source("cart_quantity")
pie.Source("tax_rate")

pie.Derive("cart_subtotal",
    pie.DependsOn("product_price", "cart_quantity"),
    func(ctx pie.Context) interface{} {
        price := ctx.Get("product_price").(float64)
        qty   := ctx.Get("cart_quantity").(float64)
        return price * qty
    },
)

pie.Derive("vat_amount",
    pie.DependsOn("cart_subtotal", "tax_rate"),
    func(ctx pie.Context) interface{} {
        subtotal := ctx.Get("cart_subtotal").(float64)
        rate     := ctx.Get("tax_rate").(float64)
        return subtotal * rate
    },
)

pie.Derive("cart_total",
    pie.DependsOn("cart_subtotal", "vat_amount"),
    func(ctx pie.Context) interface{} {
        return ctx.Get("cart_subtotal").(float64) +
               ctx.Get("vat_amount").(float64)
    },
)

// ── Runtime Usage ────────────────────────────────────────────────────

// A price update triggers full graph propagation automatically
pie.Write("product_price", 129.00)

// All derived values are already updated by the time this executes
total := pie.Read("cart_total")   // → instant, O(1)
vat   := pie.Read("vat_amount")   // → instant, O(1)
```

The developer declares *what depends on what*, once. PIE handles all propagation, ordering, and storage from that point forward.

---

## Performance Model

### Theoretical read latency

For any derived node $v_j$, read latency under PIE is bounded by the State Store lookup time:

$$T_{\text{read}}^{\text{PIE}} = T_{\text{store\_lookup}} \approx 0.1\text{ms} \quad \text{(in-memory store)}$$

Under a conventional reactive system, the equivalent read incurs:

$$T_{\text{read}}^{\text{reactive}} = T_{\text{db\_query}} + T_{\text{compute}} + T_{\text{serialize}}$$

For a moderately complex derived value (multi-table join + business logic):

$$T_{\text{read}}^{\text{reactive}} \approx 40\text{ms} – 800\text{ms}$$

### Write path overhead

PIE adds propagation overhead to each write. This is the fundamental trade-off:

$$T_{\text{write}}^{\text{PIE}} = T_{\text{write}}^{\text{baseline}} + T_{\text{propagation}}$$

$$T_{\text{propagation}} = \sum_{v_j \in \sigma} T_{\text{compute}}(v_j)$$

For systems where reads vastly outnumber writes — which describes the overwhelming majority of production workloads — this trade-off is strongly favorable.

### Break-even analysis

PIE delivers net positive performance whenever:

$$\frac{R}{W} > \frac{T_{\text{propagation}}}{T_{\text{read}}^{\text{reactive}} - T_{\text{read}}^{\text{PIE}}}$$

Where $R/W$ is the read-to-write ratio. For typical web workloads with $R/W > 100$, PIE operates well within the favorable regime across all realistic propagation costs.

---

## Comparison with Existing Approaches

| Approach | Mechanism | Why It Falls Short |
|---|---|---|
| **Redis cache** | Stores computed values manually | Developer must invalidate manually; stale data risk; no dependency awareness |
| **Materialized Views** | DB pre-computes query results | Static SQL only; no arbitrary business logic; schema-bound; no cross-service propagation |
| **Event Sourcing** | Broadcasts state change events | Notifies consumers but does not pre-compute results; each consumer still computes independently |
| **GraphQL DataLoader** | Batches and deduplicates reads | Reduces N+1 queries; does not eliminate computation; still reactive |
| **CQRS** | Separates read and write models | Architectural pattern, not a runtime; developer manually maintains read model; eventual consistency managed by hand |
| **CDN Edge Caching** | Caches full HTTP responses | Page-level granularity only; cannot cache partial state; invalidation is coarse |
| **PIE** | Causal graph propagation at write time | **Automatic, dependency-aware, arbitrary-logic, cross-service, always consistent** |

No existing tool combines: automatic dependency tracking + arbitrary computation functions + write-time propagation + cross-service scope + strong causal consistency.

---

## Limitations & Boundary Conditions

Intellectual honesty demands that the constraints of this model be stated clearly.

**Write-heavy workloads**  
PIE's performance model inverts under extreme write dominance. If writes outnumber reads, propagation cost may exceed the savings on the read path. Workloads with $R/W < 10$ should be profiled before adoption.

**High-fan-out graphs**  
A single source node with thousands of dependents creates a large affected set per write. Propagation time scales with fan-out. Graph design discipline is required to keep propagation bounded.

**Cyclic dependency prevention**  
The CDG must remain acyclic. PIE enforces this at registration time, but it places a design constraint on the application developer: derived values cannot feed back into their own ancestors.

**Cold graph startup**  
On system initialization, the State Store is empty. PIE must perform a full graph computation pass before reads can be served from pre-computed values. Warm-up time is proportional to graph size and computation cost.

**Non-deterministic functions**  
Functions that return different values for the same inputs (e.g., `time.Now()`, `rand.Float64()`) cannot be safely used as derived node computations, as propagation would produce inconsistent results.

---

## Real-World Impact Estimates

The following estimates are derived from the performance model applied to representative production workload profiles. They are theoretical projections, not measured benchmarks from deployed systems.

### E-commerce platform (100k daily active users)

| Metric | Baseline (Reactive) | With PIE | Change |
|---|---|---|---|
| Average API response time | 380ms | 18ms | −95% |
| Database queries per minute | 48,000 | 2,900 | −94% |
| Application server count | 12 | 3 | −75% |
| Monthly compute cost (est.) | $1,800 | $420 | −77% |
| P99 latency | 1,200ms | 45ms | −96% |

### SaaS analytics dashboard (500 concurrent users)

| Metric | Baseline | With PIE | Change |
|---|---|---|---|
| Dashboard load time | 2.4s | 0.09s | −96% |
| DB CPU utilization | 78% | 11% | −86% |
| Cache miss rate | 34% | <0.1% | −99.7% |

> **Disclaimer:** These figures represent theoretical upper bounds under idealized conditions. Actual results depend on workload characteristics, graph topology, infrastructure quality, and implementation specifics. Real-world deployments will be benchmarked and published separately.

---

## Research Foundations

PIE synthesizes principles from several established fields. The following references point to the foundational literature; no specific algorithmic claims are attributed to these works.

**Dataflow programming and reactive computation**  
The concept of computation triggered by data availability rather than explicit invocation traces to dataflow architectures (Dennis, 1974) and later reactive programming models (Bainomugisha et al., 2013).

**Incremental computation**  
The problem of recomputing only the affected portions of a computation when inputs change is studied formally in the incremental computation literature (Acar, 2005 — self-adjusting computation; Hammer et al., 2014 — Adapton).

**Topological ordering in dependency resolution**  
Kahn's algorithm (1962) and DFS-based topological sort (Cormen et al., *Introduction to Algorithms*) form the mathematical basis for dependency-ordered propagation.

**Materialized view maintenance**  
The database literature on incremental view maintenance (Gupta & Mumick, 1995) addresses a related but narrower problem: maintaining precomputed SQL query results under updates.

**Speculative and out-of-order execution**  
The CPU microarchitecture literature on speculative execution (Tomasulo, 1967; Smith & Pleszkun, 1985) provides the conceptual analogy for write-time pre-computation.

PIE's contribution is the synthesis of these principles into a general-purpose, language-agnostic, infrastructure-layer runtime — a layer that does not exist in any current open-source or commercial offering.

---

## IP & Legal Notice

```
Copyright © 2026 Humanet (humanet.uk). All Rights Reserved.

The architecture, algorithms, system design, formal model, and 
implementation described in this repository constitute proprietary 
intellectual property of Humanet.

Patent application pending.

This documentation is published for research transparency and 
investor review purposes only. No license — express or implied — 
is granted to implement, reproduce, distribute, or create derivative 
works based on the PIE architecture or its constituent components.

Unauthorized commercial use of the concepts, designs, or methods 
described herein may constitute infringement of intellectual property 
rights and will be pursued accordingly.

For licensing inquiries: legal@humanet.uk
```

---

## Status & Roadmap

```
Phase 0 — Formal Model & Architecture Spec    ████████████████████  COMPLETE
Phase 1 — CDG Core & Propagation Engine       ████████░░░░░░░░░░░░  IN PROGRESS
Phase 2 — Go SDK & Developer Interface        ████░░░░░░░░░░░░░░░░  IN PROGRESS
Phase 3 — Redis & PostgreSQL Adapters         ░░░░░░░░░░░░░░░░░░░░  PLANNED
Phase 4 — CDG Learner (auto-inference)        ░░░░░░░░░░░░░░░░░░░░  PLANNED
Phase 5 — Production Benchmark & Paper        ░░░░░░░░░░░░░░░░░░░░  FUTURE
Phase 6 — Managed Cloud Offering              ░░░░░░░░░░░░░░░░░░░░  FUTURE
```

### Near-term milestones

- [ ] Core propagation engine — internal benchmark against Redis naive caching
- [ ] Go SDK v0.1 — manual dependency declaration API
- [ ] First production deployment on Humanet infrastructure
- [ ] Measured performance report (real workload, real numbers)
- [ ] Academic preprint submission

---

## Contact & Collaboration

PIE is developed by [Humanet](https://humanet.uk) — a cloud infrastructure company building the next generation of intelligent hosting infrastructure.

If you are a researcher working in incremental computation, dataflow systems, or distributed state management, or an enterprise with workloads that match the profile described above, collaboration and pilot program inquiries are welcome.

**Contact:** research@humanet.uk

<br/>

---

<div align="center">

<br/>

*"Every system that computes on read is paying a tax it doesn't have to.*  
*PIE collects that debt at write time — when no one is waiting."*

<br/>

**PIE — Proleptic Inference Engine**  
Architectural Specification · Proprietary Research · © 2026 Humanet

<br/>

</div>
