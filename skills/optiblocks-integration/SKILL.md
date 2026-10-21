---
name: optiblocks-integration
description: Integrate the OptiBlocks plug-and-play Go microservices into an existing backend as components, from the published Docker Hub images. The services are real-time UI updates (backend → browser WebSocket fan-out with per-user user:<id> channels, HMAC time-token publishing), real-time chat (participant-based conversations, REST send/history, WebSocket delivery, MongoDB persistence, JWT auth) and the document module (data format conversion JSON/YAML/TOML/XML/CSV/TSV/MsgPack/Properties/XLSX, plus PDF/DOCX/XLSX generation from JSON). Covers choosing a service, adding the container to docker compose or Kubernetes, configuring secrets, minting the tokens each service expects, typed TypeScript clients, browser wiring, and smoke tests. Use this skill whenever the user mentions OptiBlocks, slincsilver/* images, real-time-ui-update-microservice, real-time-chat-microservice or document-module-microservice, or asks to add live updates, push notifications to the UI, WebSockets, chat or messaging, file or format conversion, or PDF/Word/Excel generation to a Node.js/NestJS/Express (or any) backend and these components are available.
---

# OptiBlocks integration

You are adding one or more OptiBlocks components to someone else's system. Each component is a small, independent Go service shipped as a signed Docker image. The product's own backend keeps all business logic and authorisation and calls the component over HTTP. Browsers talk to a component directly only where noted. The goal is a working, secure integration on the first try. Follow the workflow, read the reference for every service you touch, and run its smoke test before declaring success.

## The components

| Service | Image (pin this tag) | Purpose | Talks to browsers | Backend auth | Backing store | Health |
|---------|----------------------|---------|-------------------|--------------|---------------|--------|
| Real-time UI update | `slincsilver/real-time-ui-update-microservice:1.1.0` | Backend → browser event fan-out, per-user channels | yes, WebSocket (`?token=` JWT) | HMAC time token in `X-API-Token` | none | `GET /healthz` |
| Real-time chat | `slincsilver/real-time-chat-microservice:1.0.1` | User-to-user/group messaging with history | REST with `Authorization: Bearer`; WebSocket needs the header, so proxy it | user JWTs (HMAC, `id` claim) | MongoDB | `GET /health` |
| Document module | `slincsilver/document-module-microservice:1.0.1` | Format conversion; PDF/DOCX/XLSX rendering | no, private network only | **none** | none (needs writable `/tmp`) | `GET /health` |

Docker Hub: https://hub.docker.com/u/slincsilver. Repos: https://github.com/SantiagoLopezDeharo/OptiBlocks-collection.

Read the matching reference before writing any code. Each covers run config, the exact API contract, auth, a typed client, gotchas and a smoke test:
- `references/ui-update.md`
- `references/chat.md`
- `references/document.md`

## Pick the service

| The product needs… | Use | Not this |
|--------------------|-----|----------|
| "refresh when data changes", live status, dashboards, toasts, per-user notifications | UI update | chat (it is for user-authored messages) |
| users sending messages to each other, with history | chat | UI update (no persistence, clients cannot send) |
| converting partner payloads, CSV/XLSX import and export, generating invoices or reports as PDF/DOCX/XLSX | document | doing it in the Node event loop |
| guaranteed delivery or replay of events | none of these; use a queue or outbox. UI update and chat WebSockets do not replay missed messages | — |

## Shared facts (all three)

- **Images:** `FROM scratch`, running as non-root UID 65532, multi-arch (`linux/amd64`, `linux/arm64`).
  - There is no shell, `curl` or `wget` inside, so container `healthcheck` commands cannot run. Probe the health route from the orchestrator (Kubernetes `httpGet`, a load balancer, or a sidecar).
  - Run hardened: `read_only: true`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]`. The document module additionally needs `tmpfs: /tmp`.
  - Pin version tags (or digests) in production, never `latest`. Releases are cosign-signed keyless. Verify with `cosign verify <image> --certificate-identity=santiagolhgpro@gmail.com --certificate-oidc-issuer=https://accounts.google.com`.
- **Port:** every service listens on 8080 inside the container. When several run on one host, map different host ports or rely on the compose network (`http://ui-update:8080`, `http://chat:8080`, `http://document:8080`).
- **Configuration:** environment variables only. Invalid config exits at startup listing every error. A `.env` in the working directory is also read.
- **Lifecycle:** graceful shutdown on `SIGTERM`, with WebSocket clients receiving close `1001`.
- **Scaling:**
  - The document module is stateless, so it scales horizontally behind any load balancer.
  - UI update and chat keep WebSocket subscriptions in memory per instance, so do not load-balance one logical deployment across replicas.
  - UI update can run one independent node per region, and the backend publishes to each node.
  - Chat is single-instance.
- **TLS:** none of them terminate TLS. Put a reverse proxy or load balancer in front of anything browsers reach, and allow WebSocket upgrades with idle timeouts of at least 60 s.
- **pprof:** `PPROF_ADDR` enables profiling. Bind it to loopback only, or leave it unset.

## Integration workflow

1. **Inspect the target system before changing anything.**
   - Language and framework, and where outbound HTTP clients live.
   - How it deploys: compose, Kubernetes or another platform.
   - How users authenticate, and which JWT algorithm it uses. This matters: UI update and chat verify **HMAC** (`HS256/384/512`) tokens only.
   - Where secrets come from.
2. **Choose the services** with the table above and confirm the choice with the user if the need is ambiguous.
3. **Add the container(s)** using the compose block from each reference. Put them on the private network. Expose a port publicly only for the UI update WebSocket and the chat REST/WebSocket endpoints, and only through TLS. Never expose the document module publicly.
4. **Provision secrets.** Generate them with `openssl rand -hex 32` and store them in the project's secret manager or `.env` (gitignored), never in code. A secret shared between the backend and a service must match on both sides.
5. **Bridge authentication.**
   - If the product already issues HMAC JWTs with the claim the service expects, configure the service with the same secret.
   - Otherwise mint a separate short-lived "realtime" or "chat" token on the backend, behind its own authenticated endpoint. Each reference shows the exact claims.
   - Never hand a backend-only secret (time-token secret, signing keys) to the browser.
6. **Write one small client module per service** in the backend, following the typed client in the reference. Read base URLs and secrets from configuration, set timeouts, and map error statuses to the product's error handling.
7. **Wire the frontend** where applicable: the UI update browser helper with reconnect and resync, and the chat flow through the proxy pattern described in its reference.
8. **Call the component at the right moment.** Publish UI events after the database commit, and render documents off the request hot path when they are large. Never let an optional component failure (for example a UI event publish) fail the user's business transaction.
9. **Verify.** Run the reference's smoke test against the running container, then exercise the real integration path (e.g. trigger the product action and observe the browser receive the event). Report what was verified and anything left unverified.

## Security checklist

- Secrets are at least 32 random bytes, stored outside the repo, and different per environment and per service.
- User tokens for the components are short-lived (`exp` minutes, not days). The UI update and chat services check JWTs only at the handshake or per request.
- The document module and every backend-facing route (`/update`, `/publish`) are reachable only from the backend's network.
- `ALLOWED_ORIGINS` on UI update is set to the product's exact frontend origins.
- Authorisation stays in the product backend.
  - UI update shared channels are readable by any authenticated user who knows the name, so use `user:<id>` channels for personal data, or send ids and let the backend authorise the refetch.
  - Chat's only rule is "the caller is a participant", so any token holder can open a conversation with any user ids. If the product must restrict who may talk to whom (blocks, tenants, marketplace roles), route sends through a backend endpoint that checks that first.
- Payloads carry the minimum needed. Logs never print tokens. WebSocket URLs carrying `?token=` may appear in proxy logs, which is why those tokens must be short-lived.
