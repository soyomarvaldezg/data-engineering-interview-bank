# Pipeline Orchestration para Data Engineering

## ¿Por qué Orchestration?

Data pipelines = múltiples tasks (extract, transform, load).  
Orchestration = scheduling + dependency management + error handling + monitoring.

Sin orchestration: manual scheduling (cron) = frágil.  
Con orchestration: automated, resilient, observable.

## Temas cubiertos

1. **Apache Airflow** — DAGs, tasks, scheduling, error handling
2. **dbt (data build tool)** — SQL transformations, testing, lineage
3. **Workflow Patterns** — SLAs, backfills, retry strategies
4. **Monitoring & Alerting** — task failures, SLA violations

## Recomendación de estudio

**Para mid devs:**

- Entender DAGs + tasks en Airflow
- Saber cuándo usar dbt vs custom SQL

**Para seniors:**

- Dominar error handling + retry strategies
- Optimizar task dependency graphs
- Diseñar para scalability + maintainability
