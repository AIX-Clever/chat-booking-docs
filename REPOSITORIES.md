# Estructura de Repositorios

El proyecto ha migrado de una estructura monolítica a una arquitectura desacoplada con múltiples repositorios bajo la organización `AIX-Clever`.

## Repositorios Principales

### 1. `chat-booking-docs`
*   **Propósito**: Documentación central del proyecto.
*   **Contenido**: Arquitectura, guías de usuario, manuales de integración, diagramas.
*   **Stack**: Markdown.

### 2. `chat-booking-backend`
*   **Propósito**: Core de la lógica de negocio y API.
*   **Contenido**: Lambdas (Python), Esquema GraphQL, Lógica de FSM, base de datos.
*   **Infraestructura**: Contiene su propio CDK en `/infra` para desplegar: AppSync, DynamoDB, Lambdas, Cognito.
*   **Dependencias**: Ninguna.

### 3. `chat-booking-landing`
*   **Propósito**: Landing page comercial del producto (Lucia).
*   **Contenido**: HTML/CSS/JS estático.
*   **Infraestructura**: Contiene su propio CDK en `/infra` para desplegar: S3 + CloudFront (HTTPS).

### 4. `chat-booking-widget`
*   **Propósito**: El widget de chat embebible para clientes finales.
*   **Contenido**: React + Vite (Web Components).
*   **Infraestructura**: Contiene su propio CDK en `/infra` para desplegar: S3 + CloudFront (con CORS habilitado).

### 5. `chat-booking-admin`
*   **Propósito**: Dashboard SaaS para dueños de negocio (tenants).
*   **Contenido**: Next.js (App Router).
*   **Infraestructura**: Contiene su propio CDK en `/infra` para desplegar: S3 + CloudFront (Static Export).

### 6. `chat-booking-infrastructure` (Legacy / Orchestration)
*   **Propósito**: Histórico o para orquestación de alto nivel (DNS global, VPCs compartidas si las hubiera).
*   **Estado**: La infraestructura específica de cada app se ha movido a los repositorios respectivos.

### 7. `chat-booking-tests`
*   **Propósito**: Tests E2E y de integración cross-service.
*   **Contenido**: Playwright/Cypress.

## Flujo de Trabajo (CI/CD)

Cada repositorio de aplicación (`backend`, `landing`, `widget`, `admin`) tiene su propio pipeline de GitHub Actions en `.github/workflows/deploy.yml`.

*   **Trigger**: Push a `dev`, `qa`, `prod`.
*   **Auth**: OIDC con AWS.
*   **Build**: Compilación local (ej: `npm run build`).
*   **Deploy**: `cdk deploy` desde la carpeta `/infra`.

Esto permite despliegues independientes: un cambio en el Landing no requiere redesplegar el Backend.
