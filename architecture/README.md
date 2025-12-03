# Arquitectura del Sistema — SaaS Agentic Booking Chat

Este documento describe la arquitectura completa del proyecto, incluyendo:
- los componentes principales,
- el modelo multi-tenant,
- flujos de datos,
- decisiones de diseño,
- extensiones opcionales con IA (Bedrock Agent Core),
- patrones de escalabilidad.

La arquitectura está diseñada para ser:
**serverless, escalable, multi-tenant, de bajo costo, segura y simple de operar.**

---

## 🏗️ 1. Descripción General

La plataforma está compuesta por tres subsistemas principales:

1. **Widget Público (Cliente Final)**  
   - Integrado mediante un `<script>`
   - Renderizado en React (bundle UMD/IIFE)
   - Comunicación via GraphQL

2. **Backend Multi-Tenant (AppSync + Lambda + DynamoDB)**  
   - Aislado por `tenantId`
   - Costo extremadamente bajo
   - Totalmente serverless

3. **Panel Administrativo (Backend + UI)**  
   - Para gestionar servicios, profesionales, horarios, reservas, branding y API keys

---

## 🔍 2. Diagrama Conceptual

```
 ┌───────────────────────────┐
 │   Cliente / Widget Web    │
 └─────────────┬─────────────┘
               │ GraphQL
               ▼
    ┌───────────────────────────┐
    │ AWS AppSync (GraphQL API) │
    └─────────────┬─────────────┘
                  │ VTL/Lambda resolvers
                  ▼
    ┌───────────────────────────┐
    │   AWS Lambda (Python)     │
    └───────┬──────────┬────────┘
            │          │
            ▼          ▼
┌────────────────┐ ┌──────────────────┐
│ DynamoDB Core  │ │  Bedrock (IA)    │
└────────────────┘ └──────────────────┘
  Multi-Tenant DB       Opcional
```

---

## 🌍 3. Multi-Tenant dentro de DynamoDB

El aislamiento entre tenants se logra usando `tenantId` como parte de la clave primaria.

Ejemplo (tabla Services):

```
PK = TENANT#<tenantId>
SK = SERVICE#<serviceId>
```

Esto permite:

- aislar datos por tenant  
- alta escalabilidad  
- bajo costo  
- queries eficientes  
- soporte natural para entornos con miles de tenants  

Documentación completa:  
📄 `/docs/architecture/multi-tenant.md`

---

## 🧠 4. Arquitectura del Agente

El agente opera en dos modos:

---

### 4.1 Modo 1 — Agente Determinístico (FSM)

**Características:**
- No usa IA
- Costo ~0
- Conversación guiada por estados
- Ideal para plan FREE/PRO

**FSM:**

```
INIT
  → SERVICE_PENDING
  → SERVICE_SELECTED
  → PROVIDER_PENDING
  → PROVIDER_SELECTED
  → SLOT_PENDING
  → CONFIRM_PENDING
  → BOOKING_CONFIRMED
```

La FSM se implementa en Lambda y mantiene estado en `Conversations`.

---

### 4.2 Modo 2 — Agente con IA (Bedrock Agent Core)

Opcional y configurable por tenant.

**Beneficios:**
- Interpretación de intención
- Extracción de entidades
- Tono natural
- Orquestación automática de tools:
  - catalog lookup
  - availability lookup
  - booking creation

**Recomendación de modelos:**

- **Claude Haiku** → para clasificación y pasos simples  
- **Claude Sonnet** → para generación natural  
- **Titan Embeddings** → para knowledge base  
- **Bedrock Agent Core** → para contexto y orquestación con menor costo

**Ventajas:**
- Menos tokens de entrada
- Menos invocaciones al LLM
- Persistencia del estado sin reenviar todo el historial
- Gran reducción de costos

---

## 🔧 5. Backend: AppSync + Lambda

### AppSync (GraphQL)
Funciona como **gateway unificado**:

- Resuelve queries del widget (públicas usando API Key)
- Resuelve operaciones del admin (privadas usando JWT)
- Ejecuta resolvers VTL o Lambda
- Incluye validación del tenant mediante API key

### Lambdas por dominio
- `/catalog` → servicios y proveedores  
- `/availability` → horarios  
- `/booking` → creación y gestión de reservas  
- `/chat_agent` → FSM y/o invocación a IA  

**Recomendación:**
- Módulos pequeños, especializados, idempotentes

---

## 🗄️ 6. Base de Datos (DynamoDB)

**Tablas principales:**

- **Services**  
- **Providers**  
- **ProviderAvailability**  
- **Bookings**  
- **Conversations**  
- **TenantApiKeys**  
- **Tenants**  
- **TenantUsage**

**Claves multi-tenant:**
- PK = `TENANT#xxx`  
- SK = entidad  
- GSI según patrón de lectura

Documentación detallada:  
📄 `/docs/architecture/dynamodb-schema.md`

---

## 🎨 7. Widget Público

- Publicado en CloudFront (global)  
- Bundle UMD/IIFE  
- Expuesto como `window.ChatAgentWidget`  
- Configurable vía atributos `data-*`  
- Reporta eventos como:
  - `message:sent`
  - `slot:selected`
  - `booking:created`
- Permite personalización:
  - tema
  - idioma
  - mensaje de bienvenida
  - posición del botón

Detalles:  
📄 `/docs/widget/README.md`

---

## 🔒 8. Seguridad Arquitectónica

### API Keys (widget)
- almacenamiento hash  
- allowedOrigins obligatorio  
- rotación soportada  
- rate limiting por key/tenant

### JWT (admin)
- roles: owner/admin/viewer  
- claims del tenant  
- acceso estricto por GraphQL

### Aislamiento multi-tenant
- PK con `tenantId`  
- Lambdas verifican tenant por contexto  
- AppSync valida API key → tenantId

### Infraestructura
- CloudFormation + OIDC  
- IAM mínimo por Lambda  
- Logs anonimizados  
- TTL para datos sensibles (opcional)

Documentación completa:  
📄 `/docs/security/README.md`

---

## 📈 9. Escalabilidad y Límites Recomendados

### Límites sugeridos por tenant
- Servicios: 100  
- Profesionales: 50  
- Reservas por día: 10.000  
- Conversaciones activas: 5.000  

### Límites del sistema completo
- Scalamiento horizontal automático  
- DynamoDB soporta miles de tenants sin ajuste  
- Lambda escala automáticamente  
- AppSync soporta millones de requests/día  

### Multi-región (opcional)
- replicación cross-region para empresas con SLO altos  
- widget resiliente mediante CloudFront  

---

## 💵 10. Costos de arquitectura (resumen)

| Configuración | Costo mensual estimado |
|---------------|-------------------------|
| Sin IA | **USD 15–20** |
| IA con Haiku asistido | **USD 50–150** |
| IA conversacional completa | **USD 200–600** |

Más detalles:  
📄 `/docs/architecture/cost-estimation.md`

---

## 🔄 11. Trazabilidad del flujo

### Widget → AppSync → Lambda → DynamoDB → (opcional IA)

**Ejemplo de creación de reserva:**

1. El usuario selecciona servicio  
2. AppSync → Lambda `/availability`  
3. Lambda consulta disponibilidad en DynamoDB  
4. El agente confirma  
5. AppSync → Lambda `/booking`  
6. Se crea la reserva  
7. DynamoDB actualiza estado  
8. AppSync responde al widget  
9. El widget dispara `booking:created`

---

## 🧭 12. Roadmap Arquitectónico

- Integración completa de Bedrock Agent Core  
- Soporte para Google/Microsoft Calendar  
- API REST pública  
- Multi-branch por tenant  
- Canal WhatsApp/SMS opcional  
- Cache distribuido (DAX/ElastiCache) para tenants grandes

---

## 📚 Documentos relacionados

- `/docs/architecture/dynamodb-schema.md`  
- `/docs/architecture/appsync-schema.md`  
- `/docs/architecture/multi-tenant.md`  
- `/docs/security/README.md`  
- `/docs/widget/README.md`  
- `/docs/admin/README.md`
