# Esquema GraphQL — AppSync  
SaaS Agentic Booking Chat

Este documento describe el **schema GraphQL** utilizado por AppSync para servir:

- **1) API pública del widget** (reservas, disponibilidad, mensajes)  
- **2) API privada del panel admin** (gestión de servicios, providers, settings, API keys, usage)

La arquitectura es 100% multi-tenant.  
El tenant se determina por API Key (público) o JWT (admin).

---

# 🧠 1. Filosofía del diseño GraphQL

1. **Se separan claramente las operaciones públicas vs privadas.**  
2. **Todo resolver valida tenantId** antes de acceder a DynamoDB.  
3. **Los inputs nunca incluyen tenantId**, este se obtiene del contexto del request.  
4. **El schema está modularizado** por dominio.  
5. **Se permite paginación estándar (`nextToken`)**.  
6. **El widget solo tiene acceso a lectura + createBooking**.  
7. **El panel admin tiene CRUD completo**.  
8. **El agente IA usa las mismas queries internas** (modo tools).

---

# 🧩 2. Estructura general del schema

```graphql
schema {
  query: Query
  mutation: Mutation
}
```

Se organiza por módulos:

- Services
- Providers
- Availability
- Bookings
- Conversations
- Tenants (admin)
- API Keys (admin)
- Usage (admin)
- Agent (mensajes del usuario)

---

# 📥 3. Queries públicas (Widget)

Estas operaciones son accesibles vía **API Key + AllowedOrigins**:

```graphql
type Query {
  listServices: [Service!]!
  listProviders(serviceId: ID!): [Provider!]!
  getAvailability(providerId: ID!, date: AWSDate!): [AvailabilitySlot!]!
  getConversation(conversationId: ID!): Conversation
  health: String!
}
```

## Detalles

### listServices
- Devuelve servicios activos del tenant
- Orden alfabético básico

### listProviders(serviceId)
- Lista profesionales que atienden ese servicio

### getAvailability(providerId, date)
- Construye slots dinámicamente combinando:
  - reglas recurrentes
  - excepciones
  - reservas existentes

### getConversation
- Estado persistente del flujo del chat

### health
- Verifica que la API está operativa

---

# 🧾 4. Mutations públicas (Widget)

```graphql
type Mutation {
  sendAgentMessage(conversationId: ID, text: String!): AgentResponse!
  createBooking(input: CreateBookingInput!): Booking!
}
```

## 4.1 sendAgentMessage

- Envía un mensaje al agente (FSM o IA).
- Devuelve mensaje del agente + estado actualizado.

## 4.2 createBooking

- Solo permite crear reservas dentro del tenant actual.
- Valida:
  - disponibilidad
  - provider activo
  - servicio válido

---

# 🔒 5. Queries privadas (Panel Admin)

Requieren JWT de Cognito con tenantId y role.

```graphql
extend type Query {
  adminListServices: [Service!]!
  adminListProviders: [Provider!]!
  adminGetTenant: Tenant!
  adminGetUsage(period: String!): TenantUsage
  adminListApiKeys: [ApiKey!]!
  adminListBookings(date: AWSDate): [Booking!]!
}
```

---

# 🔐 6. Mutations privadas (Panel Admin)

```graphql
extend type Mutation {
  adminCreateService(input: AdminServiceInput!): Service!
  adminUpdateService(input: AdminServiceUpdateInput!): Service!
  adminDeleteService(serviceId: ID!): Boolean!

  adminCreateProvider(input: AdminProviderInput!): Provider!
  adminUpdateProvider(input: AdminProviderUpdateInput!): Provider!
  adminDeleteProvider(providerId: ID!): Boolean!

  adminCreateApiKey: ApiKey!
  adminRevokeApiKey(apiKeyId: ID!): Boolean!

  adminUpdateTenantSettings(input: TenantSettingsInput!): Tenant!
}
```

---

# 🏗️ 7. Tipos principales

## 7.1 Service

```graphql
type Service {
  serviceId: ID!
  name: String!
  description: String
  category: String
  durationMinutes: Int!
  price: Float
  active: Boolean!
}
```

## 7.2 Provider

```graphql
type Provider {
  providerId: ID!
  name: String!
  bio: String
  serviceIds: [ID!]!
  active: Boolean!
}
```

## 7.3 AvailabilitySlot

```graphql
type AvailabilitySlot {
  time: AWSTime!
  available: Boolean!
}
```

## 7.4 Booking

```graphql
type Booking {
  bookingId: ID!
  serviceId: ID!
  providerId: ID!
  datetime: AWSDateTime!
  status: BookingStatus!
  customerName: String!
  customerEmail: String
}
```

## 7.5 Conversation

```graphql
type Conversation {
  conversationId: ID!
  state: AgentState!
  serviceId: ID
  providerId: ID
  datetime: AWSDateTime
  lastMessageAt: AWSDateTime!
}
```

## 7.6 AgentResponse

```graphql
type AgentResponse {
  conversation: Conversation!
  messages: [AgentMessage!]!
}
```

## 7.7 AgentMessage

```graphql
type AgentMessage {
  sender: String!  # "user" | "agent"
  text: String!
  delayMs: Int     # opcional para UX
}
```

---

# 🧠 8. Tipos Admin

## 8.1 Tenant

```graphql
type Tenant {
  tenantId: ID!
  name: String!
  country: String
  timezone: String
  plan: TenantPlan!
  status: TenantStatus!
  settings: TenantSettings
}
```

## 8.2 ApiKey

```graphql
type ApiKey {
  apiKeyId: ID!
  publicKey: String!
  allowedOrigins: [String!]!
  status: ApiKeyStatus!
  createdAt: AWSDateTime!
  lastUsedAt: AWSDateTime
}
```

## 8.3 TenantUsage

```graphql
type TenantUsage {
  period: String!
  messages: Int!
  bookings: Int!
  tokensIA: Int!
  errors: Int!
  plan: TenantPlan!
}
```

---

# 📥 9. Inputs

Ejemplo:

```graphql
input CreateBookingInput {
  serviceId: ID!
  providerId: ID!
  datetime: AWSDateTime!
  customerName: String!
  customerEmail: String
}

input AdminProviderInput {
  name: String!
  bio: String
  serviceIds: [ID!]!
  active: Boolean
}
```

---

# 🔐 10. Autorización por resolver

## Público (API Key)
- se valida `x-api-key`
- se valida `origin`
- AppSync pasa `tenantId`

## Privado (JWT)
- require grupo/rol
- AppSync pasa `tenantId` desde claims

---

# 🛠️ 11. Resolvers

Tipos de resolvers usados:

### Lambda resolver (principal)

### Pipeline resolver para booking:
1. **Step 1:** validar provider/servicio
2. **Step 2:** validar disponibilidad
3. **Step 3:** crear booking
4. **Step 4:** enviar notificación futura (opcional)

---

# 🧪 12. Testing del Schema

- `amplify mock api` para entorno local
- Jest tests para validaciones
- Tests de integración Lambda+GraphQL
- Snapshot testing para el schema

---

# 📈 13. Roadmap

- suscripciones GraphQL (notificaciones realtime)
- schema modular interno usando GraphQL codegen
- federation (si se agrega microservicio externo)
- versionado del schema
- mejor soporte para IA (intenciones y entidades)

---

# ✔️ Fin del archivo


