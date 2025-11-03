# Data Types & Schemas en PySpark

**Tags**: #pyspark #schemas #data-types #structtype #real-interview  
**Empresas**: Amazon, Google, Databricks, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min

---

## TL;DR

Schema define estructura de DataFrame (nombre columna, tipo, nullable). Define con `StructType + StructField` (recomendado) o inferencia automática (lento, unreliable). En producción, SIEMPRE schema explícito. Tipos comunes: StringType, IntegerType, DoubleType, TimestampType, ArrayType, MapType.

---

## Concepto Core

- **Qué es**: Schema es "plano" de DataFrame. Define qué columnas existen, qué tipos, si pueden ser NULL
- **Por qué importa**: Schema explícito = production-ready. Inferencia = lento y propenso a errores. Demuestra profesionalismo
- **Principio clave**: Especifica schema siempre. La inferencia es para exploración, nunca para producción

---

## Memory Trick

**"Blueprint de casa"** — Schema es el blueprint. Si construyes sin blueprint (inferencia), todo es caótico. Con blueprint, claro y eficiente.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Schema define estructura del DataFrame: columnas, tipos, nullability"

**Paso 2**: "Hay 2 formas: inferencia (automática) vs explícita (StructType). Inferencia = lento (2 scans), explícita = rápido (1 scan)"

**Paso 3**: "En producción, SIEMPRE explícita. Inferencia es solo para exploración rápida"

**Paso 4**: "Define con StructType([StructField(...), ...]). Especifica tipo y nullable por cada columna"

---

## Código/Query ejemplo

### Datos: JSON sin estructura clara

{"name": "Alice", "age": 28, "salary": 50000.50, "hired_date": "2024-01-15", "tags": ["python", "spark"]}
{"name": "Bob", "age": 35, "salary": 60000.00, "hired_date": "2024-02-20", "tags": ["java"]}
{"name": "Charlie", "age": null, "salary": 55000.75, "hired_date": "2024-03-10", "tags": []}

text

---

### ❌ Opción 1: Inferencia Automática (No recomendado)

from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("Schema Inference").getOrCreate()

Spark INFIERE el schema automáticamente
df = spark.read.json("employees.json")

¿Qué schema creó Spark?
df.printSchema()

Problema:

- Spark lee TODO el archivo 2 veces (ineficiente)
- Si age es NULL en primeras filas, asume StringType (incorrecto)
- StringType en age = problemas después (conversiones)
  text

**Output:**
root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- salary: double (nullable = true)
|-- hired_date: string (nullable = true)
|-- tags: array (nullable = true)
| |-- element: string (containsNull = true)

text

⚠️ **Problemas**:

- hired_date es STRING (debería ser DATE)
- tags es ARRAY<STRING> (OK, pero Spark tuvo que inferir)
- 2 scans del archivo (lento)

---

### ✅ Opción 2: Schema Explícito (Recomendado)

from pyspark.sql.types import (
StructType, StructField,
StringType, IntegerType, DoubleType,
DateType, ArrayType
)

Define schema explícitamente
schema = StructType([
StructField("name", StringType(), nullable=False), # No puede ser NULL
StructField("age", IntegerType(), nullable=True), # Puede ser NULL
StructField("salary", DoubleType(), nullable=False), # No puede ser NULL
StructField("hired_date", DateType(), nullable=False), # Tipo correcto: DATE
StructField("tags", ArrayType(StringType()), nullable=True) # Array de strings
])

Lee con schema explícito
df = spark.read.schema(schema).json("employees.json")

df.printSchema()

Ventajas:

- Solo 1 scan del archivo (rápido)
- Tipos correctos (DateType en hired_date)
- Nullable especificado (contrato claro)
  text

**Output:**
root
|-- name: string (nullable = false)
|-- age: integer (nullable = true)
|-- salary: double (nullable = false)
|-- hired_date: date (nullable = false)
|-- tags: array (nullable = true)
| |-- element: string (containsNull = true)

text

---

## Data Types Comunes

| Tipo          | Python            | Rango            | Ejemplo                      |
| ------------- | ----------------- | ---------------- | ---------------------------- |
| StringType    | str               | Any length       | "Alice"                      |
| IntegerType   | int               | -2^31 to 2^31-1  | 28                           |
| LongType      | int               | -2^63 to 2^63-1  | 123456789                    |
| DoubleType    | float             | IEEE 754         | 50000.50                     |
| FloatType     | float             | IEEE 754 32-bit  | 50000.5                      |
| BooleanType   | bool              | true/false       | True                         |
| DateType      | datetime.date     | YYYY-MM-DD       | 2024-01-15                   |
| TimestampType | datetime.datetime | with timezone    | 2024-01-15 10:30:00          |
| DecimalType   | Decimal           | Precision/Scale  | 99999.99                     |
| BinaryType    | bytes             | Binary data      | b"abc"                       |
| ArrayType     | list              | Variable length  | ["a", "b"]                   |
| MapType       | dict              | Key-value        | {"key": "value"}             |
| StructType    | dict              | Nested structure | {"name": "Alice", "age": 28} |

---

## Esquemas Complejos

### Array de Structs

schema = StructType([
StructField("employee_id", IntegerType(), False),
StructField("name", StringType(), False),
StructField("orders", ArrayType(
StructType([
StructField("order_id", IntegerType(), False),
StructField("amount", DoubleType(), False),
StructField("date", DateType(), False)
])
), True)
])

Datos:
{
"employee_id": 1,
"name": "Alice",
"orders":
{"order_id": 101, "amount": 500.0, "date": "2024-01-10"},
{"order_id": 102, "amount": 750.0, "date": "2024-01-15"}
]
}
text

---

### Nested Structs

schema = StructType([
StructField("id", IntegerType(), False),
StructField("person", StructType([
StructField("name", StringType(), False),
StructField("email", StringType(), False),
StructField("address", StructType([
StructField("street", StringType(), True),
StructField("city", StringType(), False),
StructField("zip", StringType(), True)
]), True)
]), False)
])

Datos:
{
"id": 1,
"person": {
"name": "Alice",
"email": "alice@example.com",
"address": {
"street": "123 Main St",
"city": "NYC",
"zip": "10001"
}
}
}
text

---

## Inferencia desde String (Hack útil)

Si tienes schema como string (de API, config, etc.)
schema_string = "name STRING, age INT, salary DOUBLE, hired_date DATE"

df = spark.read.schema(schema_string).json("employees.json")

Equivalente a StructType explícito pero más legible
text

---

## Nullable: Qué Significa

nullable=False: Valor NUNCA puede ser NULL (constraint de integridad)
StructField("id", IntegerType(), nullable=False)

nullable=True: Valor PUEDE ser NULL (permite faltantes)
StructField("middle_name", StringType(), nullable=True)

En SQL:
nullable=False → NOT NULL constraint
nullable=True → permite NULL
text

---

## Validación de Schema

Verificar qué schema Spark inferió (para debugging)
df.printSchema()

Obtener schema como JSON (útil para guardar/documentar)
schema_json = df.schema.json()
print(schema_json)

Comparar schemas
schema1 = StructType([StructField("id", IntegerType())])
schema2 = StructType([StructField("id", LongType())])
print(schema1 == schema2) # False (IntegerType ≠ LongType)

text

---

## Performance: Schema Explícito vs Inferencia

Inferencia (2 scans)
df_inferred = spark.read.json("big_file.json") # Scan 1: Infer schema, Scan 2: Read data

Tiempo: ~20 segundos para 1 GB
Explícito (1 scan)
df_explicit = spark.read.schema(schema).json("big_file.json") # Solo 1 scan

Tiempo: ~5 segundos para 1 GB (4x más rápido)
text

---

## Errores comunes en entrevista

- **Error**: Usar inferencia en producción → **Solución**: Siempre schema explícito. Si no conoces schema, descúbrelo primero

- **Error**: Tipo incorrecto (edad como STRING en lugar de INT) → **Solución**: Datos tendrán problemas después. Define tipos correctos upfront

- **Error**: nullable=False cuando datos tienen NULLs → **Solución**: Causará error al read. Usa nullable=True o limpia data primero

- **Error**: No documentar schema → **Solución**: Guarda schema como JSON en repo (documentación)

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre DateType y TimestampType?"**
   - DateType: Solo fecha (YYYY-MM-DD)
   - TimestampType: Fecha + hora + timezone

2. **"¿Cuándo usas MapType o ArrayType?"**
   - ArrayType: Listas variable-length (tags: ["python", "spark"])
   - MapType: Key-value pairs (config: {"env": "prod", "region": "us-west"})

3. **"¿Cómo cambias tipo de columna después de leer?"**
   - `df.withColumn("age", col("age").cast(IntegerType()))`
   - Pero mejor evitar si defines schema correcto upfront

4. **"¿Cómo manejas schema mismatch si datos son inconsistentes?"**
   - Validación pre-read
   - O: `mode="permissive"` (default, NULLs para mismatch) vs `"failfast"` (error)

---

## Real-World: Production Schema Storage

Guarda schema en repo para documentación
import json

schema_dict = {
"fields": [
{"name": "id", "type": "integer", "nullable": False},
{"name": "name", "type": "string", "nullable": False},
{"name": "salary", "type": "double", "nullable": False}
]
}

En archivo: schemas/employees.json
with open("schemas/employees.json", "w") as f:
json.dump(schema_dict, f, indent=2)

En código: carga schema
with open("schemas/employees.json") as f:
schema_dict = json.load(f)

Convierte a StructType
def dict_to_struct(schema_dict):
fields = []
for field in schema_dict["fields"]:
fields.append(StructField(
field["name"],
get_type(field["type"]),
field["nullable"]
))
return StructType(fields)

schema = dict_to_struct(schema_dict)
df = spark.read.schema(schema).json("data.json")

text

---

## References

- [Data Types - PySpark Docs](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/types.html)
- [Schemas - Databricks Guide](https://docs.databricks.com/en/spark/latest/spark-sql/language-manual/syntax/data-types.html)
- [StructType & StructField - Spark API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/types.html#pyspark.sql.types.StructType)
