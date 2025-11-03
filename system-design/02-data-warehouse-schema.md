# Diseñar Data Warehouse: Star Schema & Dimensional Modeling

**Tags**: #system-design #data-warehouse #star-schema #dimensional-modeling #real-interview  
**Empresas**: Amazon, Google, Meta, Microsoft  
**Dificultad**: Senior  
**Tiempo estimado**: 28 min

---

## TL;DR

Data Warehouse = optimizado para OLAP (analytics), no OLTP (transaccional). Schema: **Star Schema** (fact + dimensions) es standard. Fact = números (sales, count, duration). Dimensions = descripciones (customer, product, date). Trade-off: Star Schema es denormalizado (rápido queries, más storage) vs Normalized (menos storage, lento queries). Para analytics: Star es ganador.

---

## Problema Real

**Escenario:**

- Retailer con 1000 tiendas, 100k productos
- Necesita responder: "Ventas por tienda, categoría, mes en último año?"
- Query speed: < 1 segundo
- Data: 500M transacciones/año
- BI team (10 personas) escriben queries daily

**Preguntas:**

1. ¿Cómo estructuras schema para analytics?
2. ¿Fact table vs Dimension table? ¿Qué va dónde?
3. ¿Cómo optimizas para este tipo de queries?
4. ¿Cómo manejas historicales (precio de producto cambia)?
5. ¿Cuándo denormalizas, cuándo normalizas?

---

## Solución: Star Schema

### Conceptos Core

FACT TABLE (números, transacciones):

sales_id (PK)

customer_id (FK)

product_id (FK)

store_id (FK)

date_id (FK)

quantity

sales_amount

discount

profit

DIMENSION TABLES (descriptores):

customer (id, name, segment, country)

product (id, name, category, subcategory)

store (id, store_name, region, city)

date (id, date, month, quarter, year)

text

### Visualización

text
DIMENSION: Product
├─ product_id (PK)
├─ name
├─ category
├─ price
└─ supplier_id
▲
│ FK
│
DIMENSION: Customer ◄─── FACT: Sales ──► DIMENSION: Store
├─ customer_id (PK) │ ├─ sales_id (PK) │ ├─ store_id (PK)
├─ name │ ├─ qty │ ├─ store_name
├─ segment │ ├─ amount │ ├─ city
└─ country │ └─ profit │ └─ region
│ │
└──► DIMENSION: Date ◄─┘
├─ date_id (PK)
├─ date
├─ month
├─ quarter
└─ year
text

---

### Definiciones SQL

-- DIMENSION: Customer
CREATE TABLE dim_customer (
customer_id INT PRIMARY KEY,
customer_name VARCHAR(100),
segment VARCHAR(50), -- Premium, Standard, Economy
country VARCHAR(50),
created_at TIMESTAMP,
updated_at TIMESTAMP
);

-- DIMENSION: Product
CREATE TABLE dim_product (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
subcategory VARCHAR(50),
price DECIMAL(10, 2),
supplier_id INT,
created_at TIMESTAMP,
updated_at TIMESTAMP
);

-- DIMENSION: Store
CREATE TABLE dim_store (
store_id INT PRIMARY KEY,
store_name VARCHAR(100),
city VARCHAR(50),
region VARCHAR(50),
country VARCHAR(50),
opened_date DATE
);

-- DIMENSION: Date
CREATE TABLE dim_date (
date_id INT PRIMARY KEY, -- 20240115
date DATE,
month INT,
month_name VARCHAR(10),
quarter INT,
year INT,
is_weekend BOOLEAN,
day_name VARCHAR(10)
);

-- FACT TABLE: Sales
CREATE TABLE fact_sales (
sales_id BIGINT PRIMARY KEY,
customer_id INT FOREIGN KEY REFERENCES dim_customer,
product_id INT FOREIGN KEY REFERENCES dim_product,
store_id INT FOREIGN KEY REFERENCES dim_store,
date_id INT FOREIGN KEY REFERENCES dim_date,
quantity INT,
sales_amount DECIMAL(10, 2),
discount_amount DECIMAL(10, 2),
profit DECIMAL(10, 2)
);

-- INDEX para speed
CREATE INDEX idx_fact_date ON fact_sales(date_id);
CREATE INDEX idx_fact_store ON fact_sales(store_id);
CREATE INDEX idx_fact_customer ON fact_sales(customer_id);

text

---

### Queries Típicas: Ahora Rápidas

-- Query 1: Ventas por categoría, mes (la que mencionaste)
SELECT
d.year,
d.month_name,
p.category,
SUM(f.quantity) as total_qty,
SUM(f.sales_amount) as total_sales,
AVG(f.profit) as avg_profit
FROM fact_sales f
JOIN dim_product p ON f.product_id = p.product_id
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.year, d.month_name, p.category
ORDER BY d.year, d.month, p.category;

-- Query 2: Top 10 clientes por gasto
SELECT
c.customer_name,
c.segment,
SUM(f.sales_amount) as total_spent,
COUNT(DISTINCT f.sales_id) as num_transactions
FROM fact_sales f
JOIN dim_customer c ON f.customer_id = c.customer_id
WHERE f.date_id >= 20240101 -- Último año
GROUP BY c.customer_id, c.customer_name, c.segment
ORDER BY total_spent DESC
LIMIT 10;

-- Query 3: Comparar store performance
SELECT
s.region,
s.store_name,
d.month_name,
SUM(f.sales_amount) as monthly_sales
FROM fact_sales f
JOIN dim_store s ON f.store_id = s.store_id
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY s.region, s.store_name, d.month_name
ORDER BY s.region, d.month;

text

---

## Dimensional Modeling: Decisiones

### Decision 1: Slowly Changing Dimensions (SCD)

**Escenario:** Producto sube de precio. ¿Cómo rastrear?

Opción 1: SCD Type 1 (Overwrite, NO history)
├─ UPDATE dim_product SET price = 50 WHERE product_id = 1
├─ ✗ Pierdes histórico (no sabes precio anterior)
├─ ✓ Simple
└─ Usa si: cambios no importantes

Opción 2: SCD Type 2 (Add new row, WITH history)
├─ Current row: product_id=1, price=45, valid_from=2024-01-01, valid_to=2024-06-30, is_current=1
├─ New row: product_id=1, price=50, valid_from=2024-07-01, valid_to=9999-12-31, is_current=1
├─ ✓ Rastrear histórico
├─ ✗ Cuidado con foreign keys (apuntan a cuál versión?)
└─ Usa si: auditoría, análisis histórico

Opción 3: SCD Type 3 (Add columns, limited history)
├─ dim_product: product_id, name, price_current, price_previous, price_previous_date
├─ ✓ Balance entre history + simplicity
├─ ✗ Solo últimos 2 valores
└─ Usa si: necesitas versión anterior nada más

text

**Recomendación para este caso:** **Type 2** (SCD Type 2). Retailers quieren saber "¿qué precio era hace 6 meses?"

-- SCD Type 2 Implementation
CREATE TABLE dim_product_v2 (
product_key INT PRIMARY KEY, -- Surrogate key (diferente a product_id)
product_id INT, -- Business key
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10, 2),
valid_from DATE,
valid_to DATE,
is_current BOOLEAN,
INDEX idx_product_id (product_id),
INDEX idx_current (is_current)
);

-- INSERT: Nuevo precio en 2024-07-01
INSERT INTO dim_product_v2
VALUES (
NULL, -- Auto-increment
1, -- product_id
"Widget A",
"Electronics",
50.00, -- Nuevo precio
'2024-07-01',
'9999-12-31',
TRUE
);

-- UPDATE: Cerrar fila anterior
UPDATE dim_product_v2
SET valid_to = '2024-06-30', is_current = FALSE
WHERE product_id = 1 AND is_current = TRUE;

-- QUERY: ¿Qué precio en 2024-05-01?
SELECT price
FROM dim_product_v2
WHERE product_id = 1
AND valid_from <= '2024-05-01'
AND valid_to >= '2024-05-01';
-- Retorna: 45.00

text

---

### Decision 2: Granularidad de Fact Table

**Opción A: Grain = Una venta (transacción)**
1 row = 1 transaction
├─ sales_id: 1001
├─ quantity: 5
├─ amount: 250

Ventaja: Máxima flexibilidad, detalle
Desventaja: Muchas filas (500M/año = lento)

text

**Opción B: Grain = Daily summary (agregado)**
1 row = total del día por store-category
├─ store_id, category, date
├─ total_qty: 1000
├─ total_amount: 50000

Ventaja: Menos filas (1000x menos = rápido)
Desventaja: No puedes preguntar por cliente específico

text

**Recomendación:** **Ambas tablas**

-- fact_sales_detail: Grain = transacción (500M rows)
-- Para queries específicas (cliente X qué compró)

-- fact_sales_daily: Grain = día (365 rows x store x category)
-- Para reportes agregados (ventas por mes)

-- BI team usa fact_sales_daily para reportes, mucho más rápido

text

---

### Decision 3: Conformed Dimensions

**Problema:** Si tienes múltiples fact tables, ¿cómo asegurar que dim_customer es consistent?

Solución: Conformed Dimensions = misma dim_customer usada por todos

fact_sales JOIN dim_customer ON fact_sales.customer_id = dim_customer.customer_id
fact_returns JOIN dim_customer ON fact_returns.customer_id = dim_customer.customer_id
fact_marketing JOIN dim_customer ON fact_marketing.customer_id = dim_customer.customer_id

Sin esto: 3 versiones diferentes de customer → análisis inconsistente
Con esto: "Source of truth" único para customer

text

---

## Optimización para Performance

### 1. Indexing Strategy

-- Índices en Fact Table (foreign keys, frequently used)
CREATE INDEX idx_fact_date ON fact_sales(date_id);
CREATE INDEX idx_fact_store ON fact_sales(store_id);
CREATE INDEX idx_fact_product ON fact_sales(product_id);

-- Composite index para queries comunes
CREATE INDEX idx_fact_store_date ON fact_sales(store_id, date_id);

-- NO indexar todo (ralentiza inserts)

text

---

### 2. Partitioning (Large Tables)

-- Particionar fact_sales por año-mes (10 GB/mes = manejable)
-- Entonces queries que filtran por mes solo leen 1 partition

CREATE TABLE fact_sales (
...
) PARTITION BY RANGE (YEAR(date_id), MONTH(date_id));

-- Query automáticamente poda particiones
SELECT \* FROM fact_sales WHERE date_id >= 20240101 AND date_id < 20240201
-- Spark/Redshift solo lee enero 2024 partition

text

---

### 3. Materialized Views (Pre-aggregated)

-- Common query: "Ventas por tienda, mes"
CREATE MATERIALIZED VIEW v_sales_by_store_month AS
SELECT
store_id,
date_trunc('month', d.date) as month,
SUM(f.sales_amount) as total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
GROUP BY store_id, date_trunc('month', d.date);

-- Query contra view (en lugar de fact table) = 100x más rápido
SELECT \* FROM v_sales_by_store_month WHERE month = '2024-01';

text

---

## Errores Comunes en Entrevista

- **Error**: Usar schema OLTP (3NF normalizado) para warehouse → **Solución**: Star Schema es mejor para analytics. Denormalizar es OK

- **Error**: No pensar en SCD → **Solución**: Dimensiones cambian (precio, categoría). Necesitas track histórico

- **Error**: Fact table demasiado granular → **Solución**: Balance detail vs performance. Considera aggregated facts

- **Error**: No indexar → **Solución**: Sin índices, queries son lentos. Pero tampoco indexar TODO

---

## Preguntas de Seguimiento

1. **"¿Cuándo usarías Snowflake Schema en lugar de Star?"**
   - Star: Denormalizado, rápido (recomendado)
   - Snowflake: Normalizado más, menos storage (rare)

2. **"¿Cómo manejas late-arriving facts?"**
   - Ventana: aceptar updates 48h tardío
   - SCD Type 2: versionar cambios

3. **"¿Cómo presupuestas storage?"**
   - 500M facts x 50 bytes/fact = 25 GB
   - Replicación, indices, overhead → 50-100 GB total

4. **"¿Cuando migras de OLTP a OLAP?"**
   - OLTP: Normalizaciones, transacciones
   - Snapshot diario → Data Warehouse
   - Luego analytics queries contra warehouse (no OLTP)

---

## References

- [Star Schema Design - Ralph Kimball](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/)
- [Slowly Changing Dimensions - Types 1-3](https://en.wikipedia.org/wiki/Slowly_changing_dimension)
- [Data Warehouse Architecture - AWS](https://aws.amazon.com/architecture/datalake/)
