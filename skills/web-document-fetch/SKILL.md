---
name: web-document-fetch
description: "usa esta skill cuando el usuario necesite descargar un documento (PDF, DOCX, XLSX u otro archivo) desde una página web o portal, con o sin un paso previo de login. úsala cuando el archivo sea accesible vía una URL que pueda requerir navegación en navegador, autenticación o interacción antes de que el archivo esté disponible."
---

# web-document-fetch

Usa la herramienta `web_fetch_download` para descargar documentos desde páginas web.

## Cuándo usar
- El usuario pide descargar, obtener o recuperar un documento desde un sitio web
- El documento está detrás de un login (usa `configProfile` para aportar credenciales)
- La URL del documento se conoce o se puede navegar directamente
- Se requiere evidencia de la descarga (captura + archivo)

## Entrada requerida
Envía estos campos a la herramienta:
- `url` — URL de la página donde el documento se encuentra o se puede descargar directamente

## Entrada opcional
Usa estos campos cuando estén disponibles:
- `configProfile` — nombre de perfil de login si se requiere autenticación antes de descargar
- `configFile` — ruta a archivo de perfiles personalizado
- `downloadSelector` — selector CSS del enlace o botón de descarga a clicar
- `waitForDownload` — si debe esperar el evento de descarga del navegador (`true` por defecto)
- `headless` — ejecutar el navegador sin interfaz (por defecto: `true`)
- `timeoutMs` — timeout en milisegundos (por defecto: `30000`)

## Comportamiento esperado
1. Si se proporciona `configProfile`, ejecuta primero el login usando las credenciales configuradas.
2. Navega a `url`.
3. Si se proporciona `downloadSelector`, haz clic en ese elemento y espera el evento de descarga.
4. Si no se proporciona selector, intenta descargar la URL de forma directa.
5. Guarda el archivo en `artifacts/`.
6. Toma una captura como evidencia.
7. Devuelve:
   - `success` / `message`
   - `filePath` — ruta al archivo descargado
   - `mimeType` — tipo MIME detectado
   - `screenshotPath` — ruta de la captura de evidencia

## Restricciones
- No navegues más allá de la página objetivo de descarga.
- No inventes selectores de descarga; pregúntale al usuario si se desconocen.
- No expongas credenciales en la salida.
- Devuelve siempre evidencia (captura), incluso en caso de fallo.

## Encadenamiento
Después de una descarga exitosa, sugiere usar la skill `doc-summarize` si el usuario quiere leer o entender el contenido:
```
✅ Documento descargado: <filename>
¿Quieres que lo lea y te haga un resumen?
```

## Manejo de fallos
Si la herramienta falla:
- Reporta el mensaje exacto del error
- Reporta en qué paso falló (login / navegación / descarga)
- Devuelve la ruta de la captura si se generó
- Usa `common-notificaciones` para formatear el error para el usuario
