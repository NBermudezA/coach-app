# coach-app — Project Context

AI-powered personal training app. Built for personal use by Nico (and possibly his brother), max ~5 users. Not intended for App Store release for now, but architected so that publishing later is possible.

Repo: `github.com/NBermudezA/coach-app` (monorepo)

---

## 1. Product scope

| Module | Description |
|---|---|
| Profile & goals | Current/target weight, goal type (bulk / cut / recomp), available days, equipment, injuries, preferences |
| Weekly plan | AI generates the week's routine as **structured JSON** (exercise, sets, reps, RPE, rest) — never free text |
| Today's session | Shows planned exercises; user logs actual weight/reps/RPE, skipped or swapped exercises; rest timer |
| Progress | Body weight, load per exercise, weekly volume, adherence |
| Meals | Free-text meal entry ("2 eggs, toast with avocado") → AI estimates macros → tells what's left for the day. Approximate by design (±15–20% is acceptable) |
| Notifications | Daily routine every morning; evening reminder if nothing logged |

Owner context: home gym (weights, squat rack, cable pulley, stationary bike, resistance bands), push-pull-legs split, also runs; uses a Garmin watch. iPhone users.

---

## 2. Core design principle

**Code calculates, AI decides.**

- Deterministic, tested code in `packages/core`: load progression (e.g. double progression), weekly volume, TDEE, calories, macros.
- The LLM receives pre-computed numbers + history and makes coaching decisions (e.g. "you skipped legs twice, reduce volume and move the session to Saturday") and writes human-readable messages.
- All AI outputs are structured and validated with Zod.

---

## 3. Stack (decided)

| Layer | Choice | Notes |
|---|---|---|
| Monorepo | Turborepo + Yarn 4 workspaces | `nodeLinker: node-modules` (PnP breaks Expo/RN) |
| Mobile app | Expo + Expo Router + NativeWind + react-native-reusables | Native from day 1 (offline logging, reliable rest timer, HealthKit later). Expo can also output web |
| API | Hono on AWS Lambda | Deployed with SST |
| Validation | Zod, shared across app / API / AI | |
| Database | Aurora Serverless v2 (PostgreSQL) with **scale-to-zero** + **Data API** | Lambdas run **outside the VPC** → no NAT, no public IPv4 costs |
| ORM | Drizzle (`drizzle-orm/aws-data-api/pg`) + drizzle-kit migrations | |
| Auth | Better Auth (Drizzle adapter, Expo client) | Not Cognito |
| AI | Vercel AI SDK + Claude | Sonnet: weekly plan generation/review. Haiku: daily message, meal parsing |
| Offline-first | TanStack Query with persisted cache + queued mutations | Hides Aurora cold-start (~15s resume) from the user |
| Notifications | Local notifications scheduled by the app (main path); Expo push from server later | Local notifications work without paid Apple account |
| Infra | **Hybrid: Terraform + SST** | See section 4 |
| Region | `us-east-1` | Cheapest, all services available |

---

## 4. Infrastructure split (Terraform + SST)

**Terraform (`infra/terraform/`) — the stable base:**
- Aurora Serverless v2 cluster (`min_capacity = 0`, Data API enabled)
- DB credentials secret
- IAM roles
- SSM Parameter Store outputs (e.g. `/coach/prod/...`) consumed by SST
- State in S3 with native locking (`use_lockfile = true`, Terraform ≥ 1.10). No DynamoDB lock table.

**SST (`sst.config.ts`) — the app layer:**
- Hono API Lambda (function URL)
- Daily cron (EventBridge Scheduler) for plan review + daily message
- Secrets (Anthropic API key, etc.)
- Reads Terraform outputs from SSM

Rule: each tool owns its own resources; they never manage the same thing.

---

## 5. Repo structure

```
coach-app/
├── apps/
│   ├── mobile/           # Expo app (iOS, Android, web)
│   └── api/              # Hono on Lambda
├── packages/
│   ├── core/             # pure domain logic + Zod schemas (tested)
│   ├── db/               # Drizzle schema + migrations
│   ├── ai/               # prompts + Claude calls
│   └── functions/        # cron handlers
├── infra/
│   └── terraform/        # Aurora, IAM, SSM
├── docs/                 # architecture.md, todos.md, ideas.md
├── sst.config.ts
└── CLAUDE.md
```

---

## 6. Conventions

| What | Convention | Example |
|---|---|---|
| Internal packages | `@coach/*` | `@coach/core`, `@coach/db` |
| DB tables/columns | `snake_case`, plural tables | `workout_logs.performed_at` |
| TS code | `camelCase`; types `PascalCase` | `calculateNextLoad()`, `WorkoutLog` |
| Files | `kebab-case` | `daily-plan.ts` |
| Zod schemas | `Schema` suffix | `workoutLogSchema` |
| AWS resources | `coach-{stage}-{resource}` | `coach-prod-aurora` |
| Commits | Conventional Commits | `feat(api): add health endpoint` |

Secrets never go in git (SSM / SST Secrets only). `*.tfstate` and `.terraform/` are gitignored.

---

## 7. Costs (targets)

| Item | USD/month |
|---|---|
| Aurora Serverless v2 (~1–2 h active/day) | ~2–5 |
| Lambda, EventBridge, SSM | ~0 (always-free) |
| NAT / public IPv4 | 0 (not used) |
| Claude API | ~5–10 |
| **Total** | **~7–15** (less while AWS credits last) |
| Apple Developer Program | US$99/year — **pay only at Phase 4** |

Set an AWS Budgets alert at US$10 on day 1.

---

## 8. Roadmap

Detailed checklist (source of truth): **`docs/todos.md`**. Update its checkboxes after each step. This section only summarizes the phases.

| Phase | Goal | Done when |
|---|---|---|
| 0 — Foundations ← CURRENT | Monorepo, AWS account, Terraform (Aurora + SSM), `packages/db`, Hono `/health`, SST deploy | `curl https://<api>/health` returns data from Aurora |
| 1 — Data & auth | Data model (design together first), migrations, Better Auth, Expo scaffold, login | Login works in the iOS simulator |
| 2 — Training core | Onboarding, `packages/core` logic + tests, AI weekly plan (Sonnet), Today screen + rest timer, offline-first, body weight | A full week trained using only the app (MVP) |
| 3 — The coach | Daily cron + local notifications, plan auto-adjustment, meals (Haiku), weekly review | Daily routine arrives without opening the app |
| 4 — Real use | Apple Developer, EAS Build + TestFlight, brother invited, progress charts, server push | — |
| 5 — Extras | HealthKit (Garmin), coach chat, Strava, web | — |

---

## 9. Open items
See "Decisiones pendientes" in `docs/todos.md`.

## 10. Known gotchas
- Yarn PnP breaks Expo → keep `nodeLinker: node-modules`.
- Aurora resume from zero takes ~15s; Data API calls may fail during resume → retry with backoff; cron wakes DB a couple of minutes before generating plans.
- Free Apple ID: app installs expire after 7 days and no push entitlement → develop with simulator/Expo Go, pay only when MVP is ready.
- Never commit Terraform state (contains secrets).

---

## 11. Working agreements (for Claude)

Global rules (ask before push/deploy/infra changes, git branching, Conventional Commits, languages, no secrets in git) live in `~/.claude/CLAUDE.md`. Project-specific additions:

- "Ask before" also covers `terraform apply/destroy` and `sst deploy/remove`: show the `terraform plan` / diff first.
- Use AWS profile `coach` and region `us-east-1`. Never use root credentials.
- Secrets go to SSM or `sst secret set`.
- Work one checkbox of `docs/todos.md` at a time, commit after each, and update the checkboxes when done.
- Prefer the decisions in this file. If something here seems wrong, raise it before changing it; don't silently switch tools or patterns.
- Explain new concepts briefly (Terraform, SST, Hono, Expo are new to the owner).

---

## 12. Docs workflow

Every PR keeps docs current:
- `docs/todos.md`: mark done tasks, add new ones.
- `docs/ideas.md`: log any new idea that comes up (owner's or Claude's), always.
- `README.md`: update when setup, commands, structure or status change.
- `docs/architecture.md`: update when an architectural decision changes (add a row to its decision log).
