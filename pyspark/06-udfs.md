# UDFs: User Defined Functions en PySpark

**Tags**: #pyspark #udfs #custom-functions #performance #real-interview

---

## TL;DR

**UDF** = función Python personalizada. Dos tipos: Python UDF (lento, flexible) y Pandas UDF (rápido, recomendado). Python UDF deserializa cada fila (overhead). Pandas UDF usa Apache Arrow (batch processing). Regla: usa Pandas UDF siempre. Python UDF solo si realmente necesitas flexibilidad.

---

## Concepto Core

- **Qué es**: UDF permite definir lógica custom que Spark no tiene built-in. Ejecuta en workers
- **Por qué importa**: A veces necesitas transformación que SQL functions no cubren. Pero UDFs pueden ser lentas (serialización overhead)
- **Principio clave**: Prefiere built-in functions > Pandas UDF > Python UDF. Cada "salto" es más lento

---

## Memory Trick

**"Envío postal"** — Python UDF = envía cada fila por separado (lento, overhead). Pandas UDF = envía batch de filas (rápido, eficiente).

---

## Cómo explicarlo en entrevista

**Paso 1**: "UDF es función Python custom que corre en Spark. Necesaria cuando built-in functions no bastan"

**Paso 2**: "Hay 2 tipos: Python UDF (flexible pero lento) y Pandas UDF (rápido, recomendado)"

**Paso 3**: "Python UDF serializa CADA fila (Python ↔ JVM). Overhead gigante. Pandas UDF usa Arrow, batch processing"

**Paso 4**: "Regla: usa built-in si posible, luego Pandas UDF, último recurso Python UDF"

---

## Código/Query ejemplo

### Escenario: Validar email

Spark no tiene función "validar email". Necesitas UDF.

---

### ❌ Opción 1: Python UDF (Lento)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import udf
from pyspark.sql.types import BooleanType
import re

spark = SparkSession.builder.appName("UDF Example").getOrCreate()

# Define función Python
def validate_email(email):
    """Validar si email es válido"""
    pattern = r'^[a-zA-Z0-9.*%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

# Registra como UDF (Python UDF)
validate_email_udf = udf(validate_email, BooleanType())

customers = spark.read.parquet("customers.parquet")

# Aplica UDF
result = customers.withColumn("email_valid", validate_email_udf(col("email")))

result.show()
```

❌ PROBLEMA:

- Cada fila: serializa a Python, valida, deserializa resultado
- 1M filas = 1M serializations (lento, overhead CPU+network)

**Performance:** ~2 minutos para 1M filas

---

### ✅ Opción 2: Pandas UDF (Rápido - Recomendado)

```python
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import BooleanType
import re
import pandas as pd

# Define función Pandas (recibe/retorna Series, no fila individual)
@pandas_udf(BooleanType())
def validate_email_pandas(emails: pd.Series) -> pd.Series:
    """Vectorizada: recibe Series de emails, retorna Series de booleans"""
    pattern = r'^[a-zA-Z0-9.*%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return emails.str.match(pattern)

customers = spark.read.parquet("customers.parquet")

# Aplica Pandas UDF
result = customers.withColumn("email_valid", validate_email_pandas(col("email")))

result.show()
```

✅ VENTAJA:

- Batch processing: 10k filas por batch (no 1 por 1)
- Arrow serialization (eficiente, columnar)
- Pandas optimizado (vectorized operations)

**Performance:** ~20 segundos para 1M filas (6x más rápido)

---

### Comparación: Python UDF vs Pandas UDF

| Aspecto           | Python UDF            | Pandas UDF           |
| ----------------- | --------------------- | -------------------- |
| **Velocidad**     | Lento (2 min / 1M)    | Rápido (20 seg / 1M) |
| **Batch Size**    | 1 fila                | ~10k filas           |
| **Serialización** | Python + JVM          | Arrow (columnar)     |
| **Sintaxis**      | `udf(func, type)`     | `@pandas_udf(type)`  |
| **Datos Entrada** | Valor escalar         | pandas.Series        |
| **Datos Salida**  | Valor escalar         | pandas.Series        |
| **Flexibilidad**  | Máxima                | Buena                |
| **Cuándo Usar**   | Rare (último recurso) | Casi siempre         |

---

## UDF por Tipo de Retorno

### Retorna Escalar (Series)

```python
# Pandas UDF que retorna Series (una columna)
@pandas_udf(StringType())
def uppercase_col(texts: pd.Series) -> pd.Series:
    return texts.str.upper()

df.withColumn("text_upper", uppercase_col(col("text")))
```

---

### Retorna Struct (Varias columnas)

```python
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

@pandas_udf(
    StructType([
        StructField("name_upper", StringType()),
        StructField("length", DoubleType())
    ])
)
def process_name(names: pd.Series) -> pd.DataFrame:
    return pd.DataFrame({
        "name_upper": names.str.upper(),
        "length": names.str.len().astype(float)
    })

df.withColumn("processed", process_name(col("name")))
```

---

### Retorna Array

```python
@pandas_udf(ArrayType(StringType()))
def split_text(texts: pd.Series) -> pd.Series:
    return texts.str.split(" ")

df.withColumn("words", split_text(col("text")))
```

---

## UDF Registrados (SQL-accessible)

```python
# Registra UDF para usar en SQL
spark.udf.register("validate_email_sql", validate_email_pandas, BooleanType())

# Ahora puedes usarlo en SQL
spark.sql("""
SELECT name, email, validate_email_sql(email) as email_valid
FROM customers
WHERE validate_email_sql(email) = false
""").show()
```

---

## Errores comunes en entrevista

- **Error**: Usar Python UDF para todo → **Solución**: Pandas UDF siempre que posible. Performance es 5-10x mejor

- **Error**: No vectorizar en Pandas UDF → **Solución**: `.str`, `.apply()` en pandas, no Python loops

- **Error**: No especificar tipo de retorno → **Solución**: Siempre declara tipo explícitamente (BooleanType(), StringType(), etc.)

- **Error**: Usar UDF cuando built-in function existe → **Solución**: Spark funciones built-in son optimizadas (Catalyst). UDF bypassa optimizer

---

## Built-in vs UDF: Cuándo Cada Uno

```python
from pyspark.sql.functions import col, regexp_replace, upper, length

# ✅ Usa BUILT-IN (rápido, optimizado)
df.withColumn("email_normalized", upper(col("email")))
df.withColumn("text_clean", regexp_replace(col("text"), r'\s+', ' '))
df.withColumn("name_length", length(col("name")))

# ❌ UDF solo si no existe built-in
# Ej: reglas de negocio custom, lógica compleja
```

---

## Performance Tips

✅ BIEN: Pandas UDF vectorizado

```python
@pandas_udf(BooleanType())
def is_valid(emails: pd.Series) -> pd.Series:
    return emails.str.match(PATTERN)  # Vectorizado
```

❌ MAL: Python loop (derrota propósito de Pandas UDF)

```python
@pandas_udf(BooleanType())
def is_valid_slow(emails: pd.Series) -> pd.Series:
    return pd.Series([bool(re.match(PATTERN, e)) for e in emails])  # ← Loop
```

Primer es 100x más rápido

---

## Preguntas de seguimiento típicas

1. **"¿Por qué Pandas UDF es más rápido?"**
   - Batch processing (10k filas a la vez)
   - Arrow serialization (columnar, comprimido)
   - Menos context switching Python ↔ JVM

2. **"¿Cuándo NO usarías UDF?"**
   - Si existe built-in function
   - Si lógica es muy simple (usa SQL expressions)
   - Si performance es crítico (última opción)

3. **"¿Cómo debuggeas UDF?"**
   - `.show()` directamente
   - Agrega `print()` en función (ojo: salida va a worker logs, no local)
   - Usa `try/except` para capturar errores

4. **"¿UDF puede tener state?"**
   - No, cada batch es independiente
   - Si necesitas state, usa `groupBy().applyInPandas()` (más complejo)

---

## Real-World: Data Validation Pipeline

```python
from pyspark.sql.functions import pandas_udf, col
from pyspark.sql.types import StructType, StructField, StringType, BooleanType
import pandas as pd

# Multi-validation UDF
@pandas_udf(
    StructType([
        StructField("is_valid", BooleanType()),
        StructField("error", StringType())
    ])
)
def validate_customer(names: pd.Series, emails: pd.Series) -> pd.DataFrame:
    results = pd.DataFrame({
        "is_valid": True,
        "error": ""
    }, index=names.index)

    # Validar nombre
    invalid_names = names.isna() | (names.str.len() < 2)
    results.loc[invalid_names, "is_valid"] = False
    results.loc[invalid_names, "error"] = "Invalid name"

    # Validar email
    invalid_emails = ~emails.str.match(r'^[\w\.-]+@[\w\.-]+\.\w+$')
    results.loc[invalid_emails, "is_valid"] = False
    results.loc[invalid_emails, "error"] = "Invalid email"

    return results

# Aplica
customers = spark.read.parquet("customers/")
validated = customers.withColumn(
    "validation",
    validate_customer(col("name"), col("email"))
)

# Explota struct
validated.select("*", "validation.*").show()
```

---

## References

- [UDFs - PySpark Docs](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html#udf)
- [Performance Guide - Spark](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
