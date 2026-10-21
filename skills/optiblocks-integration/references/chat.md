# Real-Time Chat Microservice — integration reference

One-to-one and group chat as a service. The backend calls a small REST API to send messages and read history. Recipients get each message as JSON over one WebSocket per connection, and messages are persisted to MongoDB in batches. A conversation is identified only by the set of its participants' user IDs, so there are no rooms to create, and the only access rule is "the caller is a participant".

- Image: `slincsilver/real-time-chat-microservice:1.0.1` (https://hub.docker.com/r/slincsilver/real-time-chat-microservice), also `latest`, `linux/amd64` + `linux/arm64`, built `FROM scratch`, runs as UID 65532, listens on port 8080
- Repo: https://github.com/SantiagoLopezDeharo/real-time-chat-microservice

## When to use / when not

Use it when:
- the product needs direct messages or small/medium group chats between users your backend already identifies by a string ID;
- the backend can mint HMAC JWTs, or can proxy requests and mint them per request;
- delivery plus paginated history is enough: no edits, deletes, read receipts, typing indicators, attachments or search;
- one instance is enough for the expected number of concurrent connections.

Do not use it when:
- conversations need mutable membership, names, admins or moderation: membership is the participant set itself, so adding someone creates a different conversation with empty history;
- you need several replicas for HA or scale: connection state and rate limiting are in process memory, with no cross-instance fan-out;
- you need guaranteed delivery to offline users or replay on reconnect: the socket only carries live messages, and clients must reload history;
- browsers must connect directly without a backend hop: the WebSocket handshake only accepts an `Authorization` header, which browsers cannot set;
- you only need to push server events to browsers (use the real-time UI update service) or need roles/claims-based authorization (only `id` is read).

## Run it

Requirements: MongoDB reachable at startup (tested with `mongo:7.0`, because MongoDB 8.x/9.x refuse to start on Linux kernels 6.19+, SERVER-121912) and a `JWT_SECRET` shared with your backend. The process exits with status 1 if `JWT_SECRET` is empty, any numeric setting is invalid, or MongoDB does not answer a ping within 10 s. At startup it creates the index `{participants: 1, created_at: -1, _id: -1}` on the collection.

```bash
docker network create chat
docker run -d --name chat-mongodb --network chat mongo:7.0
docker run -d --name chat-api --network chat -p 127.0.0.1:8080:8080 \
  --read-only --cap-drop=ALL --security-opt=no-new-privileges \
  -e JWT_SECRET="$CHAT_JWT_SECRET" \
  -e MONGO_URI=mongodb://chat-mongodb:27017 \
  slincsilver/real-time-chat-microservice:1.0.1
```

Compose service block to paste into an existing `docker-compose.yml` (tested with `docker compose up -d --wait`). It keeps the chat API on the internal network, so only your backend reaches it. Add `ports: ["127.0.0.1:8080:8080"]` if you need it from the host.

```yaml
services:
  chat-mongodb:
    image: mongo:7.0
    restart: unless-stopped
    volumes:
      - chat_mongodb_data:/data/db
      - chat_mongodb_config:/data/configdb
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 10s
      timeout: 5s
      retries: 6
      start_period: 20s

  chat-api:
    image: slincsilver/real-time-chat-microservice:1.0.1
    restart: unless-stopped
    environment:
      JWT_SECRET: ${CHAT_JWT_SECRET:?set CHAT_JWT_SECRET}
      MONGO_URI: mongodb://chat-mongodb:27017
      MONGO_DB: chatdb
    depends_on:
      chat-mongodb:
        condition: service_healthy
    stop_grace_period: 20s
    expose:
      - "8080"
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 512m
    cpus: 2
    pids_limit: 256

volumes:
  chat_mongodb_data:
  chat_mongodb_config:
```

Your backend then uses `http://chat-api:8080` and `ws://chat-api:8080/ws`.

Health checks: the image has no healthcheck and no shell, `curl` or `wget` (it is `scratch`), so a Docker `HEALTHCHECK`/compose `healthcheck` with `CMD` cannot work. Use an orchestrator HTTP probe on `GET /health` port 8080 (Kubernetes `httpGet` liveness/readiness, load balancer target check), or check from a dependent container. `/health` only reports that the process is serving; it does not check MongoDB.

Required env: `JWT_SECRET`. Practically required: `MONGO_URI`, because the default `mongodb://localhost:27017` points at the container itself.

Logs: plain-text `log/slog` lines on stderr, e.g. `INFO server listening port=8080`, `WARN failed to save messages ...`, `ERROR dropping messages after retries ...`.

## Configuration

All settings are environment variables. A `.env` file in the working directory is also loaded if present (not relevant for the image). An empty value means the default. Invalid values abort startup and list every error at once, e.g. `RETRY_ATTEMPTS must be >= 1, got 0`.

| Name | Default | Validation | Purpose |
|------|---------|------------|---------|
| `JWT_SECRET` | none | required, non-empty | HMAC key used to verify every JWT |
| `MONGO_URI` | `mongodb://localhost:27017` | none; must be reachable within 10 s at startup | MongoDB connection string; `mongodb+srv://` and TLS work (the CA bundle is in the image) |
| `MONGO_DB` | `chatdb` | none | Database name |
| `MONGO_COLLECTION` | `messages` | none | Collection name |
| `PORT` | `8080` | none | HTTP listen port (`:PORT`, all interfaces) |
| `RETRY_ATTEMPTS` | `5` | integer ≥ 1 | Total insert attempts per persistence batch |
| `RATE_LIMIT_RPS` | `5` | number > 0, ≤ 1e12 | Token refill rate per user ID, shared by `POST /api/messages` and `GET /api/messages/get` |
| `RATE_LIMIT_BURST` | `10` | integer ≥ 1 | Token bucket size per user ID |
| `MAX_BODY_BYTES` | `65536` | integer ≥ 1 | Max request body; larger gives `413` |
| `PPROF_ADDR` | unset (off) | none | Address for a separate `net/http/pprof` listener, e.g. `127.0.0.1:6060`; never expose it |

Fixed, not configurable: 1000 participants per message / users per connections query, page ≤ 10000, size ≤ 100, 32 KiB request headers, HTTP timeouts (read header 5 s, read 10 s, write 15 s, idle 120 s), 15 s shutdown budget, persistence queue 1024, 4 workers, batch ≤ 256, retry backoff 100 ms × attempt, 5 s per insert attempt, and the WebSocket limits in the API contract.

## Auth model

- Your backend signs every token with the shared `JWT_SECRET`. The chat service only verifies tokens and never issues them. No key IDs, issuer or audience are checked.
- Header format: exactly `Authorization: Bearer <jwt>`. The prefix is case-sensitive (`bearer` gives 401), and the token must not contain spaces. Query-string tokens and cookies are not supported.
- Accepted algorithms: `HS256`, `HS384`, `HS512`. `none`, `RS*`, `ES*` and `PS*` give 401.
- Required claim: `id`, a JSON string that is not blank after trimming. It is the user ID used as sender, as membership identity and as the rate-limit key. A numeric `id` (`{"id": 42}`) gives 401, and `sub` is ignored, so a token with only `sub` gives 401. The ID is used verbatim (not trimmed), and participant IDs are trimmed, so never mint IDs with surrounding whitespace.
- `exp` and `nbf` are validated when present, with no leeway, but are not required. `iat` is not validated. Always set a short `exp`.
- Tokens are checked once per HTTP request and once at the WebSocket handshake. An open WebSocket stays open after its token expires.
- Routes without auth: `GET /health` and `POST /api/connections`.

Minting (TypeScript, `jsonwebtoken` 9; tested):

```ts
// chat-token.ts
import jwt from 'jsonwebtoken';

const CHAT_JWT_SECRET = process.env.CHAT_JWT_SECRET ?? '';

export function mintChatToken(userId: string, ttlSeconds = 300): string {
  if (!CHAT_JWT_SECRET) throw new Error('CHAT_JWT_SECRET is not set');
  if (userId.trim() === '') throw new Error('userId must not be blank');
  return jwt.sign({ id: userId }, CHAT_JWT_SECRET, { algorithm: 'HS256', expiresIn: ttlSeconds });
}
```

Stringify numeric primary keys (`String(user.id)`) and use the same string everywhere: in tokens, `participants`, `users` and when comparing the `sender` of received messages.

## API contract

General:
- JSON responses have `Content-Type: application/json` and a trailing newline. Object keys of map responses are sorted alphabetically (e.g. `{"id":...,"status":...}`).
- Error responses are `text/plain; charset=utf-8` with a short message plus newline. Internal details are never included.
- Request bodies must be exactly one JSON value no larger than `MAX_BODY_BYTES`. Trailing data such as a second document gives 400. Unknown fields are ignored, and `Content-Type` is not checked.
- Routing: a known path with the wrong method gives `405`, an unknown path `404`, and request headers over 32 KiB `431`. `HEAD /health` works.

Errors per route, in check order:

| Status | Body | Cause |
|--------|------|-------|
| 401 | `missing authorization header` | No `Authorization` header |
| 401 | `invalid authorization header format` | Not `Bearer <token>` |
| 401 | `invalid token` | Bad signature, wrong algorithm, expired, `nbf` in the future, missing/blank/non-string `id` |
| 429 | `Too Many Requests` | Per-user bucket empty. No `Retry-After` header |
| 413 | `request body too large` | Body over `MAX_BODY_BYTES` |
| 400 | `bad request` | Malformed JSON, wrong types, trailing data |
| 400 | `participants array is required` | Missing or empty `participants` |
| 400 | `too many participants` | More than 1000 entries (before de-duplication) |
| 400 | `participant IDs must not be empty` | An entry blank after trimming |
| 403 | `forbidden: user must be part of participants` | Token `id` not in the normalized participants |
| 400 | `content is required` | `content` missing or whitespace only |
| 503 | `service unavailable` + `Retry-After: 1` | Persistence queue full, or shutting down. Nothing was stored or delivered |
| 500 | `internal error` | Unexpected failure, e.g. MongoDB down during a history read |

### `GET /health`

No auth, not rate limited. `200`:

```json
{"status":"ok","time":"2026-10-21T15:56:55Z"}
```

### `POST /api/messages`

Headers: `Authorization: Bearer <jwt>`, `Content-Type: application/json`. Rate limited.

```json
{"participants": ["alice", "bob"], "content": "Hello Bob!"}
```

- `participants: string[]` is required, holds 1 to 1000 entries and must include the caller's `id`. Entries are trimmed, de-duplicated and sorted, so any order or duplication names the same conversation. `["alice"]` alone is a valid note to self.
- `content: string` must not be blank. It is stored and delivered byte-for-byte (no trimming or sanitizing); the only size limit is the body limit.
- The sender is always the token's `id`. A `sender` field in the body is ignored.

`202 Accepted` (the message is queued for persistence and already handed to recipients' connections):

```json
{"id":"6ad8e0d73f4c364eb2ceec5f","status":"message queued"}
```

`id` is a 24-hex-char MongoDB ObjectID. It is the same `id` recipients receive and history returns, so use it to de-duplicate. The response does not include `created_at`; the canonical copy arrives via history (or the WebSocket, for recipients).

Errors: 401, 429, 413, 400, 403, 503, 500 as in the table above.

### `GET /api/messages/get`

Headers: `Authorization: Bearer <jwt>`. Rate limited, sharing the bucket with sends.

| Param | Default | Rules |
|-------|---------|-------|
| `participants` | required | Comma-separated IDs, normalized like the send body. Missing or empty gives 400 `participants query parameter is required`. An empty entry (`alice,,bob`, `alice,`) gives 400 `participant IDs must not be empty` |
| `page` | `0` | Integer ≥ 0. `> 10000` gives 400 `page out of range`. Non-numeric, negative or overflowing values silently become `0` |
| `size` | `50` | Values above 100 are clamped to `100`. Non-numeric or `< 1` silently becomes `50` |

URL-encode IDs that contain reserved characters (`URLSearchParams` does). IDs containing commas cannot be expressed here, so do not allow commas in user IDs.

`200`: a JSON array, newest first (`created_at` desc, then `id` desc), `[]` when empty or past the end:

```json
[{"id":"6ad8e1143f4c364eb2ceec60","sender":"alice","content":"hello","created_at":"2026-10-21T15:58:12.482Z","participants":["alice","bob"]}]
```

`created_at` is RFC 3339 UTC with millisecond precision; trailing zeros are dropped, so the fractional part may have fewer than 3 digits or be absent. Pagination is `skip = page × size`; there is no total count and no cursor. There is no endpoint to list a user's conversations: track conversations in your own database.

Errors: 401, 429, 400, 403, 500.

### `POST /api/connections`

No auth, not rate limited. Body: `{"users": ["alice", "bob"]}`. `users` must be a non-empty array of at most 1000 strings, otherwise 400 (`users array is required` / `too many users` / `bad request`). Body limit as usual (413).

`200`: open WebSocket connections per requested user on this instance, with duplicates collapsed:

```json
{"alice":1,"bob":0}
```

It exposes presence to anyone who can reach the port, so keep the service on an internal network.

### `GET /ws` (WebSocket)

- Handshake: `GET /ws` with `Upgrade: websocket` and `Authorization: Bearer <jwt>`. Auth failures are plain HTTP `401` before the upgrade; a non-upgrade GET gives `400`. Not rate limited. Any `Origin` is accepted; the header-only auth is the CSRF defense. No subprotocols, no query parameters, no permessage-deflate.
- Browsers cannot set `Authorization` on `new WebSocket(...)`, so a browser can never connect directly. Two supported patterns:
  1. **BFF relay (recommended)**: the browser opens a WebSocket to your backend, authenticated by your normal session. The backend checks `Origin`, mints a short-lived chat token and opens the upstream socket with the header, then forwards server frames down. See the recipe.
  2. **Server-side or native clients** (Node, mobile, desktop) connect directly with the header.
- Frames received: one text frame per message, the same JSON object as a history entry. There are no other frame types, envelopes, acks, presence or typing events, and nothing is replayed on connect.

```json
{"id":"6ad8e1143f4c364eb2ceec60","sender":"alice","content":"hello via relay","created_at":"2026-10-21T15:58:12.482Z","participants":["alice","bob"]}
```

- A connection receives every conversation its user belongs to, and every connection (tab or device) of each recipient gets a copy. The sender's own connections never get the sender's message.
- Client → server: data frames are read and discarded, never relayed. Frames over 512 bytes close the socket with `1009`, and protocol violations (e.g. unmasked frames) with `1002`. Send nothing.
- Keepalive: the server pings every 54 s and drops the connection if no pong arrives within 60 s of the last one. Standard clients (`ws`, browsers) answer automatically. The write deadline is 10 s per frame.
- Backpressure: each connection may have 4 MiB of undelivered payload queued. Exceeding it closes the connection with `1008 slow consumer`.
- Close codes from the server:

| Code | Reason text | When | Client action |
|------|-------------|------|---------------|
| 1000 | empty | Client closed, read error/timeout, or write failure | Reconnect if unexpected |
| 1001 | `server shutting down` | `SIGTERM`/`SIGINT`, or an upgrade attempted during shutdown | Reconnect with backoff, reload history |
| 1008 | `slow consumer` | Outbound queue exceeded 4 MiB | Reconnect, reload history |
| 1009 | empty | Client sent a frame > 512 bytes | Fix the client; do not send |
| 1002 | protocol error description | Malformed client frames | Fix the client |

## Backend integration recipe

The code below was run under Node 26 type stripping (`node file.ts`, with `"type": "module"`) against `1.0.1` and type-checked with `tsc --strict`. It needs Node 18+ (global `fetch`, `AbortSignal.timeout`) plus the `jsonwebtoken` and `ws` packages (`npm i jsonwebtoken ws`, `npm i -D @types/jsonwebtoken @types/ws`). In a compiled TS/NestJS project, change the `./chat-token.ts` imports to your project's convention (`./chat-token` or `./chat-token.js`).

Env for the backend: `CHAT_JWT_SECRET` (same value as the service's `JWT_SECRET`), `CHAT_HTTP_URL` (e.g. `http://chat-api:8080`), `CHAT_WS_URL` (e.g. `ws://chat-api:8080/ws`) and `CHAT_ALLOWED_ORIGINS` (comma-separated browser origins, e.g. `https://app.example.com`).

### REST client

It retries only `429` and `503`: on those, the service guarantees nothing was queued, so re-sending cannot duplicate. It does not retry timeouts or network errors on `POST /api/messages`, because the message may already be queued. Surface those errors to the caller, who can de-duplicate by `id` later.

```ts
// chat-client.ts
import { mintChatToken } from './chat-token.ts';

export interface ChatMessage {
  id: string;
  sender: string;
  content: string;
  created_at: string;
  participants: string[];
}

export class ChatApiError extends Error {
  status: number;
  body: string;
  constructor(status: number, body: string) {
    super(`chat service responded ${status}: ${body}`);
    this.status = status;
    this.body = body;
  }
}

const RETRYABLE = new Set([429, 503]);
const MAX_ATTEMPTS = 3;
const TIMEOUT_MS = 5000;

export class ChatClient {
  baseUrl: string;
  constructor(baseUrl: string) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
  }

  sendMessage(senderId: string, participants: string[], content: string) {
    return this.request<{ id: string; status: string }>('POST', '/api/messages', senderId, { participants, content });
  }

  getHistory(userId: string, participants: string[], page = 0, size = 50) {
    const query = new URLSearchParams({ participants: participants.join(','), page: String(page), size: String(size) });
    return this.request<ChatMessage[]>('GET', `/api/messages/get?${query}`, userId);
  }

  connectionCounts(users: string[]) {
    return this.request<Record<string, number>>('POST', '/api/connections', undefined, { users });
  }

  health() {
    return this.request<{ status: string; time: string }>('GET', '/health');
  }

  private async request<T>(method: string, path: string, userId?: string, body?: unknown): Promise<T> {
    for (let attempt = 1; ; attempt++) {
      const headers: Record<string, string> = {};
      if (userId !== undefined) headers.Authorization = `Bearer ${mintChatToken(userId)}`;
      if (body !== undefined) headers['Content-Type'] = 'application/json';
      const res = await fetch(this.baseUrl + path, {
        method,
        headers,
        body: body === undefined ? undefined : JSON.stringify(body),
        signal: AbortSignal.timeout(TIMEOUT_MS),
      });
      if (res.ok) return (await res.json()) as T;

      const text = (await res.text()).trim();
      if (!RETRYABLE.has(res.status) || attempt >= MAX_ATTEMPTS) throw new ChatApiError(res.status, text);
      const retryAfterSeconds = Number(res.headers.get('Retry-After')) || attempt;
      await new Promise((resolve) => setTimeout(resolve, retryAfterSeconds * 1000));
    }
  }
}
```

Usage: `const chat = new ChatClient(process.env.CHAT_HTTP_URL!)`, then `await chat.sendMessage(String(user.id), [String(user.id), String(peer.id)], text)`. Map `ChatApiError.status` to your API: 400 becomes 400, 403 becomes 403/404, 413 becomes 413, persistent 429/503 become 429/503, and 401 means a `CHAT_JWT_SECRET` mismatch, so treat it as a 500 and alert. Always pass the authenticated user's ID as `senderId`/`userId`, never an ID from the request body.

A 14-request burst through this client against the default limits (5/s, burst 10) completed with all 14 successful in about 1 s: requests 11 to 14 got 429 and succeeded on retry.

### WebSocket relay for browsers

The browser connects to your backend (e.g. `wss://app.example.com/chat/ws`) with its session cookie. The relay checks `Origin` (mandatory, because cookie auth reintroduces cross-site WebSocket hijacking), authenticates the session, opens one upstream socket per browser socket with a freshly minted token, and forwards server frames down. It ignores browser frames (`maxPayload: 1024` drops abusive clients), mirrors close codes, closes slow browsers at 4 MiB buffered, and pings browsers every 30 s.

```ts
// chat-relay.ts
import type { IncomingMessage } from 'node:http';
import type { Duplex } from 'node:stream';
import WebSocket, { WebSocketServer } from 'ws';
import { mintChatToken } from './chat-token.ts';

const CHAT_WS_URL = process.env.CHAT_WS_URL ?? 'ws://chat-api:8080/ws';
const ALLOWED_ORIGINS = new Set((process.env.CHAT_ALLOWED_ORIGINS ?? '').split(',').filter(Boolean));
const MAX_BUFFERED_BYTES = 4 << 20;
const HEARTBEAT_MS = 30_000;

const browsers = new WebSocketServer({ noServer: true, maxPayload: 1024 });

export function handleChatUpgrade(
  req: IncomingMessage,
  socket: Duplex,
  head: Buffer,
  authenticate: (req: IncomingMessage) => Promise<string | null>,
): void {
  if (!ALLOWED_ORIGINS.has(req.headers.origin ?? '')) return reject(socket, '403 Forbidden');
  authenticate(req).then(
    (userId) => {
      if (!userId) return reject(socket, '401 Unauthorized');
      browsers.handleUpgrade(req, socket, head, (browser) => relay(browser, userId));
    },
    () => reject(socket, '500 Internal Server Error'),
  );
}

function reject(socket: Duplex, status: string): void {
  socket.end(`HTTP/1.1 ${status}\r\nConnection: close\r\nContent-Length: 0\r\n\r\n`);
}

function relay(browser: WebSocket, userId: string): void {
  const upstream = new WebSocket(CHAT_WS_URL, { headers: { Authorization: `Bearer ${mintChatToken(userId)}` } });
  let alive = true;
  const heartbeat = setInterval(() => {
    if (!alive) return browser.terminate();
    alive = false;
    browser.ping();
  }, HEARTBEAT_MS);

  browser.on('pong', () => { alive = true; });
  upstream.on('message', (data, isBinary) => {
    if (browser.bufferedAmount > MAX_BUFFERED_BYTES) return browser.close(1008, 'slow consumer');
    browser.send(data, { binary: isBinary });
  });
  upstream.on('error', () => {});
  upstream.on('close', (code, reason) => {
    clearInterval(heartbeat);
    const forwardable = code === 1000 || code === 1001 || code === 1008;
    browser.close(forwardable ? code : 1011, forwardable ? reason : undefined);
  });
  browser.on('error', () => {});
  browser.on('close', () => {
    clearInterval(heartbeat);
    upstream.terminate();
  });
}
```

Wire it to the Node HTTP server. With plain Node, Express or Fastify, use the `http.Server`; with NestJS, `app.getHttpServer()` after `app.init()`/before `listen`. If you also use a Nest WebSocket gateway, give the chat relay its own path and leave other upgrade paths alone:

```ts
server.on('upgrade', (req, socket, head) => {
  if (new URL(req.url ?? '/', 'http://localhost').pathname !== '/chat/ws') return; // leave other paths to their handlers
  handleChatUpgrade(req, socket, head, async (r) => {
    const session = await yourSessionStore.fromCookieHeader(r.headers.cookie); // your auth
    return session ? String(session.userId) : null;
  });
});
```

If no other upgrade handler exists, call `socket.destroy()` for unknown paths instead of `return`. Browser side: `new WebSocket('wss://app.example.com/chat/ws')`, then `onmessage = (e) => JSON.parse(e.data)`; route by `participants`, de-duplicate by `id`, and reconnect with backoff plus a history reload on any close.

Verified behaviour of this relay against the service: a wrong Origin got 403, no session got 401, the user was delivered the message with alice's own tab receiving nothing, and stopping the container closed the browser socket with `1001 server shutting down`.

Alternative: a transparent proxy (`http-proxy` with a `proxyReqWs` handler that sets `Authorization`, or nginx `proxy_set_header Authorization` after an `auth_request`) also works. It forwards browser frames upstream, though: the service ignores them, but frames over 512 bytes then close the socket with 1009.

## Behaviour and gotchas

- **Identity**: user IDs are opaque strings compared exactly (case-sensitive). The token `id` is not trimmed but participants are. Disallow commas (history query separator) and leading/trailing whitespace in IDs.
- **Conversation key**: the normalized participant set. Store it in your DB (sorted, joined) if you need a conversation list, unread counts or metadata; the service has no such endpoints.
- **Sender exclusion**: none of the sender's connections receive the message, including other tabs and devices. Render the sent message from your own `sendMessage` result plus local state. Other tabs of the sender only see it after a history reload. A note to self (`["alice"]`) is stored and never pushed.
- **Ordering**: messages are pushed in the order the service accepted the sends. Concurrent sends to one conversation interleave; sends a sender issues sequentially (awaiting each `202`) arrive in that order. History order is `created_at` (ms) desc then `id` desc, which can differ from push order for messages accepted in the same millisecond.
- **Delivery before persistence**: a pushed message may not be in history for a short time (batched async insert). A history read right after a `202` can miss it, so de-duplicate by `id` when merging live and fetched messages.
- **Durability limits**: a message acknowledged with `202` is lost if all `RETRY_ATTEMPTS` inserts fail (logged as `dropping messages after retries`), if shutdown exceeds its 15 s budget, or on a crash or `SIGKILL` before its batch is written. Give the container a stop grace period of at least 15 s (the compose block sets `stop_grace_period: 20s`; Docker's default is 10 s; in Kubernetes set `terminationGracePeriodSeconds`).
- **Backpressure**: `503` + `Retry-After: 1` means the 1024-message queue is full (MongoDB slow or down) or the service is stopping. Nothing was stored or delivered, so retrying is safe.
- **Rate limit**: a token bucket per token `id` (not per IP), shared by send and history, defaults 5/s burst 10. Neither `/ws` nor `/api/connections` is limited. 429 has no `Retry-After`, so back off yourself. If the backend calls history on behalf of many users, each user has their own bucket; the bucket is per user, not per backend. Limiter state is never evicted, so memory grows slowly with distinct user IDs since start.
- **No replay**: (re)connecting delivers only messages sent after the handshake. After any disconnect, reload page 0 of each open conversation.
- **Pagination**: offset-based, newest first. New messages shift pages, so when loading older pages while messages arrive, de-duplicate by `id` or stop when you see an `id` you already have. `size` is silently clamped to 100 and invalid `page`/`size` silently fall back. `page` > 10000 is rejected.
- **Token lifetime**: checked only at handshake/request time. Open sockets outlive token expiry. To cut off a user, close their relay socket in your backend.
- **Slow consumers**: a socket with 4 MiB undelivered is closed with 1008. In the relay, a slow browser is closed at 4 MiB buffered, so the relay keeps reading upstream promptly.
- **Single instance**: all users' sockets must hit the same chat instance; with multiple replicas, recipients on another replica silently get nothing live (history still works). `/api/connections` counts only local sockets. Run exactly one replica.
- **Startup coupling**: the service exits if MongoDB is not reachable within 10 s, so use `restart: unless-stopped` and a MongoDB health condition (as in the compose block). MongoDB outages after startup show up as 503 on sends (once the queue fills) and 500 on history.
- **Content**: arbitrary strings, stored and delivered unchanged. Escape on render (XSS is your responsibility). The size limit is effectively `MAX_BODY_BYTES` minus JSON overhead.
- **No TLS**: plain HTTP/WS on port 8080. Keep it on an internal network and terminate TLS at your edge or BFF.

## Smoke test checklist

Run from a directory where `jsonwebtoken` and `ws` are installed (your backend project), with the service reachable at `CHAT_URL`. All commands below were run against `1.0.1` with the outputs shown.

```bash
export CHAT_URL=http://localhost:8080
export CHAT_JWT_SECRET='<same value as the service JWT_SECRET>'

# 1. health -> {"status":"ok","time":"..."}
curl -fsS "$CHAT_URL/health"

# 2. mint tokens
mint() { node -e "console.log(require('jsonwebtoken').sign({id:process.argv[1]},process.env.CHAT_JWT_SECRET,{algorithm:'HS256',expiresIn:'10m'}))" "$1"; }
export ALICE=$(mint alice) BOB=$(mint bob) MALLORY=$(mint mallory)

# 3. listen as bob, send as alice -> bob receives the message
node -e "
const WebSocket = require('ws');
const ws = new WebSocket(process.env.CHAT_URL.replace(/^http/, 'ws') + '/ws', { headers: { Authorization: 'Bearer ' + process.env.BOB } });
ws.on('unexpected-response', (_req, res) => { console.error('handshake failed', res.statusCode); process.exit(1); });
ws.on('open', () => console.error('bob connected'));
ws.on('message', (data) => { console.log('bob received', data.toString()); process.exit(0); });
setTimeout(() => { console.error('no message within 10s'); process.exit(1); }, 10000);
" &
LISTENER=$!
sleep 1
curl -fsS -X POST "$CHAT_URL/api/messages" -H "Authorization: Bearer $ALICE" -H 'Content-Type: application/json' \
  -d '{"participants":["alice","bob"],"content":"smoke test"}'   # -> {"id":"...","status":"message queued"}
wait $LISTENER && echo "PASS websocket delivery"

# 4. history (persisted within ~1 s) -> array whose first element is the smoke test message
sleep 1
curl -fsS "$CHAT_URL/api/messages/get?participants=alice,bob&size=5" -H "Authorization: Bearer $BOB"

# 5. presence -> {"alice":0,"bob":0} once the listener has exited
curl -fsS -X POST "$CHAT_URL/api/connections" -H 'Content-Type: application/json' -d '{"users":["alice","bob"]}'

# 6. negative checks
curl -s -o /dev/null -w '%{http_code} (expect 401)\n' -X POST "$CHAT_URL/api/messages" -d '{}'
curl -s -o /dev/null -w '%{http_code} (expect 403)\n' "$CHAT_URL/api/messages/get?participants=alice,bob" -H "Authorization: Bearer $MALLORY"
curl -s -o /dev/null -w '%{http_code} (expect 401)\n' "$CHAT_URL/api/messages/get?participants=alice" -H "Authorization: Bearer $(CHAT_JWT_SECRET=wrong mint alice)"
```

Through your BFF relay: open `wss://<app>/chat/ws` from the browser devtools console on an allowed origin while logged in, send a message to that user from another account, and confirm one JSON frame arrives. From any other origin the handshake must fail with 403.
