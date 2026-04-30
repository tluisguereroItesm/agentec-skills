---
name: graph-teams
description: "usa esta skill cuando el usuario necesite ver sus grupos de Microsoft Teams, explorar canales, leer mensajes de canal, enviar un mensaje a un canal de Teams, ver quién está en un equipo o acceder a sus chats."
---

# graph-teams

Usa la herramienta `graph_teams` para interactuar con Microsoft Teams.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_teams` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_teams`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`teams`, `channels`, `messages`, etc.).
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

---

## Acciones soportadas

- `teams` — lista los equipos del usuario (default)
- `channels` — canales de un equipo (requiere `teamId`)
- `messages` — mensajes recientes de un canal (requiere `teamId`, `channelId`)
- `send_message` — envía mensaje a un canal (requiere `teamId`, `channelId`, `body`)
- `chats` — lista chats 1:1 y grupales
- `members` — miembros de un equipo (requiere `teamId`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Explorar equipos y canales

```
Usuario: "¿en qué equipos de Teams estoy?"

→ Llama teams
→ Presenta lista:
  Agente: "Perteneces a 4 equipos:
           1. Agentec Dev
           2. Dirección General
           3. Soporte TI
           4. Proyecto Alpha
           ¿Quieres ver los canales de alguno o leer mensajes recientes?"
```

### Leer mensajes de un canal

```
Usuario: "¿qué se dijo en el canal General de Agentec Dev?"

→ Si no tienes teamId: llama teams → identifica 'Agentec Dev'
→ Si no tienes channelId: llama channels con teamId → identifica 'General'
→ Llama messages
→ Si hay mensajes recientes:
  Agente: "Últimos mensajes en Agentec Dev / General:
           [Juan Pérez, hace 2h]: 'El build está listo para QA'
           [María L, hace 1h]: 'Revisando, confirmo en 30 min'
           ¿Hay algo en lo que quieras profundizar o responder?"

→ Si no hay mensajes recientes:
  Agente: "No hay mensajes recientes en ese canal.
           ¿Quieres revisar otro canal o enviar un mensaje?"
```

### Enviar mensaje a un canal

```
Usuario: "manda al canal General de Agentec que el deploy quedó listo"

→ Si no tienes teamId/channelId: resuelve navegando teams → channels
→ Muestra el borrador SIEMPRE antes de enviar:
  Agente: "Voy a enviar este mensaje al canal General de Agentec Dev:
           '---
           El deploy quedó listo.
           ---'
           ¿Lo envío así o quieres modificar algo?"

→ Solo envía con aprobación explícita del usuario
→ Después del envío:
  Agente: "Mensaje enviado correctamente al canal General de Agentec Dev."
```

---

## Recuperación de errores

### Equipo no encontrado por nombre

```
teams returns: no match

Agente: "No encontré un equipo con el nombre '[nombre]'.
         Estos son los equipos a los que perteneces:
         [lista]
         ¿En cuál quieres buscar el canal?"
```

### Canal no encontrado

```
channels returns: no match

Agente: "No encontré el canal '[nombre]' en el equipo '[equipo]'.
         Los canales disponibles son:
         [lista]
         ¿En cuál quieres leer o enviar el mensaje?"
```

### Sin mensajes recientes

```
messages returns: count: 0

Agente: "No hay mensajes recientes en ese canal.
         ¿Quieres ver mensajes de otro canal, o prefieres enviar uno?"
```

### Error al enviar (403)

```
send_message returns: 403

Agente: "No tengo permisos para enviar mensajes a ese canal.
         Es posible que el canal sea de solo lectura o que el equipo requiera permisos adicionales.
         ¿Quieres enviar el mensaje a otro canal?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Más de un equipo con nombre similar | "¿Cuál de estos equipos?" |
| Más de un canal con nombre similar | "¿Cuál de estos canales?" |
| Mensaje ambiguo o muy corto | "¿Quieres agregar algo más al mensaje?" |
| No está claro el equipo destino | "¿A qué equipo quieres enviar el mensaje?" |

---

## Reglas absolutas

- **Nunca envíes un mensaje sin mostrar el borrador y recibir aprobación explícita.**
- No expongas teamIds ni channelIds internos al usuario.
- Cuando el usuario menciona un equipo o canal por nombre, primero resuélvelo con `teams`/`channels`.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_teams` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft Teams, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_teams` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
