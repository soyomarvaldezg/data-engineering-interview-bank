# Funciones de Ventana en SQL

**Tags**: #sql #window-functions #intermediate

---

## TL;DR (Respuesta en 30 segundos)

Las funciones de ventana (window functions) permiten calcular valores sobre un conjunto de filas relacionadas sin hacer colapsar los resultados en grupos. Usan `OVER()` para definir la ventana.

---

## Concepto Core

- **Qué es**: Una función de ventana calcula un valor para cada fila basándose en un conjunto de filas definido por la cláusula `OVER()`.

- **Por qué importa**: Es fundamental en data warehousing. Permite cálculos complejos (rankings, running totals) sin perder el nivel de detalle de la fila.

- **Principio clave**: Window functions = agregaciones sin GROUP BY (mantienes las filas originales)

---

## Memory Trick

**"OVER es la clave"** — Sin `OVER()`, es una función agregada normal. Con `OVER()`, es una window function.

Ejemplo mental:

- `SUM(salary)` → Te da 1 número
- `SUM(salary) OVER()` → Te da el suma total EN CADA FILA

---

## Cómo explicarlo en entrevista

**Paso 1: Define la función**
"Una window function me permite calcular valores que se necesitan preservando las filas individuales."

**Paso 2: Explica OVER()**
"La cláusula OVER() define qué filas participan en el cálculo:

- `PARTITION BY`: Agrupa lógicamente (como GROUP BY pero sin colapsar)
- `ORDER BY`: Define el orden (importante para RANK, LAG, etc.)"

**Paso 3: Da un ejemplo real**
"Si necesito el ranking de salarios POR DEPARTAMENTO, uso:
`ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)`"

---

## Código/Query ejemplo

```sql
-- Pregunta: "Encuentra el salario más alto por departamento
-- pero mantén todas las filas"

SELECT
    employee_id,
    name,
    department,
    salary,
    MAX(salary) OVER (PARTITION BY department) AS dept_max_salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept
FROM employees
ORDER BY department, salary DESC;
```

**Resultado:**
| employee_id | name | department | salary | dept_max | rank |
|---|---|---|---|---|---|
| 1 | Alice | Sales | 50000 | 60000 | 2 |
| 2 | Bob | Sales | 60000 | 60000 | 1 |
| 3 | Charlie| IT | 80000 | 80000 | 1 |

---

## Errores comunes en entrevista

- **Error**: Usar `GROUP BY` cuando necesitas mantener filas → **Solución**: Usa window functions con `OVER()`

- **Error**: Olvidar `PARTITION BY` cuando necesitas segregar datos → **Solución**: `PARTITION BY` es como GROUP BY dentro de window functions

- **Error**: Confundir `ROW_NUMBER()`, `RANK()` y `DENSE_RANK()` → **Solución**: Memóriza: ROW_NUMBER es secuencial, RANK salta números, DENSE_RANK no salta

---

## Preguntas de seguimiento típicas

1. **"¿Cuál es la diferencia entre `ROW_NUMBER()`, `RANK()` y `DENSE_RANK()`?"**
   - ROW_NUMBER: 1, 2, 3, 4
   - RANK: 1, 2, 2, 4 (salta después de empate)
   - DENSE_RANK: 1, 2, 2, 3 (no salta)

2. **"¿Cómo optimizarías un query con múltiples window functions?"**
   - Usa una sola ventana cuando sea posible
   - Cuidado con la complejidad O(n log n)

3. **"¿Cuándo usarías `LAG()` o `LEAD()`?"**
   - LAG: Acceder a fila anterior
   - LEAD: Acceder a fila siguiente
   - Caso de uso: Calcular diferencia mes-a-mes

---

## Referencias

- [Window Functions en SQL - Documentación PostgreSQL](https://www.postgresql.org/docs/current/tutorial-window.html)
- [SQL Window Functions - Mode Analytics](https://mode.com/sql-tutorial/sql-window-functions/)
