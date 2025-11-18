# Star Schema & Dimensional Modeling

**Tags**: #data-modeling #star-schema #dimensional #facts #dimensions #real-interview

---

## TL;DR

**Star Schema** = _fact table_ (números) + _dimension tables_ (descriptores). **Principio de diseño**: la _fact table_ es estrecha, las _dimensions_ son anchas. **Beneficio**: las consultas son rápidas (pre‑joined), las _dimensions_ son reutilizables (_conformed_). **Compensación**: _denormalized_ (más almacenamiento) vs consultas rápidas. Mejor práctica: _grain_ de la _fact_ = una fila por transacción/evento.

---

## Concepto Core

- **Qué es**: Star Schema = unión de _fact_ + _dimensions_ (se ve como una estrella en el diagrama)
- **Por qué importa**: Las consultas analíticas necesitan esta estructura (consultas 100× más rápidas)
- **Principio clave**: _Fact_ = lo que mides (ventas), _Dimension_ = cómo lo segmentas (tiempo, tienda, producto)

---

## Cómo explicarlo en entrevista

**Paso 1**: “Star Schema = _fact_ + _dimensions_. _Fact_ = numbers (sales, count), _Dimension_ = descriptors (date, store, customer)”

**Paso 2**: “_Fact table grain_ = granularidad (una fila por venta, no por tipo de producto)”

**Paso 3**: “_Dimensions_ are _conformed_ (reused across _fact tables_)”

**Paso 4**: “Performance: Pre‑_denormalized_ = _fast queries_, some _storage overhead_”

---

## Diagrama

```
DIM_DATE
├─ date_id (PK)
├─ date
├─ month
└─ year
▲
│
DIM_CUSTOMER ◄────── FACT_SALES ──────► DIM_PRODUCT
├─ customer_id (PK) ├─ sales_id (PK) ├─ product_id (PK)
├─ name            ├─ customer_id (FK)├─ name
├─ segment         ├─ product_id (FK) ├─ category
└─ country         ├─ store_id (FK)   └─ price
   ├─ date_id (FK) │
   ├─ qty         │
   ├─ amount      │
   └─ profit      │
▲
│
DIM_STORE
├─ store_id (PK)
├─ store_name
├─ city
└─ region
```

---

## SQL: Star Schema Tables

```sql
-- FACT TABLE (Core)
CREATE TABLE fact_sales (
    sales_id BIGINT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    store_id INT NOT NULL,
    date_id INT NOT NULL,
    quantity INT,
    sales_amount DECIMAL(10,2),
    discount DECIMAL(10,2),
    profit DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES dim_customer,
    FOREIGN KEY (product_id) REFERENCES dim_product,
    FOREIGN KEY (store_id) REFERENCES dim_store,
    FOREIGN KEY (date_id) REFERENCES dim_date
);

-- DIMENSIONS (Descriptive)
CREATE TABLE dim_customer (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    segment VARCHAR(50), -- Premium, Standard
    country VARCHAR(50),
    created_at TIMESTAMP
);

CREATE TABLE dim_product (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    subcategory VARCHAR(50),
    price DECIMAL(10,2)
);

CREATE TABLE dim_store (
    store_id INT PRIMARY KEY,
    store_name VARCHAR(100),
    city VARCHAR(50),
    region VARCHAR(50),
    country VARCHAR(50)
);

CREATE TABLE dim_date (
    date_id INT PRIMARY KEY, -- 20240115
    date DATE,
    month INT,
    month_name VARCHAR(10),
    quarter INT,
    year INT,
    day_name VARCHAR(10)
);

-- INDEXES (Critical for query performance)
CREATE INDEX idx_fact_date ON fact_sales(date_id);
CREATE INDEX idx_fact_store ON fact_sales(store_id);
CREATE INDEX idx_fact_customer ON fact_sales(customer_id);
CREATE INDEX idx_fact_product ON fact_sales(product_id);
```

---

## Queries: Star Schema Benefits

```sql
-- Query 1: Sales by category, month (simple join)
SELECT
    d.year,
    d.month_name,
    p.category,
    SUM(f.quantity) as qty_sold,
    SUM(f.sales_amount) as revenue,
    AVG(f.profit) as avg_profit
FROM fact_sales f
JOIN dim_product p ON f.product_id = p.product_id
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.year, d.month_name, p.category
ORDER BY revenue DESC;

-- Query 2: Top stores by region
SELECT
    s.region,
    s.store_name,
    SUM(f.sales_amount) as total_sales,
    COUNT(DISTINCT f.customer_id) as unique_customers
FROM fact_sales f
JOIN dim_store s ON f.store_id = s.store_id
WHERE f.date_id >= 20240101
GROUP BY s.region, s.store_name
ORDER BY total_sales DESC;

-- Performance: Index on fact_sales(date_id) helps (partition pruning if partitioned)
-- Execution time: <1 second (for 500M rows)
```

---

## Fact Table: Grain

**Grain** = nivel de detalle (¿qué representa una fila?)

**Ejemplo: Transacciones de venta**

- **Opción de Grain 1**: Una fila por **TRANSACTION**

  ```
  ├─ sales_id, customer_id, product_id, amount
  ├─ Pros: más detallado, responde cualquier pregunta
  ├─ Cons: tabla enorme (>1B filas/año)
  └─ Mejor para: analítica detallada
  ```

- **Opción de Grain 2**: Una fila por **PRODUCT PER TRANSACTION**

  ```
  ├─ sales_id, customer_id, product_id, product_qty, product_amount
  ├─ Pros: todavía detallado, ligeramente más pequeño
  └─ Cons: sigue siendo grande
  ```

- **Opción de Grain 3**: Una fila por **CUSTOMER PER DAY**
  ```
  ├─ customer_id, date_id, total_qty, total_amount
  ├─ Pros: mucho más pequeño (≈1M filas vs 1B)
  └─ Cons: se pierde detalle del producto
  ```

**_Grain_ incorrecto = respuestas incorrectas!**

_Ejemplo de error_:

```sql
-- Si el grain es PER DAY (una fila por cliente por día)
SELECT customer_id, SUM(sales_amount)
FROM fact_sales
GROUP BY customer_id;
```

**ERROR**: Suma `sales_amount` varias veces cuando hay varios productos el mismo día.  
**CORRECTO**: Definir el _grain_ claramente, por ejemplo: una fila por (customer, product, date).

---

## Conformed Dimensions

**Conformed** = la misma _dimension_ reutilizada en múltiples _fact tables_.

**Ejemplo: _Dimension Customer_**

```
Fact Table 1: fact_sales
   ├─ customer_id (FK → dim_customer)

Fact Table 2: fact_returns
   ├─ customer_id (FK → dim_customer)

Fact Table 3: fact_customer_support_calls
   ├─ customer_id (FK → dim_customer)
```

**Beneficio**: “_Conformed dimension_”

- La definición de `customer_id` es idéntica en todos lados
- Las consultas que cruzan _fact tables_ son consistentes
- No hay datos conflictivos
- Un único _source of truth_ para _attributes_ del _customer_

**vs. Non‑conformed**:

```
fact_sales → cust_id
fact_returns → customer_id
fact_calls → caller_id
```

Pesadilla: definiciones distintas, unión difícil.

**Reglas de _Conformed Dimensions_**

- La _PK_ es única en todas las _fact tables_
- Los _attributes_ son consistentes
- No hay datos específicos de una _fact table_ dentro de la _dimension_
- Actualizaciones en la _dimension_ afectan a todas las _fact tables_ uniformemente

---

## Degenerate Dimensions

**Degenerate** = _data_ de _dimension_ almacenados directamente en la _fact table_ (no hay tabla separada).

**Ejemplo**: _Order number_, _invoice number_

- **Normal**: _dimension table_ separada

  ```
  dim_order (order_id, order_number, order_date, order_status)
  fact_sales (order_id FK, amount)
  ```

- **Degenerate**: en la _fact_ directamente
  ```
  fact_sales (order_number, order_date, order_status, amount)
  ```

**Cuándo usar**  
✓ Si la _dimension_ tiene solo 1‑2 _attributes_  
✓ Si nunca será referenciada por otras _fact tables_  
✓ Si es única por fila de _fact_

**Ejemplo**:

```sql
CREATE TABLE fact_orders_degenerate (
    order_id INT PRIMARY KEY,
    order_number VARCHAR(50), -- Degenerate (no tabla separada)
    customer_id INT FK,
    date_id INT FK,
    amount DECIMAL
);
```

**Beneficio**: ligeramente menos almacenamiento, _schema_ más simple.

---

## Slowly Changing Dimensions Preview (Covered Next)

**Problema**: Los _attributes_ de una _dimension_ cambian con el tiempo

- Cambia el nombre del _customer_ (matrimonio)
- Cambia la ubicación de la _store_ (reubicación)
- Cambia el _price_ del _product_

**¿Cómo rastrear el historial?**

**Tipos _SCD_**

- **Type 1**: Sobrescribir (pierde historial)
- **Type 2**: Insertar nueva fila (mantiene historial)
- **Type 3**: Añadir columnas (historial limitado)

---

## Real-World: E‑commerce Data Warehouse

```sql
-- Fact: Orders
CREATE TABLE fact_orders (
    order_id BIGINT PRIMARY KEY,
    customer_id INT FK,
    product_id INT FK,
    store_id INT FK,
    date_id INT FK,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    discount DECIMAL(10,2),
    tax DECIMAL(10,2),
    profit DECIMAL(10,2)
);

-- Fact: Order Line Items (different grain)
CREATE TABLE fact_order_line_items (
    line_item_id BIGINT PRIMARY KEY,
    order_id BIGINT FK,
    product_id INT FK,
    quantity INT,
    unit_price DECIMAL(10,2),
    line_total DECIMAL(10,2)
);

-- Fact: Customer Activity
CREATE TABLE fact_customer_activity (
    activity_id BIGINT PRIMARY KEY,
    customer_id INT FK,
    date_id INT FK,
    num_page_views INT,
    num_clicks INT,
    session_duration_sec INT,
    purchase_amount DECIMAL(10,2)
);
```

**Consulta de ejemplo**:

```sql
SELECT
    d.month_name,
    SUM(o.total_amount) as revenue,
    SUM(o.profit) as profit,
    COUNT(DISTINCT o.customer_id) as unique_customers,
    AVG(o.total_amount) as avg_order
FROM fact_orders o
JOIN dim_date d ON o.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.month_name;
```

_Todas las consultas siguen el mismo patrón: \_fact_ + _dimensions_ (_conformed_).\_

---

## Errores Comunes en Entrevista

- **Error**: “_Grain_ is ambiguous” → **Solución**: SIEMPRE definir el _grain_ explícitamente.
- **Error**: Mezclar _grains_ en una tabla → **Solución**: Agrupaciones incorrectas = respuestas erróneas.
- **Error**: No usar _conformed dimensions_ → **Solución**: _Fact tables_ diferentes no pueden compararse.
- **Error**: _Fact table_ demasiado ancha (1000+ columnas) → **Solución**: Dividir en varias _facts_ más pequeñas.

---

## Preguntas de Seguimiento

1. **“¿Cómo defines _grain_?”**
   - Observa una fila: ¿qué representa?
   - ¿Una venta? ¿Un _product_ en la venta? ¿Un _customer_ por día?

2. **“¿_Denormalized_ vs _normalized trade‑offs_?”**
   - Star (_denormalized_): consultas rápidas, más almacenamiento.
   - _Normalized_: consultas lentas, menos almacenamiento.
   - _Data warehouse_: se elige _star_ porque el costo de consulta supera al de almacenamiento.

3. **“¿Costo de _conformed dimensions_?”**
   - Inicial: esfuerzo de diseño y coordinación.
   - Retorno: desarrollo más rápido, analítica consistente.

4. **“¿_Indexing_ de _fact table_?”**
   - _Indexes_ en **TODAS** las _foreign keys_ (date, store, customer, product).
   - Además, _compound indexes_ sobre _common filters_.

---

## References

- [_Dimensional Modeling_ - Ralph Kimball](https://www.kimballgroup.com/)
- [Star Schema - Wikipedia](https://en.wikipedia.org/wiki/Star_schema)
- [Fact and Dimension Tables](https://en.wikipedia.org/wiki/Fact_table)
