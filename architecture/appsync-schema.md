# GraphQL Schema — AppSync API

Este documento contiene el schema GraphQL completo para el backend multi-tenant del SaaS Agentic Booking Chat.

Está diseñado para ser implementado directamente en **AWS AppSync** o cualquier servidor GraphQL compatible.

---

## 📋 Schema completo

```graphql
# ===========================
# Scalars personalizados
# ===========================

scalar AWSDateTime
scalar AWSJSON

# ===========================
# Enums
# ===========================

enum BookingStatus {
  PENDING
  CONFIRMED
  CANCELLED
  NO_SHOW
}

enum PaymentStatus {
  NONE
  PENDING
  PAID
  FAILED
}

enum SenderType {
  USER
  AGENT
  SYSTEM
}

enum ConversationState {
  INIT
  SERVICE_PENDING
  SERVICE_SELECTED
  PROVIDER_PENDING
  PROVIDER_SELECTED
  SLOT_PENDING
  CONFIRM_PENDING
  BOOKING_CONFIRMED
}

# ===========================
# Types - Catálogo
# ===========================

type Service {
  id: ID!
  name: String!
  description: String
  category: String!
  durationMinutes: Int!
  price: Float
  active: Boolean!
}

type Provider {
  id: ID!
  name: String!
  bio: String
  services: [Service!]!
  timezone: String!
  active: Boolean!
}

# ===========================
# Types - Disponibilidad
# ===========================

type TimeSlot {
  providerId: ID!
  serviceId: ID!
  start: AWSDateTime!
  end: AWSDateTime!
  isAvailable: Boolean!
}

type AvailabilityRange {
  startTime: String!
  endTime: String!
}

type ProviderAvailability {
  providerId: ID!
  dayOfWeek: String!
  timeRanges: [AvailabilityRange!]!
  breaks: [AvailabilityRange!]
}

# ===========================
# Types - Reservas
# ===========================

type Booking {
  id: ID!
  serviceId: ID!
  providerId: ID!
  customerId: ID
  customerEmail: String
  customerPhone: String
  start: AWSDateTime!
  end: AWSDateTime!
  status: BookingStatus!
  paymentStatus: PaymentStatus!
  createdAt: AWSDateTime!
  service: Service
  provider: Provider
}

# ===========================
# Types - Chat
# ===========================

type Message {
  id: ID!
  sender: SenderType!
  text: String!
  createdAt: AWSDateTime!
  metadata: AWSJSON
}

type Conversation {
  id: ID!
  messages: [Message!]!
  state: ConversationState!
  updatedAt: AWSDateTime!
}

type ChatReply {
  conversationId: ID!
  message: Message!
  suggestedServices: [Service!]
  suggestedProviders: [Provider!]
  suggestedSlots: [TimeSlot!]
  booking: Booking
}

# ===========================
# Types - Tenant (Admin)
# ===========================

type Tenant {
  id: ID!
  name: String!
  slug: String!
  status: String!
  plan: String!
  settings: AWSJSON
  createdAt: AWSDateTime!
}

type TenantApiKey {
  id: ID!
  tenantId: ID!
  description: String
  status: String!
  allowedOrigins: [String!]
  createdAt: AWSDateTime!
  lastUsedAt: AWSDateTime
}

# ===========================
# Inputs - Chat
# ===========================

input SendChatMessageInput {
  conversationId: ID
  text: String!
  userContext: UserContextInput
}

input UserContextInput {
  userId: ID
  name: String
  email: String
  phone: String
}

# ===========================
# Inputs - Disponibilidad
# ===========================

input GetAvailableSlotsInput {
  serviceId: ID!
  providerId: ID!
  from: AWSDateTime!
  to: AWSDateTime!
}

# ===========================
# Inputs - Reservas
# ===========================

input CreateBookingInput {
  serviceId: ID!
  providerId: ID!
  start: AWSDateTime!
  customerId: ID
  customerEmail: String
  customerName: String
  customerPhone: String
}

input CancelBookingInput {
  bookingId: ID!
  reason: String
}

# ===========================
# Inputs - Admin
# ===========================

input CreateServiceInput {
  name: String!
  description: String
  category: String!
  durationMinutes: Int!
  price: Float
}

input UpdateServiceInput {
  serviceId: ID!
  name: String
  description: String
  category: String
  durationMinutes: Int
  price: Float
  active: Boolean
}

input CreateProviderInput {
  name: String!
  bio: String
  serviceIds: [ID!]!
  timezone: String!
}

input UpdateProviderInput {
  providerId: ID!
  name: String
  bio: String
  serviceIds: [ID!]
  timezone: String
  active: Boolean
}

input SetAvailabilityInput {
  providerId: ID!
  dayOfWeek: String!
  timeRanges: [AvailabilityRangeInput!]!
  breaks: [AvailabilityRangeInput!]
}

input AvailabilityRangeInput {
  startTime: String!
  endTime: String!
}

# ===========================
# Queries
# ===========================

type Query {
  # --- Catálogo público (widget) ---
  searchServices(text: String): [Service!]!
  getService(serviceId: ID!): Service
  listProvidersByService(serviceId: ID!): [Provider!]!
  
  # --- Disponibilidad (widget) ---
  getAvailableSlots(input: GetAvailableSlotsInput!): [TimeSlot!]!
  
  # --- Reservas (widget y admin) ---
  getBooking(bookingId: ID!): Booking
  getBookingsByUser(customerId: ID!): [Booking!]!
  
  # --- Conversaciones (widget) ---
  getConversation(conversationId: ID!): Conversation
  
  # --- Admin panel ---
  getTenant: Tenant
  listServices: [Service!]!
  listProviders: [Provider!]!
  listBookings(from: AWSDateTime, to: AWSDateTime, status: BookingStatus): [Booking!]!
  listApiKeys: [TenantApiKey!]!
}

# ===========================
# Mutations
# ===========================

type Mutation {
  # --- Chat (widget) ---
  sendChatMessage(input: SendChatMessageInput!): ChatReply!
  
  # --- Reservas (widget) ---
  createBooking(input: CreateBookingInput!): Booking!
  cancelBooking(input: CancelBookingInput!): Booking!
  
  # --- Admin: Servicios ---
  createService(input: CreateServiceInput!): Service!
  updateService(input: UpdateServiceInput!): Service!
  deleteService(serviceId: ID!): Service!
  
  # --- Admin: Profesionales ---
  createProvider(input: CreateProviderInput!): Provider!
  updateProvider(input: UpdateProviderInput!): Provider!
  deleteProvider(providerId: ID!): Provider!
  
  # --- Admin: Disponibilidad ---
  setProviderAvailability(input: SetAvailabilityInput!): ProviderAvailability!
  
  # --- Admin: API Keys ---
  createApiKey(description: String): TenantApiKey!
  revokeApiKey(apiKeyId: ID!): TenantApiKey!
  
  # --- Admin: Tenant settings ---
  updateTenantSettings(settings: AWSJSON!): Tenant!
}

# ===========================
# Subscriptions (opcional)
# ===========================

type Subscription {
  # Escuchar nuevos mensajes en una conversación
  onMessageAdded(conversationId: ID!): Message!
    @aws_subscribe(mutations: ["sendChatMessage"])
  
  # Escuchar cambios en reservas
  onBookingUpdated(customerId: ID!): Booking!
    @aws_subscribe(mutations: ["createBooking", "cancelBooking"])
}
```

---

## 🔐 Autorización en AppSync

### 1. **Widget público** (API Key)

Operaciones permitidas:
- `searchServices`
- `listProvidersByService`
- `getAvailableSlots`
- `sendChatMessage`
- `createBooking`
- `cancelBooking`
- `getBooking`
- `getBookingsByUser`
- `getConversation`

Configuración en AppSync:
```json
{
  "authorizationType": "API_KEY"
}
```

El resolver debe:
1. Leer `x-api-key` del header
2. Buscar en `TenantApiKeys` (via hash)
3. Obtener `tenantId`
4. Pasar `tenantId` a la Lambda

---

### 2. **Panel Admin** (Cognito User Pools)

Operaciones permitidas:
- Todas las operaciones admin (`createService`, `updateProvider`, etc.)
- `getTenant`
- `listServices`
- `listProviders`
- `listBookings`
- `listApiKeys`

Configuración:
```json
{
  "authorizationType": "AMAZON_COGNITO_USER_POOLS"
}
```

El JWT debe incluir:
```json
{
  "custom:tenantId": "andina",
  "cognito:groups": ["admin", "staff"]
}
```

---

## 🧩 Resolvers (Lambda)

Cada query/mutation invoca una Lambda Python.

### Ejemplo de resolver para `sendChatMessage`:

**Data Source**: Lambda `chat_agent`

**Request Mapping Template (VTL)**:
```vtl
{
  "version": "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "field": "sendChatMessage",
    "tenantId": $ctx.stash.tenantId,
    "input": $util.toJson($ctx.args.input),
    "identity": $util.toJson($ctx.identity)
  }
}
```

**Response Mapping Template**:
```vtl
$util.toJson($ctx.result)
```

---

## 📦 Patrones de uso

### Widget: Enviar mensaje y recibir sugerencias

```graphql
mutation SendMessage {
  sendChatMessage(input: {
    conversationId: "conv_123"
    text: "Necesito un masaje"
    userContext: {
      userId: "user_ext_001"
      name: "Juan Pérez"
    }
  }) {
    conversationId
    message {
      id
      text
      sender
    }
    suggestedServices {
      id
      name
      durationMinutes
      price
    }
  }
}
```

### Widget: Consultar slots disponibles

```graphql
query GetSlots {
  getAvailableSlots(input: {
    serviceId: "svc_123"
    providerId: "pro_55"
    from: "2025-12-01T00:00:00Z"
    to: "2025-12-07T23:59:59Z"
  }) {
    start
    end
    isAvailable
  }
}
```

### Widget: Crear reserva

```graphql
mutation CreateBooking {
  createBooking(input: {
    serviceId: "svc_123"
    providerId: "pro_55"
    start: "2025-12-01T17:30:00Z"
    customerEmail: "juan@example.com"
    customerName: "Juan Pérez"
    customerPhone: "+56912345678"
  }) {
    id
    status
    paymentStatus
    start
    end
    service {
      name
    }
    provider {
      name
    }
  }
}
```

### Admin: Crear servicio

```graphql
mutation CreateService {
  createService(input: {
    name: "Masaje relajante"
    description: "Masaje de 60 minutos"
    category: "masajes"
    durationMinutes: 60
    price: 25000
  }) {
    id
    name
    active
  }
}
```

---

## 🧪 Testing con GraphQL Playground

Puedes probar el schema localmente usando:

- **Apollo Server** (Node.js)
- **Strawberry** (Python)
- **AppSync Local** (SAM CLI)

Ejemplo con Apollo Server:

```javascript
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema.graphql');
const resolvers = require('./resolvers');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => ({
    tenantId: req.headers['x-tenant-id'],
    apiKey: req.headers['x-api-key'],
  }),
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

---

## 📚 Documentos relacionados

- `/architecture/dynamodb-schema.md`
- `/architecture/lambdas.md`
- `/architecture/multi-tenant.md`
- `/security/README.md`
