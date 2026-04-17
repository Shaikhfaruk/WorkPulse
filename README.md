# WorkPulse — Team Edition
### Centralized AI Work Monitor over LAN · Open Source

---

## 0. TL;DR

- **Old plan:** one user, one laptop, one local AI. Solo productivity app.
- **New plan:** one **Hub** machine on the office LAN runs the AI + database + dashboard. Every team member installs an **Agent** that auto-discovers the Hub over WiFi, streams activity, and surfaces AI-generated check-ins. A **manager dashboard** is served from the Hub in the browser.
- **Everything stays on the LAN.** No cloud, no external API keys, no internet traffic. Ollama runs on the Hub.
- **Open source (AGPL-3.0)** — because monitoring software that isn't auditable shouldn't be trusted.
- Target: **5–100 employees per Hub**. Runs on a mid-tier workstation or a dedicated mini-PC/NUC.

---

## 1. System Architecture

Three components, one private network:

```
            ┌─────────────────────────────────────────────┐
            │            OFFICE LAN (WiFi)                │
            │                                             │
            │   ┌───────────────────────────────────┐     │
            │   │         WORKPULSE HUB             │     │
            │   │   (one dedicated machine)         │     │
            │   │                                   │     │
            │   │  Caddy (reverse proxy, mTLS)      │     │
            │   │    ├─ Fastify API + Socket.IO     │     │
            │   │    ├─ Next.js Dashboard           │     │
            │   │    └─ Ollama (llama3.1 / mistral) │     │
            │   │                                   │     │
            │   │  PostgreSQL + TimescaleDB         │     │
            │   │  Redis (BullMQ job queue)         │     │
            │   │  Prometheus + Grafana             │     │
            │   │  mDNS: _workpulse._tcp.local      │     │
            │   └───────┬──────────────────┬────────┘     │
            │           │                  │              │
            │    WebSocket (wss)      HTTPS (dashboard)   │
            │           │                  │              │
            │   ┌───────┴──────┐     ┌─────┴────────┐     │
            │   │  AGENT       │     │  Manager     │     │
            │   │  (Electron)  │     │  (browser)   │     │
            │   │  × N devices │     │              │     │
            │   └──────────────┘     └──────────────┘     │
            └─────────────────────────────────────────────┘
```

### Components at a glance

| Component | Count | Role |
|---|---|---|
| **Hub** | 1 | Runs everything server-side: AI, DB, API, dashboard, metrics |
| **Agent** | N (one per employee) | Tray app, activity collector, check-in UI |
| **Dashboard** | 0 install | Web app served by the Hub at `https://workpulse.local` |

The Hub is a logical unit; internally it's a Docker Compose stack so each piece (Ollama, Postgres, API, etc.) is a separate container with its own lifecycle.

---

## 2. Network Architecture

### Discovery

The Hub advertises itself on the LAN via **mDNS** (`_workpulse._tcp.local`). Agents scan on first launch and find it automatically — no IP address to type, no DNS configuration. The Hub also registers a friendly hostname (`workpulse.local`) so the dashboard is reachable in any browser on the network.

### Transport & Security

- **mTLS**: The Hub runs its own tiny Certificate Authority (generated at install via `mkcert`). The CA root is bundled into the Agent installer. Every WebSocket connection is mutually authenticated with a cert-per-device issued during pairing. No MITM, no token replay.
- **wss://** for all Agent ↔ Hub traffic.
- **https://** for the dashboard (same CA, automatic cert via Caddy).
- **No inbound ports from the public internet.** The Hub binds to the private subnet (`192.168.x.x`, `10.x.x.x`) by default and refuses to bind publicly unless `WORKPULSE_ALLOW_PUBLIC=1` is set — with a loud warning.

### Offline resilience

- Agent keeps a **local SQLite buffer** for up to 72 hours of queued events.
- On WiFi drop, Agent keeps sampling activity → writes to buffer.
- On reconnect, Agent batch-uploads with server-side idempotency keys (no duplicates).
- Agent applies exponential backoff (1s → 2s → 4s → … → 60s cap) so it doesn't hammer the Hub.

---

## 3. Tech Stack

### Hub (central server)

| Layer | Choice | Why |
|---|---|---|
| Runtime | Node.js 20 LTS | Shared with Agent/Dashboard |
| HTTP framework | **Fastify 4** | 3–4× faster than Express, built-in Zod/JSON-schema |
| Real-time | **Socket.IO 4** (rooms per user) | Auto-reconnect, fallback, battle-tested |
| Database | **PostgreSQL 16** + **TimescaleDB** extension | Activity events are time-series; TimescaleDB hypertables compress and query 10–100× better than plain Postgres |
| ORM | **Prisma 5** | Typed queries, migrations, shared types |
| Job queue | **BullMQ** on Redis | AI inference queue, report generation, retention jobs |
| Cache / pub-sub | **Redis 7** | BullMQ backend, Socket.IO adapter for horizontal scale |
| AI runtime | **Ollama** (`llama3.1:8b-instruct`, `mistral:7b`, or `qwen2.5:7b`) | Offline, local, good enough for coaching prompts |
| Reverse proxy | **Caddy 2** | Auto-HTTPS, zero-config, handles the mTLS dance |
| Logging | **Pino** (JSON, piped to file + stdout) | Fastest Node logger, structured |
| Metrics | **Prometheus** + **Grafana** (shipped, pre-wired) | Request rates, queue depth, AI latency, active agents |
| Tracing (optional) | **OpenTelemetry → Tempo** | For debugging complex flows |
| Discovery | **bonjour-service** | mDNS broadcast + hostname |
| Secrets | OS keychain on Hub host + Docker secrets | No plaintext creds in env files in prod |
| Auth | JWT (RS256) + device pairing codes + **OIDC** for dashboard (optional) | Supports SSO if the org wants it |

### Agent (employee desktop)

| Layer | Choice | Why |
|---|---|---|
| Shell | **Electron 30+** | Cross-platform tray, notifications, auto-update |
| UI | **React 18 + Tailwind + shadcn/ui** | Fast to build, consistent design |
| Activity | **active-win** | Focused app + window title, cross-platform |
| Input events | **uiohook-napi** | Maintained fork of abandoned `iohook` — idle detection, no content capture |
| Transport | **Socket.IO client** | Auto-reconnect, offline queue hook |
| Local buffer | **better-sqlite3** | Synchronous, fast, no daemon |
| Token storage | **keytar** (OS keychain) | Never writes secrets to disk in plaintext |
| Auto-update | **electron-updater**, pointing to **Hub-hosted** update feed | Updates pull from Hub, not internet |
| Packaging | **electron-builder** | `.exe` (NSIS), `.dmg`, `.AppImage`, `.deb`, `.rpm` |
| Code signing | Windows Authenticode + macOS notarization | Required for clean installs on modern OSes |

### Dashboard (manager web UI)

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js 14** (App Router) | SSR, mounted inside the Hub on `/dashboard` |
| Data fetching | **TanStack Query** | Cache + background refetch |
| Real-time | Socket.IO client | Live team feed, presence |
| Tables | **TanStack Table** | Server-side sort/filter for big teams |
| Charts | **Recharts** + **visx** | Heatmaps, timelines, stacked activity |
| Components | **shadcn/ui** + Radix primitives | Accessible, themeable |
| Auth | JWT + role-based guards | `EMPLOYEE` (self-view) / `MANAGER` / `ADMIN` |

### Shared / tooling

| Concern | Tool |
|---|---|
| Monorepo | **pnpm workspaces** + **Turborepo** |
| Language | **TypeScript 5** everywhere |
| Schemas | **Zod** (runtime + inferred types, shared across hub/agent/dashboard) |
| Lint/format | **Biome** (one tool replaces ESLint + Prettier) |
| Testing | **Vitest** (unit) + **Playwright** (E2E) |
| CI | **GitHub Actions**: lint, typecheck, test, build, release |
| Releases | **changesets** (semver + changelog automation) |
| Dev env | **Docker Compose** for Postgres/Redis/Ollama locally |
| Docs site | **Nextra** → GitHub Pages |

---

## 4. Folder Structure (Monorepo)

```
workpulse/
├── apps/
│   ├── hub/                          # Central server — Fastify + Socket.IO
│   │   ├── src/
│   │   │   ├── api/                  # REST: /pairing, /admin, /reports, /health
│   │   │   ├── sockets/              # Socket.IO: activity, checkin, presence
│   │   │   ├── ai/
│   │   │   │   ├── ollama.ts         # HTTP client to localhost:11434
│   │   │   │   ├── context.ts        # Per-user rolling summary builder
│   │   │   │   ├── teamContext.ts    # Team aggregate (anonymized for prompts)
│   │   │   │   └── queue.ts          # BullMQ inference queue
│   │   │   ├── scheduler/            # node-cron: check-ins, EOD digest, retention
│   │   │   ├── discovery/mdns.ts     # Bonjour broadcaster
│   │   │   ├── auth/
│   │   │   │   ├── pairing.ts        # 6-digit codes, device certs
│   │   │   │   ├── jwt.ts
│   │   │   │   └── rbac.ts
│   │   │   ├── observability/
│   │   │   │   ├── logger.ts         # Pino config
│   │   │   │   ├── metrics.ts        # Prometheus registry
│   │   │   │   └── audit.ts          # Immutable audit log (append-only)
│   │   │   ├── retention/purge.ts    # Data retention cron
│   │   │   └── index.ts              # Boot
│   │   ├── test/
│   │   ├── Dockerfile
│   │   └── package.json
│   │
│   ├── agent/                        # Employee Electron app
│   │   ├── electron/
│   │   │   ├── main.ts               # App lifecycle, IPC
│   │   │   ├── tray.ts               # Menu, pause button, status
│   │   │   ├── monitor.ts            # active-win + uiohook sampler
│   │   │   ├── discovery.ts          # mDNS client
│   │   │   ├── buffer.ts             # Offline SQLite queue
│   │   │   ├── redaction.ts          # Window-title regex blocklist
│   │   │   └── updater.ts            # electron-updater → Hub feed
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── PairingScreen.tsx
│   │   │   │   ├── CheckInPopup.tsx
│   │   │   │   ├── PersonalDashboard.tsx
│   │   │   │   └── ConsentDialog.tsx # First-run transparency screen
│   │   │   ├── services/
│   │   │   │   ├── socket.ts
│   │   │   │   ├── secureStore.ts    # keytar wrapper
│   │   │   │   └── activityStream.ts
│   │   │   └── App.tsx
│   │   ├── build/                    # electron-builder config, icons
│   │   └── package.json
│   │
│   ├── dashboard/                    # Manager web UI (Next.js)
│   │   ├── app/
│   │   │   ├── login/
│   │   │   ├── team/                 # Team grid
│   │   │   ├── live/                 # Real-time feed
│   │   │   ├── member/[id]/          # Per-employee deep dive
│   │   │   ├── reports/              # EOD digests, weekly trends
│   │   │   ├── audit/                # Who viewed what, when (manager oversight)
│   │   │   └── settings/
│   │   ├── components/
│   │   └── package.json
│   │
│   └── cli/                          # workpulse-ctl — admin CLI
│       └── src/                      # pair, revoke, export, backup, restore
│
├── packages/
│   ├── protocol/                     # Zod schemas + Socket.IO event contracts
│   │   └── src/
│   │       ├── events.ts             # Single source of truth for WS messages
│   │       ├── activity.ts
│   │       ├── checkin.ts
│   │       └── user.ts
│   ├── db/                           # Prisma schema + generated client
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── src/index.ts
│   ├── ai-prompts/                   # Prompt templates (versioned)
│   │   └── src/
│   │       ├── morning.ts
│   │       ├── afternoon.ts
│   │       ├── evening.ts
│   │       └── manager-digest.ts
│   ├── config/                       # Shared config loader + validation
│   └── ui/                           # Shared React components
│
├── infra/
│   ├── compose/
│   │   ├── docker-compose.yml        # Production stack
│   │   ├── docker-compose.dev.yml    # Local dev overrides
│   │   └── .env.example
│   ├── caddy/Caddyfile               # Reverse proxy + auto mTLS
│   ├── prometheus/prometheus.yml
│   ├── grafana/dashboards/           # Pre-built team-health dashboard
│   └── scripts/
│       ├── install-hub.sh            # One-command Hub bootstrap
│       ├── backup.sh                 # pg_dump + volume snapshot
│       ├── restore.sh
│       └── rotate-ca.sh
│
├── docs/
│   ├── SETUP.md
│   ├── ARCHITECTURE.md
│   ├── PRIVACY.md                    # Every data point, plain English
│   ├── SECURITY.md                   # Threat model, reporting vulns
│   ├── OPERATIONS.md                 # Backup, upgrade, troubleshoot
│   ├── CONTRIBUTING.md
│   ├── CODE_OF_CONDUCT.md
│   └── adr/                          # Architecture Decision Records
│
├── tests/
│   ├── e2e/                          # Playwright — pairing, check-in, offline
│   └── load/                         # k6 scripts — 100 agents × 60s activity
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── release.yml
│   │   └── security-scan.yml         # Trivy, npm audit, CodeQL
│   └── ISSUE_TEMPLATE/
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── LICENSE                           # AGPL-3.0
└── README.md
```

---

## 5. Data Model (Prisma, condensed)

```prisma
enum Role { EMPLOYEE  TEAM_LEAD  MANAGER  ADMIN }
enum CheckInType { MORNING  AFTERNOON  EVENING }

model User {
  id           String    @id @default(cuid())
  name         String
  email        String?   @unique
  role         Role      @default(EMPLOYEE)
  teamId       String?
  team         Team?     @relation(fields: [teamId], references: [id])
  devices      Device[]
  tasks        Task[]
  checkIns     CheckIn[]
  activity     ActivityEvent[]
  deactivatedAt DateTime?
  createdAt    DateTime  @default(now())
}

model Team {
  id        String @id @default(cuid())
  name      String
  members   User[]
  managerId String?
}

model Device {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  hostname    String
  os          String
  certFingerprint String @unique     // mTLS cert pinned to this device
  pairedAt    DateTime @default(now())
  lastSeenAt  DateTime @default(now())
  revokedAt   DateTime?
}

// Hypertable (TimescaleDB) — partitioned by ts, compressed after 7 days
model ActivityEvent {
  id          BigInt   @id @default(autoincrement())
  userId      String
  ts          DateTime
  appName     String
  windowTitle String                  // post-redaction
  idle        Boolean
  @@index([userId, ts])
}

model Task {
  id          String   @id @default(cuid())
  userId      String
  date        DateTime @db.Date
  text        String
  completed   Boolean  @default(false)
  carriedOver Boolean  @default(false)
  createdAt   DateTime @default(now())
}

model CheckIn {
  id          String   @id @default(cuid())
  userId      String
  type        CheckInType
  aiPrompt    String
  userReply   String
  aiReply     String
  contextJson Json                    // snapshot of activity at the moment
  ts          DateTime @default(now())
}

model TeamReport {
  id        String   @id @default(cuid())
  teamId    String
  date      DateTime @db.Date
  markdown  String
  createdAt DateTime @default(now())
}

// Append-only — who (manager) looked at whom, when
model AuditLog {
  id        BigInt   @id @default(autoincrement())
  actorId   String
  action    String                    // 'VIEW_MEMBER', 'EXPORT_DATA', 'REVOKE_DEVICE'
  targetId  String?
  metadata  Json?
  ts        DateTime @default(now())
  @@index([actorId, ts])
  @@index([targetId, ts])
}
```

Key production choices:

- `ActivityEvent` is a **TimescaleDB hypertable**. At 1 event/minute × 50 users × 8h = 24k rows/day → 8.7M/year. Native Postgres struggles; Timescale keeps query times < 50ms with compression.
- `certFingerprint` is unique per device → we can revoke one device without affecting others.
- `AuditLog` is intentionally separate and append-only; it's how we keep managers honest.

---

## 6. End-to-End Flows

### 6.1 First-time setup (Admin)

```bash
curl -fsSL get.workpulse.dev/hub | bash
# or:
git clone https://github.com/you/workpulse && cd workpulse/infra/compose
cp .env.example .env && docker compose up -d
```

Boot sequence:
1. Postgres + Redis + Ollama start.
2. Hub pulls the configured model (`ollama pull llama3.1:8b`).
3. Caddy generates a local CA, issues certs for `workpulse.local`.
4. Hub generates a device-pairing root JWT keypair (RS256).
5. Bonjour broadcasts `_workpulse._tcp.local`.
6. Admin opens `https://workpulse.local/dashboard` → creates first manager account → QR code shown for Agent installers.

### 6.2 Agent pairing (Employee)

1. Employee downloads the installer from `https://workpulse.local/download` (served by Hub) — already includes the Hub's CA root.
2. First launch shows a **consent dialog** listing everything that will be collected. Employee must tick each item.
3. Agent auto-discovers the Hub via mDNS.
4. Employee enters a one-time **6-digit pairing code** (shown on the manager's dashboard, expires in 10 min).
5. Agent generates a device keypair → sends CSR → Hub issues a device certificate.
6. Cert + JWT stored in OS keychain via `keytar`.
7. Green tray icon. Paired.

### 6.3 Activity streaming (every 60s, per Agent)

```
Agent.monitor.ts
  sample active-win + idle state
  apply redaction rules (regex blocklist)
  push to socket: 'activity:snapshot'
        │
        ▼
Hub sockets/activity.ts
  zod.parse(payload)   // reject if malformed
  idempotency check    // skip duplicates on reconnect
  INSERT into activity_events
  invalidate user's rolling-context cache
```

### 6.4 Check-in cycle (scheduled by Hub)

```
09:00 scheduler fires for userId
  └─► enqueue BullMQ job 'checkin:morning'
          └─► worker builds context → Ollama → get prompt text
                └─► emit 'checkin:request' to userId's socket room
                        └─► Agent pops CheckInPopup
                                └─► user types answer → emit 'checkin:response'
                                        └─► enqueue 'checkin:ai-reply'
                                                └─► worker: context + response → Ollama
                                                        └─► stream reply back to Agent
                                                        └─► persist in check_ins table
```

Why queue it: 50 users at 9am = 50 concurrent Ollama calls would meltdown a single-GPU box. BullMQ processes them with configurable concurrency (default 4), and the UX stays fine because check-ins have a natural 5–15 min window.

### 6.5 End-of-day team digest (18:30)

```
reportCron
  for each team:
    gather today's activity + check-ins + tasks
    build team_data_json (anonymized employee IDs inside the prompt)
    single Ollama call → Markdown report
    save to team_reports
    notify manager via dashboard toast + optional SMTP on LAN
```

### 6.6 Manager views a team member

```
GET /dashboard/member/abc123
  ├─► auth guard: requires role MANAGER or TEAM_LEAD of this team
  ├─► INSERT audit_log (action='VIEW_MEMBER', target=abc123)
  ├─► fetch today's activity, check-ins, task status
  └─► render
```

The audit-log insert is deliberate and surfaced in the member's Personal Dashboard ("Your manager viewed your activity today"). Transparency is a feature.

---

## 7. Security & Privacy

This is a **surveillance-adjacent tool**. Done wrong, it's a lawsuit and/or a morale disaster. Done right, it's a consented coaching tool. Production rules are non-negotiable:

### Non-negotiable defaults

1. **LAN-only binding.** Hub listens on private subnets only unless explicitly overridden. Loud warnings if you try.
2. **mTLS everywhere.** No plaintext, no optional "insecure mode" in prod builds.
3. **No keystroke content.** `uiohook-napi` reports *that* keys were pressed (for idle detection) — never *which* keys.
4. **No screenshots.** Period. Not a v1 feature, and if ever added must be opt-in per employee per session.
5. **Window-title redaction.** Ships with a sensible default blocklist (`/banking|health|medical|personal|1password/i`) and the Agent UI lets the employee add patterns. Redaction happens **in the Agent, before anything goes on the wire.**
6. **Pause button.** Tray menu: 30 min / 1 hr / until tomorrow. The *fact* of a pause is logged (visible to manager); the content during a pause is not recorded.
7. **Self-transparency.** Every employee has a Personal Dashboard that shows exactly what the manager sees about them. No one-way glass.
8. **Data retention.** Default 90 days for activity events, 1 year for check-ins and reports. Configurable. Cron purges anything older.
9. **Right to export.** Every employee can export their own full data (JSON + CSV) from the Agent, any time.
10. **Explicit pairing.** New devices need manager approval. No silent re-enrollment.
11. **Consent at install.** Granular, per-data-type checkboxes. Declining a data type disables that collection; Agent still works with reduced signal.
12. **Audit log is immutable + visible.** Managers can view team data, but every view is logged, and employees can read the log about themselves.

### Threat model (summarized)

| Threat | Mitigation |
|---|---|
| Malicious insider exfiltrates data | mTLS, per-device cert, audit log, no API keys |
| Evil-twin WiFi impersonates Hub | mTLS with pinned CA; Agent rejects unknown certs |
| Hub disk stolen | LUKS/FileVault required at install; Postgres data dir on encrypted volume |
| Credential stuffing on dashboard | Argon2id hashing, rate limit, optional OIDC/SSO |
| Compromised Agent leaks data | Per-device revocation, JWT expiry, cert rotation |
| Rogue manager stalks one person | Every view audit-logged; employee sees the log; admin role separate from manager role |
| Supply chain (npm) | Renovate + `npm audit` + CodeQL + signed releases |

### Compliance notes

- Ship a **Data Processing Impact Assessment (DPIA)** template in `docs/`. Orgs in GDPR jurisdictions will need one.
- State clearly that WorkPulse is a **consented productivity tool** — not covert surveillance. If your jurisdiction requires notice to employees before monitoring (most do), the consent screen satisfies it by design.
- Provide an "**employer posture**" knob in settings: Strict (minimum data), Balanced (default), Full (all signals enabled). Admins pick at install, can change later, every change is audit-logged.

---

## 8. Observability & Operations

### Logs

- All services emit **structured JSON** (Pino).
- Container logs collected by Docker's `json-file` driver by default; teams wanting centralized logs can switch to Loki (shipped as optional compose profile).
- Sensitive fields redacted at log-serialization time (never log window titles, check-in content, or auth tokens).

### Metrics (Prometheus)

- `workpulse_agents_connected` (gauge)
- `workpulse_activity_events_total` (counter, by user)
- `workpulse_ai_inference_duration_seconds` (histogram, by prompt type)
- `workpulse_ai_queue_depth` (gauge)
- `workpulse_db_query_duration_seconds` (histogram)
- `workpulse_checkin_response_rate` (gauge, daily)
- HTTP/WS standard RED metrics via Fastify plugin

Grafana ships with two pre-built dashboards: **Team Health** (manager-facing) and **Hub Health** (admin-facing).

### Health checks

- `GET /health/live` — process up
- `GET /health/ready` — Postgres + Redis + Ollama all responsive
- Docker healthchecks on every container

### Backups

- `infra/scripts/backup.sh` runs nightly (cron on the host):
  - `pg_dump --format=custom` → timestamped file in `/var/workpulse/backups/`
  - Prune > 30 days
  - Optional: rsync to a NAS on the LAN
- `restore.sh` is a single command; tested in CI against a seed database.
- Full disaster recovery documented in `docs/OPERATIONS.md` with RPO = 24h, RTO = 30 min targets for a typical team.

### Upgrades

- Hub: `git pull && docker compose pull && docker compose up -d` — Prisma migrations run automatically on boot; schema changes are always additive (no destructive migrations allowed in a minor release).
- Agents: auto-update via electron-updater, pulling from the Hub's `/updates` endpoint. Managers can stage rollouts ("5% canary → 100%").

### Monitoring the monitor

An admin-only "Hub Health" view in the dashboard shows CPU/mem/GPU, Postgres size, Ollama queue depth, agent connectivity map. If the Ollama queue backs up or disk fills, it nags you before it matters.

---

## 9. Scalability Envelope

Sizing guidance, with numbers:

| Team size | Hub spec | Notes |
|---|---|---|
| 1–10 | 4-core / 16 GB / integrated GPU or CPU-only | `mistral:7b` on CPU works, ~5–10s per check-in |
| 10–30 | 8-core / 32 GB / RTX 3060 (12 GB) | `llama3.1:8b` on GPU, sub-2s inference |
| 30–100 | 12-core / 64 GB / RTX 4070 or better | Same, with BullMQ concurrency bumped to 8 |
| 100+ | Multi-host: split Ollama + Postgres | Beyond one-box territory; see `docs/ARCHITECTURE.md` §scaling |

Bottlenecks, in order:
1. **Ollama inference throughput** — GPU-bound. Solved by bigger GPU or sharding to a second Ollama host on the LAN.
2. **Postgres writes on ActivityEvent** — solved by TimescaleDB hypertables + compression after 7 days.
3. **Socket.IO connections** — Node handles ~10k idle WS fine; not a concern below 1000 agents.

---

## 10. Deployment

### Dev (one command)

```bash
pnpm install
pnpm dev             # runs hub, dashboard, Postgres, Redis, Ollama via compose
pnpm agent:dev       # runs Electron agent pointing at local hub
```

### Prod (one command)

```bash
cd infra/compose
cp .env.example .env     # set ADMIN_EMAIL, TIMEZONE, MODEL, RETENTION_DAYS
docker compose up -d
```

The compose file brings up: `hub`, `dashboard` (as a reverse-proxied Next.js server), `postgres`, `redis`, `ollama`, `caddy`, `prometheus`, `grafana`. All services restart on failure and have healthchecks.

### Agent distribution

- Hub serves the latest installer at `https://workpulse.local/download/{platform}`.
- Signed with an Authenticode cert (Windows) and Apple Developer ID (macOS notarization). On Linux, `.AppImage` + `.deb` + `.rpm`, all GPG-signed.
- CA root and Hub fingerprint are baked in at build time per-deployment (there's a `scripts/package-client.sh` that embeds them).

---

## 11. Open-Source Strategy

### License

**AGPL-3.0.** Rationale:

- Monitoring tools with closed-source forks are a nightmare — you can't audit what you can't read. AGPL forces any SaaS fork to also publish source.
- Internal use inside a company is fully permitted; the AGPL trigger is only if you offer it as a service.
- If MIT is preferred for broader adoption, that's a reasonable alternative — I'd lean AGPL given the category.

### Repo hygiene

- `README.md` — 30-second pitch, architecture diagram, quick start, link to docs.
- `LICENSE` — AGPL-3.0 full text.
- `PRIVACY.md` — every data point collected, how to disable each.
- `SECURITY.md` — vulnerability disclosure (email + GPG key), supported versions.
- `CONTRIBUTING.md` — dev setup, PR checklist, commit convention (Conventional Commits).
- `CODE_OF_CONDUCT.md` — Contributor Covenant 2.1.
- `.github/ISSUE_TEMPLATE/` — bug, feature, privacy concern, security advisory.
- Pre-commit hooks via `lefthook` — Biome + typecheck.
- Docs site auto-deployed to GitHub Pages from `docs/`.
- Signed releases via GitHub Releases + SBOM (CycloneDX) attached.

### Governance

- Start as BDFL (you) — normal for year-one projects.
- Add `MAINTAINERS.md` once 2+ regular contributors exist.
- Move to a small steering committee once >10 active contributors.

---

## 12. Build Phases

**Phase 1 — Walking skeleton (3 weeks)**
- Monorepo scaffolded, CI green.
- `packages/protocol` + `packages/db` done first.
- Hub: Fastify + Prisma + one Socket.IO room + `/health`.
- Agent: Electron shell + pairing (static code for now) + activity snapshot every 60s.
- Docker Compose with Postgres + Redis + Ollama.
- **Success:** one agent pairs, streams activity for a day without dropping events.

**Phase 2 — The AI loop (2 weeks)**
- Ollama client + BullMQ inference queue.
- Morning / afternoon / evening check-ins with the three prompt templates.
- Per-user rolling context builder.
- Offline buffer on Agent + reconnect logic.
- **Success:** three employees go through a full day; AI check-ins reference actual observed activity.

**Phase 3 — Manager dashboard (2 weeks)**
- Next.js dashboard mounted on Hub at `/dashboard`.
- Team grid, live feed, member deep-dive.
- EOD team digest cron + prompt.
- Audit log surfaced on Personal Dashboard.
- **Success:** manager logs in, sees team at a glance, reads a generated EOD digest.

**Phase 4 — Productionization (3 weeks)**
- mTLS with CA rotation script.
- mDNS discovery + pairing codes (retire static codes).
- Prometheus + Grafana dashboards.
- Backup/restore scripts with CI-tested restores.
- Redaction rules UI + pause button.
- Data export (employee self-service).
- Signed installers for Win/macOS/Linux + Hub-hosted update feed.
- `PRIVACY.md`, `SECURITY.md`, `OPERATIONS.md`.
- **Success:** v1.0 tag. Safe to install in a real 10-person team.

**Phase 5 — Scale & polish**
- TimescaleDB hypertables + compression policies.
- Load tests (k6 @ 100 agents).
- Multi-team support.
- Optional OIDC/SSO for dashboard.
- Optional LAN-SMTP for email digests.
- Anomaly detection (private to employee by default — e.g. "you've been in Slack 90 min, want to refocus?").

---

## 13. What To Build First (Today)

If you want to start coding right now, this is the order:

1. **`packages/protocol`** — Zod schemas for `ActivityEvent`, `CheckIn`, `User`, and the Socket.IO event union type. This is the contract; everything else depends on it.
2. **`packages/db`** — Prisma schema + first migration. Generate the client.
3. **`apps/hub`** — Fastify boot + Socket.IO + `POST /pairing` + one `activity:snapshot` handler that writes to Postgres. Stop when you can `curl` an event in and see it in the DB.
4. **`apps/agent`** — Electron shell + `active-win` loop + hardcoded pairing. Stream to the Hub you just built.
5. Only then: queue, check-ins, dashboard, mDNS, mTLS.

Say the word and I can scaffold `packages/protocol` + `apps/hub` with Fastify, Prisma, Socket.IO, and a working `activity:snapshot` flow — enough to have a runnable Hub by end of day.
