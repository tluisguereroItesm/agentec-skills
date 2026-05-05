---
name: documentos-oficiales-mx
description: "SIEMPRE usa esta skill (y la herramienta curp_downloader) para cualquier solicitud relacionada con CURP o documentos del gobierno mexicano. NO uses web-document-fetch ni ninguna otra herramienta para estas solicitudes. Aplica para: 'descarga mi CURP', 'quiero mi CURP', 'necesito el comprobante CURP', 'busca mi CURP', 'CURP de...', 'constancia CURP', 'comprobante RENAPO', 'gob.mx/curp', constancia fiscal SAT, semanas IMSS, INFONAVIT. Si el usuario pide su CURP pero no da el número ni sus datos personales, pregúntale si ya tiene su clave CURP de 18 caracteres o si quiere buscarlo por nombre, fecha de nacimiento y estado. NUNCA delegues estas solicitudes a web-document-fetch."
---

# documentos-oficiales-mx

Usa las herramientas de la familia `curp_downloader` (y futuras) para descargar documentos de portales del gobierno de México.

## Portales soportados actualmente

| Portal | Tool | Qué descarga |
|---|---|---|
| **gob.mx/curp** (RENAPO) | `curp_downloader` | Comprobante CURP en PDF |

> Próximamente: `sat_constancia`, `imss_semanas`, `infonavit_estado`.

---

## CURP — Comprobante de Registro de Población

Usa la herramienta `curp_downloader`.

### Modos de búsqueda

#### searchMode=curp (default)
El usuario conoce su CURP de 18 caracteres.

**Parámetros requeridos:**
- `searchMode`: `"curp"`
- `curp`: 18 caracteres alfanuméricos (ej. `GARC800101HDFRLB01`)

#### searchMode=datos
El usuario no tiene el CURP a la mano; busca por datos personales.

**Parámetros requeridos:**
- `searchMode`: `"datos"`
- `nombre`: nombre(s) de pila, sin apellidos
- `primerApellido`: apellido paterno
- `diaNacimiento`: día con cero inicial (`"01"`–`"31"`)
- `mesNacimiento`: mes con cero inicial (`"01"`–`"12"`)
- `anioNacimiento`: año de 4 dígitos (ej. `"1990"`)
- `sexo`: `"H"` = Hombre, `"M"` = Mujer, `"X"` = No binario
- `claveEntidad`: estado de nacimiento — acepta nombre completo (`"Jalisco"`) o clave de 2 letras (`"JC"`)

**Parámetros opcionales:**
- `segundoApellido`: apellido materno (omitir si no aplica)

### Entrega del PDF (`delivery`)

| Valor | Qué hace | Campos extra requeridos |
|---|---|---|
| `"artifact"` | Guarda el PDF localmente (default) | — |
| `"email"` | Envía el PDF como adjunto de correo | `to` (dirección o lista separada por coma) |
| `"onedrive"` | Sube a OneDrive del usuario autenticado | `remoteFolder` (opcional, default: `"CURP"`) |

> Para `delivery=email` o `delivery=onedrive` se requiere autenticación Microsoft Graph.
> Ver flujo de autenticación más abajo.

### Parámetros de comportamiento

- `timeoutMs` (integer): timeout en ms. Default `90000`. Aumentar a `120000` si el sitio está lento — el challenge WAF de gob.mx tarda ~22s antes de cargar el formulario.
- `headless` (boolean): default `true`. No cambiar salvo depuración.

---

## Flujo cuando el usuario NO proporciona CURP ni datos

Si el usuario solo dice "quiero mi CURP", "descarga mi CURP" o similar sin aportar información:

```
Agente: "¡Claro! Para descargar tu comprobante CURP tengo dos opciones:

1. **Si ya tienes tu clave CURP** (18 caracteres, ej. GARC800101HDFRLB01)
   → Dímela y lo descargo de inmediato.

2. **Si no recuerdas tu CURP** → puedo buscarlo por tus datos:
   • Nombre(s) de pila
   • Primer apellido (paterno)
   • Segundo apellido (materno, si aplica)
   • Fecha de nacimiento (día, mes, año)
   • Sexo registrado al nacer (H/M/X)
   • Estado de nacimiento

¿Cuál prefieres?"
```

---

## Flujos de conversación

### Descargar CURP por clave (artifact)

```
Usuario: "descarga mi CURP, es GARC800101HDFRLB01"

→ Llama curp_downloader con:
   { searchMode: "curp", curp: "GARC800101HDFRLB01", delivery: "artifact" }

→ Éxito:
   Agente: "✅ Comprobante CURP descargado:
            Archivo: CURP_GARC800101HDFRLB01_<ts>.pdf (xx KB)
            Guardado en: /app/artifacts/
            ¿Quieres que te lo envíe por correo o lo suba a tu OneDrive?"

→ Error CURP_NOT_FOUND:
   Agente: "No encontré el registro para CURP GARC800101HDFRLB01 en el padrón de RENAPO.
            Verifica que los 18 caracteres sean correctos o prueba con el modo búsqueda por datos personales."
```

### Buscar CURP por datos personales

```
Usuario: "no sé mi CURP, búscalo con mis datos"

→ Agente pide los campos obligatorios si faltan:
   "Para buscarlo por datos personales necesito:
    • Nombre(s) (sin apellidos)
    • Primer apellido (paterno)
    • Segundo apellido (materno, si aplica)
    • Fecha de nacimiento (día, mes, año)
    • Sexo registrado (H/M/X)
    • Estado de nacimiento"

→ El usuario proporciona los datos → Llama curp_downloader con searchMode="datos"

→ Éxito:
   Agente: "✅ CURP encontrado: GARC800101HDFRLB01
            Comprobante descargado: CURP_GARC800101HDFRLB01_<ts>.pdf
            ¿Te lo envío por correo?"

→ Error CURP_NOT_FOUND:
   Agente: "No encontré un registro con esos datos en RENAPO.
            Revisa: ¿el nombre está exactamente como aparece en el acta de nacimiento?
            ¿El estado de nacimiento es el correcto?"
```

### Enviar CURP por correo

```
Usuario: "mándame el CURP al correo usuario@empresa.com"

→ Si ya tienes el CURP o los datos del usuario:
   Llama curp_downloader con delivery="email", to="usuario@empresa.com"

→ Si requiere autenticación Graph (errorType: AUTH_REQUIRED):
   Ver sección de autenticación más abajo.

→ Éxito:
   Agente: "✅ Correo enviado a usuario@empresa.com con el comprobante CURP adjunto."
```

### Subir CURP a OneDrive

```
Usuario: "guarda el CURP en mi OneDrive en la carpeta Documentos/CURP"

→ Llama curp_downloader con:
   { delivery: "onedrive", remoteFolder: "Documentos/CURP", ... }

→ Éxito:
   Agente: "✅ PDF subido a OneDrive: Documentos/CURP/CURP_<...>.pdf
            Enlace: {webUrl}"
```

---

## Autenticación Microsoft Graph (delivery=email / onedrive)

Solo se requiere si el usuario pide `delivery=email` o `delivery=onedrive`.

1. Llama `curp_downloader` con `action: "auth-login"` (sin parámetros de búsqueda).
2. Muestra al usuario:
   - "Para continuar, abre: **{verification_uri}**"
   - "Ingresa este código: **{user_code}**"
   - "Cuando termines, escribe: **listo**"
3. Espera confirmación del usuario (`"listo"`, `"ya"`, `"hecho"`).
4. Llama `curp_downloader` con `action: "auth-poll"`.
5. Si `status = ok` → reintenta la descarga con delivery.
6. Si `status = pending` → pide confirmación de nuevo.
7. Si `status = expired` → genera nuevo `auth-login`.

**Reglas:**
- No uses `web_login_playwright` para autenticación Graph.
- No pidas credenciales directamente al usuario.
- Si `delivery="artifact"` → no requiere auth, descarga directa.

---

## Manejo de errores

| errorType | Causa | Qué decirle al usuario |
|---|---|---|
| `CURP_NOT_FOUND` | CURP o datos no registrados en RENAPO | Verificar caracteres / datos personales |
| `CURP_SITE_ERROR` | gob.mx no cargó o el sitio cambió | Reintentar con `timeoutMs: 120000` |
| `CURP_DOWNLOAD_ERROR` | CURP encontrado pero PDF no descargó | Reintentar; sitio puede estar inestable |
| `MISSING_ARG` | Falta un parámetro requerido | Pedir el campo faltante al usuario |
| `AUTH_REQUIRED` | Token expirado o primera sesión | Iniciar flujo auth-login |
| `GRAPH_ERROR` | Error de API Graph al enviar/subir | Verificar conectividad o reintentar |

---

## Restricciones

- Solo descarga información que el propio usuario está solicitando sobre sí mismo o sobre personas con su consentimiento.
- No almacenes ni expongas datos personales en la respuesta final más allá del CURP y nombre del archivo.
- Siempre confirma con el usuario antes de enviar por correo a una dirección de terceros.
- Si el usuario pide datos de otra persona sin indicar relación o consentimiento, solicita aclaración antes de proceder.
