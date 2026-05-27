---
name: graph-approvals
description: "usa esta skill cuando el usuario necesite ver aprobaciones pendientes, revisar solicitudes de aprobación, consultar historial de aprobaciones o gestionar flujos de aprobación de Power Automate en Microsoft 365."
---

# graph-approvals

Usa la herramienta `graph_approvals` para gestión de aprobaciones de Power Automate.

> **IMPORTANTE**: Las aprobaciones viven en el mismo servicio de Power
> Automate que los flujos (mismo recurso de Azure AD:
> `https://service.flow.microsoft.com`). El primer login se hace vía
> device code con `action: "auth-login"` de esta misma tool. Si ya
> autenticaste en `graph_flows` en la misma sesión, esta tool puede
> reutilizar el refresh_token sin pedir un segundo login.
> NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_approvals`,
**o si alguna llamada previa retornó `errorType: AUTH_ERROR`**, inicia siempre `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`pending`, `all`, `history`).
6. Si `status = pending`, vuelve a pedir confirmación sin continuar.
7. Si `status = expired`, genera un nuevo `auth-login`.

## Formato UX obligatorio para autenticación (sin consola)

Cuando la herramienta responda `errorType: AUTH_ERROR`, responde SIEMPRE con este formato:

1. "Para continuar, abre: **{verification_uri}**"
2. "Ingresa este código: **{user_code}**"
3. "Cuando termines, escribe: **listo**"

Reglas estrictas:
- No afirmar que la integración no existe si hay URL+código.
- No pedir consola al usuario final.
- No redirigir a `web_login_playwright`.
- Tras "listo", reintentar la acción original.

## Selección de environment

Power Automate organiza approvals por **environment** (entorno), no por
buzón ni UPN. El default de la mayoría de tenants es `Default-{tenantId}`
y la herramienta lo resuelve automáticamente si no se especifica.

Solo pasa el parámetro **`environment`** cuando el usuario mencione
explícitamente un environment distinto (ej. "el de pruebas", "el de
marketing"). Si menciona algo ambiguo, primero ejecuta sin parámetro
para usar el default y, si el resultado no es lo que esperaba, pregunta
qué environment usar.

El parámetro `user` que existe en otras graph-tools no aplica aquí — la
API de approvals filtra por la identidad del token, no por UPN explícito.

---

## Acciones soportadas

- `pending` — aprobaciones pendientes de acción (default)
- `all` — todas las aprobaciones recientes con resumen por status
- `history` — historial de aprobaciones ya completadas
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Revisar aprobaciones pendientes

```
Usuario: "¿tengo algo pendiente de aprobar?"

→ Llama pending
→ Si hay aprobaciones pendientes:
  Agente: "Tienes 3 aprobaciones pendientes:
           URGENTES (esperando más de 24h):
           1. [Compras] Solicitud de equipos TI — Carlos Pérez — esperando 2 días
              Monto: $45,000
           RECIENTES:
           2. [RH] Solicitud de vacaciones — María Sánchez — hace 4h
           3. [Facilities] Mantenimiento oficina — Juan Torres — hace 6h
           ¿Quieres tomar alguna acción sobre alguna de estas?"

→ Si no hay aprobaciones pendientes:
  Agente: "No tienes aprobaciones pendientes en este momento.
           ¿Quieres revisar el historial de aprobaciones recientes?"
```

### Revisar el detalle de una aprobación

```
Usuario: "¿de qué trata la solicitud de Carlos?"

→ Presenta el detalle disponible (campo `details` del resultado):
  Agente: "Solicitud de Carlos Pérez (Compras):
           Título: Solicitud de equipos TI
           Descripción: Compra de 5 laptops para nuevos ingresos
           Monto: $45,000
           Creada: hace 2 días
           Opciones de respuesta disponibles: Aprobar, Rechazar
           ¿Quieres aprobarlo, rechazarlo o necesitas más información?"
```

### Historial de aprobaciones

```
Usuario: "¿qué aprobaciones completé esta semana?"

→ Llama history
→ Filtra por fecha y presenta:
  Agente: "Esta semana completaste 4 aprobaciones:
           Aprobadas:
           - Vacaciones María S. (lunes)
           - Presupuesto Q2 Marketing (martes)
           Rechazadas:
           - Solicitud equipo extra Carlos (miércoles)
           - Viaje viernes próximo (jueves)"
```

---

## Recuperación de errores

### Permisos de Power Automate no consentidos en la app (CONSENT_ERROR)

```
graph_approvals returns: errorType: CONSENT_ERROR

Agente: "Los permisos de Power Automate aún no están habilitados en la
         app registrada en Azure AD. Esto requiere que un administrador
         del tenant los agregue en App registrations → API permissions
         (buscar 'Microsoft Flow Service' o 'Power Automate') y haga
         grant admin consent. No es algo que se resuelva con un nuevo
         login. Mientras tanto no puedo consultar tus aprobaciones."
```

**Importante**: ante `CONSENT_ERROR`, **NO intentes `auth-login`** ni reintentes la acción. Un nuevo login no cambia los permisos de la app de Azure AD.

### Sin aprobaciones pendientes

```
pending returns: total: 0

Agente: "No tienes aprobaciones pendientes en este momento.
         ¿Quieres revisar el historial, o prefieres que te avise cuando llegue alguna?"
```

### Sin historial

```
history returns: total: 0

Agente: "No encontré aprobaciones completadas en el período consultado.
         ¿Quieres ver un período diferente?"
```

### Error de la API (APPROVALS_ERROR)

```
graph_approvals returns: APPROVALS_ERROR: [403] ...

Agente: "No tengo permisos para acceder a las aprobaciones de este
         environment. Es posible que la cuenta esté restringida en el
         tenant. ¿Quieres que intente con otro environment?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Usuario quiere aprobar/rechazar directamente | Muestra el detalle y pide confirmación |
| La solicitud no queda clara por el título | Presenta el detalle disponible antes de actuar |
| El usuario menciona un environment ambiguo | Pregunta el ID exacto o sugiere intentar con el default |

---

## Reglas absolutas

- No expongas IDs internos de aprobación al usuario salvo que los pida explícitamente.
- Siempre muestra las aprobaciones urgentes (más de 24h en espera, campo `isUrgent`) al inicio.
- Si la API devuelve `CONSENT_ERROR`, explica que es configuración de la app y NO reintentes con auth-login.
- Si la API devuelve `APPROVALS_ERROR: [403]`, explica que es permiso de tenant/environment, distinto al permiso de la app.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_approvals` con `{"action": "auth-login"}`.
> "Para autenticarte en Power Automate, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_approvals` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.

**No confundir `AUTH_ERROR` con `CONSENT_ERROR`**: `AUTH_ERROR` significa que la sesión expiró (fix: nuevo login); `CONSENT_ERROR` significa que la app de Azure AD no tiene los permisos consentidos (fix: el admin del tenant, no el usuario; no se resuelve con login).