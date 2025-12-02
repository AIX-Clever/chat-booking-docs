# Esquema DynamoDB — Multi-Tenant SaaS Agentic Booking Chat

Este documento define **todas las tablas de DynamoDB**, sus claves primarias, GSIs, ejemplos de items y convenciones para implementar el backend multi-tenant del SaaS.

Está diseñado para ser **machine-friendly**, permitiendo a herramientas como Codex generar los modelos, IaC y validaciones automáticamente.

---

## 🧩 Principios de diseño DynamoDB

1. **Cada tabla está segmentada por tenant**, usando `tenantId` en la clave primaria.
2. **Se evita el acceso cross-tenant** mediante claves que requieren `tenantId`.
3. **Las claves de rango permiten queries eficientes** por servicios, proveedores o reservas.
4. **Las tablas están optimizadas para serverless** (accesos simples y consistentes).
5. **No se usa modelo single-table**, ya que separar dominios mejora claridad y mantenimiento.

---

## 📚 Listado de tablas

1. `Tenants`
2. `TenantApiKeys`
3. `Services`
4. `Providers`
5. `ProviderAvailability`
6. `Bookings`
7. `ConversationState`

Cada tabla se define con:

- Clave primaria (PK/SK)
- GSIs
- Ejemplo de Item
- Reglas de acceso
- Patrones de query

---

## 🟦 1. Tabla: `Tenants`

Contiene la información maestra de cada empresa.

### **Clave primaria**
```
PK = tenantId
```

### **Campos**
| Campo | Tipo | Descripción |
|-------|------|--------------|
| tenantId | STRING | ID único del tenant |
| name | STRING | Nombre comercial |
| slug | STRING | Nombre breve para URLs |
| status | STRING | ACTIVE, SUSPENDED, TRIAL, CANCELLED |
| plan | STRING | FREE, PRO, ENTERPRISE |
| settings | MAP | Configuraciones del tenant |
| createdAt | STRING ISO | Fecha creación |
| ownerUserId | STRING | Usuario creador |
| billingEmail | STRING | Email contacto |

### **GSI1 — por dueño**
```
GSI1PK = ownerUserId
GSI1SK = createdAt
```

### **Ejemplo de Item**
```json
{
  "tenantId": "andina",
  "name": "Coca-Cola Andina",
  "slug": "andina",
  "status": "ACTIVE",
  "plan": "PRO",
  "createdAt": "2025-01-01T00:00:00Z",
  "ownerUserId": "user_123",
  "billingEmail": "billing@andina.com",
  "settings": {
    "language": "es-CL",
    "defaultTimezone": "America/Santiago",
    "widget": {
      "primaryColor": "#f44336",
      "position": "bottom-right"
    }
  }
}
```

---

## 🟩 2. Tabla: `TenantApiKeys`

Asigna claves públicas a un tenant. Permite resolver `tenantId` a partir de una API Key enviada por el widget.

### **Clave primaria**
```
PK = tenantId
SK = apiKeyId
```

### **GSI1 — búsqueda rápida por hash**
```
GSI1PK = apiKeyHash
GSI1SK = tenantId
```

### **Campos**
| Campo | Descripción |
|-------|-------------|
| apiKeyPublic | parte visible de la key |
| apiKeyHash | hash irreversible (SHA256/HMAC) |
| status | ACTIVE, REVOKED |
| allowedOrigins | Lista de dominios permitidos |
| createdAt | Fecha emisión |
| lastUsedAt | Última vez utilizada |
| rateLimit | Límite por minuto |

### **Ejemplo de Item**
```json
{
  "tenantId": "andina",
  "apiKeyId": "key_001",
  "apiKeyHash": "sha256_hash_here",
  "status": "ACTIVE",
  "allowedOrigins": ["https://www.andina.cl"],
  "rateLimit": 100,
  "createdAt": "2025-01-01T10:00:00Z"
}
```

---

## 🟨 3. Tabla: `Services`

Catálogo de servicios de cada tenant.

### **Clave primaria**
```
PK = tenantId
SK = serviceId
```

### **Campos**
| Campo | Tipo |
|-------|------|
| name | STRING |
| description | STRING |
| durationMinutes | NUMBER |
| price | NUMBER |
| category | STRING |
| active | BOOLEAN |

### **GSI opcional — por categoría**
```
GSI1PK = tenantId#category
GSI1SK = name
```

### **Ejemplo de Item**
```json
{
  "tenantId": "andina",
  "serviceId": "svc_123",
  "name": "Masaje descontracturante",
  "description": "Masaje de 60 minutos",
  "durationMinutes": 60,
  "price": 25000,
  "category": "masajes",
  "active": true
}
```

---

## 🟧 4. Tabla: `Providers`

Profesionales asociados a los servicios.

### **Clave primaria**
```
PK = tenantId
SK = providerId
```

### **Campos**
| Campo | Tipo |
|-------|------|
| name | STRING |
| bio | STRING |
| services | LIST<STRING> |
| timezone | STRING |
| active | BOOLEAN |

### **Ejemplo**
```json
{
  "tenantId": "andina",
  "providerId": "pro_55",
  "name": "María González",
  "bio": "Masajista profesional",
  "services": ["svc_123", "svc_456"],
  "timezone": "America/Santiago",
  "active": true
}
```

---

## 🟫 5. Tabla: `ProviderAvailability`

Disponibilidad recurrente del proveedor.

### **Clave primaria**
```
PK = tenantId#providerId
SK = dayOfWeek   // ej: "MON", "TUE"
```

### **Campos**
| Campo | Descripción |
|-------|-------------|
| timeRanges | Horarios disponibles por día |
| breaks | Pausas |
| exceptions | Días libres específicos |

### **Ejemplo**
```json
{
  "PK": "andina#pro_55",
  "SK": "MON",
  "timeRanges": [
    {"startTime": "09:00", "endTime": "13:00"},
    {"startTime": "15:00", "endTime": "19:00"}
  ],
  "breaks": [{"startTime": "11:00", "endTime": "11:15"}],
  "exceptions": []
}
```

---

## 🟥 6. Tabla: `Bookings`

Registro de reservas confirmadas.

### **Clave primaria**
```
PK = tenantId#providerId
SK = startTime  // ISO string
```

Esto permite prevenir overbooking con una condición atómica:

```python
ConditionExpression: attribute_not_exists(PK)
```

### **Campos**
| Campo | Detalle |
|-------|---------|
| bookingId | ID único |
| serviceId | STRING |
| customerId | STRING |
| endTime | STRING ISO |
| status | PENDING, CONFIRMED, CANCELLED |
| paymentStatus | NONE, PENDING, PAID |
| createdAt | STRING ISO |

### **GSI — por usuario**
```
GSI1PK = tenantId#customerId
GSI1SK = startTime
```

### **Ejemplo**
```json
{
  "PK": "andina#pro_55",
  "SK": "2025-12-01T17:30:00Z",
  "bookingId": "book_789",
  "tenantId": "andina",
  "providerId": "pro_55",
  "serviceId": "svc_123",
  "customerId": "cust_001",
  "endTime": "2025-12-01T18:30:00Z",
  "status": "CONFIRMED",
  "paymentStatus": "PAID",
  "createdAt": "2025-12-01T10:00:00Z"
}
```

---

## 🟪 7. Tabla: `ConversationState`

Estado del agente conversacional por usuario.

### **Clave primaria**
```
PK = tenantId
SK = conversationId
```

### **Campos**
| Campo | Detalle |
|-------|---------|
| state | INIT, SERVICE_PENDING… |
| serviceId | si ya lo definió |
| providerId | si ya lo definió |
| slotStart / slotEnd | si ya seleccionó horario |
| updatedAt | ISO |
| userContext | MAP |

### **Ejemplo**
```json
{
  "PK": "andina",
  "SK": "conv_abc123",
  "state": "PROVIDER_SELECTED",
  "serviceId": "svc_123",
  "providerId": "pro_55",
  "slotStart": null,
  "slotEnd": null,
  "updatedAt": "2025-12-01T15:30:00Z",
  "userContext": {
    "userId": "user_ext_001",
    "name": "Juan Pérez"
  }
}
```

---

## 📐 Patrones de acceso por tabla

### **Tenants**
- `GetItem(tenantId)` — Leer info del tenant
- `Query(GSI1PK=ownerUserId)` — Listar tenants de un usuario

### **TenantApiKeys**
- `Query(GSI1PK=apiKeyHash)` — Resolver API Key → tenant
- `Query(PK=tenantId)` — Listar keys de un tenant

### **Services**
- `Query(PK=tenantId)` — Listar servicios del tenant
- `GetItem(tenantId, serviceId)` — Obtener servicio específico

### **Providers**
- `Query(PK=tenantId)` — Listar profesionales del tenant
- Filtrar por `services` contiene `serviceId` (post-query)

### **ProviderAvailability**
- `Query(PK=tenantId#providerId)` — Disponibilidad semanal

### **Bookings**
- `Query(PK=tenantId#providerId, SK between dates)` — Reservas del profesional
- `Query(GSI1PK=tenantId#customerId)` — Reservas del cliente
- `PutItem with ConditionExpression` — Crear reserva sin overbooking

### **ConversationState**
- `GetItem(tenantId, conversationId)` — Estado de conversación
- `UpdateItem` — Actualizar estado del FSM

---

## 🛠️ Implementación con IaC

### CloudFormation (ejemplo para Tenants)

```yaml
TenantsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: Tenants
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: tenantId
        AttributeType: S
      - AttributeName: ownerUserId
        AttributeType: S
      - AttributeName: createdAt
        AttributeType: S
    KeySchema:
      - AttributeName: tenantId
        KeyType: HASH
    GlobalSecondaryIndexes:
      - IndexName: GSI1
        KeySchema:
          - AttributeName: ownerUserId
            KeyType: HASH
          - AttributeName: createdAt
            KeyType: RANGE
        Projection:
          ProjectionType: ALL
```

### CDK (TypeScript ejemplo)

```typescript
const tenantsTable = new dynamodb.Table(this, 'Tenants', {
  partitionKey: { name: 'tenantId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
});

tenantsTable.addGlobalSecondaryIndex({
  indexName: 'GSI1',
  partitionKey: { name: 'ownerUserId', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'createdAt', type: dynamodb.AttributeType.STRING },
});
```

---

## 🔐 Seguridad

1. **IAM Policies** — Lambdas tienen acceso solo a las tablas necesarias.
2. **Condition Expressions** — Prevención de overbooking en `Bookings`.
3. **TTL opcional** — Para limpiar conversaciones antiguas.
4. **Encryption at Rest** — Habilitar en todas las tablas.
5. **Point-in-Time Recovery** — Para producción.

---

## 📊 Métricas y Observabilidad

Cada operación debe loggear:

- `tenantId`
- `apiKeyId` (si aplica)
- Operación (query/put/update)
- Latencia
- Errores

Esto permite:

- Facturación por tenant
- Detección de abuso
- Optimización de queries
- Auditoría completa

---

## 📚 Documentos relacionados

- `/architecture/multi-tenant.md`
- `/architecture/appsync-schema.md`
- `/architecture/lambdas.md`
- `/security/README.md`
