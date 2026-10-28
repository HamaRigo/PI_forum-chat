# 🏗️ System Architecture

This document explains **how** and **why** Tunisia Camp is structured the way it is — the decisions a reviewer should understand before reading the code.

---

## 1. High-Level Design

Tunisia Camp is a **two-tier SPA application** with two complementary communication paradigms:

| Paradigm | Used for | Technology |
|----------|----------|------------|
| **Request/Response (HTTP/REST)** | Forum CRUD: posts, comments, reactions, auth, uploads | Spring MVC `@RestController` |
| **Publish/Subscribe (WebSocket/STOMP)** | Chat messages, read receipts fan-out, login/logout presence | `spring-boot-starter-websocket`, `SimpMessagingTemplate` |

Both paradigms live in **one Spring Boot runtime**, sharing the same JPA repositories — a chat message saved over WebSocket is immediately queryable over REST.

```
Browser (Angular SPA)
   │  ▲
   │  │ HTTP /api endpoints            STOMP frames over /ichat
   ▼  │
┌────────────────────────────────────────────────────────┐
│                   Spring Boot 2.7 (:4201)              │
│  /tunisia-camp                                         │
│                                                        │
│  RestControllers ──► Services ──► Spring Data Repos ──►│──► MySQL
│  ChatController  ──► (same repos)                      │
│       ▲                                                │
│  WebSocketConfig (STOMP broker: /channel/**,           │
│                    app prefix: /app/**)                │
└────────────────────────────────────────────────────────┘
```

---

## 2. Backend Layering

```
tn.esprit.tunisiacampbackend/
├── RestControllers/   # HTTP verbs, status codes, request sanitization  (no business logic)
├── Services/          # Business rules: pagination, trending, moderation, ratings
├── DAO/
│   ├── Entities/      # JPA entities (the persistence model)
│   ├── DTO/           # API contracts + ToDtoConverter (decoupling layer)
│   └── Repositories/  # Spring Data JPA interfaces, derived & native queries
├── config/            # WebSocket/STOMP broker configuration
├── exception/         # Domain-specific unchecked exceptions
└── security/          # CORS servlet filter
```

### Key decisions & rationale

- **DTO separation (`PostDto`, `CommentDto`, `ReactDto`, `UserDTO`)** — entities own JPA relationships (`@JsonIgnoreProperties`, lazy `@ManyToOne`); DTOs own what the client should see. A dedicated `ToDtoConverter` keeps the mapping in one place, so the persistence model can evolve without breaking the API contract.
- **Write-time content sanitization** — `PostController` and `CommentController` run `ProhibitedWordService.sanitizeText()` *before* persisting. Moderation happens exactly once, at the trust boundary, and the raw text never reaches the database. This is the "never trust the client" principle applied server-side.
- **Domain exceptions, not generic ones** — `PostException`, `CommentException`, `ReactException`, `UsernameAlreadyUsedException` let each controller translate *intent* into the correct HTTP status (`404` not-found, `403` duplicate-username) instead of a blanket `500`.
- **Derived + native queries side by side** — pagination and sorting via `PageRequest`/`Sort` (e.g. trending posts, per-user feeds), and a hand-tuned native `UPDATE` for bulk read receipts where JPQL would be needlessly verbose.
- **`@Transient` reaction aggregates** — `likesCount`/`dislikesCount` are computed on read (`ReactRepository.countByPostAndTypeEquals`) rather than stored, avoiding cache-invalidation bugs on every reaction toggle.

### Data model

```
User 1 ──* Post 1 ──* Comment
              │          
              └──* React (type: LIKE | DISLIKE — enum, type-safe)

Message (chat)   — channel, sender, content, timestamp, readDate   (channel = chat room key)
ProhibitedWord   — moderation dictionary, editable at runtime via POST /prohibited-words
```

Relationships use `CascadeType.REMOVE`, so deleting a post garbage-collects its comments and reactions automatically — referential integrity is enforced in the domain layer, not left to chance.

---

## 3. Real-Time Subsystem

`WebSocketConfig` registers:

- **STOMP endpoint**: `/ichat` (CORS-open for dev)
- **Broker prefix**: `/channel/**` — clients subscribe here to receive (e.g. `/channel/chat/{roomName}`)
- **App prefix**: `/app/**` — clients send here (e.g. `/app/messages`)

**Chat send lifecycle:**

```
Client ──STOMP SEND /app/messages──► ChatController.@MessageMapping
                                          │ stamp timestamp + persist (JPA)
                                          ▼
        Client ◄──SUBSCRIBE /channel/chat/{channel}── SimpMessagingTemplate.convertAndSend
```

Every message is **persisted before broadcast**, so history survives reconnects — the WebSocket layer is a delivery mechanism, MySQL is the source of truth.

**Presence:** `ConnectionController.login/logout` push the affected `User` to `/channel/login` and `/channel/logout`, letting every connected client update the online-user list without polling.

**Known scaling limit (deliberate trade-off):** the broker is the Spring *simple in-memory* broker, i.e. single-node. The natural next step — externalizing to RabbitMQ/STOMP relay with `setUserDestinationPrefix` — is a drop-in change thanks to the broker abstraction.

---

## 4. Frontend Architecture (Angular 14)

### Module topology — route-level code splitting

```
app-routing.module.ts (forRoot + PreloadAllModules)
 ├── home/     (lazy)  — feed, filters, pagination
 ├── article/  (lazy)  — post detail, comments, react buttons + resolvers
 ├── editor/   (lazy)  — create/edit with markdown preview + resolvers
 ├── profile/  (lazy)  — user's posts & liked posts tabs
 ├── core/     (eager singleton) — models, API services, HTTP interceptor
 └── shared/   (eager) — reusable UI: article list, react/favorite buttons, layout
```

- **`core/`** owns every domain model (`Article`, `Comment`, `React`, `User`) and one service per resource (`ArticlesService`, `CommentsService`, `ReactsService`, `UserService`) layered over a single `ApiService` that centralizes the base URL and `HttpParams` serialization — the backend URL appears exactly once in the app.
- **`HttpTokenInterceptor`** attaches credentials to every request in one place — auth is a cross-cutting concern handled by the framework, not per-call boilerplate.
- **Route resolvers** (`ArticleResolver`, `EditableArticleResolver`) guarantee a component's data exists *before* instantiation: no loading-state flicker, no orphan templates, and 404s redirect at the router level.
- **`MarkdownPipe`** wraps `marked` for safe rendered post bodies; **SweetAlert2** standardizes confirmations.
- **`ReactButtonComponent`** encapsulates the optimistic-like UI state machine (idle → liked/disliked → toggled), talking to `ReactsService` through RxJS subscriptions — the flagship interactive component.

### Unidirectional data flow
`Component → Service → ApiService → HttpClient → RxJS → Component` — components never touch `HttpClient` directly, keeping them testable and the network layer swappable.

---

## 5. Cross-Cutting Concerns

| Concern        | Where it's handled |
|----------------|--------------------|
| CORS           | Backend: `CORSFilter` (servlet) + `@CrossOrigin` on controllers |
| Validation     | Bean Validation on entities (`@NotNull`, `@Size`, `@Min`) — fails at the persistence boundary |
| Error mapping  | try/catch domain exceptions → typed HTTP responses per endpoint |
| API discovery  | springdoc-openapi → Swagger UI at `/tunisia-camp/swagger-ui.html` |
| Environment    | `application.properties` (DB, port 4201, context path); Angular `environments/*` |
| Bootstrapping  | `ddl-auto=update` schema evolution for dev velocity |

---

## 6. DevOps / Delivery Architecture

```
push / PR ──► GitHub Actions CI ──► npm ci + matrix build (Node 16, LTS)
main        ──► CodeQL security scan · Dependabot updates · stale automation
             ──► Jenkins: mvn package ──► SonarQube quality gate
             ──► (staged) Docker image build/push ──► Nexus artifact repo
prod           nginx container serves the static Angular bundle
```

Quality tooling (SonarQube, Nexus) runs locally via `docker-compose`, so the exact pipeline services are reproducible on any developer machine.

---

## 7. Honest Trade-offs (what I'd do next)

Documenting weaknesses is part of owning an architecture:

1. **Auth is session-trust style** — no JWT issuance/validation on the backend yet; the frontend interceptor is already prepared for it. Next step: `spring-boot-starter-security` + JWT filter.
2. **In-memory STOMP broker** — single-node by design; scale-out path is a RabbitMQ relay, configuration-only.
3. **Image upload writes to the filesystem** — swap for object storage (S3/local MinIO) + return the URL instead of mutating the latest post.
4. **`ddl-auto=update`** — fast for development; production would use Flyway/Liquibase migrations.
5. **Moderation list is applied with regex `replaceAll`** — could be upgraded to an Aho-Corasick matcher and extended to chat messages (currently posts/comments only).
