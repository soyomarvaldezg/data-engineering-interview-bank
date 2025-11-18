# Apache Kafka: Arquitectura de Streaming y Patrones

**Tags**: #streaming #kafka #event-driven #pub-sub #real-time #real-interview

---

## TL;DR

**Kafka** = plataforma de streaming de eventos distribuida. **Core**: Topics (canales) → Productores (escriben) → Consumidores (leen). **Características clave**: Durabilidad (eventos almacenados por X días), Replay (leer historial), Particiones (paralelismo). **Ventajas**: Desacopla productores/consumidores, escala a billones de eventos/día. **Compromiso**: Complejidad vs garantías de throughput.

---

## Concepto Core

- **Qué es**: Kafka = log distribuido. Escribe una vez, lee muchas veces
- **Por qué importa**: Los pipelines en tiempo real necesitan Kafka (o equivalente). Estándar de la industria
- **Principio clave**: Event stream = log inmutable (solo append, nunca elimina)

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│        CLUSTER KAFKA (Brokers)         │
├─────────────────────────────────────────────┤
│                                         │
│ Topic: "orders" (3 particiones)         │
│ ├─ Partición 0: [msg0, msg1, msg2, ...] │
│ │  Offset: 0, 1, 2, ...                │
│ │  Réplicas: Broker 1 (líder), Broker 2 (réplica)│
│ │                                         │
│ ├─ Partición 1: [msg100, msg101, ...]   │
│ │  Offset: 0, 1, 2, ...                │
│ │  Réplicas: Broker 2 (líder), Broker 3 (réplica)│
│ │                                         │
│ └─ Partición 2: [msg200, msg201, ...]   │
│    Offset: 0, 1, 2, ...                │
│    Réplicas: Broker 3 (líder), Broker 1 (réplica)│
│                                         │
│ Retención: 7 días (luego auto-elimina)    │
│                                         │
└─────────────────────────────────────────────┘
```

---

## Productores y Consumidores

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# ===== PRODUCTOR =====
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Enviar evento (pedido)
event = {
    'order_id': 12345,
    'customer_id': 999,
    'amount': 150.00,
    'timestamp': '2024-01-15T10:30:00Z'
}

future = producer.send('orders', value=event)
record_metadata = future.get(timeout=10)

print(f"Enviado a partición {record_metadata.partition}, offset {record_metadata.offset}")
# Output: Enviado a partición 1, offset 42
```

```python
# ===== CONSUMIDOR =====
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-consumer-group',  # Grupo de consumidores
    auto_offset_reset='earliest',  # Comenzar desde el principio si no hay offset
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Leer eventos
for message in consumer:
    event = message.value
    print(f"Pedido {event['order_id']}: ${event['amount']}")

    # Procesar evento (actualizar dashboard, detectar fraude, etc.)

# Offset confirmado automáticamente (rastrea dónde estamos)

# Si el consumidor falla: Reinicia desde el último offset confirmado
```

---

## Topics, Particiones y Grupos de Consumidores

Topic: Canal lógico (ej: "orders")
├─ Partición: División física para paralelismo
│ └─ Partición 0: pedidos[0-99]
│ └─ Partición 1: pedidos[100-199]
│ └─ Partición 2: pedidos[200-299]
│
├─ Cada partición: Log independiente (ordenado dentro de partición)
├─ La clave determina la partición: hash(order_id) % num_particiones
│ └─ El mismo order_id siempre va a la misma partición (¡ordenamiento!)
│
├─ Replicación: Cada partición replicada (por defecto 3 réplicas)
└─ Líder: Maneja lecturas/escrituras
└─ Réplica: En espera (si el líder falla, la réplica se convierte en líder)

Grupo de Consumidores: Múltiples consumidores leyendo el mismo topic
├─ Consumidor 1: Lee partición 0
├─ Consumidor 2: Lee partición 1
├─ Consumidor 3: Lee partición 2
├─ Paralelismo: 3 particiones = hasta 3 consumidores en paralelo
├─ Si un consumidor falla: La partición se reasigna a los consumidores restantes
└─ Cada grupo de consumidores rastrea offsets independientemente

Ejemplo:
Topic: "orders" (3 particiones, 10M mensajes/día)

Grupo de Consumidores 1 ("analytics"):
├─ Consumidor 1a: Partición 0, offset 5000000
├─ Consumidor 1b: Partición 1, offset 5000000
└─ Consumidor 1c: Partición 2, offset 5000000
└─ Puede reiniciar desde offset 5000000 (sin pérdida de datos)

Grupo de Consumidores 2 ("fraud-detection"):
├─ Consumidor 2a: Partición 0, offset 3000000
├─ Consumidor 2b: Partición 1, offset 3000000
└─ Consumidor 2c: Partición 2, offset 3000000
└─ Offset diferente (procesamiento dependiente)

---

## Gestión de Offsets

```python
from kafka import KafkaConsumer, OffsetAndMetadata

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='my-group',
    enable_auto_commit=False  # Gestión manual de offsets
)

for message in consumer:
    try:
        # Procesar mensaje
        process_order(message.value)

        # Solo confirmar si el procesamiento tuvo éxito
        consumer.commit(
            offsets={
                message.topic_partition: OffsetAndMetadata(
                    message.offset + 1,
                    metadata='procesado exitosamente'
                )
            }
        )
        print(f"Confirmado offset {message.offset}")

    except Exception as e:  # No confirmar en caso de error
        print(f"Error procesando: {e}, se reintentará la próxima vez")
        # La próxima vez que el consumidor inicie: Releer desde el último offset confirmado
        # Reprocesamiento automático (se requiere idempotencia)
```

Cronología:

1. Leer offset 100
2. Procesar (éxito)
3. Confirmar offset 101 (la próxima lectura comenzará desde 101)
4. El consumidor falla
5. Reiniciar: Comenzar desde offset 101 (sin duplicados, sin pérdidas)

---

## Patrones de Kafka

### Patrón 1: Fan-out (Múltiples consumidores, mismos datos)

```
Productor → Topic: pedidos
├─ Consumidor 1 (Analytics): Todos los pedidos
├─ Consumidor 2 (Detección de fraude): Todos los pedidos
└─ Consumidor 3 (Almacén): Todos los pedidos

Cada consumidor procesa independientemente.
Usa diferentes grupos de consumidores.

Ventajas:
✓ Los productores no conocen a los consumidores
✓ Los consumidores pueden procesar independientemente
✓ Se puede agregar un nuevo consumidor en cualquier momento (leer desde el principio)
```

### Patrón 2: Event Sourcing

Almacenar todos los eventos (solo append)
topic = "customer_events"

Evento 1: Cliente creado

```
{'event_type': 'created', 'customer_id': 1, 'name': 'Alice', 'timestamp': '2024-01-01'}
```

Evento 2: Email cambiado

```
{'event_type': 'email_changed', 'customer_id': 1, 'email': 'alice@example.com', 'timestamp': '2024-01-15'}
```

Evento 3: Suscripción actualizada

```
{'event_type': 'subscription_upgraded', 'customer_id': 1, 'plan': 'premium', 'timestamp': '2024-01-20'}
```

Estado actual: Reproducir todos los eventos

```python
def get_customer_state(customer_id):
    consumer = KafkaConsumer(bootstrap_servers=['localhost:9092'])
    consumer.subscribe(['customer_events'])

    state = {}
    for message in consumer:
        event = message.value
        if event['customer_id'] == customer_id:
            apply_event_to_state(state, event)

    return state

state = get_customer_state(1)
# {'customer_id': 1, 'name': 'Alice', 'email': 'alice@example.com', 'plan': 'premium'}
```

Ventajas:
✓ Rastro de auditoría completo (todos los cambios registrados)
✓ Se puede reproducir el historial (depuración, recuperación)
✓ Fuente única de verdad (eventos)

---

### Patrón 3: CQRS (Command Query Responsibility Segregation)

Comando: Escribir en Kafka (eventos)
Consulta: Leer desde vistas materializadas (precomputadas)

Flujo de escritura:
Comando de usuario → Evento → Kafka → Base de datos actualizada

Flujo de lectura:
Consulta de usuario → Vista materializada (precomputada) → Respuesta rápida

Ejemplo:
Comando: "Agregar item al carrito"

```
└─ Evento: {'event': 'item_added_to_cart', 'user_id': 123, 'item_id': 456}
└─ Escrito en topic "cart_events"
└─ Consumidor actualiza tabla materializada "user_carts"
```

Consulta: "¿Qué hay en el carrito del usuario 123?"

```
└─ Select * from user_carts where user_id = 123
└─ Respuesta: [item_456, item_789] (instantáneo, sin computación)
```

Ventajas:
✓ Escrituras optimizadas (solo append a Kafka)
✓ Lecturas optimizadas (vistas precomputadas)
✓ Escalabilidad independiente (escritura ≠ lectura)

---

## Caso Real: Procesamiento de Transacciones de Pago

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# ===== PRODUCTOR: Servicio de Pago =====
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

def process_payment(order_id, amount, card):
    """Procesar pago, emitir evento"""
    try:
        # Procesar pago (llamar a pasarela de pago)
        transaction_id = call_payment_gateway(card, amount)

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

# ===== CONSUMIDOR 1: Fulfillment de Pedidos =====
consumer1 = KafkaConsumer(
    'payments',
    bootstrap_servers=['localhost:9092'],
    group_id='fulfillment-group'
)

for message in consumer1:
    event = message.value
    if event['event'] == 'payment_success':
        order_id = event['order_id']

        # Enviar pedido
        print(f"Enviando pedido {order_id}")
        update_order_status(order_id, 'shipped')

# ===== CONSUMIDOR 2: Detección de Fraude =====
consumer2 = KafkaConsumer(
    'payments',
    bootstrap_servers=['localhost:9092'],
    group_id='fraud-detection-group'
)

for message in consumer2:
    event = message.value
    if event['event'] == 'payment_success':

        # Verificar si es sospechoso
        if is_suspicious(event):
            print(f"ALERTA: Transacción sospechosa {event['transaction_id']}")
            flag_for_review(event)

# ===== CONSUMIDOR 3: Analíticas =====
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
        print(f"Ingresos diarios hasta ahora: ${daily_revenue}")
        update_metrics(daily_revenue)
```

Todos los consumidores independientes:

- Si el fulfillment falla: Los otros consumidores aún procesan
- Si la detección de fraude está caída: Los pagos aún se procesan
- Desacoplamiento = resiliencia

---

## Semántica Exactly-Once

Desafío: Garantizar que cada pago se procese EXACTAMENTE una vez
(no 0 veces = pago perdido, no 2+ veces = cargo duplicado)

Kafka proporciona: At-least-once (los mensajes podrían duplicarse)

Solución 1: Consumidor idempotente
├─ Procesar el mismo evento múltiples veces
├─ El resultado siempre es el mismo (idempotente)
└─ La base de datos deduplica (restricción única)

Solución 2: Productor idempotente
├─ Kafka rastrea ID del productor + número de secuencia
├─ Los envíos duplicados se detectan y se deduplican
└─ Exactly-once en la escritura

Solución 3: Procesamiento transaccional
├─ Leer de Kafka (offset X)
├─ Procesar mensaje
├─ Escribir resultado + confirmar offset atómicamente
└─ Si falla: Ambas operaciones se revierten (sin duplicados)

Implementación:

```python
consumer.enable_auto_commit = False
for message in consumer:
    result = process(message.value)  # Idempotente
    save_to_db(result, message.offset)  # Con offset
    consumer.commit(message.offset)  # Confirmar solo después de guardar
```

Si falla entre guardar y confirmar:

- Reiniciar desde el último offset confirmado
- Reprocesar el mismo mensaje
- Idempotencia + restricción única = sin duplicados

---

## Errores Comunes en Entrevista

- **Error**: "Kafka almacena todos los eventos para siempre" → **Solución**: Retención predeterminada de 7 días (configurable)

- **Error**: "Los consumidores compiten por mensajes" → **Solución**: Depende de los grupos de consumidores (mismo grupo = dividen; diferentes grupos = todos reciben)

- **Error**: Ignorar la gestión de offsets → **Solución**: Offset = "dónde estamos", crítico para la recuperación

- **Error**: "Kafka es lento" → **Solución**: Kafka = 1M eventos/seg fácil. La mayor latencia proviene de tu código

---

## Preguntas de Seguimiento Típicas

1. **"¿Diferencia entre Kafka y RabbitMQ?"**
   - Kafka: Distribuido, duradero, escala masiva
   - RabbitMQ: Simple, pero no distribuido

2. **"¿Particiones vs Topics?"**
   - Topic: Canal lógico
   - Partición: División física (paralelismo)

3. **"¿Rebalanceo de grupos de consumidores?"**
   - Consumidores se unen/dejan del grupo
   - Pausa breve durante el rebalanceo
   - Puede causar un pico de latencia

4. **"¿Rendimiento: ¿cuántas particiones?"**
   - Más particiones = más paralelismo
   - Pero más sobrecarga
   - Ajustar según necesidades de throughput

---

## Referencias

- [Apache Kafka - Documentación Oficial](https://kafka.apache.org/documentation/)
- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
