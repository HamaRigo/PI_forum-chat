# ⚙️ Backend — Spring Boot Forum & Real-Time Chat API

Spring Boot 2.7 / Java 17 service exposing **two APIs in one runtime**: a REST API for the forum domain and a STOMP WebSocket broker for chat & presence.

## Capabilities
- **Forum**: posts (markdown, images), comments, LIKE/DISLIKE reactions, rating points, trending ("most liked") feed, paginated feeds with dynamic sort.
- **Chat**: persistent messages per channel, paginated history, bulk read receipts (single native `UPDATE`), broadcast fan-out via `SimpMessagingTemplate`.
- **Presence**: login/logout pushed to `/channel/login` / `/channel/logout`.
- **Moderation**: prohibited-word dictionary applied at write-time on posts & comments (`sanitizeText` → `****`).
- **Docs**: springdoc-openapi → `http://localhost:4201/tunisia-camp/swagger-ui.html`.

## Package layout
```
tn.esprit.tunisiacampbackend/
├── RestControllers/  ConnectionController · PostController · CommentController
│                     ReactController · ChatController (@MessageMapping)
│                     MessageController · ProhibitedWordController
├── Services/         PostService · CommentService · ReactService
│                     ProhibitedWordService · UserService(Impl)
├── DAO/              Entities (JPA) · DTOs + ToDtoConverter · Repositories (Spring Data)
├── config/           WebSocketConfig (STOMP broker /channel/**, app prefix /app)
├── exception/        domain-specific exceptions → mapped HTTP statuses
└── security/         CORSFilter
```

## Run
```bash
./mvnw spring-boot:run          # MySQL on localhost:3306 (tunisiaCamp auto-created)
docker-compose up -d            # optional: SonarQube :9000 + Nexus :8081 for the pipeline
```

## CI/CD
`Jenkinsfile`: `mvn clean package` → SonarQube static analysis (quality gate) → staged Docker build/push & Nexus publishing. Full endpoint contract: [../docs/API.md](../docs/API.md).
