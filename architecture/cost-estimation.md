# Estimación de Costos — SaaS Agentic Booking Chat

Este documento resume los costos mensuales estimados de operar la plataforma en AWS, considerando los distintos modos del agente (FSM / IA asistida / IA completa), la infraestructura multi-tenant y los niveles de escala.

Todos los valores son aproximados y se basan en precios de AWS en 2024–2025.

---

# 🏗️ 1. Resumen general de costos

El backend serverless de este SaaS es **extremadamente económico**.

| Modo del agente | Costo total mensual estimado |
|------------------|------------------------------|
| **FSM (sin IA)** | **USD 15–20** |
| **IA asistida (Haiku / NLP)** | **USD 50–150** |
| **IA completa (Sonnet / Agent Core)** | **USD 200–600** |

---

# 🧱 2. Desglose de costos por componente

Los componentes fijos (backend sin IA) cuestan:

| Servicio | Costo mensual (estimado) |
|----------|---------------------------|
| AppSync | $2–4 |
| Lambda | $0.20–1 |
| DynamoDB | $1–3 |
| CloudFront | $8–20 |
| S3 (storage widget) | $0.05–0.30 |
| Cognito | $0 (hasta 50k MAU) |
| CloudWatch Logs | $1–3 |

🎯 **Total infra fija:**  
➡️ **USD ~15–20 / mes**  
(para 50–100 tenants de tamaño pequeño/mediano)

---

# 🤖 3. Costos de IA (Bedrock)

El costo de IA depende del:

- modelo utilizado  
- tokens input/output  
- cantidad de mensajes procesados  
- uso o no de Bedrock Agent Core  
- eficiencia del prompting  

## 3.1 Costos por modelo (referencial)

### Claude 3.5 Sonnet
- Input: ~$0.003 / 1K tokens  
- Output: ~$0.015 / 1K tokens  

### Claude Haiku
- Input: ~$0.00025 / 1K tokens  
- Output: ~$0.00125 / 1K tokens  

### Llama 3.1 70B (si se habilita)
- Input/output: más barato que Sonnet, similar a Haiku/Sonnet mix  

### Titan Embeddings
- ~0.0001 / 1K tokens  
- insignificante en costo  

---

# 🧠 4. Cálculo de IA por flujo conversacional

## 4.1 Costo aproximado por mensaje (modo IA completa)

Supongamos promedio:
- Input: 300 tokens  
- Output: 150 tokens  

Costo por mensaje:
- Input: 0.3k × 0.003 = 0.0009  
- Output: 0.15k × 0.015 = 0.00225  
= **~0.00315 USD por mensaje**

Si hay 500.000 mensajes/mes:
➡️ **~USD 1,575 / mes**

## 4.2 Con Bedrock Agent Core (optimizado)

Agent Core reduce:
- hasta **70–90%** de tokens  
- hasta **50–80%** de llamadas  

Costo real por mensaje:
- **0.0004–0.0015 USD**

Para 500.000 mensajes/mes:
➡️ **USD 80–300 / mes**

Esto transforma IA completa en algo **sostenible** económicamente.

---

# 🧮 5. Escenarios completos

## 5.1 Escenario Pequeño (50 tenants)
- 500k requests AppSync  
- 100 GB CloudFront  
- IA opcional

| Modo | Costo total |
|-------|--------------|
| FSM | $15–20 |
| IA asistida | $40–70 |
| IA completa (Agent Core) | $80–150 |

---

## 5.2 Escenario Mediano (300 tenants)
- 2M requests AppSync  
- 500 GB CloudFront  
- IA moderada (Haiku)

| Modo | Costo total |
|-------|--------------|
| FSM | $40–60 |
| IA asistida | $100–180 |
| IA completa | $250–400 |

---

## 5.3 Escenario Grande (2500 tenants)
- 10M requests  
- 2 TB CloudFront  
- IA completa con Agent Core

| Modo | Costo total |
|-------|--------------|
| FSM | $150–220 |
| IA asistida | $400–800 |
| IA completa | $900–1500 |

---

# 💰 6. Costo por Tenant

En promedio:

| Plan | Costo real para ti | Precio recomendado |
|------|-----------------------|----------------------|
| LITE | < $0.10 | $0 (Trial) -> $9 (Base) |
| PRO | ~$0.30–$1 | $49 |
| BUSINESS | $2–$8 | $149 |
| ENTERPRISE | $10–$40 | $299–499 |

Margen por tenant:
- 95%–99% en PRO  
- 90% en BUSINESS  
- 85–95% en ENTERPRISE  

---

# 📦 7. Costo por Reserva (Booking)

Costo backend por reserva:
- DynamoDB R/W: ~$0.000002  
- Lambda: ~$0.00001  
- AppSync: ~$0.000004  

Total:
➡️ **$0.000016 por reserva**  
(0.0016 centavos)

Incluso si un tenant procesa 10k reservas al mes:
➡️ **~USD 0.16**

---

# 🧠 8. Costo por Conversación

Depende del modo:

### FSM:
→ ~USD 0.000001 por interacción  
(casi cero)

### IA asistida:
→ ~USD 0.0001–0.0002

### IA completa (optimizada):
→ ~USD 0.0004–0.0015

---

# 📉 9. Costo marginal por nuevos tenants

Agregar un nuevo tenant cuesta:

➡️ **~USD 0.01 / mes**  
(No hay crecimiento lineal, es amortizado por la infraestructura.)

Por esto este modelo SaaS escala extremadamente bien.

---

# ⚖️ 10. Comparación vs hosting tradicional

| Arquitectura | Costo | Escalabilidad |
|--------------|--------|----------------|
| EC2 / Docker | Medio–alto | manual |
| Kubernetes | Alto | alto pero caro |
| Serverless (este SaaS) | **Muy bajo** | **automática** |

El modelo actual es de los más costo-efectivos de AWS.

---

# 🔮 11. Roadmap de Optimización de Costos

- caching AppSync resolvers  
- DAX para tablas de alto tráfico  
- IA condicional: Sonnet solo cuando es necesario  
- throttling por tenant para evitar abusos  
- colas SQS para smoothing de tráfico  
- optimización de prompts IA  
- agent memory compacta  
- análisis de logs con TTL corto  

---

# ✔️ Fin del archivo
