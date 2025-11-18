# Slowly Changing Dimensions (SCD)

**tags**: #data-modeling #scd #dimensions #history #tracking #real-interview

---

## En resumen (TL;DR)

**SCD** = cómo trackear (rastrear) cambios en las **dimensions** a lo largo del tiempo. **Tipos**: (1) Overwrite (sin **history**), (2) Add row (**history** completa), (3) Add columns (**history** reciente), (4) Hybrid. **Realidad**: La mayoría de los **analytics** = **Type 2** (trackea todo). **Reto**: Sin **SCD** = análisis histórico incorrecto ("el **price** era de $100" pero la **history** dice $50).

---

## Concepto Core

- **Qué es**: Los **attributes** de las **dimensions** cambian (**price**, **location**, **name**). Necesidad de trackear **history**.
- **Por qué importa**: Si ignoras los cambios, el análisis histórico es INCORRECTO.
- **Principio clave**: Diseñar **SCD** de antemano, no después de que los **data** se corrompan.

---

## Problema: Por qué SCD es importante

Ejemplo: **dimension** de **Product**

Escenario: Nike Air Max
├─ 2024-01-01: **Price** = $120, Color = Red
├─ 2024-06-01: **Price** = $99 (sale)
├─ 2024-08-01: Color = Blue (restocked)

Pregunta: "¿Por cuánto vendimos Air Max en el Q2 de 2024?"

Sin **SCD** (**overwrite**):
├─ **dim_product**: **product_id**=1, **name**="Air Max", **price**=$99, color=Blue
├─ Old **prices**/colors: PERDIDOS
├─ Respuesta: "Vendimos por $99" (¡INCORRECTO! Era $120 en Q2)

Con **SCD Type 2**:
├─ Row 1: **product_id**=1, **valid_from**=2024-01-01, **valid_to**=2024-06-01, **price**=$120
├─ Row 2: **product_id**=1, **valid_from**=2024-06-01, **valid_to**=2024-08-01, **price**=$99
├─ Row 3: **product_id**=1, **valid_from**=2024-08-01, **valid_to**=9999-12-31, **price**=$99
├─ Old **data**: PRESERVADOS
├─ Respuesta: "Las **sales** de Q2 fueron de $120" (¡CORRECTO!)

---

## SCD Type 1: Overwrite (Simple, Con Pérdida)

```sql
CREATE TABLE dim_product_type1 (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
updated_at TIMESTAMP
);

-- Initial insert
INSERT INTO dim_product_type1
VALUES (1, 'Nike Air Max', 'Shoes', 120.00, NOW());

-- Price changes
UPDATE dim_product_type1 SET price = 99.00, updated_at = NOW() WHERE product_id = 1;
-- El old price (120) se ha PERDIDO

-- Query: "¿Cuál era el price en Q2?"
SELECT price FROM dim_product_type1 WHERE product_id = 1;
-- Retorna: 99.00 (¡INCORRECTO! El price de Q2 era 120)
```

Pros:
✓ Simple (solo **UPDATE**)
✓ Poco **storage**

Contras:
✗ Sin **history** (no puede responder "¿el **price** era diferente antes?")
✗ **Analytics** incorrectos
✗ Pesadilla de **compliance**

Casos de uso:

Solo **attributes** no críticos (**attributes** poco usados)

La **Correctness** no es importante

---

## SCD Type 2: Add Row (History Completa, Recomendado)

```sql
CREATE TABLE dim_product_type2 (
product_key INT PRIMARY KEY, -- Surrogate key (auto-increment)
product_id INT, -- Business key (lo que los clientes conocen)
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
valid_from DATE,
valid_to DATE,
is_current BOOLEAN,
INDEX idx_product_id (product_id),
INDEX idx_current (is_current)
);

-- Initial insert
INSERT INTO dim_product_type2
VALUES (
NULL, -- Auto-increment
1, -- Business key
'Nike Air Max',
'Shoes',
120.00,
'2024-01-01',
'9999-12-31',
TRUE
);
-- Resultado: product_key=1000, product_id=1

-- Price changes el 2024-06-01
UPDATE dim_product_type2 SET valid_to = '2024-05-31', is_current = FALSE WHERE product_key = 1000;

INSERT INTO dim_product_type2
VALUES (
NULL,
1,
'Nike Air Max',
'Shoes',
99.00,
'2024-06-01',
'9999-12-31',
TRUE
);
-- Resultado: product_key=1001, product_id=1

-- Query: "¿Cuál era el price en Q2 2024?"
SELECT price FROM dim_product_type2
WHERE product_id = 1
AND valid_from <= '2024-06-15'
AND valid_to >= '2024-06-15';
-- Retorna: 120.00 (¡CORRECTO!)

-- Query: "Current price"
SELECT price FROM dim_product_type2
WHERE product_id = 1 AND is_current = TRUE;
-- Retorna: 99.00 (¡CORRECTO!)

-- Query: "Price history"
SELECT price, valid_from, valid_to FROM dim_product_type2
WHERE product_id = 1
ORDER BY valid_from;
-- Retorna: 120 (Ene-May), 99 (Jun+) con dates
```

Pros:
✓ Full **history** (puede responder cualquier pregunta de tiempo)
✓ **Analytics** correctos
✓ **Compliance**-friendly (**audit trail**)

Contras:
✗ Más **rows** (**overhead** de **storage**)
✗ Requiere **surrogate key** (**business key** != **PK**)
✗ **Queries** más complejos (condiciones de **date**)

Casos de uso:

La mayoría de los **analytics** (opción **default**)

Industrias auditadas (**HIPAA**, **PCI**, **GDPR**)

Análisis **price**/**cost sensitive**

---

## SCD Type 3: Add Columns (History Reciente)

```sql
CREATE TABLE dim_product_type3 (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price_current DECIMAL(10,2),
price_previous DECIMAL(10,2),
price_updated_date DATE,
color_current VARCHAR(50),
color_previous VARCHAR(50),
color_updated_date DATE,
updated_at TIMESTAMP
);

-- Initial
INSERT INTO dim_product_type3
VALUES (1, 'Nike Air Max', 'Shoes', 120.00, NULL, '2024-01-01', 'Red', NULL, NULL, NOW());

-- Price changes
UPDATE dim_product_type3
SET price_previous = price_current,
price_current = 99.00,
price_updated_date = NOW(),
updated_at = NOW()
WHERE product_id = 1;
-- Resultado: price_current=99, price_previous=120

-- Query: "Current and previous price"
SELECT price_current, price_previous FROM dim_product_type3 WHERE product_id = 1;
-- Retorna: current=99, previous=120 (CORRECTO para reciente)

-- Query: "¿Cuál era el price hace 3 meses?"
-- ¡No se puede responder! Solo se tiene current + 1 previous
```

Pros:
✓ Balance entre **Type 1** y **Type 2**
✓ Un poco más de **history** que **Type 1**
✓ Menos **storage** que **Type 2**

Contras:
✗ No puede responder "**what if**" para **old dates**
✗ Limited **history** (solo 2 **versions**)
✗ **Schema** complejo (muchas columnas [**field**]\_current, [**field**]\_previous)

Casos de uso:

Solo se necesita el **previous value** reciente

**Storage** limitado

Raro

---

## SCD Type 4: History Table (Hybrid)

```sql
-- Current table (Type 1)
CREATE TABLE dim_product (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
updated_at TIMESTAMP
);

-- History table (estilo Type 2)
CREATE TABLE dim_product_history (
product_history_id INT PRIMARY KEY,
product_id INT FK,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
valid_from DATE,
valid_to DATE
);

-- Patrón de Insert/update:
-- 1. Registrar los current values en la history table (antes del change)
-- 2. Updatear la current table

-- Query current:
SELECT price FROM dim_product WHERE product_id = 1;

-- Query historical:
SELECT price FROM dim_product_history WHERE product_id = 1 AND valid_to >= date;
```

Pros:
✓ **Current queries** rápidas (sin **filtering** por **dates**)
✓ **History** preservada (**separate table**)

Contras:
✗ Más complejo (2 **tables** para **maintain**)
✗ **Logic** de la **application** (necesidad de **manage history**)

Casos de uso:

Necesidad de **fast current** + **historical queries**

Large **dimensions**

---

## Ejemplo de Implementación: Retail

```sql
-- Dimension: Store
CREATE TABLE dim_store_type2 (
store_key INT PRIMARY KEY,
store_id INT,
store_name VARCHAR(100),
city VARCHAR(50),
region VARCHAR(50),
manager_name VARCHAR(100),
square_feet INT,
valid_from DATE,
valid_to DATE,
is_current BOOLEAN,
INDEX idx_store_id (store_id),
INDEX idx_current (is_current)
);

-- Escenario: Store se reubica, name cambia, manager cambia
-- 2024-01-01: Store abierta, manager = John
-- 2024-06-01: Manager = Sarah
-- 2024-09-01: Store reubicada, new city/region

-- Insert 1: Initial
INSERT INTO dim_store_type2 VALUES (
1000, 1, 'Downtown Mall Store', 'Boston', 'Northeast', 'John', 5000,
'2024-01-01', '9999-12-31', TRUE
);

-- Insert 2: Manager change (2024-06-01)
UPDATE dim_store_type2 SET valid_to = '2024-05-31', is_current = FALSE WHERE store_key = 1000;
INSERT INTO dim_store_type2 VALUES (
1001, 1, 'Downtown Mall Store', 'Boston', 'Northeast', 'Sarah', 5000,
'2024-06-01', '9999-12-31', TRUE
);

-- Insert 3: Relocation (2024-09-01)
UPDATE dim_store_type2 SET valid_to = '2024-08-31', is_current = FALSE WHERE store_key = 1001;
INSERT INTO dim_store_type2 VALUES (
1002, 1, 'Harbor Plaza Store', 'Salem', 'Northeast', 'Sarah', 7000,
'2024-09-01', '9999-12-31', TRUE
);

-- Analytics query: "Sales por manager, 2024"
SELECT
ds.manager_name,
SUM(fs.sales_amount) as total_sales
FROM fact_sales fs
JOIN dim_store_type2 ds ON fs.store_id = ds.store_id AND fs.date_id >= ds.valid_from AND fs.date_id <= ds.valid_to
WHERE YEAR(ds.valid_from) = 2024
GROUP BY ds.manager_name;
-- Atribuye correctamente las sales al manager que estuvo allí en ese momento
```

---

## Performance: Optimización de Queries de SCD Type 2

```sql
-- Slow: Múltiples conditions
SELECT price FROM dim_product_type2
WHERE product_id = 1
AND valid_from <= '2024-06-15'
AND valid_to >= '2024-06-15';
-- El Date filtering puede ser slow

-- Fast: Usar el flag is_current para la mayoría de los queries
SELECT price FROM dim_product_type2
WHERE product_id = 1 AND is_current = TRUE;
-- El Index en is_current lo hace fast

-- Fast: Partition por date
CREATE TABLE dim_product_type2_2024_q2 AS (
SELECT * FROM dim_product_type2
WHERE valid_from >= '2024-04-01' AND valid_to <= '2024-06-30'
);
-- Query a la partitioned table (smaller scans)
```

---

## Real-World: History de Pricing en E-commerce

Escenario: Amazon trackea los **product prices** para **analytics**

Pregunta: "Análisis de Elasticidad: cuando bajamos el **price** de $100 a $80, ¿cuántas **units** más se venden?"

Sin **SCD**: No se puede responder (solo se tiene el **current price**)
Con **SCD Type 2**: Full **price history** → se pueden correlacionar los **price changes** con el **sales volume**

Implementación:

**Nightly**: Verificar si el **price** cambió

Si cambió: Cerrar la old **dim_product row**, abrir una **new row** con el **new price**

**Analytics**: **JOIN fact_sales** con **dim_product** en el **date range**

Resultado: Accurate **price** por **transaction date**

---

## Errores Comunes en Entrevista

- **Error**: "Type 1 is sufficient" → **Solución**: ¡Incorrecto! El **historical analysis** se rompe

- **Error**: Olvidar la **surrogate key** (usar **business key** como **PK**) → **Solución**: La **business key** puede cambiar, se necesita un **PK stable**

- **Error**: No filtrar **dates** en los **joins** → **Solución**: Un **Join** sin **date range** = pesadilla de **cartesian product**

- **Error**: Demasiadas **Type 2 dimensions** → **Solución**: El **Storage** explota. Usar **Type 1** para **attributes** que cambian raramente

---

## Preguntas de Seguimiento

1.  **"¿Cuándo Type 1 vs Type 2?"**
    - **Type 1**: Non-critical (description, color)
    - **Type 2**: Audited (**price**, **location**, **status**)

2.  **"¿Performance de los Type 2 queries?"**
    - Slower (**date filtering**)
    - Mitigar: **is_current flag**, **partitioning**, **indexing**

3.  **"¿Deletes en SCD Type 2?"**
    - No **deletear**, solo setear **valid_to** a **yesterday**
    - Preserva **history**, **soft delete**

4.  **"¿SCD para dimensions y facts?"**
    - **Dimension**: Siempre **SCD** (trackea **attribute changes**)
    - **Fact**: Nunca **SCD** (**facts** no cambian, solo se añaden)

---

## Referencias

- [SCD Types - Ralph Kimball](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/slowly-changing-dimensions/)
