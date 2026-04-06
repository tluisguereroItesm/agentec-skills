---
name: web-login-monitor
description: use this skill when the user needs to validate or execute a login flow in a web portal, especially for monitoring, smoke testing, demo flows, or authentication verification. use it when a browser-based login must be executed through an approved tool and evidence of the result is required.
---

# web-login-monitor

Use the tool `web-login-playwright` for login-only scenarios.

## Required input
Send these fields to the tool:
- `url`
- `username`
- `password`
- `usernameSelector`
- `passwordSelector`
- `submitSelector`

## Optional input
Use these when available:
- `successIndicator`
- `headless`
- `timeoutMs`

## Expected behavior
1. Validate that the request is only for login.
2. Invoke `web-login-playwright`.
3. Return a clear result:
   - success or failure
   - message
   - screenshot path
   - result path

## Constraints
- Do not continue to post-login navigation in this version.
- Do not invent selectors.
- If selectors are missing, request or use the agreed values from the configured flow.
- Always return evidence if the tool produced it.

## Failure handling
If the tool fails:
- report the exact failure message
- indicate which step failed if available
- return the screenshot path if generated