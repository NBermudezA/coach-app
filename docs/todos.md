# TODOs — blueprint del proyecto

> **Fuente de verdad** del trabajo pendiente. Se actualiza en cada PR: se marcan las tareas hechas y se agregan las nuevas.
> Cada casilla es un paso pequeño = un commit (Conventional Commits).
> La sección 8 de `CLAUDE.md` solo resume las fases y apunta aquí.

Leyenda: `[ ]` pendiente · `[x]` hecho · 🔒 requiere aprobación explícita (deploy, AWS, push) · ❓ decisión pendiente

---

## Decisiones pendientes ❓

- [ ] **Cuenta AWS:** ¿cuenta existente (capa gratuita antigua, previa a julio de 2025) o nueva (modelo de créditos)?
- [ ] **Stages:** ¿desplegamos un stage `dev` además de `prod`? Propuesta: sí, pero con su propio cluster Aurora pausado (coste ≈ solo almacenamiento). Alternativa: solo `prod` + `sst dev` en local contra la DB de prod (más barato, más riesgo).
- [ ] **Estructura Terraform por stage:** carpetas `infra/terraform/envs/{dev,prod}` con un módulo compartido, o workspaces de Terraform. Propuesta: carpetas (más explícito).
- [ ] **Versión de Node:** hoy hay Node 20 instalado. Propuesta: fijar Node 22 LTS con `.nvmrc` y `engines`.
- [ ] **Modelo de datos** completo (se diseña juntos al empezar la fase 1).

---

## Fase 0 — Fundamentos (~1 semana) ← ACTUAL

**Hecho cuando:** `curl https://<api>/health` devuelve datos de Aurora.

### 0.1 Repositorio y documentación
- [x] Crear repo en GitHub (`coach-app`)
- [x] Ramas `dev` (por defecto) y `prod`; eliminar `main`
- [x] `docs/`: arquitectura, todos, ideas; README (PR #1)

### 0.2 Scaffold del monorepo
- [ ] Instalar herramientas locales: Node 22 LTS, Terraform ≥ 1.10 (no instalado), SST CLI (vía paquete)
- [ ] `package.json` raíz con Yarn 4 (`packageManager`), workspaces `apps/*` y `packages/*`
- [ ] `.yarnrc.yml` con `nodeLinker: node-modules`
- [ ] `.gitignore` (node_modules, `.turbo`, `.sst`, `*.tfstate*`, `.terraform/`, `.env*`, builds de Expo)
- [ ] `turbo.json` con tareas `build`, `dev`, `lint`, `typecheck`, `test`
- [ ] `tsconfig` base compartido
- [ ] Lint + formato (propuesta: Biome, una sola herramienta) y `.editorconfig`
- [ ] Vitest configurado para los paquetes
- [ ] Paquetes vacíos `@coach/core`, `@coach/db`, `@coach/ai`, `@coach/functions` que compilan

### 0.3 Cuenta AWS 🔒
- [ ] Root: MFA activado, sin claves de acceso
- [ ] IAM Identity Center: usuario admin + permission set
- [ ] Perfil local `coach` (`aws configure sso`), región `us-east-1`
- [ ] AWS Budgets: alerta a 10 USD
- [ ] Solicitar créditos de onboarding (si aplica)

### 0.4 Terraform bootstrap 🔒
- [ ] `infra/terraform/bootstrap`: bucket S3 de estado (versionado, cifrado, acceso público bloqueado)
- [ ] Backend S3 con `use_lockfile = true` para el resto de la configuración

### 0.5 Terraform: base de datos 🔒
- [ ] Cluster Aurora Serverless v2 PostgreSQL (`coach-{stage}-aurora`), `min_capacity = 0`, `max_capacity` bajo, Data API activada
- [ ] Credenciales gestionadas en Secrets Manager
- [ ] Rol/política IAM para acceso por Data API
- [ ] Parámetros SSM `/coach/{stage}/db/{cluster-arn,secret-arn,name}`
- [ ] `terraform plan` revisado → `apply` aprobado

### 0.6 `packages/db`
- [ ] Drizzle con `drizzle-orm/aws-data-api/pg` + cliente RDS Data
- [ ] Wrapper con reintentos y backoff para la reanudación de Aurora
- [ ] `drizzle.config.ts` (driver aws-data-api) y tabla de prueba
- [ ] Primera migración aplicada 🔒

### 0.7 `apps/api`
- [ ] Hono + adaptador `hono/aws-lambda`
- [ ] `GET /health` que consulta la DB (p. ej. `select now()`)
- [ ] Test del handler

### 0.8 SST 🔒
- [ ] `sst.config.ts` (app `coach`, región `us-east-1`, perfil `coach`)
- [ ] Leer parámetros SSM de Terraform
- [ ] Función de la API con Function URL y permisos Data API mínimos
- [ ] `sst deploy --stage prod` aprobado → `curl /health` OK ✅

---

## Fase 1 — Datos y auth (~1 semana)

**Hecho cuando:** el login funciona en el simulador de iOS.

- [ ] Sesión de diseño del modelo de datos (documentarlo en `architecture.md`)
- [ ] Schemas Drizzle: `users`, `goals`, `body_metrics`, `exercises`, `training_plans`, `planned_workouts`, `planned_sets`, `workout_logs`, `set_logs`, `meals`
- [ ] Schemas Zod equivalentes en `@coach/core`
- [ ] Migraciones aplicadas 🔒
- [ ] Seed del catálogo de ejercicios (equipamiento del home gym)
- [ ] Better Auth en Hono (adaptador Drizzle, `/api/auth/*`), secreto en SST Secrets 🔒
- [ ] Middleware de sesión para rutas protegidas
- [ ] Scaffold de `apps/mobile`: Expo + Expo Router + NativeWind + react-native-reusables
- [ ] Verificar que Expo funciona en el monorepo (Metro + Yarn node-modules)
- [ ] Cliente Better Auth Expo (SecureStore)
- [ ] Pantallas de registro / login / logout
- [ ] Cliente API tipado (compartiendo schemas Zod)

---

## Fase 2 — Núcleo de entrenamiento (~2 semanas)

**Hecho cuando:** se entrena una semana completa usando solo la app (MVP).

### Lógica (`@coach/core`, con tests primero)
- [ ] Doble progresión de carga (`calculateNextLoad`)
- [ ] Volumen semanal por grupo muscular
- [ ] Adherencia (planificado vs realizado)
- [ ] TDEE, objetivo calórico y macros según objetivo (bulk / cut / recomp)
- [ ] Schemas del plan semanal (ejercicio, series, reps, RPE, descanso)

### IA (`@coach/ai`)
- [ ] Setup Vercel AI SDK + proveedor Anthropic; clave en SST Secrets 🔒
- [ ] Prompt de plan semanal (Sonnet) con `generateObject` + validación Zod + reintento
- [ ] Tests con respuestas grabadas (sin llamar a la API en CI)

### API
- [ ] Endpoints de perfil / objetivos
- [ ] `POST` generar plan semanal, `GET` plan de la semana y sesión de hoy
- [ ] Endpoints de registro de series y peso corporal (idempotentes, aptos para la cola offline)

### App
- [ ] Onboarding: peso actual/objetivo, tipo de objetivo, días, equipamiento, lesiones, preferencias
- [ ] Pantalla "Hoy": ejercicios planificados, registrar peso/reps/RPE, saltar o cambiar ejercicio
- [ ] Temporizador de descanso fiable (sigue en segundo plano; notificación al terminar)
- [ ] Offline-first: cache persistido + mutaciones en cola + reintentos
- [ ] Registro de peso corporal
- [ ] Vista del plan semanal

---

## Fase 3 — El coach (~1–2 semanas)

**Hecho cuando:** la rutina diaria llega por la mañana sin abrir la app.

- [ ] Cron diario en SST 🔒: despertar Aurora → revisar plan → mensaje diario (Haiku)
- [ ] Notificaciones locales: rutina por la mañana, recordatorio por la tarde si no hay registro
- [ ] Auto-ajuste del plan (Sonnet) a partir de adherencia/progresión pre-calculadas
- [ ] Comidas: texto libre → macros estimados (Haiku) → lo que queda del día
- [ ] Revisión semanal

---

## Fase 4 — Uso real (~1 semana)

- [ ] Pagar Apple Developer Program (99 USD/año)
- [ ] EAS Build + TestFlight
- [ ] Invitar al hermano
- [ ] Gráficos de progreso (peso corporal, carga por ejercicio, volumen, adherencia)
- [ ] Push desde servidor (Expo push)

---

## Fase 5 — Extras

- [ ] HealthKit (datos de Garmin vía Apple Health)
- [ ] Chat libre con el coach
- [ ] Strava
- [ ] Versión web
