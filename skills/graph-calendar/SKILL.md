---
name: graph-calendar
description: "usa esta skill cuando el usuario quiera revisar su agenda, ver qué reuniones tiene hoy o esta semana, crear una nueva reunión o evento, actualizar o cancelar un evento existente, encontrar espacios libres o gestionar invitaciones de calendario en Microsoft 365."
---

# graph-calendar

Usa la herramienta `graph_calendar` para leer y gestionar el calendario de Microsoft 365.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_calendar` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_calendar`,
**o si alguna llamada previa retornó `errorType: AUTH_ERROR`**, inicia siempre `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`today`, `week`, `month`, `create`, `update`, etc.).
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

Zona horaria: la herramienta opera en la zona local del owner por defecto. 
No envíes el parámetro timezone en create/update salvo que el usuario 
explícitamente pida agendar en otra zona (ej: "agenda esta llamada en hora de Madrid"). 
Cuando el usuario dice "mañana a las 10", asume hora local; no conviertas 
a UTC ni a ningún offset antes de pasar start/end.

Pasa start y end como hora naive (sin Z, sin offset, sin sufijo). Ejemplo correcto: 
"2026-05-21T10:00:00". Ejemplos incorrectos: "2026-05-21T10:00:00Z", 
"2026-05-21T10:00:00-06:00". La zona la gestiona la herramienta por separado.

---

## Acciones soportadas

- `today` — eventos de hoy
- `week` — agenda de los próximos 7 días
- `month` — agenda del mes (30 días)
- `read` — detalle de un evento específico (requiere `id`)
- `create` — crear evento o reunión (requiere `subject`, `start`, `end`)
- `update` — modificar evento existente (requiere `id`)
- `delete` — cancelar/eliminar evento (requiere `id`)
- `availability` — slots libres y ocupados
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Consultar agenda

```
Usuario: "¿qué tengo hoy?"

→ Llama today
→ Si hay eventos:
  Agente: "Tienes 3 eventos hoy:
           09:00 — Standup diario (Teams, 30 min)
           11:00 — Revisión presupuesto con CFO (Sala Monterrey, 1h)
           16:00 — 1:1 con el equipo (Teams, 30 min)
           ¿Quieres ver el detalle de alguno o agendar algo más?"

→ Si no hay eventos:
  Agente: "No tienes eventos agendados para hoy.
           ¿Quieres ver la agenda de la semana o agendar una reunión?"
```

### Crear una reunión

```
Usuario: "agenda una reunión con Juan y María mañana a las 10"

→ Agente identifica los datos que tiene: subject falta
  Agente: "Entendido. ¿Cuál sería el tema o título de la reunión?"

→ Usuario responde: "revisión del sprint"
  Agente: "Voy a crear este evento:
           Título: Revisión del sprint
           Fecha: mañana (25 abr) a las 10:00 - 11:00
           Invitados: Juan, María
           ¿Es reunión de Teams o presencial?"

→ Usuario: "Teams"
→ Agente confirma antes de crear:
  Agente: "¿Creo la reunión de Teams con estos datos?
           - Título: Revisión del sprint
           - 25 abr, 10:00–11:00
           - Invitados: juan@empresa.com, maria@empresa.com"

→ Usuario confirma → llama create → confirma con webLink
  Agente: "Reunión creada. Enlace de Teams: [unirse a la reunión](webLink)"
```

### Buscar tiempo libre

```
Usuario: "¿cuándo tengo tiempo libre esta semana para una reunión de 1 hora?"

→ Llama availability
→ Presenta slots sugeridos:
  Agente: "Basándome en tu calendario, tienes estos espacios disponibles esta semana:
           - Martes 26 abr: 14:00–17:00 (3 horas libres)
           - Miércoles 27 abr: 09:00–11:00 y 15:00–17:00
           - Viernes 29 abr: toda la tarde desde las 13:00
           ¿Quieres que agende la reunión en alguno de estos horarios?"
```

### Cancelar un evento

```
Usuario: "cancela la reunión del miércoles con el equipo de finanzas"

→ Si no tienes id: busca en week con search="finanzas"
  Agente: "Encontré este evento el miércoles:
           11:00 — Revisión Q1 con Finanzas (Teams)
           ¿Es este el que quieres cancelar?"

→ El usuario confirma → muestra advertencia antes de eliminar:
  Agente: "¿Confirmas que quieres cancelar 'Revisión Q1 con Finanzas'
           del miércoles 27 a las 11:00? Los invitados recibirán la cancelación."

→ Solo cancela con confirmación → llama delete
  Agente: "Evento cancelado. Los invitados han sido notificados automáticamente."
```

---

## Recuperación de errores

### Sin eventos en el período solicitado

```
today/week/month returns: count: 0

Agente: "No tienes eventos agendados para [período].
         ¿Quieres ver otro período o crear un nuevo evento?"
```

### Conflicto de horario al crear

```
create returns: conflicto de disponibilidad

Agente: "Ya tienes un evento en ese horario: '[nombre del evento]' de [hora] a [hora].
         ¿Quieres:
         1. Programar la reunión a una hora diferente
         2. Ver tu disponibilidad del día para encontrar un hueco
         3. Crear el evento de todos modos (habrá conflicto)"
```

### Evento no encontrado para actualizar/cancelar

```
update/delete returns: 404

Agente: "No encontré el evento con ese identificador.
         Es posible que ya fue cancelado o que haya cambiado.
         ¿Quieres que busque eventos similares en tu calendario?"
```

### Datos de creación incompletos

```
Si faltan subject, start o end:

→ NO ejecutes la creación
→ Pide el dato faltante de forma específica:
  Agente: "Para crear la reunión necesito saber: [dato faltante].
           Por ejemplo: '10am', 'lunes próximo a las 3pm', etc."
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| No hay subject para crear | "¿Cuál sería el título o tema de la reunión?" |
| No hay hora de fin | "¿Cuánto tiempo durará? (30 min, 1 hora...)" |
| No está claro si es Teams o presencial | "¿Es reunión virtual de Teams o presencial?" |
| Invitados mencionados por nombre sin email | "¿Cuál es el correo de [nombre]?" |
| Fecha ambigua ("el lunes", "esta semana") | Confirma la fecha exacta antes de crear |
| Cancelar sin confirmación del evento | Muestra el evento encontrado y pide confirmación |

---

## Reglas absolutas

- **Nunca canceles o elimines un evento sin confirmación explícita del usuario.**
- Siempre muestra un resumen del evento a crear antes de ejecutar.
- Presenta las horas tomando start.dateTime y end.dateTime tal como vienen de la herramienta, sin aplicar conversiones de zona horaria. La herramienta ya devuelve las fechas en la zona local del owner (campo startTz/endTz). Reformatea solo el estilo de visualización (ej: convierte "2026-05-21T10:00:00.0000000" a "21 abr a las 10:00"), no el valor.
- Si el usuario menciona "Teams" o "virtual" al crear, agrega `isOnline: true`.
- Cuando haya conflicto de horario, informa antes de crear.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_calendar` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft 365, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_calendar` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
