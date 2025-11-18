# Pivotar Datos: Convertir Meses en Columnas

**Tags**: #sql #pivot #data-transformation #real-interview

---

## TL;DR

Usa `CASE WHEN` con `SUM` para pivotar: agrupa por cliente, luego usa CASE para crear columnas por mes. Alternativa: `PIVOT` (si tu BD lo soporta).

---

## Concepto Core

- **Qué es**: Transformar filas (meses como valores) en columnas (enero, febrero, etc.)
- **Por qué importa**: Muy común en reportes BI y data warehousing. Demuestra control sobre transformaciones complejas
- **Principio clave**: Combina `GROUP BY` + `CASE WHEN` para simular pivot, o usa `PIVOT` si disponible

---

## Memory Trick

**"De vertical a horizontal"** — Los datos están "verticales" (meses como filas). `PIVOT` los gira "horizontales" (meses como columnas).

---

## Cómo explicarlo en entrevista

**Paso 1**: "Tengo una tabla con meses como valores en una columna. Necesito convertirlos a columnas"

**Paso 2**: "Uso `GROUP BY customer_id` y dentro uso `SUM(CASE WHEN month = 'January' THEN amount END) as January`"

**Paso 3**: "Repito el CASE WHEN para cada mes. El resultado tiene una fila por cliente y una columna por mes"

---

## Código/Query ejemplo

**Entrada (datos "verticales"):**

```
customer_id | month | amount
1 | January | 100
1 | February | 150
1 | March | 200
2 | January | 50
2 | February | 75
```

**Salida deseada (datos "horizontales"):**

```
customer_id | January | February | March
1 | 100 | 150 | 200
2 | 50 | 75 | NULL
```

### Solución 1: CASE WHEN (funciona en todos los motores)

```sql
SELECT
  customer_id,
  SUM(CASE WHEN month = 'January' THEN amount ELSE 0 END) as January,
  SUM(CASE WHEN month = 'February' THEN amount ELSE 0 END) as February,
  SUM(CASE WHEN month = 'March' THEN amount ELSE 0 END) as March,
  SUM(CASE WHEN month = 'April' THEN amount ELSE 0 END) as April,
  SUM(CASE WHEN month = 'May' THEN amount ELSE 0 END) as May,
  -- ... continúa para todos los meses
  SUM(amount) as Total
FROM sales
GROUP BY customer_id
ORDER BY customer_id;
```

### Solución 2: PIVOT (SQL Server, Oracle)

```sql
SELECT *
FROM sales
PIVOT (
  SUM(amount)
  FOR month IN ('January', 'February', 'March', 'April', 'May')
)
ORDER BY customer_id;
```

### Solución 3: Dynamic PIVOT (Spark SQL)

```sql
SELECT *
FROM sales
PIVOT (
  SUM(amount)
  FOR month IN (
    SELECT DISTINCT month FROM sales ORDER BY month
  )
);
```

---

## Errores comunes en entrevista

- **Error**: Olvidar el `SUM()` alrededor del CASE WHEN → **Solución**: Sin SUM(), cada fila es una nueva columna. Con SUM(), agrega valores del mismo cliente-mes

- **Error**: No usar `GROUP BY` → **Solución**: Sin GROUP BY, el resultado sería inutilizable. Siempre agrupa por la dimensión principal (customer_id)

- **Error**: Usar `IF()` en lugar de `CASE WHEN` → **Solución**: `IF()` es más simple pero CASE WHEN es más estándar y legible

---

## Preguntas de seguimiento típicas

1. **"¿Y si hay valores NULL para algunos clientes-meses?"**
   - "Con `CASE WHEN` devuelve NULL. Puedo usar `COALESCE(total, 0)` si prefiero 0"

2. **"¿Cómo lo optimizarías para 12 meses y 1 millón de clientes?"**
   - "El query es O(n). Con índice en `(customer_id, month)` es rápido. En Spark, partición por cliente antes de pivot"

3. **"¿Diferencia entre PIVOT y CASE WHEN?"**
   - "PIVOT es sintaxis especial (no todos lo soportan). CASE WHEN es más universal. Resultados iguales"

4. **"¿Y si necesito pivotar 24 meses dinámicamente?"**
   - "En Spark SQL, puedo usar `PIVOT ... FOR month IN (SELECT DISTINCT month FROM ...)`. En SQL tradicional, necesito hardcodear"

---

## Variante: Despivotar (Lo Opuesto)

A veces necesitas ir de horizontal a vertical (UNPIVOT):

```sql
-- De esto (pivotado):
SELECT customer_id, January, February, March FROM sales_pivot;

-- A esto (despivotado):
SELECT customer_id, 'January' as month, January as amount FROM sales_pivot
UNION ALL
SELECT customer_id, 'February' as month, February as amount FROM sales_pivot
UNION ALL
SELECT customer_id, 'March' as month, March as amount FROM sales_pivot;
```

O usa `UNPIVOT` si tu BD lo soporta:

```sql
SELECT customer_id, month, amount
FROM sales_pivot
UNPIVOT (
  amount FOR month IN (January, February, March)
);
```

---

## Referencias

- [PIVOT en SQL Server - Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot)
- [CASE WHEN en SQL - W3Schools](https://www.w3schools.com/sql/sql_case.asp)
