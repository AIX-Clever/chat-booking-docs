# SaaS Agentic Booking Chat — Plan de Proyecto

Este directorio contiene toda la documentación técnica necesaria para implementar un sistema SaaS de chat agéntico con sistema de reservas embebible.

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

## 🎯 Uso de esta documentación

Esta documentación está diseñada para:

1. **Desarrolladores humanos** — entender la arquitectura completa
2. **AI Copilots (GitHub Copilot, Cursor, Codex)** — generar código automáticamente
3. **Herramientas IaC** — provisionar infraestructura (CloudFormation, CDK, Terraform)
4. **Equipos de producto** — comprender funcionalidades y limitaciones

## 🚀 Inicio rápido

1. Lee `/overview/README.md` para entender el producto
2. Revisa `/architecture/README.md` para la visión técnica
3. Consulta `/architecture/multi-tenant.md` para el modelo SaaS
4. Implementa según `/deployment/README.md`

## 📖 Orden de lectura recomendado

Para implementadores:
1. Overview
2. Architecture → multi-tenant
3. Architecture → dynamodb-schema
4. Architecture → appsync-schema
5. Architecture → lambdas
6. Widget → README
7. Deployment → README

Para integradores (clientes del SaaS):
1. Widget → README
2. Widget → embedding-guide
3. Widget → api-reference

## 💡 Contribuciones

Cada archivo está diseñado para ser autocontenido y completo, permitiendo que herramientas de IA puedan generar implementaciones correctas sin ambigüedades.
