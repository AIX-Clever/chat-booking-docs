# Embedding Guide — Widget del Chat Agentic  
SaaS Agentic Booking Chat

Esta guía explica cómo integrar el widget en cualquier tipo de sitio web:

- HTML tradicional  
- React  
- Next.js  
- Angular  
- Vue  
- Shopify  
- WordPress  
- Webflow  
- Wix  
- Aplicaciones internas (intranets, portales corporativos)

Además cubre:

- requisitos de CSP,
- self-hosting opcional,
- manejo de versiones,
- troubleshooting avanzado.

---

# 🚀 1. Inserción básica (HTML)

La forma más simple de integrar el widget es añadir este `<script>`:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="TENANT_123"
        data-public-key="pk_live_xxx"
        data-language="es"
        data-position="right"
        data-auto-open="false"></script>
```

Al cargar, aparecerá un botón flotante en la esquina inferior.

## 1.1 Requisitos

- Insertarlo justo antes de `</body>`
- El dominio debe estar en `allowedOrigins` del panel admin
- El tenant debe tener una API key activa

---

# 🧩 2. Integración en React

Insertar el script dinámicamente dentro de un `useEffect`:

```javascript
useEffect(() => {
  const s = document.createElement("script");
  s.src = "https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js";
  s.dataset.tenantId = "TENANT_123";
  s.dataset.publicKey = "pk_live_xxx";
  document.body.appendChild(s);
}, []);
```

## 2.1 Abrir el widget desde React

```jsx
<button onClick={() => window.ChatAgentWidget.open()}>
  Reservar ahora
</button>
```

---

# 📦 3. Integración en Next.js

Agregar el script en `_app.tsx` o mediante el componente `next/script`:

```jsx
<Script
  src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
  strategy="afterInteractive"
  data-tenant-id="TENANT_123"
  data-public-key="pk_live_xxx"
/>
```

Si se usa App Router:

```jsx
<Script src="..." />
```

---

# 🅰️ 4. Integración en Angular

En `index.html`:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="TENANT_123"
        data-public-key="pk_live_xxx"></script>
```

Y declarar:

```typescript
declare var ChatAgentWidget: any;
```

---

# 🖖 5. Integración en Vue

Agregar script en `public/index.html`:

```html
<script src="https://cdn.tu-saas.com/chat-widget/latest/chat-widget.js"
        data-tenant-id="TENANT_123"
        data-public-key="pk_live_xxx"></script>
```

---

# 🛍️ 6. Integración en Shopify

**Shopify Themes (Online Store > Theme > Edit Code)**

Agregar en `theme.liquid` justo antes de `</body>`:

```html
<script src="https://cdn..."></script>
```

---

# 🌐 7. Integración en WordPress

## Método A: WPCode

- Plugin → WPCode
- "Add Snippet"
- "Custom HTML"
- Pegar el script.

## Método B: Editor del tema

Insertar en `footer.php` antes de `</body>`.

---

# 🧱 8. Integración en Webflow

En el proyecto: **Settings → Custom Code → Footer**

Pegar `<script>`

Publicar cambios.

---

# 🧬 9. Integración en Wix

**Settings**

- Custom Code
- Add Code to All Pages
- Ubicar en "Body - end"

---

# 🏢 10. Integración en Portales Corporativos

El widget funciona perfectamente en:

- Liferay
- SharePoint
- Portales internos
- Iframes de intranet

Solo se requiere:

- permitir el dominio del portal en `allowedOrigins`
- que el portal permita scripts externos

---

# 📌 11. Versionado del Script

Hay 3 formas de cargar el widget:

## 11.1 Última versión estable

```html
<script src="https://cdn.../chat-widget/latest/chat-widget.js"></script>
```

## 11.2 Versión fija (para control empresarial)

```html
<script src="https://cdn.../chat-widget/v1.2.5/chat-widget.js"></script>
```

## 11.3 Canary (para pruebas)

```html
<script src="https://cdn.../chat-widget/canary/chat-widget.js"></script>
```

---

# 🔐 12. Requisitos de Seguridad / CSP

Si el sitio usa Content Security Policy, agregar:

```
script-src 'self' https://cdn.tu-saas.com;
connect-src https://api.tu-saas.com;
img-src data: https://cdn.tu-saas.com;
style-src 'unsafe-inline';
```

---

# 🧯 13. Troubleshooting

## 13.1 El widget no aparece

- revisar consola → ¿error CSP?
- revisar panel admin → ¿dominio en `allowedOrigins`?
- revisar API key → ¿activa?

## 13.2 Error: ORIGIN_NOT_ALLOWED

- agregar dominio al panel admin → "Allowed Origins"

## 13.3 Error: AUTH_FAILED

- regenerar API key
- actualizar snippet

## 13.4 El widget aparece pero no responde

- AppSync inaccesible
- revisar red
- revisar logs en CloudWatch

## 13.5 Conflicto con z-index

Usar:

```javascript
ChatAgentWidget.init({ zIndex: 999999 });
```

---

# 🧠 14. Self-Hosting Opcional

El tenant enterprise puede auto-hospedar el widget:

1. Descargar el build: `chat-widget.js`
2. Subir a su propio bucket:
   ```
   https://cdn.cliente.com/widget/chat.js
   ```
3. Actualizar la referencia del script:
   ```html
   <script src="https://cdn.cliente.com/widget/chat.js">
   ```

**Limitaciones:**

- no recibe updates automáticos
- no obtiene mejoras de seguridad inmediata
- requiere permitir su dominio en `allowedOrigins`

---

# 🔮 15. Roadmap Embedding Guide

- scripts "one-line installer"
- modo iFrame sandbox completo
- modo inline dentro de componentes
- plugin para React / Vue / Angular
- integración directa con Shopify App Store

---

# ✔️ Fin del archivo
