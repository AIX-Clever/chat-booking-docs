# CI/CD & Deployment — SaaS Agentic Booking Chat

Este documento describe el proceso completo de **despliegue**, **versionado**, **promociones entre entornos**, y **automatización CI/CD** para la plataforma.

La arquitectura contempla:

- Widget embebible (React) → desplegado en **CloudFront + S3**
- Backend serverless → desplegado con **CloudFormation**
- Multi-entorno: **dev**, **qa**, **prod**
- Integración segura con **GitHub Actions (OIDC)**

---

## 🏗️ 1. Estructura de Despliegue

La plataforma se divide en 2 artefactos principales:

### 1. Widget (Frontend público)
- Construido con React + MUI
- Empaquetado en UMD/IIFE
- Publicado en:
  - `s3://<bucket-widget>/<version>/chat-widget.js`
  - `s3://<bucket-widget>/latest/chat-widget.js`
- Distribuido por CloudFront
- Invalidación automática al desplegar

### 2. Backend (Infraestructura & Lambdas)
Incluye:
- AppSync GraphQL API
- Lambdas en Python
- DynamoDB (tablas multi-tenant)
- IAM roles
- SQS/EventBridge (si aplica)
- API Keys
- Cognito (admin panel)

Desplegado con:
- `./cloudformation/deploy.sh <environment>`

---

## 🔐 2. Autenticación segura con GitHub Actions (OIDC)

Se evita completamente el uso de claves IAM en GitHub.

### Beneficios:
- cero secretos estáticos  
- permisos temporales  
- rotación automática  
- cumplimiento modern best practices  

### Requisitos:
- Rol IAM con `sts:AssumeRoleWithWebIdentity`
- Identity Provider: `token.actions.githubusercontent.com`

Ejemplo de trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::<ACCOUNT>:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## 🔄 3. Flujo de CI/CD Completo

```
Push to GitHub
 ├─▶ Lint + Test + Build
 ├─▶ Build Widget
 │     ├─▶ Upload to S3
 │     └─▶ CloudFront Invalidation
 ├─▶ Build Backend (CloudFormation)
 │     └─▶ Deploy to Dev
 └─▶ Manual Approval → Deploy to QA/PROD
```

**Pasos:**

1. **Linting & Tests**
   - Jest (widget)
   - Pytest (lambdas)
   - ESLint

2. **Build del Widget**
   - `npm run build:widget`
   - Artefacto UMD/IIFE generado en `/widget/build`

3. **Upload a S3**
   - `aws s3 sync build/ s3://widget-bucket/<version>/`
   - actualizar `latest/` según política de release

4. **Invalidación de CloudFront**
   - se invalida `/latest/*`

5. **Deploy del Backend**
   - `./cloudformation/deploy.sh <env>`

6. **Promoción QA/PROD**
   - requiere aprobación manual
   - backend y widget via pipeline

---

## 🪄 4. Versionado del Widget

### 4.1 Tagging Semántico

Cada release incluye:

- `v1.0.0` (versión exacta)
- `latest` (etiqueta móvil)
- `canary` (para pruebas de clientes específicos)

### 4.2 Rutas en S3

```
s3://widget-bucket/v1.0.0/chat-widget.js
s3://widget-bucket/v1.1.0/chat-widget.js
s3://widget-bucket/latest/chat-widget.js
s3://widget-bucket/canary/chat-widget.js
```

### 4.3 Usos en producción

**Tenant típico usa:**

```html
<script src="https://cdn.../chat-widget/latest/chat-widget.js">
```

**Cliente enterprise puede usar un release fijo:**

```html
<script src="https://cdn.../chat-widget/v1.0.0/chat-widget.js">
```

---

## 📦 5. Versionado del Backend

Se recomienda:

- versionar la API con graphql-schema versionado en Git
- usar `cloudformation validate` para detectar errores
- incluir CHANGELOG.md para breaking changes

**Estrategias:**

**A) Deploy directo (desarrollo)**
- rápido
- ideal para dev

**B) Deploy azul/verde (producción)**
- duplicar stack con un sufijo (-blue, -green)
- cambiar alias DNS o output de AppSync
- eliminar stack anterior luego de validar

---

## 🧩 6. Múltiples entornos: Dev / QA / Prod

### Recomendación

| Entorno | Propósito | Qué incluye |
|---------|-----------|-------------|
| Dev | desarrollo, testing manual | DynamoDB dev, Lambdas dev, widget dev |
| QA | validación interna | datos semilla, release candidate |
| Prod | clientes reales | multi-tenant productivo |

### Autorización de promociones

- Dev → (auto)
- QA → (manual approval)
- Prod → (manual approval + checklist)

---

## 📜 7. Rollback seguro

### Widget

1. cambiar `latest` → version anterior
2. invalidar CloudFront
3. listo (rollback instantáneo)

### Backend

**Opciones:**

- `aws cloudformation update-stack --use-previous-template`
- mantener stacks paralelos (blue/green)
- activar flag `DISABLE_IA` si la IA falla (safety)

---

## 🛠️ 8. Desarrollo local (Local Dev)

### Lambda Local

Usar AWS SAM:

```bash
sam build
sam local invoke ChatAgentFunction -e events/test.json
```

### AppSync Local (Mock)

```bash
amplify mock
```

### DynamoDB Local

```bash
docker run -p 8000:8000 amazon/dynamodb-local
```

### Widget Local

```bash
cd widget
npm install
npm start
```

---

## 🌐 9. Despliegue multi-tenant

**No se despliega un backend por tenant.**  
Un solo backend sirve a todos los tenants.

Durante despliegues:

- no se afectan API keys existentes
- los tenants siguen operando sin downtime
- AppSync permite múltiples versiones de resolvers

---

## 🔧 10. Infraestructura como Código (CloudFormation)

Recomendación:

Usar **CloudFormation con nested stacks** para backend:

- tablas DynamoDB
- roles IAM
- Lambdas
- API GraphQL AppSync
- conexiones a Bedrock

Scripts de deployment:

```bash
# Validar templates
./cloudformation/validate.sh

# Deploy a dev
./cloudformation/deploy.sh dev

# Deploy a prod
./cloudformation/deploy.sh prod

# Teardown
./cloudformation/teardown.sh dev
```

---

## 📈 11. Monitoring & Observability

Se monitorean:

- latencia AppSync
- error rate
- tokens IA por tenant
- DynamoDB throttles
- Lambda concurrency
- CloudFront cache hit ratio

**Alertas recomendadas:**

- reserva fallida
- excesos de tokens Bedrock
- respuestas lentas (>2s)
- tendencia a throttling
- tenant sobrepasando su plan

---

## 🏁 12. Roadmap de CI/CD

- release channels automáticos (alpha, beta, stable)
- integración con Sentry
- pipeline para Bedrock Agents (export/import)
- pruebas visuales del widget
- smoke tests post-deploy

---

## 📚 Documentos relacionados

- `/docs/deployment/README.md`
- `/docs/architecture/README.md`
- `/docs/security/README.md`
