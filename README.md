## Table of Contents

<!-- TOC -->
- [Test Scenario](#test-scenario)
  - [What](#what)
  - [Why](#why)
  - [How](#how)
    - [0. Consistent across all layers](#0-consistent-across-all-layers)
    - [Layer 1 — Base behavior (steady state)](#layer-1--base-behavior-steady-state)
    - [Layer 2 — Scaling behavior (high volume)](#layer-2--scaling-behavior-high-volume)
    - [Layer 3 — Throughput stress (high throughput)](#layer-3--throughput-stress-high-throughput)
    - [Layer 4 — Dynamic behavior (heavy state)](#layer-4--dynamic-behavior-heavy-state)
<!-- /TOC -->

# Test Scenario

## What

Characterize the native behavior of vector search engines (Cosmos DB, PostgreSQL) to find the operating characteristics and limits.

Observe how each system behaves across:

- index types (flat, quantized, DiskANN, etc.)
- data scale (small → large)
- throughput regimes (low → high QPS)
- system stress (concurrency, mutations, caching)

## Why

Understand fit-for-purpose boundaries and scaling ceilings

Identify:

- When each index type starts to degrade (latency, recall, cost)
- When each system changes behavior (e.g., tail latency spikes, RU throttling, memory pressure)

Reveal:

- What workloads each engine is naturally optimized for
- Where the breaking points / saturation zones are

Enable:

- Clear guidance of "Use this when…”, “Avoid when…”
- Output is decision heuristics, not benchmark scores.

## How

Progressively stress each system to expose its intrinsic behavior

### 0. Consistent across all layers

**[Data Foundation]**

    Use real embeddings from real text:

    - fixed 1536 dimensions (cosine) for realistic production pressure

    This ensures meaningful vector distribution and index behavior.

**[Monitoring]**

    Use Grafana for monitoring:

    - latency: p50 / p95 / p99
    - throughput: QPS
    - recall: vs exact (flat baseline)
    - resource signals:
      - Cosmos → RU / throttling
      - Postgres → CPU / memory / IO

**[Progressive test layers]**

### Layer 1 — Base behavior (steady state)

Scenarios including:

- small dataset (100K–1M)

Observe:

- latency profile
- recall vs exact (flat)
- index build characteristics

Goal:

- Understand baseline properties per index

### Layer 2 — Scaling behavior (high volume)

Scenarios including:

- increased dataset size
- increased dimension

Observe:

- index growth
- query cost increase (RU / CPU / memory)
- recall stability

Goal:

- Identify scaling curves

### Layer 3 — Throughput stress (high throughput)

Scenarios including:

- increased concurrency (QPS)
- mixed workloads of read-only, read + write

Observe:

- p95/p99 latency divergence
- throttling / queueing / saturation
- failure modes

Goal:

- Find operational limits

### Layer 4 — Dynamic behavior (heavy state)

Scenarios including:

- continuous ingestion + queries
- filtered search + vector search
- caching layer

Observe:

- index stability under mutation
- recall drift
- latency jitter

Goal:

- Reveal real-world behavior under change
