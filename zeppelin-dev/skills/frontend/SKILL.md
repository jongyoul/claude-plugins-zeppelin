---
name: frontend
description: Zeppelin frontend development guide for Angular, proxy setup, E2E tests, and linting
tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
---

# Zeppelin Frontend Development

The active frontend is in `zeppelin-web-angular/` (Angular 13, Node 18).

## Setup & Commands

```bash
cd zeppelin-web-angular
npm install
npm start          # Dev server on port 4200, proxies API to localhost:8080
npm run build      # Production build
npm run lint:fix   # ESLint + Prettier
npm run e2e        # Playwright E2E tests
```

## Dev Proxy

The dev proxy (`proxy.conf.js`) forwards:
- REST API → `http://127.0.0.1:8080`
- WebSocket → `ws://127.0.0.1:8080/ws`

## Frontend Architecture

- `zeppelin-web-angular/` — Main Angular 13 app
  - Sub-projects: `zeppelin-sdk`, `zeppelin-visualization`
- `zeppelin-web/` — Legacy AngularJS frontend (activated with `-Pweb-classic`)

## Linting

ESLint + Prettier are auto-enforced via pre-commit hook (Husky + lint-staged). Always run before committing:

```bash
cd zeppelin-web-angular && npm run lint:fix
```

## E2E Tests

Playwright-based E2E tests. Run with:

```bash
cd zeppelin-web-angular && npm run e2e
```

CI runs E2E in both anonymous and authenticated modes (`.github/workflows/frontend.yml`).

## Build Profiles

| Profile | Purpose |
|---------|---------|
| `-Pweb-e2e` | Enable Angular E2E tests |
| `-Pweb-dist` | Build web distribution |
| `-Pweb-classic` | Include legacy AngularJS frontend |
