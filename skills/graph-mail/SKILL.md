---
name: graph-mail
description: "usa esta skill cuando el usuario necesite leer, resumir, buscar, clasificar o analizar correo de Microsoft 365 mediante Microsoft Graph usando un perfil local reutilizable de tenant. úsala para correos no leídos, resúmenes ejecutivos, extracción de tareas, hilos pendientes, radar de proyectos y sugerencias de respuesta."
---

# graph-mail

Usa la herramienta `graph_mail` para escenarios de correo de Microsoft 365 vía Microsoft Graph.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_mail` con `action: "auth-login"`. NUNCA uses `web_login_playwright` o `web_login_playwright_py` para Graph.

---

## Acciones soportadas

- `unread` — correos no leídos en bandeja
- `recent` — todos los correos recientes (leídos y no leídos)
- `digest` — resumen ejecutivo del buzón (requiere `period`: `day` o `week`)
- `search` — búsqueda semántica (requiere `query`)
- `read` — cuerpo completo de un correo (requiere `id`)
- `send` — enviar correo nuevo (requiere `to`, `subject`, `body`)
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

```
Usuario: "manda un correo a ana@empresa.com diciéndole que el reporte ya está listo"

→ Agente siempre muestra el borrador antes de enviar:
  Agente: "Voy a enviar este correo:
           Para: ana@empresa.com
           Asunto: Reporte listo
           Cuerpo: 'Hola Ana, te informo que el reporte ya está listo.
           Quedo a tus órdenes para cualquier duda.'
           ¿Lo envío así o prefieres modificar algo?"

→ Solo envía después de aprobación explícita del usuario
```

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

```
send/reply returns: error

Agente: "No pude enviar el correo. Posibles causas:
         - La dirección de correo puede tener un error tipográfico
         - La sesión puede haber expirado
         Dirección que usé: [to]. ¿Es correcta?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Solicitud de envío sin destinatario claro | "¿A quién lo envío? (nombre o correo)" |
| Solicitud de envío sin asunto | "¿Cuál sería el asunto del correo?" |
| Responder sin especificar correo | Busca el correo y muéstralo para confirmar antes de responder |
| Búsqueda con término muy genérico | "¿Puedes ser más específico? ¿Quién lo envió o cuándo fue?" |

---

## Reglas absolutas

- **Nunca envíes un correo sin mostrar el borrador al usuario y recibir aprobación explícita.**
- No expongas IDs de correos, tokens ni identificadores internos al usuario.
- Clasifica los correos por prioridad cuando presentes la bandeja: URGENTE, REQUIERE ACCIÓN, INFORMATIVO.
- Cuando extraigas tareas (`tasks`), preséntales con fecha límite si se menciona en el correo.

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
