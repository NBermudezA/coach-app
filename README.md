# coach-app

Entrenador personal con IA: genera el plan semanal de entrenamiento, registra cada sesión, sigue el progreso y estima los macros de las comidas. Uso personal (máx. ~5 usuarios, iPhone), pero construido para poder publicarse más adelante.

**Principio:** el código calcula, la IA decide. Los números (progresión de carga, volumen, TDEE, macros) salen de código determinista y testeado; Claude toma decisiones de coaching sobre esos números y siempre responde JSON validado con Zod.

> **Estado:** Fase 0 — Fundamentos. Todavía no hay código; ver [docs/todos.md](docs/todos.md).

## Funcionalidades

| Módulo | Qué hace |
|---|---|
| Perfil y objetivos | Peso actual/objetivo, bulk / cut / recomp, días disponibles, equipamiento, lesiones |
| Plan semanal | La IA genera la rutina de la semana como JSON estructurado (ejercicio, series, reps, RPE, descanso) |
| Sesión de hoy | Ejercicios del día, registro de peso/reps/RPE, saltar o cambiar ejercicios, temporizador de descanso |
| Progreso | Peso corporal, carga por ejercicio, volumen semanal, adherencia |
| Comidas | Texto libre → macros estimados → lo que queda del día (±15–20 %) |
| Notificaciones | Rutina por la mañana; recordatorio por la tarde si no se registró nada |

## Stack

| Capa | Tecnología |
|---|---|
| Monorepo | Turborepo + Yarn 4 (`nodeLinker: node-modules`) |
| App móvil | Expo, Expo Router, NativeWind, react-native-reusables, TanStack Query (offline-first) |
| API | Hono en AWS Lambda (Function URL) |
| Base de datos | Aurora Serverless v2 PostgreSQL (escala a cero, Data API) + Drizzle ORM |
| Auth | Better Auth |
| IA | Vercel AI SDK + Claude (Sonnet para planes, Haiku para mensajes diarios y comidas) |
| Validación | Zod, compartido entre app, API e IA |
| Infra | Terraform (DB, IAM, SSM) + SST (API, cron, secretos) en `us-east-1` |

Detalle y diagramas: [docs/architecture.md](docs/architecture.md).

## Estructura del repo

```
coach-app/
├── apps/
│   ├── mobile/        # App Expo (iOS, Android, web)
│   └── api/           # Hono en Lambda
├── packages/
│   ├── core/          # lógica de dominio pura + schemas Zod (con tests)
│   ├── db/            # schema Drizzle + migraciones
│   ├── ai/            # prompts + llamadas a Claude
│   └── functions/     # handlers del cron
├── infra/terraform/   # Aurora, IAM, SSM
├── docs/              # arquitectura, todos, ideas
└── sst.config.ts
```

*(Las carpetas se crean a lo largo de la fase 0.)*

## Requisitos

- Node 22 LTS y Yarn 4 (vía Corepack)
- Terraform ≥ 1.10
- AWS CLI v2 con el perfil `coach` configurado (IAM Identity Center)
- Xcode + simulador de iOS

## Puesta en marcha

*Se completará cuando exista el scaffold del monorepo (fase 0.2).*

## Flujo de trabajo

- Rama por defecto: **`dev`**. Producción: **`prod`**. `main` no se usa.
- Cada tarea en su rama (`feat/...`, `fix/...`, `docs/...`) → PR a `dev` → PR de `dev` a `prod` cuando está estable.
- Commits en inglés con [Conventional Commits](https://www.conventionalcommits.org/) (`feat(api): add health endpoint`).
- Cada PR actualiza `docs/todos.md` y, si aplica, este README y `docs/architecture.md`.
- Los secretos nunca van a git: SSM o `sst secret set`.

## Documentación

| Documento | Contenido |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Componentes, infraestructura, entornos, seguridad, costes y decisiones |
| [docs/todos.md](docs/todos.md) | Blueprint completo: todo lo que falta, por fase |
| [docs/ideas.md](docs/ideas.md) | Ideas nuevas que aún no están en el roadmap |
| [CLAUDE.md](CLAUDE.md) | Contexto y reglas de trabajo para Claude Code |

## Costes objetivo

~7–15 USD/mes (Aurora ~2–5, Claude ~5–10, resto en la capa gratuita). Apple Developer (99 USD/año) solo en la fase 4.

## Licencia

[MIT](LICENSE)
