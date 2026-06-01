---
name: graph-mail
description: "usa esta skill cuando el usuario necesite leer, resumir, buscar, clasificar o analizar correo de Microsoft 365 mediante Microsoft Graph usando un perfil local reutilizable de tenant. úsala para correos no leídos, resúmenes ejecutivos, extracción de tareas, hilos pendientes, radar de proyectos y sugerencias de respuesta."
---

# graph-mail

Usa la herramienta `graph_mail` para escenarios de correo de Microsoft 365 vía Microsoft Graph.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_mail` con `action: "auth-login"`. NUNCA uses `web_login_playwright` o `web_login_playwright_py` para Graph.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_mail`,
**o si alguna llamada previa retornó `errorType: AUTH_ERROR`**, inicia siempre `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`unread`, `recent`, `search`, etc.).
6. Si `status = pending`, vuelve a pedir confirmación sin continuar.
7. Si `status = expired`, genera un nuevo `auth-login`.

## Formato UX obligatorio para autenticación (sin consola)

Cuando la herramienta responda `requiresAuth: true` o `errorType: AUTH_REQUIRED`, responde SIEMPRE con este formato (sin inventar pasos alternos):

1. "Para continuar, abre: **{verification_uri}**"
2. "Ingresa este código: **{user_code}**"
3. "Cuando termines, escribe: **listo**"

Reglas estrictas:
- No digas que la integración "no existe" si hay `verification_uri` y `user_code`.
- No sugieras consola al usuario final.
- No cambies a herramientas `web_login_playwright` o similares.
- Si el usuario responde "listo", reintenta la acción original (el backend intentará `auth-poll` automáticamente cuando aplique).

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

- `unread` — correos no leídos en bandeja
- `recent` — todos los correos recientes (leídos y no leídos)
- `digest` — resumen ejecutivo del buzón (requiere `period`: `day` o `week`)
- `search` — búsqueda semántica (requiere `query`)
- `read` — cuerpo completo de un correo (requiere `id`)
- `send` — enviar correo nuevo (requiere `to`, `subject`, `body`; opcional `cc`). El `body` se envía como **HTML**.
- `reply` — responder a un correo (requiere `id`, `body`)
- `mark_read` — marcar como leído/no leído (requiere `id`)
- `tasks` — extrae tareas de correos recientes
- `pending` — correos enviados sin respuesta
- `radar` — correos de un proyecto (requiere `project`)
- `suggest` — borradores de respuesta (requiere `id`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Revisar correos

```
Usuario: "¿tengo correos?"

→ Llama unread
→ Si hay correos: presenta agrupados por prioridad
  Agente: "Tienes 5 correos no leídos:
           URGENTE (requieren respuesta hoy):
           - [Juan] Re: Propuesta presupuesto → llegó hace 2h
           INFORMATIVO:
           - [Newsletter] Actualizaciones del sistema
           ¿Quieres leer alguno, responder o extraer las tareas pendientes?"

→ Si no hay correos:
  Agente: "No tienes correos no leídos en este momento.
           ¿Quieres ver los correos recientes de los últimos días o buscar alguno específico?"
```

### Buscar un correo específico

```
Usuario: "busca el correo sobre el contrato de Proveedor X"

→ Llama search con query="contrato Proveedor X"
→ Si encuentra resultados:
  Agente: "Encontré 2 correos relacionados:
           1. [María García] 'Contrato Proveedor X - revisión' — hace 3 días
           2. [Legal] 'RE: Contrato Proveedor X firmado' — ayer
           ¿Cuál quieres leer?"

→ Si no encuentra:
  Agente: "No encontré correos sobre 'contrato Proveedor X'.
           ¿Quieres que busque con otros términos? Por ejemplo 'Proveedor X',
           'contrato' a secas, o me indicas el remitente."
```

### Enviar o responder un correo

```
Usuario: "respóndele a Juan que confirmo la reunión del lunes"

→ Primero localiza el correo si no tienes el id:
  Llama search con query="Juan reunión lunes"
  Agente: "Encontré este correo de Juan Pérez del martes pasado:
           Asunto: 'Confirmación reunión lunes 27'
           ¿Es a este al que quieres responder?"

→ El usuario confirma → llama reply con id + body generado
→ Antes de enviar, muestra el borrador:
  Agente: "Voy a enviar esta respuesta:
           Para: juan.perez@empresa.com
           'Hola Juan, confirmo mi asistencia a la reunión del lunes 27. Nos vemos ahí.'
           ¿Está bien o quieres ajustar algo?"

→ El usuario aprueba → ejecuta reply → confirma
  Agente: "Respuesta enviada correctamente."
```

### Enviar correo nuevo

Parámetros:
- `to` (obligatorio): destinatario o destinatarios. Acepta una dirección,
  una lista, o varias direcciones separadas por coma.
- `subject` (obligatorio): asunto.
- `body` (obligatorio): cuerpo en **HTML**. Para saltos de línea usa
  `<br>` o envuelve párrafos en `<p>…</p>`. Nunca uses `\n` literal — el
  servicio lo renderiza como HTML y el correo llegará como un bloque
  monolítico.
- `cc` (opcional): copia. Mismo formato que `to`.

Reglas para el cuerpo HTML:
- El usuario dicta texto plano. Tú te encargas de la conversión a HTML
  antes de llamar la herramienta — el usuario nunca debe pensar en
  etiquetas.
- En el borrador que muestras al usuario, preséntalo como texto plano
  legible (sin tags). En la llamada a la herramienta, manda el HTML
  correcto.
- Si el correo lleva listas, viñetas o énfasis, usa `<ul>`, `<li>`,
  `<strong>` apropiadamente.

Flujo:

Usuario: "manda un correo a ana@empresa.com diciéndole que el reporte ya
está listo y pon en copia a su jefa, marta@empresa.com"

→ Agente muestra siempre el borrador antes de enviar, en texto plano:
  Agente: "Voy a enviar este correo:

           Para:    ana@empresa.com
           CC:      marta@empresa.com
           Asunto:  Reporte listo

           ---
           Hola Ana,

           Te informo que el reporte ya está listo. Quedo a tus órdenes
           para cualquier duda.

           Saludos
           ---

           ¿Lo envío así o quieres ajustar algo?"

→ El usuario aprueba → llama `send` con body convertido a HTML:
  body = "<p>Hola Ana,</p><p>Te informo que el reporte ya está listo.
  Quedo a tus órdenes para cualquier duda.</p><p>Saludos</p>"

→ Confirma el envío:
  Agente: "Correo enviado a ana@empresa.com (con copia a marta@empresa.com)."

---

## Recuperación de errores

### No hay correos no leídos

```
unread returns: count: 0

Agente: "No tienes correos no leídos en este momento.
         ¿Quieres que revise los correos de los últimos días,
         busque alguno en particular, o te muestre las tareas pendientes?"
```

### Búsqueda sin resultados

```
search returns: count: 0

Agente: "No encontré correos con '[término]'.
         ¿Quieres intentar con:
         1. Otro término de búsqueda
         2. Buscar por remitente específico
         3. Ampliar el rango de fechas"
```

### Correo con ID no encontrado

```
read returns: error 404

Agente: "No pude encontrar ese correo. Es posible que haya sido eliminado o movido.
         ¿Quieres que lo busque por asunto o remitente?"
```

### Error al enviar

send/reply retorna error. Lee el `errorType` y el mensaje:

- `AUTH_REQUIRED` o `AUTH_ERROR` → entra al flujo de autenticación
  (`auth-login`) descrito abajo.
- Mensaje contiene "Insufficient privileges", "ErrorAccessDenied" o
  código 403 → "No pude enviar el correo. Mi sesión no tiene permiso
  para enviar (scope `Mail.Send`). Esto requiere reautenticar con el
  scope correcto, y posiblemente que el administrador del tenant lo
  apruebe. ¿Quieres que inicie el flujo de auth de nuevo?"
- Mensaje contiene "InvalidRecipients" o el código sugiere problema de
  dirección → "La dirección parece tener un problema de formato.
  Dirección que usé: [to]. ¿La verificamos?"
- Otros → "No pude enviar el correo. Mensaje del servidor: [error].
  ¿Quieres que reintente o revisamos el contenido?"

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Solicitud de envío sin destinatario claro | "¿A quién lo envío? (nombre o correo)" |
| Solicitud de envío sin asunto | "¿Cuál sería el asunto del correo?" |
| Responder sin especificar correo | Busca el correo y muéstralo para confirmar antes de responder |
| Búsqueda con término muy genérico | "¿Puedes ser más específico? ¿Quién lo envió o cuándo fue?" |
| Usuario menciona "copia" o "CC" pero no la dirección | "¿A qué dirección pongo en copia?" |

---

## Reglas absolutas

- **Nunca envíes un correo sin mostrar el borrador al usuario y recibir aprobación explícita.**
- No expongas IDs de correos, tokens ni identificadores internos al usuario.
- Clasifica los correos por prioridad cuando presentes la bandeja: URGENTE, REQUIERE ACCIÓN, INFORMATIVO.
- Cuando extraigas tareas (`tasks`), preséntales con fecha límite si se menciona en el correo.
- El `body` de `send` y `reply` se envía como HTML. Convierte siempre el
  texto plano del usuario a HTML válido (`<p>`, `<br>`, `<ul>`, etc.)
  antes de pasarlo a la herramienta. Nunca pases `\n` ni texto sin
  formato como `body`.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_mail` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft 365, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_mail` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
