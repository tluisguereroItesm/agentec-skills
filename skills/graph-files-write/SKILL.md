---
name: graph-files-write
description: "usa esta skill cuando el usuario quiera subir un archivo a OneDrive, crear una carpeta, mover o renombrar un archivo, copiar un documento, eliminar un archivo o compartir un documento con un enlace en Microsoft 365."
---

# graph-files-write

Usa la herramienta `graph_files_write` para todas las operaciones de escritura en OneDrive y SharePoint.

> **IMPORTANTE**: Para leer archivos usa `graph_files`. Para escribir/subir/eliminar/compartir usa `graph_files_write`.
> Para autenticación, usa `graph_files_write` con `action: "auth-login"`. NUNCA `web_login_playwright`.

## Preflight obligatorio de autenticación (primer uso por sesión)

Antes de ejecutar cualquier acción que **no** sea `auth-login` o `auth-poll`:

1. Si en la sesión actual no existe confirmación de autenticación para `graph_files_write`, inicia SIEMPRE `auth-login` primero.
2. Muestra al usuario exactamente:
  - URL: `{verification_uri}`
  - Código: `{user_code}`
  - Mensaje: "Abre esa URL, ingresa el código y avísame con **listo**".
3. Espera confirmación explícita del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Ejecuta `auth-poll`.
5. Solo si `status = ok`, continúa con la solicitud original (`upload`, `create_folder`, `rename`, `move`, `copy`, `delete`, `share`).
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

- `upload` — sube un archivo local a OneDrive (requiere `localPath`, `remotePath`)
- `create_folder` — crea una carpeta (requiere `name`; opcional `parent`)
- `rename` — renombra un archivo o carpeta (requiere `id`, `name`)
- `move` — mueve un item a otra carpeta (requiere `id`, `destinationId`)
- `copy` — copia un item (requiere `id`, `destinationId`)
- `delete` — elimina un archivo o carpeta (requiere `id`)
- `share` — genera enlace compartido (requiere `id`; opcional `linkType`, `scope`)
- `auth-login` / `auth-poll` — autenticación device code

---

## Flujos de conversación

### Subir un archivo

```
Usuario: "sube el reporte del mes a OneDrive"

→ Si la ruta destino no está especificada:
  Agente: "¿En qué carpeta de OneDrive lo subo? Por ejemplo: 'Documentos/Reportes/Abril 2025'"

→ Usuario especifica la ruta
→ Agente confirma antes de subir:
  Agente: "Voy a subir:
           Archivo: reporte-abril.pdf
           Destino: Documentos/Reportes/Abril 2025
           ¿Continuamos?"

→ Usuario confirma → llama upload
  Agente: "Archivo subido correctamente. [Ver en OneDrive](webUrl)"
```

### Compartir un archivo

```
Usuario: "compárteme el contrato con un enlace"

→ Si no tienes el id: sugiere usar graph_files para encontrar el archivo primero
  Agente: "Para compartir el archivo necesito encontrarlo primero. ¿Me das más detalles
           como el nombre o ubicación? También puedo buscarlo con graph_files."

→ Una vez con el id: verifica el scope antes de compartir
  Agente: "¿El enlace es para personas dentro de la empresa o también externos?"

→ Usuario: "interno"
→ Llama share con scope="organization"
  Agente: "Enlace generado (solo para la organización):
           [Enlace de acceso](webUrl)"

→ Si usuario dice "externos" o "público":
  Agente: "Voy a generar un enlace público. Cualquier persona con el enlace podrá
           acceder al archivo. ¿Confirmas?"
```

### Eliminar un archivo

```
Usuario: "elimina el borrador viejo del contrato"

→ Si no tienes el id: sugiere buscar con graph_files primero
→ Una vez localizado, SIEMPRE pide confirmación:
  Agente: "Encontré este archivo:
           'Contrato Borrador v1.docx' — OneDrive / Documentos / Contratos
           ¿Confirmas que quieres eliminarlo permanentemente?
           Esta acción no se puede deshacer desde aquí."

→ Usuario confirma → llama delete
  Agente: "Archivo eliminado. (Puede estar en la papelera de reciclaje de OneDrive.)"
```

### Renombrar o mover

```
Usuario: "renombra el archivo a 'Contrato Final 2025'"

→ Si no tienes el id: sugiere buscar con graph_files primero
→ Confirma antes de ejecutar:
  Agente: "Voy a renombrar 'Contrato Borrador v1.docx' → 'Contrato Final 2025.docx'.
           ¿Confirmas?"

→ Usuario confirma → llama rename
  Agente: "Archivo renombrado correctamente."
```

---

## Recuperación de errores

### Archivo no encontrado (404 en delete/rename/move)

```
delete/rename/move returns: 404

Agente: "No encontré el archivo con ese identificador. Es posible que ya haya sido
         eliminado o que la ubicación haya cambiado.
         ¿Quieres que lo busque de nuevo?"
```

### Carpeta destino no encontrada (en move/copy)

```
move/copy returns: error carpeta destino

Agente: "No encontré la carpeta de destino. Verifica que exista.
         ¿Quieres que primero cree la carpeta?"
```

### Error al subir (archivo demasiado grande o sin permisos)

```
upload returns: error

Agente: "No pude subir el archivo. Posibles causas:
         - El archivo es demasiado grande para el destino
         - No tienes permisos de escritura en esa carpeta
         - La ruta de destino no existe
         Ruta intentada: [remotePath]. ¿Quieres intentar con otra ubicación?"
```

---

## Reglas de clarificación

Pregunta ANTES de ejecutar cuando:

| Situación | Pregunta al usuario |
|-----------|---------------------|
| No hay ruta destino para upload | "¿En qué carpeta de OneDrive lo subo?" |
| Scope de share no especificado | "¿Es para la organización o para personas externas?" |
| Operación destructiva (delete) | Siempre muestra el archivo y pide confirmación |
| move sin destinationId | "¿A qué carpeta lo muevo?" |

---

## Reglas absolutas

- **Nunca elimines un archivo sin mostrar su nombre y recibir confirmación explícita del usuario.**
- Si el scope es `anonymous`, advierte que cualquiera con el enlace puede acceder.
- Si el usuario necesita primero el `id` del archivo, indícale que use `graph_files` para encontrarlo.

---

## Flujo de autenticación (device code)

Cuando la herramienta devuelva `errorType: AUTH_ERROR`:

**Paso 1:** Llama `graph_files_write` con `{"action": "auth-login"}`.
> "Para autenticarte en OneDrive, abre **{verification_uri}** en tu navegador
> e ingresa el código **{user_code}**. Avísame cuando hayas completado el login."

**Paso 2:** Cuando el usuario confirme, llama `graph_files_write` con `{"action": "auth-poll"}`.
- `status: "ok"` → retoma la solicitud original.
- `status: "pending"` → "Parece que aún no completaste el login. ¿Ya ingresaste el código?"
- `status: "expired"` → "El código expiró. Voy a generar uno nuevo." → repite desde Paso 1.
