---
name: graph-flows
description: "usa esta skill cuando el usuario quiera ver sus flujos de Power Automate, revisar cuáles están activos o deshabilitados, consultar el historial de ejecuciones de un flujo, ejecutar un flujo manualmente o habilitar/deshabilitar un flujo."
---

# graph-flows

Usa la herramienta `graph_flows` para gestionar flujos de Power Automate vía Power Platform API.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_flows` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_flows`,
**o si alguna llamada previa retornó `errorType: AUTH_ERROR`**, inicia siempre `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`list`, `read`, `runs`, `trigger`, etc.).
6. Si `status = pending`, vuelve a pedir confirmación sin continuar.
7. Si `status = expired`, genera un nuevo `auth-login`.

## Formato UX obligatorio para autenticación (sin consola)

Cuando la herramienta responda `requiresAuth: true` o `errorType: AUTH_REQUIRED`, responde SIEMPRE con este formato:

1. "Para continuar, abre: **{verification_uri}**"
2. "Ingresa este código: **{user_code}**"
3. "Cuando termines, escribe: **listo**"

Reglas estrictas:
- No afirmar que la integración no existe si hay URL+código.
- No pedir consola al usuario final.
- No redirigir a `web_login_playwright`.
- Tras "listo", reintentar la acción original.

## Identificación del usuario

La herramienta opera por defecto sobre la sesión Graph del **owner** del
profile configurado (el humano que hizo `auth-login` originalmente).
**No pases el parámetro `user`** salvo que tengas instrucciones explícitas
para gestionar sesiones de múltiples usuarios bajo el mismo profile.

Si necesitas operar sobre un buzón distinto al del owner (por ejemplo, un
buzón compartido o el de otro empleado al que tienes permisos delegados),
usa el parámetro **`graphUserId`** con el UPN o ID del usuario destino.
Eso no cambia la sesión: usa la misma autenticación, solo redirige las
llamadas a `/users/{graphUserId}` en vez de `/me`.

---

## Acciones soportadas

- `list` — lista todos los flows del usuario
- `read` — detalle de un flow específico (requiere `flowId`)
- `runs` — historial de ejecuciones (requiere `flowId`)
- `trigger` — ejecuta un flow manual (requiere `flowId`)
- `enable` — activa un flow deshabilitado (requiere `flowId`)
- `disable` — deshabilita un flow activo (requiere `flowId`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Revisar flows

```
Usuario: "¿qué flujos tengo activos?"

→ Llama list
→ Presenta por estado:
  Agente: "Tienes 5 flujos de Power Automate:
           Activos:
           - Notificaciones de facturación (automático, última ejecución: hace 2h)
           - Bienvenida nuevos usuarios (manual)
           - Alerta aprobaciones pendientes (automático)
           Deshabilitados:
           - Reporte semanal ventas
           - Sincronización CRM
           ¿Quieres ver el historial de alguno o ejecutar uno?"

→ Si no hay flows:
  Agente: "No encontré flujos de Power Automate en tu cuenta.
           ¿Es posible que estén en otro entorno?"
```

### Verificar un flow específico

```
Usuario: "¿está activo el flujo de notificaciones?"

→ Llama list con search implícito
→ Si encuentra el flow:
  Agente: "Sí, el flujo 'Notificaciones de facturación' está activo.
           Última ejecución: hace 2 horas (exitosa).
           ¿Quieres ver el historial completo o hacer algo con él?"

→ Si no lo encuentra:
  Agente: "No encontré un flow con ese nombre. Estos son tus flows actuales:
           [lista]
           ¿Era alguno de éstos?"
```

### Ejecutar un flow manual

```
Usuario: "ejecuta el flujo de bienvenida"

→ Localiza el flow (list si es necesario)
→ Verifica que tiene trigger manual:
  Si no tiene trigger manual:
  Agente: "El flujo '[nombre]' no tiene un trigger manual configurado.
           Solo se puede ejecutar automáticamente. ¿Hay algo más que pueda hacer?"

→ Si tiene trigger manual: pide confirmación:
  Agente: "¿Confirmas que quieres ejecutar el flujo 'Bienvenida nuevos usuarios' ahora?
           Esto iniciará el proceso inmediatamente."

→ Usuario confirma → llama trigger
  Agente: "Flujo iniciado correctamente. La ejecución corre en segundo plano.
           Puedes revisar el resultado con el historial de ejecuciones."
```

### Deshabilitar / habilitar un flow

```
Usuario: "deshabilita el flujo de reportes semanales"

→ Localiza el flow → pide confirmación:
  Agente: "¿Confirmas que quieres deshabilitar 'Reporte semanal ventas'?
           El flujo dejará de ejecutarse automáticamente hasta que lo reactives."

→ Usuario confirma → llama disable
  Agente: "Flujo deshabilitado. Puedes reactivarlo cuando quieras."
```

---

## Recuperación de errores

### Flow no encontrado (404)

```
read/runs/trigger returns: 404

Agente: "No encontré ese flow. Es posible que el nombre haya cambiado.
         Estos son tus flows actuales:
         [lista de list]
         ¿Era alguno de éstos?"
```

### Flow sin trigger manual

```
trigger returns: error de tipo de trigger

Agente: "Este flow no se puede ejecutar manualmente. Está configurado para
         dispararse automáticamente por [tipo de trigger].
         ¿Quieres revisar su historial de ejecuciones o hacer otra cosa?"
```

### Error de permisos (403)

```
any action returns: 403

Agente: "No tengo permisos para [acción] este flow.
         Es posible que pertenezca a otro usuario o entorno.
         ¿Quieres que busque en otro entorno o revises los permisos?"
```

### Sin historial de ejecuciones

```
runs returns: count: 0

Agente: "Este flow no ha tenido ejecuciones recientes.
         ¿Quieres ejecutarlo manualmente o revisar su configuración?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Varios flows con nombre similar | "¿Cuál de estos flows?" |
| Trigger o disable sin confirmación | Siempre pide confirmación explícita |
| Usuario pide ejecutar sin especificar flow | "¿Qué flow quieres ejecutar?" |

---

## Reglas absolutas

- **Nunca ejecutes, habilites ni deshabilites un flow sin confirmación explícita del usuario.**
- `trigger` solo funciona para flows con trigger manual configurado; infórmalo si no aplica.
- Si un flow falla frecuentemente (historial con múltiples errores), sugiérelo al usuario.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_flows` con `{"action": "auth-login"}`.
> "Para autenticarte en Power Automate, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_flows` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
