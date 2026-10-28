# 🚀 Tunisia Camp — Forum & Real-Time Chat Platform

> **Full-stack capstone project**: an Angular 14 SPA on top of a Spring Boot API that combines a moderated forum (markdown posts, comments, reactions, image uploads) with a real-time STOMP/WebSocket chat (rooms, presence, read receipts) — delivered through GitHub Actions CI + a Jenkins/SonarQube quality pipeline.

![Angular](https://img.shields.io/badge/Angular-14-DD0031?logo=angular&style=flat)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.7-6DB33F?logo=springboot&style=flat)
![WebSocket](https://img.shields.io/badge/STOMP-over-WebSocket-blue)
![MySQL](https://img.shields.io/badge/MySQL-8-4479A1?logo=mysql&style=flat)
![CI](https://img.shields.io/badge/CI-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)

---

## What it does

- 📰 **Forum** — create/edit posts with **Markdown** + live preview, attach images, comment, and react with 👍/👎; feeds support **pagination, dynamic sorting and a trending "most-liked" view**.
- 💬 **Real-time chat** — WebSocket rooms with instant broadcast, **online presence** pushed on login/logout, **paginated history** and **read receipts**.
- 🛡️ **Content moderation** — banned words are sanitized **server-side at write time**, with a runtime-editable dictionary.
- 👤 **Profiles** — per-user paginated post lists and liked-post lists.

## Highlights that matter to reviewers

| Area | What was done |
|---|---|
| **Frontend architecture** | Fully modular Angular app: 4 lazy-loaded feature modules + `core` (single source of models/services) + `shared` (reusable UI); `PreloadAllModules`; route **resolvers** fetch data before component init; **HTTP interceptor** centralizes auth tokens; unidirectional `Component → Service → HttpClient → RxJS` flow. |
| **Backend architecture** | Clean `Controller → Service → Repository` layering; **DTO layer** decouples the JPA model from the API contract; **domain-specific exceptions** mapped to correct HTTP statuses; Bean Validation on entities; auto-generated **OpenAPI/Swagger** docs. |
| **Real-time** | STOMP broker with per-room topics (`/channel/chat/{room}`), messages **persisted before broadcast**, single bulk native `UPDATE` for read receipts (no N+1). |
| **DevOps** | GitHub Actions CI (Node matrix build on every PR), **CodeQL** security scanning, Dependabot, Jenkins → **SonarQube quality gate**, Docker Compose for tooling, nginx-served production frontend; pre-commit lint hook. |
| **Trade-off awareness** | Known limits and upgrade paths documented honestly (in-memory broker → RabbitMQ relay, filesystem upload → object storage, `ddl-auto` → Flyway). |

---

## Repository layout

```
PI_forum-chat/
├── tunisia-camp-backend/     # Spring Boot REST + WebSocket API (Java 17, MySQL)
├── tunisia-camp-frontend/    # Angular 14 SPA (TypeScript, RxJS)
└── docs/
    ├── ARCHITECTURE.md       # design decisions, data model, request lifecycles
    └── API.md                # full REST + STOMP endpoint reference
```

## Quick start

```bash
# Backend (requires Java 17 + MySQL on localhost:3306)
cd tunisia-camp-backend && ./mvnw spring-boot:run     # → http://localhost:4201/tunisia-camp

# Frontend (requires Node ≥ 16)
cd tunisia-camp-frontend && npm ci && npm start       # → http://localhost:4200
```

## Try the API

Swagger UI: `http://localhost:4201/tunisia-camp/swagger-ui.html` — or see the annotated endpoint tables in [docs/API.md](docs/API.md), including the STOMP chat protocol (`/ichat` endpoint, `/app/messages` sends, `/channel/chat/{room}` subscriptions).

---

*Full system design, rationale and scaling notes: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).*
