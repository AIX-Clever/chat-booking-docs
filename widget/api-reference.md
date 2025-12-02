# Widget API JavaScript — Referencia Completa

Esta es la documentación completa de la API JavaScript del widget.

---

## 🚀 Inicialización

### `ChatAgentWidget.init(config)`

Inicializa el widget programáticamente.

#### Parámetros

```typescript
interface WidgetConfig {
  // Requeridos
  tenantId: string;
  publicKey: string;
  
  // Opcionales
  env?: 'prod' | 'qa' | 'dev';
  language?: string;
  primaryColor?: string;
  position?: 'bottom-right' | 'bottom-left';
  autoOpen?: boolean;
  greetingMessage?: string;
  zIndex?: number;
  
  // Contexto del usuario
  userContext?: {
    userId?: string;
    name?: string;
    email?: string;
    phone?: string;
    metadata?: Record<string, any>;
  };
  
  // Callbacks
  onReady?: () => void;
  onOpen?: () => void;
  onClose?: () => void;
  onBookingCreated?: (booking: Booking) => void;
  onError?: (error: Error) => void;
  
  // Mensajes personalizados
  messages?: {
    greeting?: string;
    placeholder?: string;
    sendButton?: string;
    errorConnection?: string;
    bookingSuccess?: string;
  };
}
```

#### Ejemplo

```javascript
ChatAgentWidget.init({
  tenantId: 'andina',
  publicKey: 'pk_live_abc123',
  language: 'es-CL',
  primaryColor: '#e91e63',
  position: 'bottom-right',
  autoOpen: false,
  
  userContext: {
    userId: 'user_12345',
    name: 'Juan Pérez',
    email: 'juan@example.com'
  },
  
  onReady: () => {
    console.log('Widget listo');
  },
  
  onBookingCreated: (booking) => {
    console.log('Reserva creada:', booking);
    gtag('event', 'booking_completed', {
      value: booking.service.price
    });
  }
});
```

---

## 🎯 Métodos principales

### `ChatAgentWidget.open()`

Abre el chat programáticamente.

```javascript
ChatAgentWidget.open();
```

**Ejemplo**: Abrir chat cuando usuario hace clic en botón personalizado

```javascript
document.getElementById('btn-help').addEventListener('click', () => {
  ChatAgentWidget.open();
});
```

---

### `ChatAgentWidget.close()`

Cierra el chat.

```javascript
ChatAgentWidget.close();
```

---

### `ChatAgentWidget.toggle()`

Alterna entre abierto/cerrado.

```javascript
ChatAgentWidget.toggle();
```

---

### `ChatAgentWidget.isOpen()`

Retorna `true` si el chat está abierto.

```javascript
const isOpen = ChatAgentWidget.isOpen();
console.log('Chat abierto:', isOpen);
```

---

### `ChatAgentWidget.sendMessage(text)`

Envía un mensaje programáticamente en nombre del usuario.

```javascript
ChatAgentWidget.sendMessage('Necesito un masaje');
```

**Uso común**: Botones de acceso rápido

```javascript
document.getElementById('btn-booking').addEventListener('click', () => {
  ChatAgentWidget.open();
  ChatAgentWidget.sendMessage('Quiero hacer una reserva');
});
```

---

### `ChatAgentWidget.destroy()`

Destruye completamente el widget y libera recursos.

```javascript
ChatAgentWidget.destroy();
```

**Uso**: React, SPA, cuando desmontas un componente.

---

## 📡 Eventos

### `ChatAgentWidget.on(event, callback)`

Escucha eventos del widget.

#### Eventos disponibles

| Evento | Descripción | Datos |
|--------|-------------|-------|
| `ready` | Widget cargado | - |
| `opened` | Chat abierto | - |
| `closed` | Chat cerrado | - |
| `message:sent` | Usuario envió mensaje | `{ text, timestamp }` |
| `message:received` | Agente respondió | `{ text, sender, timestamp }` |
| `booking:created` | Reserva confirmada | `Booking` |
| `error` | Error | `{ code, message }` |

#### Ejemplo

```javascript
ChatAgentWidget.on('ready', () => {
  console.log('✅ Widget listo');
});

ChatAgentWidget.on('opened', () => {
  gtag('event', 'chat_opened');
});

ChatAgentWidget.on('booking:created', (booking) => {
  console.log('Nueva reserva:', booking);
  
  // Mostrar notificación
  alert(`¡Reserva confirmada! ${booking.service.name} con ${booking.provider.name}`);
  
  // Enviar a analytics
  dataLayer.push({
    event: 'booking_completed',
    service: booking.service.name,
    provider: booking.provider.name,
    value: booking.service.price
  });
  
  // Enviar a tu backend
  fetch('/api/webhooks/booking-created', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(booking)
  });
});

ChatAgentWidget.on('error', (error) => {
  console.error('Error en widget:', error);
  Sentry.captureException(error);
});
```

---

### `ChatAgentWidget.off(event, callback)`

Deja de escuchar un evento.

```javascript
function handleOpen() {
  console.log('Chat abierto');
}

ChatAgentWidget.on('opened', handleOpen);

// Más tarde...
ChatAgentWidget.off('opened', handleOpen);
```

---

## 🔍 Métodos de consulta

### `ChatAgentWidget.getConversationId()`

Obtiene el ID de la conversación actual.

```javascript
const conversationId = ChatAgentWidget.getConversationId();
console.log('Conversation ID:', conversationId);
```

---

### `ChatAgentWidget.getUserContext()`

Obtiene el contexto actual del usuario.

```javascript
const userContext = ChatAgentWidget.getUserContext();
console.log('Usuario:', userContext);
// { userId: 'user_123', name: 'Juan Pérez', ... }
```

---

### `ChatAgentWidget.setUserContext(context)`

Actualiza el contexto del usuario en tiempo de ejecución.

```javascript
// Usuario se autentica después de cargar el widget
user.login().then((userData) => {
  ChatAgentWidget.setUserContext({
    userId: userData.id,
    name: userData.name,
    email: userData.email
  });
});
```

---

## 🎨 Personalización en tiempo real

### `ChatAgentWidget.updateConfig(config)`

Actualiza configuración sin recargar.

```javascript
ChatAgentWidget.updateConfig({
  primaryColor: '#f44336',
  language: 'pt-BR'
});
```

---

### `ChatAgentWidget.setTheme(theme)`

Cambia el tema del widget.

```javascript
// Tema oscuro/claro
ChatAgentWidget.setTheme('dark'); // o 'light'
```

---

## 📦 Tipos TypeScript

Si usas TypeScript, el widget exporta tipos:

```typescript
interface Booking {
  id: string;
  serviceId: string;
  providerId: string;
  start: string; // ISO DateTime
  end: string;
  status: 'PENDING' | 'CONFIRMED' | 'CANCELLED' | 'NO_SHOW';
  paymentStatus: 'NONE' | 'PENDING' | 'PAID' | 'FAILED';
  service: {
    id: string;
    name: string;
    durationMinutes: number;
    price: number;
  };
  provider: {
    id: string;
    name: string;
  };
}

interface Message {
  id: string;
  sender: 'USER' | 'AGENT' | 'SYSTEM';
  text: string;
  createdAt: string;
}

interface ChatError {
  code: string;
  message: string;
  details?: any;
}
```

Uso:

```typescript
import type { Booking, Message } from 'chat-agent-widget';

ChatAgentWidget.on('booking:created', (booking: Booking) => {
  console.log(booking.service.name);
});
```

---

## 🧪 Testing

### Verificar que el widget está cargado

```javascript
if (typeof ChatAgentWidget !== 'undefined') {
  console.log('Widget disponible');
} else {
  console.error('Widget no cargado');
}
```

---

### Mock para tests

Si necesitas hacer tests sin el widget real:

```javascript
// jest.setup.js
global.ChatAgentWidget = {
  init: jest.fn(),
  open: jest.fn(),
  close: jest.fn(),
  sendMessage: jest.fn(),
  on: jest.fn(),
  off: jest.fn(),
  isOpen: jest.fn(() => false),
  destroy: jest.fn()
};
```

---

## 🔒 Seguridad

### No exponer información sensible

❌ **Nunca** hagas esto:

```javascript
ChatAgentWidget.setUserContext({
  userId: user.id,
  password: user.password,  // ❌ NUNCA
  creditCard: user.card     // ❌ NUNCA
});
```

✅ **Solo datos necesarios**:

```javascript
ChatAgentWidget.setUserContext({
  userId: user.id,
  name: user.name,
  email: user.email
});
```

---

## 📚 Ejemplos completos

### Integración con Google Analytics

```javascript
ChatAgentWidget.init({
  tenantId: 'andina',
  publicKey: 'pk_live_abc123',
  
  onReady: () => {
    gtag('event', 'widget_loaded');
  },
  
  onOpen: () => {
    gtag('event', 'chat_opened');
  },
  
  onClose: () => {
    gtag('event', 'chat_closed');
  },
  
  onBookingCreated: (booking) => {
    gtag('event', 'purchase', {
      transaction_id: booking.id,
      value: booking.service.price,
      currency: 'CLP',
      items: [{
        item_id: booking.service.id,
        item_name: booking.service.name,
        quantity: 1,
        price: booking.service.price
      }]
    });
  }
});
```

### Integración con Facebook Pixel

```javascript
ChatAgentWidget.on('booking:created', (booking) => {
  fbq('track', 'Purchase', {
    value: booking.service.price,
    currency: 'CLP',
    content_name: booking.service.name,
    content_ids: [booking.service.id],
    content_type: 'product'
  });
});
```

### Integración con HubSpot

```javascript
ChatAgentWidget.on('booking:created', (booking) => {
  _hsq.push(['trackEvent', {
    id: 'Booking Created',
    value: booking.service.price
  }]);
});
```

---

## 🆘 Debugging

### Habilitar modo debug

```javascript
ChatAgentWidget.init({
  tenantId: 'andina',
  publicKey: 'pk_live_abc123',
  debug: true  // Muestra logs detallados en consola
});
```

### Ver estado interno

```javascript
console.log(ChatAgentWidget.getState());
// {
//   isOpen: true,
//   conversationId: 'conv_123',
//   currentState: 'SERVICE_SELECTED',
//   messagesCount: 5
// }
```

---

## 📚 Documentos relacionados

- [Guía principal del widget](/widget/README.md)
- [Embedding en plataformas](/widget/embedding-guide.md)
- [Arquitectura del sistema](/architecture/README.md)
