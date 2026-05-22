---
name: graph-sharepoint-search
description: "usa esta skill cuando el usuario necesite buscar documentos, archivos o contenido en sitios de SharePoint u OneDrive de la organización. úsala para encontrar contratos, reportes, minutas, políticas o cualquier documento organizacional por tema o palabra clave."
---

# graph-sharepoint-search

Usa la herramienta `graph_sharepoint_search` para búsquedas organizacionales de documentos y contenido vía Microsoft Search API.

> **IMPORTANTE**: Para autenticación de Microsoft Graph, usa SIEMPRE `graph_sharepoint_search` con `action: "auth-login"`. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_sharepoint_search`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
   - URL: `{verification_uri}`
   - Código: `{user_code}`
   - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`search`, `list-sites`).
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

- `search` — búsqueda en todo SharePoint y OneDrive (requiere `query`)
- `list-sites` — lista los sitios de SharePoint de la organización
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Buscar un documento

```
Usuario: "busca el contrato de Proveedor X"

→ Llama search con query="contrato Proveedor X"
→ Si hay resultados:
  Agente: "Encontré 3 documentos relacionados:
           1. 'Contrato Marco Proveedor X 2024.pdf' — SharePoint Compras — modificado hace 3 días
              [Ver documento](webUrl)
           2. 'Proveedor X — Addendum Enero.docx' — OneDrive María L.
              [Ver documento](webUrl)
           3. 'Evaluación Proveedor X Q1.xlsx' — SharePoint Finanzas
              [Ver documento](webUrl)
           ¿Quieres leer el contenido de alguno o necesitas otro?"

→ Si no hay resultados:
  (ver Recuperación de errores)
```

### Buscar sin saber el término exacto

```
Usuario: "busca el acta de la última reunión del consejo"

→ Llama search con query="acta reunión consejo"
→ Si encuentra documentos recientes de actas:
  Agente: "Encontré estas actas relacionadas con el consejo:
           1. 'Acta Consejo Directivo Marzo 2025.docx' — hace 2 semanas
           2. 'Acta Consejo Feb 2025.pdf' — hace 6 semanas
           ¿Es alguno de estos?"
```

### Explorar sitios de SharePoint

```
Usuario: "¿qué sitios de SharePoint tenemos?"

→ Llama list-sites
→ Presenta la lista:
  Agente: "La organización tiene 6 sitios de SharePoint:
           - Compras — [Abrir sitio](url)
           - RH — [Abrir sitio](url)
           - TI / agentec — [Abrir sitio](url)
           [...]
           ¿Quieres buscar documentos en alguno en específico?"
```

---

## Recuperación de errores

### Sin resultados para el término de búsqueda

```
search returns: count: 0

Agente: "No encontré documentos con el término '[query]'.
         ¿Quieres intentar con:
         1. Un término más corto o general (ej. 'contrato' en vez de 'contrato marco proveedor')
         2. Solo parte del nombre del documento
         3. Buscar en un sitio de SharePoint específico
         También puedo mostrarte la lista de sitios disponibles."
```

### Muchos resultados poco relevantes

```
search returns: muchos resultados con término genérico

Agente: "Encontré [N] resultados para '[query]'. Los más recientes:
         [top 3 resultados]
         Si no está aquí, ¿puedes darme más contexto? Por ejemplo: ¿en qué sitio está,
         quién lo creó, o en qué fecha aproximada?"
```

### Error de permisos (403)

```
search returns: 403

Agente: "No tengo acceso para buscar en los repositorios de la organización.
         Es posible que se necesiten permisos adicionales en el tenant.
         Contacta al administrador de Microsoft 365."
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Término muy genérico | "¿Tienes más contexto? ¿En qué sitio o quién lo creó?" |
| El usuario quiere abrir o leer el contenido | Sugiere usar `graph_files` para leer el archivo |

---

## Reglas absolutas

- No expongas IDs internos de SharePoint ni drive IDs al usuario.
- Presenta `webUrl` como enlaces cliqueables `[Ver documento](url)`.
- Si el usuario quiere leer el contenido de un archivo, indícale usar `graph_files` con el enlace.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_sharepoint_search` con `{"action": "auth-login"}`.
> "Para autenticarte en Microsoft 365, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_sharepoint_search` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
