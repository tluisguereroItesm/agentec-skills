---
name: common-notificaciones
description: "usa esta skill para producir mensajes de salida consistentes y estructurados al final de cualquier flujo del agente. úsala para confirmaciones de éxito, reportes de error, avisos de escalamiento y actualizaciones de estado en progreso. aplícala como paso final antes de responder al usuario."
---

# common-notificaciones

Patrones reutilizables de mensajes de salida. Aplícalos al final de cualquier flujo para producir respuestas consistentes de cara al usuario.

## Cuándo usar
- Después de que una herramienta devuelve un resultado (éxito o fallo)
- Cuando un error necesita escalarse a una persona o a otro equipo
- Cuando una operación larga está en progreso y se necesita una actualización de estado
- Cuando un flujo termina y el usuario necesita una confirmación clara

## Tipos de mensaje

### 1. Éxito
Úsalo cuando la herramienta complete la acción correctamente.

Template:
```
✅ <action_summary>

<result_detail>

<next_step_or_follow_up>  ← solo si aplica
```

Example:
```
✅ Documento descargado correctamente.

Archivo guardado: certificado-2026.pdf
Páginas: 3 · Tamaño: 128 KB

Puedes pedirme que lo lea y te haga un resumen.
```

### 2. Error — Recuperable
Úsalo cuando la herramienta falle pero el usuario pueda reintentar o aportar más información.

Template:
```
⚠️ No se pudo completar la acción: <brief_reason>

<what_went_wrong>

Para continuar necesito:
- <missing_info_1>
- <missing_info_2>
```

### 3. Error — Requiere escalamiento
Úsalo cuando el fallo no pueda resolverse por el usuario y requiera intervención humana.

Template:
```
❌ Error que requiere atención: <brief_reason>

<what_went_wrong>

Acción recomendada: <who_to_contact_or_what_to_do>

Referencia técnica: <error_type_or_code>  ← solo si es seguro compartirla
```

### 4. En progreso
Úsalo cuando una acción esté ejecutándose y el usuario deba esperar.

Template:
```
⏳ Procesando: <action_description>

Esto puede tardar unos segundos. Te avisaré cuando termine.
```

### 5. Resultado parcial
Úsalo cuando se recuperó parte de la información, pero no toda.

Template:
```
⚠️ Resultado parcial: <what_was_found>

No se encontró: <what_is_missing>

<next_step_suggestion>
```

## Restricciones
- Usa siempre el idioma del usuario (español por defecto en este stack).
- No expongas mensajes crudos de error, stack traces, tokens ni IDs internos en salidas para el usuario.
- No expongas rutas del sistema de archivos salvo que sean enlaces de descarga o referencias de artefactos útiles para el usuario.
- Mantén los mensajes concisos: una pantalla o menos.
- No repitas información ya visible en la conversación.

## Manejo de fallos
Si no puede determinarse el tipo de mensaje desde el resultado de la herramienta:
- Usa por defecto la plantilla **Error — Recuperable**
- Reporta el campo `message` del resultado como la sección `what_went_wrong`
