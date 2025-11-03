# Data Lineage & Governance at Scale

**Tags**: #system-design #data-lineage #governance #compliance #metadata #real-interview  
**Empresas**: Amazon, Google, Meta, Goldman Sachs, JPMorgan  
**Dificultad**: Senior  
**Tiempo estimado**: 25 min  

---

## TL;DR

**Data Lineage** = rastrear dónde viene cada dato (source) y dónde va (target). **Governance** = políticas de acceso, ownership, quality, compliance. Problema: 100+ tables, 50+ pipelines → ¿quién owner? ¿de dónde vino? ¿es compliant? Soluciones: Metadata platform (Apache Atlas, Collibra, Alation), data catalog, policy enforcement.

---

## Problema Real

**Escenario:**

- Financial company (regulada por SEC)
- 500+ tables, 200+ pipelines, 100+ data owners
- Compliance requirement: Audit trail (dónde cada dato, quién accede)
- Problema 1: Data scientist crea query, 6 meses después se pregunta "¿de dónde vino este dato?"
- Problema 2: GDPR: "Delete all data for customer X" → ¿cuáles tablas tienen X?
- Problema 3: "Why did analytics change?" → ¿qué pipeline cambió?

**Preguntas:**

1. ¿Cómo rastrreas lineage end-to-end?
2. ¿Cómo implementas data catalog?
3. ¿Quién es "owner" de una tabla?
4. ¿Cómo enforces compliance (GDPR, HIPAA)?
5. ¿Cómo escalas a 1000+ tables?

---

## Solución 1: Data Lineage

### Concepto

Lineage = grafo de: Source → Transforms → Target

Ejemplo:

text
database.customers → [ETL Job 1]  → warehouse.dim_customer
                                  ↓
                              [Spark Transform]
                                  ↓
                    warehouse.fact_customer_summary
                                  ↓
                              [Aggregation]
                                  ↓
                          dashboard.customer_metrics
text

### Implementación: Metadata Tracking

Pseudocódigo: Capturar lineage en cada pipeline
class LineageTracker:

text
def __init__(self, pipeline_name):
    self.pipeline_name = pipeline_name
    self.lineage = {
        "name": pipeline_name,
        "owner": "data-engineering@company.com",
        "sources": [],
        "targets": [],
        "transformations": []
    }

def add_source(self, source_table, database):
    """Registra tabla de entrada"""
    self.lineage["sources"].append({
        "table": source_table,
        "database": database,
        "type": "source"
    })

def add_transformation(self, step_name, operation):
    """Registra cada step del proceso"""
    self.lineage["transformations"].append({
        "step": step_name,
        "operation": operation,
        "timestamp": datetime.now()
    })

def add_target(self, target_table, database):
    """Registra tabla de salida"""
    self.lineage["targets"].append({
        "table": target_table,
        "database": database,
        "type": "target"
    })

def save_lineage(self, metadata_store):
    """Guarda en metadata platform"""
    metadata_store.insert(self.lineage)
===== USAGE =====
tracker = LineageTracker("daily_customer_pipeline")

Etapa 1: Extract
tracker.add_source("customers", "postgres")
tracker.add_source("orders", "postgres")

Etapa 2: Transform
tracker.add_transformation("deduplicate_customers", "SELECT DISTINCT * ...")
tracker.add_transformation("join_with_orders", "SELECT c.* FROM customers c JOIN orders o ...")
tracker.add_transformation("aggregate_daily", "GROUP BY customer_id, DATE(date)")

Etapa 3: Load
tracker.add_target("dim_customer", "redshift")
tracker.add_target("fact_customer_daily", "redshift")

Guardar
tracker.save_lineage(metadata_platform)

text

---

### Lineage Visualization

┌────────────────────┐
│ postgres.customers │
└─────────┬──────────┘
│
▼
[ETL Pipeline]
│
┌─────┴─────────┐
│ │
▼ ▼
┌──────────┐ ┌──────────────┐
│dim_cust │ │fact_daily │
└─────┬────┘ └──────┬───────┘
│ │
└───────┬───────┘
│
[dbt Transform]
│
▼
┌──────────────┐
│analytics_tbl │
└──────┬───────┘
│
┌──────┴──────┐
│ │
▼ ▼
[Dashboard] [Report]

text

---

## Solución 2: Data Catalog & Metadata

### Estructura

Metadata Platform Schema
table:
name: customers
database: postgres
owner: alice@company.com

description: "Customer master table - source of truth"

columns:
- name: customer_id
type: INT
description: "Unique customer identifier"
pii: true # Personal Identifiable Info

text
- name: email
  type: STRING
  description: "Customer email address"
  pii: true
  
- name: age
  type: INT
  description: "Age at signup"
  pii: false
lineage:
upstream: [] # No tiene source
downstream:
- table: dim_customer
pipeline: daily_etl_job

quality_metrics:
null_percentage:
email: 0.1%
age: 1.2%
duplicate_count: 0

access_control:
readers: ["analytics-team", "finance-team"]
writers: ["data-engineering"]
pii_access: ["compliance-officer"]

compliance:
gdpr: true
hipaa: false
pci: false
retention_days: 365

text

---

### Data Catalog Platform (Example: Alation)

┌─────────────────────────────────────────┐
│ DATA CATALOG PLATFORM │
├─────────────────────────────────────────┤
│ ├─ Data Dictionary │
│ │ └─ Every table/column documented │
│ │ │
│ ├─ Business Glossary │
│ │ └─ Terms (Customer, Revenue, etc) │
│ │ │
│ ├─ Lineage Graph │
│ │ └─ Source → Transform → Target │
│ │ │
│ ├─ Data Governance │
│ │ └─ Owner, PII, Compliance │
│ │ │
│ └─ Access Logs │
│ └─ Who accessed what, when │
└─────────────────────────────────────────┘

text

---

## Solución 3: Access Control & Governance

### Role-Based Access Control (RBAC)

roles:
analytics_engineer:
permissions:
- SELECT on warehouse.*
- CREATE TEMPORARY tables
- Cannot DELETE or TRUNCATE

data_scientist:
permissions:
- SELECT on warehouse.analytics_*
- Cannot access raw_* (raw data is private)

compliance_officer:
permissions:
- SELECT on ALL tables (including PII)
- Cannot WRITE
- Full audit log access

pii_processor:
permissions:
- SELECT on PII-marked columns
- WRITE only to approved tables

text

---

### Policy Enforcement: Apache Ranger

<!-- Ranger Policy: Block non-PII-processors from accessing emails --> <policy> <name>protect-pii-emails</name> <resources> <resource> <type>database</type> <value>warehouse</value> </resource> <resource> <type>table</type> <value>*</value> </resource> <resource> <type>column</type> <value>email, phone, ssn</value> <!-- PII columns --> </resource> </resources> <policyItems> <policyItem> <accesses> <permission>select</permission> </accesses> <users>pii-processor, compliance-officer</users> <effect>Allow</effect> </policyItem>
text
<policyItem>
  <accesses>
    <permission>select</permission>
  </accesses>
  <users>*</users>  <!-- Everyone else -->
  <effect>Deny</effect>
</policyItem>
</policyItems> </policy> ```
Solución 4: Compliance & Audit Trail
GDPR: Delete Customer Data
text
class GDPRCompliance:
    
    def delete_customer_data(self, customer_id):
        """Delete all traces de customer (GDPR right to be forgotten)"""
        
        # 1. Query lineage: Qué tablas tienen este customer?
        affected_tables = self.query_lineage(
            "Find all tables containing customer_id = ?",
            customer_id
        )
        # Result: [customers, orders, payments, loyalty, marketing_events, ...]
        
        # 2. Delete from cada tabla
        for table in affected_tables:
            db.execute(f"DELETE FROM {table} WHERE customer_id = ?", customer_id)
        
        # 3. Log deletion (audit trail)
        self.audit_log.insert({
            "action": "GDPR_DELETE",
            "customer_id": customer_id,
            "tables_affected": affected_tables,
            "timestamp": datetime.now(),
            "requested_by": "gdpr-processor@company.com"
        })
        
        # 4. Verify deletion
        for table in affected_tables:
            remaining = db.query(f"SELECT COUNT(*) FROM {table} WHERE customer_id = ?", customer_id)
            if remaining > 0:
                alert(f"Deletion verification failed for {table}")

# ===== USAGE =====
compliance = GDPRCompliance()
compliance.delete_customer_data(customer_id=12345)
Audit Trail Example
text
{
  "audit_log": [
    {
      "timestamp": "2024-01-15 10:30:00",
      "user": "alice@company.com",
      "action": "SELECT",
      "table": "customers",
      "rows_returned": 50000,
      "ip": "192.168.1.100"
    },
    {
      "timestamp": "2024-01-15 10:31:00",
      "user": "bob@company.com",
      "action": "EXPORT",
      "table": "customers",
      "format": "CSV",
      "destination": "s3://exports/",
      "rows_exported": 50000,
      "pii_present": true
    },
    {
      "timestamp": "2024-01-15 10:45:00",
      "user": "system",
      "action": "SCHEMA_CHANGE",
      "table": "customers",
      "change": "ADD COLUMN marketing_opt_in BOOLEAN",
      "reason": "GDPR compliance"
    }
  ]
}
Solución 5: Metadata at Scale
Centralized Metadata Store
text
┌──────────────────────────────────────┐
│ METADATA PLATFORM (Apache Atlas)     │
├──────────────────────────────────────┤
│                                      │
│ Sources:                             │
│  ├─ Hive Metastore (push metadata)   │
│  ├─ Spark lineage (push)             │
│  ├─ dbt manifest (push)              │
│  ├─ Airflow DAGs (push)              │
│  └─ Manual (UI)                      │
│                                      │
│ Storage:                             │
│  ├─ Tables metadata                  │
│  ├─ Columns metadata                 │
│  ├─ Lineage graph                    │
│  ├─ Access logs                      │
│  └─ Quality metrics                  │
│                                      │
│ APIs:                                │
│  ├─ REST API (search, update)        │
│  ├─ Python SDK (scripting)           │
│  └─ UI (browsing)                    │
│                                      │
└──────────────────────────────────────┘
Integration: dbt + Atlas
text
# dbt_project.yml
name: 'analytics'
version: '1.0.0'

meta:
  owner: 'analytics-team'
  tier: 'gold'  # Data quality tier
  
models:
  - name: customers
    meta:
      owner: 'alice@company.com'
      description: 'Customer master table'
      pii: true
      retention_days: 365
      
    columns:
      - name: customer_id
        meta:
          pii: false
          sensitive: false
          
      - name: email
        meta:
          pii: true
          sensitive: true
          masking_required: true
dbt generates metadata → Atlas ingests → Catalog updated

Errores Comunes en Entrevista
Error: "Lineage es opcional" → Solución: CRITICAL para compliance, debugging, impact analysis

Error: No tracking access (quién accede?) → Solución: Audit logs son mandatory para GDPR/HIPAA

Error: Manual metadata (wiki, docs) → Solución: Desincroniza rápido. Automated > manual

Error: No enforcement de policies → Solución: Ranger/Sentry enforce, no "trust"

Preguntas de Seguimiento
"¿Cómo debuggeas si data cambia unexpectedly?"

Lineage: "qué pipeline escribe esto?"

Audit logs: "quién/cuándo cambió?"

Schema versioning: "qué cambió en estructura?"

"¿Cómo manejas data sprawl (datos esparcidos everywhere)?"

Catalog: Descubrir todos los datasets

Governance: Consolidar duplicados

Lineage: Vs sources of truth

"¿Cost de metadata platform?"

Platform: $10-50k/año (Alation, Collibra)

vs Manual effort: 100+ hours/year → $$$ wastage

"¿Cómo adaptas governance según industry?"

Finance: GDPR, PCI-DSS (strict)

Healthcare: HIPAA (very strict)

E-commerce: GDPR (moderate)

Real-World: Compliance Report
text
class ComplianceReport:
    
    def generate_gdpr_report(self, date_range):
        """Generate GDPR compliance report"""
        
        report = {
            "period": date_range,
            "pii_datasets": self.count_pii_datasets(),
            "data_requests_processed": self.count_deletion_requests(),
            "access_violations": self.count_access_violations(),
            "retention_policy_violations": self.count_retention_violations(),
            "audit_trail": self.get_audit_log(date_range)
        }
        
        return report

# ===== OUTPUT =====
{
  "period": "2024-01-01 to 2024-01-31",
  "pii_datasets": 125,
  "data_requests_processed": 42,
  "access_violations": 2,  # Alert: need investigation
  "retention_policy_violations": 0,
  "audit_trail": [...]
}
References
Apache Atlas - Data Governance

Apache Ranger - Access Control

dbt Metadata - dbt Docs

GDPR Compliance - EU Regulation

