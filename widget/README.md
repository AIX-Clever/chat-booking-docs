# Widget Embebible — Guía de Integración

Este documento describe cómo integrar el **chat agéntico** en cualquier sitio web usando el widget JavaScript embebible.

---

## 🎯 Integración básica

### Paso 1: Obtener credenciales

Desde el panel administrativo, obtén:

- **Tenant ID**: identificador único de tu empresa
- **Public API Key**: clave pública para el widget (formato: `pk_live_...`)

### Paso 2: Insertar el script

Agrega el siguiente código antes del cierre del `</body>`:

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="YOUR_TENANT_ID"
  data-public-key="YOUR_PUBLIC_KEY"
></script>
```

### Paso 3: ¡Listo!

El widget aparecerá automáticamente como un botón flotante en la esquina inferior derecha.

---

## ⚙️ Configuración avanzada

### Opciones disponibles (atributos `data-*`)

| Atributo | Tipo | Default | Descripción |
|----------|------|---------|-------------|
| `data-tenant-id` | string | **requerido** | ID de tu tenant |
| `data-public-key` | string | **requerido** | API Key pública |
| `data-env` | string | `prod` | Ambiente: `prod`, `qa`, `dev` |
| `data-language` | string | `es-CL` | Idioma del widget |
| `data-primary-color` | string | `#1976d2` | Color principal (hex) |
| `data-position` | string | `bottom-right` | Posición: `bottom-right`, `bottom-left` |
| `data-auto-open` | boolean | `false` | Abrir automáticamente al cargar |
| `data-greeting-message` | string | (auto) | Mensaje inicial personalizado |
| `data-z-index` | number | `9999` | z-index del widget |

### Ejemplo con opciones personalizadas

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="andina"
  data-public-key="pk_live_abc123xyz"
  data-language="es-CL"
  data-primary-color="#e91e63"
  data-position="bottom-left"
  data-auto-open="false"
  data-greeting-message="¡Hola! ¿En qué puedo ayudarte hoy?"
></script>
```

---

## 🧑‍💻 API JavaScript avanzada

### Inicialización programática

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
