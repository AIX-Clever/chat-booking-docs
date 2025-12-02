# Diseño de Lambdas Python — Backend del Chat Agéntico

Este documento describe la arquitectura de las **Lambdas Python** que implementan la lógica de negocio del SaaS Agentic Booking Chat.

Está diseñado para que herramientas como Codex puedan generar automáticamente la estructura de código.

---

## 🧩 Lambdas principales

| Lambda | Propósito | Trigger |
|--------|-----------|---------|
| `chat_agent` | FSM conversacional, orquestación del agente | AppSync resolver |
| `catalog` | Consultas de servicios y profesionales | AppSync resolver |
| `availability` | Cálculo de slots disponibles | AppSync resolver |
| `booking` | Creación y cancelación de reservas | AppSync resolver |
| `auth_resolver` | Resolución de `tenantId` desde API Key | AppSync authorizer |

---

## 📁 Estructura de carpetas recomendada

```
/lambda
  /chat_agent
    handler.py          # Entry point Lambda
    fsm.py              # Finite State Machine
    states.py           # Definición de estados
    nlp.py              # NLP/AI opcional (Bedrock)
    responses.py        # Plantillas de respuesta
    requirements.txt
  
  /catalog
    handler.py
    services.py         # Lógica de servicios
    providers.py        # Lógica de profesionales
    requirements.txt
  
  /availability
    handler.py
    slots.py            # Generación de slots
    calendar.py         # Lógica de calendarios
    requirements.txt
  
  /booking
    handler.py
    create.py           # Crear reserva
    cancel.py           # Cancelar reserva
    validate.py         # Validaciones
    requirements.txt
  
  /auth_resolver
    handler.py
    api_keys.py         # Resolver API Key → tenantId
    requirements.txt
  
  /shared
    dynamodb.py         # Helper DynamoDB
    utils.py            # Utilidades comunes
    models.py           # Modelos de datos
```

---

## 🔧 1. Lambda: `chat_agent`

### Propósito
Implementar la máquina de estados conversacional que guía al usuario desde la intención inicial hasta la reserva confirmada.

### Input (desde AppSync)
```json
{
  "field": "sendChatMessage",
  "tenantId": "andina",
  "input": {
    "conversationId": "conv_123",
    "text": "Necesito un masaje",
    "userContext": {
      "userId": "user_ext_001",
      "name": "Juan Pérez"
    }
  }
}
```

### Output
```json
{
  "conversationId": "conv_123",
  "message": {
    "id": "msg_456",
    "sender": "AGENT",
    "text": "Perfecto, ¿qué tipo de masaje buscas?",
    "createdAt": "2025-12-01T15:30:00Z"
  },
  "suggestedServices": [
    {
      "id": "svc_123",
      "name": "Masaje descontracturante",
      "durationMinutes": 60,
      "price": 25000
    }
  ],
  "suggestedProviders": null,
  "suggestedSlots": null,
  "booking": null
}
```

### Lógica interna (FSM)

Estados:
- `INIT`
- `SERVICE_PENDING`
- `SERVICE_SELECTED`
- `PROVIDER_PENDING`
- `PROVIDER_SELECTED`
- `SLOT_PENDING`
- `CONFIRM_PENDING`
- `BOOKING_CONFIRMED`

Flujo:
1. Leer estado actual desde `ConversationState` (DynamoDB)
2. Analizar mensaje del usuario
3. Determinar transición de estado
4. Ejecutar acción correspondiente:
   - Buscar servicios
   - Listar profesionales
   - Consultar slots
   - Crear reserva
5. Actualizar estado en DynamoDB
6. Retornar respuesta estructurada

### Módulos internos

#### `handler.py`
```python
import json
import boto3
from fsm import ConversationFSM
from nlp import analyze_intent

def lambda_handler(event, context):
    tenant_id = event['tenantId']
    input_data = event['input']
    
    fsm = ConversationFSM(tenant_id)
    
    result = fsm.process_message(
        conversation_id=input_data.get('conversationId'),
        text=input_data['text'],
        user_context=input_data.get('userContext')
    )
    
    return result
```

#### `fsm.py`
```python
from states import State, INIT, SERVICE_PENDING, SERVICE_SELECTED
from catalog import search_services
from availability import get_available_slots
from booking import create_booking

class ConversationFSM:
    def __init__(self, tenant_id):
        self.tenant_id = tenant_id
        self.state_handlers = {
            'INIT': self.handle_init,
            'SERVICE_PENDING': self.handle_service_pending,
            'SERVICE_SELECTED': self.handle_service_selected,
            # ... otros estados
        }
    
    def process_message(self, conversation_id, text, user_context):
        # Leer estado actual
        current_state = self.load_state(conversation_id)
        
        # Procesar según estado
        handler = self.state_handlers.get(current_state['state'])
        result = handler(current_state, text, user_context)
        
        # Guardar nuevo estado
        self.save_state(conversation_id, result['new_state'])
        
        return result['response']
    
    def handle_init(self, state, text, user_context):
        # Detectar intención
        intent = analyze_intent(text)
        
        if intent == 'book_service':
            services = search_services(self.tenant_id, text)
            return {
                'new_state': {'state': 'SERVICE_PENDING'},
                'response': {
                    'message': {'text': '¿Cuál de estos servicios te interesa?'},
                    'suggestedServices': services
                }
            }
```

#### `nlp.py` (opcional, con Bedrock)
```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime')

def analyze_intent(text):
    prompt = f"""Analiza la siguiente solicitud y clasifícala:
    - "book_service" si quiere reservar algo
    - "query" si solo pregunta
    - "other" si no está claro
    
    Texto: "{text}"
    
    Responde solo con la categoría."""
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'prompt': prompt,
            'max_tokens': 10,
            'temperature': 0
        })
    )
    
    result = json.loads(response['body'].read())
    return result['completion'].strip()

def extract_service_keywords(text):
    # Extraer palabras clave relacionadas con servicios
    pass
```

---

## 🗂️ 2. Lambda: `catalog`

### Propósito
Consultar servicios y profesionales del catálogo del tenant.

### Operaciones
- `searchServices(tenantId, query)` → Lista de servicios
- `listProvidersByService(tenantId, serviceId)` → Lista de profesionales

### Ejemplo: `services.py`
```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
services_table = dynamodb.Table('Services')

def search_services(tenant_id, query=None):
    """Busca servicios del tenant, opcionalmente filtrando por query"""
    
    response = services_table.query(
        KeyConditionExpression=Key('tenantId').eq(tenant_id)
    )
    
    services = response['Items']
    
    if query:
        # Filtrado simple por nombre/descripción
        query_lower = query.lower()
        services = [
            s for s in services 
            if query_lower in s['name'].lower() or 
               query_lower in s.get('description', '').lower()
        ]
    
    return [s for s in services if s.get('active', True)]

def get_service(tenant_id, service_id):
    """Obtener un servicio específico"""
    response = services_table.get_item(
        Key={'tenantId': tenant_id, 'serviceId': service_id}
    )
    return response.get('Item')
```

---

## 📅 3. Lambda: `availability`

### Propósito
Calcular slots disponibles para un servicio, profesional y rango de fechas.

### Lógica
1. Leer disponibilidad base del profesional (`ProviderAvailability`)
2. Leer duración del servicio
3. Generar slots candidatos
4. Filtrar slots ya ocupados (`Bookings`)
5. Retornar slots disponibles

### Ejemplo: `slots.py`
```python
import boto3
from datetime import datetime, timedelta
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
availability_table = dynamodb.Table('ProviderAvailability')
bookings_table = dynamodb.Table('Bookings')

def get_available_slots(tenant_id, service_id, provider_id, from_date, to_date, duration_minutes):
    """Genera slots disponibles"""
    
    # 1. Leer disponibilidad base
    availability = load_provider_availability(tenant_id, provider_id)
    
    # 2. Generar slots candidatos
    candidate_slots = generate_candidate_slots(
        availability, from_date, to_date, duration_minutes
    )
    
    # 3. Filtrar ocupados
    booked_slots = load_bookings(tenant_id, provider_id, from_date, to_date)
    available_slots = filter_available(candidate_slots, booked_slots)
    
    return available_slots

def generate_candidate_slots(availability, from_date, to_date, duration_minutes):
    """Genera slots cada X minutos según disponibilidad"""
    slots = []
    current_date = from_date
    
    while current_date < to_date:
        day_of_week = current_date.strftime('%a').upper()  # MON, TUE, etc.
        
        if day_of_week in availability:
            for time_range in availability[day_of_week]['timeRanges']:
                start_time = datetime.combine(
                    current_date.date(),
                    datetime.strptime(time_range['startTime'], '%H:%M').time()
                )
                end_time = datetime.combine(
                    current_date.date(),
                    datetime.strptime(time_range['endTime'], '%H:%M').time()
                )
                
                current_slot = start_time
                while current_slot + timedelta(minutes=duration_minutes) <= end_time:
                    slots.append({
                        'start': current_slot.isoformat(),
                        'end': (current_slot + timedelta(minutes=duration_minutes)).isoformat()
                    })
                    current_slot += timedelta(minutes=15)  # Intervalo de slots
        
        current_date += timedelta(days=1)
    
    return slots
```

---

## 📝 4. Lambda: `booking`

### Propósito
Crear y cancelar reservas de forma segura, evitando overbooking.

### Operación crítica: `create_booking`

```python
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
bookings_table = dynamodb.Table('Bookings')

def create_booking(tenant_id, service_id, provider_id, start, end, customer_info):
    """Crea una reserva con condición atómica para evitar overbooking"""
    
    booking_id = generate_booking_id()
    
    item = {
        'PK': f"{tenant_id}#{provider_id}",
        'SK': start,
        'bookingId': booking_id,
        'tenantId': tenant_id,
        'serviceId': service_id,
        'providerId': provider_id,
        'customerId': customer_info.get('customerId'),
        'customerEmail': customer_info.get('email'),
        'customerName': customer_info.get('name'),
        'customerPhone': customer_info.get('phone'),
        'endTime': end,
        'status': 'PENDING',
        'paymentStatus': 'NONE',
        'createdAt': datetime.utcnow().isoformat()
    }
    
    try:
        bookings_table.put_item(
            Item=item,
            ConditionExpression='attribute_not_exists(PK) AND attribute_not_exists(SK)'
        )
        return item
    
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            raise Exception('Slot no longer available')
        raise
```

---

## 🔐 5. Lambda: `auth_resolver`

### Propósito
Resolver `tenantId` desde una API Key en los headers.

### Flujo
1. Recibir `x-api-key` desde AppSync
2. Calcular hash
3. Buscar en `TenantApiKeys` (GSI por hash)
4. Validar estado, origenes, rate limit
5. Retornar `tenantId`

### Ejemplo
```python
import hashlib
import boto3

dynamodb = boto3.resource('dynamodb')
api_keys_table = dynamodb.Table('TenantApiKeys')

def resolve_tenant_from_api_key(api_key, origin):
    """Resuelve tenantId desde API Key"""
    
    # Hash de la key
    key_hash = hashlib.sha256(api_key.encode()).hexdigest()
    
    # Buscar en GSI
    response = api_keys_table.query(
        IndexName='GSI1',
        KeyConditionExpression=Key('apiKeyHash').eq(key_hash)
    )
    
    if not response['Items']:
        raise Exception('Invalid API Key')
    
    key_data = response['Items'][0]
    
    # Validaciones
    if key_data['status'] != 'ACTIVE':
        raise Exception('API Key is not active')
    
    if origin not in key_data.get('allowedOrigins', []):
        raise Exception('Origin not allowed')
    
    # TODO: Rate limiting check
    
    return key_data['tenantId']
```

---

## 📦 Shared Layer

Crear un **Lambda Layer** con código común:

```
/shared
  dynamodb.py
  utils.py
  models.py
  constants.py
```

### `dynamodb.py`
```python
import boto3

class DynamoDBHelper:
    def __init__(self, table_name):
        self.table = boto3.resource('dynamodb').Table(table_name)
    
    def get_item(self, key):
        response = self.table.get_item(Key=key)
        return response.get('Item')
    
    def query(self, key_condition, **kwargs):
        response = self.table.query(
            KeyConditionExpression=key_condition,
            **kwargs
        )
        return response['Items']
```

---

## 🧪 Testing

Cada Lambda debe tener tests unitarios:

```python
# tests/test_chat_agent.py
import pytest
from chat_agent.fsm import ConversationFSM

def test_init_state():
    fsm = ConversationFSM('test_tenant')
    result = fsm.process_message(None, 'Necesito un masaje', {})
    assert result['message']['sender'] == 'AGENT'
    assert len(result['suggestedServices']) > 0
```

---

## 📚 Documentos relacionados

- `/architecture/dynamodb-schema.md`
- `/architecture/appsync-schema.md`
- `/usage/README.md`
- `/deployment/README.md`
