# Highway Operations Command (HOC)

A GIS-enabled, mobile-first field operations platform for highway maintenance contracts. HOC replaces WhatsApp-and-paper coordination across a ~200 km national-highway contract with a single, auditable system of record — every incident geo-tagged, every work order tracked against an SLA, every closure backed by photo evidence.

> **Status:** Phase 1 — Foundation. The monorepo skeleton is being set up under issue [F-001](https://github.com/pupli/highway-operation-command/issues/1). Most apps and services below are scaffolded but not yet implemented.

---

## Documentation

| Document | What it covers |
|---|---|
| [`HOC-Project-Brief.md`](./HOC-Project-Brief.md) · [HTML](./HOC-Project-Brief.html) | Operational model, data model, architecture, roadmap — the client-facing solution blueprint. |
| [`HOC-Engineering-Playbook.html`](./HOC-Engineering-Playbook.html) | Feature board (53 features across 3 phases) + engineering practices: branching, commits, PRs, code review, Definition of Done. **Read this before opening a PR.** |
| [`initial-understanding/`](./initial-understanding/) | The original requirement and design prompts that produced the brief. |

---

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| **Node.js** | 20 LTS | [nodejs.org](https://nodejs.org) or via `nvm install` (we ship a `.nvmrc`) |
| **pnpm** | 9.x | `corepack enable && corepack prepare pnpm@latest --activate` |
| **Docker** | latest | for local Postgres + Redis + S3-compatible storage |
| **Git** | 2.40+ | with a GitHub account that has access to this repo |

For mobile development you'll additionally need:
- **Xcode 15+** (iOS, macOS only)
- **Android Studio** with an emulator image, or a physical device
- **Expo CLI** is installed automatically as part of `pnpm install`

---

## Quick start

```bash
# 1. Clone
git clone git@github.com:pupli/highway-operation-command.git
cd highway-operation-command

# 2. Install dependencies (this also installs Husky hooks)
pnpm install

# 3. Bring up local infra (Postgres + PostGIS, Redis, MinIO)
docker compose -f infra/docker/compose.yml up -d

# 4. Run database migrations and seed sample highway data
pnpm db:migrate
pnpm db:seed

# 5. Start everything
pnpm dev
```

After step 5 you should have:

| URL | Service |
|---|---|
| http://localhost:3000 | Web dashboard (Next.js) |
| http://localhost:4000 | API gateway |
| http://localhost:8081 | Expo dev server (scan QR with Expo Go) |

---

## Repository layout

```
highway-operation-command/
├── apps/
│   ├── mobile/                 # React Native + Expo — Engineer, Supervisor, Worker, QA apps
│   └── web/                    # Next.js — Owner dashboard, NHAI read-only view
│
├── services/                   # Modular monolith — one folder per bounded context
│   ├── api-gateway/            # Auth, rate limit, request log
│   ├── incident-service/       # Incident capture, triage, lifecycle
│   ├── work-order-service/     # Work order state machine, SLA hooks
│   ├── workforce-service/      # Engineers, supervisors, workers, contractors
│   ├── evidence-service/       # Photo uploads, EXIF + GPS integrity
│   ├── geo-service/            # Highway segments, chainage, assets
│   ├── sla-service/            # SLA timers, escalation engine
│   ├── notification-service/   # FCM push, SMS (Twilio / MSG91)
│   └── reporting-service/      # Daily / weekly / monthly NHAI reports
│
├── packages/                   # Shared libraries
│   ├── domain-types/           # TypeScript types shared across apps & services
│   ├── api-client/             # SDK generated from the OpenAPI specs
│   ├── ui-kit/                 # Shared design system (web + mobile)
│   ├── geo-utils/              # Chainage, snap-to-segment, distance math
│   ├── tsconfig/               # Shared TypeScript configs
│   └── eslint-config/          # Shared lint rules
│
├── infra/
│   ├── terraform/              # AWS infra as code (ap-south-1)
│   └── docker/                 # Local dev compose stack
│
├── docs/
│   ├── api/                    # OpenAPI specs per service
│   └── runbooks/               # Ops procedures
│
└── tools/
    ├── seed/                   # Sample highway segment data
    └── scripts/                # One-off CLI helpers
```

Service-boundary rule: **each service owns its own database tables**. Cross-service data goes via the service's API or domain events — never via a JOIN. See [§K of the playbook](./HOC-Engineering-Playbook.html#data) for details.

---

## Common commands

All commands run from the repo root and use Turborepo to dispatch across the workspace.

| Command | What it does |
|---|---|
| `pnpm dev` | Start every app and service in watch mode |
| `pnpm dev --filter=web` | Start just the web dashboard |
| `pnpm dev --filter=mobile` | Start just the mobile app (Expo) |
| `pnpm dev --filter=incident-service` | Start a single service |
| `pnpm build` | Build every package |
| `pnpm lint` | ESLint across the workspace (zero-warnings policy) |
| `pnpm type-check` | `tsc --noEmit` across the workspace |
| `pnpm test` | Unit + integration tests |
| `pnpm test:e2e` | Playwright (web) + Detox/Maestro (mobile) — slower |
| `pnpm db:migrate` | Apply pending migrations |
| `pnpm db:rollback` | Roll back the last migration |
| `pnpm db:seed` | Seed sample highway, engineers, work orders |

Most are wrappers around `pnpm turbo run <task>` — use the raw form if you want Turborepo's flags (`--filter`, `--continue`, etc.).

---

## Workflow

We follow a trunk-based workflow with short-lived feature branches and squash merges. The full process is in the [Engineering Playbook](./HOC-Engineering-Playbook.html#workflow), but the short version:

1. **Pick** an open feature on the [board](./HOC-Engineering-Playbook.html#features).
2. **Claim** it in the team channel.
3. **Branch** off `main`: `feat/F-XXX-short-slug`.
4. **Commit** in [Conventional Commits](./HOC-Engineering-Playbook.html#commits) style, small and often.
5. **Open a draft PR** early; mark it ready when CI is green.
6. **Get one review**, address comments, squash-merge to `main`.
7. **Delete the branch** and mark the feature done.

`main` is protected — every change goes through a PR. CI runs lint + type-check + tests on every PR and must pass before merge.

---

## Tech stack at a glance

| Layer | Choice |
|---|---|
| Mobile | React Native + Expo |
| Web | Next.js (React) + TypeScript |
| Maps | MapLibre GL + OpenStreetMap |
| Backend | Node.js + NestJS, modular monolith |
| Database | PostgreSQL + PostGIS |
| Object storage | S3-compatible (AWS S3 / Cloudflare R2; MinIO locally) |
| Cache / queues | Redis (BullMQ) |
| Auth | Keycloak or Auth0, role-based |
| Notifications | FCM push + Twilio / MSG91 SMS |
| Hosting | AWS Mumbai (`ap-south-1`) |
| Observability | OpenTelemetry → Grafana / Loki / Tempo |

Rationale and architecture diagram are in [§7 of the brief](./HOC-Project-Brief.md#7-technical-architecture).

---

## Contributing

This repo is for the HOC build team (2–3 engineers). Read these before your first PR:

- [Branching rules](./HOC-Engineering-Playbook.html#branching)
- [Commit format](./HOC-Engineering-Playbook.html#commits)
- [PR template & lifecycle](./HOC-Engineering-Playbook.html#prs)
- [Code review etiquette](./HOC-Engineering-Playbook.html#review)
- [Definition of Done](./HOC-Engineering-Playbook.html#dod)

If something in the playbook is wrong or unclear, open a `docs/` PR — it's a living document.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `pnpm install` complains about Node version | Wrong Node | `nvm use` (we ship a `.nvmrc`) |
| Husky hooks don't run | Hooks not installed | `pnpm install` runs `husky install`; if that didn't fire, run `pnpm prepare` |
| `pnpm dev` can't reach DB | Docker stack down | `docker compose -f infra/docker/compose.yml up -d` |
| Expo can't find your phone | Phone on different network | Use a USB tunnel: `pnpm dev --filter=mobile -- --tunnel` |
| Pre-commit hook is wrong about a secret | False positive in scanner | Once-off override: `HUSKY=0 git commit -m "..."` — then tell the team |

---

## License

Proprietary. © Highway Operations Command contributors. Internal use only — do not redistribute.
