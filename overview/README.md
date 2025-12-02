# SaaS Agentic Booking Chat — Overview

Este proyecto ofrece un sistema SaaS que permite a múltiples empresas integrar un **chat agéntico con reservas**, totalmente personalizable, dentro de sus sitios web mediante un simple snippet de JavaScript.

---

## 🎯 Objetivo del producto

Entregar a cualquier empresa la capacidad de:

- Mostrar un **chat inteligente** en su sitio web.
- Entender lo que el usuario quiere mediante lenguaje natural.
- Encontrar el servicio adecuado.
- Mostrar profesionales disponibles.
- Consultar horarios libres.
- Completar una **reserva end-to-end**.
- Registrar y mostrar la información en un panel administrativo.
- Personalizar el widget: colores, idioma, comportamiento.

Todo esto operando bajo un **modelo multi-tenant**, donde múltiples empresas usan el mismo backend, aisladas por `tenantId`.

---

## 🧩 Componentes principales

### 1. **Widget Web (JavaScript)**  
Código ligero que se embebe en cualquier sitio mediante:

```html
<script
  src="https://cdn.tu-saas.com/chat-widget.js"
  data-tenant-id="TENANT_ID"
  data-public-key="PUBLIC_KEY"
></script>
```

El widget:

- Muestra el chat.
- Envía mensajes al backend vía GraphQL.
- Construye el flujo conversacional.
- Maneja experiencias de branding e idioma.

### 2. **Panel Administrativo (Web App)**
Sitio protegido por Cognito donde cada empresa puede:

- Administrar servicios.
- Administrar profesionales.
- Definir horarios de disponibilidad.
- Gestionar API Keys.
- Configurar idioma, colores y comportamiento del widget.
- Ver reservas.
- Ver estadísticas de uso.

### 3. **Backend GraphQL (AWS AppSync)**
Provee la API central del SaaS.

Incluye:

- Queries para catálogo.
- Mutaciones para manejar conversación y reservas.
- Subscriptions (opcional) para chat en tiempo real.
- Mecanismos multi-tenant basados en `tenantId`.

### 4. **Lambdas Python (Lógica de negocio)**
Cada módulo ejecuta una parte del sistema:

- `chat_agent.py` → Lógica del agente y FSM.
- `booking.py` → Creación segura de reservas.
- `availability.py` → Generación de slots disponibles.
- `catalog.py` → Consultas de servicios y profesionales.

### 5. **Base de datos en DynamoDB (Multi-tenant)**
Tablas:

- `Tenants`
- `TenantApiKeys`
- `Services`
- `Providers`
- `ProviderAvailability`
- `Bookings`
- `ConversationState`

Cada tabla está particionada por `tenantId` para garantizar aislamiento lógico.

### 6. **AI opcional (NLP / Bedrock)**
Puede integrarse Amazon Bedrock u otros modelos para:

- Detección de intención.
- Clasificación de servicio.
- Construcción de diálogos más naturales.

---

## 🧱 Arquitectura general

```
Cliente Web → Widget → AppSync → Lambdas Python → DynamoDB
                                               ↘ AI (opcional)
```

---

## 💡 Beneficios clave

- **SaaS real**: múltiples empresas, una sola plataforma.
- **Fácil integración**: basta con un `<script>`.
- **Alto rendimiento**: 100% serverless.
- **Extensible**: se puede agregar AI, pagos, recordatorios, analytics.
- **Seguro**: API Keys por tenant con hash, rate-limiting, allowed origins.
- **Moderno**: Chat agéntico con lógica conversacional y FSM.

---

## 📍 Próximos pasos en la documentación

Este overview se complementa con los otros documentos:

- `/architecture/README.md`
- `/architecture/multi-tenant.md`
- `/widget/README.md`
- `/security/README.md`

Cada archivo provee instrucciones claras para que Copilot / Codex puedan construir automáticamente la arquitectura.
