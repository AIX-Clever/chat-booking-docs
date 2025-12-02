# Panel Administrativo — Guía de Uso

Este documento describe cómo usar el panel administrativo del SaaS para gestionar servicios, profesionales, disponibilidad y configuraciones.

---

## 🔑 Acceso al panel

URL: `https://app.tu-saas.com`

**Autenticación**: AWS Cognito

Cada usuario admin tiene acceso solo a su tenant.

---

## 🏠 Dashboard principal

Al iniciar sesión verás:

- **Reservas de hoy**: Lista de bookings confirmados
- **Mensajes activos**: Conversaciones en curso
- **Estadísticas del mes**:
  - Total de reservas
  - Ingresos generados
  - Usuarios atendidos
  - Tasa de conversión del chat

---

## 🛠️ Gestión de Servicios

### Crear nuevo servicio

1. Ve a **Servicios → Nuevo servicio**
2. Completa:
   - **Nombre**: Ej. "Masaje descontracturante"
   - **Descripción**: Detalle para el cliente
   - **Categoría**: Ej. "Masajes", "Peluquería", etc.
   - **Duración**: En minutos (Ej. 60)
   - **Precio**: Opcional
3. Clic en **Guardar**

### Editar servicio existente

1. Ve a **Servicios**
2. Clic en el servicio
3. Modifica campos necesarios
4. **Guardar cambios**

### Desactivar un servicio

En lugar de eliminarlo, márcalo como **Inactivo**.

Esto evita que aparezca en el widget, pero conserva histórico de reservas.

---

## 👥 Gestión de Profesionales

### Agregar profesional

1. Ve a **Profesionales → Nuevo profesional**
2. Completa:
   - **Nombre**: Ej. "María González"
   - **Bio**: Breve descripción
   - **Servicios que ofrece**: Seleccionar de la lista
   - **Zona horaria**: Ej. "America/Santiago"
3. **Guardar**

### Editar profesional

1. Ve a **Profesionales**
2. Clic en el profesional
3. Modifica información
4. **Guardar**

### Asignar servicios a un profesional

En la vista de edición del profesional:

1. Sección **Servicios**
2. Seleccionar checkboxes de los servicios que presta
3. **Guardar**

---

## 📅 Disponibilidad

### Configurar horarios semanales

1. Ve a **Profesionales → [Nombre] → Disponibilidad**
2. Por cada día de la semana:
   - Marcar si está disponible
   - Definir rangos de hora (Ej. 09:00 - 13:00, 15:00 - 19:00)
   - Agregar pausas/breaks si aplica
3. **Guardar**

**Ejemplo**:

```
Lunes:
  ✅ Disponible
  Rango 1: 09:00 - 13:00
  Rango 2: 15:00 - 19:00
  Break: 11:00 - 11:15

Martes:
  ✅ Disponible
  Rango 1: 10:00 - 18:00

Miércoles:
  ❌ No disponible
```

### Días libres / excepciones

En **Disponibilidad → Excepciones**:

1. Agregar fecha específica
2. Marcar como **No disponible**
3. Guardar

Ejemplo: Vacaciones, feriados, eventos especiales.

---

## 📋 Gestión de Reservas

### Ver reservas

Ve a **Reservas** para ver:

- **Próximas**: Reservas confirmadas
- **Pasadas**: Historial
- **Canceladas**: Reservas anuladas

**Filtros disponibles**:
- Por fecha
- Por profesional
- Por servicio
- Por estado

### Cancelar una reserva

1. Busca la reserva
2. Clic en **Cancelar**
3. Opcional: Agregar motivo
4. Confirmar

El cliente recibirá notificación automática (si está configurado).

### Exportar reservas

Clic en **Exportar** para descargar CSV con:
- Fecha y hora
- Cliente
- Servicio
- Profesional
- Estado
- Monto

---

## 🔑 API Keys

### Crear nueva API Key

1. Ve a **Configuración → API Keys**
2. Clic en **Nueva clave**
3. Completa:
   - **Descripción**: Ej. "Widget sitio web principal"
   - **Dominios permitidos**: Ej. `https://www.tuempresa.com`
4. **Crear**

Se genera una clave tipo `pk_live_abc123xyz`.

⚠️ **Importante**: Copia la clave inmediatamente. No se mostrará de nuevo.

### Revocar una API Key

1. Ve a **API Keys**
2. Encuentra la clave
3. Clic en **Revocar**
4. Confirmar

La clave dejará de funcionar inmediatamente.

---

## 🎨 Configuración del Widget

### Personalizar apariencia

Ve a **Configuración → Widget**:

- **Color principal**: Selector de color (hex)
- **Posición**: Bottom-right / Bottom-left
- **Mensaje de bienvenida**: Texto personalizado
- **Idioma por defecto**: es-CL, pt-BR, etc.
- **Auto-abrir**: Sí/No

**Vista previa** en vivo del widget aparece al lado.

### Configurar políticas de reserva

En **Configuración → Reservas**:

- **Anticipación mínima**: Minutos antes de poder reservar
- **Anticipación máxima**: Días hacia adelante
- **Permitir cancelación**: Sí/No
- **Horas antes para cancelar**: Límite

---

## 🤖 Configuración del Agente

### Mensajes personalizados

Ve a **Configuración → Agente → Mensajes**:

Personaliza respuestas:

- Saludo inicial
- Servicio no encontrado
- Sin disponibilidad
- Confirmación de reserva

**Variables disponibles**:
- `{userName}` - Nombre del usuario
- `{serviceName}` - Servicio seleccionado
- `{providerName}` - Profesional
- `{dateTime}` - Fecha y hora

**Ejemplo**:

```
Mensaje de confirmación:
"¡Listo, {userName}! Tu reserva de {serviceName} con {providerName} está confirmada para el {dateTime}. Te enviaremos un recordatorio 24 horas antes."
```

### IA opcional

Si tu plan incluye AI:

**Configuración → Agente → IA**:

- **Proveedor**: Bedrock, OpenAI, etc.
- **Modelo**: Claude 3 Sonnet, GPT-4, etc.
- **Temperatura**: 0 (preciso) a 1 (creativo)
- **Prompt del sistema**: Instrucciones base

---

## 📊 Reportes y Analytics

### Métricas disponibles

**Dashboard → Reportes**:

- **Conversiones**:
  - Conversaciones iniciadas
  - Conversaciones que terminaron en reserva
  - Tasa de conversión
- **Uso del chat**:
  - Mensajes por día
  - Horarios de mayor actividad
- **Reservas**:
  - Total por servicio
  - Total por profesional
  - Ingresos generados
- **Clientes**:
  - Nuevos vs. recurrentes
  - Servicios más solicitados

### Exportar reportes

Clic en **Exportar** para descargar en:
- CSV
- Excel
- PDF

---

## 👥 Gestión de Usuarios

### Invitar nuevo admin

1. Ve a **Configuración → Usuarios**
2. Clic en **Invitar usuario**
3. Completa:
   - Email
   - Rol: Admin, Staff, Viewer
4. Enviar invitación

El usuario recibirá email con link de activación.

### Roles disponibles

| Rol | Permisos |
|-----|----------|
| **Admin** | Acceso total |
| **Staff** | Ver reservas, gestionar disponibilidad |
| **Viewer** | Solo lectura |

---

## 💳 Planes y facturación

### Ver plan actual

**Configuración → Plan**:

- Plan contratado (FREE, PRO, ENTERPRISE)
- Límites mensuales:
  - Mensajes
  - Reservas
  - Tokens IA
- Uso actual vs. límite

### Actualizar plan

Clic en **Actualizar plan** para ver opciones.

### Métodos de pago

**Configuración → Facturación**:

- Agregar tarjeta de crédito
- Ver historial de pagos
- Descargar facturas

---

## 🔔 Notificaciones

### Configurar notificaciones

**Configuración → Notificaciones**:

**Email**:
- Nueva reserva
- Cancelación
- Recordatorio (24h antes)
- Reporte diario

**SMS** (si está habilitado):
- Confirmación de reserva al cliente
- Recordatorio

**Webhook** (avanzado):
- URL de tu sistema
- Enviar eventos en tiempo real

---

## 🆘 Soporte

Desde el panel, puedes:

- **Chat de soporte**: Botón inferior derecho
- **Documentación**: Link en menú
- **Estado del servicio**: status.tu-saas.com

---

## 📚 Documentos relacionados

- [Widget — Guía de integración](/widget/README.md)
- [Arquitectura del sistema](/architecture/README.md)
- [Seguridad](/security/README.md)
