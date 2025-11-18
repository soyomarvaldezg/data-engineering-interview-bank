# Diseñar Sistema en Tiempo Real: Kafka + Streaming

**Tags**: #system-design #streaming #kafka #real-time #architecture #real-interview

---

## TL;DR

Sistema en tiempo real = Kafka (mensajes duraderos) + Streaming (procesamiento). Arquitectura: Productores → Kafka → Consumidores (Spark/Flink) → Almacenamiento (Data Lake). Decisiones clave: Particionamiento (clave de partición, número), Configuración de Kafka (réplicas, retención), Estrategia de Consumo (grupos, reintentos), Monitoreo (métricas, alertas). Trade-off: Latencia vs Complejidad vs Costo.

---

## Concepto Core

- **Qué es**: Sistema que procesa eventos a medida que llegan (tiempo real)
- **Por qué importa**: Fundamental para detección de fraude, dashboards en tiempo real, recomendaciones en tiempo real
- **Principio clave**: Kafka = log inmutable. Los consumidores leen pero no eliminan mensajes

---

## Problema Real

**Escenario**:

- Plataforma de comercio electrónico con 100,000 transacciones/minuto
- Necesita detección de fraude en tiempo real (< 500ms)
- Dashboard de métricas en tiempo real
- Equipo de analítica (10 personas) + 2 ingenieros de datos
- Presupuesto: $8,000/mes en infraestructura en la nube

**Preguntas**:

1. ¿Cómo diseñas el sistema?
2. ¿Qué particiones y réplicas para Kafka?
3. ¿Cómo manejas el estado del usuario?
4. ¿Cómo garantizas exactamente una vez?
5. ¿Cómo monitoreas el rendimiento?

---

## Solución Propuesta

### Arquitectura General

```
┌─────────────────────────────────────────────────────┐
│                  PRODUCTORES                    │
├─────────────────────────────────────────────────────┤
│  Web App │ Mobile App │ Payment Gateway │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│                  KAFKA CLUSTER                        │
├─────────────────────────────────────────────────────┤
│ Topic: "transactions" (10 particiones, 3 réplicas)       │
│ Topic: "events" (5 particiones, 3 réplicas)           │
│ Topic: "user_actions" (20 particiones, 3 réplicas)         │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│                  CONSUMIDORES                       │
├─────────────────────────────────────────────────────┤
│  Spark Streaming (3 ejecutores)                │
│  Flink Streaming (2 ejecutores)                │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│                  STORAGE                           │
├─────────────────────────────────────────────────────┤
│  Data Lake (S3)                               │
│  State Stores (Redis, RocksDB)                   │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│                  SERVICIOS                         │
├─────────────────────────────────────────────────────┤
│  API Gateway (decisiones de fraude)              │
│  Dashboard (métricas en tiempo real)             │
└─────────────────────────────────────────────────────┘
```

### Pila Tecnológica

| Componente         | Tecnología           | Razón                           | Costo       |
| ------------------ | -------------------- | ------------------------------- | ----------- |
| **Productores**    | Java/Kafka Producer  | Alto rendimiento, idioma nativo | $1k/mes     |
| **Broker**         | Apache Kafka         | Durabilidad, escalabilidad      | $2k/mes     |
| **Consumidores**   | Spark Streaming      | Procesamiento distribuido       | $3k/mes     |
| **Almacenamiento** | S3 + Redis           | Durabilidad, escalable          | $1.5k/mes   |
| **Estado**         | Redis                | Acceso rápido a estado          | $0.5k/mes   |
| **Monitoreo**      | Prometheus + Grafana | Métricas en tiempo real         | $0.5k/mes   |
| **Total**          |                      |                                 | **$8k/mes** |

---

### Flujo de Datos

```
Productor → Kafka (Topic: transactions)
└─ Transacción completada con transaction_id, timestamp, user_id, amount
└─ Incluye metadatos: device_id, IP, user_agent

Kafka (Topic: transactions)
└─ Particionado por transaction_id (hash)
└─ 3 réplicas para durabilidad
└─ Retención: 7 días (para análisis)

Consumidor Spark
└─ Lee de Kafka (desde el último offset)
└─ Parsea JSON a objetos
└─ Enriquece con datos del usuario (desde Redis)
└─ Aplica modelo de fraude
└─ Escribe decisiones a Kafka (topics: approved/blocked)
```

---

### Configuración de Kafka

#### Particionamiento

```bash
# Particionamiento por clave de negocio
# transactions: particionado por transaction_id (hash)
# user_actions: particionado por user_id (hash)
# events: particionado por event_type

# Número de particiones = (throughput_target / avg_message_size) / partitions_per_broker
# Para 100,000 msg/min con 1KB/msg: 10 particiones
```

#### Réplicas y Factor de Replicación

```bash
# Factor de replicación = min(3, # brokers)
# Para 3 brokers, 3 réplicas = 9 copias de cada partición
# Más réplicas = mayor durabilidad pero mayor latencia de escritura
```

#### Retención

```bash
# Configuración de retención
# transactions: 7 días (para análisis de fraude)
# events: 3 días (para análisis de comportamiento)
# user_actions: 30 días (para análisis de usuario)
```

---

### Manejo de Estado

#### Estado del Usuario

```python
# Estado en Redis (clave: user_id)
{
    "user_12345": {
        "name": "John Doe",
        "email": "john@example.com",
        "registration_date": "2023-01-15",
        "last_login": "2024-01-20",
        "total_spent": 1250.50,
        "transaction_count": 25,
        "device_type": "mobile"
    }
}
```

#### Código del Consumidor

```python
from pyspark.sql.functions import col, from_json
from pyspark.sql.types import StructType, StructField, StringType

# Esquema para transacciones
transaction_schema = StructType([
    StructField("transaction_id", StringType()),
    StructField("timestamp", StringType()),
    {"name": "user_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "device_type", "type": "string"}
])

# Estado del usuario en Redis
def get_user_profile(user_id):
    redis_client = redis.StrictRedis(host='redis-cluster', port=6379, db=0)
    profile_data = redis_client.get(f"user:{user_id}")
    return json.loads(profile_data) if profile_data else None

# Enriquecer con estado del usuario
enriched_df = parsed_df.withColumn(
    "user_profile",
    get_user_profile(col("user_id"))
)
```

---

### Detección de Fraude

#### Modelo de Puntuación

```python
import joblib
import numpy as np

class FraudDetectionModel:
    def __init__(self, model_path):
        self.model = joblib.load(model_path)

    def predict(self, features):
        proba = self.model.predict_proba(features)
        return proba[0][0]  # Probabilidad de fraude

    def save_model(self, model_path):
        joblib.dump(self.model, model_path)
```

#### Aplicación en Streaming

```python
from pyspark.sql.functions import col, when

# Aplicar modelo de fraude
def apply_fraud_model(df):
    model = FraudDetectionModel("models/fraud_detection.pkl")

    features = df.select(
        "transaction_id",
        "amount",
        "device_type",
        "hour_of_day",
        "days_since_registration"
    )

    predictions = model.predict(features)

    return df.withColumn(
        "fraud_score",
        predictions[0]
    ).withColumn(
        "decision",
        when(col("fraud_score") > 0.8, "BLOCKED")
        .otherwise("APPROVED")
    )
```

---

### Garantías de Entrega

#### Entrega Exactamente Una Vez

```python
# Productor con idempotencia
def send_transaction(transaction):
    transaction_id = generate_unique_id()

    # Verificar si ya fue enviado
    if not transaction_exists(transaction_id):
        producer.send('transactions', value=transaction)
        record_transaction(transaction_id)

    # Retornar ID para seguimiento
    return transaction_id
```

#### Consumidor con Checkpointing

```python
# Checkpoint cada 100 transacciones
checkpoint_interval = 100

def process_stream():
    df = spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "kafka:9092")
        .option("subscribe", "transactions")
        .load()

    parsed_df = parse_transactions(df)

    enriched_df = enriched_df(parsed_df)

    fraud_df = apply_fraud_model(enriched_df)

    # Checkpoint cada 100 transacciones
    query = fraud_df.writeStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "kafka:9092")
        .option("topic", "fraud_decisions")
        .option("checkpointLocation", "/tmp/checkpoint")
        .outputMode("append")
        .trigger(processingTime="1 minute")
        .start()

    query.awaitTermination()
```

---

### Monitoreo

#### Métricas Clave

```python
# Métricas de Kafka
kafka_producer_metrics = {
    "record_send_rate": producer.metrics(),
    "record_error_rate": producer.metrics()['record-error-rate'],
    "request_latency_avg_ms": producer.metrics()['request-latency-avg']
}

# Métricas de Consumidor
consumer_metrics = {
    "records_consumed_rate": consumer.metrics()['records-consumed-rate'],
    "records_lag_max": consumer.metrics()["records-lag-max"],
    "consumer_lag_avg": consumer.metrics()['consumer-lag-avg']
}

# Métricas de Procesamiento
processing_metrics = {
    "processing_time_avg_ms": processing_time_avg,
    "fraud_detection_rate": fraud_count / total_count,
    "decision_distribution": {
        "APPROVED": approved_count,
        "BLOCKED": blocked_count
    }
}
```

#### Alertas

```python
def alert_on_metrics(metrics):
    if metrics["consumer_lag_max"] > 10000:
        alert("ALERT: Consumer lag too high", f"Max lag: {metrics['consumer_lag_max']}")

    if metrics["processing_time_avg_ms"] > 500:
        alert("ALERT: Processing too slow", f"Avg processing time: {metrics['processing_time_avg_ms']} ms")

    if metrics["decision_distribution"]["BLOCKED"] / total_count > 0.1:
        alert("ALERT: High block rate", f"Block rate: {metrics['decision_distribution']['BLOCKED'] / total_count}")
```

---

## Escalabilidad: De 100k a 1M Transacciones/Minuto

### Cambios Necesarios

#### Kafka

- Particiones: 10 → 20 (para mayor paralelismo)
- Réplicas: 3 → 5 (para mayor durabilidad)
- Brokers: 3 → 6 (para mayor rendimiento)

#### Consumidores

- Ejecutores de Spark: 3 → 6 (para mayor rendimiento)
- Memoria por ejecutor: 8GB → 16GB (para manejar estado)

#### Estado

- Redis: 3 nodos → 6 nodos (para mayor disponibilidad)
- Particionamiento de estado por clave (hash de user_id)

#### Almacenamiento

- S3: Particionamiento más granular (por hora además de fecha)
- Lifecycle: Hot (7 días) → Warm (30 días) → Cold (S3 Glacier)

---

## Compromisos Analizados

| Decisión             | Opción A             | Opción B            | Elección             | Razón                  |
| -------------------- | -------------------- | ------------------- | -------------------- | ---------------------- |
| **Particionamiento** | Por clave de negocio | Por tiempo          | Por clave de negocio | Mejor para rendimiento |
| **Entrega**          | Al menos una vez     | Exactamente una vez | Exactamente una vez  | Garantiza consistencia |
| **Estado**           | En memoria           | En base de datos    | En memoria           | Rápido acceso          |
| **Costo**            | Bajo                 | Alto                | Bajo                 | Equilibrio             |

---

## Alternativas Consideradas

### Alternativa 1: Puro SQL (Kafka + Materialized Views)

**Pros**:

- Simplicidad del desarrollo
- Sin procesamiento de streaming
- Menor costo de infraestructura

**Contras**:

- Latencia más alta (consultas SQL)
- Menos flexibilidad para lógica compleja

**Decisión**: No adoptado. Los requisitos de baja latencia (<500ms) y lógica compleja de fraude justifican el procesamiento de streaming.

### Alternativa 2: AWS Kinesis

**Pros**:

- Totalmente gestionado
- Integración con AWS

**Contras**:

- Limitaciones de tamaño de mensaje
- Menos flexibilidad

**Decisión**: No adoptado. Kafka ofrece más flexibilidad y control.

### Alternativa 3: Apache Pulsar

**Pros**:

- Entrega garantizada
- Baja latencia

**Contras**:

- Menos ecosistema maduro
- Menos casos de uso

**Decisión**: No adoptado. Kafka es más maduro y ampliamente utilizado.

---

## Errores Comunes en Entrevista

- **Error**: "Kafka es solo para logs" → **Solución**: Kafka es para eventos de negocio, no solo para logs

- **Error**: "Los consumidores compiten por mensajes" → **Solución**: Los consumidores en el mismo grupo no compiten; diferentes grupos consumen todas las particiones

- **Error**: "No considerar el estado del consumidor" → **Solución**: El estado debe ser persistente y recuperable

- **Error**: "Ignorar la gestión de offsets" → **Solución**: Los offsets son críticos para la recuperación tras fallos

---

## Preguntas de Seguimiento Típicas

1. **"¿Cómo manejas el desorden de los eventos?"**
   - Particionar por clave de negocio para mantener el orden
   - Usar marcas de tiempo para manejar eventos tardíos

2. **"¿Cómo garantizas exactamente una vez?"**
   - Productor con idempotencia
   - Consumidor con checkpoints
   - Claves únicas en la tabla de hechos

3. **"¿Cómo manejas el estado del consumidor si falla?"**
   - Recuperar desde el último checkpoint
   - El estado debe ser persistente

4. **"¿Cómo escalas a 10M transacciones/minuto?"**
   - Más particiones y réplicas
   - Más consumidores
   - Estado particionado

---

## Referencias

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Stream Processing with Apache Flink](https://flink.apache.org/)
