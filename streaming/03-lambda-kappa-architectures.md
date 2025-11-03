# Real-Time vs Batch: Lambda & Kappa Architectures

**Tags**: #architecture #lambda #kappa #real-time #batch #trade-offs #real-interview  
**Empresas**: Netflix, Uber, Amazon, LinkedIn  
**Dificultad**: Senior  
**Tiempo estimado**: 18 min  

---

## TL;DR

**Lambda** = batch + speed layer (dual pipelines). **Kappa** = streaming only (single pipeline, replay capable). **Trade-off**: Lambda = correctness guaranteed but complex; Kappa = simpler but requires perfect streaming. **Reality**: Most companies use Kappa now (Kafka replay makes it work).

---

## Concepto Core

- **Qué es**: Two competing architectures for real-time analytics
- **Por qué importa**: Affects infrastructure, complexity, correctness
- **Principio clave**: Streaming ≠ perfect. Need strategy for corrections

---

## Lambda Architecture

Data Source
│
├─────────────────────────┬──────────────────────────┐
│ │ │
▼ ▼ ▼
BATCH LAYER SPEED LAYER SERVING LAYER
(Slow, Correct) (Fast, Approx) (Merged results)

Batch:
├─ Historical data
├─ Run nightly
├─ Spark/EMR (recompute all)
├─ Takes 2-4 hours
├─ Absolutely correct
└─ Results → Database

Speed:
├─ Real-time events
├─ Kafka consumers
├─ Spark Streaming (incremental)
├─ 10-30 sec latency
├─ Approximate (may have bugs)
└─ Results → Cache/Database

Serving:
├─ Merge both layers
├─ For query: Check speed + batch
├─ If discrepancy: Trust batch
├─ Eventual consistency (eventually correct)

Architecture:
User query
│
├─ Check speed layer (latest)
├─ Check batch layer (correct)
└─ Merge & return

Timeline:
2AM - Batch layer starts (recompute all of yesterday)
6AM - Batch layer finishes (correct results)
6-24h - Speed layer runs (approximate results)
Next 2AM - Speed layer discarded, replaced by batch

text

### Lambda Pros & Cons

Pros:
✓ Correctness guaranteed (batch layer = source of truth)
✓ Easy to fix bugs (re-run batch)
✓ Simple logic (batch = known tools)

Cons:
✗ Dual maintenance (batch code + streaming code)
✗ Complexity (merging two systems)
✗ Latency trade-off (can't be both fast AND correct in dual system)
✗ Data inconsistency (speed ≠ batch for 12-22 hours)

text

---

## Kappa Architecture

Data Source
│
▼
STREAMING PIPELINE (Single system, always correct)
├─ Kafka (immutable event log)
├─ Stream processor (Spark, Flink)
├─ State store (Redis, RocksDB)
└─ Results → Database

Key insight: Kafka is replayed
├─ New bug discovered? Reprocess from start
├─ Logic change? Restart consumer from offset 0
├─ Results regenerated from source events
└─ Simplicity: Single system

Architecture:
Kafka topic "events"
├─ Contains ALL events (ever)
├─ Consumer: Current offset (e.g., event 5M)
├─ To reprocess: Seek to offset 0
├─ Run again → Same result (idempotency)

Timeline:
2AM - Stream processor running (always)
Bug discovered at 8AM
├─ Fix code
├─ Rewind Kafka offset to 0
├─ Reprocess all events with fixed code
├─ 1 hour later: Results correct (no 22 hour delay!)

text

### Kappa Pros & Cons

Pros:
✓ Single system (simpler)
✓ Always correct (just replay)
✓ No consistency window (eventual consistency faster)
✓ Easy to debug (replay from any point)

Cons:
✗ Requires Kafka (immutable log needed)
✗ Replay can be slow (reprocess all history)
✗ State management complexity (tracking replays)
✗ Not suitable for complex batch logic (too slow)

text

---

## Comparison

| Aspect | Lambda | Kappa |
|--------|--------|-------|
| **Complexity** | High (2 systems) | Low (1 system) |
| **Correctness** | Eventual (after batch) | Guaranteed (replay) |
| **Latency** | Mixed (speed + batch) | Consistent (streaming) |
| **Recovery** | Re-run batch | Re-process from Kafka |
| **Maintenance** | Dual codebases | Single codebase |
| **Cost** | High (compute + storage) | Mid (streaming only) |
| **Learning curve** | Steep | Moderate |
| **Best for** | Complex analytics | Real-time systems |

---

## Real-World: E-commerce Orders

### Lambda Approach

Batch Layer (Nightly):
├─ Read: raw_orders (S3, all history)
├─ Compute: revenue, top products, customer segments
├─ Write: aggregate tables (Redshift)
└─ Latency: 2-4 hours

Speed Layer (Real-time):
├─ Read: Kafka orders (latest 30 min)
├─ Compute: Incremental aggregates
├─ Write: Cache (Redis, expires in 30 min)
└─ Latency: 30 seconds

Serving:
├─ Query "revenue today"
├─ Check Redis (latest, ~2 hours old)
├─ Check Redshift (correct, ~22 hours old if batch just ran)
├─ Return: Merge or trust batch

Result:
10AM query:
├─ Speed: $150k (today so far, approximate)
├─ Batch: $120k (yesterday, correct)
└─ Return to user: $150k + caveat "includes last 2 hours, batch updates at 2AM"

Problem:
├─ User sees $150k now
├─ At 2AM batch runs
├─ User sees $145k (lower!)
├─ User confused (went down?)
└─ Reality: Batch correct, speed was approximation

text

### Kappa Approach

Streaming Layer (Always):
├─ Read: Kafka orders (immutable log)
├─ Compute: Incremental aggregates
├─ Write: Database (updated continuously)
└─ Latency: 30 seconds

Query "revenue today":
├─ Check database (always correct)
└─ Return: $150k (guaranteed correct)

Bug discovered:
├─ Revenue calculation wrong
├─ Fix code
├─ Kafka offset = 0 (replay from start)
├─ Recompute: 10 min (all order events)
├─ Database updated with correct revenue
└─ No 22-hour delay!

Result:
├─ Simpler (one system)
├─ Always correct (or always wrong until fixed)
├─ Easier to recover from bugs

text

---

## Decision: Lambda vs Kappa

Choose LAMBDA if:
├─ Complex batch logic (hard to stream)
├─ Batch already optimized
├─ Correctness can wait 12-24 hours
├─ Cost matters (batch cheaper for bulk compute)
└─ Example: Weekly/monthly analytics

Choose KAPPA if:
├─ Real-time required (<1 min latency)
├─ Streaming logic manageable
├─ Immediate bug fixes needed
├─ Consistency > cost
└─ Example: Real-time dashboards, fraud detection, recommendations

Modern trend:
├─ Companies moving to KAPPA
├─ Why: Kafka mature, streaming frameworks better
├─ Trade-off: Simpler > cost savings (for tech-heavy companies)
└─ Netflix, Uber = mostly KAPPA now

text

---

## Hybrid: LAMBDA with KAPPA benefits

Idea: Use Kappa-style replay in Lambda

Architecture:
├─ Batch layer: Nightly, re-runs all (same)
├─ Speed layer: Kafka stream, but ALSO replay capable
├─ On bug: Don't just reprocess speed layer
├─ Solution: Rewind Kafka, reprocess speed + ingest to batch again
└─ Result: Quick correctness (Kappa) with batch safety (Lambda)

Example:
6AM - Bug found in revenue calculation
├─ Fix code
├─ Rewind Kafka to yesterday 2AM
├─ Reprocess: Speed layer replays (30 min)
├─ Ingest to batch database (immediate)
├─ User sees corrected revenue (not 22 hour wait!)

Benefits:
✓ Batch safety (correctness after nightly run)
✓ Kappa speed (immediate replay on bugs)
✓ Best of both worlds
✗ Complexity: Still dual-pipeline, but smart reprocessing

text

---

## Streaming Gotchas

Gotcha 1: Out-of-order events
├─ Event A timestamp 10:00, arrives 10:05
├─ Event B timestamp 10:02, arrives 10:01
├─ If streaming sequential: Wrong order!
├─ Solution: Windowing by event time + watermarks

Gotcha 2: Stateful processing
├─ "How many orders from customer X in last 24 hours?"
├─ Need to maintain state (customer → order list)
├─ If consumer crashes: State lost!
├─ Solution: State store (RocksDB, Redis)

Gotcha 3: Exactly-once delivery
├─ "Process each order exactly once"
├─ Kafka can duplicate
├─ Solution: Idempotent processing + deduplication

Gotcha 4: Backpressure
├─ Producer too fast for consumer
├─ Consumer can't keep up
├─ Solution: Buffer in Kafka, scale consumers

Gotcha 5: Late data
├─ Order arrived late (network delay)
├─ Already closed aggregation window
├─ Should we include?
├─ Solution: Grace period, allow late updates

text

---

## Errores Comunes en Entrevista

- **Error**: "Kappa replaces Lambda completely" → **Solución**: Depends on workload. Complex batch = Lambda still needed

- **Error**: "Streaming is always better" → **Solución**: More complex, higher operational burden

- **Error**: Ignoring consistency window → **Solución**: Lambda = 12-24 hour inconsistency acceptable?

- **Error**: "Batch is dead" → **Solución**: False. Batch still best for bulk analytics (cheaper)

---

## Preguntas de Seguimiento

1. **"¿Cuándo Lambda vs Kappa?"**
   - Lambda: Complex logic, batch optimized
   - Kappa: Real-time, simple logic, replay-capable

2. **"¿Kappa replay performance?"**
   - Depends on Kafka retention (storage) + processor speed
   - If 1 year retention: Replay can take hours
   - Trade-off: Storage vs flexibility

3. **"¿Consistency window acceptable?"**
   - Lambda: Yes, if 12-24h acceptable (most analytics)
   - Real-time dashboards: No, needs Kappa

4. **"¿Netflix/Uber architecture?"**
   - Mostly Kappa now (streaming primary)
   - Batch for heavy compute (ML training, etc)
   - Hybrid: Streaming + batch integration

---

## References

- [Lambda Architecture - Nathan Marz](http://lambda-architecture.net/)
- [Kappa Architecture - Jay Kreps](https://www.confluent.io/blog/questioning-the-lambda-architecture/)
- [Real-time Big Data - O'Reilly](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/)

