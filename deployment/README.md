# Deployment — Guía de Despliegue en AWS

Esta guía describe cómo desplegar el SaaS Agentic Booking Chat completo en AWS.

---

## 🎯 Prerequisitos

### Herramientas necesarias

- **AWS CLI** v2.x
- **AWS CDK** (recomendado) o **Serverless Framework**
- **Python** 3.9+
- **Node.js** 16+
- **Git**
- **Docker** (para Lambda layers)

### Cuenta AWS

- Cuenta AWS activa
- Permisos de administrador (o IAM con permisos suficientes)
- Región recomendada: `us-east-1` o `sa-east-1`

---

## 📦 Arquitectura de despliegue

```
Infraestructura AWS:
- CloudFront (CDN para widget)
- S3 (hosting widget estático)
- AppSync (GraphQL API)
- Lambda (Python 3.9)
- DynamoDB (7 tablas)
- Cognito User Pool
- CloudWatch Logs
- Secrets Manager
- Route53 (DNS)
- Certificate Manager (SSL)
```

---

## 🚀 Opción 1: Despliegue con AWS CDK

### Paso 1: Clonar repositorio

```bash
git clone https://github.com/tu-org/chat-booking-saas.git
cd chat-booking-saas
```

### Paso 2: Instalar dependencias

```bash
# CDK
npm install -g aws-cdk
npm install

# Python dependencies para Lambdas
cd lambda
pip install -r requirements.txt
cd ..
```

### Paso 3: Configurar ambiente

```bash
# Archivo .env o usar AWS Secrets Manager
export AWS_REGION=us-east-1
export DOMAIN_NAME=tu-saas.com
export STAGE=prod
```

### Paso 4: Bootstrap CDK (primera vez)

```bash
cdk bootstrap aws://ACCOUNT-ID/us-east-1
```

### Paso 5: Sintetizar stack

```bash
cdk synth
```

Revisa el CloudFormation generado.

### Paso 6: Desplegar

```bash
# Deploy completo
cdk deploy --all

# O por stacks individuales
cdk deploy ChatBookingSaaSStack-Database
cdk deploy ChatBookingSaaSStack-API
cdk deploy ChatBookingSaaSStack-Auth
cdk deploy ChatBookingSaaSStack-Widget
```

### Paso 7: Configurar DNS

```bash
# Obtener CloudFront distribution
aws cloudformation describe-stacks \
  --stack-name ChatBookingSaaSStack-Widget \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDomain`].OutputValue' \
  --output text

# Crear CNAME en Route53
cdn.tu-saas.com -> d1234abcd.cloudfront.net
```

---

## 🏗️ Opción 2: Despliegue con Serverless Framework

### Paso 1: Instalar Serverless

```bash
npm install -g serverless
```

### Paso 2: Configurar `serverless.yml`

```yaml
service: chat-booking-saas

provider:
  name: aws
  runtime: python3.9
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  
  environment:
    STAGE: ${self:provider.stage}
    REGION: ${self:provider.region}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
          Resource: "arn:aws:dynamodb:${self:provider.region}:*:table/*"

functions:
  chatAgent:
    handler: lambda/chat_agent/handler.lambda_handler
    events:
      - appsync:
          name: sendChatMessage
          dataSource: chatAgentDataSource
  
  booking:
    handler: lambda/booking/handler.lambda_handler
  
  catalog:
    handler: lambda/catalog/handler.lambda_handler
  
  availability:
    handler: lambda/availability/handler.lambda_handler

resources:
  Resources:
    # DynamoDB Tables
    TenantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.stage}-Tenants
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: tenantId
            AttributeType: S
        KeySchema:
          - AttributeName: tenantId
            KeyType: HASH
    
    # (más tablas...)

plugins:
  - serverless-appsync-plugin
  - serverless-python-requirements
```

### Paso 3: Desplegar

```bash
serverless deploy --stage prod
```

---

## 🗃️ Crear tablas DynamoDB

### Opción A: Con CDK

```typescript
// lib/database-stack.ts
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class DatabaseStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Tabla Tenants
    const tenantsTable = new dynamodb.Table(this, 'Tenants', {
      tableName: 'Tenants',
      partitionKey: { name: 'tenantId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED,
    });

    tenantsTable.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'ownerUserId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'createdAt', type: dynamodb.AttributeType.STRING },
    });

    // Tabla Services
    const servicesTable = new dynamodb.Table(this, 'Services', {
      tableName: 'Services',
      partitionKey: { name: 'tenantId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'serviceId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    });

    // Más tablas...
  }
}
```

### Opción B: Con CLI

```bash
# Crear tabla Tenants
aws dynamodb create-table \
  --table-name Tenants \
  --attribute-definitions \
    AttributeName=tenantId,AttributeType=S \
    AttributeName=ownerUserId,AttributeType=S \
    AttributeName=createdAt,AttributeType=S \
  --key-schema AttributeName=tenantId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes \
    '[{
      "IndexName": "GSI1",
      "KeySchema": [
        {"AttributeName":"ownerUserId","KeyType":"HASH"},
        {"AttributeName":"createdAt","KeyType":"RANGE"}
      ],
      "Projection": {"ProjectionType":"ALL"}
    }]'

# Más tablas...
```

---

## 🔐 Configurar Cognito

### Con CDK

```typescript
import * as cognito from 'aws-cdk-lib/aws-cognito';

const userPool = new cognito.UserPool(this, 'AdminUserPool', {
  userPoolName: 'chat-booking-admins',
  selfSignUpEnabled: false,
  signInAliases: {
    email: true,
  },
  passwordPolicy: {
    minLength: 12,
    requireLowercase: true,
    requireUppercase: true,
    requireDigits: true,
    requireSymbols: true,
  },
  mfa: cognito.Mfa.OPTIONAL,
  mfaSecondFactor: {
    sms: true,
    otp: true,
  },
});

const userPoolClient = new cognito.UserPoolClient(this, 'AdminClient', {
  userPool,
  generateSecret: false,
  authFlows: {
    userPassword: true,
    userSrp: true,
  },
});
```

### Con CLI

```bash
aws cognito-idp create-user-pool \
  --pool-name chat-booking-admins \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 12,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": true
    }
  }' \
  --mfa-configuration OPTIONAL \
  --auto-verified-attributes email
```

---

## 🔗 Desplegar AppSync API

### Schema GraphQL

Sube el schema completo desde `/architecture/appsync-schema.md`

### Crear Data Sources

```bash
# Lambda data source para chat_agent
aws appsync create-data-source \
  --api-id YOUR_API_ID \
  --name chatAgentDataSource \
  --type AWS_LAMBDA \
  --lambda-config lambdaFunctionArn=arn:aws:lambda:us-east-1:ACCOUNT:function:chat_agent
```

### Crear Resolvers

Ejemplo para `sendChatMessage`:

**Request mapping template**:
```vtl
{
  "version": "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "field": "sendChatMessage",
    "tenantId": $ctx.stash.tenantId,
    "input": $util.toJson($ctx.args.input)
  }
}
```

**Response mapping template**:
```vtl
$util.toJson($ctx.result)
```

---

## 🌐 Desplegar Widget en CloudFront

### Paso 1: Build del widget

```bash
cd widget
npm install
npm run build

# Genera carpeta /dist con:
# - chat-widget.js
# - chat-widget.css
# - assets/
```

### Paso 2: Subir a S3

```bash
aws s3 sync ./dist s3://chat-booking-widget-prod/ \
  --cache-control "public, max-age=31536000, immutable"
```

### Paso 3: Crear CloudFront Distribution

```bash
aws cloudfront create-distribution \
  --origin-domain-name chat-booking-widget-prod.s3.amazonaws.com \
  --default-root-object index.html \
  --default-cache-behavior '{
    "TargetOriginId": "S3-chat-widget",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }'
```

### Paso 4: Configurar CNAME

```bash
# En Route53
cdn.tu-saas.com -> CloudFront distribution
```

---

## 🧪 Verificar despliegue

### Test de conectividad

```bash
# Probar AppSync
curl -X POST https://YOUR_API_ID.appsync-api.us-east-1.amazonaws.com/graphql \
  -H "x-api-key: YOUR_API_KEY" \
  -d '{"query": "query { __typename }"}'

# Probar widget
curl https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js
```

### Test end-to-end

1. Crear tenant de prueba en DynamoDB
2. Crear API Key para ese tenant
3. Insertar widget en página HTML de prueba
4. Abrir chat y enviar mensaje
5. Verificar logs en CloudWatch

---

## 📊 Monitoreo post-deployment

### CloudWatch Alarms

```bash
# Alarm para errores en Lambda
aws cloudwatch put-metric-alarm \
  --alarm-name chat-agent-errors \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=chat_agent
```

### Dashboard

Crear dashboard con:
- Lambda invocations y errores
- AppSync requests y latencia
- DynamoDB consumed capacity
- CloudFront requests y cache hit rate

---

## 🔄 Rollback

### Con CDK

```bash
# Rollback automático si falla
cdk deploy --rollback

# Rollback manual
cdk destroy --force
# Luego redeploy versión anterior
```

### Con CloudFormation

```bash
aws cloudformation cancel-update-stack --stack-name ChatBookingSaaSStack
```

---

## 🧹 Limpieza (desarrollo)

```bash
# Eliminar stacks
cdk destroy --all

# Eliminar buckets S3
aws s3 rb s3://chat-booking-widget-prod --force

# Eliminar tablas DynamoDB
aws dynamodb delete-table --table-name Tenants
# (repetir para todas las tablas)
```

---

## 📚 Documentos relacionados

- [Pipeline CI/CD](/deployment/pipeline-ci-cd.md)
- [Arquitectura](/architecture/README.md)
- [Seguridad](/security/README.md)
