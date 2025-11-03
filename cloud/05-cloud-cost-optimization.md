# Cloud Cost Optimization & FinOps

**Tags**: #cloud #cost-optimization #finops #reserved-instances #spot #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix, Uber  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min  

---

## TL;DR

**FinOps** = Financial operations. Cloud costs = 30-50% of IT budget. **Optimization**: Reserved Instances (save 30-70%), Spot instances (save 70-90%, risky), auto-scaling (right-size), data transfer optimization (biggest hidden cost). **Reality**: Most companies waste 30-40% of cloud spend. Opportunity: Save $100k-$1M/month with smart decisions.

---

## Concepto Core

- **Qué es**: FinOps = discipline de optimizar cloud costs (like budgeting)
- **Por qué importa**: Cloud costs explode sin monitoring. $1M/month → $2M/month quickly
- **Principio clave**: Cost visibility + governance + optimization = "magic"

---

## Memory Trick

**"Dinero que gotea"** — Sin FinOps, cloud costs gotean. Con FinOps, se seca el goteo.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Cloud costs = major operational expense. 30-40% typically wasted"

**Paso 2**: "Optimization levers: Reserved Instances, Spot, right-sizing, data transfer"

**Paso 3**: "Savings: 30-70% with RI, 70-90% with Spot (but risky)"

**Paso 4**: "Most hidden cost: Data transfer out ($0.09/GB in AWS)"

---

## Código/Ejemplos

### Part 1: Cost Components

Cloud cost breakdown for typical company
infrastructure = {
"Compute": {
"EC2 (on-demand)": 600_000, # $600k/month
"EMR/Spark clusters": 200_000,
"Lambda/Serverless": 50_000,
},
"Storage": {
"S3": 100_000, # $100k/month
"Database": 150_000,
},
"Networking": {
"Data transfer out": 300_000, # $300k/month (ouch!)
"NAT Gateway": 50_000,
"VPN": 10_000,
},
"Other": {
"Managed services": 100_000,
"Support": 50_000,
}
}

total = sum(sum(v.values() if isinstance(v, dict) else [v]) for v in infrastructure.values())
print(f"Total monthly AWS: ${total:,}")

Total monthly AWS: $1,610,000
Typical without optimization:
- 40% wasted (unused capacity, over-provisioning, data transfer)
- Savings potential: $1.6M × 0.4 = $640k/month
text

---

### Part 2: Reserved Instances (RI)

What: Pay upfront for compute capacity, get 30-70% discount

Pricing (AWS EC2):
┌────────────────┬──────────────┬─────────────┬──────────┐
│ Instance Type │ On-Demand │ 1-Year RI │ Savings │
├────────────────┼──────────────┼─────────────┼──────────┤
│ m5.xlarge │ $0.192/hr │ $0.105/hr │ 45% │
│ m5.2xlarge │ $0.384/hr │ $0.210/hr │ 45% │
│ c5.large │ $0.085/hr │ $0.047/hr │ 45% │
└────────────────┴──────────────┴─────────────┴──────────┘

3-Year RI (deeper discount):
m5.xlarge: $0.078/hr (60% savings!)

Real-world calculation:

Scenario: Production database server, always on

Instance: m5.xlarge

On-demand: $0.192/hr × 730 hours/month = $140.16/month

1-Year RI: $0.105/hr × 730 = $76.65/month

Savings: $63.51/month per instance

For 100 servers:

Monthly savings: $63.51 × 100 = $6,351/month

Annual commitment: $0.105/hr × 8760 hours × 100 = $91,980

Break-even: ~15 months ✓ (worth it)

Strategy:

Baseline: Identify compute that runs 24/7 (permanent production)

RI recommendation: 60-70% of compute

Remaining 30-40%: Spot (bursty), on-demand (flexible)

text

---

### Part 3: Spot Instances (Risky but Cheap)

What: Unused EC2 capacity, buy at 70-90% discount, but AWS can terminate anytime

Pricing (AWS):

On-demand: $0.192/hr

Spot: $0.057/hr (70% savings!)

But: Can terminate with 2-min warning

When to use:
✓ Batch jobs (can retry)
✓ Spark clusters (can re-run)
✓ Non-critical workloads

❌ Don't use:
✗ Databases (cannot tolerate interruption)
✗ Real-time services (SLA critical)

Real-world: Spot fleet

Spark cluster for ETL (batch job, can retry)
Cluster composition:
├─ Master: 1× m5.xlarge on-demand ($0.192/hr = $141/month)
├─ Workers: 10× c5.2xlarge on-demand ($0.34/hr each)
└─ vs Workers: 10× c5.2xlarge Spot ($0.10/hr each) = 70% savings!

Cost:

All on-demand: ($0.192 + 10×$0.34) × 730 = $2,629/month

Mixed (RI master + Spot workers):

Master: $0.105 × 730 = $76.65/month (RI)

Workers: $0.10 × 10 × 730 = $730/month (Spot)

Total: $806.65/month

Savings: $1,822/month (69% reduction!)

Risk: 2-3x per week, cluster auto-replaces with new Spot instances
Acceptable for batch ETL (job retries anyway)

text

---

### Part 4: Data Transfer Costs (Hidden Killer)

Most expensive mistake: Not optimizing data transfer!

AWS Data Transfer Pricing:
┌─────────────────────────────────────────┬───────────┐
│ Data Transfer OUT (internet) │ $0.09/GB │
├─────────────────────────────────────────┼───────────┤
│ Inter-AZ (same region) │ $0.01/GB │
├─────────────────────────────────────────┼───────────┤
│ Inter-region │ $0.02/GB │
├─────────────────────────────────────────┼───────────┤
│ To internet (expensive!) │ $0.09/GB │
└─────────────────────────────────────────┴───────────┘

Real-world: Exporting data for customers

Scenario: Analytics company exports 1TB/day to customers

S3 → CloudFront (CDN) → Customer

Transfer: 1TB × 30 days × $0.09/GB = 30TB × $0.09/GB = $2,700/month

Optimization 1: Compress first

Parquet + gzip: 90% compression

1TB → 100GB × 30 × $0.09 = $270/month

Savings: $2,430/month (90%)!

Optimization 2: Use CloudFront (CDN caching)

Repeat downloads (same file): Cached, $0.085/GB (cheaper)

Savings: Additional 5-10%

Optimization 3: Keep data in AWS ecosystem

Don't export, let customer query via Athena/BigQuery Federation

Savings: 100% (no transfer!)

Optimization 4: Regional placement

If customers in EU, EU data center

Inter-region transfer: $0.02/GB vs $0.09/GB

5x cheaper!

Total Optimization Example:
Without: 1TB × 30 × $0.09 = $2,700/month
With: 100GB × 30 × $0.085 (CDN) + internal queries = $255/month
Savings: $2,445/month (91%!)

text

---

### Part 5: Right-Sizing & Auto-Scaling

Problem: Over-provisioning (buy bigger than needed)

Example: Database server

Current: r5.4xlarge (16GB RAM, 12 vCPU, $2.688/hr = $1,962/month)
Usage: Peak 2GB RAM, 1 vCPU (only 12.5% utilized!)

Optimization: Downsize to r5.large (16GB RAM, 2 vCPU, $1.008/hr = $736/month)
Savings: $1,226/month (62% reduction)

Auto-scaling example (Spark cluster):

Without auto-scaling:

50 nodes always running ($5,000/month)

Typical load: 10 nodes

Waste: 40 nodes idle ($4,000/month)

With auto-scaling:

Min: 2 nodes (always on)

Max: 50 nodes (scale for peaks)

Average: 15 nodes during business hours, 2 at night

Cost: $1,500/month (70% savings!)

Implementation (AWS):
autoscaling_group = {
"min_size": 2,
"max_size": 50,
"desired_capacity": 10,
"scale_up_threshold": CPU > 70%,
"scale_down_threshold": CPU < 20%,
"cooldown_period": 300 # Wait 5 min before scaling again
}

text

---

### Part 6: Monitoring & Governance

Cost monitoring dashboard
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')

Get daily costs for last 30 days
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

Parse and alert if over budget
daily_costs = {}
for result in response['ResultsByTime']:
date = result['TimePeriod']['Start']
for group in result['Groups']:
service = group['Keys']
cost = float(group['Metrics']['UnblendedCost']['Amount'])
daily_costs[service] = daily_costs.get(service, 0) + cost

Alert if EC2 > budget
if daily_costs.get('EC2', 0) > 100:
alert(f"EC2 cost spike: ${daily_costs['EC2']}")

Print breakdown
for service, cost in sorted(daily_costs.items(), key=lambda x: x, reverse=True):
print(f"{service}: ${cost:.2f}")

text

---

## Real-World: Cutting Cloud Costs by 40%

Company: E-commerce startup
Current spend: $500k/month
Problem: Unsustainable, burning cash

Audit findings:

On-demand compute (should use RI): $200k/month
→ Convert to RI: Save $90k/month (45%)

Spot opportunities: $100k EMR clusters for batch
→ Use Spot: Save $70k/month (70%)

Data transfer bloat: $50k/month
→ Compress + optimize: Save $40k/month (80%)

Unused resources: $80k/month (idle databases, stopped clusters)
→ Delete: Save $80k/month (100%)

Right-sizing: Over-provisioned servers: $70k/month
→ Downsize: Save $30k/month (43%)

Total savings: $90 + $70 + $40 + $80 + $30 = $310k/month (62%)
New spend: $500k - $310k = $190k/month

Timeline:

Month 1: Quick wins (delete unused, compress data): $100k saved

Month 2: RI commitments (1-year): $150k saved

Month 3: Architecture changes (Spot, right-sizing): $310k saved

Result:

3 months to optimize

Savings: $310k/month ongoing

1-year savings: $3.7M

Engineering cost: $50k (hire contractor)

ROI: 74x in year 1!

text

---

## FinOps Best Practices

Visibility
└─ Cost allocation tags (team, project, environment)
└─ Daily cost reports (email or Slack)
└─ Budget alerts (if spending > threshold)

Governance
└─ RI commitments (require CFO approval)
└─ Spot instances (whitelist non-critical workloads)
└─ Cost limits per environment (dev: $1k, prod: $100k)

Optimization
└─ Monthly cost review (look for trends)
└─ Quarterly right-sizing (adjust compute)
└─ Annual RI planning (forecast next year needs)

Culture
└─ Engineers own costs (not just OPs)
└─ Rewards for cost savings (team bonus)
└─ Penalties for waste (budget owners responsible)

text

---

## Errores Comunes en Entrevista

- **Error**: "RI es siempre mejor" → **Solución**: Solo si workload es predictable/permanent (60-70% of compute)

- **Error**: Usando Spot para critical workloads → **Solución**: Spot terminations = SLA breach

- **Error**: Ignoring data transfer costs → **Solución**: Often 30% of bill, highly optimizable

- **Error**: No tagging for cost allocation → **Solución**: Cannot optimize what you don't measure

---

## Preguntas de Seguimiento

1. **"¿Cuánta savings realistically?"**
   - 30-40% typical (RI + right-sizing + data transfer)
   - 60%+ if very wasteful initially

2. **"¿RI vs Savings Plans?"**
   - RI: Fixed instance, 60-70% savings
   - Savings Plans: Flexible (any instance type), 50% savings
   - RI is better if you know exact needs

3. **"¿Cómo manejas cost optimization en arquitectura?"**
   - Design for cost from start (serverless, spot-friendly)
   - vs Optimize existing (hard retrofits)

4. **"¿Multi-cloud cost comparison?"**
   - Usually: Primary cloud (negotiated rates) + secondary (cheaper for specific service)
   - Data transfer between clouds = major cost

---

## References

- [AWS Cost Optimization - Best Practices](https://docs.aws.amazon.com/cost-management/latest/userguide/cost-optimization.html)
- [GCP Pricing Calculator](https://cloud.google.com/pricing/calculator)
- [Azure Cost Management](https://azure.microsoft.com/en-us/products/cost-management/)
- [FinOps Foundation](https://www.finops.org/)

