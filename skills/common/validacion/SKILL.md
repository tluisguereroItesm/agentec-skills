---
name: common-validacion
description: "usa esta skill como paso reutilizable de validación cuando cualquier otra skill necesite verificar campos requeridos, formatos, rangos de valores o completitud de datos antes de invocar una herramienta. úsala para verificaciones previas sobre entradas del usuario, datos de formulario o parámetros de herramientas."
---

# common-validacion

Lógica reutilizable de validación. Aplícala antes de invocar cualquier herramienta que requiera entrada estructurada.

## Cuándo usar
- Antes de llamar cualquier herramienta con campos requeridos
- Cuando el usuario proporciona información parcial o ambigua
- Cuando se debe verificar formato o rango antes de continuar

## Reglas de validación

### Precondición de autenticación (Microsoft Graph)
Para cualquier skill/herramienta Microsoft Graph (por ejemplo: `graph_mail`, `graph_files`, `graph_files_write`, `graph_calendar`, `graph_teams`, `graph_users`, `graph_sharepoint_search`, `graph_approvals`, `graph_flows`, `graph_powerbi`):

1. Antes de ejecutar acciones de negocio, verifica si ya existe autenticación confirmada en la sesión activa.
2. Si no existe autenticación confirmada, detén la ejecución funcional e inicia el flujo:
   - `auth-login`
   - Mostrar `verification_uri` y `user_code`
   - Esperar confirmación del usuario (`listo`/`ya`/`hecho`)
   - `auth-poll`
3. Solo cuando `auth-poll` sea exitoso, continuar con la acción solicitada.
4. Si `auth-poll` devuelve `pending`, mantener la espera y no ejecutar la acción de negocio.
5. Si devuelve `expired`, reiniciar con `auth-login`.

### Campos requeridos
1. Identifica todos los campos requeridos para la acción actual.
2. Verifica que cada campo esté presente y no vacío.
3. Si falta algún campo requerido, detente y pídelo al usuario antes de continuar.
4. No asumas ni inventes valores para campos requeridos.

### Validación de formato
Aplica estas verificaciones cuando se conozca el tipo de campo:

| Tipo | Regla |
|------|------|
| Email | Debe contener `@` y un dominio |
| Fecha | Debe poder parsearse (preferido ISO 8601: YYYY-MM-DD) |
| Número | Debe ser entero o decimal válido; verificar rango si está definido |
| URL | Debe comenzar con `http://` o `https://` |
| ID / código | Debe coincidir con el patrón esperado (alfanumérico, longitud) |
| Selector | Debe ser un string CSS no vacío |

### Rango y longitud
- Valores numéricos: valida min/max si la skill invocadora lo define.
- Valores de texto: valida longitud máxima si está definida.
- Listas: valida conteo mínimo/máximo si está definido.

## Comportamiento esperado
1. Recibe la lista de campos y sus valores desde la skill invocadora.
2. Ejecuta primero la verificación de campos requeridos.
3. Ejecuta validación de formato para cada campo con tipo conocido.
4. Ejecuta validación de rango/longitud cuando existan límites definidos.
5. Devuelve uno de estos resultados:
   - **valid**: todas las verificaciones pasaron, continúa a invocación de herramienta.
   - **invalid**: lista cada campo fallido con un mensaje claro orientado al usuario.

## Formato de salida
Cuando sea inválido, reporta cada problema como bullet separado:
```
- Campo requerido faltante: <field_name>
- Formato incorrecto en <field_name>: se esperaba <expected_format>
- Valor fuera de rango en <field_name>: debe estar entre <min> y <max>
```

## Restricciones
- No modifiques valores de campos para corregir errores de validación.
- No continúes a invocación de herramienta si falta algún campo requerido.
- No expongas nombres internos de campos en mensajes al usuario; usa etiquetas legibles.

## Manejo de fallos
Si la validación no puede completarse (por ejemplo, no se proporciona esquema):
- Reporta el problema claramente
- No continúes a invocación de herramienta
