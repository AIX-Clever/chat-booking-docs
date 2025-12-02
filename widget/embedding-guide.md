# Widget Embedding Guide — Plataformas Específicas

Esta guía muestra cómo integrar el widget en diferentes plataformas y CMS.

---

## 🌐 HTML Estático

### Integración básica

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Mi Sitio</title>
</head>
<body>
  <h1>Bienvenido</h1>
  <p>Contenido de tu sitio...</p>
  
  <!-- Widget del chat agéntico -->
  <script
    src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
    data-tenant-id="YOUR_TENANT_ID"
    data-public-key="YOUR_PUBLIC_KEY"
    data-primary-color="#e91e63"
  ></script>
</body>
</html>
```

---

## ⚛️ React

### Opción 1: Script directo en `index.html`

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="es">
<head>
  <!-- ... -->
</head>
<body>
  <div id="root"></div>
  
  <script
    src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
    data-tenant-id="%REACT_APP_TENANT_ID%"
    data-public-key="%REACT_APP_PUBLIC_KEY%"
  ></script>
</body>
</html>
```

Configurar en `.env`:
```
REACT_APP_TENANT_ID=andina
REACT_APP_PUBLIC_KEY=pk_live_abc123
```

### Opción 2: Componente React con `useEffect`

```jsx
// src/components/ChatWidget.jsx
import { useEffect } from 'react';

export default function ChatWidget() {
  useEffect(() => {
    // Cargar script
    const script = document.createElement('script');
    script.src = 'https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js';
    script.async = true;
    script.setAttribute('data-tenant-id', process.env.REACT_APP_TENANT_ID);
    script.setAttribute('data-public-key', process.env.REACT_APP_PUBLIC_KEY);
    script.setAttribute('data-primary-color', '#1976d2');
    
    document.body.appendChild(script);
    
    return () => {
      // Limpiar al desmontar
      document.body.removeChild(script);
      if (window.ChatAgentWidget) {
        window.ChatAgentWidget.destroy();
      }
    };
  }, []);
  
  return null; // No renderiza nada
}
```

Usar en `App.jsx`:
```jsx
import ChatWidget from './components/ChatWidget';

function App() {
  return (
    <>
      <ChatWidget />
      {/* Tu app */}
    </>
  );
}
```

### Opción 3: Inicialización programática

```jsx
// src/hooks/useChatWidget.js
import { useEffect } from 'react';

export function useChatWidget(config) {
  useEffect(() => {
    const script = document.createElement('script');
    script.src = 'https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js';
    script.async = true;
    
    script.onload = () => {
      window.ChatAgentWidget.init({
        tenantId: config.tenantId,
        publicKey: config.publicKey,
        primaryColor: config.primaryColor,
        userContext: config.userContext,
        onBookingCreated: (booking) => {
          console.log('Reserva creada:', booking);
          // Tracking, analytics, etc.
        }
      });
    };
    
    document.body.appendChild(script);
    
    return () => {
      if (window.ChatAgentWidget) {
        window.ChatAgentWidget.destroy();
      }
      document.body.removeChild(script);
    };
  }, [config]);
}

// Usar en componente
function App() {
  const user = useAuth(); // tu hook de auth
  
  useChatWidget({
    tenantId: process.env.REACT_APP_TENANT_ID,
    publicKey: process.env.REACT_APP_PUBLIC_KEY,
    primaryColor: '#1976d2',
    userContext: {
      userId: user?.id,
      name: user?.name,
      email: user?.email
    }
  });
  
  return <div>...</div>;
}
```

---

## 🔷 Next.js

### App Router (Next.js 13+)

```jsx
// app/layout.jsx
import Script from 'next/script';

export default function RootLayout({ children }) {
  return (
    <html lang="es">
      <body>
        {children}
        
        <Script
          src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
          strategy="lazyOnload"
          data-tenant-id={process.env.NEXT_PUBLIC_TENANT_ID}
          data-public-key={process.env.NEXT_PUBLIC_PUBLIC_KEY}
        />
      </body>
    </html>
  );
}
```

### Pages Router (Next.js 12 y anteriores)

```jsx
// pages/_app.jsx
import Script from 'next/script';

function MyApp({ Component, pageProps }) {
  return (
    <>
      <Component {...pageProps} />
      
      <Script
        src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        strategy="lazyOnload"
        data-tenant-id={process.env.NEXT_PUBLIC_TENANT_ID}
        data-public-key={process.env.NEXT_PUBLIC_PUBLIC_KEY}
      />
    </>
  );
}

export default MyApp;
```

---

## 📘 WordPress

### Método 1: Plugin "Insert Headers and Footers"

1. Instala el plugin "Insert Headers and Footers"
2. Ve a **Configuración → Insert Headers and Footers**
3. Pega en "Scripts in Footer":

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="YOUR_TENANT_ID"
  data-public-key="YOUR_PUBLIC_KEY"
></script>
```

### Método 2: Editar `footer.php` del tema

```php
<!-- wp-content/themes/tu-tema/footer.php -->

<?php wp_footer(); ?>

<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="<?php echo get_option('chat_widget_tenant_id'); ?>"
  data-public-key="<?php echo get_option('chat_widget_public_key'); ?>"
></script>

</body>
</html>
```

### Método 3: Plugin personalizado

Crear plugin en `wp-content/plugins/chat-widget/chat-widget.php`:

```php
<?php
/**
 * Plugin Name: Chat Agéntico Widget
 * Description: Integra el chat de reservas
 * Version: 1.0
 */

function chat_widget_enqueue_script() {
    wp_enqueue_script(
        'chat-agent-widget',
        'https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js',
        array(),
        null,
        true
    );
    
    wp_add_inline_script('chat-agent-widget', '
        document.currentScript.setAttribute("data-tenant-id", "' . get_option('chat_tenant_id') . '");
        document.currentScript.setAttribute("data-public-key", "' . get_option('chat_public_key') . '");
    ', 'before');
}
add_action('wp_enqueue_scripts', 'chat_widget_enqueue_script');

// Página de configuración en admin
function chat_widget_settings_page() {
    add_options_page(
        'Chat Widget Settings',
        'Chat Widget',
        'manage_options',
        'chat-widget',
        'chat_widget_settings_html'
    );
}
add_action('admin_menu', 'chat_widget_settings_page');

function chat_widget_settings_html() {
    ?>
    <div class="wrap">
        <h1>Configuración Chat Widget</h1>
        <form method="post" action="options.php">
            <?php
            settings_fields('chat_widget');
            do_settings_sections('chat_widget');
            submit_button();
            ?>
        </form>
    </div>
    <?php
}
```

---

## 🛍️ Shopify

### Método 1: Editar tema

1. Ve a **Tienda online → Temas → Acciones → Editar código**
2. Abre `layout/theme.liquid`
3. Antes del cierre `</body>`, agrega:

```liquid
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="{{ settings.chat_tenant_id }}"
  data-public-key="{{ settings.chat_public_key }}"
></script>
```

4. En `config/settings_schema.json`, agrega configuración:

```json
{
  "name": "Chat Widget",
  "settings": [
    {
      "type": "text",
      "id": "chat_tenant_id",
      "label": "Tenant ID"
    },
    {
      "type": "text",
      "id": "chat_public_key",
      "label": "Public Key"
    }
  ]
}
```

---

## 🎨 Webflow

1. Ve a **Project Settings → Custom Code**
2. En **Footer Code**, pega:

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="YOUR_TENANT_ID"
  data-public-key="YOUR_PUBLIC_KEY"
></script>
```

3. Publica el sitio

---

## 🔶 Wix

1. Ve a **Configuración → Gestionar código personalizado**
2. Clic en **+ Agregar código personalizado**
3. Pega el script
4. Selecciona:
   - **Ubicación**: Body - final
   - **Páginas**: Todas las páginas
5. Guarda y publica

---

## 📦 Squarespace

1. Ve a **Configuración → Avanzado → Inyección de código**
2. En **Footer**, pega:

```html
<script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  data-tenant-id="YOUR_TENANT_ID"
  data-public-key="YOUR_PUBLIC_KEY"
></script>
```

3. Guarda

---

## 📱 React Native / Mobile

El widget está diseñado para web. Para apps móviles:

### Opción 1: WebView con widget

```jsx
import { WebView } from 'react-native-webview';

export default function ChatScreen() {
  const html = `
    <!DOCTYPE html>
    <html>
    <head>
      <meta name="viewport" content="width=device-width, initial-scale=1">
    </head>
    <body>
      <div id="chat-container"></div>
      <script
        src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="YOUR_TENANT_ID"
        data-public-key="YOUR_PUBLIC_KEY"
        data-auto-open="true"
      ></script>
    </body>
    </html>
  `;
  
  return <WebView source={{ html }} />;
}
```

### Opción 2: API GraphQL directa

Usar directamente la API GraphQL del backend sin widget.

---

## 🧪 Google Tag Manager

Si usas GTM:

1. Ve a **Etiquetas → Nueva**
2. Tipo: **HTML personalizado**
3. Pega:

```html
<script>
(function() {
  var script = document.createElement('script');
  script.src = 'https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js';
  script.setAttribute('data-tenant-id', 'YOUR_TENANT_ID');
  script.setAttribute('data-public-key', 'YOUR_PUBLIC_KEY');
  document.body.appendChild(script);
})();
</script>
```

4. Activador: **Todas las páginas**
5. Guarda y publica

---

## 📚 Documentos relacionados

- [Guía principal del widget](/widget/README.md)
- [API JavaScript completa](/widget/api-reference.md)
