# Seguridad & Compliance en la Nube

**Tags**: #cloud #security #compliance #gdpr #hipaa #vpc #iam #encryption #real-interview

---

## TL;DR

**Cloud Security** = modelo de responsabilidad compartida (proveedor de nube asegura infraestructura, tú aseguras data + acceso). **Capas clave**: VPC (aislamiento de red), IAM (control de acceso), encryption (at-rest, in-transit), audit logs. **Compliance**: GDPR (datos personales), HIPAA (datos de salud), PCI-DSS (tarjetas de crédito), SOC 2 (auditoría). La mayoría de empresas: GDPR + SOC 2 como mínimo.

---

## Concepto Core

- **Qué es**: Security = proteger data + infrastructure. Compliance = seguir reglas (GDPR, HIPAA, etc)
- **Por qué importa**: Data breach = $4M average cost + regulatory fines + reputation damage
- **Principio clave**: "Defense in depth" — múltiples layers, no single point of failure

---

## Truco de Memoria

**"Castillo con murallas"** — VPC = muralla. IAM = guardias (quién entra). Encryption = bóveda (data segura). Audit logs = vigilancia (quién hizo qué).

---

## Cómo explicarlo en entrevista

**Paso 1**: "Cloud security = responsabilidad compartida. Cloud provider = infraestructura, tú = data + acceso"

**Paso 2**: "Capas: VPC (red), IAM (acceso), encryption (datos), audit (logging)"

**Paso 3**: "Compliance = GDPR (EU), HIPAA (salud), PCI-DSS (pago), SOC 2 (auditoría)"

**Paso 4**: "Real: aislamiento VPC, IAM con least-privilege, encryption en todos lados, audit trails"

---

## Código/Ejemplos

### Part 1: VPC (Aislamiento de Red)

Qué es: Virtual Private Cloud (tu propia red en la nube)
Por qué: Aislar recursos, controlar tráfico, proteger datos

Arquitectura:

```
┌─────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16 (mi red privada)           │
├─────────────────────────────────────────────┤
│                                             │
│ Public Subnet 1 (10.0.1.0/24)               │
│ ├─ NAT Gateway (salida)                     │
│ └─ Bastion Host (SSH jump)                  │
│                                             │
│ Private Subnet 1 (10.0.10.0/24)             │
│ ├─ Base de datos (sin acceso a internet)    │
│ └─ Servidores de aplicación                 │
│                                             │
│ Private Subnet 2 (10.0.20.0/24) [us-east-2] │
│ └─ Base de datos backup (redundancia)      │
│                                             │
│ VPN Connection (acceso on-prem)             │
│ └─ VPN site-to-site con oficina central     │
│                                             │
└─────────────────────────────────────────────┘
```

Security Groups (firewall por instancia):

```text
├─ Web tier: Permitir puerto 443 (HTTPS) desde internet
├─ App tier: Permitir puerto 8080 solo desde web tier
└─ DB tier: Permitir puerto 5432 solo desde app tier
```

Network ACLs (firewall por subnet):

```text
├─ Inbound: Permitir IPs/puertos específicos
└─ Outbound: Negar internet (solo VPN)
```

Beneficios:

```text
✓ Aislamiento: No se puede acceder a la base de datos desde internet
✓ Control: Saber exactamente qué tráfico está permitido
✓ Compliance: Requisito de segmentación de red (HIPAA, PCI)
```

### Part 2: IAM (Gestión de Identidad y Acceso)

Qué es: Quién puede hacer qué con los recursos de nube
Por qué: Principio de least-privilege (acceso mínimo necesario)

Estructura:

```text
┌─────────────────────────────────┐
│ AWS Account (root = ¡nunca!)    │
├─────────────────────────────────┤
│ Usuarios                        │
│ ├─ alice@company.com            │
│ ├─ bob@company.com              │
│ └─ ci-cd-service                │
│                                 │
│ Grupos                          │
│ ├─ DataEngineers                │
│ ├─ Analysts                     │
│ └─ DevOps                       │
│                                 │
│ Roles (para servicios)          │
│ ├─ LambdaExecutionRole          │
│ ├─ EC2InstanceRole               │
│ └─ SparkClusterRole             │
│                                 │
│ Políticas (permisos)            │
│ ├─ ReadS3                       │
│ ├─ WriteRedshift                │
│ └─ ManageLambda                 │
└─────────────────────────────────┘
```

Ejemplo de Least-Privilege:

❌ MALA Política (demasiado permisiva):

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

¡Alice puede eliminar todo el data lake!

✅ BUENA Política (least-privilege):

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": ["arn:aws:s3:::data-lake/raw/*"]
}
```

Alice solo puede LEER archivos en bucket específico

Autenticación Multi-Factor (MFA):

```text
┌─ Usuario ingresa usuario + contraseña
├─ AWS solicita código 2FA (teléfono/app)
└─ Solo luego: acceso concedido
```

Credenciales temporales (para Lambda/EC2):

```text
┌─ Lambda role: LambdaExecutionRole
├─ Permisos: lectura S3, escritura DynamoDB
└─ Session token (válido 1 hora, expira automáticamente)
```

### Part 3: Encryption (Cifrado)

Qué es: Codificar datos para que solo usuarios autorizados puedan leer
Por qué: Requisito de compliance + mejor práctica de seguridad

Tipos:

Cifrado At-Rest (datos almacenados):

```text
├─ S3: Habilitar cifrado por defecto (AES-256)
├─ Base de datos: RDS con cifrado habilitado
├─ Volúmenes EBS: Snapshots cifrados
└─ Gestión de claves: AWS KMS (Key Management Service)
```

Cifrado In-Transit (datos en movimiento):

```text
├─ HTTPS (TLS 1.2+)
├─ VPN (túnel cifrado on-prem a cloud)
├─ Conexiones a base de datos: SSL/TLS requerido
└─ Lambda a S3: Siempre HTTPS
```

Cifrado Client-Side (capa de app):

```text
├─ Cifrar antes de subir a S3
├─ Enmascarar PII en código de aplicación
└─ Tokenización (reemplazar SSN → token)
```

Implementación real:

AWS S3 con cifrado por defecto:

```bash
aws s3api put-bucket-encryption \
  --bucket my-data-lake \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

Base de datos con cifrado:

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-db \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/xxx
```

Connection string (obliga TLS):

```
postgresql://user:pass@db.aws.com:5432/mydb?sslmode=require
```

Impacto en performance:

| Tipo       | Overhead                  |
| ---------- | ------------------------- |
| At-rest    | ~5% (negligible)          |
| In-transit | ~2% (negligible)          |
| Encryption | Obligatorio (no opcional) |

### Part 4: Auditoría y Logging & Compliance

Qué es: Registrar quién hizo qué, cuándo, dónde
Por qué: Detectar brechas, requisito de compliance, debugging

Audit trails (Pistas de auditoría):

AWS CloudTrail (logging de API):

```text
┌─ Cada llamada a API de AWS se registra
├─ Quién: Usuario/role que hizo la llamada
├─ Qué: Acción realizada
├─ Cuándo: Timestamp
├─ Dónde: Región, dirección IP
└─ Recurso: Qué recurso fue afectado
```

Ejemplo:

```json
{
  "eventName": "CreateDBInstance",
  "eventTime": "2024-01-15T10:30:45Z",
  "userIdentity": { "principalId": "alice@company.com" },
  "sourceIPAddress": "203.0.113.42",
  "awsRegion": "us-east-1",
  "requestParameters": {
    "dBInstanceIdentifier": "prod-db-1",
    "storageEncrypted": true
  }
}
```

VPC Flow Logs (logging de red):

```text
├─ Cada paquete de red se registra
├─ IP origen/destino
├─ Puerto, protocolo
├─ Aceptado/rechazado
└─ Bytes transferidos
```

S3 Access Logs (logging de objetos):

```text
├─ Quién accedió
├─ Nombre del objeto
└─ Cuándo
```

CloudWatch Alarms (Alertas):

```text
├─ Alerta si se usa cuenta root
├─ Alerta si se escriben datos sin cifrar
├─ Alerta si patrón de acceso inusual
└─ Alerta si violación de compliance
```

Configuration auditing (AWS Config):

```text
┌─ Registrar cambios en configuración de recursos
├─ Alerta si security group es modificado
├─ Alerta si cifrado es deshabilitado
└─ Alerta si recurso creado en región incorrecta
```

Política de retención:

```text
├─ CloudTrail: 7 años (requisito de compliance)
├─ VPC Logs: 1 año (operacional)
└─ S3 Access: 90 días (costo de almacenamiento)
```

### Part 5: Compliance Frameworks

GDPR (Regulación General de Protección de Datos - EU)

Qué es: Proteger datos personales de ciudadanos de la EU
Se aplica: Cualquier empresa que procesa datos de residentes de la EU

Requisitos:

```text
├─ Consentimiento: Obtener permiso antes de recopilar datos
├─ Derecho de Acceso: Usuario puede solicitar sus datos
├─ Derecho de Eliminación: Usuario puede eliminar sus datos ("derecho al olvido")
├─ Minimización de Datos: Recopilar solo lo necesario
├─ Encryption: At-rest e in-transit
├─ Audit trails: Registrar quién accedió datos
└─ Reporte de incidentes: Reportar brechas dentro de 72h
```

Implementación en cloud:

```text
┌─ Encryption en todos lados (at-rest, in-transit)
├─ VPC + IAM (control de acceso)
├─ CloudTrail logging (auditoría)
├─ Políticas de retención de datos (eliminar datos viejos)
└─ Plan de respuesta a incidentes
```

HIPAA (Ley de Portabilidad y Responsabilidad del Seguro Médico - USA)

Qué es: Proteger información de salud (PHI - Protected Health Information)
Se aplica: Proveedores de salud, aseguradoras, proveedores de nube con datos de salud

Requisitos:

```text
├─ Encryption (AES-256 o más fuerte)
├─ Controles de acceso (autenticación, autorización)
├─ Audit logs (registrar todos los accesos a PHI)
├─ Eliminación segura (no dejar datos en discos desmantelados)
├─ Notificación de brechas (notificar pacientes si datos se filtran)
└─ Acuerdos Asociado de Negocios (BAAs - contratos con proveedores)
```

Más estricto que GDPR (la salud es sensible)

PCI-DSS (Estándar de Seguridad de Datos de Industria de Tarjetas de Pago - USA)

Qué es: Proteger datos de tarjetas de crédito
Se aplica: Cualquier empresa que procesa tarjetas de crédito

Requisitos:

```text
├─ Segmentación de red (datos de tarjetas en red separada)
├─ Encryption (datos de tarjetas siempre cifrados)
├─ Sin contraseñas por defecto (¡cambiar contraseña root de AWS!)
├─ Controles de acceso fuertes (roles IAM)
├─ Audit logs (registrar acceso a datos de tarjetas)
├─ Escaneo de vulnerabilidades (penetration testing)
├─ Plan de respuesta a incidentes
└─ Auditoría de compliance anual (auditor externo)
```

El más estricto (fraude de pago = problema inmediato)

SOC 2 (Service Organization Control - USA)

Qué es: Auditoría de seguridad/compliance del proveedor de servicios
Se aplica: Empresas B2B SaaS, proveedores de nube

Requisitos:

```text
├─ Seguridad (prevenir acceso no autorizado)
├─ Disponibilidad (compromisos de uptime)
├─ Integridad de procesamiento (datos precisos y completos)
├─ Confidencialidad (datos no se divulgan)
└─ Privacidad (datos personales manejados correctamente)
```

Tipo 1: Snapshot en un punto en el tiempo

Tipo 2: Auditoría sobre 6-12 meses (preferido por clientes)

Costo: $50k-200k por auditoría (anual)

---

## Mundo Real: Checklist de Compliance

Checklist de auditoría de compliance:

```python
class ComplianceAudit:
    def gdpr_audit(self):
        """Protección de datos personales de EU"""
        checks = {
            "Cifrado at-rest": self.check_s3_encryption(),
            "Cifrado in-transit": self.check_tls_enforcement(),
            "Controles de acceso": self.check_iam_policies(),
            "Audit logs": self.check_cloudtrail_enabled(),
            "Retención de datos": self.check_retention_policies(),
            "Respuesta a incidentes": self.check_incident_plan(),
        }

        if not all(checks.values()):
            return "FALLA: No es compliant con GDPR"
        return "PASA: Es compliant con GDPR"

    def hipaa_audit(self):
        """Protección de datos de salud (más estricto que GDPR)"""
        checks = {
            "Cifrado": self.check_aes256_encryption(),
            "Aislamiento de red": self.check_vpc_isolation(),
            "Acceso logging": self.check_hipaa_audit_logs(),
            "BAA firmado": self.check_baa_with_aws(),
            "Respuesta de brechas": self.check_breach_procedure(),
        }

        if not all(checks.values()):
            return "FALLA: No es compliant con HIPAA"
        return "PASA: Es compliant con HIPAA"

    def check_s3_encryption(self):
        """Verificar que cifrado por defecto de S3 está habilitado"""
        response = s3.get_bucket_encryption(Bucket='my-bucket')
        return 'Rules' in response.get('ServerSideEncryptionConfiguration', {})
```

Resultado: Reportar a auditores (SOC 2 type 2, GDPR, HIPAA)

---

## Errores Comunes en Entrevista

- **Error**: "Cloud público no es seguro" → **Solución**: Cloud = más seguro que on-prem (AWS invierte miles de millones)

- **Error**: "Encryption es suficiente" → **Solución**: Encryption es UNA capa. También necesitas: IAM, VPC, audit logs

- **Error**: "Compliance es responsabilidad de IT" → **Solución**: Todos (ingenieros, managers) somos responsables de compliance

- **Error**: Guardar credenciales en código → **Solución**: Usa IAM roles (credenciales temporales, auto-rotate)

---

## Preguntas de Seguimiento

1.  **"¿Cuándo usar VPC vs Security Groups?"**
    - VPC: Aislamiento a nivel de red (infraestructura)
    - Security Groups: Reglas a nivel de aplicación (por instancia)
    - Ambos necesarios (defense in depth)

2.  **"¿Cuenta root - alguna vez usarla?"**
    - NO (excepto setup inicial de AWS account)
    - Siempre usa usuario/role de IAM con MFA

3.  **"¿Seguridad on-prem vs Cloud?"**
    - Cloud: Más fácil (AWS maneja seguridad física, actualizaciones)
    - On-prem: Más control (pero más trabajo)

4.  **"¿Respuesta a incidentes: cuál es el primer paso?"**
    - Aislar (detener sangrado)
    - Luego: Investigar, notificar clientes/reguladores

---

## References

- [GDPR Compliance - GDPR.eu](https://gdpr-info.eu/)
- [HIPAA Compliance - HHS.gov](https://www.hhs.gov/hipaa/)
- [PCI-DSS Standards](https://www.pcisecuritystandards.org/)
