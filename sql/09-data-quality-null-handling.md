# Calidad de Datos: Manejo de NULLs y Detección de Duplicados

**Tags**: #sql #data-quality #null-handling #duplicates #data-engineering #real-interview

---

## TL;DR

Maneja NULLs con `IS NULL`, `COALESCE()`, `NULLIF()`. Detecta duplicados con `ROW_NUMBER()` o `GROUP BY ... HAVING COUNT(*) > 1`. Valida datos con `CASE WHEN` checks. En data engineering, 80% del trabajo es limpiar datos, 20% es análisis.

---

## Concepto Core

- **Qué es**: La calidad de datos asegura que los datos sean correctos, completos y sin duplicados. Los NULLs son valores faltantes que causan problemas
- **Por qué importa**: Datos sucios = análisis sucios = decisiones malas. Los ingenieros de datos pasan 80% del tiempo limpiando. Habilidad crítica
- **Principio clave**: NULL ≠ 0 ≠ cadena vacía. Cada uno se maneja distinto. Nunca ignores los NULLs

---

## Memory Trick

**"Basura dentro, basura fuera"** (Garbage in, garbage out) — Si los datos están sucios (NULLs, duplicados, valores inválidos), el resultado será basura.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Los datos reales siempre tienen problemas: valores faltantes (NULLs), duplicados, valores inválidos"

**Paso 2**: "Para manejar NULLs: `COALESCE()` para valores por defecto, `IS NULL` para filtrar, `NULLIF()` para casos especiales"

**Paso 3**: "Para duplicados: `ROW_NUMBER()` y filtro donde rank = 1, o `GROUP BY` con `HAVING COUNT(*) > 1`"

**Paso 4**: "Valida datos con `CASE WHEN` checks y alertas"

---

## Código/Query ejemplo

### Tabla: customers (con datos sucios)

```
customer_id | name | email | phone | age | registration_date
1 | Alice | alice@example.com | NULL | 28 | 2024-01-15
2 | Bob | bob@example.com | 555-1234 | NULL| 2024-01-20
3 | Charlie | NULL | 555-5678 | 35 | 2024-02-01
4 | Alice | alice@example.com | NULL | 28 | 2024-01-15 (DUPLICATE!)
5 | David | david@example.com | 555-9999 | -5 | 2024-03-01 (INVALID!)
6 | Eve | eve@example.com | NULL | 45 | NULL (INCOMPLETE!)
7 | Frank | frank@example.com | 555-1234 | 32 | 2024-02-15
```

---

### Problema 1: Manejar NULLs con COALESCE

```sql
-- ❌ Problema: NULL values en output
SELECT
  customer_id,
  name,
  email,
  phone,
  age
FROM customers;

-- ✅ Solución: COALESCE para default values
SELECT
  customer_id,
  name,
  COALESCE(email, 'no-email@unknown.com') as email,
  COALESCE(phone, 'Not provided') as phone,
  COALESCE(age, 0) as age,
  CASE
    WHEN email IS NULL THEN 'Missing Email'
    WHEN phone IS NULL THEN 'Missing Phone'
    WHEN age IS NULL THEN 'Missing Age'
    ELSE 'Complete'
  END as data_quality_flag
FROM customers
ORDER BY customer_id;
```

**Resultado:**

```
customer_id | name | email | phone | age | data_quality_flag
1 | Alice | alice@example.com | Not provided | 28 | Missing Phone
2 | Bob | bob@example.com | 555-1234 | 0 | Missing Age
3 | Charlie | no-email@unknown.com | 555-5678 | 35 | Missing Email
...
```

---

### Problema 2: Detectar Duplicados

```sql
-- ¿Cuáles customers están duplicados?
SELECT
  customer_id,
  name,
  email,
  COUNT(*) as occurrences
FROM customers
GROUP BY name, email
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;
```

**Resultado:**

```
customer_id | name | email | occurrences
1 / 4 | Alice | alice@example.com | 2
```

---

### Problema 3: Eliminar Duplicados (Deduplication)

```sql
-- ✅ Opción 1: ROW_NUMBER (más flexible)
WITH deduped AS (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY name, email ORDER BY customer_id) as rn
  FROM customers
)
SELECT *
FROM deduped
WHERE rn = 1; -- Mantén solo el primero de cada duplicado

-- ✅ Opción 2: DISTINCT (solo si todos los campos son idénticos)
SELECT DISTINCT *
FROM customers;

-- ✅ Opción 3: GROUP BY (útil si necesitas agregación)
SELECT
  MIN(customer_id) as customer_id, -- Toma el ID más bajo
  name,
  email,
  MIN(phone) as phone,
  MAX(age) as age,
  MIN(registration_date) as registration_date
FROM customers
GROUP BY name, email;
```

---

### Problema 4: Validar Datos (Business Rules)

```sql
-- ¿Qué datos violan las reglas de negocio?
SELECT
  customer_id,
  name,
  age,
  registration_date,
  CASE
    -- Validaciones
    WHEN age < 0 OR age > 150 THEN 'Invalid: age out of range'
    WHEN age < 18 THEN 'Warning: underage'
    WHEN registration_date IS NULL THEN 'Error: missing registration date'
    WHEN registration_date > CURRENT_DATE THEN 'Error: future date'
    WHEN email IS NULL AND phone IS NULL THEN 'Error: no contact info'
    WHEN email ~ '^[A-Za-z0-9.*%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN 'Valid email'
    ELSE 'OK'
  END as data_quality_check
FROM customers
ORDER BY data_quality_check;
```

**Resultado:**

```
customer_id | name | age | registration_date | data_quality_check
5 | David | -5 | 2024-03-01 | Invalid: age out of range
6 | Eve | 45 | NULL | Error: missing registration date
2 | Bob | NULL| 2024-01-20 | Error: underage (if NULL means < 18)
...
```

---

### Problema 5: NULLIF (Convertir valores a NULL)

```sql
-- Convertir valores específicos a NULL para mejor análisis
SELECT
  customer_id,
  name,
  NULLIF(age, 0) as age, -- Convertir 0 a NULL (porque 0 es placeholder)
  NULLIF(phone, '') as phone, -- Convertir string vacío a NULL
  COALESCE(NULLIF(age, 0), 25) as age_with_default -- Si age = 0, usa 25
FROM customers;
```

---

### Problema 6: Reporte de Calidad de Datos

```sql
-- Reporte de calidad de datos
SELECT
  'Customers' as table_name,
  COUNT(*) as total_rows,
  COUNT(*) FILTER (WHERE customer_id IS NULL) as null_customer_id,
  COUNT(*) FILTER (WHERE name IS NULL) as null_name,
  COUNT(*) FILTER (WHERE email IS NULL) as null_email,
  COUNT(*) FILTER (WHERE phone IS NULL) as null_phone,
  COUNT(*) FILTER (WHERE age IS NULL) as null_age,
  COUNT(*) FILTER (WHERE age < 0 OR age > 150) as invalid_age,
  COUNT(DISTINCT name, email) as unique_customers,
  COUNT(*) - COUNT(DISTINCT name, email) as potential_duplicates
FROM customers;
```

**Resultado:**

```
table_name | total_rows | null_customer_id | null_name | null_email | null_phone | null_age | invalid_age | unique_customers | potential_duplicates
Customers | 7 | 0 | 0 | 1 | 2 | 1 | 1 | 6 | 1
```

---

## Comportamiento de NULL en SQL

| Operación            | Resultado        | Razón                                                               |
| -------------------- | ---------------- | ------------------------------------------------------------------- |
| `NULL = NULL`        | NULL (not true!) | En SQL, NULL = desconocido, desconocido = desconocido = desconocido |
| `NULL IS NULL`       | TRUE             | Forma correcta de comparar NULL                                     |
| `NULL + 5`           | NULL             | Operación con NULL = NULL                                           |
| `SUM(col)` con NULLs | Ignora NULLs     | COUNT, SUM, AVG ignoran NULLs automáticamente                       |
| `COUNT(*)` con NULLs | Incluye          | `COUNT(*)` cuenta filas, `COUNT(col)` ignora NULLs                  |

---

## Errores comunes en entrevista

- **Error**: Usar `WHERE col = NULL` → **Solución**: `WHERE col IS NULL`

- **Error**: Olvidar que NULLs afectan JOINs → **Solución**: `WHERE col1 = col2` no matchea si alguno es NULL. Usa `COALESCE` si necesitas

- **Error**: No validar datos antes de análisis → **Solución**: Siempre haz data quality checks primero

- **Error**: Asumir que DISTINCT elimina duplicados cuando hay NULLs → **Solución**: DISTINCT con NULLs es tricky. Usa ROW_NUMBER para control fino

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre COALESCE y IFNULL?"**
   - COALESCE: retorna primer valor no-NULL (soporta múltiples)
   - IFNULL: solo dos argumentos (más limitado)
   - COALESCE es más estándar

2. **"¿Cómo determinas si duplicados son reales o errores?"**
   - Analiza timestamps: si son idénticos = error
   - Analiza IDs: si son diferentes = posible entidad duplicada
   - Habla con data owner para reglas

3. **"¿Qué haces con duplicados: eliminas o archivas?"**
   - Nunca elimines sin documentar
   - Archiva en tabla histórica
   - Marca como "deduped_source_id" para trazabilidad

4. **"¿Cómo manejas NULLs en agregaciones?"**
   - SUM/AVG/COUNT ignoran automáticamente
   - Si necesitas contar NULLs: `COUNT(*) - COUNT(col)`

---

## Real-World: Ingestión en Data Warehouse

```sql
-- ETL: Limpiar datos antes de cargar a warehouse
WITH raw_data AS (
  SELECT * FROM staging.raw_customers
),

cleaned_data AS (
  SELECT
    customer_id,
    TRIM(name) as name, -- Remove spaces
    LOWER(email) as email, -- Standardize
    COALESCE(phone, 'Unknown') as phone,
    CASE
      WHEN age < 0 OR age > 150 THEN NULL -- Invalid becomes NULL
      ELSE age
    END as age,
    registration_date,
    CURRENT_TIMESTAMP as loaded_at
  FROM raw_data
  WHERE customer_id IS NOT NULL -- Must have ID
),

deduplicated AS (
  SELECT *
  FROM (
    SELECT
      *,
      ROW_NUMBER() OVER (PARTITION BY email ORDER BY registration_date) as rn
    FROM cleaned_data
  ) t
  WHERE rn = 1
)

INSERT INTO analytics.dim_customers
SELECT * FROM deduplicated;
```

---

## Checklist de Calidad de Datos

- ✅ Valores NULL identificados y manejados
- ✅ Duplicados detectados
- ✅ Valores inválidos validados (edad, fecha, etc.)
- ✅ Valores faltantes documentados
- ✅ Tipos de datos correctos
- ✅ Relaciones referenciales válidas (FK checks)
- ✅ Datos históricos preservados (audit trail)

---

## Referencias

- [NULL Handling - PostgreSQL Docs](https://www.postgresql.org/docs/current/functions-comparison.html)
- [Data Quality in SQL - Mode Analytics](https://mode.com/sql-tutorial/)
