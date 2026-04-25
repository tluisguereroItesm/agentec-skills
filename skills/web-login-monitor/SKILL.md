---
name: web-login-monitor
description: usa esta skill cuando el usuario necesite validar o ejecutar un flujo de inicio de sesión en un portal web, especialmente para monitoreo, pruebas de humo, flujos de demo o verificación de autenticación. úsala cuando un login en navegador deba ejecutarse con una herramienta aprobada y se requiera evidencia del resultado.
---

# web-login-monitor

Usa la herramienta `web_login_playwright` para escenarios de solo login.

## Entrada requerida
Envía estos campos a la herramienta:
- `url`
- `username`
- `password`
- `usernameSelector`
- `passwordSelector`
- `submitSelector`

## Entrada opcional
Usa estos campos cuando estén disponibles:
- `successIndicator`
- `headless`
- `timeoutMs`

## Comportamiento esperado
1. Valida que la solicitud sea solo para login.
2. Invoca `web_login_playwright`.
3. Devuelve un resultado claro:
   - éxito o fallo
   - mensaje
   - ruta de captura
   - ruta de resultado

## Restricciones
- No continúes navegación posterior al login en esta versión.
- No inventes selectores.
- Si faltan selectores, solicítalos o usa los valores acordados del flujo configurado.
- Devuelve siempre evidencia si la herramienta la generó.

## Manejo de fallos
Si la herramienta falla:
- reporta el mensaje de error exacto
- indica qué paso falló, si está disponible
- devuelve la ruta de la captura si se generó