# Procesamiento de Streams: Windowing y Gestión de Estado

**Tags**: #streaming #windowing #state #aggregation #spark-streaming #real-interview

---

## TL;DR

**Stream processing** = procesar flujos infinitos de eventos en tiempo real. **Windowing** = agrupar eventos por tiempo (ventana de 5 minutos, 1 hora). **Aggregation** = contar, sumar dentro de la ventana. **State** = mantener contexto (sesión de usuario, total del pedido). **Desafío**: Manejar eventos tardíos, datos fuera de orden, límites de ventana.

---

## Concepto Core

- **Qué es**: Windowing = "dividir el tiempo en trozos", agregar dentro de cada trozo
- **Por qué importa**: Las analíticas en tiempo real necesitan ventanas (ventas de 5 minutos, errores por hora)
- **Principio clave**: El tiempo es complicado en streaming (tiempo de evento vs tiempo de procesamiento)

---

## Tipos de Ventanas

Tipo 1: TUMBLING (Fija, no superpuesta)
├─ Tamaño de ventana: 5 minutos
├─ Uso: Ventas por hora, conteos diarios de usuarios

Tipo 2: SLIDING (Fija, superpuesta)
├─ Tamaño de ventana: 5 minutos
├─ Deslizamiento: 1 minuto (cada minuto, nueva ventana)
├─ Uso: Promedio móvil, métricas en tiempo real

Tipo 3: SESSION (Dinámica, basada en usuario)
├─ Brecha de inactividad: 10 minutos
├─ [Usuario A clic 00:00] → Iniciar sesión
├─ [Usuario A clic 00:02] → Extender sesión
├─ [Usuario A sin actividad > 10 min] → Finalizar sesión
├─ [Usuario A clic 00:15] → NUEVA sesión
└─ Longitud de sesión variable

Tipo 4: DELAY (Después del plazo, actualización)
├─ Ventana se cierra en tiempo T
├─ Pero permite eventos tardíos hasta tiempo T + 10 min
├─ Actualiza resultados a medida que llegan datos tardíos
└─ Resultado final después del período de gracia

---

## Ejemplos de Windowing en SQL

```sql
-- Ventana de Tumbling: agregación de ventas de 5 minutos
SELECT
  TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
  TUMBLE_END(event_time, INTERVAL '5' MINUTE) as window_end,
  product_id,
  COUNT(*) as num_sales,
  SUM(amount) as revenue
FROM orders
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE), product_id;

-- Resultado:
-- window_start window_end product_id num_sales revenue
-- 2024-01-15 10:00:00 2024-01-15 10:05:00 1 100 1000
-- 2024-01-15 10:05:00 2024-01-15 10:10:00 1 110 1100
```

```sql
-- Ventana de Sliding: promedio móvil de 1 hora (calculado cada 10 min)
SELECT
  HOP_START(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR) as window_start,
  HOP_END(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR) as window_end,
  AVG(latency_ms) as avg_latency
FROM requests
GROUP BY HOP(event_time, INTERVAL '10' MINUTE, INTERVAL '1' HOUR);

-- Resultado: Actualizado cada 10 min con promedio de 1 hora
```

```sql
-- Ventana de Sesión: comportamiento del usuario
SELECT
  SESSION_START(event_time, INTERVAL '10' MINUTE) as session_start,
  SESSION_END(event_time, INTERVAL '10' MINUTE) as session_end,
  user_id,
  COUNT(*) as num_events,
  ARRAY_AGG(event_type) as event_sequence
FROM user_events
GROUP BY SESSION(event_time, INTERVAL '10' MINUTE), user_id;

-- Resultado:
-- session_start session_end user_id num_events event_sequence
-- 2024-01-15 10:00:00 2024-01-15 10:12:00 1 5 ['click','add_cart','checkout','pay','confirm']
-- 2024-01-15 10:30:00 2024-01-15 10:35:00 1 2 ['login','view_profile']
```

---

## Semántica del Tiempo

Concepto clave: 3 tiempos diferentes en streaming

TIEMPO DE EVENTO
└─ Cuándo ocurrió el evento
└─ Ejemplo: Procesamiento del pago a las 10:05:32

TIEMPO DE PROCESAMIENTO
└─ Cuándo el evento llega a Kafka
└─ Ejemplo: Llega a las 10:05:35 (3 segundos después)

WATERMARK
└─ "Hemos visto todos los eventos hasta el tiempo X"
└─ Ejemplo: Marca de agua a las 10:05:00 = no hay más eventos antes de 10:05:00
└─ Señales: Ventana completa, tiempo de emitir resultados

Cronología:
Evento ocurre: 10:05:32 (tiempo de evento)
Evento enviado: 10:05:33
Evento llega a Kafka: 10:05:35 (tiempo de procesamiento)
Latencia: 3 segundos

La ventana debe usar TIEMPO DE EVENTO (no tiempo de procesamiento)
De lo contrario, los resultados dependen de los retrasos de la red, no de los eventos reales

---

## Gestión de Estado

Desafío: Algunas operaciones necesitan CONTEXTO
Simple (sin estado): Contar eventos
Resultado: 5 eventos en la ventana
Con estado: Sesión de usuario
Necesidad: ¿Qué eventos pertenecen a la misma sesión?

```python
from pyspark.sql.functions import col, window, session_window
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("session_tracking").getOrCreate()

# Stream: user_events
df = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "user_events")
  .load()

# Parsear eventos
events = df.select(
  col("value").cast("string"),
  col("timestamp").cast("timestamp").alias("event_time")
)

# Ventana de sesión: 10 min de brecha de inactividad
sessions = events
  .groupBy(
    session_window(col("event_time"), "10 minutes"),
    col("user_id")
  )
  .agg(
    count("*").alias("num_events"),
    collect_list("event_type").alias("event_sequence")
  )

# Resultado: Por cada usuario, por sesión:
# ├─ Inicio/fin de sesión
# ├─ Número de eventos
# └─ Secuencia de eventos
```

Implicaciones de estado:
├─ Debe mantener el estado del usuario a través de ventanas
├─ Memoria: O(# de sesiones activas)
└─ Tiempo de espera: Limpiar después de 10+ minutos de inactividad

---

## Manejo de Datos Tardíos

```python
from pyspark.sql.functions import col, window

Escenario: Evento retrasado en tránsito
Evento 1: Pedido realizado a las 10:05:32, llega a las 10:05:35
Evento 2: Pedido realizado a las 10:06:15, llega a las 10:06:18
Evento 3: Pedido realizado a las 10:05:58, llega a las 10:07:00 (¡TARDE!)

Ventana:
├─ Eventos esperados: Evento 1, Evento 2
├─ Evento 3 llega DESPUÉS de que la ventana se cerró
├─ Decisión: ¿Ignorar o actualizar?
Enfoque de Spark: Permitir actualización si está dentro del período de gracia
query = df
  .groupBy(
    window(col("event_time"), "5 minutes", "1 minute"), # Ventana deslizante
    col("product_id")
  )
  .agg(count("*").alias("num_orders"))
  .writeStream
  .outputMode("update") # Emitir actualizaciones (no solo añadir)
  .option("watermarkDelayThreshold", "10 minutes") # Período de gracia
  .start()

Cronología:
10:05:35 - Evento 1 llega →
10:06:18 - Evento 2 llega →
10:07:00 - Evento 3 (TARDE) llega →
10:15:00 - Marca de agua pasa 10:15 →
Modos de salida:
├─ append: Solo nuevas filas (ventanas asentadas)
├─ update: Filas modificadas (datos tardíos)
└─ complete: Todas las filas (recomputado)
```

---

## Caso Real: Dashboard en Tiempo Real

```sql
-- Stream: user_events (clics, compras, etc.)
-- Tarea: Dashboard de 5 minutos (eventos, ingresos, productos principales)

CREATE TABLE dashboard_5min AS
SELECT
  TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_time,
  COUNT(*) as total_events,
  COUNT(CASE WHEN event_type = 'purchase' THEN 1 END) as num_purchases,
  SUM(CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END) as revenue,
  AVG(session_duration_sec) as avg_session_duration,
  ARRAY_AGG(DISTINCT product_id ORDER BY COUNT(*) DESC LIMIT 5) as top_5_products
FROM user_events
WHERE event_time > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE);

-- Resultado se actualiza cada 5 minutos:
-- window_time total_events num_purchases revenue avg_session_duration top_5_products
-- 2024-01-15 10:00:00 50000 5000 50000 420
-- 2024-01-15 10:05:00 52000 5200 52000 425
-- 2024-01-15 10:10:00 51000 5100 51000 422
```

---

## Errores Comunes en Entrevista

- **Error**: Windowing por TIEMPO DE PROCESAMIENTO en lugar de TIEMPO DE EVENTO → **Solución**: Siempre usar tiempo de evento (los retrasos de red sesgan los resultados)

- **Error**: Ignorar datos tardíos → **Solución**: Los sistemas reales tienen eventos retrasados, deben manejarse con gracia

- **Error**: No limpiar estado (fuga de memoria) → **Solución**: Establecer tiempo de espera = TTL para el estado de la sesión

- **Error**: Ventanas TUMBLING superpuestas → **Solución**: Tumbling = no superpuestas por definición. Si necesitas superposición, usa Sliding

---

## Preguntas de Seguimiento Típicas

1. **"¿Diferencia entre Tumbling y Sliding?"**
   - Tumbling: No superpuestas (eficiente)
   - Sliding: Superpuestas (más suave, más computación)

2. **"¿Cómo manejas eventos fuera de orden?"**
   - Windowing por tiempo de evento (no tiempo de procesamiento)
   - Período de gracia permite llegadas tardías
   - Marca de agua señala el cierre de la ventana

3. **"¿Implementación de ventana de sesión?"**
   - Rastrear: último tiempo de evento por usuario
   - Nuevo evento: Si hay brecha > umbral de inactividad = nueva sesión
   - De lo contrario: Extender sesión existente

4. **"¿Explosión de estado?"**
   - Monitorear: Número de sesiones activas
   - Limpieza: Eliminar sesiones inactivas > tiempo de espera
   - Escalabilidad: Mover a tienda de estado distribuida (Redis, RocksDB)

---

## Referencias

- [Watermarks & Grace Period - Beam](https://beam.apache.org/documentation/programming-guide/#watermarks-and-late-data)
