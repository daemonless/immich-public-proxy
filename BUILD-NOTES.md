# immich-public-proxy — FreeBSD daemonless build notes

| | |
|---|---|
| **Upstream** | `alangrainger/immich-public-proxy` @ `v3.0.1` (AGPL-3.0) |
| **Runtime** | node22 (no `engines` pin upstream; lowest node with no FreeBSD traps — no Temporal, no native addons) |
| **Database** | none (stateless proxy, no persistence, no volumes) |
| **Base** | `ghcr.io/daemonless/base:15` (service, s6) |

## FreeBSD-specific changes (and WHY)

- Two-stage build: node22 + git-lite + npm in the builder only; runtime installs
  just `node22` — nothing to compile, all 8 runtime deps are pure JS.
- The npm project root is `app/`, not the repo root (upstream monorepo layout).
- `npm ci --omit=dev` after compiling drops typescript/tsx/vitest from the
  shipped `node_modules`.
- `chmod -R a+rX /app` after the cross-stage COPY — builder-stage files land
  unreadable to the unprivileged `bsd` user (cookbook: root-umask/mode-700 trap).
- CIT health endpoint is `/` (the home page, 200 with no Immich behind it).
  Upstream's real `/healthcheck` pings the configured Immich and returns 503
  when it is unreachable — correct in production, guaranteed-fail in CIT. So:
  `.daemonless/config.yaml` CIT uses `/`; the `io.daemonless.healthcheck-url`
  label and `/healthz` script use `/healthcheck` for production semantics.
- `cit.ready: "Server started on port"` — IPP's startup line matches none of
  dbuild's default ready patterns; without it every CIT stalls 120 s
  (cookbook: "No ready signal after 120s").

## Patches (`patches/`) — why each exists

None. Upstream builds and runs unmodified on FreeBSD.

## Native dependencies handled

| Dep | FreeBSD status | How handled |
|---|---|---|
| — | all 8 runtime deps pure JS (archiver, cookie-session, dayjs, dotenv, express, preact, preact-render-to-string, tslib) | nothing to handle |

## Verified

- `dbuild build` ✅
- `dbuild test` ✅ — health-mode CIT: HTTP 200 on `/`, PUID/PGID re-chown pass.
  Boots and listens with no `IMMICH_URL` set (lazy proxy — Immich is only
  contacted per-request).
- `scripts/smoke-test.sh` ✅ — functional probe: home page 200, a bogus share
  key answers 404 via a real round-trip to a dummy Immich (a second instance
  of this image sharing the netns), and the process is still alive afterwards.

## Guards in place

- None needed: no patches (no patch-rot exposure), no injected native deps
  (no version-drift exposure). Upstream tag is pinned via `IPP_VERSION`.

## Known limitations

- `/healthcheck` (and therefore the container healthcheck in tunnel/standard
  mode) reports unhealthy while the backing Immich instance is unreachable.
  That is upstream's intended semantic, not a fault in this image.
- If `ipp.showHomePage` is disabled in a user's `config.json`, `/` returns 404;
  monitoring should target `/healthcheck`, not `/`.
- **Upstream bug (not FreeBSD-specific): a `/share/<key>` request while
  `IMMICH_URL` is unreachable crashes the whole Node process** — upstream's
  `fetchShareByKey` (`app/src/immich.ts`, `await fetch` with no try/catch)
  lets the rejection escape the async Express handler. s6 restarts the service
  (5s), so the image self-heals, but in-flight requests die whenever Immich is
  down and someone opens a share link. Found by `scripts/smoke-test.sh`
  (2026-07-04); worth reporting upstream. Same behaviour in upstream's own
  Docker image; deliberately NOT patched here to avoid diverging from upstream
  runtime semantics.
