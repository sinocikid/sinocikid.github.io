---
title: "Technical Note: Resolving ERR_MODULE_NOT_FOUND in clawdhub (Node.js v25)"
date: 2026-01-28
tags: ["Node.js", "Troubleshooting", "DevTools", "Clawdhub"]
---

# Technical Note: Resolving `ERR_MODULE_NOT_FOUND` in `clawdhub` (Node.js v25)

## TL;DR
If `npx clawdhub` crashes on Node.js v25 with a `Cannot find package 'undici'` error, the quickest fix is to manually inject the dependency:
1. Navigate to the installed package directory: `cd .../lib/node_modules/clawdhub`
2. Run `npm install undici`

---

## Issue Overview
When attempting to initialize `clawdhub` via `npx` on a macOS environment running the latest Node.js release (v25.4.0), the process terminates immediately with a module resolution error. The runtime fails to locate the `undici` package, which is required by the tool's HTTP module.

## Environment
* **OS:** macOS (Apple Silicon)
* **Runtime:** Node.js v25.4.0
* **Tool:** `clawdhub` (invoked via `npx`)

## Error Stack Trace
```bash
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'undici' imported from /Users/[user]/.nvm/versions/node/v25.4.0/lib/node_modules/clawdhub/dist/http.js
    at Object.getPackageJSONURL (node:internal/modules/package_json_reader:316:9)
    at packageResolve (node:internal/modules/esm/resolve:768:81)
    ...
    code: 'ERR_MODULE_NOT_FOUND'