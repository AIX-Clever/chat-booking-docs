# Panel Administrativo — SaaS Agentic Booking Chat

Panel web donde cada empresa (tenant) gestiona servicios, profesionales, reservas, configuraciones del widget, API keys y modos de IA.

El panel está diseñado para ser **simple**, **rápido**, **multi-tenant**, y completamente desacoplado del widget público.

---

## 🧭 1. Objetivo del Panel

Permitir que cada empresa configure todo lo necesario para operar el sistema:

1. Crear servicios que ofrece  
2. Crear profesionales  
3. Configurar disponibilidad semanal  
4. Definir excepciones (feriados)  
5. Gestionar reservas  
6. Configurar el chat (branding, idioma, IA)  
7. Administrar API keys y allowedOrigins  
8. Gestionar usuarios internos del tenant (owner/admin/viewer)

El panel corresponde al "Backoffice" del SaaS.

---

## 🚀 2. Onboarding de un Tenant (primeros 10 minutos)

El onboarding debe aparecer como una guía en la primera sesión del usuario.

### Paso 1 — Crear Servicios  
Ejemplo: "Masaje Relajación", "Consulta Dermatológica", "Clases de Yoga".

### Paso 2 — Registrar Profesionales  
Indicar:
- nombre  
- bio  
- servicios que atiende  
- zona horaria  

### Paso 3 — Configurar Disponibilidad  
- horarios semanales por profesional  
- excepciones (feriados/vacaciones)

### Paso 4 — Generar API Key  
- copiar snippet de integración  
- definir allowedOrigins  
- validar conectividad

### Paso 5 — Personalizar el Widget  
- color  
- idioma  
- mensaje de bienvenida  
- posición  
- habilitar IA (si el plan lo permite)

### Paso 6 — Probar la conversación  
Abrir el widget en un entorno de prueba.

---

## 🧩 3. Roles de Usuario

Los roles se almacenan en Cognito como claims.

| Rol | Permisos | Uso típico |
|-----|----------|------------|
| **Owner** | Todo, incluyendo facturación, usuarios, API keys | Dueño del tenant |
| **Admin** | Gestión operativa: servicios, disponibilidad, reservas | Administradores |
| **Viewer** | Lectura de reservas/servicios | Personal de apoyo |

---

## 📂 4. Secciones del Panel

El menú principal recomendado es:

1. **Dashboard**
2. **Servicios**
3. **Profesionales**
4. **Disponibilidad**
5. **Reservas**
6. **Widget & Branding**
7. **Configuración de IA**
8. **API Keys & Seguridad**
9. **Usuarios del Tenant**
10. **Uso & Métricas**

A continuación, cada sección con su funcionalidad y contratos de API.

---

## 📊 5. Dashboard

Información de negocio:

- reservas del día / semana / mes  
- servicios más utilizados  
- profesionales más agendados  
- uso del widget  
- errores más frecuentes  
- límites del plan y consumo  

> Los datos se obtienen desde `TenantUsage` o consultas agregadas en DynamoDB.

---

## 🧾 6. Servicios

### 6.1 Funcionalidad

- Crear servicio  
- Editar  
- Activar/desactivar  
- Eliminar (soft delete)

### 6.2 Campos

- `name`  
- `description`  
- `durationMinutes`  
- `category` (opcional)  
- `price` (opcional)  
- `active`

### 6.3 GraphQL

```graphql
type Query {
  adminListServices: [Service!]!
  adminGetService(id: ID!): Service
}

type Mutation {
  adminCreateService(input: AdminCreateServiceInput!): Service!
  adminUpdateService(input: AdminUpdateServiceInput!): Service!
  adminDeleteService(id: ID!): Boolean!
}
```

El backend obtiene `tenantId` del JWT. No se envía en los inputs.

---

## 👩‍⚕️ 7. Profesionales

### 7.1 Funcionalidad

- Crear profesional
- Asignar servicios
- Editar datos
- Activar/desactivar
- Asignar zona horaria (por profesional)

### 7.2 GraphQL

```graphql
type Query {
  adminListProviders: [Provider!]!
  adminGetProvider(id: ID!): Provider
}

type Mutation {
  adminCreateProvider(input: AdminCreateProviderInput!): Provider!
  adminUpdateProvider(input: AdminUpdateProviderInput!): Provider!
  adminDeleteProvider(id: ID!): Boolean!
}
```

---

## 🗓 8. Disponibilidad

### 8.1 Funcionalidad

- disponibilidad recurrente (Lunes–Domingo)
- múltiples ventanas horarias por día
- excepciones
- manejo de timezones
- preview del calendario por profesional

### 8.2 GraphQL

```graphql
type Mutation {
  adminSetProviderAvailability(input: AdminSetProviderAvailabilityInput!): Boolean!
}
```

---

## 📅 9. Reservas

### 9.1 Operaciones

- listar
- ver detalle
- cancelar
- re-agendar (futuro)
- filtrar por profesional, servicio, rango de fecha, estado

### 9.2 GraphQL

```graphql
type Query {
  adminListBookings(filter: AdminListBookingsFilter): [Booking!]!
  adminGetBooking(id: ID!): Booking
}

type Mutation {
  adminCancelBooking(id: ID!): Booking!
}
```

---

## 🎨 10. Widget & Branding

**Configuración soportada:**

- color primario
- posición del botón (izquierda/derecha)
- idioma
- mensaje de bienvenida
- auto-open
- logo opcional
- plan-dependent: soporte para temas avanzados

**GraphQL:**

```graphql
type Mutation {
  adminUpdateWidgetSettings(input: WidgetSettingsInput!): TenantSettings!
}
```

---

## 🤖 11. Configuración de IA

Cada tenant puede activar/desactivar IA según su plan.

### Modos soportados

| Modo | Descripción | Costo | Plan |
|------|-------------|-------|------|
| FSM | Conversación determinística | 0 USD | FREE/PRO |
| NLP asistido | Haiku para intención y entidades | Bajo | PRO/BUSINESS |
| IA completa | Bedrock Agent Core + Sonnet | Medio/Alto | BUSINESS/ENTERPRISE |

**GraphQL:**

```graphql
type Mutation {
  adminUpdateAISettings(input: AISettingsInput!): TenantSettings!
}
```

---

## 🔑 12. API Keys & Seguridad

### Funcionalidad

- crear API key
- editar allowedOrigins
- rotar keys
- revocar keys
- ver última vez utilizada

### Seguridad

- las keys se almacenan como hash
- nunca se muestra una key completa después de creada
- allowedOrigins obligatorio
- rate limiting por tenant y por key

### GraphQL

```graphql
type Mutation {
  adminCreateApiKey(input: AdminCreateApiKeyInput!): TenantApiKey!
  adminUpdateApiKey(input: AdminUpdateApiKeyInput!): TenantApiKey!
  adminRevokeApiKey(id: ID!): Boolean!
}
```

---

## 👤 13. Usuarios del Tenant

**Permite:**

- invitar usuarios
- asignar roles
- desactivar usuarios
- ver actividad reciente

Se recomienda integrar Cognito Hosted UI o un IdP corporativo en planes Enterprise.

---

## 📈 14. Uso & Métricas

**Visión del consumo:**

- mensajes del widget
- reservas creadas
- tokens consumidos por IA
- límites según plan
- peaks de tráfico

Estos datos se almacenan por día/mes en `TenantUsage`.

---

## 🛑 15. Errores frecuentes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| ORIGIN_NOT_ALLOWED | dominio no está en allowedOrigins | actualizar API key |
| AUTH_FAILED | API key inválida/revocada | generar nueva |
| reserva no aparece | profesional sin disponibilidad | revisar disponibilidad |
| widget no carga | CSP del sitio bloquea script | permitir cdn.tu-saas.com |

---

## 🧭 16. Roadmap del Panel

- gestión de sucursales (multi-branch)
- etiquetas para servicios
- métricas de conversión del widget
- editor visual de flujos para el agente
- plantillas de disponibilidad

---

## 📚 Relacionado

- `/docs/architecture/appsync-schema.md`
- `/docs/architecture/dynamodb-schema.md`
- `/docs/security/README.md`
- `/docs/widget/README.md`
