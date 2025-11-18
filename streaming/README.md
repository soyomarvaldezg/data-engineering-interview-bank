# Streaming & Real-Time Data Processing

## ¿Por qué Streaming?

Batch = "process all data at once, daily".  
Streaming = "process data as it arrives, seconds latency".  
En sistemas modernos: orders, clicks, payments → se necesitan real-time insights (fraud detection, dashboards, alerts).

## Temas cubiertos

1. **Kafka Architecture & Patterns** — topics, partitions, consumer groups, offsets
2. **Stream Processing** — windowing, state management, joins
3. **Real-Time vs Batch Trade-offs** — Lambda vs Kappa architectures
4. **Streaming Frameworks** — Kafka Streams, Spark Streaming basics

## Recomendación de estudio

**Para mid devs:**

- Entender Kafka basics (topics, producers, consumers)
- Saber cuándo usar batch vs streaming

**For seniors:**

- Domina consumer groups + offset management
- Optimiza windowing strategies
- Diseña fault-tolerant streaming pipelines
