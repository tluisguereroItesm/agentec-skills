---
name: web-login-monitor-py
description: "usa esta skill cuando el usuario necesite validar o ejecutar un flujo de inicio de sesión en un portal web usando el backend de Python, especialmente para depuración local, pruebas de humo, validación en shadow o pruebas de paridad entre backends. úsala cuando el login en navegador deba ejecutarse mediante la herramienta de Python aprobada y se requiera evidencia."
---

# web-login-monitor-py

Usa la herramienta `web_login_playwright_py` para escenarios de solo login cuando se prefiera el backend de Python.

## Entrada requerida
Envía estos campos a la herramienta:
- `username`
- `password`

Y además uno de estos:
- `configProfile`

o bien los campos explícitos del portal:
- `url`
- `usernameSelector`
- `passwordSelector`
- `submitSelector`

## Entrada opcional
Usa estos campos cuando estén disponibles:
- `configFile`
- `successIndicator`
- `headless`
- `timeoutMs`

## Comportamiento esperado
1. Valida que la solicitud sea solo para login.
2. Resuelve la configuración del portal desde `configProfile` cuando se proporcione.
3. Invoca `web_login_playwright_py`.
4. Devuelve un resultado claro:
   - éxito o fallo
   - mensaje
   - ruta de captura
   - ruta de resultado
   - backend utilizado

## Restricciones
- No continúes navegación posterior al login en esta versión.
- No inventes selectores.
- Si existe un perfil para el portal, prefírelo sobre selectores ad-hoc.
- Devuelve siempre evidencia si la herramienta la generó.

## Manejo de fallos
Si la herramienta falla:
- reporta el mensaje de error exacto
- indica si falló el perfil o el paso del navegador
- devuelve la ruta de la captura si se generó
