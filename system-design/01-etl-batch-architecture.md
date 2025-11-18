# Diseñar Pipeline ETL Batch: Arquitectura a Escala

**Tags**: #system-design #etl #batch #architecture #real-interview

---

## TL;DR

ETL batch = Extraer (fuente) → Transformar (proceso) → Cargar (destino), ejecutado en horarios (diario, horario). Arquitectura: Data Lake (raw) → Procesamiento (Spark) → Data Warehouse (analytics). Decisiones clave: Spark vs SQL, Particionamiento vs Completo, Incremental vs Completo, Programación (Airflow/Cron) vs Monitoreo. Compromiso: Complejidad vs Rendimiento vs Costo.

---

## Concepto Core

- **Qué es**: Pipeline batch que procesa datos en lotes en lugar de en tiempo real
- **Por qué importa**: Fundamental para la analítica empresarial. Demuestra comprensión de arquitectura de datos a gran escala
- **Principio clave**: El diseño depende del volumen de datos, la latencia requerida y los costos de infraestructura

---

## Problema Real

**Escenario**:

- E-commerce con 1 millón de transacciones/día
- 50+ fuentes de datos (bases de datos, APIs, S3)
- Necesita informe consolidado cada mañana
- Equipo de analítica (5 personas) + 2 ingenieros de datos
- Presupuesto: $10,000/mes en infraestructura en la nube

**Preguntas**:

1. ¿Cómo diseñas el pipeline?
2. ¿Qué tecnologías usas (Spark? SQL? Serverless?)?
3. ¿Cómo manejas la frescura de los datos?
4. ¿Cómo monitoreas si algo falla?
5. ¿Cómo escalas a 10 millones de transacciones/día?

---

## Solución Propuesta

### Arquitectura (3 Capas)

```
┌─────────────────────────────────────────────────────┐
│                FUENTES DE DATOS (50+ sistemas)      │
├─────────────────────────┬───────────────────────────┤
│  Bases de datos      │ APIs      │ S3 (archivos)       │
│  (MySQL, PostgreSQL) │ (REST, SOAP) │ (CSV, JSON, logs)   │
└─────────────────────────┴───────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│              DATA LAKE (crudo) (S3)             │
├─────────────────────────────────────────────────────┤
│  s3://data-lake/raw/mysql/customers/date=2024-01-15/ │
│  s3://data-lake/raw/api/payments/date=2024-01-15/ │
│  s3://data-lake/raw/s3/logs/date=2024-01-15/   │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│            PROCESAMIENTO (Spark en EMR/Databricks)   │
├─────────────────────────────────────────────────────┤
│  Validación (manejo de nulos, detección de duplicados) │
│  Transformación (joins, agregaciones)            │
│ Enriquecimiento (lógica de negocio)                │
│  Salida a s3://data-lake/processed/             │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│            ALMACÉN DE DATOS (Redshift/BigQuery)    │
├─────────────────────────────────────────────────────┤
│  Tablas dimensionales (clientes, productos)      │
│  Tablas de hechos (transacciones, eventos)     │
│  Vistas materializadas para informes           │
└─────────────────────────────────────────────────────┘
```

### Pila Tecnológica

| Componente                   | Tecnología           | ¿Por qué?                                                      | Costo        |
| ---------------------------- | -------------------- | -------------------------------------------------------------- | ------------ |
| **Orquestación**             | Apache Airflow       | Visibilidad de DAG, reintentos, monitoreo                      | $1k/mes      |
| **Procesamiento**            | Spark en EMR         | Distribuido, escalable, compatible con Python/Scala            | $3k/mes      |
| **Almacenamiento crudo**     | Amazon S3            | Económico, duradero, escalable                                 | $1k/mes      |
| **Almacenamiento procesado** | Amazon S3            | Económico para datos procesados                                | $0.5k/mes    |
| **Almacén analítico**        | Amazon Redshift      | Rendimiento para consultas, integración con herramientas de BI | $4k/mes      |
| **Monitoreo**                | CloudWatch + DataDog | Alertas, métricas, registros                                   | $0.5k/mes    |
| **Total**                    |                      |                                                                | **$10k/mes** |

---

### Pipeline Diario: Ejecución

05:00 AM - INICIO: Programador de Airflow activa DAG

```
Tarea 1: Extracción incremental de MySQL
├─ Extraer transacciones de las últimas 24 horas
├─ WHERE updated_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
├─ Salida a s3://data-lake/raw/mysql/transactions/date=2024-01-15/

Tarea 2: Extracción de APIs
├─ Extraer pagos de API de pagos con reintentos (3x)
├─ Salida a s3://data-lake/raw/api/payments/date=2024-01-15/

Tarea 3: Descarga de archivos S3
├─ Descargar archivos de logs del día anterior
├─ Salida a s3://data-lake/raw/s3/logs/date=2024-01-15/

Tarea 4: PROCESAMIENTO (Spark)
├─ Leer datos crudos de S3
├─ Validar datos (nulos, duplicados)
├─ Transformar (joins, agregaciones)
├─ Enriquecer (lógica de negocio)
├─ Salida a s3://data-lake/processed/date=2024-01-15/

Tarea 5: CARGA (Redshift)
├─ Copiar datos procesados a tablas de Redshift
├─ Actualizar vistas materializadas
├─ Invalidar caché de consultas

Tarea 6: VALIDACIÓN DE CALIDAD DE DATOS
├─ Verificar conteos de filas (vs. día anterior ±5%)
├─ Verificar porcentajes de nulos
├─ Verificar duplicados
├─ Enviar alertas si hay problemas

06:00 AM - FIN: Pipeline completado, informes listos para análisis
```

---

### Decisiones de Diseño Clave

#### 1. Spark vs SQL

**Spark** (Elegido):

- Transformaciones complejas (UDFs, lógica personalizada)
- Procesamiento de grandes volúmenes (terabytes)
- Flexibilidad con Python/Scala

**SQL** (Limitado a):

- Transformaciones simples (donde el SQL es suficiente)
- Cargas directas cuando no se necesita lógica compleja

#### 2. Particionamiento vs. Completitud

**Particionamiento** (Elegido):

- Datos particionados por fecha para el paralelismo
- `s3://data-lake/processed/date=YYYY-MM-DD/`
- Permite que Spark procese solo datos del día actual

**Completitud** (No utilizado):

- Sería necesario si los datos no tienen una partición natural
- Más complejo de gestionar

#### 3. Incremental vs. Completo

**Incremental** (Elegido):

- Solo procesa datos de las últimas 24 horas
- Más rápido y rentable
- Requiere seguimiento de marcas de agua/high watermarks

**Completo** (No utilizado):

- Procesaría todos los datos cada día
- Más lento y costoso
- Sería necesario para correcciones de datos históricos

#### 4. Programación: Airflow vs. Cron

**Airflow** (Elegido):

- Visibilidad del flujo de trabajo (DAG)
- Reintentos automáticos en caso de fallo
- Monitoreo integrado

**Cron** (No utilizado):

- Más simple pero menos resistente a fallos
- Sin visibilidad del flujo de trabajo
- Sin reintentos automáticos

---

### Monitoreo y Alertas

```python
# Validaciones post-pipeline
class DataQualityChecks:
    def row_count_validation(self, table, yesterday_count):
        today_count = get_count(table)
        pct_change = abs((today_count - yesterday_count) / yesterday_count) * 100

        if pct_change > 10:  # +/- 10% es el umbral
            alert(f"Row count variance", f"{table}: {pct_change}%")

    def null_percentage_validation(self, table, column, threshold=5):
        null_pct = get_null_percentage(table, column)

        if null_pct > threshold:
            alert(f"Null percentage", f"{table}.{column}: {null_pct}%")

    def duplicate_check(self, table, key_cols):
        dups = count_duplicates(table, key_cols)

        if dups > 0:
            alert(f"Duplicates detected", f"{table}: {dups} duplicate rows")
```

---

### Escalabilidad: De 1M a 10M Transacciones/Día

**Cambios necesarios**:

1. **Procesamiento**:
   - Cluster de Spark: 8 → 20 nodos trabajadores
   - Memoria por trabajador: 8GB → 16GB
   - Paralelismo aumentado

2. **Almacenamiento**:
   - S3: Particionamiento más granular (por hora además de fecha)
   - Redshift: Más nodos (2 → 4)
   - Almacenamiento en capas frías para datos antiguos

3. **Programación**:
   - Airflow: Más trabajadores de Airflow
   - Ejecución paralela de tareas
   - Colas de tareas para optimizar la ejecución

4. **Costos**:
   - Presupuesto aumentaría a ~$25k/mes
   - Justificación: Mayor volumen de datos requiere mayor inversión

---

## Compromisos Analizados

| Decisión             | Opción A    | Opción B  | Elección    | Por qué                     |
| -------------------- | ----------- | --------- | ----------- | --------------------------- |
| **Procesamiento**    | Spark       | SQL       | Spark       | Transformaciones complejas  |
| **Particionamiento** | Por fecha   | Por clave | Por fecha   | Datos con partición natural |
| **Carga**            | Incremental | Completa  | Incremental | Eficiencia para 1M/día      |
| **Programación**     | Airflow     | Cron      | Airflow     | Monitoreo y reintentos      |
| **Almacenamiento**   | S3          | HDFS      | S3          | Costo-efectividad           |

---

## Alternativas Consideradas

### Alternativa 1: Serverless (AWS Glue)

**Pros**:

- Sin administración de clúster
- Pago por uso real

**Contras**:

- Límites de tiempo de ejecución
- Menos control sobre el entorno

**Decisión**: No adoptado. Para 1M transacciones/día, se necesita control total sobre el procesamiento.

### Alternativa 2: Puro SQL (Redshift Spectrum)

**Pros**:

- Sin transferencia de datos a S3
- Procesamiento más rápido

**Contras**:

- Límites de tamaño de consulta
- Menos flexibilidad para transformaciones complejas

**Decisión**: No adoptado. Las transformaciones complejas requieren la flexibilidad de Spark.

### Alternativa 3: Híbrido: Lambda + Kappa

**Pros**:

- Velocidad para datos críticos
- Corrección eventual a través de Kappa

**Contras**:

- Mayor complejidad operativa
- Costos más altos

**Decisión**: No adoptado. La arquitectura batch actual es suficiente para las necesidades de latencia.

---

## Errores Comunes en Entrevista

- **Error**: "Usar Hadoop/MapReduce" → **Solución**: Spark es más nuevo, más rápido y más fácil de usar
- **Error**: "Procesar TODO cada día" → **Solución**: El procesamiento incremental es más eficiente
- **Error**: "No monitorear la calidad de los datos" → **Solución**: Las validaciones automatizadas son críticas
- **Error**: "Ignorar los costos de infraestructura" → **Solución**: El equilibrio entre rendimiento y costo es clave

---

## Preguntas de Seguimiento Típicas

1. **"¿Cómo manejas los datos que llegan tarde?"**
   - Ventanas de retraso con períodos de gracia
   - Vistas materializadas actualizadas periódicamente
   - Marca de tiempo en los eventos

2. **"¿Cómo coordinas con 50 equipos de fuentes de datos?"**
   - Contratos de nivel de servicio (SLA) para la disponibilidad de datos
   - Matriz de propiedad (quién es responsable de qué fuente)
   - Bucle de retroalimentación para equipos que no cumplen

3. **"¿Cómo manejas los cambios en el esquema?"**
   - Evolución del esquema (versionado de esquemas)
   - Procesamiento compatible con versiones anteriores y nuevas
   - Vistas en la capa de almacenamiento de datos para abstraer la complejidad

4. **"¿Cómo optimizas el rendimiento de Redshift?"**
   - Claves de distribución adecuadas
   - Clasificación de almacenamiento
   - Compresión de columnas
   - Vistas materializadas para consultas frecuentes

---

## Referencias

- [AWS EMR + Spark Architecture](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-spark.html)
- [Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/index.html)
- [Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/tuning-best-practices.html)
