# 📚 API Reference

Base URL (dev): `http://localhost:4201/tunisia-camp`
Interactive docs (springdoc-openapi): `/tunisia-camp/swagger-ui.html` · OpenAPI JSON: `/tunisia-camp/v3/api-docs`

All request/response bodies are `application/json` unless noted. Responses marked 🛡️ run **server-side prohibited-word sanitization** before persisting.

---

## 🔌 WebSocket (STOMP)

| Item | Value |
|------|-------|
| Endpoint | `ws://localhost:4201/tunisia-camp/ichat` |
| Client sends to | `/app/messages` |
| Client subscribes to | `/channel/chat/{channel}` (room feed), `/channel/login`, `/channel/logout` (presence) |

### Send a chat message
`SEND /app/messages` — body: `{ "channel": "room1", "sender": "alice", "content": "hi" }`
Server stamps the timestamp, **persists the message**, then broadcasts it to `/channel/chat/room1`.

---

## 📰 Posts (Forum Articles) — `/articles`

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `GET` | `/articles` | All posts, newest first | `{ articles, articlesCount }` |
| `GET` | `/articles/liked?limit&userId` | **Trending feed** — most-liked posts, optionally filtered by author | sorted by like count, limited |
| `GET` | `/articles/paginated?page&limit&type` | Paginated feed | `type` = sort property (default `dateTimeOfPost`, desc) |
| `GET` | `/articles/paginatedByUser?page&limit&userId` | A user's posts, paginated | used by profile tabs |
| `GET` | `/articles/{id}` | Single post with like/dislike counts | `404` if missing |
| `POST` | `/articles` | Create post | 🛡️ sanitized |
| `PUT` | `/articles` | Update post | 🛡️ sanitized, `404` if missing |
| `DELETE` | `/articles/{id}` | Delete post (cascades comments + reactions) | `204` |
| `POST` | `/articles/upload` | Multipart image upload attached to a post | `multipart/form-data`, field: `file` |
| `PUT` | `/articles/{id}/rate?` | Rate post — body is button state (`1` = up, `0` = down) | adjusts `ratingPoints` |

**Post shape** (DTO): `id`, `title` (3–40 chars), `content` (markdown, ≤5000), `dateTimeOfPost`, `imageUrl`, `ratingPoints`, `likesCount`, `dislikesCount`, `author`.

---

## 💬 Comments — `/comments`

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `POST` | `/comments` | Add comment to a post | 🛡️ sanitized |
| `PUT` | `/comments` | Update comment | 🛡️ sanitized |
| `DELETE` | `/comments/{id}` | Delete comment | `204` |

Comments are exposed through the post detail payload (`Article.comments`).

---

## 👍 Reactions — `/reactions`

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/reactions` | Create reaction (`type`: `LIKE` \| `DISLIKE` enum) |
| `PUT` | `/reactions` | Toggle/update reaction |
| `DELETE` | `/reactions/{id}` | Remove reaction (`204`) |

One reaction row per user per post; like/dislike aggregates are computed on read and returned inside post DTOs.

---

## 🗨️ Chat History & Read Receipts — `/messages`

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `GET` | `/messages/{channelId}?page&size&sort` | Paginated message history for a room | Spring `Page<Message>` |
| `POST` | `/messages` | **Mark as read**: body `{ "channel", "username" }` | one native bulk `UPDATE` sets `readDate` on all unread messages of that sender in that channel |

---

## 👤 Users & Presence

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `POST` | `/login` | Register/connect user | broadcasts presence on `/channel/login`; `403` if username taken |
| `POST` | `/logout` | Disconnect user | broadcasts on `/channel/logout`, `204` |
| `GET` | `/listUsers` | All users | feed for the online list |
| `GET` | `/defaultUser` | Default/guest user DTO | onboarding convenience |
| `GET` | `/clearAlluser` | Wipe all users | dev/testing utility |

---

## 🛡️ Moderation — `/prohibited-words`

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `POST` | `/prohibited-words?words=a,b,c` | Add words to the moderation dictionary (comma-separated, batch) | applied immediately to subsequent post/comment writes |

Sanitization replaces each banned word with `*` of the same length (e.g. `damn` → `****`).

---

## Error semantics

| Status | When |
|--------|------|
| `200` | successful reads/writes |
| `204` | successful deletes / logout |
| `403` | username already in use on `/login` |
| `404` | post / comment / reaction not found (message body explains which) |
