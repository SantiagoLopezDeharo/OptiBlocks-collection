# Real-Time UI Update Microservice — integration reference

Pushes backend events to browsers over WebSockets. The backend makes one authenticated `POST` per event, and the service fans the JSON payload out to every client subscribed to the channel. It has no database and keeps no history.

- Image: `slincsilver/real-time-ui-update-microservice:1.1.0` (https://hub.docker.com/r/slincsilver/real-time-ui-update-microservice), `linux/amd64` + `linux/arm64`
- Repo: https://github.com/SantiagoLopezDeharo/real-time-ui-update-microservice

## When to use / when not

Use it for:
- live dashboards, order or job status, notifications, "record changed, refetch" hints, and back-office consoles;
- per-user notifications through `user:<id>` channels;
- keeping thousands of idle sockets out of a Node.js process.

Do not use it for:
- **Guaranteed delivery or replay.** Messages to offline or reconnecting clients are lost. Keep the source of truth in the backend and treat events as hints.
- **Client-to-client messaging.** Client frames are discarded; use the chat service for that.
- **Wildcard or multi-channel subscriptions.** Each socket subscribes to exactly one channel, so open one socket per channel.
- **Payloads that are not JSON.** They are rejected with `400`.

## Run it

```shell
docker run -d --name ui-update -p 8080:8080 \
  --read-only --cap-drop=ALL --security-opt=no-new-privileges \
  -e JWT_SECRET="$JWT_SECRET" \
  -e TIME_TOKEN_SECRET="$TIME_TOKEN_SECRET" \
  -e ALLOWED_ORIGINS="https://app.example.com" \
  slincsilver/real-time-ui-update-microservice:1.1.0
```

Compose service to paste into an existing stack:

```yaml
  ui-update:
    image: slincsilver/real-time-ui-update-microservice:1.1.0
    restart: unless-stopped
    environment:
      JWT_SECRET: ${UI_UPDATE_JWT_SECRET:?set UI_UPDATE_JWT_SECRET}
      TIME_TOKEN_SECRET: ${UI_UPDATE_TIME_TOKEN_SECRET:?set UI_UPDATE_TIME_TOKEN_SECRET}
      ALLOWED_ORIGINS: ${UI_UPDATE_ALLOWED_ORIGINS:-}
    ports:
      - "8080:8080"          # browsers connect here (put TLS in front in production)
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 512m
    pids_limit: 256
```

- The image is `scratch`: there is no shell, `curl` or `wget`, so a Docker `HEALTHCHECK`/compose `healthcheck` command cannot run inside it. Probe `GET /healthz` from the orchestrator instead (Kubernetes `httpGet`, or a load balancer target check).
- Browsers must reach it directly (WSS through a TLS-terminating proxy or load balancer). The backend reaches it over the private network, e.g. `http://ui-update:8080`.
- Proxies must allow WebSocket upgrades and long-lived connections. The server pings every 30 s, so a proxy idle timeout of 60 s or more is enough.
- Verify the image before deploying (optional): `cosign verify slincsilver/real-time-ui-update-microservice:1.1.0 --certificate-identity=santiagolhgpro@gmail.com --certificate-oidc-issuer=https://accounts.google.com`.

## Configuration

| Variable | Default | Validation | Purpose |
|----------|---------|------------|---------|
| `JWT_SECRET` | required | non-empty | HMAC key that verifies user JWTs on `/ws` |
| `TIME_TOKEN_SECRET` | required | non-empty | HMAC key for backend time tokens on `/update` and `/publish` |
| `PORT` | `8080` | — | Listen port |
| `TIME_WINDOW_SECONDS` | `3600` | integer ≥ 1 | Time token window size; the backend must use the same value |
| `ALLOWED_CLOCK_SKEW` | `1` | integer ≥ 0 | Windows accepted on each side of the current one |
| `MAX_BODY_BYTES` | `1048576` | integer ≥ 1 | Max publish body, otherwise `413` |
| `ALLOWED_ORIGINS` | empty (any) | comma separated | Exact `Origin` allowlist for browser WebSockets; requests without `Origin` are always allowed |
| `PPROF_ADDR` | off | address | Optional pprof listener; bind to loopback only |

Invalid configuration makes the process exit at startup, reporting every error at once. A `.env` file in the working directory is loaded if present.

## Auth model

There are two independent secrets.

**Backend → service: time token** in the `X-API-Token` header, signed with `TIME_TOKEN_SECRET`:

```
window = floor(unix_seconds / TIME_WINDOW_SECONDS)                      // decimal string
token  = base64url( window + ":" + base64url( HMAC-SHA256(secret, window) ) )   // unpadded
```

- A token is valid for its window ± `ALLOWED_CLOCK_SKEW` windows, so mint a fresh one per request (it's cheap). It is read only from the header, never from the query string.
- Keep `TIME_WINDOW_SECONDS` short in production (e.g. 30–300) so a leaked token expires quickly. Backend and service clocks must be NTP-synced.

**Browser → service: user JWT** in the `token` query parameter (browsers cannot set headers on WebSockets):

- HMAC only: `HS256`, `HS384` or `HS512`, signed with `JWT_SECRET`. RS256 and ES256 tokens (Auth0, Cognito, Firebase) are rejected with `401`.
- `sub` must be a string if present. It is the user id used by `user:<id>` channels. A token without `sub` is still accepted on shared channels.
- `exp` and `nbf` are enforced when present, but `exp` is not required. **Always set a short `exp`.**
- The token is only checked at the handshake. An open socket stays open after `exp`, until it disconnects.

If the product's own auth uses asymmetric tokens, the backend mints a separate short-lived **realtime token** for the socket:

```ts
import jwt from "jsonwebtoken";

export function realtimeToken(userId: string, secret = process.env.UI_UPDATE_JWT_SECRET!): string {
  return jwt.sign({}, secret, { algorithm: "HS256", subject: userId, expiresIn: "5m" });
}
// Expose it via an authenticated endpoint, e.g. GET /realtime/token → { token }
```

Query strings can end up in proxy access logs. That is another reason to keep realtime tokens short-lived and separate from session tokens.

## API contract

### `GET /ws?token=<JWT>&channel=<name>` (authenticated subscribe)

- `channel` is optional and defaults to `default`. It is an opaque, case-sensitive string with no wildcards. If repeated, the first value counts.
- Allowed channels are every shared channel plus `user:<sub>` for the token's own `sub`.
- Status codes:
  - `101` upgraded.
  - `401` missing, invalid, expired or non-HMAC JWT, or a non-string `sub`. An `Authorization` header is ignored.
  - `403` the channel belongs to another user, or `Origin` is not allowed. This is returned before the upgrade.
  - `400` not a WebSocket handshake.

### `GET /ws/public?channel=<name>` (anonymous subscribe)

- Any channel except `user:*`, which gets `403`. Public and authenticated subscribers are isolated even on the same channel name.

### Frames (server → client)

- Each publish becomes one text frame whose content is the exact JSON body that was published. Frames are never merged or wrapped.
- Messages from one publisher arrive in publish order.
- Client → server frames are read and discarded. A client frame over 512 bytes closes the socket with `1009`.

| Close code | Meaning | Client action |
|------------|---------|---------------|
| `1000` | normal close | reconnect if still needed |
| `1001` | server shutting down | reconnect with backoff (another node or restart) |
| `1008` + `slow consumer` | client fell more than 4 MiB behind | reconnect, then refetch state from the backend |
| `1009` | client sent a frame over 512 bytes | fix the client; do not send data |
| `1006` | network drop, or no pong within 60 s | reconnect with backoff |

### `POST /update?channel=<name>` and `POST /publish?channel=<name>`

- `/update` delivers to `/ws` (authenticated) subscribers, and `/publish` to `/ws/public` subscribers.
- Headers: `X-API-Token: <time token>` (required). `Content-Type: application/json` is recommended but not checked.
- Body: any valid JSON value, up to `MAX_BODY_BYTES`.

| Status | Body (text/plain) | Cause |
|--------|-------------------|-------|
| `202` | empty | queued for every current subscriber; also returned when there are none |
| `400` | `Invalid JSON` | body is not valid JSON (an empty body included) |
| `400` | `User channels are private` | `/publish` targeted `user:*`; use `/update` |
| `400` | `Error reading body` | the body stream failed |
| `401` | `Token required` / `Invalid or expired token` | missing or invalid `X-API-Token` |
| `405` | — | wrong method |
| `413` | `Body too large` | body over `MAX_BODY_BYTES` |

A `202` does not tell you how many clients received the message, and there is no acknowledgement from clients.

### `GET /healthz`

Returns `200` with an empty body once the server is listening.

## Backend integration recipe

Typed client (Node 18+, no dependencies):

```ts
import { createHmac } from "node:crypto";

export interface UiUpdateOptions {
  baseUrl: string;              // e.g. "http://ui-update:8080"
  timeTokenSecret: string;
  timeWindowSeconds?: number;   // must equal the service's TIME_WINDOW_SECONDS (default 3600)
  timeoutMs?: number;
}

export class UiUpdateError extends Error {
  constructor(readonly status: number, message: string) {
    super(`ui-update ${status}: ${message}`);
  }
}

export class UiUpdateClient {
  private readonly windowSeconds: number;
  private readonly timeoutMs: number;

  constructor(private readonly opts: UiUpdateOptions) {
    this.windowSeconds = opts.timeWindowSeconds ?? 3600;
    this.timeoutMs = opts.timeoutMs ?? 5000;
  }

  /** Broadcast to authenticated subscribers of a shared channel. */
  update(channel: string, payload: unknown): Promise<void> {
    return this.post("/update", channel, payload);
  }

  /** Deliver only to the tabs of one user (JWT sub === userId). */
  notifyUser(userId: string, payload: unknown): Promise<void> {
    return this.post("/update", `user:${userId}`, payload);
  }

  /** Broadcast to anonymous /ws/public subscribers. */
  publish(channel: string, payload: unknown): Promise<void> {
    return this.post("/publish", channel, payload);
  }

  timeToken(nowMs = Date.now()): string {
    const window = Math.floor(nowMs / 1000 / this.windowSeconds).toString();
    const mac = createHmac("sha256", this.opts.timeTokenSecret).update(window).digest("base64url");
    return Buffer.from(`${window}:${mac}`).toString("base64url");
  }

  private async post(path: string, channel: string, payload: unknown): Promise<void> {
    const res = await fetch(`${this.opts.baseUrl}${path}?channel=${encodeURIComponent(channel)}`, {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-API-Token": this.timeToken() },
      body: JSON.stringify(payload),
      signal: AbortSignal.timeout(this.timeoutMs),
    });
    if (res.status !== 202) {
      throw new UiUpdateError(res.status, (await res.text()).trim());
    }
  }
}
```

Usage, e.g. in a NestJS provider or an Express service:

```ts
const ui = new UiUpdateClient({
  baseUrl: process.env.UI_UPDATE_URL!,
  timeTokenSecret: process.env.UI_UPDATE_TIME_TOKEN_SECRET!,
  timeWindowSeconds: Number(process.env.UI_UPDATE_TIME_WINDOW_SECONDS ?? 3600),
});

// after the DB transaction commits:
await ui.update("orders", { event: "order.updated", id: order.id, version: order.version });
await ui.notifyUser(order.customerId, { event: "order.shipped", id: order.id });
```

Rules for the backend:
- Publish **after** the database commit, never inside the transaction.
- Keep payloads small: ids and versions, not whole records. Clients refetch over the normal API, so authorisation stays in the backend.
- Treat a failed publish as best-effort: log it and move on. Don't fail the business request because of it. Retrying is safe only if clients dedupe, so include an `id` or `version` in the payload.
- Several nodes (e.g. one per region) don't share state. Publish to every node whose users must receive the event, e.g. with `Promise.allSettled` over clients.
- Map product concepts to channels deliberately: `org:<orgId>:orders` for tenant-wide events, `user:<userId>` for personal events. Shared channels are visible to **any** authenticated user who knows the name, so never put tenant data on a guessable shared channel. Use `user:` channels, or send ids only and let the backend authorise the refetch.

Browser client with reconnect and resync:

```js
export function subscribe({ baseWsUrl, channel, getToken, onMessage, onResync }) {
  let socket, retry = 0, stopped = false;

  async function connect() {
    const token = await getToken();                       // fetch a fresh short-lived realtime JWT
    const url = `${baseWsUrl}/ws?channel=${encodeURIComponent(channel)}&token=${encodeURIComponent(token)}`;
    socket = new WebSocket(url);
    socket.onopen = () => { retry = 0; onResync?.(); };   // refetch state: messages sent while offline are lost
    socket.onmessage = (e) => onMessage(JSON.parse(e.data));
    socket.onclose = () => {
      if (stopped) return;
      const delay = Math.min(30000, 500 * 2 ** retry++) * (0.5 + Math.random() / 2);
      setTimeout(connect, delay);
    };
  }

  connect();
  return () => { stopped = true; socket?.close(1000); };
}
```

A failed handshake (`401`/`403`) surfaces in the browser as a close, without a readable status. If reconnects keep failing, check that the token and channel are correct rather than retrying forever.

## Behaviour and gotchas

- **No persistence or replay.** Reconnecting clients must resync from the backend (`onResync`).
- **One channel per socket.** Each extra channel is another socket; browsers allow many per origin, but keep the number small.
- **Ordering** holds per publisher. Concurrent publishers interleave arbitrarily, so carry a `version` or timestamp when order matters across publishers.
- **Slow consumers are evicted** once a client has 4 MiB undelivered (close `1008`). Large payloads to slow mobile clients trigger this sooner.
- **User channel match is exact**: `user:Alice` ≠ `user:alice`, with no trimming. Ids containing `:` are safe (`user:a` never matches `user:a:b`). There are no sub-channels, so put the event kind in the payload.
- **Public vs authenticated** are separate namespaces: `/update` to `news` never reaches `/ws/public?channel=news`.
- **JWT expiry is checked only at connect.** To cut a user off, they must disconnect. There is no server-side kick API.
- **The `Origin` allowlist is exact**: scheme, host and port must match, and it is case-sensitive.
- **Graceful shutdown:** on `SIGTERM` clients receive `1001`. Rolling deploys therefore cause one reconnect per client.
- **Capacity:** roughly 16 KiB RSS per idle connection, and ~237k delivered msg/s on an M3 Pro. Throughput is bounded by write syscalls.

## Smoke test checklist

```shell
export JWT_SECRET=dev-jwt TIME_TOKEN_SECRET=dev-time
docker run -d --rm --name ui-update-smoke -p 8080:8080 -e JWT_SECRET -e TIME_TOKEN_SECRET \
  -e TIME_WINDOW_SECONDS=30 slincsilver/real-time-ui-update-microservice:1.1.0

# 1. health
curl -fsS -o /dev/null -w '%{http_code}\n' http://localhost:8080/healthz           # 200

# 2. backend token (same window as the service)
TOKEN=$(node -e 'const c=require("crypto");const w=Math.floor(Date.now()/1000/30).toString();
const m=c.createHmac("sha256",process.env.TIME_TOKEN_SECRET).update(w).digest("base64url");
process.stdout.write(Buffer.from(w+":"+m).toString("base64url"))')

# 3. subscribe as user 42 and receive a personal message (Node 22+ has a global WebSocket)
node -e '
const c=require("crypto"),b=s=>Buffer.from(s).toString("base64url");
const h=b(JSON.stringify({alg:"HS256",typ:"JWT"})),p=b(JSON.stringify({sub:"42",exp:Math.floor(Date.now()/1000)+300}));
const jwt=h+"."+p+"."+c.createHmac("sha256",process.env.JWT_SECRET).update(h+"."+p).digest("base64url");
const ws=new WebSocket("ws://localhost:8080/ws?channel=user:42&token="+jwt);
ws.onmessage=e=>{console.log("received",e.data);process.exit(0)};
setTimeout(()=>{console.error("timeout");process.exit(1)},5000);' &
sleep 1
curl -s -o /dev/null -w '%{http_code}\n' -X POST 'http://localhost:8080/update?channel=user:42' \
  -H "X-API-Token: $TOKEN" -H 'Content-Type: application/json' -d '{"event":"smoke"}'   # 202
wait                                                                                     # received {"event":"smoke"}

# 4. negative checks
curl -s -o /dev/null -w '%{http_code}\n' -X POST 'http://localhost:8080/update' -d '{}'  # 401
curl -s -o /dev/null -w '%{http_code}\n' -X POST 'http://localhost:8080/publish?channel=user:42' \
  -H "X-API-Token: $TOKEN" -d '{}'                                                      # 400

docker stop ui-update-smoke
```
