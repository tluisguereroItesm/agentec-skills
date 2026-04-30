---
name: graph-files
description: "usa esta skill cuando el usuario necesite inspeccionar, buscar, leer o resumir archivos de OneDrive o SharePoint mediante Microsoft Graph usando un perfil local reutilizable de tenant. úsala para documentos recientes, extracción de contenido, búsqueda semántica y resúmenes ligeros de documentos."
---

# graph-files

Usa la herramienta `graph_files` para escenarios de archivos en OneDrive y SharePoint vía Microsoft Graph.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_files` con `action: "auth-login"`. NUNCA uses `web_login_playwright` o `web_login_playwright_py` para Graph: esas herramientas son para portales web, no para autenticación de identidad de Microsoft.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_files`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
   - URL: `{verification_uri}`
   - Código: `{user_code}`
   - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`recent`, `search`, `read`, `summarize`).
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

## Entrada requerida
Envía estos campos a la herramienta:
- `action`

Opcional pero recomendado:
- `profile`

## Acciones soportadas
- `recent`
- `search`
- `read`
- `summarize`
- `auth-login` — inicia el flujo de device code, devuelve `user_code` y `verification_uri`
- `auth-poll`  — completa el login después de que el usuario se autentique en el navegador

## Entrada opcional
Usa estos campos cuando estén disponibles:
- `top`
- `query`
- `id`
- `maxChars`
- `siteHostname`
- `sitePath`
- `driveMode`
- `tenantIdOverride`
- `clientIdOverride`

## Comportamiento esperado
1. Resolver el perfil de Graph desde la configuración del stack cuando se proporcione `profile`.
2. Resolver el drive objetivo (`me` o basado en sitio) desde configuración o entrada explícita.
3. Invocar `graph_files`.
4. Devolver un resultado claro:
   - éxito o fallo
   - mensaje
   - datos estructurados del archivo
   - contenido extraído o resumen cuando aplique
   - ruta de artefacto cuando esté disponible

## Flujo de autenticación (device code)
Cuando la herramienta devuelva `errorType: AUTH_ERROR` o `"no existe sesión"`:

**Paso 1 — Iniciar login:**
Llama `graph_files` con `{"action": "auth-login"}`.
La respuesta incluye `user_code` y `verification_uri`.
Presenta al usuario exactamente este mensaje (en español):
> "Para autenticarte, abre **{verification_uri}** en tu navegador e ingresa el código **{user_code}**. Avísame cuando hayas iniciado sesión."

**Paso 2 — Completar login:**
Cuando el usuario confirme que ya se autenticó, llama `graph_files` con `{"action": "auth-poll"}`.
- Si `status: "ok"` → continúa con la solicitud original.
- Si `status: "pending"` → indica al usuario que complete el login y pídeles confirmación.
- Si `status: "expired"` → repite desde el Paso 1.

## Restricciones
- No elimines archivos.
- No subas ni modifiques archivos en v1.
- Prefiere perfiles/sitios por defecto configurados sobre valores hardcodeados.
- Mantén el contenido extraído acotado con `maxChars`.

## Manejo de fallos
Si la herramienta falla:
- reporta el mensaje de error exacto
- reporta el `errorType` cuando se proporcione
- indica si el fallo fue por autenticación, configuración o acceso a archivos
