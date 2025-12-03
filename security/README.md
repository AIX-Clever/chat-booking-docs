# Seguridad — SaaS Agentic Booking Chat

Este documento describe todas las medidas de seguridad necesarias para operar un SaaS multi-tenant que utiliza un widget público, un backend GraphQL, Lambdas serverless, IA opcional y un panel administrativo.

Seguridad es un componente fundamental, dado que:

- el widget es público,
- múltiples tenants comparten infraestructura,
- se manejan reservas (PII),
- hay integración con agentes de IA (Bedrock),
- se usan API keys y JWT,
- cada tenant requiere aislamiento estricto.

---

## 🔐 1. Modelo general de seguridad

El sistema separa seguridad en dos planos:

### 1) **Plano Público (Widget)**
Basado en:
- API key pública (truncada, scope restringido)
- allowedOrigins
- rate limiting por tenant y key

### 2) **Plano Privado (Admin Panel)**
Basado en:
- Cognito + JWT
- Claims `tenantId` + `role`
- Autorización estricta en AppSync

Ambos convergen en AppSync, que aplica políticas diferentes según el tipo de request.

---

## 🧩 2. Seguridad del Widget Público

### ¿Qué puede hacer el widget?
Solo operaciones de lectura y reserva:

- listar servicios
- listar profesionales
- obtener disponibilidad
- enviar mensajes al agente
- crear reserva

⚠️ **Nunca** puede:
- listar todos los clientes
- eliminar datos
- ver usuarios del panel
- ajustar settings del tenant
- crear API keys
- acceder a otro tenant

### Controles aplicados

| Control | Descripción |
|---------|-------------|
| **API Key pública** | Incluida en el script y enviada en headers |
| **Allowed Origins** | Se verifica que el dominio actual esté en la lista del tenant |
| **Scope de AppSync** | Solo resolvers públicos están habilitados |
| **Rate limiting** | Previene abuso por bots |
| **Hashing de API keys** | En la DB no se almacena la API key en claro |
| **Rotación de keys** | Owners pueden rotar keys sin downtime |

---

## 🔑 3. API Keys — formato, seguridad y rotación

### 3.1 Generación

Cuando el tenant crea una API key en el panel:

- se genera una key larga, del tipo `pk_live_<random>`
- se muestra **solo una vez**
- se almacena su **hash** (bcrypt/scrypt/argon2id)
- se almacena:
  - `publicKey`
  - `allowedOrigins`
  - `lastUsedAt`
  - `status` (`ACTIVE`, `REVOKED`)

### 3.2 Validación en AppSync

Cada request pública incluye:

```
x-api-key: pk_live_XXXX
origin: https://dominio-del-cliente.com
```

El resolver Lambda verifica:

1. existe ese hash  
2. está activa  
3. el origin está permitido  
4. no excede límites del plan  
5. obtiene `tenantId`  

### 3.3 Rotación

El tenant puede:

- crear una nueva key  
- actualizar snippet del widget  
- revocar key antigua  

---

## 👥 4. Seguridad del Panel Administrativo

El panel usa:

- **Cognito Hosted UI** o login embebido  
- Claims importantes en el JWT:
  - `tenantId`
  - `role` (`OWNER`, `ADMIN`, `VIEWER`)

### 4.1 Autorización en AppSync (admin)

Los resolvers admin verifican:

| Claim | Uso |
|-------|-----|
| `tenantId` | determinar acceso a datos |
| `role` | reglas RBAC |

Ejemplo:

```
owner → acceso total
admin → sin acceso a API keys ni usuarios
viewer → solo queries
```

---

## 🏛️ 5. Seguridad Multi-Tenant

Todo el backend aplica **aislamiento lógico** por tenant mediante:

- `tenantId` embebido en PK de DynamoDB
- validación estricta de `tenantId` en todas las Lambdas
- claims de Cognito en el panel admin
- API keys asociadas exclusivamente a un tenant

### Garantía:
"Un tenant jamás puede acceder a datos de otro tenant, incluso si manipula requests."

---

## 🔐 6. IAM minimalista

Reglas recomendadas:

### 6.1 Lambda

Cada Lambda debe tener permisos **solo** a su entidad:

Ejemplo `/booking`:

```
dynamodb:GetItem
dynamodb:PutItem
dynamodb:UpdateItem
```

### 6.2 AppSync

- IAM role por entorno  
- políticas separadas para resolvers públicos y privados  
- acceso indirecto a DynamoDB (nunca directo desde front)

### 6.3 CI/CD con OIDC

- GitHub Actions → OIDC role  
- Solo permite deploy en carpeta específica  
- No se guardan credentials en GitHub  

---

## 📦 7. Seguridad del Agente IA (Bedrock)

Si el tenant utiliza IA:

### 7.1 Riesgos cubiertos

- No se envía información sensible del usuario final al modelo  
- El prompt system-level corta PII innecesaria  
- Se utiliza context trimming  
- Se evita el reenvío de histórico completo  
- Los "tools" (Lambdas) tienen límites estrictos  
- No se exponen secretos, keys ni configuraciones internas  

### 7.2 Tokens y Costos

- Se monitorean tokens por tenant  
- Cortes automáticos si excede el plan  
- Mensaje al usuario:  
  "El asistente no está disponible en este momento."

---

## 🛡️ 8. Amenazas y mitigaciones

| Amenaza | Mitigación |
|---------|-------------|
| Inyección GraphQL | VTL sanitizado + resolvers Lambda |
| Uso fraudulento de API key | allowedOrigins + rate limiting + rotación |
| Intento de acceso entre tenants | validación estricta de tenantId |
| Exceso de tráfico | throttling + CloudFront caching |
| Fuga de JWT | expiración corta + refresh seguro |
| Exceso de IA tokens | límites por tenant + alertas |
| CSP del sitio bloquea widget | guía de configuración CSP |
| Spam en reservas | captcha (opcional) |

---

## 🚨 9. Playbook: qué hacer si se filtra una API key

1. **Revocar la API key inmediatamente**  
   Panel → API Keys → "Revocar".

2. **Crear una nueva key**  
   Panel → API Keys → "Crear nueva".

3. **Actualizar el snippet** en el sitio del cliente.

4. **Revisar logs de uso**  
   CloudWatch Insights → filtrar por esa publicKey.

5. **Monitorear límites del tenant**  
   Revisar `TenantUsage`.

6. **Notificar al tenant** (si corresponde).

---

## 🔍 10. Logging seguro y privacidad

### 10.1 Logs anonimizados
- Nunca loguear correo del usuario final  
- Nunca loguear texto completo si contiene PII  
- En modo IA: truncar mensajes > 500 chars

### 10.2 Retención
- 30 días recomendado  
- 7 días si se almacena conversación completa  
- TTL configurado por tabla (opcional)

### 10.3 GDPR-ready / LGPD-ready
- Fácil eliminación de datos por tenant  
- Conversaciones con TTL  
- Reservas pueden mantenerse según requisitos del cliente

---

## 🔎 11. Auditoría y Monitoreo

### Monitoreo recomendado:

- AppSync error rate  
- Lambda duration/errors  
- DynamoDB throttles  
- Consumo por API key  
- Tokens Bedrock por tenant  
- AllowedOrigin failures  
- Intentos de acceso entre tenants  

### Alarmas CloudWatch:
- más de X reservas por minuto  
- más de X tokens IA en 1h  
- aumento repentino de 5xx en AppSync  

---

## 🧭 12. Roadmap de Seguridad

- soporte para Web Application Firewall (WAF)  
- signed-requests por tenant (Admin API avanzada)  
- integración con CloudTrail Lake  
- DLP (Data Loss Prevention) para prompts IA  
- auditoría avanzada por tenant  

---

## 📚 Documentación Legacy (referencia)

### Panel Administrativo (Cognito)

**Método**: AWS Cognito User Pools

**Flujo**:
1. Usuario accede a `https://app.tu-saas.com`
2. Login con email + password
3. Cognito valida credenciales
4. Retorna JWT con claims:
   ```json
   {
     "sub": "user_123",
     "email": "admin@empresa.com",
     "custom:tenantId": "andina",
     "cognito:groups": ["admin"]
   }
   ```
5. JWT incluido en cada request a AppSync

**Seguridad adicional**:
- MFA opcional
- Política de contraseñas fuerte
- Sesiones con expiración
- Refresh tokens rotados

---

### 2. Widget Público (API Keys)

**Método**: API Key firmada

**Flujo**:
1. Widget envía `x-api-key: pk_live_abc123`
2. AppSync/Lambda calcula hash de la key
3. Busca en tabla `TenantApiKeys` (GSI por hash)
4. Valida:
   - Key activa
   - Dominio en `allowedOrigins`
   - No excede rate limit
5. Resuelve `tenantId`
6. Request procede con contexto del tenant

**Generación de API Keys**:
```python
import secrets
import hashlib

def generate_api_key():
    # Generar key aleatoria
    key = f"pk_live_{secrets.token_urlsafe(32)}"
    
    # Hash para guardar en DB
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    
    return key, key_hash
```

**Almacenamiento**:
- ❌ Nunca guardar la key en texto plano
- ✅ Solo guardar el hash
- ✅ La key se muestra solo una vez al crearla

---

## 🛡️ Aislamiento Multi-Tenant

### Nivel de base de datos

Cada tabla en DynamoDB incluye `tenantId` en su clave primaria:

```
PK = tenantId
PK = tenantId#providerId
PK = tenantId#serviceId
```

**Imposibilita** queries accidentales cross-tenant.

### Nivel de aplicación

Toda Lambda recibe `tenantId` como parámetro obligatorio:

```python
def lambda_handler(event, context):
    tenant_id = event['tenantId']
    
    # Todas las operaciones usan tenant_id
    services = get_services(tenant_id)
```

### Validación en cada capa

```mermaid
sequenceDiagram
    Widget->>AppSync: Request + API Key
    AppSync->>Auth Lambda: Validar key
    Auth Lambda->>TenantApiKeys: Buscar hash
    TenantApiKeys-->>Auth Lambda: tenantId="andina"
    Auth Lambda-->>AppSync: tenantId validado
    AppSync->>Business Lambda: Invocar con tenantId
    Business Lambda->>DynamoDB: Query(PK=tenantId)
```

---

## 🚪 Autorización (RBAC)

### Roles disponibles

| Rol | Panel Admin | Widget Config | Ver Reservas | Gestionar Catálogo |
|-----|-------------|---------------|--------------|-------------------|
| **Admin** | ✅ | ✅ | ✅ | ✅ |
| **Staff** | ✅ | ❌ | ✅ | ✅ |
| **Viewer** | ✅ | ❌ | ✅ | ❌ |

### Implementación en Cognito

Roles se asignan como **Cognito Groups**.

El JWT incluye:
```json
{
  "cognito:groups": ["admin", "reports"]
}
```

AppSync valida permisos en los resolvers:

```vtl
#if( !$ctx.identity.claims.get("cognito:groups").contains("admin") )
  $util.unauthorized()
#end
```

---

## 🔒 API Keys — Seguridad avanzada

### Allowed Origins (CORS)

Cada API Key tiene lista de dominios permitidos:

```json
{
  "apiKeyId": "key_001",
  "allowedOrigins": [
    "https://www.empresa.com",
    "https://empresa.com",
    "https://app.empresa.com"
  ]
}
```

**Validación**:
```python
def validate_origin(api_key, origin):
    key_data = get_api_key(api_key)
    
    if origin not in key_data['allowedOrigins']:
        raise Exception('Origin not allowed')
```

### Scopes (permisos granulares)

Cada key puede tener scopes limitados:

```json
{
  "scopes": [
    "widget:chat",
    "widget:booking",
    "widget:catalog"
  ]
}
```

Validas en cada operación:
```python
if 'widget:booking' not in api_key_data['scopes']:
    raise Exception('Insufficient permissions')
```

### Rotación de keys

**Buena práctica**: Rotar keys periódicamente.

En el panel admin:
1. Crear nueva key
2. Actualizar widget en sitio
3. Esperar propagación (24-48h)
4. Revocar key antigua

### Expiración automática

Puedes configurar keys con TTL:

```json
{
  "apiKeyId": "key_001",
  "expiresAt": "2026-01-01T00:00:00Z"
}
```

Lambda valida antes de usar:
```python
from datetime import datetime

def validate_expiration(api_key_data):
    if 'expiresAt' in api_key_data:
        expires = datetime.fromisoformat(api_key_data['expiresAt'])
        if datetime.utcnow() > expires:
            raise Exception('API Key expired')
```

---

## 🚦 Rate Limiting

### Por API Key

Cada key tiene límite de requests/minuto:

```json
{
  "apiKeyId": "key_001",
  "rateLimitPerMinute": 100
}
```

**Implementación** (Redis o DynamoDB):

```python
import time

def check_rate_limit(api_key_id, limit):
    key = f"rate_limit:{api_key_id}:{int(time.time() / 60)}"
    
    current = redis.incr(key)
    redis.expire(key, 60)
    
    if current > limit:
        raise Exception('Rate limit exceeded')
```

### Por Tenant

Límite global por tenant (según plan):

| Plan | Mensajes/mes | Reservas/mes | Requests/min |
|------|--------------|--------------|--------------|
| FREE | 500 | 50 | 10 |
| PRO | 10,000 | 1,000 | 100 |
| ENTERPRISE | Ilimitado | Ilimitado | 1,000 |

Validación en Lambda:
```python
def check_tenant_limits(tenant_id, operation):
    tenant = get_tenant(tenant_id)
    usage = get_monthly_usage(tenant_id)
    
    limits = PLAN_LIMITS[tenant['plan']]
    
    if operation == 'message':
        if usage['messages'] >= limits['messages']:
            raise Exception('Monthly message limit exceeded')
```

---

## 🔐 Encriptación

### En tránsito

- ✅ Todo el tráfico usa **HTTPS/TLS 1.3**
- ✅ AppSync solo acepta conexiones seguras
- ✅ Certificados gestionados por AWS Certificate Manager

### En reposo

- ✅ DynamoDB con **encryption at rest** habilitado
- ✅ S3 (si se usa) con **SSE-S3** o **SSE-KMS**
- ✅ Lambda environment variables encriptadas con KMS

### Datos sensibles

**API Keys**:
- Guardadas como hash (SHA256)
- Salt por tenant

**Información de clientes**:
- Emails y teléfonos encriptados en DynamoDB (opcional)
- PII no se envía a servicios de AI externos

---

## 📝 Auditoría y Logging

### CloudWatch Logs

Todas las Lambdas loggean:

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(json.dumps({
        'tenantId': event['tenantId'],
        'operation': 'createBooking',
        'timestamp': datetime.utcnow().isoformat(),
        'userId': event.get('userId'),
        'apiKeyId': event.get('apiKeyId')
    }))
```

### DynamoDB Streams (auditoría completa)

Habilitar streams en tablas críticas:
- `Bookings`
- `TenantApiKeys`
- `Tenants`

Lambda procesa eventos:
```python
def audit_stream_handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'MODIFY':
            old = record['dynamodb']['OldImage']
            new = record['dynamodb']['NewImage']
            
            # Guardar en tabla Audit
            save_audit_log({
                'tenantId': new['tenantId']['S'],
                'table': record['eventSourceARN'].split('/')[1],
                'operation': 'UPDATE',
                'changedBy': context.identity,
                'changes': diff(old, new)
            })
```

### CloudTrail

Habilitar CloudTrail para registrar:
- Accesos a AWS Console
- Cambios en configuración de AppSync
- Modificaciones de IAM roles

---

## 🛡️ Protección contra ataques

### SQL Injection

❌ No aplica (no usamos SQL).

DynamoDB usa queries parametrizadas nativamente.

### XSS (Cross-Site Scripting)

Widget sanitiza inputs:

```javascript
function sanitizeMessage(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}
```

Backend también valida:
```python
import html

def sanitize(text):
    return html.escape(text)
```

### CSRF

AppSync usa:
- JWT en headers (no cookies)
- Origin validation

### DDoS

Mitigación en capas:

1. **CloudFront**: AWS Shield Standard
2. **AppSync**: Rate limiting integrado
3. **Lambda**: Throttling configurado
4. **DynamoDB**: On-demand scaling

---

## 🔍 Monitoreo de seguridad

### Alertas configuradas

- Intento de acceso con key revocada
- Exceso de requests fallidos (posible ataque)
- Cambio en configuración de tenant
- Acceso desde IP sospechosa

### Herramientas

- **CloudWatch Alarms**
- **GuardDuty** para detección de amenazas
- **Security Hub** para compliance
- **Macie** para detectar PII expuesta

---

## 📋 Checklist de seguridad

### Para desarrollo

- [ ] API Keys nunca en código
- [ ] Variables sensibles en AWS Secrets Manager
- [ ] Logs no incluyen PII
- [ ] Tests de seguridad automatizados

### Para producción

- [ ] MFA habilitado para admins
- [ ] CloudTrail activo
- [ ] Encryption at rest en todas las tablas
- [ ] Backup automático de DynamoDB
- [ ] WAF configurado en CloudFront
- [ ] Rate limiting activo
- [ ] Monitoreo 24/7
- [ ] Plan de respuesta a incidentes

---

## 🆘 Respuesta a incidentes

### Procedimiento

1. **Detectar**: Alarma en CloudWatch
2. **Contener**: Revocar API Keys afectadas
3. **Investigar**: Revisar CloudWatch Logs y CloudTrail
4. **Remediar**: Rotar keys, actualizar permisos
5. **Comunicar**: Notificar a tenants afectados
6. **Documentar**: Postmortem del incidente

---

## 📚 Documentos relacionados

- [Arquitectura Multi-Tenant](/architecture/multi-tenant.md)
- [Schema DynamoDB](/architecture/dynamodb-schema.md)
- [Deployment](/deployment/README.md)
