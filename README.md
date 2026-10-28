# 🗨️ Tunisia Camp — Forum & Real-Time Chat Platform

> A production-grade, full-stack social platform: a **threaded forum** (posts, comments, reactions, markdown, image uploads, content moderation) combined with a **real-time group chat** (STOMP over WebSockets, per-channel rooms, read receipts, paginated history).

<p align="center">
  <img alt="Spring Boot" src="https://img.shields.io/badge/Spring%20Boot-2.7.5-6DB33F?logo=springboot&style=for-the-badge">
  <img alt="Angular" src="https://img.shields.io/badge/Angular-14-DD0031?logo=angular&style=for-the-badge">
  <img alt="WebSocket" src="https://img.shields.io/badge/STOMP-WebSocket-1E4C8F?logo=socketdotio&style=for-the-badge">
  <img alt="MySQL" src="https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&style=for-the-badge">
  <img alt="CI" src="https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=githubactions&style=for-the-badge">
  <img alt="CD" src="https://img.shields.io/badge/Quality-Jenkins%20%2B%20SonarQube-4C9AC7?logo=jenkins&style=for-the-badge">
</p>

---

## 💡 The Idea in One Line

A single web application where a community can **discuss asynchronously** (forum articles with markdown, comments and like/dislike reactions) and **synchronously** (live chat rooms) — with built-in **content moderation** and an **automated quality pipeline**.

---

## ✨ Key Features & Technical Highlights

### Real-Time Chat Engine
- ⚡ **Bidirectional WebSocket communication** using STOMP (`spring-boot-starter-websocket`) — messages are broadcast to per-room channels (`/channel/chat/{channel}`) via `SimpMessagingTemplate`.
- 👀 **Read receipts**: a single native bulk `UPDATE` marks all of a user's unread messages in a channel as read — one statement, no N+1 round-trips.
- 📜 **Paginated message history** per channel through Spring Data `Pageable`, so conversations stay fast at any size.
- 🟢 **Live presence**: login/logout events are pushed over WebSocket to every connected client in real time.

### Forum / Social
- 📝 **Markdown authoring** with a live editor and a dedicated `MarkdownPipe` on the frontend.
- 👍👎 **Like/Dislike reactions** (typed enum) with aggregate rating points and a **"Most Liked" trending feed** endpoint.
- 💬 **Nested comments** per post, with cascade removal on post deletion.
- 🖼️ **Image uploads** (multipart) attached to posts.
- 🛡️ **Server-side content moderation**: prohibited words are sanitized at write-time (`Post` create/update), replacing matches with `*` — enforcement lives in the backend, never trust the client.

### Architecture Quality
- 🧩 **Fully modular, lazy-loaded Angular frontend** — 5 feature modules (`home`, `article`, `editor`, `profile`, `shared`, `core`) loaded on demand via the router with `PreloadAllModules`.
- 🏗️ **Clean layered backend**: `Controller → Service → Repository → Entity`, with DTOs + a dedicated converter decoupling the persistence model from the API contract.
- 🚑 **Domain-specific exception handling** (`PostException`, `ReactException`, …) mapped to correct HTTP status codes (`404`, `403`, `204`).
- 🔒 **Centralized CORS** via a servlet filter + annotation, and a **JWT-ready HTTP interceptor** on the frontend.
- 📖 **Auto-generated OpenAPI documentation** (springdoc) served with the app.

### DevOps & Quality
- 🔁 **Jenkins pipeline**: Maven build → **SonarQube static analysis** (quality gates), with staged Docker build/push and Nexus artifact publishing.
- 🤖 **GitHub Actions CI**: matrix build (Node 16 + LTS) on every push/PR, plus CodeQL security analysis, Dependabot and stale-issue automation.
- 🐳 **Dockerized tooling**: `docker-compose` for SonarQube + Nexus, `Dockerfile` + `nginx.conf` for the frontend production image.
- ✅ **Pre-commit hook** enforcing lint before every commit.

---

## 🏗️ Architecture at a Glance

```
┌─────────────────────────┐        ┌──────────────────────────────┐
│  Angular 14 SPA         │  HTTP  │  Spring Boot 2.7 REST API    │
│  ─ lazy feature modules │ ─────▶ │  ─ Controllers / Services    │
│  ─ core models+services │        │  ─ DTO layer + exceptions    │
│  ─ JWT interceptor      │        │  ─ Spring Data JPA           │
│                         │        │                              │
│  @stomp/native WebSocket│ ◀───── │  STOMP Broker (/channel/**)  │
└─────────────────────────┘  WS    └──────────────┬───────────────┘
        nginx (prod)                              │
        GitHub Actions CI/CD                      ▼
                                       ┌──────────────────┐
                                       │      MySQL       │
                                       │  (Hibernate ddl) │
                                       └──────────────────┘
```

📐 Full design rationale, data model and request lifecycles: **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**
📚 Complete endpoint reference (REST + WebSocket): **[docs/API.md](docs/API.md)**

---

## 🛠️ Tech Stack

| Layer            | Technology                                            | Why it matters here |
|------------------|-------------------------------------------------------|---------------------|
| Frontend         | Angular 14, TypeScript 4.6, RxJS 6                    | Modular, scalable SPA with lazy loading & DI |
| UI/UX            | SweetAlert2, marked (markdown)                        | polished UX, rich post formatting |
| Backend          | Spring Boot 2.7, Java 17                              | Robust REST + messaging in one runtime |
| Real-time        | Spring WebSocket + STOMP, simple in-memory broker     | Low-latency pub/sub chat rooms |
| Persistence      | Spring Data JPA / Hibernate, MySQL 8                  | Rapid, safe CRUD + pagination |
| API docs         | springdoc-openapi (Swagger UI)                        | Self-documenting API |
| Code quality     | Lombok, ModelMapper, ESLint, SonarQube                | Less boilerplate, measurable quality |
| CI/CD            | GitHub Actions, CodeQL, Jenkins, Dependabot            | Automated build, test, security scan |
| Containerization | Docker, docker-compose, nginx                          | Reproducible envs & prod serving |

---

## 🚀 Getting Started

### Prerequisites
- **Java 17** + Maven · **Node.js ≥ 16** · **MySQL** (or use the bundled compose setup)

### Backend
```bash
cd tunisia-camp-backend
# Configure DB in src/main/resources/application.properties if needed (default: root, no password)
./mvnw spring-boot:run
# → API at http://localhost:4201/tunisia-camp
# → STOMP endpoint at ws://localhost:4201/tunisia-camp/ichat
```

### Quality tooling (optional)
```bash
cd tunisia-camp-backend && docker-compose up -d   # SonarQube :9000, Nexus :8081
```

### Frontend
```bash
cd tunisia-camp-frontend
npm ci
npm start                                          # → http://localhost:4200
```

---

## 📁 Project Structure

```
PI_forum-chat/
├── tunisia-camp-backend/          # Spring Boot API + WebSocket broker
│   └── src/main/java/.../
│       ├── RestControllers/       # 7 REST/message controllers
│       ├── Services/              # Business logic (posts, chat, moderation…)
│       ├── DAO/                   # Entities, DTOs, Spring Data repositories
│       ├── config/                # WebSocket/STOMP configuration
│       ├── exception/             # Domain exceptions
│       └── security/              # CORS filter
├── tunisia-camp-frontend/         # Angular 14 SPA
│   └── src/app/
│       ├── core/                  # Models, API services, JWT interceptor
│       ├── shared/                # Reusable components (react buttons, list, layout)
│       ├── home | article | editor | profile/   # Lazy-loaded feature modules
│       └── .github/workflows/     # CI, CD, CodeQL, Dependabot
└── docs/                          # Architecture & API documentation
```

---

## 🎯 What This Project Demonstrates

- **End-to-end ownership** — from database schema and REST/messaging APIs to a modular SPA and the CI/CD that ships it.
- **Real-time systems** — hands-on with WebSocket protocols (STOMP), publish/subscribe channel design, and server push patterns (presence, chat).
- **API design discipline** — resource-oriented routes, DTO separation, proper HTTP semantics, OpenAPI docs.
- **Security & content-safety mindset** — server-side moderation, centralized CORS, interceptor-based auth plumbing.
- **Engineering culture** — static analysis gates, automated builds on PRs, dependency automation, lint-enforced commits.

---

*Built as a project at Tunisia Camp (Esprit School of Engineering).*
