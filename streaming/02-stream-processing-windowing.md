# Stream Processing: Windowing & State Management

**Tags**: #streaming #windowing #state #aggregation #spark-streaming #real-interview  
**Empresas**: Netflix, Uber, Amazon, Airbnb  
**Dificultad**: Senior  
**Tiempo estimado**: 20 min  

---

## TL;DR

**Stream processing** = process infinite event streams in real-time. **Windowing** = group events by time (5-min window, 1-hour window). **Aggregation** = count, sum within window. **State** = maintain context (user session, order total). **Challenge**: Handle late events, out-of-order data, windowing boundaries.

---

## Concepto Core

- **Qué es**: Windowing = "slice time into chunks", aggregate within each chunk
- **Por qué importa**: Real-time analytics need windows (5-min sales, hourly errors)
- **Principio clave**: Time is tricky in streaming (event time vs processing time)

---

## Window Types

Type 1: TUMBLING (Fixed, non-overlapping)
├─ Window size: 5 minutes
├─

Use: Hourly sales, daily user counts

Type 2: SLIDING (Fixed, overlapping)
├─ Window size: 5 minutes
├─ Slide: 1 minute (every min, new window)
├─

Use: Moving average, real-time metrics

Type 3: SESSION (Dynamic, user-based)
├─ Inactivity gap: 10 minutes
├─ [User A click 00:00] → Start session
├─ [User A click 00:02] → Extend session
├─ [User A no activity > 10 min] → End session
├─ [User A click 00:15] → NEW session
└─ Session length varies

Use: User behavior, session analytics

Type 4: DELAY (After deadline, update)
├─ Window closes at time T
├─ But allow late events until time T + 10 min
├─ Update results as late data arrives
└─ Final result after grace period

Use: Accounting for late-arriving data

text

---

## SQL Windowing Examples

-- Tumbling window: 5-minute sales aggregation
SELECT
TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
TUMBLE_END(event_time, INTERVAL '5' MINUTE) as window_end,
product_id,
COUNT(*) as num_sales,
SUM(amount) as revenue
FROM orders
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE), product_id;

-- Result:
-- window_start window_end product_id num_sales revenue
-- 2024-01-15 10:00:00 2024-01-15 10:05:00 1 100 1000
-- 2024-01-15 10:05:00 2024-01-15 10:10:00 1 110 1100

-- Sliding window: 1-hour moving average (computed every 10 min)
SELECT
HOP_START(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR) as window_start,
HOP_END(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR) as window_end,
AVG(latency_ms) as avg_latency
FROM requests
GROUP BY HOP(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR);

-- Result: Updated every 10 min with 1-hour average

-- Session window: User behavior
SELECT
SESSION_START(event_time, INTERVAL '10' MINUTE) as session_start,
SESSION_END(event_time, INTERVAL '10' MINUTE) as session_end,
user_id,
COUNT(*) as num_events,
ARRAY_AGG(event_type) as event_sequence
FROM user_events
GROUP BY SESSION(event_time, INTERVAL '10' MINUTE), user_id;

-- Result:
-- session_start session_end user_id num_events event_sequence
-- 2024-01-15 10:00:00 2024-01-15 10:12:00 1 5 ['click','add_cart','checkout','pay','confirm']
-- 2024-01-15 10:30:00 2024-01-15 10:35:00 1 2 ['login','view_profile']

text

---

## Time Semantics

Key insight: 3 different times in streaming

EVENT TIME
└─ When the event ACTUALLY happened
└─ Example: Payment processed at 10:05:32

PROCESSING TIME
└─ When the event ARRIVES at Kafka
└─ Example: Arrives at 10:05:35 (3 sec later)

WATERMARK
└─ "We've seen all events up to time X"
└─ Example: Watermark at 10:05:00 = no more events before 10:05:00
└─ Signals: Window complete, time to emit results

Example timeline:
Event happens: 10:05:32 (event time)
Event sent: 10:05:33
Event arrives Kafka: 10:05:35 (processing time)
Latency: 3 seconds

Windowing should use EVENT TIME (not processing time!)
Otherwise: Results depend on network delays, not actual events

Window

Result emission:
├─ Intermediate: As window progresses (allow late)
├─ On-time: When watermark passes window boundary
├─ Late: If data arrives after watermark
└─ Final: After grace period (no more updates)

text

---

## State Management

Challenge: Some operations need CONTEXT
Simple (no state): COUNT events
Result: 5 events in window
With state: User SESSION
Need to track: Which events belong to same user?
from pyspark.sql.functions import col, window, session_window
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("session_tracking").getOrCreate()

Stream: user_events
df = spark.readStream
.format("kafka")
.option("kafka.bootstrap.servers", "localhost:9092")
.option("subscribe", "user_events")
.load()

Parse events
events = df.select(
col("value").cast("string"),
col("timestamp").cast("timestamp").alias("event_time")
)

Session window: 10 min inactivity gap
sessions = events
.groupBy(
session_window(col("event_time"), "10 minutes"),
col("user_id")
)
.agg(
count("*").alias("num_events"),
collect_list("event_type").alias("event_sequence")
)

Result: For each user, per session:
├─ Session start/end
├─ Number of events
└─ Event sequence
State implications:
├─ Must maintain user state across windows
├─ Memory: O(# active sessions)
└─ Timeout: Clean up after 10+ min inactivity
text

---

## Late Data Handling

from pyspark.sql.functions import col, window

Scenario: Event delayed in transit
Event 1: Order placed at 10:05:32, arrives 10:05:35
Event 2: Order placed at 10:06:15, arrives 10:06:18
Event 3: Order placed at 10:05:58, arrives 10:07:00 (LATE!)
Window
├─ Expected events: Event 1, Event 2
├─ Event 3 arrives AFTER window closed
├─ Decision: Ignore or update?
Spark approach: Allow update if within grace period
query = df
.groupBy(
window(col("event_time"), "5 minutes", "1 minute"), # Sliding
col("product_id")
)
.agg(count("*").alias("num_orders"))
.writeStream
.outputMode("update") # Emit updates (not just append)
.option("watermarkDelayThreshold", "10 minutes") # Grace period
.start()

Timeline:
10:05:35 - Event 1 arrives →
10:06:18 - Event 2 arrives →
10:07:00 - Event 3 (LATE) arrives →
10:15:00 - Watermark passes 10:15 →
Output modes:
├─ append: New rows only (settled windows)
├─ update: Modified rows (late data)
└─ complete: All rows (recomputed)
text

---

## Real-World: Real-Time Dashboard

-- Stream: user_events (clicks, purchases, etc)
-- Task: 5-minute dashboard (num events, revenue, top products)

CREATE TABLE dashboard_5min AS
SELECT
TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_time,
COUNT() as total_events,
COUNT(CASE WHEN event_type = 'purchase' THEN 1 END) as num_purchases,
SUM(CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END) as revenue,
APPROX_PERCENTILE(session_duration_sec, 0.95) as p95_session_duration,
ARRAY_AGG(DISTINCT product_id ORDER BY COUNT() DESC LIMIT 5) as top_5_products
FROM user_events
WHERE event_time > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE);

-- Result refreshes every 5 minutes:
-- window_time total_events num_purchases revenue p95_session_duration top_5_products
-- 2024-01-15 10:00:00 50000 5000 50000 420
-- 2024-01-15 10:05:00 52000 5200 52000 425
-- 2024-01-15 10:10:00 51000 5100 51000 422

-- Dashboard UI updates every 5 min with fresh data
-- Latency: ~10 sec (processing + network)

text

---

## Errores Comunes en Entrevista

- **Error**: Windowing by PROCESSING TIME instead of EVENT TIME → **Solución**: Always use event time (network delays skew results)

- **Error**: Ignoring late data → **Solución**: Real systems have delayed events, must handle gracefully

- **Error**: No state cleanup (memory leak)** → **Solución**: Set timeout = TTL for session state

- **Error**: Tumbling windows overlap** → **Solución**: Tumbling = non-overlapping by definition. If overlapping needed = use sliding

---

## Preguntas de Seguimiento

1. **"¿Tumbling vs Sliding?"**
   - Tumbling: Non-overlapping (efficient)
   - Sliding: Overlapping (smoother, more computation)

2. **"¿How to handle out-of-order events?"**
   - Windowing by event time (not process time)
   - Grace period allows late arrivals
   - Watermark signals window close

3. **"¿Session window implementation?"**
   - Track: Last event time per user
   - New event: If gap > inactivity threshold = new session
   - Otherwise: Extend existing session

4. **"¿State size explosion?"**
   - Monitor: Number of active sessions
   - Cleanup: Remove sessions idle > timeout
   - Scale: Move to distributed state store (Redis, RocksDB)

---

## References

- [Spark Structured Streaming - Windowing](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#window-operations-on-event-time)
- [Event Time vs Processing Time - Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/concepts/time-and-watermarks/event_time/)
- [Watermarks & Grace Period - Beam](https://beam.apache.org/documentation/programming-guide/#watermarks-and-late-data)

