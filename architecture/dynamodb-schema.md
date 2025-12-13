# DynamoDB — Esquema Multi-Tenant
SaaS Agentic Booking Chat

Este documento define el diseño de las tablas DynamoDB utilizadas por la plataforma, incluyendo:

- estructura multi-tenant (PK/SK),
- GSIs,
- patrones de acceso,
- ejemplos de ítems,
- recomendaciones de capacidad y TTL.

---

# 🧠 1. Principios de diseño

1. **Multi-tenant por clave**  
   Cada ítem incluye `tenantId` y se almacena con un PK que comienza con `TENANT#<tenantId>`.

2. **Tablas por dominio**  
   Separamos dominios lógicos: servicios, profesionales, disponibilidad, reservas, conversaciones, tenants, API keys, uso.

3. **Accesos típicos primero**  
   El diseño parte de las queries principales:
   - listar servicios por tenant
   - listar profesionales por tenant
   - obtener disponibilidad por profesional
   - buscar reservas por fecha / profesional
   - lookup de API key
   - lookup de settings del tenant

4. **On-Demand billing**  
   Se recomienda `PAY_PER_REQUEST` (on-demand) para elasticidad y simplicidad.

---

# 📚 2. Vista general de tablas

| Tabla | Propósito |
|--------|-----------|
| `Services` | Catálogo de servicios por tenant |
| `Providers` | Profesionales por tenant |
| `ProviderAvailability` | Horarios recurrentes y reglas por profesional |
| `BookingExceptions` | Días u horas bloqueadas |
| `Bookings` | Reservas confirmadas |
| `Conversations` | Estado del chat por conversación |
| `Tenants` | Configuración y plan de cada tenant |
| `TenantApiKeys` | API keys públicas para el widget |
| `ChatBooking-Categories` | Categorías para agrupar servicios |
| `TenantUsage` | Métricas mensuales por tenant |

---

# 🧾 3. Tabla: `Services`

Catálogo de servicios ofrecidos por cada tenant.

## 3.1 Esquema

- **PK**: `TENANT#<tenantId>`
- **SK**: `SERVICE#<serviceId>`
- **GSI1** (por nombre de servicio):
  - `GSI1PK`: `TENANT#<tenantId>`
  - `GSI1SK`: `NAME#<normalizedName>`

## 3.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "SERVICE#svc_123",
  "tenantId": "DERMASKIN",
  "serviceId": "svc_123",
  "name": "Limpieza Facial",
  "description": "Limpieza profunda de rostro",
  "category": "Dermatología",
  "durationMinutes": 60,
  "price": 35000,
  "active": true,
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-02T12:00:00Z"
}
```

## 3.3 Patrones de acceso

- **Listar servicios** → Query por `PK = TENANT#...` + `begins_with(SK, "SERVICE#")`
- **Buscar servicio por nombre** → Query en GSI1 por `TENANT#` + `NAME#<normalized>` (útil en IA)

- **Buscar servicio por nombre** → Query en GSI1 por `TENANT#` + `NAME#<normalized>` (útil en IA)

---

# 📂 4. Tabla: `Categories`

Agrupación lógica de servicios (ej: "Facial", "Corporal", "Consultas").

## 4.1 Esquema

- **PK**: `tenantId` (String)
- **SK**: `categoryId` (String)

> Nota: Esta tabla usa un esquema simple PK/SK sin prefijos compuestos como `TENANT#`.

## 4.2 Atributos

```json
{
  "tenantId": "DERMASKIN",
  "categoryId": "cat_001",
  "name": "Facial",
  "description": "Tratamientos para el rostro",
  "isActive": true,
  "displayOrder": 1,
  "metadata": {},
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-02T12:00:00Z"
}
```

## 4.3 Patrones de acceso

- **Listar categorías por tenant** → Query `PK = tenantId`
- **Obtener categoría** → GetItem `PK = tenantId`, `SK = categoryId`

---

# 👩‍⚕️ 5. Tabla: `Providers`

Profesionales del tenant.

## 4.1 Esquema

- **PK**: `TENANT#<tenantId>`
- **SK**: `PROVIDER#<providerId>`
- **GSI1** (por servicio):
  - `GSI1PK`: `TENANT#<tenantId>#SERVICE#<serviceId>`
  - `GSI1SK`: `PROVIDER#<providerId>`

## 4.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "PROVIDER#pro_456",
  "tenantId": "DERMASKIN",
  "providerId": "pro_456",
  "name": "Dra. Martínez",
  "bio": "Dermatóloga con 10 años de experiencia",
  "serviceIds": ["svc_123", "svc_456"],
  "timezone": "America/Santiago",
  "active": true,
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-02T12:00:00Z",
  "GSI1PK": "TENANT#DERMASKIN#SERVICE#svc_123",
  "GSI1SK": "PROVIDER#pro_456"
}
```

## 4.3 Patrones de acceso

- **Listar providers por tenant** → Query `PK = TENANT#...` AND `begins_with(SK, "PROVIDER#")`
- **Listar providers por servicio** → Query en GSI1 por `TENANT#...#SERVICE#<id>`

---

# 🗓 5. Tabla: `ProviderAvailability`

Define reglas recurrentes de disponibilidad por profesional.

## 5.1 Esquema

- **PK**: `TENANT#<tenantId>#PROVIDER#<providerId>`
- **SK**: `RULE#<ruleId>`

Cada regla describe la disponibilidad semanal.

## 5.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN#PROVIDER#pro_456",
  "SK": "RULE#1",
  "tenantId": "DERMASKIN",
  "providerId": "pro_456",
  "ruleId": "1",
  "dayOfWeek": "MON",
  "timeRanges": [
    {"start": "09:00", "end": "13:00"},
    {"start": "15:00", "end": "18:00"}
  ],
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-02T12:00:00Z"
}
```

## 5.3 Patrones de acceso

- **Obtener disponibilidad base de un provider** → Query por `PK = TENANT#...#PROVIDER#<id>`

---

# 🚫 6. Tabla: `BookingExceptions`

Bloqueos o excepciones a la disponibilidad estándar.

## 6.1 Esquema

- **PK**: `TENANT#<tenantId>#PROVIDER#<providerId>`
- **SK**: `EXCEPTION#<date>#<sequence>`

## 6.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN#PROVIDER#pro_456",
  "SK": "EXCEPTION#2025-01-10#1",
  "tenantId": "DERMASKIN",
  "providerId": "pro_456",
  "date": "2025-01-10",
  "reason": "Vacaciones",
  "allDay": true,
  "timeRanges": [],
  "createdAt": "2025-01-01T10:00:00Z"
}
```

## 6.3 Patrones de acceso

- **Excepciones por provider** → Query `PK = TENANT#...#PROVIDER#<id>` AND `begins_with(SK, "EXCEPTION#")`
- **Excepciones por día** → se puede modelar un GSI si se requiere alta frecuencia.

---

# 📅 7. Tabla: `Bookings`

Reservas confirmadas.

## 7.1 Esquema

Acceso principal: reservas por tenant, por provider y por fecha.

- **PK**: `TENANT#<tenantId>#PROVIDER#<providerId>`
- **SK**: `BOOKING#<date>#<time>#<bookingId>`
- **GSI1** — por tenant + fecha
  - `GSI1PK`: `TENANT#<tenantId>#DATE#<YYYY-MM-DD>`
  - `GSI1SK`: `BOOKING#<time>#<bookingId>`

## 7.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN#PROVIDER#pro_456",
  "SK": "BOOKING#2025-01-10#15:30#bkg_789",
  "tenantId": "DERMASKIN",
  "bookingId": "bkg_789",
  "serviceId": "svc_123",
  "providerId": "pro_456",
  "customerName": "Juan Pérez",
  "customerEmail": "juan@example.com",
  "datetime": "2025-01-10T18:30:00Z",
  "status": "CONFIRMED",
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-01T10:00:00Z",
  "GSI1PK": "TENANT#DERMASKIN#DATE#2025-01-10",
  "GSI1SK": "BOOKING#15:30#bkg_789"
}
```

## 7.3 Patrones de acceso

- **Listar reservas de un provider por día**
  → Query en tabla principal por `PK = TENANT#...#PROVIDER#` + `begins_with(SK, "BOOKING#<fecha>")`

- **Listar todas las reservas de un tenant por día**
  → Query GSI1 `GSI1PK = TENANT#...#DATE#<fecha>`

- **Buscar reserva por bookingId**
  → Lookup requiere GSI2, o se puede guardarlo en Conversations y en panel con bookmarks.

Si se requiere acceso directo por bookingId, se puede agregar un GSI2 con `GSI2PK = BOOKING#<bookingId>`.

---

# 💬 8. Tabla: `Conversations`

Almacena el estado del chat por conversationId.

## 8.1 Esquema

- **PK**: `TENANT#<tenantId>`
- **SK**: `CONV#<conversationId>`
- **GSI1** — por fecha de última actividad (opcional)
  - `GSI1PK`: `TENANT#<tenantId>#CONV`
  - `GSI1SK`: `<lastMessageAt>`

## 8.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "CONV#conv_123",
  "tenantId": "DERMASKIN",
  "conversationId": "conv_123",
  "state": "SERVICE_PENDING",
  "serviceId": null,
  "providerId": null,
  "datetime": null,
  "context": {
    "language": "es",
    "channel": "web"
  },
  "lastMessageAt": "2025-01-10T10:15:30Z",
  "ttl": 1736534400
}
```

`ttl` se usa para eliminar conversaciones expiradas.

---

# 🧱 9. Tabla: `Tenants`

Información y configuración de cada tenant.

## 9.1 Esquema

- **PK**: `TENANT#<tenantId>`
- **SK**: `META#TENANT`

## 9.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "META#TENANT",
  "tenantId": "DERMASKIN",
  "name": "Clínica Dermaskin",
  "country": "CL",
  "timezone": "America/Santiago",
  "plan": "PRO",
  "billingEmail": "admin@dermaskin.cl",
  "status": "ACTIVE",
  "settings": {
    "widget": { "primaryColor": "#4A90E2", "language": "es" },
    "ai": { "mode": "FSM" }
  },
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-02T12:00:00Z"
}
```

---

# 🔑 10. Tabla: `TenantApiKeys`

API keys para el widget.

## 10.1 Esquema

Acceso frecuente: por key.

- **PK**: `TENANT#<tenantId>`
- **SK**: `APIKEY#<apiKeyId>`
- **GSI1** — lookup por key hash
  - `GSI1PK`: `APIKEY#<publicKeyPrefix>` (ej: primeros N caracteres)
  - `GSI1SK`: `<apiKeyId>`

O bien:
- `GSI1PK`: `TENANT#<tenantId>`
- `GSI1SK`: `PUBLICKEY#<publicKey>`

(Si se quiere lookup directo, aunque es menos seguro; normalmente se usa hash parcial).

## 10.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "APIKEY#key_001",
  "tenantId": "DERMASKIN",
  "apiKeyId": "key_001",
  "publicKey": "pk_live_derma_01",
  "apiKeyHash": "<bcrypt-hash>",
  "status": "ACTIVE",
  "allowedOrigins": ["https://dermaskin.cl"],
  "createdAt": "2025-01-01T10:00:00Z",
  "lastUsedAt": "2025-01-10T10:30:00Z",
  "GSI1PK": "APIKEY#pk_live_derma_01",
  "GSI1SK": "key_001"
}
```

---

# 📈 11. Tabla: `TenantUsage`

Métricas por tenant y período.

## 11.1 Esquema

- **PK**: `TENANT#<tenantId>`
- **SK**: `USAGE#<YYYY-MM>`

## 11.2 Atributos

```json
{
  "PK": "TENANT#DERMASKIN",
  "SK": "USAGE#2025-01",
  "tenantId": "DERMASKIN",
  "period": "2025-01",
  "messages": 12345,
  "bookings": 200,
  "tokensIA": 500000,
  "errors": 15,
  "plan": "PRO",
  "updatedAt": "2025-01-31T23:59:59Z"
}
```

---

# ⚙️ 12. Configuración recomendada de tablas

- **Billing mode**: `PAY_PER_REQUEST` (on-demand)
- **PITR** (Point-in-time recovery): habilitado para:
  - Bookings
  - Tenants
  - TenantApiKeys
- **TTL**:
  - Conversations: 1–7 días
  - Logs internos (si se usan): 7–30 días

---

# 🧪 13. Testing del esquema

Pruebas recomendadas:

**Crear tenant y correr flujo completo:**
- alta de servicios
- alta de providers
- seteo de disponibilidad
- creación de reserva
- recuperación de datos por distintas keys

**Test de aislamiento:**
- simular dos tenants e intentar acceder cruzado
- verificar que queries siempre incluyen tenantId correcto

---

# ✔️ Fin del archivo
