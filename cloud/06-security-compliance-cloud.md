# Security & Compliance en Cloud

**Tags**: #cloud #security #compliance #gdpr #hipaa #vpc #iam #encryption #real-interview  
**Empresas**: Amazon, Google, Meta, JPMorgan, Goldman Sachs  
**Dificultad**: Senior  
**Tiempo estimado**: 20 min  

---

## TL;DR

**Cloud Security** = shared responsibility model (cloud provider secures infrastructure, you secure data + access). **Key layers**: VPC (network isolation), IAM (access control), encryption (at-rest, in-transit), audit logs. **Compliance**: GDPR (personal data), HIPAA (health data), PCI-DSS (credit cards), SOC 2 (audit). Most companies: GDPR + SOC 2 minimum.

---

## Concepto Core

- **Qué es**: Security = proteger data + infrastructure. Compliance = seguir reglas (GDPR, HIPAA, etc)
- **Por qué importa**: Data breach = $4M average cost + regulatory fines + reputation damage
- **Principio clave**: "Defense in depth" — múltiples layers, no single point of failure

---

## Memory Trick

**"Castillo con murallas"** — VPC = muralla. IAM = guardias (quién entra). Encryption = bóveda (data segura). Audit logs = vigilancia (quién hizo qué).

---

## Cómo explicarlo en entrevista

**Paso 1**: "Cloud security = shared responsibility. Cloud provider = infrastructure, you = data + access"

**Paso 2**: "Layers: VPC (network), IAM (access), encryption (data), audit (logging)"

**Paso 3**: "Compliance = GDPR (EU), HIPAA (health), PCI-DSS (payment), SOC 2 (audit)"

**Paso 4**: "Real-world: VPC isolation, least-privilege IAM, encryption everywhere, audit trails"

---

## Código/Ejemplos

### Part 1: VPC (Network Isolation)

What: Virtual Private Cloud (your own network in cloud)
Why: Isolate resources, control traffic, protect data

Architecture:
┌─────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16 (my private network) │
├─────────────────────────────────────────────┤
│ │
│ Public Subnet 1 (10.0.1.0/24) │
│ ├─ NAT Gateway (for egress) │
│ └─ Bastion Host (SSH jump) │
│ │
│ Private Subnet 1 (10.0.10.0/24) │
│ ├─ Database (no internet access) │
│ └─ Application servers │
│ │
│ Private Subnet 2 (10.0.20.0/24) [us-east-2]
│ └─ Backup database (redundancy) │
│ │
│ VPN Connection (on-prem access) │
│ └─ Site-to-site VPN to head office │
│ │
└─────────────────────────────────────────────┘

Security Groups (firewall per instance):
├─ Web tier: Allow port 443 (HTTPS) from internet
├─ App tier: Allow port 8080 from web tier only
└─ DB tier: Allow port 5432 from app tier only

Network ACLs (firewall per subnet):
├─ Inbound: Allow specific IPs/ports
└─ Outbound: Deny internet (only VPN)

Benefits:
✓ Isolation: Cannot access database from internet
✓ Control: Know exactly what traffic is allowed
✓ Compliance: Network segmentation requirement (HIPAA, PCI)

text

### Part 2: IAM (Identity & Access Management)

What: Who can do what with cloud resources
Why: Least-privilege principle (minimal access needed)

Structure:
┌─────────────────────────────────┐
│ AWS Account (root = never use!) │
├─────────────────────────────────┤
│ Users │
│ ├─ alice@company.com │
│ ├─ bob@company.com │
│ └─ ci-cd-service │
│ │
│ Groups │
│ ├─ DataEngineers │
│ ├─ Analysts │
│ └─ DevOps │
│ │
│ Roles (for services) │
│ ├─ LambdaExecutionRole │
│ ├─ EC2InstanceRole │
│ └─ SparkClusterRole │
│ │
│ Policies (permissions) │
│ ├─ ReadS3 │
│ ├─ WriteRedshift │
│ └─ ManageLambda │
└─────────────────────────────────┘

Least-Privilege Example:

❌ BAD Policy (too permissive):
{
"Effect": "Allow",
"Action": "s3:", # ALL S3 actions
"Resource": "" # ALL S3 buckets
}

Alice can delete entire data lake!
✅ GOOD Policy (least-privilege):
{
"Effect": "Allow",
"Action": [
"s3:GetObject",
"s3:ListBucket"
],
"Resource": [
"arn:aws:s3:::data-lake/raw/*"
]
}

Alice can only READ files in specific bucket
Multi-factor authentication (MFA):
┌─ User enters username + password
├─ AWS asks for 2FA code (phone/app)
└─ Only then: access granted

Temporary credentials (for Lambda/EC2):
┌─ Lambda role: LambdaExecutionRole
├─ Permissions: S3 read, DynamoDB write
└─ Session token (valid 1 hour, expires automatically)

text

### Part 3: Encryption

What: Scrambling data so only authorized users can read
Why: Compliance requirement + security best practice

Types:

At-Rest Encryption (data stored)
├─ S3: Enable default encryption (AES-256)
├─ Database: RDS with encryption enabled
├─ EBS volumes: Encrypted snapshots
└─ Key management: AWS KMS (Key Management Service)

In-Transit Encryption (data moving)
├─ HTTPS (TLS 1.2+)
├─ VPN (encrypted tunnel on-prem to cloud)
├─ Database connections: SSL/TLS required
└─ Lambda to S3: Always HTTPS

Client-Side Encryption (app layer)
├─ Encrypt before uploading to S3
├─ PII masking in application code
└─ Tokenization (replace SSN → token)

Real-world implementation:

AWS S3 with default encryption
aws s3api put-bucket-encryption
--bucket my-data-lake
--server-side-encryption-configuration '{
"Rules": [{
"ApplyServerSideEncryptionByDefault": {
"SSEAlgorithm": "AES256"
}
}]
}'

Database with encryption
aws rds create-db-instance
--db-instance-identifier prod-db
--storage-encrypted
--kms-key-id arn:aws:kms:us-east-1:123456789:key/xxx

Connection string (enforces TLS)
postgresql://user:pass@db.aws.com:5432/mydb?sslmode=require

Performance impact:

At-rest: ~5% overhead (negligible)

In-transit: ~2% overhead (negligible)

Encryption = mandatory (not optional)

text

### Part 4: Audit Logging & Compliance

What: Track who did what, when, where
Why: Detect breaches, compliance requirement, debugging

Audit trails:

AWS CloudTrail (API logging):
┌─ Every AWS API call logged
├─ Who: User/role that made call
├─ What: Action performed
├─ When: Timestamp
├─ Where: Region, IP address
└─ Resource: Which resource affected

Example:
{
"eventName": "CreateDBInstance",
"eventTime": "2024-01-15T10:30:45Z",
"userIdentity": {"principalId": "alice@company.com"},
"sourceIPAddress": "203.0.113.42",
"awsRegion": "us-east-1",
"requestParameters": {
"dBInstanceIdentifier": "prod-db-1",
"storageEncrypted": true
}
}

VPC Flow Logs (network logging):
┌─ Every network packet logged
├─ Source/destination IP
├─ Port, protocol
├─ Accept/reject
└─ Bytes transferred

S3 Access Logs (object logging):
┌─ Every S3 GET/PUT/DELETE logged
├─ Who accessed
├─ Object name
└─ When

CloudWatch Alarms:
├─ Alert if root account used
├─ Alert if non-encrypted data written
├─ Alert if unusual access pattern
└─ Alert if compliance violation

Configuration auditing (AWS Config):
┌─ Track resource configuration changes
├─ Alert if security group modified
├─ Alert if encryption disabled
└─ Alert if resource created in wrong region

Retention policy:
├─ CloudTrail: 7 years (compliance requirement)
├─ VPC Logs: 1 year (operational)
└─ S3 Access: 90 days (storage cost trade-off)

text

### Part 5: Compliance Frameworks

GDPR (General Data Protection Regulation - EU)

What: Protect personal data of EU citizens
Applies: Any company processing EU residents' data

Requirements:
├─ Consent: Get permission before collecting data
├─ Right to Access: User can request their data
├─ Right to Deletion: User can delete their data ("right to be forgotten")
├─ Data Minimization: Collect only what needed
├─ Encryption: At-rest and in-transit
├─ Audit trails: Log who accessed data
└─ Incident reporting: Report breaches within 72h

Implementation in cloud:
┌─ Encryption everywhere (at-rest, in-transit)
├─ VPC + IAM (access control)
├─ CloudTrail logging (audit)
├─ Data retention policies (delete old data)
└─ Incident response plan

HIPAA (Health Insurance Portability and Accountability - USA)

What: Protect health information (PHI - Protected Health Information)
Applies: Healthcare providers, insurers, cloud providers storing health data

Requirements:
├─ Encryption (AES-256 or stronger)
├─ Access controls (authentication, authorization)
├─ Audit logs (track all PHI access)
├─ Secure deletion (don't leave data on decommissioned disks)
├─ Breach notification (notify patients if data leaked)
└─ Business Associate Agreements (BAAs - contracts with vendors)

More strict than GDPR (healthcare is sensitive)

PCI-DSS (Payment Card Industry Data Security Standard - USA)

What: Protect credit card data
Applies: Any company processing credit cards

Requirements:
├─ Network segmentation (card data in separate network)
├─ Encryption (card data always encrypted)
├─ No default passwords (change AWS root password!)
├─ Strong access controls (IAM roles)
├─ Audit logs (track card data access)
├─ Vulnerability scanning (penetration testing)
├─ Incident response plan
└─ Annual compliance audit (external auditor)

Strictest (payment fraud = immediate problem)

SOC 2 (Service Organization Control - USA)

What: Audit of security/compliance of service provider
Applies: B2B SaaS companies, cloud providers

Requirements:
├─ Security (prevent unauthorized access)
├─ Availability (uptime commitments)
├─ Processing integrity (data accurate and complete)
├─ Confidentiality (data not disclosed)
└─ Privacy (personal data handled correctly)

Type 1: Snapshot at a point in time
Type 2: Audit over 6-12 months (preferred by customers)

Cost: $50k-200k per audit (annual)

text

---

## Real-World: Compliance Checklist

Compliance audit checklist
class ComplianceAudit:
def gdpr_audit(self):
"""EU personal data protection"""
checks = {
"Encryption at-rest": self.check_s3_encryption(),
"Encryption in-transit": self.check_tls_enforcement(),
"Access controls": self.check_iam_policies(),
"Audit logs": self.check_cloudtrail_enabled(),
"Data retention": self.check_retention_policies(),
"Incident response": self.check_incident_plan(),
}

text
    if not all(checks.values()):
        return "FAIL: Not GDPR compliant"
    return "PASS: GDPR compliant"

def hipaa_audit(self):
    """Health data protection (stricter than GDPR)"""
    checks = {
        "Encryption": self.check_aes256_encryption(),
        "Network isolation": self.check_vpc_isolation(),
        "Access logging": self.check_hipaa_audit_logs(),
        "BAA signed": self.check_baa_with_aws(),
        "Breach response": self.check_breach_procedure(),
    }
    
    if not all(checks.values()):
        return "FAIL: Not HIPAA compliant"
    return "PASS: HIPAA compliant"

def check_s3_encryption(self):
    # Verify S3 default encryption enabled
    response = s3.get_bucket_encryption(Bucket='my-bucket')
    return 'Rules' in response.get('ServerSideEncryptionConfiguration', {})
Result: Report to auditors (SOC 2 type 2, GDPR, HIPAA)
text

---

## Errores Comunes en Entrevista

- **Error**: "Public cloud no es seguro" → **Solución**: Cloud = más seguro que on-prem (AWS invierte billions)

- **Error**: "Encryption es suficiente" → **Solución**: Encryption is ONE layer. Also need: IAM, VPC, audit logs

- **Error**: "Compliance es IT responsibility" → **Solución**: Everyone (engineers, managers) owns compliance

- **Error**: Storing credentials in code → **Solución**: Use IAM roles (temporary credentials, auto-rotate)

---

## Preguntas de Seguimiento

1. **"¿Cuándo usar VPC vs Security Groups?"**
   - VPC: Network-level isolation (infrastructure)
   - Security Groups: Application-level rules (per instance)
   - Both needed (defense in depth)

2. **"¿Root account - ever use it?"**
   - NO (except AWS account setup)
   - Always use IAM user/role with MFA

3. **"¿On-prem vs Cloud security?"**
   - Cloud: Easier (AWS handles physical security, updates)
   - On-prem: More control (but more work)

4. **"¿Incident response: what's first step?"**
   - Isolate (stop bleeding)
   - Then: Investigate, notify customers/regulators

---

## References

- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)
- [GDPR Compliance - GDPR.eu](https://gdpr-info.eu/)
- [HIPAA Compliance - HHS.gov](https://www.hhs.gov/hipaa/)
- [PCI-DSS Standards](https://www.pcisecuritystandards.org/)
- [SOC 2 Framework](https://www.aicpa.org/interestareas/informationmanagement/soc2.html)

