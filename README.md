# SaaS Agentic Booking Chat  
Plataforma SaaS multi-tenant para reservas asistida por un agente conversacional inteligente

Este proyecto ofrece un **chat agentico de nueva generación**, fácil de integrar en cualquier sitio web y diseñado para ayudar a usuarios a **encontrar servicios**, **seleccionar profesionales** y **confirmar reservas**, todo mediante una conversación natural.

---

## 🎯 ¿Para quién es este producto?

Este SaaS está diseñado para empresas que agendan horas:

- Clínicas médicas, dentales o estéticas  
- Salones de belleza, barberías y spa  
- Gimnasios y entrenadores personales  
- Servicios profesionales (coaching, consultoría, terapias)  
- Centros de estudio, tutorías, clases particulares  
- Cualquier negocio que necesite reservas rápidas desde su web

---

## 💡 Beneficios

- **Incrementa reservas** → menos fricción para el usuario  
- **Trabaja 24/7** → atención fuera de horario  
- **Reduce carga operativa** → menos llamadas, WhatsApp o recepción  
- **Integración en 2 minutos** → un solo `<script>`  
- **Multi-tenant** → ideal para un SaaS comercial  
- **Configurable** → branding, idioma, disponibilidad, servicios  
- **Modo IA opcional** → experiencia avanzada conversacional  

---

## 🚀 Integración ultrarrápida

El cliente final solo necesita agregar esto en su sitio:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="TENANT_ID"
        data-public-key="PUBLIC_KEY"></script>
```

Una vez agregado:

1. Aparece un botón flotante
2. El usuario abre el chat
3. El agente guía la conversación
4. Los horarios se consultan en tiempo real
5. Se crea la reserva

---

## 🧠 Modos del Agente

El agente puede funcionar en dos modos, dependiendo del plan del tenant:

### 1. FSM Determinístico (sin IA)

- Conversación estructurada
- Bajo costo (prácticamente 0 USD)
- Perfecto para planes FREE / PRO

### 2. IA Conversacional (Bedrock Agent Core + LLM)

- Interpretación de intención
- Extracción de entidades automatizada
- Conversación flexible y natural
- Ideal para planes BUSINESS / ENTERPRISE

El sistema permite activar/desactivar IA por tenant.

---

## 🏗️ Arquitectura General

```
Cliente ─▶ Widget (JS) ─▶ AppSync (GraphQL)
                               │
                               ▼
                     Lambdas (Python)
                               │
                               ▼
                        DynamoDB (Multi-Tenant)
```

**Características:**

- 100% serverless
- Multi-tenant nativo
- Costos muy bajos
- Fácil escalabilidad
- API interna y pública bien separadas
- Integración rápida con Bedrock Agent Core (opcional)

---

## 🔧 Decisiones clave de arquitectura (ADR-lite)

**AppSync en vez de REST:**  
GraphQL encaja mejor para sugerencias dinámicas (servicios → profesionales → horarios).

**DynamoDB vs RDS:**  
Multi-tenant natural, con aislamiento a nivel de PK, costo extremadamente bajo.

**FSM + IA opcional en vez de LLM-only:**  
Control total del flujo; IA solo agrega naturalidad, pero el flujo no depende de ella.

**Widget embebible via CloudFront:**  
Permite integración universal, caching global y protección mediante allowedOrigins.

**CloudFormation + GitHub Actions (OIDC):**  
Despliegues sin claves IAM y CI/CD moderno.

---

## 💸 Costos operativos (estimados)

| Modo del sistema | Costo mensual estimado |
|------------------|------------------------|
| Solo FSM (sin IA) | USD 15–20 |
| IA asistida (intención + entidades) | USD 50–150 |
| IA conversacional completa | USD 200–600 |

- La infraestructura AWS base (AppSync, Lambda, DynamoDB, CloudFront, Cognito) cuesta menos de USD 20/mes.
- La IA con Bedrock se escala según tokens utilizados.

---

## 📦 Componentes del repositorio

```
/docs
  /widget
  /admin
  /architecture
  /security
  /deployment
  /usage

/infrastructure
  /cloudformation
    master-stack.yaml
    /nested-stacks

/lambda
  chat_agent/
  booking/
  availability/
  catalog/

/widget
  src/
  build/

/scripts
  create_tenant.py
  create_api_key.py
```

---

## 🧩 Flujo de trabajo del usuario final

1. Abre el chat en el sitio del cliente
2. Dice lo que necesita: *"Quiero un masaje mañana en la tarde"*
3. El agente:
   - Identifica el servicio
   - Propone profesionales
   - Sugiere horarios
   - Confirma reserva
4. El usuario recibe detalle de su cita
5. El panel admin muestra las reservas en tiempo real

---

## 🧭 Panel Administrativo

Los tenants pueden:

- Crear servicios
- Agregar profesionales
- Configurar disponibilidad
- Generar API keys
- Definir allowedOrigins
- Personalizar widget
- Ver reservas
- Controlar consumo y plan

**Documentación completa:**  
📄 `/docs/admin/README.md`

---

## 🧪 Testing

- Tests unitarios Lambda (pytest)
- Testing de FSM del agente
- DynamoDB Local
- Tests de integración del widget (Jest + Playwright)

---

## 🔄 CI/CD

- GitHub Actions + OIDC
- Deploy automático del widget (S3 + CloudFront)
- Deploy del backend (CloudFormation)
- Promociones a QA/PROD con approvals

**Documentación:**  
📄 `/docs/deployment/pipeline-ci-cd.md`

---

## 🧭 Roadmap

- Integración completa Bedrock Agent Core
- Calendarios Google/Microsoft
- API pública REST
- Multi-tenant avanzado con sub-branch offices
- Analytics enriquecido
- Plantillas de conversación por rubro

---

## 📚 Estructura de la documentación

### `/overview` — Visión general
- Objetivos del proyecto
- Componentes principales
- Beneficios clave

### `/architecture` — Arquitectura técnica
- Diseño de arquitectura AWS
- Estrategia multi-tenant
- Esquemas DynamoDB
- Schema GraphQL (AppSync)
- Diseño de Lambdas Python

### `/widget` — Widget embebible
- Guía de integración
- Guía de embedding en diferentes plataformas
- API JavaScript de referencia

### `/admin` — Panel administrativo
- Funcionalidades del panel
- Gestión de servicios y profesionales
- Configuración del widget

### `/security` — Seguridad
- Modelo de seguridad SaaS
- Gestión de API Keys
- Aislamiento multi-tenant

### `/usage` — Casos de uso
- Flujos de negocio
- FSM del agente conversacional
- Diagramas de secuencia

### `/deployment` — Despliegue
- Instrucciones de deployment en AWS
- Pipeline CI/CD
- Configuración de CDN

---

## 🚀 Inicio rápido

1. Lee `/overview/README.md` para entender el producto
2. Revisa `/architecture/README.md` para la visión técnica
3. Consulta `/architecture/multi-tenant.md` para el modelo SaaS
4. Implementa según `/deployment/README.md`

---

## 📖 Orden de lectura recomendado

**Para implementadores:**
1. Overview
2. Architecture → multi-tenant
3. Architecture → dynamodb-schema
4. Architecture → appsync-schema
5. Architecture → lambdas
6. Widget → README
7. Deployment → README

**Para integradores (clientes del SaaS):**
1. Widget → README
2. Widget → embedding-guide
3. Widget → api-reference

---

## 💡 Contribuciones

Cada archivo está diseñado para ser autocontenido y completo, permitiendo que herramientas de IA puedan generar implementaciones correctas sin ambigüedades.
