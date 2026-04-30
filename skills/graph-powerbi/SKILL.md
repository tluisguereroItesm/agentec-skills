---
name: graph-powerbi
description: "usa esta skill cuando el usuario quiera explorar workspaces de Power BI, encontrar reportes o dashboards, hacer preguntas respondidas con datos reales de un dataset (sin alucinaciones), abrir un reporte en el navegador, ver páginas del reporte, mosaicos del dashboard, esquema del dataset o gestionar refreshes del dataset."
---

# graph-powerbi

Usa la herramienta `graph_powerbi` para todas las interacciones de Power BI.

> **IMPORTANTE**: Power BI requiere su propio token de autenticación (separado de correo/archivos). Usa SIEMPRE `graph_powerbi` con `action: "auth-login"` cuando se necesite autenticación. NUNCA uses `web_login_playwright` para esto.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_powerbi`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`workspaces`, `reports`, `query`, etc.).
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

- `workspaces` — lista workspaces accesibles (filtra con `search`)
- `reports` — lista/busca reportes en un workspace (requiere `workspaceId`)
- `dashboards` — lista/busca dashboards en un workspace (requiere `workspaceId`)
- `datasets` — lista datasets de un workspace (requiere `workspaceId`)
- `schema` — tablas, columnas y medidas de un dataset (requiere `workspaceId`, `datasetId`)
- `query` — ejecuta DAX y devuelve datos REALES (requiere `workspaceId`, `datasetId`, `dax`)
- `open` — URL del reporte para abrir en navegador (requiere `workspaceId`, `reportId`)
- `pages` — páginas/pestañas de un reporte (requiere `workspaceId`, `reportId`)
- `tiles` — mosaicos de un dashboard (requiere `workspaceId`, `dashboardId`)
- `refresh` — historial de refreshes o dispara uno nuevo (requiere `workspaceId`, `datasetId`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Exploración de reporte (el usuario sabe lo que quiere)

```
Usuario: "muéstrame el reporte de ventas"

→ Si no tienes workspaceId: llama workspaces primero
→ Si hay más de un workspace: pregunta cuál
  Agente: "Tengo 3 workspaces disponibles:
           1. Finanzas
           2. Operaciones
           3. PruebasF2
           ¿En cuál quieres buscar el reporte?"

→ Con workspaceId: llama reports con search="ventas"
→ Si encuentra exactamente uno: presenta y pregunta qué quiere hacer
  Agente: "Encontré el reporte 'Dashboard Ventas 2026'.
           ¿Quieres abrirlo en el navegador, ver sus páginas o consultar datos específicos?"

→ Si encuentra más de uno: lista y pregunta cuál
  Agente: "Encontré 2 reportes sobre ventas:
           1. Dashboard Ventas 2026
           2. Ventas por Región Q1
           ¿Cuál te interesa?"

→ Si no encuentra ninguno: ofrece alternativas (ver recuperación de errores abajo)
```

### Pregunta de datos (el usuario quiere una cifra real)

```
Usuario: "¿cuánto vendimos en marzo?"

→ Paso 1: workspaces → identifica workspace relevante
  (Si hay ambigüedad: "¿Los datos de ventas están en Finanzas o en Operaciones?")

→ Paso 2: datasets con workspaceId → identifica dataset de ventas
  (Si hay varios: "¿Usamos el dataset 'Ventas_2026' o 'Historico_Ventas'?")

→ Paso 3: schema con workspaceId + datasetId
  (Necesario para conocer nombres exactos de tablas y columnas)

→ Paso 4: query con DAX basado en el schema obtenido
  NUNCA inventes nombres de tablas o columnas: usa exactamente los del esquema

→ Presenta los datos reales:
  Agente: "Según los datos del dataset 'Ventas_2026':
           Total vendido en marzo 2026: $1,234,567
           (Dato obtenido en tiempo real del dataset)"
```

### Abrir reporte en navegador

```
Usuario: "abre el reporte de KPIs ejecutivos"

→ Llama reports (con search si aplica) → obtiene reportId
→ Llama open con workspaceId + reportId
→ Presenta como enlace:
  Agente: "Aquí está el enlace para abrir el reporte:
           [KPIs Ejecutivos 2026](https://app.powerbi.com/...)
           También puedes preguntarme sobre datos específicos dentro del reporte."
```

---

## Recuperación de errores

### Reporte no encontrado

```
graph_powerbi returns: count: 0 / lista vacía

Agente: "No encontré reportes con el nombre '[búsqueda]' en el workspace '[nombre]'.
         ¿Quieres que:
         1. Busque en otro workspace
         2. Busque con un término diferente
         3. Muestre todos los reportes disponibles"

→ Espera respuesta del usuario antes de continuar
→ NO asumas ni inventes resultados
```

### Dataset sin datos en el rango consultado

```
query returns: rows: []

Agente: "La consulta no devolvió datos para '[período/filtro]'.
         Posibles causas:
         - El rango de fechas no tiene registros
         - El nombre del campo del filtro es diferente al esperado
         ¿Quieres que revise el esquema del dataset para confirmar los campos disponibles,
         o prefieres consultar un período diferente?"
```

### Error de permisos (403)

```
graph_powerbi returns: PBI_ERROR: [403]

Agente: "No tengo acceso a este workspace o dataset.
         Esto ocurre cuando el reporte es privado o no se ha compartido contigo.
         ¿Quieres buscar en otro workspace al que sí tengas acceso?"
```

### Error de DAX (falla la query)

```
graph_powerbi returns: error en executeQueries

Agente: "La consulta DAX falló. Voy a revisar el schema del dataset para
         confirmar los nombres exactos de tablas y columnas, y reintentaré."

→ Llama schema nuevamente → ajusta el DAX → reintenta
→ Si falla 2 veces:
  Agente: "No pude completar la consulta automáticamente.
           Los campos disponibles en el dataset son: [lista del schema].
           ¿Quieres reformular la pregunta?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| Hay más de 1 workspace | "¿En cuál de estos workspaces está lo que buscas?" |
| Hay más de 1 reporte con nombre similar | "¿Cuál de estos reportes?" |
| El usuario pide datos sin especificar período | "¿Para qué período? (mes, trimestre, año...)" |
| El usuario pide datos sin especificar medida | "¿Qué métrica exactamente? (ventas, unidades, margen...)" |
| Se va a disparar un refresh | "¿Confirmas que quieres actualizar el dataset '[nombre]'? El proceso puede tardar varios minutos." |

---

## Reglas absolutas

- **NUNCA inventes datos numéricos.** Toda cifra debe provenir de `action: "query"` con DAX real.
- **Usa siempre el schema antes de construir DAX** para garantizar nombres exactos.
- Presenta números con formato legible: `$1,234,567` no `1234567`.
- Cuando presentes un reporte, ofrece siempre la opción de consultar datos específicos.
- Si el usuario hace una pregunta de negocio ("¿cómo van las ventas?"), tradúcela a DAX — no respondas con texto genérico.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_powerbi` con `{"action": "auth-login"}`.
> "Para conectarte a Power BI, abre **{verification_uri}** en tu navegador e ingresa el código **{user_code}**.
> Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_powerbi` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original desde el principio.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código en la página?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
