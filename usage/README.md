# Guía de Uso del Agente Conversacional
SaaS Agentic Booking Chat

Este documento explica cómo funciona el flujo conversacional del agente, cómo se mantienen los estados, cómo se manejan los mensajes del usuario y cómo interactúa el agente con el backend (con o sin IA).

---

# 🎯 1. Objetivo del Agente Conversacional

El agente guía a los usuarios a reservar un servicio, seleccionando:

1) un tipo de servicio  
2) un profesional  
3) un horario disponible  
4) confirmación final  

El agente busca ser:
- rápido  
- claro  
- adaptable  
- configurable  
- capaz de operar en modo determinístico o con IA

---

# 🧠 2. Modos del Agente

El agente soporta **dos modos**, configurables por tenant:

---

## 2.1 Modo FSM (sin IA)

FSM = Finite State Machine  
El bot sigue una secuencia fija:

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

Características:
- Predecible  
- Muy barato (0 USD cost)  
- Ideal para FREE/PRO  

---

## 2.2 Modo IA (Bedrock Agent Core)

El agente usa IA para:
- entender lenguaje natural  
- interpretar intención  
- extraer entidades (servicio, fecha, hora)  
- delegar pasos a "tools" (Lambdas del backend)

Beneficios:
- Conversación natural  
- Menos fricción  
- Maneja ambigüedad  
- Capacidad de explicar opciones  

Flujo típico:

```
Mensaje usuario
→ Bedrock Agent analiza intención
→ si requiere acción → Tool (Lambda)
→ Agent genera respuesta
→ estado se actualiza
```

Modelos recomendados:

- Claude 3.5 Sonnet (generación)
- Claude Haiku (intención)
- Bedrock Agent Core (orquestación)

---

# 🔄 3. Flujo Conversacional Completo (Independiente del modo)

```
Usuario abre el widget
↓
Usuario escribe: "Quiero reservar un masaje"
↓
Backend detecta (modo FSM o IA)
↓
Agente responde: "¿Qué servicio deseas exactamente?"
↓
Usuario selecciona o escribe
↓
Backend consulta disponibilidad
↓
Usuario elige horario
↓
Se crea la reserva
↓
Agent confirma
```

---

# 🧩 4. Máquina de Estados (FSM)

## 4.1 Estados principales

| Estado | Descripción |
|--------|-------------|
| `INIT` | conversación recién iniciada |
| `SERVICE_PENDING` | agente pide servicio |
| `SERVICE_SELECTED` | servicio determinado |
| `PROVIDER_PENDING` | agente pide profesional |
| `PROVIDER_SELECTED` | profesional determinado |
| `SLOT_PENDING` | agente pide horario |
| `CONFIRM_PENDING` | agente pide confirmación |
| `BOOKING_CONFIRMED` | reserva realizada |

## 4.2 Transiciones

```
INIT → SERVICE_PENDING
SERVICE_PENDING → SERVICE_SELECTED
SERVICE_SELECTED → PROVIDER_PENDING
PROVIDER_PENDING → PROVIDER_SELECTED
PROVIDER_SELECTED → SLOT_PENDING
SLOT_PENDING → CONFIRM_PENDING
CONFIRM_PENDING → BOOKING_CONFIRMED
```

## 4.3 Estados persistentes

Los datos se guardan en DynamoDB (`Conversations`):

```
{
  conversationId,
  tenantId,
  state,
  serviceId,
  providerId,
  datetime,
  metadata,
  lastMessageAt
}
```

---

# 📡 5. Comportamiento del Agente

## 5.1 Si el usuario escribe en lenguaje natural

### Ejemplo:
> "Necesito una limpieza facial mañana a las 5pm"

FSM:
- intenta detectar entidad usando regex simples  
- si no entiende → pregunta explícita  
- "¿Qué servicio necesitas?"

IA:
- usa LLM Haiku/Sonnet para:
  - detectar "limpieza facial"
  - detectar "mañana"
  - detectar "17:00"
  - buscar disponibilidad automáticamente  

## 5.2 Si el usuario se salta pasos
Ejemplo:
> "El viernes con María"

FSM:
- detecta que falta servicio  
- responde: "Primero, ¿qué servicio deseas?"

IA:
- entiende contexto previo  
- reordena datos faltantes  
- puede inferir servicio si fue mencionado antes  

## 5.3 Si el usuario pregunta algo irrelevante
IA responde:
> "Puedo ayudarte a agendar una reserva. ¿Qué servicio necesitas?"

FSM:
> "¿Qué servicio deseas reservar?"

---

# 🔧 6. Tools (Lambdas) del Agente IA

Cuando opera con IA, cada acción se delega como "tool":

### Tool: `findServices`
```json
{
  "serviceName": "masaje"
}
```

### Tool: findProviders
```json
{
  "serviceId": "XYZ"
}
```

### Tool: findAvailability
```json
{
  "providerId": "ABC",
  "date": "2025-01-10"
}
```

### Tool: createBooking
```json
{
  "providerId": "...",
  "serviceId": "...",
  "datetime": "..."
}
```

Bedrock Agent Core decide cuándo llamar un tool.

---

# 🧪 7. Ejemplos de Conversación

## Ejemplo 1 — FSM

**Usuario:**
> Quiero un masaje

**Agente:**
> ¿Qué tipo de masaje deseas?

**Usuario:**
> Descontracturante

**Agente:**
> Perfecto. ¿Con qué profesional te gustaría?

...

## Ejemplo 2 — IA Completa

**Usuario:**
> Me gustaría agendarme con la doctora Martínez mañana después del almuerzo.

**IA detecta:**
- servicio: dermatología (si es su único servicio)
- profesional: Martínez
- fecha: mañana
- horario: rango "después del almuerzo"

Tool → disponibilidad

**IA responde:**
> Tengo el siguiente horario disponible mañana a las 15:30.
> ¿Lo confirmamos?

---

# ⚠️ 8. Manejo de Errores

| Caso | Respuesta |
|------|-----------|
| no hay horarios | "No tengo horarios disponibles ese día. ¿Quieres otra fecha?" |
| profesional inactivo | "Ese profesional no está disponible hoy. ¿Te muestro otras opciones?" |
| servicio no existe | "No encontré ese servicio. Los disponibles son…" |
| mensaje no entendible | "No logré comprender eso. ¿Qué servicio necesitas?" |
| backend falló | "Estamos teniendo problemas. Intenta de nuevo más tarde." |

---

# 🔄 9. Expiración de Conversación

- TTL configurable (10 min, 30 min, 24h)
- cuando expiró, se retorna a INIT

---

# 🧩 10. Personalización del Agente por Tenant

Cada tenant puede personalizar:

- primer mensaje
- tono (formal / casual / neutral)
- idioma
- quick replies sugeridas
- habilitar IA o no
- tiempo de expiración
- fallback phrases

Ejemplo en settings:

```javascript
AISettings {
  mode: "HAIKU" | "SONNET" | "FSM"
  style: "friendly"
  locale: "es"
  welcomeMessage: "¡Hola! Estoy para ayudarte a reservar."
}
```

---

# 🔒 11. Seguridad del Agente

- nunca incluye datos de otros tenants
- mensajes no se exponen como logs completos
- IA recibe prompts con PII truncada
- tools no pueden acceder a más datos que el tenant actual
- tokens IA se contabilizan por tenant

---

# 📈 12. Métricas del Agente

Por cada tenant se registra:

- mensajes enviados
- mensajes recibidos
- pasos completados exitosamente
- reservas generadas
- errores
- tokens IA consumidos

Todo esto alimenta TenantUsage.

---

# 🧭 13. Roadmap Conversacional

- soporte multistep reasoning
- integración con calendarios externos
- agent memory avanzada por tenant
- FAQs específicas por rubro
- plantillas de conversación por industria

---

# ✔️ Fin del archivo


