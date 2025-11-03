# Segundo Salario Más Alto por Departamento

**Tags**: #sql #window-functions #ranking #real-interview  
**Empresas**: Accenture, Bank of America, Amazon  
**Dificultad**: Mid  
**Tiempo estimado**: 12 min

---

## TL;DR

Usa `DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC)` para rankear salarios por departamento, luego filtra donde rank = 2. Si no hay segundo salario, la fila no aparece.

---

## Concepto Core

- **Qué es**: Necesitas encontrar el segundo valor más alto dentro de cada grupo (departamento)
- **Por qué importa**: Es un patrón muy común en entrevistas: "top N por grupo". Demuestra que entiendes window functions y PARTITION BY
- **Principio clave**: DENSE_RANK no salta números en empates, ideal cuando múltiples personas tienen el mismo salario

---

## Memory Trick

**"Ranking con burbujas departamentales"** — Cada departamento es una burbuja. DENSE_RANK rankea dentro de cada burbuja sin saltar números incluso con empates.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Necesito rankear salarios en orden descendente dentro de cada departamento"

**Paso 2**: "Uso `DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC)` para crear rangos. Si dos personas en Ventas ganan 50k, ambas son rank 1, la siguiente es rank 2"

**Paso 3**: "Filtro donde rank = 2 para obtener el segundo salario. Si no hay segundo salario (solo 1 persona), esa fila no aparece"

---

## Código/Query ejemplo

-- Opción 1: Con DENSE_RANK (recomendado)
WITH ranked_salaries AS (
SELECT
employee_id,
employee_name,
department,
salary,
DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees
)
SELECT
employee_id,
employee_name,
department,
salary
FROM ranked_salaries
WHERE rank = 2
ORDER BY department, salary DESC;

text

**Resultado esperado:**
employee_id | employee_name | department | salary
2 | Bob | Sales | 45000
4 | David | IT | 75000

text

---

## Variante: ¿Qué pasa con empates?

Si dos personas tienen el mismo segundo salario:

-- DENSE_RANK muestra ambos
department | salary | rank
Sales | 45000 | 2
Sales | 45000 | 2 ← ambos aparecen

-- ROW_NUMBER solo muestra uno
Sales | 45000 | 2
Sales | 40000 | 3 ← salta a 3

text

---

## Errores comunes en entrevista

- **Error**: Usar `RANK()` en lugar de `DENSE_RANK()` → **Solución**: RANK salta números (1, 2, 2, 4), DENSE_RANK no (1, 2, 2, 3). Para "segundo salario" usa DENSE_RANK

- **Error**: Olvidar el `WHERE rank = 2` y devolver todos los rangos → **Solución**: Siempre filtra específicamente el rank que necesitas

- **Error**: Usar `GROUP BY` con `MAX(salary)` dos veces en lugar de window functions → **Solución**: GROUP BY colapsaría el resultado. Window functions preservan cada fila

---

## Preguntas de seguimiento típicas

1. **"¿Y si hay un empate en el segundo salario, qué haces?"**
   - "Con DENSE_RANK, ambos aparecen. Si necesitas solo uno, cambio a ROW_NUMBER"

2. **"¿Cómo lo optimizarías para 10 millones de filas?"**
   - "Aseguro que hay índice en `(department, salary DESC)`. Spark: repartición por departamento antes de la window function"

3. **"¿Qué pasa si un departamento solo tiene 1 empleado?"**
   - "Esa fila NO aparecerá en el resultado porque no hay rank = 2. Si necesitas mostrarlos, uso COALESCE o LEFT JOIN"

4. **"¿Diferencia entre RANK y DENSE_RANK?"**
   - "RANK: 1, 2, 2, 4 (salta). DENSE_RANK: 1, 2, 2, 3 (no salta). Para 'segundo' uso DENSE_RANK"

---

## Código alternativo (Sin CTE, directo)

SELECT
employee_id,
employee_name,
department,
salary
FROM (
SELECT
employee_id,
employee_name,
department,
salary,
DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees
) subquery
WHERE rank = 2
ORDER BY department, salary DESC;

text

⚠️ **Nota**: Algunos motores no permiten WHERE directo sobre window functions. Por eso la solución con CTE es más robusta.

---

## Referencias

- [Window Functions - PostgreSQL Official Docs](https://www.postgresql.org/docs/current/tutorial-window.html)
- [RANK vs DENSE_RANK vs ROW_NUMBER - Mode Analytics](https://mode.com/sql-tutorial/sql-window-functions/)
- [Accenture Data Engineer Interview - Real Question](https://github.com/DataExpert-io/data-engineer-handbook)
