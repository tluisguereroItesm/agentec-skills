# SKILL: cleanup — recolector de limpieza de Agentec

## Propósito
Limpiar artifacts acumulados (JSONs de herramientas, screenshots), logs y archivos temporales del stack Agentec.
Los artifacts contienen datos sensibles (correos, calendarios, reportes Power BI) y deben purgarse regularmente.

## Herramienta
`cleanup` — Acciones: `status` | `artifacts` | `logs` | `purge`

---

## REGLA CRÍTICA
**SIEMPRE confirma con el usuario antes de borrar** (`dry_run` primero, luego pide confirmación).
Única excepción: si el usuario pide explícitamente una limpieza completa sin confirmación.

---

## FLUJOS

### Flujo 1 — Ver estado del disco
```
cleanup({ action: "status" })
```
- Muestra conteo de archivos, tamaño total, y cuántos tienen más de 7 y 30 días.
- No borra nada.
- Usa cuando el usuario pregunta: "¿qué hay en artifacts?", "¿cuánto espacio ocupan?", "¿qué se puede limpiar?"

### Flujo 2 — Limpiar artifacts por antigüedad (patrón recomendado)
**Paso 1 — Dry-run:**
```
cleanup({ action: "artifacts", dry_run: true, older_than_days: 7 })
```
**Paso 2 — Mostrar resultado al usuario y pedir confirmación.**
**Paso 3 — Ejecutar real:**
```
cleanup({ action: "artifacts", dry_run: false, older_than_days: 7 })
```

### Flujo 3 — Limpiar logs
```
cleanup({ action: "logs", dry_run: true, older_than_days: 30 })
```
Luego con `dry_run: false` si el usuario confirma.

### Flujo 4 — Borrar archivos específicos (purge)
Cuando el usuario menciona archivos por nombre:
```
cleanup({
  action: "purge",
  dry_run: true,
  target_dir: "artifacts",
  filenames: ["graph-mail-list-1777044903.json", "login-1775862860584.png"]
})
```
Confirmar y luego dry_run: false.

### Flujo 5 — Limpieza rápida total (el usuario pide "limpiar todo")
```
// Paso 1: Revisar estado
cleanup({ action: "status" })
// Paso 2: Dry-run de artifacts (más recientes primero, 7 días)
cleanup({ action: "artifacts", dry_run: true, older_than_days: 7 })
// Paso 3: Confirmar con el usuario
// Paso 4: Ejecutar
cleanup({ action: "artifacts", dry_run: false, older_than_days: 7 })
```

---

## PREGUNTAS CLARIFICADORAS

Antes de ejecutar cualquier borrado (dry_run: false), si el usuario NO fue explícito:

1. **¿Antigüedad?** — "¿Borrar artifacts de más de 7 días? (default) o prefieres un número diferente?"
2. **¿Qué tipo?** — "¿Solo JSONs, también los screenshots (.png), o todo?"
3. **¿Logs también?** — Si el usuario dijo "todo", confirma si incluye logs del stack.

---

## RECUPERACIÓN DE ERRORES

| Situación | Qué hacer |
|-----------|-----------|
| `"Patrón inválido"` | El patrón tiene `..` o `/`. Pide un patrón de solo nombre de archivo (ej: `*.json`) |
| `"target_dir inválido"` | Solo acepta `artifacts`, `logs`, `tokens`. Corregir. |
| `"directorio no encontrado"` | El stack puede no tener ese directorio montado. Usar `status` para verificar qué existe. |
| Herramienta no disponible | Sugiere al usuario correr `scripts/cleanup.sh --dry-run` directamente en el servidor. |

---

## TABLA DE REFERENCIA RÁPIDA

| Usuario dice | Acción | Parámetros clave |
|---|---|---|
| "¿cuánto espacio hay?" | `status` | — |
| "limpia los artifacts" | `artifacts` | `dry_run: true` primero |
| "borra todo lo de hace más de 30 días" | `artifacts` + `logs` | `older_than_days: 30` |
| "borra este archivo específico" | `purge` | `filenames: [...]` |
| "limpia logs" | `logs` | `older_than_days: 30` |
| "limpieza general" | `status` → `artifacts` → `logs` | siempre dry_run primero |

---

## REGLAS ABSOLUTAS

1. **Nunca** usar `dry_run: false` sin mostrar primero qué se va a borrar.
2. **Nunca** inventar nombres de archivo — usa el resultado de `status` o `artifacts (dry_run: true)`.
3. **Siempre** mencionar que los artifacts contienen datos sensibles y que el borrado es irreversible.
4. Si el usuario pide limpiar `tokens` (auth), advertir que requerirá hacer login de nuevo.

---

## OPERACIÓN EN SERVIDOR (para administradores)

Esta sección es para operadores que tienen acceso al servidor donde corre el stack.
La limpieza **no es automática** — debe dispararse manualmente o por cron.

### Script `cleanup.sh`

Ubicación: `agentec-openclaw-stack/scripts/cleanup.sh`

**Modo interactivo (recomendado)** — corre sin argumentos y hace preguntas paso a paso:

```bash
./scripts/cleanup.sh
```

El wizard pregunta lo siguiente en orden:
1. **¿Limpiar artifacts?** → sí/no + cuántos días de antigüedad (default: 7)
   - Incluye JSONs de herramientas, screenshots de login (.png), PDFs temporales
2. **¿Limpiar logs del stack?** → sí/no + días (default: 30)
3. **¿Limpiar logs y canvas de OpenClaw?** → sí/no + días (default: 14)
4. **¿Limpiar sesiones y cookies del navegador?** → sí/no
   - Limpia `~/.openclaw/devices/pending.json`, directorios temporales de Playwright/Chromium
   - Opción adicional: borrar historial de ejecuciones (`runs.sqlite`)
5. **¿Limpiar tokens de auth pendientes?** → sí/no (tokens OAuth no completados)
6. **¿Limpiar Docker?** → a) build cache / b) limpieza total / c) nada
   - Muestra el espacio actual antes de preguntar
7. **¿Dry-run primero?** → por defecto sí, muestra qué se borraría sin borrar nada
8. **Resumen de configuración + confirmación final** antes de ejecutar cualquier borrado

**Modo directo (flags)** — para scripts y cron, sin preguntas:

```bash
# Ver qué se borraría sin borrar nada
./scripts/cleanup.sh --dry-run

# Limpiar artifacts de más de 7 días
./scripts/cleanup.sh --artifacts-days 7

# Limpiar logs de más de 30 días
./scripts/cleanup.sh --logs-days 30

# Limpiar Docker (imágenes dangling + build cache, conserva 2 GB)
./scripts/cleanup.sh --docker

# Limpiar TODO Docker (imágenes, volúmenes — peligroso en producción)
./scripts/cleanup.sh --docker-all

# Limpiar sesiones y cookies del navegador web
./scripts/cleanup.sh --browser-sessions

# Limpieza completa de todo (artifacts + logs + tokens + browser + Docker)
./scripts/cleanup.sh --all

# Combinación personalizada
./scripts/cleanup.sh --artifacts-days 14 --logs-days 60 --docker
```

**Retención recomendada:**

| Tipo | Retención sugerida | Razón |
|---|---|---|
| Artifacts JSON (herramientas) | 7 días | Datos sensibles — correos, calendarios, Power BI |
| Screenshots de login (`.png`) | 7 días | Contienen imagen visual del proceso de autenticación |
| Logs del stack | 30 días | Útiles para troubleshooting de incidentes recientes |
| Logs de OpenClaw (`~/.openclaw/logs`) | 14 días | Sesiones de agente — pueden contener prompts del usuario |
| Build cache Docker | Mantener 2 GB | Se limpian automáticamente con `--docker` |

---

### Automatización con cron

Si quieres que la limpieza corra sola, agrega una entrada en `crontab -e` del servidor:

```bash
# Limpieza diaria a las 3 AM — artifacts de más de 7 días y logs de más de 30 días
0 3 * * *  /ruta/al/agentec-openclaw-stack/scripts/cleanup.sh --artifacts-days 7 --logs-days 30 >> /home/lpgg/agentec/logs/cleanup-cron.log 2>&1

# Limpieza de Docker los domingos a las 4 AM
0 4 * * 0  /ruta/al/agentec-openclaw-stack/scripts/cleanup.sh --docker >> /home/lpgg/agentec/logs/cleanup-cron.log 2>&1
```

Reemplaza `/ruta/al/` con la ruta real al stack (ej: `/home/lpgg/agentec/agentec-openclaw-stack`).

---

### Actualizar la versión de OpenClaw de forma segura

La imagen de OpenClaw está en `:latest` por defecto, lo que puede introducir breaking changes sin aviso.
Usa `update-openclaw.sh` para actualizar con control:

```bash
# Paso 1 — Ver si hay una versión nueva (no hace cambios)
./scripts/update-openclaw.sh

# Paso 2 — Si hay update, revisar el CHANGELOG antes de aplicar:
# https://github.com/openclaw/openclaw/releases

# Paso 3 — Aplicar el update y pinear el digest SHA256 en .env
./scripts/update-openclaw.sh --apply
# → Te pide confirmación antes de modificar .env
# → Actualiza OPENCLAW_IMAGE=ghcr.io/openclaw/openclaw@sha256:<digest>

# Paso 4 — Reiniciar el stack para aplicar
docker compose down && docker compose up -d
```

> **Por qué fijar con SHA256 y no con tag de versión:**  
> Un tag como `v1.2.3` puede ser reescrito en el registry (aunque es poco probable en proyectos serios).  
> El SHA256 del digest es inmutable — garantiza que siempre se usa exactamente el mismo binario.
