# Pipeline Orchestration para Data Engineering

## ¿Por qué Orchestration?

Data pipelines = múltiples tasks (extract, transform, load). Orchestration = scheduling + dependency management + error handling + monitoring.

Sin orchestration: Manual scheduling (cron) = frágil. Con orchestration: Automated, resilient, observable.

## Temas Cubiertos

1. **Apache Airflow** — DAGs, tasks, scheduling, error handling
2. **dbt (data build tool)** — SQL transformations, testing, lineage
3. **Workflow Patterns** — SLAs, backfills, retry strategies
4. **Monitoring & Alerting** — Task failures, SLA violations

## Recomendación de Estudio

**Para Mid devs:**
- Understand DAGs + tasks in Airflow
- Know when to use dbt vs custom SQL

**For Seniors:**
- Domina error handling + retry strategies
- Optimize task dependency graphs
- Design for scalability + maintainability

