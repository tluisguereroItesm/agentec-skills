---
name: graph-approvals
description: "usa esta skill cuando el usuario necesite ver aprobaciones pendientes, revisar solicitudes de aprobación, consultar historial de aprobaciones o gestionar flujos de aprobación de Power Automate en Microsoft 365."
---

# graph-approvals

Usa la herramienta `graph_approvals` para gestión de aprobaciones de Power Automate vía Microsoft Graph.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_approvals` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

---

## Acciones soportadas

- `pending` — aprobaciones pendientes de acción (default)
- `all` — todas las aprobaciones recientes con resumen
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

→ Presenta el detalle disponible:
  Agente: "Solicitud de Carlos Pérez (Compras):
           Título: Solicitud de equipos TI
           Descripción: Compra de 5 laptops para nuevos ingresos
           Monto: $45,000
           Creada: hace 2 días
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

### Sin aprobaciones pendientes

```
pending returns: count: 0

Agente: "No tienes aprobaciones pendientes en este momento.
         ¿Quieres revisar el historial, o prefieres que te avise cuando llegue alguna?"
```

### Sin historial

```
history returns: count: 0

Agente: "No encontré aprobaciones completadas en el período consultado.
         ¿Quieres ver un período diferente?"
```

### Error de permisos (403)

```
any action returns: 403

Agente: "No tengo permisos para acceder a las aprobaciones.
         Las aprobaciones de Power Automate pueden requerir consentimiento adicional.
         ¿Quieres que el administrador revise los permisos necesarios?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Usuario quiere aprobar/rechazar directamente | Muestra el detalle y pide confirmación |
| La solicitud no queda clara por el título | Presenta el detalle disponible antes de actuar |

---

## Reglas absolutas

- No expongas IDs internos de aprobación al usuario.
- Siempre muestra las aprobaciones urgentes (más de 24h en espera) al inicio.
- Si la API devuelve error de permisos, explica que Power Automate puede requerir consentimiento adicional.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_approvals` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft 365, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_approvals` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
