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
