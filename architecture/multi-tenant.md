# Multi-Tenant Architecture — SaaS Agentic Booking Chat

Este documento explica cómo la plataforma implementa el modelo multi-tenant para permitir que miles de empresas utilicen el servicio en una sola infraestructura, con aislamiento fuerte, seguridad garantizada y costos extremadamente bajos.

---

# 🎯 1. ¿Qué significa "multi-tenant" en este proyecto?

- **Un único backend** sirve a **múltiples empresas** (tenants).  
- Cada tenant posee:
  - claves propias,
  - configuraciones propias,
  - branding propio,
  - usuarios propios,
  - servicios, profesionales y horarios aislados,
  - modo IA configurable,
  - límites según plan,
  - estadísticas independientes.

Ningún tenant puede acceder a los datos de otro.

---

# 🧠 2. Aislamiento: la regla fundamental

El aislamiento se implementa mediante 3 capas:

1. **Identificación del tenant → API Key o JWT**  
2. **Aislamiento físico/lógico → DynamoDB con PK = TENANT#id**  
3. **Validación en backend → resolvers que siempre verifican tenantId**  

Ejemplo de acceso:

```
x-api-key → determina tenantId
AppSync → pasa tenantId al resolver
Lambda → restringe lecturas/escrituras a tenantId
DynamoDB → PK = TENANT#xxx evita lecturas cruzadas
```

---

# 🔑 3. Resolución del Tenant

Dependiendo del tipo de request:

---

## 3.1 Widget Público → API Key

El widget envía:

```
x-api-key: pk_live_XXXX
origin: https://dominio.cliente.com
```

AppSync + Lambda:

1) recuperan API key  
2) validan si está activa  
3) validan allowedOrigins  
4) obtienen su `tenantId`  
5) añaden `tenantId` al contexto del resolver  

---

## 3.2 Panel Admin → JWT Cognito

El JWT contiene:

```
tenantId: "TENANT_123"
role: "ADMIN"
```

AppSync usa estos claims para:

- asegurar acceso  
- autorizar operaciones admin  
- filtrar queries por `tenantId`

---

# 🗄️ 4. Diseño Multi-Tenant en DynamoDB

### Patrón base:

```
PK: TENANT#<tenantId>
SK: <ENTITY>#<entityId>
```

Ejemplo Services:

```
PK = TENANT#DERMASKIN
SK = SERVICE#123
```

### Beneficios:

- Aislamiento natural  
- Escalabilidad horizontal automática  
- Costo extremadamente bajo  
- Queries rápidas por tenant  
- Evita necesidad de múltiples instancias o clusters  

### Tablas multi-tenant:

- Services  
- Providers  
- ProviderAvailability  
- BookingExceptions  
- Bookings  
- Conversations  
- TenantApiKeys  
- Tenants  
- TenantUsage

---

# 🧩 5. Multi-Tenant en AppSync

Cada operación tiene una política clara:

### Público (Widget)
- requiere API key válida  
- requiere origin permitido  
- solo tiene acceso a:
  - servicios
  - proveedores
  - disponibilidad
  - reservar

### Privado (Admin)
- requiere JWT con tenantId  
- claims definen permisos  
- acceso total SOLO al tenant del JWT

---

# 📦 6. Configuración por Tenant (Settings)

Cada tenant puede configurar:

- branding del widget  
- idioma  
- servicios ofrecidos  
- profesionales  
- disponibilidad  
- IA activada/desactivada  
- plan contratado  
- límites por plan  
- API Keys  
- dominios permitidos  

Ejemplo en DynamoDB:

```
PK = TENANT#DERMASKIN
SK = SETTINGS#GLOBAL
settings = {
  widget: {...},
  ai: {...},
  booking: {...},
  limits: {...},
}
```

---

# 🧠 7. Multi-Tenant y Modo IA

Cada tenant controla su modo de agente:

### 1. **FSM (sin IA)**  
0 costo adicional.

### 2. **NLP asistido (Bedrock Haiku)**  
Costo bajo → ideal PRO/BUSINESS.

### 3. **IA completa (Bedrock Agent Core + Sonnet)**  
Costo medio/alto → ENTERPRISE.

El modo está almacenado en:

```
PK = TENANT#123
SK = SETTINGS#AI
```

Las Lambdas leen este setting en cada request del agente.

---

# 👥 8. Multi-Tenant y Usuarios

Los usuarios del panel admin también están aislados:

- `tenantId` en JWT obliga a que solo vean su información  
- roles disponibles:
  - owner  
  - admin  
  - viewer  

Nunca se usa un pool compartido sin claims de tenant.

---

# 📈 9. Multi-Tenant y Límites de Plan (Planes SaaS)

Cada tenant tiene límites configurables:

| Plan | Mensajes | Reservas | Profesionales | IA | Costo |
|-------|-----------|------------|----------------|--------|--------|
| FREE | 500 | 50 | 1 | FSM | $0 |
| PRO | 20k | 2k | 3 | NLP Haiku | $49 |
| BUSINESS | 100k | 10k | 10 | IA parcial | $149 |
| ENTERPRISE | ilimitado | ilimitado | 50 | full AI | $299–499 |

Estos límites están guardados en `TenantUsage` y `TenantSettings`.

---

# 📊 10. Multi-Tenant y Métricas

Se rastrea por tenant:

- número de mensajes  
- reservas creadas  
- tokens IA consumidos  
- errores  
- peak usage  
- orígenes usados  

Esto permite:
- facturación  
- restricciones por plan  
- dashboards individuales  

---

# 🌎 11. Multi-Región / Multi-Cuenta

Recomendado para escalabilidad o cumplimiento:

### Multi-Región
- CloudFront → distribución global  
- DynamoDB Global Table → resiliencia regional  
- AppSync multi-region → failover

### Multi-Cuenta
- DEV → QA → PROD  
- Tenants enterprise en cuenta dedicada (opcional)  
- API keys y datos se replican entre regiones si es necesario  

---

# 🔒 12. Seguridad Multi-Tenant

| Riesgo | Mitigación |
|--------|-------------|
| Acceso cruzado de tenant | PK = TENANT# en Dynamo, validación en Lambda |
| API key filtrada | Rotación, hashing, allowedOrigins |
| Ataques de inyección | VTL sanitizado, Lambdas a prueba de tampering |
| Abuso del widget | rate limiting por API Key |
| Sobrecarga IA | límites por tenant |

---

# ⚙️ 13. Operaciones Multi-Tenant

### Crear tenant
1. entry en tabla Tenants  
2. settings iniciales  
3. API key generada  
4. onboarding automático

### Migrar tenant
- copiar servicios y profes  
- mover settings  
- reemitir API key  
- update DNS si se usa domain específico

### Eliminar tenant
- soft delete (flag `status=DELETED`)  
- borrar datos con TTL opcional  

---

# 🧭 14. Roadmap Multi-Tenant

- soporte multi-branch sub-tenants  
- entornos dedicados por tenant (enterprise)  
- replicación total en múltiples regiones  
- límites configurables dinámicamente  
- tenant health score  

---

# ✔️ Fin del archivo
