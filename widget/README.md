# Widget Público — SaaS Agentic Booking Chat

El widget es el componente que se integra en el sitio del cliente final para permitir que los usuarios interactúen con el agente conversacional y reserven servicios.

Este archivo explica:

- cómo integrarlo,  
- cómo configurarlo,  
- cómo funcionan los eventos,  
- cómo diagnosticar problemas,  
- cómo operar en modo multi-tenant.

---

## 🎯 1. Objetivo del Widget

El widget debe ser:

- **fácil de integrar** (solo un `<script>`),
- **seguro** (API key pública + allowedOrigins),
- **ligero** (bundle UMD optimizado),
- **personalizable** (tema, idioma, IA),
- **multi-tenant** (cada empresa tiene su configuración),
- **comunicado por GraphQL** con AppSync.

---

## 🚀 2. Integración básica (script)

El cliente final solo debe incluir:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="TENANT_123"
        data-public-key="pk_live_XXXX"
        data-language="es"
        data-position="right"
        data-auto-open="false"></script>
```

Al cargar, automáticamente aparece el botón flotante del chat.

---

## ✨ 3. Atributos soportados

| Atributo | Tipo | Descripción |
|----------|------|-------------|
| `data-tenant-id` | string | Identificador único del tenant |
| `data-public-key` | string | API key pública del tenant |
| `data-language` | es, en, pt | Idioma inicial del widget |
| `data-position` | left, right | Ubicación del botón |
| `data-auto-open` | true, false | Si el chat se abre automáticamente |
| `data-theme-color` | string | Color principal (hex o rgb) |

Estos atributos pueden ser sobrescritos con un `ChatAgentWidget.init()`.

---

## 🎨 4. Personalización (branding)

Desde el panel admin, el tenant puede configurar:

- color primario
- mensaje de bienvenida
- idioma
- posición
- auto-open
- logo opcional
- texto del header

El widget obtiene estas configuraciones automáticamente desde AppSync.

---

## 🧠 5. Modos de Agente (FSM / IA)

El widget no sabe si el agente opera con IA o modo determinístico; simplemente envía eventos al backend y recibe mensajes.

### Modo FSM (sin IA)

- Conversación guiada
- Bajo costo
- Ideal para planes LITE y PRO

### Modo IA (Bedrock Agent Core)

- Conversación natural
- Interpretación de intención
- Ideal para BUSINESS / ENTERPRISE

La activación de IA depende de la configuración del tenant.

---

## 🛡 6. UX y Manejo de Estados

El widget implementa mejoras de experiencia de usuario para evitar inconsistencias:

### Protección Anti-Spam (Debounce UI)
- Los componentes interactivos (`OptionsChips`, `ServiceChips`, `TimeSlots`) entran en estado **disabled** automáticamente cuando el chat está procesando (`isLoading`).
- Se reduce la opacidad para indicar visualmente que no se permiten clicks adicionales.

### Limpieza de Datos
- Al recibir una nueva respuesta del agente, las opciones antiguas (servicios, slots anteriores) se limpian del estado para evitar que el usuario interactúe con flujo "pasado".

### Confirmación de Reserva
- Soporte para mensajes tipo `confirmation` que renderizan acciones críticas (Confirmar/Cancelar) con estilos diferenciados.

---

## 🔌 7. API JavaScript del Widget

Después de cargar el script, se expone:

```
window.ChatAgentWidget
```

### Métodos disponibles

#### 7.1 Inicializar manualmente

```javascript
ChatAgentWidget.init({
  tenantId: "TENANT_123",
  publicKey: "pk_live_XXXX",
  language: "es",
  position: "right",
  themeColor: "#FF4B8C",
  autoOpen: false
});
```

#### 7.2 Abrir el chat programáticamente

```javascript
ChatAgentWidget.open();
```

#### 7.3 Cerrar el chat

```javascript
ChatAgentWidget.close();
```

#### 7.4 Escuchar eventos del widget

```javascript
ChatAgentWidget.on("booking:created", (payload) => {
  console.log("Nueva reserva:", payload);
});
```

Eventos completos más abajo.

---

## 📡 8. Comunicación con el Backend

El widget se comunica exclusivamente mediante GraphQL hacia AppSync, usando la API key del tenant.

**Operaciones típicas:**

- buscar servicios
- listar profesionales
- consultar disponibilidad
- enviar mensaje al agente
- crear reserva

**Cada request incluye:**

```
x-api-key: <publicKey>
x-tenant-id: <tenantId>
origin: window.location.origin
```

**AppSync valida:**

- que la API key sea válida
- que el dominio sea permitido
- que el tenant exista
- límites por plan

---

## 📬 9. Eventos del Widget

| Evento | Cuándo ocurre | Payload |
|--------|---------------|---------|
| widget:opened | el usuario abre el chat | — |
| widget:closed | el usuario cierra el chat | — |
| message:sent | el usuario envía un mensaje | { text } |
| message:received | el agente responde | { text } |
| slot:selected | el usuario elige un horario | { serviceId, providerId, slot } |
| booking:created | se confirma la reserva | { bookingId, providerId, datetime } |
| error | cualquier error del widget | { code, message } |

**Ejemplo:**

```javascript
ChatAgentWidget.on("booking:created", (booking) => {
  gtag("event", "booking_created", booking);
});
```

---

## 🌐 10. Multi-idioma

**Soportado:** es, en, pt

El widget detecta idioma así:

1. Si existe `data-language`, usa ese
2. Si no, intenta detectar idioma del navegador
3. Si no, usa el idioma del tenant
4. Fallback: es

Se pueden agregar idiomas adicionales por tenant.

---

## 🧪 11. Testing del Widget

Recomendado:

- **Jest** para unidad
- **Playwright** para interacción real
- **Storybook** para probar componentes internos (si se usa internamente)

**Testing básico:**

```bash
npm run test
npm run e2e
```

**Simulación de GraphQL:**

usar "Mock AppSync server" o grabar respuestas con MSW.

---

## 🚨 12. Troubleshooting (problemas comunes)

| Problema | Causa | Solución |
|----------|-------|----------|
| Widget no aparece | CSP del sitio bloquea cdn.tu-saas.com | agregar dominio al CSP |
| Error: ORIGIN_NOT_ALLOWED | dominio no está en allowedOrigins | agregar en panel admin |
| Error: AUTH_FAILED | API key incorrecta o revocada | regenerar key |
| No carga disponibilidad | profesional sin disponibilidad | revisar panel admin |
| Chat nunca responde | error de red o AppSync bloquea request | revisar logs en CloudWatch |

---

## 🚀 13. Ejemplo completo (caso real)

### Caso: "Clínica Dermaskin"

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="DERMASKIN_CL"
        data-public-key="pk_live_derma_01"
        data-language="es"
        data-position="left"
        data-theme-color="#4A90E2"
        data-auto-open="true"></script>
```

**Comportamiento esperado:**

1. El widget aparece bottom-left
2. Al abrirlo: "Hola 👋 ¿Qué servicio necesitas hoy?"
3. El usuario escribe: "Consulta dermatológica para mañana"
4. El agente:
   - identifica "dermatología"
   - muestra profesionales
   - propone horarios
   - crea reserva
5. Se dispara: `booking:created`

---

## 🧭 14. Roadmap del Widget

- iFrame secure mode
- Dark mode automático
- API de extensiones
- Soporte para WhatsApp/Instagram Chat (futuro)
- Animaciones mejoradas
- Modo minimalista para móviles

---

## 📚 Documentos relacionados

- `/docs/widget/api-reference.md`
- `/docs/widget/embedding-guide.md`
- `/docs/architecture/README.md`
- `/docs/admin/README.md`

---

## 🧑‍💻 Referencia de API JavaScript (legacy)

### Inicialización programática avanzada

Si prefieres controlar el widget por código:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"></script>
<script>
  ChatAgentWidget.init({
    tenantId: 'andina',
    publicKey: 'pk_live_abc123xyz',
    language: 'es-CL',
    primaryColor: '#e91e63',
    position: 'bottom-right',
    autoOpen: false,
    greetingMessage: 'Hola, ¿cómo te puedo ayudar?',
    
    // Contexto del usuario (opcional)
    userContext: {
      userId: 'user_12345',
      name: 'Juan Pérez',
      email: 'juan@example.com',
      phone: '+56912345678'
    },
    
    // Callbacks
    onReady: function() {
      console.log('Widget listo');
    },
    onOpen: function() {
      console.log('Chat abierto');
    },
    onClose: function() {
      console.log('Chat cerrado');
    },
    onBookingCreated: function(booking) {
      console.log('Reserva creada:', booking);
      // Puedes enviar a Google Analytics, etc.
    },
    onError: function(error) {
      console.error('Error:', error);
    }
  });
</script>
```

### Métodos disponibles

```javascript
// Abrir el chat programáticamente
ChatAgentWidget.open();

// Cerrar el chat
ChatAgentWidget.close();

// Alternar abierto/cerrado
ChatAgentWidget.toggle();

// Enviar un mensaje programáticamente
ChatAgentWidget.sendMessage('Necesito un masaje');

// Obtener estado actual
const isOpen = ChatAgentWidget.isOpen();

// Destruir el widget
ChatAgentWidget.destroy();
```

---

## 📡 Eventos del widget

Puedes escuchar eventos para integrar con analytics, CRM, etc.

```javascript
ChatAgentWidget.on('ready', function() {
  console.log('Widget cargado');
});

ChatAgentWidget.on('opened', function() {
  // Enviar evento a Google Analytics
  gtag('event', 'chat_opened');
});

ChatAgentWidget.on('closed', function() {
  gtag('event', 'chat_closed');
});

ChatAgentWidget.on('booking:created', function(data) {
  console.log('Nueva reserva:', data);
  
  // Ejemplo: enviar a tu CRM
  fetch('/api/crm/new-booking', {
    method: 'POST',
    body: JSON.stringify(data)
  });
  
  // Google Analytics
  gtag('event', 'booking_completed', {
    service: data.service.name,
    provider: data.provider.name,
    value: data.service.price
  });
});

ChatAgentWidget.on('error', function(error) {
  console.error('Error en widget:', error);
  // Sentry, Rollbar, etc.
});
```

### Eventos disponibles

| Evento | Descripción | Datos |
|--------|-------------|-------|
| `ready` | Widget cargado | - |
| `opened` | Chat abierto | - |
| `closed` | Chat cerrado | - |
| `message:sent` | Usuario envió mensaje | `{ text, timestamp }` |
| `message:received` | Agente respondió | `{ text, sender, timestamp }` |
| `booking:created` | Reserva confirmada | `{ bookingId, service, provider, start, end }` |
| `error` | Error ocurrido | `{ code, message }` |

---

## 🎨 Personalización visual

### Colores

Puedes personalizar el color principal:

```html
<script
  data-primary-color="#e91e63"
  ...
></script>
```

El widget aplicará automáticamente ese color a:
- Botón flotante
- Burbujas de mensajes del agente
- Chips de opciones
- Botones de acción

### CSS personalizado (avanzado)

Si necesitas más control, puedes sobrescribir estilos:

```html
<style>
  /* Personalizar el botón flotante */
  #chat-agent-widget-launcher {
    bottom: 20px !important;
    right: 20px !important;
    background: linear-gradient(45deg, #e91e63, #f06292) !important;
    box-shadow: 0 4px 20px rgba(233, 30, 99, 0.4) !important;
  }
  
  /* Personalizar ventana del chat */
  #chat-agent-widget-window {
    border-radius: 16px !important;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2) !important;
  }
  
  /* Personalizar burbujas de mensaje */
  .chat-message-agent {
    background: #f5f5f5 !important;
    color: #333 !important;
  }
  
  .chat-message-user {
    background: #e91e63 !important;
    color: white !important;
  }
</style>
```

---

## 🌍 Internacionalización (i18n)

### Idiomas soportados

- `es-CL` (Español Chile)
- `es-MX` (Español México)
- `es-AR` (Español Argentina)
- `pt-BR` (Portugués Brasil)
- `en-US` (Inglés)

### Configurar idioma

```html
<script
  data-language="pt-BR"
  ...
></script>
```

### Mensajes personalizados

Puedes sobrescribir textos específicos:

```javascript
ChatAgentWidget.init({
  tenantId: 'andina',
  publicKey: 'pk_live_...',
  language: 'es-CL',
  
  messages: {
    greeting: 'Hola, ¿en qué te puedo ayudar?',
    placeholder: 'Escribe tu mensaje...',
    sendButton: 'Enviar',
    errorConnection: 'No se pudo conectar. Intenta de nuevo.',
    bookingSuccess: '¡Reserva confirmada! Te enviamos un email.',
  }
});
```

---

## 📱 Responsive y móvil

El widget es completamente responsive:

- **Desktop**: Ventana flotante en esquina
- **Mobile**: Fullscreen cuando se abre
- **Tablet**: Adapta tamaño según viewport

No necesitas configuración adicional.

---

## 🔒 Seguridad

### CORS

El widget envía el `origin` del sitio en cada request.

Asegúrate de configurar los **dominios permitidos** en el panel administrativo:

```
https://www.tuempresa.com
https://tuempresa.com
https://app.tuempresa.com
```

Cualquier otro dominio será rechazado automáticamente.

### API Key

La `publicKey` es segura para el frontend porque:

- Está limitada a dominios específicos
- Solo permite operaciones de lectura y creación de reservas
- No permite modificar catálogo ni configuraciones

**Nunca expongas tu API Key privada del panel admin.**

---

## ⚡ Performance

El widget está optimizado para carga rápida:

- **Tamaño**: ~50KB gzip
- **Lazy loading**: Se carga de forma asíncrona
- **CDN global**: CloudFront con edge locations
- **Caché**: Versiones cacheadas por 1 año

### Cargar de forma asíncrona

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  async
  defer
  ...
></script>
```

---

## 🧪 Testing en desarrollo

Para probar localmente sin modificar el código de producción:

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="andina"
  data-public-key="pk_test_abc123"
  data-env="dev"
></script>
```

El ambiente `dev` apunta a un backend de desarrollo.

---

## 🆘 Troubleshooting

### El widget no aparece

1. Verifica que el script esté correctamente insertado antes del `</body>`
2. Revisa la consola del navegador por errores
3. Verifica que `tenantId` y `publicKey` sean correctos
4. Asegúrate de que tu dominio esté en la lista de `allowedOrigins`

### Errores de CORS

```
Access to fetch at 'https://api.tu-saas.com' has been blocked by CORS policy
```

**Solución**: Agrega tu dominio en el panel admin → API Keys → Allowed Origins

### El agente no responde

1. Verifica que los servicios y profesionales estén activos en el panel
2. Revisa que haya disponibilidad configurada
3. Consulta los logs en CloudWatch

---

## 📚 Próximos pasos

- [Guía de embedding en diferentes plataformas](/widget/embedding-guide.md)
- [API JavaScript completa](/widget/api-reference.md)
- [Ejemplos de integración](/widget/examples.md)

---

## 💬 Soporte

Si tienes problemas:

- Email: support@tu-saas.com
- Documentación: https://docs.tu-saas.com
- Status: https://status.tu-saas.com
