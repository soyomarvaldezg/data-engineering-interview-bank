# Tiempo Real vs Batch: Arquitecturas Lambda y Kappa

**Tags**: #architecture #lambda #kappa #real-time #batch #trade-offs #real-interview

---

## TL;DR

**Lambda** = batch + capa de velocidad (pipelines duales). **Kappa** = solo streaming (pipeline único, capaz de replay). **Compromiso**: Lambda = corrección garantizada pero complejo; Kappa = más simple pero requiere streaming perfecto. **Realidad**: La mayoría de las empresas usan Kappa ahora (Kafka replay lo hace funcionar).

---

## Concepto Core

- **Qué es**: Dos arquitecturas competidoras para analítica en tiempo real
- **Por qué importa**: Afecta la infraestructura, complejidad, corrección
- **Principio clave**: Streaming ≠ perfecto. Necesita estrategia para correcciones

---

## Arquitectura Lambda

```
Fuente de Datos
│
├─────────────────────────┬──────────────────────────┐
│ │
▼ ▼ ▼
CAPA BATCH LENTA CAPA DE VELOCIDAD RÁPIDA
(Lento, Correcto) (Rápido, Aproximado) (Resultados fusionados)
```

Batch:
├─ Datos históricos
├─ Se ejecuta nocturnamente
├─ Spark/EMR (recalcula todo)
├─ Tarda 2-4 horas
├─ Absolutamente correcto
└─ Resultados → Base de datos

Velocidad:
├─ Eventos en tiempo real
├─ Consumidores de Kafka
├─ Spark Streaming (incremental)
├─ 10-30 segundos de latencia
├─ Aproximado (puede tener errores)
└─ Resultados → Caché/Base de datos

Servicio:
├─ Consulta de usuario
│
├─ Verificar capa de velocidad (últimos ~2 horas)
├─ Verificar capa batch (completo)
├─ Fusionar resultados
├─ Si hay discrepancia: confiar en batch
└─ Devolver al usuario

Arquitectura:
Consulta de usuario
│
├─ Verificar capa de velocidad (últimos ~2 horas)
├─ Verificar capa batch (completo)
├─ Fusionar resultados
├─ Si hay discrepancia: confiar en batch
└─ Devolver al usuario

Cronología:
2AM - La capa batch comienza (recalcula todo)
6AM - La capa batch termina (resultados correctos)
6-24h - La capa de velocidad se ejecuta (resultados aproximados)
Siguiente 2AM - La capa de velocidad se descarta, reemplazada por batch

```

### Pros y Contras de Lambda

**Pros:**
✓ Corrección garantizada (capa batch = fuente de verdad)
✓ Fácil corregir errores (volver a ejecutar batch)
✓ Lógica simple (herramientas batch conocidas)

**Contras:**
✗ Mantenimiento dual (código batch + código streaming)
✗ Complejidad (fusionar dos sistemas)
✗ Inconsistencia de datos (velocidad ≠ batch por 12-22 horas)
✗ Costo más alto (calcular + almacenar dos veces)

---

## Arquitectura Kappa

```

Fuente de Datos
│
▼
PIPELINE DE STREAMING (único sistema, siempre correcto)
├─ Kafka (log inmutable)
├─ Procesador de stream (Spark, Flink)
├─ Tienda de estado (Redis, RocksDB)
└─ Resultados → Base de datos

```

Visión clave: Kafka es replay
├─ Contiene TODOS los eventos (inmutable)
├─ Nuevo error descubierto
├─ Rebobinar Kafka desde el principio
├─ Todos los resultados regenerados (correctos)
└─ Sin "ventana de inconsistencia"

Arquitectura:
Tema de Kafka "eventos"
├─ Contiene TODOS los eventos (inmutable)
├─ Consumidor: offset actual (ej: evento 5M)
├─ Error descubierto
├─ Código corregido
├─ Reiniciar consumidor: offset = 0
├─ Reprocesar todos los eventos
└─ Base de datos actualizada (correcta)

Cronología:
2AM - Error descubierto
2:05AM - Código corregido
2:10AM - Reiniciar consumidor (offset = 0)
3AM - Todos los eventos reprocesados
3:05AM - Base de datos correcta (sin ventana de 22 horas)

```

### Pros y Contras de Kappa

**Pros:**
✓ Sistema único (más simple)
✓ Siempre correcto (o fácilmente corregible)
✓ Sin ventana de inconsistencia
✓ Recuperación más rápida (reprocesar desde Kafka)

**Contras:**
✗ Requiere Kafka (log inmutable)
✗ Replay puede ser lento (reprocesar todo el historial)
✗ Gestión de estado compleja (para sesiones, agregaciones)
✗ No adecuado para lógica batch compleja (ML entrenamiento, etc.)

---

## Comparación

| Aspecto           | Lambda                                | Kappa                           |
| ----------------- | ------------------------------------- | ------------------------------- |
| **Complejidad**   | Alta (2 sistemas)                     | Baja (1 sistema)                |
| **Corrección**    | Eventual (después de batch)           | Garantizada (replay)            |
| **Latencia**      | Mixta (velocidad rápida, batch lento) | Consistente (siempre streaming) |
| **Recuperación**  | Volver a ejecutar batch               | Reprocesar desde Kafka          |
| **Mantenimiento** | Dual (código batch + streaming)       | Único (solo streaming)          |
| **Costo**         | Alto (calcular + almacenar dos veces) | Medio (solo streaming)          |
| **Adecuado para** | Analítica compleja                    | Sistemas en tiempo real         |

---

## Caso Real: Pedidos de E-commerce

### Enfoque Lambda

Capa Batch (nocturna):
├─ Leer: pedidos crudos (S3, todo el historial)
├─ Calcular: ingresos, productos principales, segmentos de clientes
├─ Escribir: tablas agregadas (Redshift)
└─ Latencia: 2-4 horas

Capa de Velocidad (tiempo real):
├─ Leer: Kafka pedidos (últimos 30 minutos)
├─ Calcular: agregados incrementales
├─ Escribir: caché (Redis, expira en 30 minutos)
└─ Latencia: 30 segundos

Servicio:
Consulta "ingresos hoy"
├─ Verificar caché (últimos ~30 minutos, ~2 horas)
├─ Verificar Redshift (completo, hasta ayer)
├─ Fusionar resultados
├─ Si hay discrepancia: confiar en Redshift
└─ Devolver: $150k + nota "últimas 2 horas aproximadas"

Problema:
├─ Usuario ve $150k a las 10 AM
├─ A las 2 AM, batch termina: $145k
├─ Usuario ve $145k a las 10:05 AM (¡bajó!)
├─ Confusión: "¿dónde fueron mis $5k?"
└─ Explicación: "Ajuste de batch, los datos de ayer se corrigieron"

### Enfoque Kappa

Pipeline de Streaming (siempre):
├─ Leer: Kafka pedidos (todos los eventos)
├─ Calcular: agregados incrementales
├─ Escribir: base de datos (actualizada continuamente)
└─ Latencia: 30 segundos

Consulta "ingresos hoy":
├─ Verificar base de datos (siempre correcta)
├─ Devolver: $150k
└─ Sin notas, sin confusión

Error descubierto:
├─ Error en cálculo de ingresos (impuesto faltante)
├─ Código corregido
├─ Reiniciar consumidor Kafka (offset = 0)
├─ Reprocesar todos los eventos (1 hora)
├─ Base de datos corregida
└─ Sin ventana de inconsistencia

---

## Decisión: Lambda vs Kappa

Elegir Lambda si:
├─ Lógica batch compleja (ML, informes semanales/mensuales)
├─ La corrección puede esperar 12-24 horas
├─ El costo es una preocupación importante
├─ Ya tienes sistemas batch optimizados

Elegir Kappa si:
├─ Tiempo real es crítico (<1 minuto de latencia)
├─ La lógica de streaming es manejable
├─ La corrección inmediata es necesaria
├─ Tienes Kafka (o log inmutable equivalente)

Tendencia actual:
├─ Empresas moviéndose a Kappa
├─ Razón: Kafka maduro, frameworks de streaming mejorados
├─ Compromiso: Simplicidad > ahorro de costos (para empresas con muchos ingenieros)

---

## Híbrido: Lambda con Beneficios de Kappa

Idea: Usar estilo Kappa replay en arquitectura Lambda

Arquitectura:
├─ Capa batch: nocturna, como siempre
├─ Capa de velocidad: Kafka + capaz de replay
├─ Error encontrado: no solo reiniciar capa de velocidad
├─ Solución: rebobinar Kafka, reprocesar capa de velocidad + ingestar a batch
└─ Resultado: corrección rápida (30 minutos) sin esperar a batch nocturno

Ventajas:
✓ Seguridad de batch (resultados finales correctos)
✓ Corrección rápida de Kappa (reprocesamiento inmediato)
✓ Mejor de ambos mundos

---

## Trucos del Streaming

Truco 1: Eventos fuera de orden
├─ Evento A timestamp 10:00, llega 10:05
├─ Evento B timestamp 10:02, llega 10:01
├─ Si se procesa secuencial: orden incorrecto
├─ Solución: Windowing por tiempo de evento + watermarks

Truco 2: Estado con fugas de memoria
├─ "¿Cuántos pedidos del cliente X en las últimas 24 horas?"
├─ Necesitas estado: cliente → lista de pedidos
├─ Si el consumidor falla: estado perdido
├─ Solución: Tienda de estado externa (Redis, RocksDB)

Truco 3: Entrega exactamente una vez
├─ "Procesar cada pago exactamente una vez"
├─ Kafka puede duplicar mensajes
├─ Solución: Procesamiento idempotente + clave única en base de datos

Truco 4: Presión de datos (backpressure)
├─ Productor demasiado rápido para el consumidor
├─ Consumidor no puede mantenerse al día
├─ Solución: Buffer en Kafka, escalar consumidores

---

## Errores Comunes en Entrevista

- **Error**: "Kappa reemplaza completamente a Lambda" → **Solución**: Depende del caso de uso. Lambda todavía es mejor para analytics complejos

- **Error**: "Streaming siempre es mejor" → **Solución**: Más complejo, mayor carga operativa

- **Error**: "Kafka almacena eventos para siempre" → **Solución**: Retención predeterminada (7 días por defecto)

- **Error**: "Los consumidores compiten por mensajes" → **Solución**: Depende del grupo de consumidores (mismo grupo = dividen; diferentes grupos = todos reciben)

---

## Preguntas de Seguimiento Típicas

1. **"¿Diferencia entre Lambda y Kappa?"**
   - Lambda: Batch + velocidad
   - Kappa: Solo streaming
   - Lambda: Corrección eventual; Kappa: Corrección inmediata (con replay)

2. **"¿Kappa replay performance?"**
   - Depende de la retención de Kafka + velocidad del procesador
   - Si 1 año de retención: replay puede tomar horas
   - Trade-off: Almacenamiento vs flexibilidad

3. **"¿Manejo de estado en Kappa?"**
   - Tienda de estado externa (Redis, RocksDB)
   - Para sesiones, agregaciones complejas
   - Clave: particionar por clave de estado

4. **"¿Netflix/Uber architecture?"**
   - Principalmente Kappa ahora
   - Batch para ML entrenamiento, informes pesadas
   - Híbrido: Streaming principal con batch secundario

---

## Referencias

- [Real-time Big Data - O'Reilly](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/)
