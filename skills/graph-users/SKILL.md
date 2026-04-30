---
name: graph-users
description: "usa esta skill cuando el usuario necesite encontrar a una persona en la organización, consultar el correo o teléfono de alguien, identificar quién es su manager, ver reportes directos o listar usuarios por departamento en Microsoft 365."
---

# graph-users

Usa la herramienta `graph_users` para escenarios de directorio y organigrama de Microsoft 365.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_users` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_users`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`search`, `list`, `me`, `manager`, `reports`).
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

- `search` — busca usuarios por nombre o email
- `list` — lista usuarios por departamento (opcional `department`)
- `me` — perfil del usuario autenticado
- `manager` — jefe de una persona (requiere `query`)
- `reports` — reportes directos de una persona (requiere `query`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Buscar una persona

```
Usuario: "¿cuál es el correo de Ana Gómez?"

→ Llama search con query="Ana Gómez"
→ Si encuentra un resultado exacto:
  Agente: "Ana Gómez, Gerente de Proyectos
           Correo: ana.gomez@empresa.com
           Teléfono: +52 81 1234 5678
           ¿Necesitas algo más de Ana, como ver su jefe o su equipo?"

→ Si hay múltiples resultados:
  Agente: "Encontré varias personas con ese nombre:
           1. Ana Gómez Martínez — Finanzas (ana.gomez@empresa.com)
           2. Ana Gómez Ríos — RH (ana.gomez.rios@empresa.com)
           ¿Cuál de las dos?"
```

### Ver jerarquía organizacional

```
Usuario: "¿quién es el jefe de Carlos Ramos?"

→ Llama manager con query="Carlos Ramos"
→ Si encuentra:
  Agente: "El jefe de Carlos Ramos es:
           Luis Torres — Director de Operaciones
           luis.torres@empresa.com
           ¿Quieres ver también el equipo de Carlos (sus reportes directos)?"

→ Si Carlos no tiene manager registrado:
  Agente: "Carlos Ramos no tiene manager registrado en el directorio.
           Es posible que sea director o que su jerarquía no esté configurada.
           ¿Quieres ver sus reportes directos o buscar otra persona?"
```

### Listar equipo por departamento

```
Usuario: "¿quiénes están en el departamento de TI?"

→ Llama list con department="TI" o "Tecnología"
→ Si hay resultados:
  Agente: "Hay 8 personas en el área de TI:
           - Carlos Pérez — Arquitecto de Software
           - María Salinas — DevOps Engineer
           [...]
           ¿Quieres ver el detalle de alguno o ver su jefe?"

→ Si no hay resultados con ese departamento:
  Agente: "No encontré usuarios con el departamento '[nombre]'.
           El nombre puede estar configurado diferente en el directorio.
           ¿Quieres intentar con otro nombre como 'Tecnología', 'Sistemas' o 'IT'?"
```

---

## Recuperación de errores

### Persona no encontrada

```
search returns: count: 0

Agente: "No encontré a nadie con ese nombre en el directorio.
         ¿Quieres intentar con:
         1. Un nombre diferente o más corto (solo apellido)
         2. El correo electrónico directamente
         3. Buscar por departamento"
```

### Manager no encontrado (no tiene manager configurado)

```
manager returns: 404 o vacío

Agente: "[Nombre] no tiene un manager registrado en el directorio.
         Puede que sea parte de la alta dirección o que la jerarquía no esté configurada.
         ¿Quieres ver sus reportes directos o buscar a alguien más?"
```

### Sin reportes directos

```
reports returns: count: 0

Agente: "[Nombre] no tiene reportes directos registrados.
         ¿Quieres ver su perfil completo o buscar información de otra persona?"
```

### Departamento no encontrado

```
list returns: count: 0 con department especificado

Agente: "No encontré usuarios con el departamento '[nombre]'.
         Intenta con el nombre exacto como aparece en el directorio de la empresa.
         ¿Quieres que muestre todos los usuarios para ver los departamentos disponibles?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Múltiples usuarios con el mismo nombre | Muestra la lista y pide aclaración |
| Nombre muy común o ambiguo | "¿Tienes el apellido completo o el correo?" |
| Usuario pide info de "mi equipo" | Primero resuelve con `me` + `reports` |

---

## Reglas absolutas

- No expongas IDs internos de usuario al usuario final.
- Presenta `displayName` como nombre principal, no `userPrincipalName`.
- Si hay múltiples coincidencias, SIEMPRE muestra todas y pregunta cuál quiere.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_users` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft 365, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_users` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
