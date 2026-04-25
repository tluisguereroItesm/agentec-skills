---
name: doc-summarize
description: "usa esta skill cuando el usuario necesite leer, entender, extraer información de, o resumir un archivo de documento (PDF, DOCX, XLSX, TXT, MD). úsala después de una descarga, cuando haya una ruta de archivo disponible, o cuando el usuario pegue o refiera un documento local. no la uses para contenido web en vivo: usa web-document-fetch para eso."
---

# doc-summarize

Orquesta la extracción de texto y el resumen de documentos usando la herramienta `doc_reader` y el modelo.

## Cuándo usar
- El usuario tiene un archivo local y quiere entender su contenido
- Se acaba de completar una descarga y el usuario quiere un resumen
- El usuario pide "leer", "explicar", "extraer" o "resumir" un documento
- El documento es PDF, DOCX, XLSX, TXT o Markdown

## Entrada requerida
Uno de los siguientes:
- `filePath` — ruta absoluta o relativa al archivo local
- `content` — texto ya extraído (omite la invocación de la herramienta y pasa directo al resumen)

## Entrada opcional
- `maxChars` — máximo de caracteres a extraer (por defecto: `8000`). Usa valores menores en archivos grandes.
- `summaryType` — tipo de resumen solicitado:
  - `general` (por defecto) — puntos principales y conclusión
  - `key-data` — enfoque en números, fechas, nombres, montos
  - `tasks` — extracción de acciones o pendientes
  - `legal` — cláusulas, obligaciones, fechas, partes

## Comportamiento esperado
1. Si se proporciona `filePath` y no `content`, invoca `doc_reader` con `filePath` y `maxChars`.
2. Si la herramienta devuelve `success: false`, reporta el error y detente.
3. Usa el `content` extraído del resultado de la herramienta (o el `content` proporcionado) como entrada del modelo.
4. Genera un resumen según `summaryType`:

### estructura de resumen general
```
## Resumen del documento

**Título / Tipo:** <detectado o inferido>
**Páginas:** <pageCount>  ·  **Palabras:** <wordCount>

### Puntos principales
- <point_1>
- <point_2>
- <point_3>

### Conclusión
<una o dos oraciones>
```

### estructura de resumen de datos clave
```
## Datos clave

| Campo | Valor |
|-------|-------|
| <label> | <value> |
```

### estructura de resumen de tareas
```
## Tareas y pendientes identificados

- [ ] <task_1>
- [ ] <task_2>
```

### estructura de resumen legal
```
## Resumen legal

**Partes:** <parties>
**Fecha de firma:** <date>
**Vigencia:** <validity>
**Obligaciones principales:** <obligations>
**Cláusulas importantes:** <clauses>
```

## Restricciones
- No inventes contenido que no esté en el documento.
- Si la extracción de texto está incompleta (recortada por `maxChars`), indícalo claramente.
- No expongas rutas internas de archivos en el resumen, salvo que el usuario lo pida.
- Si el archivo no es legible (protegido con contraseña, corrupto o formato no soportado), repórtalo y detente.

## Manejo de fallos
Si `doc_reader` falla:
- Reporta el `errorType` si está disponible
- Sugiere al usuario verificar la ruta y el formato del archivo
- Usa `common-notificaciones` para formatear el error
