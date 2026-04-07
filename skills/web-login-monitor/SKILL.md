---
name: web-login-monitor
description: use this skill when a web portal login must be executed or validated through an approved local browser automation tool and evidence of the result is required.
---

# web-login-monitor

Use the approved local tool `web-login-playwright` for login-only flows.

## Scope
This version only supports:
- open login page
- fill username
- fill password
- click submit
- verify success indicator
- return evidence

## Required fields
- url
- username
- password
- usernameSelector
- passwordSelector
- submitSelector

## Optional fields
- successIndicator
- headless
- timeoutMs

## Constraints
- Do not continue with post-login navigation.
- Do not invent selectors.
- Always preserve evidence paths returned by the tool.
