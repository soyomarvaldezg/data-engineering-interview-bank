# Apache Kafka: Streaming Architecture & Patterns

**Tags**: #streaming #kafka #event-driven #pub-sub #real-time #real-interview  
**Empresas**: Netflix, Uber, Amazon, LinkedIn, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 22 min  

---

## TL;DR

**Kafka** = distributed event streaming platform. **Core**: Topics (channels) → Producers (write) → Consumers (read). **Key feature**: Durability (events stored for X days), Replay (read history), Partitions (parallelism). **Benefits**: Decouples producers/consumers, scales to trillion events/day. **Trade-off**: Complexity vs throughput guarantees.

---

## Concepto Core

- **Qué es**: Kafka = distributed log. Write once, read many times
- **Por qué importa**: Real-time pipelines need Kafka (or equivalent). Industry standard
- **Principio clave**: Event stream = immutable log (append-only, never delete)

---

## Architecture

┌─────────────────────────────────────────────────────┐
│ KAFKA CLUSTER (Brokers) │
├─────────────────────────────────────────────────────┤
│ │
│ Topic: "orders" (3 partitions) │
│ ├─ Partition 0: [msg0, msg1, msg2, ...] │
│ │ Offset: 0, 1, 2, ... │
│ │ Replicas: Broker 1 (leader), Broker 2 (replica)│
│ │ │
│ ├─ Partition 1: [msg100, msg101, ...] │
│ │ Offset: 0, 1, 2, ... │
│ │ Replicas: Broker 2 (leader), Broker 3 (replica)│
│ │ │
│ └─ Partition 2: [msg200, msg201, ...] │
│ Offset: 0, 1, 2, ... │
│ Replicas: Broker 3 (leader), Broker 1 (replica)│
│ │
│ Retention: 7 days (then auto-delete) │
│ │
└─────────────────────────────────────────────────────┘

text

---

## Producers & Consumers

from kafka import KafkaProducer, KafkaConsumer
import json

===== PRODUCER =====
producer = KafkaProducer(
bootstrap_servers=['localhost:9092'],
value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

Send event (order)
event = {
'order_id': 12345,
'customer_id': 999,
'amount': 150.00,
'timestamp': '2024-01-15T10:30:00Z'
}

future = producer.send('orders', value=event)
record_metadata = future.get(timeout=10)

print(f"Sent to partition {record_metadata.partition}, offset {record_metadata.offset}")

Output: Sent to partition 1, offset 42
===== CONSUMER =====
consumer = KafkaConsumer(
'orders',
bootstrap_servers=['localhost:9092'],
group_id='analytics-consumer-group', # Consumer group
auto_offset_reset='earliest', # Start from beginning if no offset
value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

Read events
for message in consumer:
event = message.value
print(f"Order {event['order_id']}: ${event['amount']}")
# Process event (update dashboard, check fraud, etc)

text
# Offset automatically committed (tracks where we are)
# If consumer crashes: Restart from last committed offset
text

---

## Topics, Partitions, Consumer Groups

Topic: Logical channel (e.g., "orders")
├─ Partition: Physical split for parallelism
│ └─ Partition 0: orders[0-99]
│ └─ Partition 1: orders[100-199]
│ └─ Partition 2: orders[200-299]
│
├─ Each partition: Independent log (ordered within partition)
├─ Key determines partition: hash(order_id) % num_partitions
│ └─ Same order_id always goes to same partition (ordering!)
│
└─ Replication: Each partition replicated (default 3 replicas)
└─ Leader: Handles reads/writes
└─ Replica: Standby (if leader dies, replica becomes leader)

Consumer Group: Multiple consumers reading same topic
├─ Consumer 1: Reads partition 0
├─ Consumer 2: Reads partition 1
├─ Consumer 3: Reads partition 2
├─ Parallelism: 3 partitions = up to 3 consumers in parallel
├─ If consumer crashes: Partition reassigned to remaining consumers
└─ Offset tracking: Each consumer group independently tracks offset

Example:
Topic: "orders" (3 partitions, 10M messages/day)

Consumer Group 1 ("analytics"):
├─ Consumer 1a: Partition 0, offset 5000000
├─ Consumer 1b: Partition 1, offset 5000000
└─ Consumer 1c: Partition 2, offset 5000000
└─ Can restart from offset 5000000 (no data loss)

Consumer Group 2 ("fraud-detection"):
├─ Consumer 2a: Partition 0, offset 3000000
├─ Consumer 2b: Partition 1, offset 3000000
└─ Consumer 2c: Partition 2, offset 3000000
└─ Different offset (independent processing)

text

---

## Offset Management

from kafka import KafkaConsumer, OffsetAndMetadata

consumer = KafkaConsumer(
'orders',
bootstrap_servers=['localhost:9092'],
group_id='my-group',
enable_auto_commit=False # Manual offset management
)

for message in consumer:
try:
# Process message
process_order(message.value)

text
    # Only commit if processing succeeded
    consumer.commit(
        offsets={
            message.topic_partition: OffsetAndMetadata(
                message.offset + 1,
                metadata='processed successfully'
            )
        }
    )
    print(f"Committed offset {message.offset}")

except Exception as e:
    # Don't commit on error
    print(f"Error processing: {e}, will retry next time")
    # Next time consumer starts: Reread from last committed offset
    # Automatic reprocessing (idempotency required!)
Timeline:
1. Read offset 100
2. Process (success)
3. Commit offset 101 (next read will start from 101)
4. Consumer crashes
5. Restart: Begin from offset 101 (no dupes, no missed)
text

---

## Kafka Patterns

### Pattern 1: Fan-out (Multiple consumers, same data)

Producer → Topic: orders
├─ Consumer 1 (Analytics): All orders
├─ Consumer 2 (Fraud detection): All orders
└─ Consumer 3 (Warehouse): All orders

Each consumer independently processes.
Use different consumer groups.

Benefits:
✓ Producers don't know about consumers
✓ Consumers can process independently
✓ Can add new consumer anytime (read from beginning)

text

### Pattern 2: Event Sourcing

Store all events (append-only)
topic = "customer_events"

Event 1: Customer created
{'event_type': 'created', 'customer_id': 1, 'name': 'Alice', 'timestamp': '2024-01-01'}

Event 2: Email changed
{'event_type': 'email_changed', 'customer_id': 1, 'email': 'alice@example.com', 'timestamp': '2024-01-15'}

Event 3: Subscription upgraded
{'event_type': 'subscription_upgraded', 'customer_id': 1, 'plan': 'premium', 'timestamp': '2024-01-20'}

Current state: Replay all events
def get_customer_state(customer_id):
consumer = KafkaConsumer(bootstrap_servers=['localhost:9092'])
consumer.subscribe(['customer_events'])

text
state = {}
for message in consumer:
    event = message.value
    if event['customer_id'] == customer_id:
        apply_event_to_state(state, event)

return state
state = get_customer_state(1)

{'customer_id': 1, 'name': 'Alice', 'email': 'alice@example.com', 'plan': 'premium'}
Benefits:
✓ Complete audit trail (all changes tracked)
✓ Can replay history (debug, recover)
✓ Single source of truth (events)

text

### Pattern 3: CQRS (Command Query Responsibility Segregation)

Command: Write to Kafka (events)
Query: Read from derived views (materialized)

Write Flow:
User command → Event → Kafka → Database updated

Read Flow:
User query → Materialized view (pre-computed) → Fast response

Example:
Command: "Add item to cart"
└─ Event: {'event': 'item_added_to_cart', 'user_id': 123, 'item_id': 456}
└─ Written to topic "cart_events"
└─ Consumer updates materialized table "user_carts"

Query: "What's in user 123's cart?"
└─ Select * from user_carts where user_id = 123
└─ Response: [item_456, item_789] (instant, no computation)

Benefits:
✓ Writes optimized (just append to Kafka)
✓ Reads optimized (pre-computed materialized views)
✓ Independent scaling (write ≠ read throughput)

text

---

## Real-World: Payment Transaction Processing

from kafka import KafkaProducer, KafkaConsumer
import json

===== PRODUCER: Payment Service =====
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

def process_payment(order_id, amount, card):
"""Process payment, emit event"""
try:
# Process payment (call payment gateway)
transaction_id = call_payment_gateway(card, amount)

text
    event = {
        'event': 'payment_success',
        'order_id': order_id,
        'amount': amount,
        'transaction_id': transaction_id,
        'timestamp': datetime.now()
    }
    
    producer.send('payments', value=event)
    return transaction_id

except Exception as e:
    event = {
        'event': 'payment_failed',
        'order_id': order_id,
        'error': str(e),
        'timestamp': datetime.now()
    }
    producer.send('payments', value=event)
    raise
===== CONSUMER 1: Order Fulfillment =====
consumer1 = KafkaConsumer(
'payments',
bootstrap_servers=['localhost:9092'],
group_id='fulfillment-group'
)

for message in consumer1:
event = message.value
if event['event'] == 'payment_success':
order_id = event['order_id']
# Ship order
print(f"Shipping order {order_id}")
update_order_status(order_id, 'shipped')

===== CONSUMER 2: Fraud Detection =====
consumer2 = KafkaConsumer(
'payments',
bootstrap_servers=['localhost:9092'],
group_id='fraud-detection-group'
)

for message in consumer2:
event = message.value
if event['event'] == 'payment_success':
# Check for fraud
if is_suspicious(event):
print(f"ALERT: Suspicious transaction {event['transaction_id']}")
flag_for_review(event)

===== CONSUMER 3: Analytics =====
consumer3 = KafkaConsumer(
'payments',
bootstrap_servers=['localhost:9092'],
group_id='analytics-group'
)

daily_revenue = 0
for message in consumer3:
event = message.value
if event['event'] == 'payment_success':
daily_revenue += event['amount']
print(f"Daily revenue so far: ${daily_revenue}")
update_metrics(daily_revenue)

All consumers independent:
- If fulfillment fails: Other consumers still process
- If fraud detection down: Payments still processed
- Decoupling = resilience
text

---

## Exactly-Once Semantics

Challenge: Guarantee each payment processed EXACTLY once
(not 0 times = lost payment, not 2+ times = duplicate charge)

Kafka provides: At-least-once (messages might duplicate)

Solution 1: Idempotent consumer
├─ Process same event multiple times
├─ Result always same (idempotent!)
└─ Database deduplicates (unique constraint)

Solution 2: Idempotent producer
├─ Kafka tracks producer ID + sequence number
├─ Duplicate sends detected, deduplicated
└─ Exactly-once on write

Solution 3: Transactional processing
├─ Read from Kafka (offset X)
├─ Process message
├─ Write result + commit offset atomically
└─ If crash: Both rollback (no dupes)

Implementation:
consumer.enable_auto_commit = False
for message in consumer:
result = process(message.value) # Idempotent
save_to_db(result, message.offset) # With offset
consumer.commit(message.offset) # Commit only after save

If crash between save and commit:
- Restart reads from last committed offset
- Reprocess same message
- Idempotency + unique constraint = no dupes
text

---

## Errores Comunes en Entrevista

- **Error**: "Kafka stores all events forever" → **Solución**: Default retention 7 days (configurable)

- **Error**: "Consumers compete for messages" → **Solución**: Depends on consumer groups (same group = split; different group = all get)

- **Error**: Ignoring offset management → **Solución**: Offset = "where are we", critical for recovery

- **Error**: "Kafka is slow" → **Solución**: Kafka = 1M events/sec easy. Most latency from your code

---

## Preguntas de Seguimiento

1. **"¿Kafka vs RabbitMQ?"**
   - Kafka: Distributed, durable, large scale
   - RabbitMQ: Simple, but not distributed

2. **"¿Partitions vs Topics?"**
   - Topic: Logical channel
   - Partition: Physical split (parallelism)

3. **"¿Consumer group rebalancing?"**
   - Consumer joins/leaves: Partitions reassigned
   - Brief pause during rebalance
   - Can cause brief latency spike

4. **"¿Performance: partition count?"**
   - More partitions = more parallelism
   - But more overhead
   - Tune based on throughput needs

---

## References

- [Apache Kafka - Official Docs](https://kafka.apache.org/documentation/)
- [Kafka Best Practices - Confluent](https://www.confluent.io/blog/design-patterns-for-apache-kafka/)
- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)

