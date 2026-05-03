---
name: web-document-fetch
description: "usa esta skill cuando el usuario necesite descargar un documento (PDF, DOCX, XLSX u otro archivo) desde una página web o portal mediante múltiples pasos en navegador (login, navegación y click de descarga). también úsala para extraer el ID y formato de un video de YouTube (sin descargar video). NUNCA uses esta skill para solicitudes de CURP, documentos del gobierno mexicano (gob.mx/curp, SAT, IMSS, INFONAVIT) — usa documentos-oficiales-mx en su lugar."
---

# web-document-fetch

Usa la herramienta `web_fetch_download` para flujos web multistep.

## Cuándo usar
- El usuario pide descargar, obtener o recuperar un documento desde un sitio web
- El documento está detrás de un login o navegación por varias pantallas
- La URL del documento se conoce o se puede navegar directamente
- Se requiere evidencia de la descarga (captura + archivo)
- El usuario necesita extraer `videoId` y formato (`watch`, `shorts`, `live`, etc.) de un enlace YouTube

## Entrada requerida
Envía estos campos a la herramienta:
- `url` — URL de la página donde el documento se encuentra o se puede descargar directamente
- `action` — `download-document` (default) o `extract-youtube-id`

## Entrada opcional
Usa estos campos cuando estén disponibles:
- `steps` — lista de pasos secuenciales para login/navegación/descarga
   - tipos soportados: `goto`, `fill`, `click`, `waitForSelector`, `waitForTimeout`, `downloadClick`, `extractAttribute`, `extractText`
- `downloadSelector` — selector CSS del enlace o botón de descarga a clicar
- `waitForDownload` — si debe esperar el evento de descarga del navegador (`true` por defecto)
- `headless` — ejecutar el navegador sin interfaz (por defecto: `true`)
- `timeoutMs` — timeout en milisegundos (por defecto: `30000`)

## Comportamiento esperado
1. Navega a `url`.
2. Si se proporcionan `steps`, ejecútalos en orden.
3. Para `action: download-document`:
   - prioriza `downloadClick` dentro de `steps`
   - si no existe, usa `downloadSelector`
   - si no existe, intenta descarga directa con `waitForDownload=true`
4. Para `action: extract-youtube-id`:
   - extrae `videoId` de la URL actual/final o de valores extraídos en `steps`
   - detecta y devuelve `youtube.format` (`watch`, `shorts`, `live`, `embed`, `youtu.be`, `unknown`)
   - incluye `youtube.inputFormat` (formato del enlace enviado) y `youtube.resolvedFormat` (formato tras redirecciones)
   - devuelve también `youtube.kind` (`video`, `short`, `live`, `unknown`) y URL canónica
5. Toma captura de evidencia.
6. Devuelve:
   - `success` / `message`
   - `filePath` — ruta al archivo descargado
   - `youtube.videoId` — cuando aplique
   - `youtube.format` / `youtube.inputFormat` / `youtube.resolvedFormat` / `youtube.kind` — cuando aplique
   - `extracted` — campos extraídos durante los pasos
   - `screenshotPath` — ruta de la captura de evidencia

## Restricciones
- No ejecutes acciones no solicitadas por el usuario.
- No inventes selectores de descarga; pregúntale al usuario si se desconocen.
- No expongas credenciales en la salida.
- Devuelve siempre evidencia (captura), incluso en caso de fallo.
- Para YouTube, limita el uso a extracción de ID/metadata; no prometas descargas si no hay autorización y base legal.

## Encadenamiento
Después de una descarga exitosa, sugiere usar la skill `doc-summarize` si el usuario quiere leer o entender el contenido:
```
✅ Documento descargado: <filename>
¿Quieres que lo lea y te haga un resumen?
```

## Manejo de fallos
Si la herramienta falla:
- Reporta el mensaje exacto del error
- Reporta en qué paso falló (`steps[i]`, navegación o descarga)
- Devuelve la ruta de la captura si se generó
- Usa `common-notificaciones` para formatear el error para el usuario
