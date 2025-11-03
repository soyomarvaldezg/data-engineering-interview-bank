# Multi-Cloud Strategy & Trade-offs

**Tags**: #cloud #strategy #aws-vs-gcp-vs-azure #trade-offs #architecture #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix, Uber  
**Dificultad**: Senior  
**Tiempo estimado**: 18 min

---

## TL;DR

**Multi-cloud** = usar múltiples providers. **Por qué**: Vendor lock-in avoidance, negotiating power, regional coverage, best-of-breed services. **Trade-off**: Complexity, operational overhead, data movement costs. **Recomendación**: Pick 1 primary (AWS/GCP), add 2nd solo si specific need. Avoid 3+ clouds.

---

## Concepto Core

- **Qué es**: Usar >1 cloud provider en la misma arquitectura
- **Por qué importa**: Strategic decision. Afecta costo, complejidad, operaciones por años
- **Principio clave**: Costs de abstraction >> vendor lock-in costs (para mayoría)

---

## Cuándo Multi-Cloud

### ✅ Razones VÁLIDAS

Vendor Lock-in Avoidance
└─ Si quieres "portabilidad", esto es caro
└─ Realidad: Pocas compañías logran true portability

Best-of-Breed Services
└─ BigQuery (GCP) es mejor que Redshift (AWS)
└─ SageMaker (AWS) es mejor que Azure ML
└─ Pero: Operacional overhead es alto

Regional Coverage
└─ Provider A no tiene región X, Provider B sí
└─ Ejemplo: AWS no en China (GCP sí)
└─ Legit reason para 2-cloud

Negotiating Power
└─ "We use both AWS and GCP, reduce Azure prices or we switch"
└─ Real en deals >$1M/year

Risk Mitigation
└─ Don't want single provider outage affecting everything
└─ Multi-AZ/Multi-region > multi-cloud (cheaper)

text

### ❌ Razones INVÁLIDAS

"We use what each team prefers"
└─ Road to chaos
└─ Nightmare for operations

"Portability is important"
└─ False: Cost of abstraction >> value
└─ 99% of companies stay with primary cloud once chosen

"Future-proofing"
└─ Technology changes too fast to predict
└─ Better: Choose best today, migrate later if needed

text

---

## Real-World: Multi-Cloud Companies

### Company A: AWS + GCP (Best-of-Breed)

Scenario: Airbnb (hypothetically)

Primary: AWS
├─ EC2 (compute)
├─ S3 (storage)
├─ DynamoDB (NoSQL)
└─ Lambda (serverless)

Secondary: GCP
├─ BigQuery (warehouse, better than Redshift)
├─ Dataflow (ML pipeline)
└─ Vertex AI (model serving)

Data Flow:
Airbnb bookings (AWS) → ETL (AWS Glue) → S3 → BigQuery (GCP) → ML (Vertex AI) → Predictions → AWS Lambda

Integration:

Data transfer AWS → GCP: ~$0.02/GB (costly for high volume)

BigQuery federated query into AWS: Complex setup

Orchestration: Airflow (multi-cloud aware)

Cost Implications:

AWS: $2M/month

GCP: $500k/month (just warehouse + ML)

Data transfer: $200k/month (ouch!)

Ops/Engineering: +20% effort (multi-cloud complexity)

Was it worth?

BigQuery saved $300k/month vs Redshift

But data transfer = $200k/month

Net savings: $100k/month

Engineering cost: -$200k/month

Result: NOT WORTH IT unless BigQuery was >20% better

text

### Company B: AWS + Azure (Enterprise)

Scenario: JPMorgan (on-prem + cloud migration)

Primary: AWS (new cloud workloads)
├─ EC2
├─ S3
├─ EMR
└─ Redshift

Secondary: Azure (existing Microsoft stack)
├─ Active Directory (Identity)
├─ SQL Server databases
└─ Synapse (warehouse for corporate apps)

Why Multi-Cloud?

Existing on-prem SQL Server licenses → keep on Azure

Investment in Azure AD (enterprise identity)

Hybrid cloud easier with Azure (on-prem + cloud)

Not true multi-cloud choice, just migration phases

Cost:

AWS: $1M/month (data workloads)

Azure: $1M/month (corporate apps)

Total: $2M/month

Could consolidate to AWS in 2-3 years (migration plan)

text

---

## Multi-Cloud: Comparison Matrix

| Aspect           | AWS       | GCP         | Azure      |
| ---------------- | --------- | ----------- | ---------- |
| **Market Share** | 32%       | 11%         | 23%        |
| **Warehouse**    | Redshift  | BigQuery ✓  | Synapse    |
| **Streaming**    | Kinesis   | Pub/Sub ✓   | Event Hubs |
| **ML/AI**        | SageMaker | Vertex AI ✓ | Azure ML   |
| **Cost**         | Mid       | Low ✓       | Mid        |
| **Ease**         | Low       | High ✓      | Mid        |
| **Enterprise**   | Good      | Developing  | Best ✓     |

**✓ = Best in category**

---

## Multi-Cloud Architecture Patterns

### Pattern 1: Primary + Secondary (Best-of-Breed)

Primary: AWS (everything except warehouse)
Secondary: GCP (just BigQuery)

Architecture:
AWS Data Lake (S3)
↓
Glue/EMR (transform)
↓
S3 (processed)
↓
GCP BigQuery (federated query, or manual load)
↓
Looker/Power BI (dashboard)

Trade-offs:

Best warehouse (BigQuery)

Simpler than full multi-cloud

Data transfer cost

Federated query complexity

Operational overhead (2 systems)

text

### Pattern 2: Geographic Distribution

Region: US-East
└─ Provider: AWS (best coverage)

Region: EU
└─ Provider: AWS (also best coverage, but GCP cheaper in EU)

Region: Asia-Pacific
└─ Provider: GCP (or Azure, region-specific choice)

Architecture:

Each region = different provider based on best choice

Data sync between regions (costly!)

Nightmare for operations

Better Alternative:

Stick to 1 provider, use multi-region/multi-AZ

Cost: Cheaper

Complexity: Lower

text

### Pattern 3: Fallback/Disaster Recovery

Primary: AWS
Failover: GCP (backup, in case AWS region fails)

Architecture:
AWS (active)
└─ Continuous replication to GCP
└─ On AWS outage: Failover to GCP (RTO/RPO trade-offs)

Reality:

Single-cloud multi-region is cheaper/easier

Multi-cloud fallback is expensive ($$$)

Usually not worth it unless extreme HA requirements

text

---

## Cost Comparison: Single vs Multi-Cloud

### Scenario: 1TB warehouse query per day

**Single Cloud: AWS**
Redshift: 16 DC2 nodes × $0.43/hr × 24hr = $165.12/day
Bandwidth out: ~50GB × $0.09 = $4.50/day
Total: ~$170/day (~$5k/month)
Ops: 1 person (familiar with AWS)

text

**Multi-Cloud: AWS + GCP**
AWS:

S3: $20/day

Glue/Lambda: $10/day
Subtotal: $30/day

GCP:

BigQuery: 1TB × $6.25 = $6.25/day
Subtotal: $6.25/day

Data Transfer (AWS → GCP):

~500GB/day × $0.02/GB = $10/day
Subtotal: $10/day

Ops:

AWS expert: 0.5 person

GCP expert: 0.3 person

Integration/Data engineer: 0.5 person

Total: 1.3 people (~$80k/month)

Total Cost: ($30 + $6.25 + $10) × 30 + $80k = $2.4k + $80k = $82.4k/month
vs Single cloud: $5k/month

❌ NOT WORTH IT unless BigQuery is 15x better (it's not)

text

---

## When to Choose Each Cloud

### Choose AWS If:

✓ Enterprise (most adoption)
✓ Need all services (widest ecosystem)
✓ Regional coverage critical
✓ Cost-conscious (competitive pricing)
✓ Hybrid cloud (on-prem + AWS)

Example Companies: Netflix, Airbnb, LinkedIn

text

### Choose GCP If:

✓ Analytics first (BigQuery is king)
✓ ML/AI heavy (Vertex AI, AutoML)
✓ Data science teams
✓ Budget conscious (lowest cost)
✓ Real-time streaming (Pub/Sub)

Example Companies: Spotify, Twitter, Shopify

text

### Choose Azure If:

✓ Enterprise with Microsoft stack
✓ Existing on-prem Windows/SQL Server
✓ Active Directory / hybrid scenarios
✓ Compliance/governance critical (Purview)
✓ Power BI dashboards

Example Companies: JPMorgan, Goldman Sachs, Accenture

text

### Avoid Multi-Cloud Unless:

✓ >$10M/year cloud spend (negotiating power)
✓ Best-of-breed service gap is critical
✓ Geographic/regulatory requirements
✓ True disaster recovery SLA (99.99%+)

Most companies should NOT do multi-cloud.

text

---

## Multi-Cloud Anti-Patterns

### Anti-Pattern 1: "Let Teams Choose"

Team A: AWS
Team B: GCP
Team C: Azure

Result:

No standardization

Cannot share infrastructure

3x ops cost

Nightmare to migrate data

Skills fragmented

Solution: CTO mandate (1-2 clouds max)

text

### Anti-Pattern 2: "Abstraction Layer Over Everything"

Idea: Kubernetes, Terraform, etc to make clouds interchangeable
Reality:

Kubernetes doesn't abstract data layer (most expensive)

Terraform works but loses cloud benefits

Abstraction layer = 20% overhead

Rarely pays off

Better: Pick best cloud for use case, stick with it

text

### Anti-Pattern 3: "We'll Migrate Later"

Decision: Start in GCP (cheap BigQuery)
Idea: "We can migrate to AWS later if needed"

Reality:

After 2-3 years: 100+ systems built on GCP

Migration = $1M+ project, 6+ months

Never happens, you're stuck anyway

Better: Choose wisely upfront, migrations rare/expensive

text

---

## Decision Framework

┌─ Start: Multi-cloud needed?
│ │
│ ├─ Yes: ──→ Do you have specific service gap?
│ │ │
│ │ ├─ Yes (e.g., BigQuery > Redshift) ──→ Primary: AWS, Secondary: GCP
│ │ │ Cost/Benefit: Calculate data transfer
│ │ │
│ │ └─ No ──→ Single cloud is better
│ │
│ └─ No: ──→ Which is best for primary workload?
│ │
│ ├─ Analytics ──→ GCP (BigQuery)
│ ├─ Enterprise ──→ AWS or Azure
│ ├─ ML-heavy ──→ GCP or AWS (SageMaker)
│ └─ Microsoft stack ──→ Azure
│
└─ Final: Make decision, stick 3+ years (migration costs are real)

text

---

## Errores Comunes en Entrevista

- **Error**: "Multi-cloud is always good" → **Solución**: Complexity > benefits para mayoría

- **Error**: "Portability matters" → **Solución**: False. Lock-in cost es low vs abstraction cost

- **Error**: Underestimating data transfer costs → **Solución**: $0.02/GB × 1TB/day = $600/day = $18k/month

- **Error**: Not counting ops overhead → **Solución**: Multi-cloud = +30-50% ops effort

---

## Preguntas de Seguimiento

1. **"¿AWS vs GCP vs Azure para your company?"**
   - Depends on workload, current stack, budget
   - Most: Primary AWS, secondary GCP for analytics

2. **"¿Cuándo migras de cloud?"**
   - Rare (cost too high)
   - Only if business drivers change dramatically

3. **"¿Hybrid cloud vs multi-cloud?"**
   - Hybrid: On-prem + 1 cloud
   - Multi-cloud: 2+ clouds
   - Hybrid is more common (easier)

4. **"¿Kubernetes helps multi-cloud?"**
   - Compute layer: Yes (but only 20% of cost)
   - Data layer: No (still cloud-specific)
   - Net benefit: Limited

---

## References

- [AWS vs GCP vs Azure - Gartner Magic Quadrant](https://www.gartner.com/reviews/market/cloud-infrastructure-platforms)
- [Multi-Cloud Strategy - McKinsey](https://www.mckinsey.com/featured-insights/shared-value/how-to-get-the-most-out-of-your-multi-cloud-environment)
- [Cloud Cost Management - Forrester](https://www.forrester.com/research/topic/cloud-cost-management)
