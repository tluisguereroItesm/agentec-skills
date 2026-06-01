# agentec-skills

Repositorio central de skills (instrucciones de contexto) para AgenTEC/OpenClaw. Cada skill es un archivo `SKILL.md` que OpenClaw monta en su contexto y que define cuándo activarse, cómo invocar la tool correspondiente y cómo manejar errores y autenticación.

---

## Posición en el ecosistema

```
agentec-skills          ← este repo
  skills/<nombre>/
    SKILL.md            → instrucciones para el agente
    agents/             → configuraciones específicas por proveedor (openai.yaml, etc.)
    references/         → diagramas y referencias de flujo

        │
        ▼  aprobadas en
agentec-catalog
  skills/approved-skills.yaml

        │
        ▼  montadas como volumen :ro en
agentec-openclaw-stack → OpenClaw gateway
  /home/node/.openclaw/skills/
```

Las skills **no ejecutan código**. Son instrucciones de lenguaje natural que el agente sigue al recibir una solicitud. La ejecución real ocurre en la tool correspondiente (de `agentec-tools`), invocada a través del servidor MCP de `agentec-catalog`.

---

## Catálogo de skills

### Web / Automatización de navegador

| Skill | Tool que usa | Cuándo activarla |
|---|---|---|
| `web-login-monitor` | `web_login_playwright` | Login web con Playwright (Node.js): monitoreo, pruebas de humo, validación de autenticación |
| `web-login-monitor-py` | `web_login_playwright_py` | Login web con Playwright (Python): depuración local, shadow mode, paridad de backends |
| `web-document-fetch` | `web_fetch_download` | Descarga de documentos detrás de login/navegación multistep; extracción de `videoId` de YouTube |

### Microsoft 365 / Microsoft Graph

Todas las skills Graph usan el mismo flujo de autenticación device code. Ver [Flujo de autenticación Graph](#auth-graph).

| Skill | Tool que usa | Cuándo activarla |
|---|---|---|
| `graph-mail` | `graph_mail` | Leer, resumir, buscar o analizar correo de Microsoft 365 |
| `graph-files` | `graph_files` | Inspeccionar, buscar, leer o descargar archivos de OneDrive / SharePoint |
| `graph-files-write` | `graph_files_write` | Subir, crear carpetas, mover, renombrar, eliminar o compartir archivos |
| `graph-calendar` | `graph_calendar` | Ver agenda, crear/editar/cancelar eventos, buscar espacios libres |
| `graph-teams` | `graph_teams` | Ver equipos, canales, leer/enviar mensajes en Microsoft Teams |
| `graph-users` | `graph_users` | Buscar personas, ver email/teléfono, manager, reportes directos o listar por departamento |
| `graph-sharepoint-search` | `graph_sharepoint_search` | Buscar documentos y contenido en SharePoint / OneDrive de la organización |
| `graph-approvals` | `graph_approvals` | Ver aprobaciones pendientes, historial y gestionar flujos de aprobación de Power Automate |
| `graph-flows` | `graph_flows` | Listar, ver estado, ejecutar, habilitar/deshabilitar flujos de Power Automate |
| `graph-powerbi` | `graph_powerbi` | Explorar workspaces, reportes, datasets; hacer preguntas sobre datos reales de Power BI |

### Documentos y contenido

| Skill | Tool que usa | Cuándo activarla |
|---|---|---|
| `doc-summarize` | `doc_reader` | Leer, extraer o resumir documentos locales (PDF, DOCX, XLSX, TXT, MD) |
| `documentos-oficiales-mx` | `curp_downloader` | Cualquier solicitud de CURP o documentos de gobierno mexicano (RENAPO, gob.mx/curp) |

### Operaciones del stack

| Skill | Tool que usa | Cuándo activarla |
|---|---|---|
| `cleanup` | `cleanup` | Limpiar artifacts acumulados (JSONs, screenshots), logs y archivos temporales del stack |

### Skills transversales (sin tool directa)

Estas skills no invocan herramientas: definen patrones reutilizables aplicados por otras skills.

| Skill | Propósito |
|---|---|
| `common-validacion` | Validar campos requeridos, formatos y precondiciones de autenticación antes de invocar cualquier tool |
| `common-notificaciones` | Plantillas de mensajes de salida: éxito, error, escalamiento, progreso |

---

<a id="auth-graph"></a>
## Flujo de autenticación Microsoft Graph

Todas las skills `graph-*` siguen el mismo protocolo de autenticación device code. La skill correspondiente lo inicia si no hay sesión activa o si la tool devuelve `errorType: AUTH_ERROR`:

```
1. skill llama: graph_<tool>({ action: "auth-login" })
   → respuesta: { verification_uri, user_code }

2. El agente muestra al usuario:
   "Abre: <verification_uri>"
   "Ingresa el código: <user_code>"
   "Cuando termines, escribe: listo"

3. Usuario confirma → skill llama: graph_<tool>({ action: "auth-poll" })
   → status: ok  → continuar con la acción original
   → status: pending → volver a esperar confirmación
   → status: expired → reiniciar con auth-login
```

**Regla estricta**: nunca usar `web_login_playwright` ni `web_login_playwright_py` para autenticar herramientas Graph. Son herramientas distintas para portales web, no para identidad de Microsoft.

---

## Estructura de una skill

```
skills/<nombre>/
├── SKILL.md              # instrucciones para OpenClaw (requerido)
├── agents/
│   └── openai.yaml       # system prompt o config específica del proveedor (opcional)
└── references/
    └── flow.md           # diagrama o descripción del flujo (opcional)
```

El frontmatter YAML de `SKILL.md` define cuándo el agente activa la skill:

```yaml
---
name: <nombre>
description: "condición de activación en lenguaje natural"
---
```

---

## Agregar una nueva skill

1. Crear el directorio `skills/<nombre>/`.
2. Crear `skills/<nombre>/SKILL.md` con frontmatter `name` y `description`.
3. Definir en el cuerpo: cuándo usarla, entrada requerida/opcional, comportamiento esperado, restricciones y manejo de errores.
4. (Opcional) Agregar `agents/openai.yaml` si la skill tiene configuración específica por proveedor.
5. Registrar la skill en `agentec-catalog/skills/approved-skills.yaml` con `status: approved`.

---

## Repositorios relacionados

| Repo | Rol |
|---|---|
| [agentec-tools](https://github.com/tluisguereroItesm/agentec-tools) | Implementaciones ejecutables de cada tool que las skills invocan |
| [agentec-catalog](https://github.com/tluisguereroItesm/agentec-catalog) | Lista de aprobación (`approved-skills.yaml`) y servidor MCP |
| [agentec-openclaw-stack](https://github.com/tluisguereroItesm/agentec-openclaw-stack) | Monta este repo como volumen `:ro` en el contenedor de OpenClaw |
| [openclaw](https://github.com/openclaw/openclaw) | Gateway de IA que carga y ejecuta las skills en tiempo de ejecución |
