# dockerized-typescript-api

A minimal TypeScript REST API, containerized with Docker and gated by a GitHub Actions pipeline that enforces every check — including a clean Docker build — on every commit.

[![CI](https://github.com/Thomas-J-Barreras-Consulting/dockerized-typescript-api/actions/workflows/ci.yml/badge.svg)](https://github.com/Thomas-J-Barreras-Consulting/dockerized-typescript-api/actions/workflows/ci.yml)
![Node](https://img.shields.io/badge/node-20-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/typescript-5-3178C6?logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/docker-multi--stage-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Overview

The service itself is deliberately small — two HTTP endpoints over Express — so the surrounding infrastructure is the subject, not a distraction. The project exists to show a working end-to-end setup: typed source, in-process tests against the Express app, a multi-stage container image, and a CI workflow that will not let the repository drift into a broken state.

---

## What this demonstrates

- **Testability by construction.** [src/app.ts](src/app.ts) builds the Express application; [src/server.ts](src/server.ts) owns the `listen()` call. Tests in [test/health.test.ts](test/health.test.ts) exercise the app in-process via supertest — no port binding, no test-only branches, no mocks.

- **Multi-stage Docker image.** [Dockerfile](Dockerfile) compiles TypeScript inside a full `node:20` image, then copies only `dist/` and production dependencies into `node:20-slim`. Dev dependencies and source never reach the runtime image.

- **CI as a contract.** [.github/workflows/ci.yml](.github/workflows/ci.yml) runs lint → test → `tsc` build → `docker build`. The Docker build is the final step on purpose: if the Dockerfile drifts out of sync with a source change, CI fails on the pull request rather than at deploy time.

- **Single sources of truth.** Service name and version come from [package.json](package.json); Node version is pinned in [.nvmrc](.nvmrc). No duplicated constants, no drift.

---

## API

| Method | Path      | Purpose                                     |
|--------|-----------|---------------------------------------------|
| GET    | `/`       | Service metadata (name, version, env)       |
| GET    | `/health` | Liveness check with ISO-8601 timestamp      |

Response shapes are pinned by [test/health.test.ts](test/health.test.ts).

---

## Stack

| Layer     | Choice                                         |
|-----------|------------------------------------------------|
| Runtime   | Node.js 20                                     |
| Language  | TypeScript 5                                   |
| HTTP      | Express                                        |
| Tests     | Jest + supertest                               |
| Lint      | ESLint 9 (flat config) + typescript-eslint 8   |
| Container | Docker (multi-stage) + Docker Compose          |
| CI        | GitHub Actions                                 |

---

## Project layout

```
src/
  app.ts              — Express app factory (no listen call, testable)
  server.ts           — Entry point; reads PORT, calls app.listen()
  routes/
    health.ts         — GET / and GET /health handlers
test/
  health.test.ts      — supertest integration tests
Dockerfile            — Multi-stage build (builder → slim runtime)
docker-compose.yml    — Local orchestration on port 3000
.github/workflows/
  ci.yml              — lint → test → build → docker build
```

---

## Running it

**With Node directly** (fastest feedback loop):

```bash
npm install
npm run dev       # ts-node; no build step
npm test
npm run lint
```

**With Docker:**

```bash
docker compose up --build    # http://localhost:3000
```

Compose is intentionally thin — the image itself is the interesting artifact.

---

## CI pipeline

Every push to `main` and every pull request targeting it runs:

1. Checkout
2. Set up Node (version from `.nvmrc`, npm cache enabled)
3. `npm ci`
4. `npm run lint`
5. `npm test`
6. `npm run build`
7. `docker build`

Step 7 guards the container surface the same way steps 4–6 guard the code surface: the Dockerfile must build cleanly from a fresh checkout, every time.

---

## License

MIT
