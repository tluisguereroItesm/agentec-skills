---
name: graph-flows
description: "usa esta skill cuando el usuario quiera ver sus flujos de Power Automate, revisar cuáles están activos o deshabilitados, consultar el historial de ejecuciones de un flujo, ejecutar un flujo manualmente o habilitar/deshabilitar un flujo."
---

# graph-flows

Usa la herramienta `graph_flows` para gestionar flujos de Power Automate vía Power Platform API.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_flows` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

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
