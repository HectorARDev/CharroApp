# CharroApp — Scoring System: Implementation Plan

> Este documento es el plan operativo detallado, para consumo de agentes/LLM (o para
> ejecutar sin ambigüedad): sub-fases, archivos exactos, SQL literal, criterios de
> aceptación. La versión para lectura humana (decisiones y su porqué, sin el detalle de
> ejecución) es `plan.md`. Al editar uno, sincronizar el otro.

## PROJECT
- **Name**: CharroApp
- **Domain**: Real-time sports scoring for charrería (Mexican rodeo)
- **Users**: Judges (field, tablet/phone), Admins (web), Public (display screen)
- **Critical constraint**: Offline-first — venues have unreliable connectivity
- **Platforms**: Web browser + iOS native + Android native (single codebase)
- **Scope**: All 9 official suertes (Federación Mexicana de Charrería)

---

## TECH STACK

```
Layer              Package                        Version           Purpose
────────────────────────────────────────────────────────────────────────────────────────
Build              vite                           última mayor      Bundler
Build              vite-plugin-pwa                latest            PWA + Workbox SW (solo app-shell, ver nota abajo)
Language           typescript                     última mayor      Type safety
UI framework       react                          18 o 19           Component model
Mobile wrapper     @capacitor/core                última mayor      iOS/Android native
Mobile wrapper     @capacitor/cli                 última mayor      CLI build tool
Mobile storage     @capacitor/preferences         última mayor      Key-value config
Mobile network     @capacitor/network             última mayor      Detección de conectividad confiable en nativo (navigator.onLine no es confiable en WebView iOS/Android — ver Fase 0.5.2)
Styling            tailwindcss                    v4                Utility CSS
UI components      shadcn/ui                      latest            Accessible components
State              zustand                        v5                Global state
Data fetching      @tanstack/react-query          v5                Server state + cache (lecturas no-críticas, ver módulo sync)
Offline DB         dexie                          última mayor      IndexedDB ORM (TS-first)
Backend            @supabase/supabase-js          ^2                PostgreSQL + Realtime + Auth
Form               react-hook-form + zod          latest            Validation
Routing            react-router                   última mayor      Client-side routing (/calificar/:id, /display/:torneoId)
Testing            vitest + @testing-library/react latest           Unit / component tests
Testing            @playwright/test               latest            E2E — incluye simulación de offline (setOffline)
Lint/format        eslint + prettier              latest            Calidad de código
```

> Nota: se listan versiones como "última mayor estable" en vez de pines fijos porque el
> stack original tenía versiones que ya podrían estar desactualizadas para un proyecto que
> arranca hoy; fijar el número exacto se hace al correr `npm install` en Fase 0.
>
> Ver `## ALTERNATIVAS CONSIDERADAS` al final del documento para el razonamiento detrás de
> cada elección y qué alternativas se descartaron y por qué.

---

## ARCHITECTURE

### Data flow
```
[Judge device]
  Dexie (IndexedDB) ──sync-on-connect──▶ Supabase PostgreSQL
                                               │
                                    Supabase Realtime channels
                                          ┌────┴────┐
                                    [Admin web]  [Public display /display]
```

### Offline-first sync protocol
1. Judge scores → write to Dexie immediately (`synced: false`, `updated_at: now`)
2. Listen: `window.addEventListener('online')` + `supabase.channel().on('system', {event: 'connected'})`
3. On connect: query `Dexie.calificaciones.where('synced').equals(0).toArray()`
4. Upsert to Supabase: `supabase.from('calificaciones').upsert(records, {onConflict: 'id'})`
5. Conflict resolution: `last_write_wins` via `updated_at` timestamp (sufficient — each judge owns their own rows via RLS)
6. On success: mark records `synced: true` in Dexie
7. Supabase Realtime broadcasts insert/update → Admin + Display receive via `supabase.channel('calificaciones').on('postgres_changes', ...)`

**Nota — alcance del Service Worker (Workbox)**: `vite-plugin-pwa` solo sirve al despliegue
*web* (juez/admin usando el navegador sin instalar la app nativa) — el build de Capacitor
carga los assets empaquetados localmente y no depende del SW. El Service Worker debe
precachear únicamente el app-shell (JS/CSS/HTML/estáticos); **no debe** interceptar ni
cachear llamadas a Supabase — ese trabajo ya lo hace Dexie. Mezclar ambas capas de cacheo
es la causa más común de bugs de "datos viejos" en apps offline-first.

---

## DATABASE SCHEMA (Supabase / PostgreSQL)

```sql
-- torneos
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
nombre      text NOT NULL
fecha       date NOT NULL
lugar       text
estado      text CHECK (estado IN ('borrador','activo','finalizado')) DEFAULT 'borrador'
created_at  timestamptz DEFAULT now()
-- constraint: solo un torneo activo a la vez (evita ambigüedad en RLS pública, SelectJuez, /display)
create unique index one_active_torneo on torneos (estado) where estado = 'activo';

-- equipos
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
torneo_id   uuid REFERENCES torneos(id) ON DELETE CASCADE
nombre      text NOT NULL
numero      int

-- charros
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
equipo_id   uuid REFERENCES equipos(id) ON DELETE CASCADE
nombre      text NOT NULL
numero      int

-- participaciones
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
torneo_id   uuid REFERENCES torneos(id)
charro_id   uuid REFERENCES charros(id)   -- NULL if team suerte
equipo_id   uuid REFERENCES equipos(id)   -- NULL if individual suerte
suerte      text NOT NULL  -- enum values below
orden       int            -- turn order within suerte
estado      text CHECK (estado IN ('pendiente','en_curso','completado')) DEFAULT 'pendiente'

-- jueces (sin columna pin — el PIN ES el password real de auth.users, ver módulo auth)
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()  -- = auth.users.id
torneo_id   uuid REFERENCES torneos(id)
nombre      text NOT NULL
rol         text CHECK (rol IN ('admin','juez')) DEFAULT 'juez'

-- calificaciones
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
participacion_id  uuid REFERENCES participaciones(id)
juez_id     uuid REFERENCES jueces(id)
puntaje     numeric(5,1) NOT NULL
detalle     jsonb                 -- NULL salvo terna_en_ruedo: {"piales": n, "derribo": n, "floreo": n}
notas       text
updated_at  timestamptz NOT NULL  -- used for conflict resolution
synced_at   timestamptz           -- NULL = not synced to server yet
-- RLS: juez can only INSERT/UPDATE WHERE juez_id = auth.uid()
-- RLS: admin can SELECT/UPDATE all
-- RLS: lectura pública (sin auth) en torneos/participaciones/calificaciones del torneo activo — requerido por /display
-- RLS: lectura pública (sin auth) en jueces del torneo activo con rol='juez' — requerido por SelectJuez.tsx (nunca expone cuentas admin)
```

---

## SUERTES — SCORING RULES

```
Suerte                  Type        Range   Step  Timer?  Fase  Notes
────────────────────────────────────────────────────────────────────────────────
cala_de_caballo         individual  0–10    0.5   No      1     Horse maneuver quality — MVP suerte
piales_en_lienzo        individual  0–10    0.5   Yes     2     Successful piales + time
coleadero               individual  0–10    0.5   Yes     2     Knockdown + style + time
jineteada_de_toro       individual  0–10    0.5   Yes     2     Time + style on bull
terna_en_ruedo          team        0–30    0.5   No      2     3 sub-scores: piales(0-10) + derribo(0-10) + floreo(0-10)
jineteo_de_yegua        individual  0–10    0.5   Yes     2     Time + style on mare
manganas_a_pie          individual  0–10    0.5   No      2     Successful mangana (on foot)
manganas_a_caballo      individual  0–10    0.5   No      2     Successful mangana (on horse)
paso_de_la_muerte       individual  0–10    0.5   Yes     2     Time + execution
```

**Nota de complejidad para Fase 2**: 5 de las 9 suertes requieren un componente de cronómetro/tiempo que no existe en el MVP (`piales_en_lienzo`, `coleadero`, `jineteada_de_toro`, `jineteo_de_yegua`, `paso_de_la_muerte`) — se construye una sola vez como componente compartido (`useStopwatch` / `<Cronometro>`) y se reutiliza en las 5. `terna_en_ruedo` es la única de equipo (`equipo_id`, no `charro_id`) y la única con 3 sub-puntajes en vez de 1 — requiere su propio layout de formulario.

---

## MODULES

### 1. auth
- Judge login (2 pasos, ver Fase 0.4): 1) `SelectJuez.tsx` — lista de jueces del torneo
  activo (lectura pública, sin auth); 2) `LoginJuez.tsx` — PIN pad → con el `juez.id` ya
  elegido, `supabase.auth.signInWithPassword` (email derivado de `juez.id@charroapp.internal`,
  password = PIN). El PIN es el password real de `auth.users`, no se guarda un hash aparte.
- Admin login: email + password
- Session persisted in `@capacitor/preferences` for offline access
- Hook: `useAuth()` → `{ user, role, isLoading, signInJuez(juezId, pin), signInAdmin(email, pass), signOut }`
- Alta de jueces: Edge Function `create-juez` (service role) crea `auth.users` + fila en `jueces` en una sola operación (ver Fase 1.1) — nunca se crea un juez insertando directo en la tabla pública.

### 2. torneos
- CRUD for torneos, equipos, charros (admin only)
- Bulk import charros via CSV (Fase 2)
- Hook: `useTorneo(id)` → torneo + participaciones cached in Dexie

### 3. calificacion (core)
- Route: `/calificar/:participacionId`
- `SuerteRouter.tsx` renders suerte-specific component based on `participacion.suerte`
- Each suerte component in `src/features/calificacion/suertes/<suerte>.tsx`
- UI requirements: buttons ≥ 60px tap target, high contrast, no keyboard required
- Score input: stepper (+/- 0.5) or large button grid, NOT free text input
- On submit: write to Dexie → trigger sync → navigate to next participacion
- **Fase 1**: solo `CalaDeCaballo.tsx` implementado. `SuerteRouter.tsx` ya contempla el `switch`/mapa completo de las 9 suertes desde el inicio (para no re-arquitecturar en Fase 2), pero las otras 8 rutas devuelven un placeholder ("Próximamente") hasta Fase 2.

### 4. sync
- File: `src/lib/sync.ts`
- `syncPendingCalificaciones()`: reads Dexie dirty records → upsert Supabase → mark synced
- `useSyncStatus()` hook: `{ isOnline, pendingCount, lastSyncedAt }`
- Offline indicator banner in layout when `!isOnline || pendingCount > 0`
- **División de responsabilidad con TanStack Query**: el camino crítico de escritura
  (calificaciones de jueces) pasa exclusivamente por Dexie + este sync engine — nunca por
  TanStack Query. TanStack Query se usa solo para lecturas no-críticas que sí toleran un
  modelo red-primero-cache-después (listados de torneos, datos del dashboard admin). No
  duplicar esta responsabilidad entre ambas capas.

### 5. dashboard (admin)
- Real-time table of all calificaciones for active torneo
- Supabase Realtime subscription: `postgres_changes` on `calificaciones`
- Edit/override with audit log (separate `auditoria` table, Fase 2)
- Export: `json → csv` client-side via `papaparse`

### 6. tablero (public display)
- Route: `/display/:torneoId` — no auth required, read-only
- Full-screen, large text, auto-updates via Realtime
- Shows: current suerte, last 5 scores, ranking table
- No user controls

---

## FOLDER STRUCTURE

```
CharroApp/
├── src/
│   ├── routes/
│   │   ├── index.tsx               # HashRouter — ver Fase 0.2
│   │   └── NotFound.tsx
│   ├── layouts/
│   │   └── AppLayout.tsx
│   ├── features/
│   │   ├── auth/
│   │   │   ├── SelectJuez.tsx     # paso 1: elegir juez del torneo activo
│   │   │   ├── LoginJuez.tsx      # paso 2: PIN pad
│   │   │   ├── LoginAdmin.tsx     # email+pass
│   │   │   ├── ProtectedRoute.tsx # pass-through en 0.2, lógica real en 0.4.4
│   │   │   └── useAuth.ts
│   │   ├── torneos/
│   │   │   ├── TorneoList.tsx
│   │   │   ├── TorneoForm.tsx
│   │   │   ├── ParticipacionForm.tsx
│   │   │   └── useTorneo.ts
│   │   ├── calificacion/
│   │   │   ├── SuerteRouter.tsx      # mapa de las 9 suertes desde Fase 1 (evita re-arquitectura)
│   │   │   ├── CalificacionForm.tsx
│   │   │   └── suertes/
│   │   │       ├── CalaDeCaballo.tsx     # ← Fase 1 (MVP), único implementado
│   │   │       ├── PialesEnLienzo.tsx    # Fase 2 — placeholder en Fase 1
│   │   │       ├── Coleadero.tsx         # Fase 2 — placeholder en Fase 1
│   │   │       ├── JineteadaDeToro.tsx   # Fase 2 — placeholder en Fase 1
│   │   │       ├── TernaEnRuedo.tsx      # Fase 2 — placeholder en Fase 1
│   │   │       ├── JineteoDeYegua.tsx    # Fase 2 — placeholder en Fase 1
│   │   │       ├── ManganasPie.tsx       # Fase 2 — placeholder en Fase 1
│   │   │       ├── ManganasCaballo.tsx   # Fase 2 — placeholder en Fase 1
│   │   │       └── PasoDeLaMuerte.tsx    # Fase 2 — placeholder en Fase 1
│   │   ├── sync/
│   │   │   ├── SyncBanner.tsx
│   │   │   └── useSyncStatus.ts
│   │   ├── dashboard/
│   │   │   └── AdminDashboard.tsx
│   │   └── tablero/
│   │       └── TablPublico.tsx
│   ├── db/
│   │   ├── schema.ts              # Dexie table definitions
│   │   └── db.ts                  # Dexie instance export
│   ├── lib/
│   │   ├── supabase.ts            # createClient + typed DB
│   │   └── sync.ts                # syncPendingCalificaciones()
│   └── hooks/
│       └── useNetworkStatus.ts
├── supabase/
│   ├── migrations/
│   │   ├── 001_init.sql
│   │   └── 002_rls.sql
│   ├── seed.sql                    # 1 admin + 1 juez de prueba, versionado (Fase 0.3)
│   └── functions/
│       └── create-juez/
│           └── index.ts           # crea auth.users + fila en jueces (Fase 1.1)
├── .github/
│   └── workflows/
│       └── ci.yml                 # lint + test + build en cada push/PR (Fase 0.6)
├── capacitor.config.ts
├── vite.config.ts
├── .env.example
└── index.html
```

---

## DEXIE SCHEMA

```typescript
// src/db/schema.ts
interface DexieCalificacion {
  id: string            // uuid, locally generated
  participacion_id: string
  juez_id: string
  puntaje: number
  notas?: string
  updated_at: number    // Date.getTime()
  synced: 0 | 1        // Dexie boolean index
}

interface DexieParticipacion {
  id: string
  torneo_id: string
  suerte: string
  charro_id?: string
  equipo_id?: string
  orden: number
  estado: string
}

interface DexieJuez {
  id: string
  torneo_id: string
  nombre: string
  // sin PIN — nunca se cachea localmente, se ingresa en cada login (LoginJuez.tsx)
}

// db.ts
const db = new Dexie('CharroApp')
db.version(1).stores({
  calificaciones: '++id, participacion_id, juez_id, synced',
  participaciones: 'id, torneo_id, suerte',
  torneos: 'id',
  jueces: 'id, torneo_id',   // cacheado para poblar SelectJuez.tsx offline (Fase 0.5)
})
```

---

## BUILD & DEPLOY COMMANDS

```bash
# Dev
npm run dev                          # Vite dev server

# Build web
npm run build                        # dist/

# Native
npx cap sync                         # copy dist/ to native projects
npx cap run android                  # run on Android emulator/device
npx cap run ios                      # run on iOS simulator/device

# Supabase
npx supabase init
npx supabase db push                 # apply migrations
npx supabase gen types typescript    # regenerate DB types → src/lib/database.types.ts
```

---

## VERIFICATION CHECKLIST

### Fase 0 (Fundación)
```
[ ] npm install && npm run dev levanta la app sin pasos manuales fuera de .env.example
[ ] npm run lint, npm run test, npm run build pasan
[ ] CI bloquea un push con lint/test roto a propósito (0.6)
[ ] Admin de prueba (seed) puede loguearse con email+password
[ ] Juez de prueba (seed) puede loguearse vía selección de nombre + PIN
[ ] Sesión persiste tras cerrar/reabrir la app (sin pedir credenciales de nuevo)
[ ] Con access token vencido y sin red, la app sigue permitiendo operar localmente (no bloquea por sesión)
[ ] RLS: un juez no puede leer calificaciones de otro juez (probado manualmente)
[ ] RLS: un anónimo puede leer /display del torneo activo pero no escribir nada
[ ] Insertar un segundo torneo con estado='activo' falla (índice único de torneo activo)
[ ] La app corre visualmente correcta en al menos un simulador nativo (iOS o Android)
[ ] @capacitor/network reporta el estado de conexión correctamente en el simulador/dispositivo
[ ] Ningún archivo de credenciales (.env.local) está commiteado
```

### Fase 1 (MVP — solo cala_de_caballo)
```
[ ] Offline: disable network → score cala_de_caballo → Dexie has record with synced=0
[ ] Sync: re-enable network → record appears in Supabase within 2s
[ ] Realtime: Admin dashboard updates <1s after judge submits
[ ] Realtime: /display updates <1s after judge submits
[ ] Conflict: two judges submit different scores for same participacion → latest updated_at wins
[ ] Native iOS: full scoring flow works via Capacitor
[ ] Native Android: full scoring flow works via Capacitor
[ ] Tap targets: all interactive elements ≥ 60px height
[ ] PIN login: judge can log in without keyboard (numeric pad only)
[ ] Offline auth: judge can open app and score without network (uses cached session)
```

### Fase 2 (adicional — al agregar las 8 suertes restantes)
```
[ ] Cronómetro: componente de tiempo funciona en las 5 suertes que lo requieren
    (piales_en_lienzo, coleadero, jineteada_de_toro, jineteo_de_yegua, paso_de_la_muerte)
[ ] Terna en ruedo: formulario de equipo captura los 3 sub-puntajes y persiste el total (0-30) correcto
[ ] Terna en ruedo: calificación se asocia a equipo_id, no a charro_id
[ ] Las 9 suertes aparecen correctamente en SuerteRouter.tsx sin placeholders
[ ] Regresión: cala_de_caballo (Fase 1) sigue funcionando sin cambios de comportamiento
```

### Gate — antes del primer torneo real en producción
```
[ ] Supabase: proyecto subido de free tier a plan Pro (elimina auto-pausa por inactividad y habilita backups)
[ ] Backups automáticos confirmados activos
[ ] Variables de entorno de producción configuradas por separado del proyecto dev
```

---

## ALCANCE POR FASE

Cada fase está dividida en sub-fases con objetivo, tareas, archivos clave y criterio de
aceptación (DoD = Definition of Done) propio, para poder verificar progreso sin ambigüedad
y evitar descubrir errores de diseño a medio camino.

### FASE 0 — Fundación

**Objetivo global**: cualquiera clona el repo, corre `npm install && npm run dev`, puede
loguearse como admin o como juez (contra Supabase real), la sesión persiste entre
reinicios, la app corre al menos una vez en un simulador nativo, y CI corre lint+test+build
en cada push. Cero lógica de calificación todavía.

**Cobertura por área** (sigue siendo un solo codebase — no son 3 proyectos separados; ver
`## ALTERNATIVAS CONSIDERADAS` si en algún momento se reconsidera):

| Área | Sub-fases que la cubren en Fase 0 | Qué queda para Fase 1+ |
|---|---|---|
| Backend (Supabase) | 0.3.1–0.3.4 (proyecto, schema, RLS, tooling+seed) | Edge Function `create-juez` (1.1), schema de terna (2.4) |
| App de calificación (juez) | 0.4.1 + 0.4.3 (login juez + modelo de sesión offline), 0.5.1 (Dexie), 0.5.2 (Capacitor + `@capacitor/network`) | Suertes (Fase 1/2), sync engine (1.2–1.3) |
| Dashboard admin (incluye `/display` — ambos son "web", sin Capacitor) | 0.4.2 + 0.4.4 (login admin + rutas protegidas) | CRUD de torneos (1.1), `AdminDashboard.tsx`/`TablPublico.tsx` (1.4) |
| Compartido por las 3 | 0.1 (scaffolding), 0.2 (routing+layout), 0.6 (CI) | — |

**Dependencias reales** (la numeración 0.1→0.6 es agrupación temática, no obliga secuencia
estricta salvo donde se indica): 0.1 es el único prerrequisito universal. A partir de ahí,
**0.2, 0.3.1, 0.5.1 y 0.5.2 no dependen entre sí** y pueden hacerse en cualquier orden o en
paralelo. **0.6 (CI) solo depende de 0.1** — conviene activarlo apenas terminen los scripts
de lint/test/build, no esperar hasta el final, para que proteja el resto de Fase 0 mientras
se construye. Las únicas cadenas estrictas son: `0.3.1 → 0.3.2 → 0.3.3 → 0.3.4` (cada una
depende del schema/RLS anterior) y `0.4 depende de 0.3.3 (RLS) + 0.2 (ProtectedRoute existe)`.

**0.1 — Scaffolding (sin backend)**
- `npm create vite@latest . -- --template react-ts` — el directorio ya tiene `plan.md`/`plan_llm.md`/`.git`; mover ambos `.md` a `docs/` antes de scaffolding (o usar `--force`) para evitar que el CLI rechace un directorio no vacío
- `tsconfig.json` con alias `@/*`, `strict: true`; fijar versión de Node con `.nvmrc`/`engines` en `package.json` (evita "en mi máquina sí funciona")
- ESLint + Prettier configurados
- Tailwind v4 + `npx shadcn@latest init` en **modo variables CSS** (no utility classes) — define un tono base único en `:root`/`.dark` vía `@theme`. **Decisión de alcance**: solo se deja la arquitectura lista para swapear el tono editando esas variables; NO se construye un selector de color visible para el usuario en Fase 0 (candidato a Fase 3 si hace falta)
- Estructura de carpetas completa (`src/features/*`, `src/db`, `src/lib`, `src/hooks`) con stubs mínimos
- Vitest + Testing Library con un smoke test de `<App />`
- Archivos clave: `vite.config.ts`, `tsconfig.json`, `components.json`, `src/main.tsx`, `src/App.tsx`
- **DoD**: `npm run dev` muestra una página con un botón de shadcn/ui estilado con el tono base; `npm run test` y `npm run lint` pasan.

**0.2 — Routing base + layout**
- Instalar `react-router` **v7, en modo Declarative** (API equivalente a v6:
  `<HashRouter>`/`<Routes>`/`<Route>`) — se descarta a propósito el modo Framework/Data de
  v7 (loaders/actions), que está pensado para apps full-stack con su propio servidor, no
  para esta SPA. **Decisión: `HashRouter`, no `BrowserRouter`** — el WebView de Capacitor
  sirve assets desde un scheme local (`capacitor://`/`file://`); routing basado en History
  API requiere config de servidor que no aplica ahí, mientras que hash routing funciona
  igual en web y nativo sin tocar nada.
- Rutas: `/login`, `/admin/*` (layout con nested routes), `/calificar/:participacionId`, `/display/:torneoId`
- `ProtectedRoute.tsx`: se crea aquí como wrapper "pass-through" (deja pasar a cualquiera,
  todavía no hay `useAuth()`). La lógica real de bloqueo por rol se conecta en **0.4.4**, no
  aquí — evita ambigüedad sobre en qué sub-fase vive la protección de rutas.
- Ruta catch-all `*` → `NotFound.tsx` + error boundary básico en `AppLayout.tsx` — evita pantalla en blanco silenciosa ante una ruta o error inesperado
- **Code-splitting por área**: `/admin/*` y `/display/:torneoId` se cargan con `React.lazy()`
  (chunks separados de `/calificar/:id`) — así el bundle que se empaqueta en Capacitor para
  el juez no descarga ni ejecuta el JS del dashboard admin, aunque comparten el mismo
  codebase/repo. Da buena parte del beneficio de "separar" sin la complejidad de un monorepo.
- Placeholders de texto por ruta; `AppLayout.tsx` con header/nav mínimos
- Archivos: `src/routes/index.tsx`, `src/routes/NotFound.tsx`, `src/layouts/AppLayout.tsx`, `src/features/auth/ProtectedRoute.tsx`
- **DoD**: navegar manualmente entre las 4 rutas funciona en `npm run dev`; una ruta inexistente muestra `NotFound`, no pantalla en blanco.

**0.3 — Supabase: proyecto, schema, RLS**

Se divide en 4 checkpoints propios porque mezclaba niveles de riesgo muy distintos —
en particular, RLS necesita su propio gate de verificación antes de seguir, no compartir
DoD con "crear el proyecto".

**0.3.1 — Proyecto + entorno**
- **Decisión de hosting**: free tier de Supabase desde el día 1 para Fase 0–2 (desarrollo/MVP)
  — sus límites duros (500MB DB, 5GB bandwidth/mes, 200 conexiones Realtime concurrentes,
  50k MAU) están muy por encima de lo que esta app necesitará. El riesgo real del free tier
  no son las cuotas: (a) **auto-pausa el proyecto tras 7 días de inactividad** — trampa
  concreta para una app de uso periódico por evento, y (b) **sin backups automáticos** — no
  aceptable para resultados de torneos reales. Por eso: **antes del primer torneo real en
  producción, subir a plan Pro** (elimina ambos riesgos). Ver gate en `## VERIFICATION CHECKLIST`.
- **Decisión de entorno**: se trabaja directo contra el proyecto Supabase hosteado (free
  tier, ver arriba) desde el día 1 — **no** se monta stack local vía `supabase start`
  (Docker). Simplifica el setup (sin dependencia de Docker Desktop) y es consistente con ya
  tener un proyecto real desde Fase 0. Migraciones se aplican con `npx supabase db push`.
- Crear proyecto Supabase (dev); `.env.local` + `.env.example`, confirmar `.gitignore` cubre `.env.local` ANTES del primer commit
- Archivos: `.env.example`
- **DoD**: proyecto creado; `.env.local`/`.env.example` existen y `.gitignore` cubre `.env.local` (el cliente `supabase.ts` todavía no existe — eso se verifica hasta 0.3.4, no aquí).

**0.3.2 — Schema**
- `supabase/migrations/001_init.sql` con las 6 tablas (schema de arriba, sin columna `pin`), incluyendo el **índice único parcial que garantiza un solo torneo activo a la vez** (ver `## DATABASE SCHEMA`) — sin esto, dos torneos con `estado='activo'` simultáneos dejan `/display` y el login de jueces en un estado indeterminado
- `npx supabase db push`
- Archivos: `supabase/migrations/001_init.sql`
- **DoD**: la migración aplica sin errores y las 6 tablas existen; probar el índice único con un insert manual por SQL (dos `insert into torneos (...) values (..., 'activo')`, el segundo debe fallar) — esta prueba NO depende del seed (que llega hasta 0.3.4).

**0.3.3 — RLS (gate de seguridad — checkpoint obligatorio antes de 0.4)**
- `supabase/migrations/002_rls.sql`: policy `juez` (solo sus propias filas en
  `calificaciones`), policy `admin` (todo), **policy de lectura pública** para
  `torneos`/`participaciones`/`calificaciones` del torneo activo (requisito de `/display`).
  Predicado exacto (no basta `USING (true)` — hay que filtrar por torneo activo vía join):
  ```sql
  create policy "public read active torneo" on torneos
    for select to anon using (estado = 'activo');

  create policy "public read participaciones of active torneo" on participaciones
    for select to anon using (
      exists (select 1 from torneos t where t.id = participaciones.torneo_id and t.estado = 'activo')
    );

  create policy "public read calificaciones of active torneo" on calificaciones
    for select to anon using (
      exists (
        select 1 from participaciones p join torneos t on t.id = p.torneo_id
        where p.id = calificaciones.participacion_id and t.estado = 'activo'
      )
    );

  create policy "public read jueces of active torneo" on jueces
    for select to anon using (
      rol = 'juez'
      and exists (select 1 from torneos t where t.id = jueces.torneo_id and t.estado = 'activo')
    );
  ```
  La policy sobre `jueces` es la que hace posible `SelectJuez.tsx` (0.4.3) — sin ella, la
  pantalla de selección de juez no tendría nada que leer. Se filtra por `rol='juez'` para
  no exponer cuentas admin en una lista pública sin auth.
- Archivos: `supabase/migrations/002_rls.sql`
- **Prueba manual obligatoria antes de seguir a 0.4**: confirmar que un juez no puede leer
  calificaciones de otro juez, y que un anónimo puede leer `/display` pero no escribir nada
- **DoD**: ambas pruebas de acceso documentadas y pasando — este DoD debe cumplirse ANTES de empezar 0.4, no en paralelo.

**0.3.4 — Tooling + seed**
- `npx supabase gen types typescript` → `src/lib/database.types.ts`
- `src/lib/supabase.ts` (`createClient<Database>()`)
- **Seed reproducible**: `supabase/seed.sql` versionado en el repo — **1 torneo con
  `estado='activo'` + 1 admin + 1 juez de prueba asociado a ese torneo** (sin el torneo
  activo, el juez no tiene `torneo_id` válido y `SelectJuez.tsx` no tendría nada que listar)
  — en vez de crear datos a mano en el dashboard, así cualquiera que clone el repo parte de
  los mismos datos
- Archivos: `src/lib/supabase.ts`, `src/lib/database.types.ts`, `supabase/seed.sql`
- **DoD**: aplicar migraciones + seed desde cero deja el proyecto en un estado reproducible e idéntico para cualquiera que lo intente.

**0.4 — Auth (selección de juez + PIN, admin)**

Se divide en 4 porque login-admin (trivial) y login-juez (el flujo con más riesgo real de
todo Fase 0) tenían un solo DoD compartido, ocultando que el modelo de sesión offline
necesita su propia verificación explícita.

**0.4.1 — Contrato de `useAuth()` + modelo de sesión offline**
- Forma del hook: `useAuth()` → `{ user, role, isLoading, signInJuez(juezId, pin), signInAdmin(email, pass), signOut() }`
- Al loguear con éxito: guardar en `@capacitor/preferences` el `access_token`/`refresh_token`
  **y además, por separado, `juez_id` + `role` como valores planos** (no depender de decodificar
  el JWT para saber el rol offline — más simple y no se rompe si cambia el formato del token)
- **Decisión de arquitectura crítica — modelo de sesión offline**: escribir en Dexie
  **nunca depende de tener una sesión de Supabase válida**. Solo se necesita el `juez_id`
  cacheado en Preferences desde el último login exitoso. La razón: el access token de
  Supabase expira (~1h) y en un evento de varias horas sin red el SDK no puede refrescarlo;
  si el flujo de calificación dependiera de una sesión válida, "offline auth" se rompería
  justo cuando más se necesita. La sesión de Supabase (con su refresh automático) solo
  importa en el momento de **sincronizar** (Fase 1.3) — si el token expiró y no hay red
  para refrescarlo, el sync simplemente reintenta cuando vuelva la conexión, sin bloquear
  la calificación local. Primer login siempre requiere conectividad (no hay forma de
  resolver PIN→identidad sin red la primera vez) — esto es una restricción aceptada, no un bug.
- Archivos: `src/features/auth/useAuth.ts`
- **DoD**: el hook compila y expone la forma correcta (aún sin ninguna pantalla de login conectada).

**0.4.2 — Login admin**
- `LoginAdmin.tsx`: email+password (react-hook-form + zod) → `signInWithPassword`
- Archivos: `LoginAdmin.tsx`
- **DoD**: el admin del seed (0.3.4) puede loguearse contra Supabase real.

**0.4.3 — Login juez (el checkpoint más crítico de Fase 0)**
- `SelectJuez.tsx`: lista de jueces del torneo activo (lectura pública, cacheable en Dexie)
- `LoginJuez.tsx`: PIN pad grande (≥60px, sin teclado) → con el `juezId` ya elegido, arma
  `email = {juezId}@charroapp.internal` y llama `signInWithPassword` con el PIN como password real
- Archivos: `SelectJuez.tsx`, `LoginJuez.tsx`
- **DoD**: el juez del seed puede loguearse; y el test que realmente importa — forzar un
  access token vencido y desconectar la red, y confirmar que escribir en Dexie sigue
  funcionando sin error (valida en la práctica la decisión de 0.4.1).

**0.4.4 — Integración: rutas protegidas end-to-end**
- Conectar la lógica real en `ProtectedRoute.tsx` (creado como pass-through en 0.2): ahora sí lee `role` (de Preferences, no del JWT) y redirige si no corresponde
- Archivos: `src/features/auth/ProtectedRoute.tsx`
- **DoD**: acceder a `/admin/*` sin sesión redirige a `/login`; un juez autenticado no puede entrar a `/admin/*`.

**0.5 — Dexie + Capacitor nativo**

Se divide en 2 porque son técnicamente independientes — un problema de Xcode/Android SDK
en la parte de Capacitor no debería bloquear ni confundirse con el estado de Dexie.

**0.5.1 — Dexie schema**
- `src/db/schema.ts` + `db.ts` (tablas `calificaciones`, `participaciones`, `torneos`, y
  también `jueces` cacheados para poblar `SelectJuez.tsx` offline)
- Archivos: `src/db/schema.ts`, `src/db/db.ts`
- **DoD**: Dexie abre en el navegador (DevTools → Application → IndexedDB) con las 4 tablas creadas y vacías.

**0.5.2 — Capacitor nativo**
- `npx cap init` + plataformas `ios`/`android`; `capacitor.config.ts` (`webDir: 'dist'`,
  `server.url` solo para dev con live-reload, documentado como "quitar en build de producción")
- **`@capacitor/network` instalado y usado para detectar conectividad** en vez de depender 
  solo de `navigator.onLine`/evento `online` del navegador — este es conocido por no
  dispararse de forma confiable en WebView de iOS/Android al recuperar señal real; en web
  (sin Capacitor) se mantiene el fallback a la API del navegador
- `npm run build && npx cap sync && npx cap run android` (o `ios`) — confirmar que carga la pantalla de login
- Archivos: `capacitor.config.ts`, `src/hooks/useNetworkStatus.ts`
- **DoD**: la app corre visualmente correcta en al menos un simulador nativo; activar/desactivar modo avión en el simulador/dispositivo hace que `@capacitor/network` reporte el cambio correctamente.

**0.6 — CI básico**
- GitHub Actions (u equivalente): workflow que corre en cada push/PR: `npm install`, `npm run lint`, `npm run test`, `npm run build`
- Archivos: `.github/workflows/ci.yml`
- **DoD**: un push con un error de lint o de test introducido a propósito hace fallar el workflow — prueba que el gate realmente bloquea, no solo que "existe".

---

### Casos de uso — Fase 0

A diferencia de Fase 1+ (que tendrá casos de uso de negocio: calificar, sincronizar, ver
el tablero), los de Fase 0 son de **infraestructura y acceso** — validan que las
decisiones de arquitectura de arriba resuelven escenarios reales, no solo que "el comando
corrió sin error". Cada uno referencia qué sub-fase(s) verifica.

- **UC-0.1 Onboarding de desarrollador**: alguien nuevo clona el repo, corre `npm install && npm run dev`, sin pasos manuales fuera de lo documentado en `.env.example`. → 0.1, 0.3.1
- **UC-0.2 Primer login de admin (con red)**: el admin de prueba entra con email+password y llega a `/admin`. → 0.4.2, 0.4.4
- **UC-0.3 Primer login de juez (con red)**: el juez ve la lista de jueces del torneo activo, elige su nombre, teclea su PIN, entra a la app. → 0.3.3, 0.4.3
- **UC-0.4 Juez reabre la app sin red, horas después**: con sesión cacheada de un login previo y el access token ya vencido, el juez abre la app en modo avión y la app no lo bloquea. → 0.4.1, 0.4.3 — **este es el caso de uso que valida la promesa central de "offline-first" de todo el proyecto**, no solo de Fase 0.
- **UC-0.5 Público visita `/display` sin cuenta**: un anónimo abre `/display/:torneoId` del torneo activo y ve datos, sin poder loguearse ni escribir nada. → 0.3.3
- **UC-0.6 Acceso cruzado entre jueces (debe fallar)**: un juez autenticado intenta leer/escribir calificaciones de otro juez vía llamada directa a la API — RLS lo rechaza. → 0.3.3
- **UC-0.7 Primera corrida en simulador nativo**: la app compilada corre en un simulador Android o iOS y llega hasta la pantalla de login. → 0.5.2
- **UC-0.8 Regresión de CI**: un desarrollador introduce un error de lint a propósito y el push queda bloqueado por CI antes de llegar a `main`. → 0.6

---

**Riesgos Fase 0**: no fijar Hash vs Browser router desde el inicio obliga a refactor
doloroso después; no probar RLS antes de construir UI encima esconde huecos de seguridad
hasta tarde; no confirmar `.gitignore` antes del primer `git add` puede commitear secretos;
confiar solo en `navigator.onLine` sin `@capacitor/network` puede dar falsos positivos de
conectividad en nativo; sin el índice único de torneo activo, dos torneos activos
simultáneos rompen `/display` y el login de jueces de forma silenciosa.

---

### FASE 1 — MVP: `cala_de_caballo` end-to-end

**Objetivo global**: probar el pipeline completo (Dexie → sync → Supabase → Realtime →
Admin/Display) con el menor riesgo, calificando una sola suerte.

**1.1 — Gestión mínima de torneos**
- `TorneoForm.tsx` (nombre, fecha, lugar); alta de charros con participación en `cala_de_caballo`
- Edge Function `create-juez`: crea `auth.users` (admin API, PIN como password) + fila en
  `jueces` en una sola operación — implementa la decisión de auth de Fase 0
- `useTorneo(id)`: torneo + participaciones, cacheadas en Dexie
- Archivos: `TorneoForm.tsx`, `ParticipacionForm.tsx` (nuevo), `useTorneo.ts`, `supabase/functions/create-juez/index.ts`
- **DoD**: admin crea un torneo con 3+ charros con participación en `cala_de_caballo`

**1.2 — Calificación (offline write)**
- `CalaDeCaballo.tsx`: stepper 0-10 paso 0.5, "Guardar y siguiente"
- `SuerteRouter.tsx`: switch completo de las 9 suertes (placeholder "Próximamente" en las otras 8, para no re-arquitecturar en Fase 2)
- Al enviar: uuid local, escribir en Dexie (`synced: 0`, `updated_at: Date.now()`), navegar al siguiente pendiente por `orden`
- **DoD**: con red desactivada, calificar dos charros deja 2 registros en Dexie con `synced: 0`

**1.3 — Sync engine**
- `src/lib/sync.ts`: `syncPendingCalificaciones()` — lee dirty de Dexie, `upsert` por `id`
  (evita duplicados en reintentos), marca `synced: 1` solo tras confirmación; flag
  `isSyncing` en memoria para evitar sync concurrente si `online` se dispara más de una vez
- Listeners: `window.addEventListener('online')` + reconexión de canal Realtime
- `useSyncStatus()` + `SyncBanner.tsx`
- **DoD**: reconectar sube los pendientes a Supabase en <2s, sin duplicados aunque `online` se dispare varias veces

**1.4 — Realtime: dashboard + display**
- `AdminDashboard.tsx`: tabla + `postgres_changes` filtrado por `torneo_id`
- `TablPublico.tsx` en `/display/:torneoId`: sin auth (usa la policy de lectura pública de 0.3), misma suscripción
- **DoD**: calificar desde un tercer dispositivo actualiza ambas pantallas en <1s

**1.5 — Verificación en dispositivo real**
- Recorrer el checklist "Fase 1" (sección `## VERIFICATION CHECKLIST`) en hardware real con
  conectividad intermitente real, no solo throttling de DevTools

**Riesgos Fase 1**: el comportamiento del evento `online` en WebView nativo puede diferir
del navegador de escritorio — probar en dispositivo real es obligatorio; el id generado en
Dexie debe ser el mismo que se sube (upsert por id) o se duplican filas; `SelectJuez.tsx`
debe filtrar correctamente por torneo activo.

---

### FASE 2 — Cobertura completa del reglamento

**2.1 — Cronómetro compartido**: `useStopwatch()` + `<Cronometro>`, con test unitario antes
de integrarlo en cualquier suerte (evita repetir el mismo bug 5 veces). Debe sobrevivir si
la app pierde foco brevemente (cambio de app en tablet).

**2.2 — Suertes individuales con cronómetro** (`piales_en_lienzo`, `coleadero`,
`jineteada_de_toro`, `jineteo_de_yegua`, `paso_de_la_muerte`): implementar
`jineteada_de_toro` primero (la más simple: tiempo + 1 puntaje de estilo) para validar el
patrón `<Cronometro> + stepper`, luego replicar en las 4 restantes.

**2.3 — Suertes individuales sin cronómetro restantes** (`manganas_a_pie`,
`manganas_a_caballo`): mismo patrón que `CalaDeCaballo.tsx`, sin cambios de arquitectura.

**2.4 — Terna en ruedo (equipo)**: usa `equipo_id` en vez de `charro_id` en
`participaciones`; formulario captura 3 sub-puntajes (piales/derribo/floreo) → total 0-30
en `puntaje` + detalle en la columna `detalle jsonb` (ver `## DATABASE SCHEMA`). Selector de
equipo en vez de charro individual.

**2.5 — Regresión + extensión dashboard/display**: correr checklist "Fase 2" (sección
`## VERIFICATION CHECKLIST`); extender `AdminDashboard.tsx`/`TablPublico.tsx` a las 9
suertes (ranking, filtro por suerte); re-correr el checklist completo de Fase 1 para
confirmar que `cala_de_caballo` sigue funcionando sin cambios de comportamiento.

---

### FASE 3 — Extras

Cada ítem es un sub-proyecto independiente con su propia migración SQL separada (no
mezclar en un solo archivo, para poder habilitar/deshabilitar cada uno sin afectar a otros):

- 3.1 Export PDF/Excel (client-side; `papaparse` ya cubre CSV, agregar librería de PDF si aplica)
- 3.2 Bulk CSV import de charros
- 3.3 Audit log (tabla `auditoria` nueva, registro en cada UPDATE de calificación por admin)
- 3.4 Históricos/estadísticas por charro entre torneos
- 3.5 Push notifications para jueces
- 3.6 Escaramuza charra (reglas de scoring distintas — requiere su propio análisis de reglamento antes de tocar schema)

---

## ALTERNATIVAS CONSIDERADAS Y DESCARTADAS

Registro de por qué se eligió cada pieza del stack sobre su alternativa más obvia, para no
volver a discutirlo en el futuro sin una razón nueva.

```
Alternativa                          Por qué se descarta
────────────────────────────────────────────────────────────────────────────────────────
Firebase/Firestore (vs Supabase)     Modelo de documentos NoSQL obliga a desnormalizar el
                                      schema relacional ya diseñado y complica reportes/
                                      export (Fase 3); reglas de seguridad no son SQL, más
                                      difícil de auditar que RLS de Postgres.

Dexie Cloud (vs sync propio)         Movería la fuente de verdad fuera de Postgres/Supabase
                                      — se perdería SQL para reportes y habría dos backends
                                      en vez de uno. El modelo de conflicto necesario
                                      (last-write-wins por fila propia) es demasiado simple
                                      para justificar un servicio de sync completo.

RxDB (vs Dexie)                      Resolución de conflictos más sofisticada de la que este
                                      proyecto necesita; varios plugins clave (replicación
                                      avanzada, adjuntos, encriptación) requieren licencia
                                      comercial de pago — costo/complejidad no justificado.

PowerSync / ElectricSQL              Legítimo para conflictos multi-escritor complejos; aquí
(motor de sync dedicado sobre        cada juez solo escribe sus propias filas (RLS ya lo
Postgres, vs sync propio)            garantiza), así que el sync engine hecho a mano es
                                      proporcional al problema real. Revisar de nuevo solo si
                                      Fase 3 introduce ediciones concurrentes admin+juez sobre
                                      la misma fila.

React Native (vs Capacitor)          Mejor rendimiento nativo puro, pero exige codebase
                                      separado del web — incumple el requisito explícito de
                                      "single codebase".

Tauri Mobile (vs Capacitor)          Aún inmaduro para producción en una app que depende de
                                      eventos en vivo sin margen de error.

Redux Toolkit (vs Zustand)           Boilerplate no justificado para el tamaño real del
                                      estado global de esta app (auth + estado de sync + UI).

TanStack Router (vs React Router)    Mejor type-safety en params de ruta, pero React Router
                                      tiene más ejemplos/guías específicas para apps híbridas
                                      Capacitor (hash routing en WebView nativo,
                                      deep-linking) — se prioriza esto para Fase 0.
```
