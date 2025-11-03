# Optimización de Costos en la Nube & FinOps

**Etiquetas**: #cloud #cost-optimization #finops #reserved-instances #spot #real-interview
**Empresas**: Amazon, Google, Meta, Netflix, Uber
**Dificultad**: Intermedio
**Tiempo estimado**: 20 min

---

## TL;DR

**FinOps** = Operaciones Financieras. Los costos en la nube = 30-50% del presupuesto de IT. **Optimización**: Reserved Instances (ahorran 30-70%), Spot instances (ahorran 70-90%, riesgosas), auto-scaling (ajusta el tamaño correctamente), optimización de transferencia de datos (mayor costo oculto). **Realidad**: La mayoría de empresas desperdician 30-40% del gasto en la nube. **Oportunidad**: Ahorrar $100k-$1M/mes con decisiones inteligentes.

---

## Concepto Core

- **Qué es**: FinOps = disciplina para optimizar costos en la nube (como hacer presupuestos)
- **Por qué importa**: Sin monitoreo, los costos en la nube explotan. $1M/mes → $2M/mes rápidamente
- **Principio clave**: Visibilidad de costos + gobernanza + optimización = "magia"

---

## Truco de Memoria

**"Dinero que gotea"** — Sin FinOps, los costos gotean. Con FinOps, se seca el goteo.

---

## Cómo Explicarlo en una Entrevista

**Paso 1**: "Los costos en la nube = mayor gasto operacional. Típicamente se desperdicia 30-40%"

**Paso 2**: "Palancas de optimización: Reserved Instances, Spot, ajuste de tamaño, transferencia de datos"

**Paso 3**: "Ahorros: 30-70% con RI, 70-90% con Spot (pero riesgoso)"

**Paso 4**: "Mayor costo oculto: Transferencia de datos ($0.09/GB en AWS)"

---

## Código y Ejemplos

### Parte 1: Componentes de Costos

Desglose de costos típico en la nube para una empresa:

```python
infraestructura = {
    "Compute": {
        "EC2 (on-demand)": 600_000,  # $600k/mes
        "EMR/Spark clusters": 200_000,
        "Lambda/Serverless": 50_000,
    },
    "Storage": {
        "S3": 100_000,  # $100k/mes
        "Database": 150_000,
    },
    "Networking": {
        "Data transfer out": 300_000,  # $300k/mes (¡ay!)
        "NAT Gateway": 50_000,
        "VPN": 10_000,
    },
    "Other": {
        "Managed services": 100_000,
        "Support": 50_000,
    }
}

total = sum(sum(v.values() if isinstance(v, dict) else [v]) for v in infraestructura.values())
print(f"Total mensual en AWS: ${total:,}")

# Total mensual en AWS: $1,610,000
# Típico sin optimización:
# - 40% desperdiciado (capacidad sin usar, over-provisioning, transferencia de datos)
# - Potencial de ahorro: $1.6M × 0.4 = $640k/mes
```

---

### Parte 2: Reserved Instances (RI)

**Qué es**: Paga por adelantado la capacidad de cómputo, obtén 30-70% de descuento.

**Precios (AWS EC2)**:

```text
┌────────────────┬──────────────┬─────────────┬──────────┐
│ Tipo de Inst.  │ On-Demand    │ 1-Year RI   │ Ahorro   │
├────────────────┼──────────────┼─────────────┼──────────┤
│ m5.xlarge      │ $0.192/hr    │ $0.105/hr   │ 45%      │
│ m5.2xlarge     │ $0.384/hr    │ $0.210/hr   │ 45%      │
│ c5.large       │ $0.085/hr    │ $0.047/hr   │ 45%      │
└────────────────┴──────────────┴─────────────┴──────────┘

3-Year RI (descuento más profundo):
m5.xlarge: $0.078/hr (¡60% de ahorro!)
```

**Cálculo real del mundo**:

```python
# Escenario: Servidor de base de datos en producción, siempre encendido
instancia = "m5.xlarge"

# On-demand
costo_on_demand = 0.192 * 730  # $140.16/mes

# 1-Year RI
costo_ri = 0.105 * 730  # $76.65/mes

# Ahorro
ahorro = costo_on_demand - costo_ri  # $63.51/mes por instancia

# Para 100 servidores
ahorro_total = ahorro * 100  # $6,351/mes

# Compromiso anual
compromiso_anual = 0.105 * 8760 * 100  # $91,980

# Punto de equilibrio: ~15 meses ✓ (vale la pena)

# Estrategia:
# - Baseline: Identifica el cómputo que corre 24/7 (producción permanente)
# - Recomendación RI: 60-70% del cómputo total
# - Restante 30-40%: Spot (intermitente), on-demand (flexible)
```

---

### Parte 3: Spot Instances (Riesgosas pero Baratas)

**Qué es**: Capacidad EC2 sin usar, compra a 70-90% descuento, pero AWS puede terminarla en cualquier momento.

**Precios (AWS)**:

```text
On-demand: $0.192/hr
Spot: $0.057/hr (¡70% de ahorro!)
Pero: Puede terminar con aviso de 2 minutos
```

**Cuándo usar**:

- ✓ Trabajos batch (pueden reintentarse)
- ✓ Clusters Spark (pueden re-ejecutarse)
- ✓ Cargas no críticas

**Cuándo NO usar**:

- ✗ Bases de datos (no toleran interrupciones)
- ✗ Servicios en tiempo real (SLA crítico)

**Ejemplo real: Flota Spot**:

```text
Cluster Spark para ETL (trabajo batch, puede reintentar)

Composición:
├─ Master: 1× m5.xlarge on-demand ($0.192/hr = $141/mes)
├─ Workers: 10× c5.2xlarge on-demand ($0.34/hr cada)
└─ vs Workers: 10× c5.2xlarge Spot ($0.10/hr cada) = ¡70% ahorro!

Costo:

Todo on-demand: ($0.192 + 10×$0.34) × 730 = $2,629/mes

Mixto (RI master + Spot workers):
├─ Master: $0.105 × 730 = $76.65/mes (RI)
├─ Workers: $0.10 × 10 × 730 = $730/mes (Spot)
└─ Total: $806.65/mes

Ahorro: $1,822/mes (¡69% de reducción!)

Riesgo: 2-3 veces por semana, el cluster se auto-reemplaza con nuevas instancias Spot.
Aceptable para ETL batch (el trabajo se reintenta de todas formas)
```

---

### Parte 4: Costos de Transferencia de Datos (Asesino Oculto)

**Error más costoso**: ¡No optimizar la transferencia de datos!

**Precios de Transferencia de Datos en AWS**:

```text
┌─────────────────────────────────────────┬───────────┐
│ Tipo de Transferencia                   │ Precio    │
├─────────────────────────────────────────┼───────────┤
│ Data Transfer OUT (internet)            │ $0.09/GB  │
│ Inter-AZ (misma región)                 │ $0.01/GB  │
│ Inter-región                            │ $0.02/GB  │
│ A internet (¡caro!)                     │ $0.09/GB  │
└─────────────────────────────────────────┴───────────┘
```

**Ejemplo real: Exportar datos para clientes**:

```python
# Escenario: Empresa de analítica exporta 1TB/día a clientes

# S3 → CloudFront (CDN) → Cliente
transferencia = 1  # TB

# Costo sin optimización
costo_sin_opt = 1 * 30 * 1024 * 0.09  # 30TB × $0.09/GB = $2,700/mes

# Optimización 1: Comprime primero
# Parquet + gzip: 90% de compresión
# 1TB → 100GB
costo_comprimido = 100 * 30 * 0.09  # 3TB × $0.09/GB = $270/mes
ahorro_1 = 2_700 - 270  # $2,430/mes (¡90%!)

# Optimización 2: Usa CloudFront (caché CDN)
# Descargas repetidas (mismo archivo): cachadas, $0.085/GB (más barato)
# Ahorro adicional: 5-10%

# Optimización 3: Mantén datos dentro del ecosistema AWS
# No exportes, deja que el cliente consulte vía Athena/BigQuery Federation
# Ahorro: 100% (¡sin transferencia!)

# Optimización 4: Coloca datos regionalmente
# Si clientes en EU, centro de datos EU
# Transferencia inter-región: $0.02/GB vs $0.09/GB
# ¡5x más barato!

# Ejemplo de optimización total:
sin_optimizar = 1 * 30 * 1024 * 0.09  # $2,700/mes
optimizado = 100 * 30 * 0.085  # $255/mes (CDN + comprimido)
ahorro_total = sin_optimizar - optimizado  # $2,445/mes (¡91%!)
```

---

### Parte 5: Ajuste de Tamaño & Auto-Scaling

**Problema**: Over-provisioning (compra más grande de lo necesario).

**Ejemplo: Servidor de base de datos**:

```python
# Actual: r5.4xlarge (16GB RAM, 12 vCPU, $2.688/hr = $1,962/mes)
# Uso: Pico 2GB RAM, 1 vCPU (¡solo 12.5% utilizado!)

# Optimización: Reduce a r5.large (16GB RAM, 2 vCPU, $1.008/hr = $736/mes)
# Ahorro: $1,226/mes (¡62% de reducción!)
```

**Ejemplo: Auto-scaling (Cluster Spark)**:

```python
# Sin auto-scaling:
nodos_siempre_corriendo = 50  # $5,000/mes
carga_tipica = 10  # nodos
desperdicio = 40  # nodos ociosos ($4,000/mes)

# Con auto-scaling:
min_nodos = 2  # siempre encendidos
max_nodos = 50  # escala para picos
promedio_dia = 15  # nodos durante horario laboral, 2 en noche

costo_escalado = 1_500  # /mes (¡70% de ahorro!)

# Configuración (AWS):
configuracion_autoscaling = {
    "min_size": 2,
    "max_size": 50,
    "desired_capacity": 10,
    "scale_up_threshold": "CPU > 70%",
    "scale_down_threshold": "CPU < 20%",
    "cooldown_period": 300  # Espera 5 min antes de escalar de nuevo
}
```

---

### Parte 6: Monitoreo y Gobernanza

**Panel de control de costos**:

```python
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')

# Obtén costos diarios de los últimos 30 días
response = ce.get_cost_and_usage(
    TimePeriod={
        'Start': (datetime.now() - timedelta(days=30)).strftime('%Y-%m-%d'),
        'End': datetime.now().strftime('%Y-%m-%d')
    },
    Granularity='DAILY',
    Metrics=['UnblendedCost'],
    GroupBy=[
        {'Type': 'DIMENSION', 'Key': 'SERVICE'},
    ]
)

# Analiza y alerta si supera presupuesto
costos_diarios = {}
for resultado in response['ResultsByTime']:
    fecha = resultado['TimePeriod']['Start']
    for grupo in resultado['Groups']:
        servicio = grupo['Keys'][0]
        costo = float(grupo['Metrics']['UnblendedCost']['Amount'])
        costos_diarios[servicio] = costos_diarios.get(servicio, 0) + costo

# Alerta si EC2 > presupuesto
if costos_diarios.get('EC2', 0) > 100:
    alerta(f"Pico de costo EC2: ${costos_diarios['EC2']}")

# Imprime desglose
for servicio, costo in sorted(costos_diarios.items(), key=lambda x: x[1], reverse=True):
    print(f"{servicio}: ${costo:.2f}")
```

---

## Caso Real: Reducir Costos en la Nube en 40%

**Empresa**: Startup de e-commerce
**Gasto actual**: $500k/mes
**Problema**: Insostenible, quemando dinero

**Hallazgos de auditoría**:

```text
On-demand compute (debería usar RI): $200k/mes
→ Convierte a RI: Ahorra $90k/mes (45%)

Oportunidades Spot: $100k en clusters EMR para batch
→ Usa Spot: Ahorra $70k/mes (70%)

Transferencia de datos inflada: $50k/mes
→ Comprime + optimiza: Ahorra $40k/mes (80%)

Recursos sin usar: $80k/mes (bases de datos ociosas, clusters parados)
→ Elimina: Ahorra $80k/mes (100%)

Right-sizing: Servidores over-provisioned: $70k/mes
→ Reduce tamaño: Ahorra $30k/mes (43%)

Ahorro total: $90 + $70 + $40 + $80 + $30 = $310k/mes (62%)
Nuevo gasto: $500k - $310k = $190k/mes
```

**Línea de tiempo**:

```text
Mes 1: Quick wins (elimina sin usar, comprime datos): $100k ahorrados
Mes 2: Compromisos RI (1-año): $150k ahorrados
Mes 3: Cambios de arquitectura (Spot, right-sizing): $310k ahorrados

Resultado:
├─ 3 meses para optimizar
├─ Ahorro: $310k/mes en curso
├─ Ahorro 1-año: $3.7M
├─ Costo de ingeniería: $50k (contractor)
└─ ROI: 74x en año 1!
```

---

## Mejores Prácticas de FinOps

```text
Visibilidad
└─ Etiquetas de asignación de costos (equipo, proyecto, ambiente)
└─ Reportes diarios de costos (email o Slack)
└─ Alertas de presupuesto (si gasto > límite)

Gobernanza
└─ Compromisos RI (requiere aprobación CFO)
└─ Spot instances (whitelist de cargas no críticas)
└─ Límites de costo por ambiente (dev: $1k, prod: $100k)

Optimización
└─ Revisión mensual de costos (busca tendencias)
└─ Right-sizing trimestral (ajusta cómputo)
└─ Planificación anual de RI (proyecta necesidades del próximo año)

Cultura
└─ Ingenieros dueños de costos (no solo Ops)
└─ Recompensas por ahorros (bonificación de equipo)
└─ Penalizaciones por desperdicio (dueños de presupuesto responsables)
```

---

## Errores Comunes en Entrevista

- **Error**: "RI es siempre mejor" → **Solución**: Solo si la carga es predecible/permanente (60-70% del cómputo)

- **Error**: Usar Spot para cargas críticas → **Solución**: Las terminaciones de Spot = incumplimiento de SLA

- **Error**: Ignorar costos de transferencia de datos → **Solución**: Frecuentemente 30% del presupuesto, altamente optimizable

- **Error**: Sin etiquetado para asignación de costos → **Solución**: No puedes optimizar lo que no mides

---

## Preguntas de Seguimiento

1. **"¿Cuántos ahorros realísticamente?"**
   - 30-40% típico (RI + right-sizing + transferencia de datos)
   - 60%+ si inicialmente hay mucho desperdicio

2. **"¿RI vs Savings Plans?"**
   - RI: Instancia fija, 60-70% de ahorro
   - Savings Plans: Flexible (cualquier tipo de instancia), 50% de ahorro
   - RI es mejor si conoces exactamente lo que necesitas

3. **"¿Cómo manejas optimización de costos en arquitectura?"**
   - Diseña para costo desde el inicio (serverless, amigable con Spot)
   - vs Optimizar lo existente (retrofits difíciles)

4. **"¿Comparación de costos multi-nube?"**
   - Típicamente: Nube primaria (tasas negociadas) + secundaria (más barata para servicio específico)
   - Transferencia de datos entre nubes = costo mayor

---

## Referencias

- [AWS Cost Optimization - Best Practices](https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization.html)
- [GCP Pricing Calculator](https://cloud.google.com/pricing/calculator)
- [Azure Cost Management](https://azure.microsoft.com/en-us/products/cost-management/)
- [FinOps Foundation](https://www.finops.org/)
