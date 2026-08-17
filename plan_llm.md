# CharroApp — Scoring System: Implementation Plan

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
│   ├── routes.tsx                 # HashRouter — ver Fase 0.2
│   ├── layouts/
│   │   └── AppLayout.tsx
│   ├── features/
│   │   ├── auth/
│   │   │   ├── SelectJuez.tsx     # paso 1: elegir juez del torneo activo
│   │   │   ├── LoginJuez.tsx      # paso 2: PIN pad
│   │   │   ├── LoginAdmin.tsx     # email+pass
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
reinicios, y la app corre al menos una vez en un simulador nativo. Cero lógica de
calificación todavía.

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
  todavía no hay `useAuth()`). La lógica real de bloqueo por rol se conecta en **0.4**, no
  aquí — evita ambigüedad sobre en qué sub-fase vive la protección de rutas.
- Placeholders de texto por ruta; `AppLayout.tsx` con header/nav mínimos
- Archivos: `src/routes.tsx`, `src/layouts/AppLayout.tsx`, `src/features/auth/ProtectedRoute.tsx`
- **DoD**: navegar manualmente entre las 4 rutas funciona en `npm run dev`.

**0.3 — Supabase: proyecto, schema, RLS**
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
- Crear proyecto Supabase (dev); `supabase/migrations/001_init.sql` con las 6 tablas (schema de arriba, sin columna `pin`)
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
  ```
- `npx supabase gen types typescript` → `src/lib/database.types.ts`
- `src/lib/supabase.ts` (`createClient<Database>()`), `.env.local` + `.env.example`,
  confirmar `.gitignore` cubre `.env.local` ANTES del primer commit con esto
- **Seed reproducible**: `supabase/seed.sql` versionado en el repo (1 admin + 1 juez de
  prueba) en vez de crear datos a mano en el dashboard — cualquiera que clone el repo parte
  de los mismos datos
- **Prueba manual obligatoria antes de seguir**: confirmar que un juez no puede leer
  calificaciones de otro juez, y que un anónimo puede leer `/display` pero no escribir nada
- **DoD**: políticas RLS verificadas manualmente (documentar qué se probó, no solo "aplicado")

**0.4 — Auth (selección de juez + PIN, admin)**
- `useAuth()`: `{ user, role, isLoading, signInJuez(juezId, pin), signInAdmin(email, pass), signOut() }`
- `LoginAdmin.tsx`: email+password (react-hook-form + zod) → `signInWithPassword`
- `SelectJuez.tsx`: lista de jueces del torneo activo (lectura pública, cacheable en Dexie)
- `LoginJuez.tsx`: PIN pad grande (≥60px, sin teclado) → con el `juezId` ya elegido, arma
  `email = {juezId}@charroapp.internal` y llama `signInWithPassword` con el PIN como password real
- Al loguear con éxito: guardar en `@capacitor/preferences` el `access_token`/`refresh_token`
  **y además, por separado, `juez_id` + `role` como valores planos** (no depender de decodificar
  el JWT para saber el rol offline — más simple y no se rompe si cambia el formato del token)
- Conecta aquí la lógica real de `ProtectedRoute.tsx` (creado como pass-through en 0.2): ahora sí lee `role` y redirige si no corresponde
- Archivos: `src/features/auth/useAuth.ts`, `LoginAdmin.tsx`, `SelectJuez.tsx`, `LoginJuez.tsx`

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
- **DoD**: login de admin y de juez funcionan contra Supabase real; cerrar/reabrir la app no
  vuelve a pedir credenciales; calificar con el access token ya vencido y sin red sigue
  escribiendo en Dexie sin error

**0.5 — Dexie + Capacitor nativo**
- `src/db/schema.ts` + `db.ts` (tablas `calificaciones`, `participaciones`, `torneos`, y
  ahora también `jueces` cacheados para poblar `SelectJuez.tsx` offline)
- `npx cap init` + plataformas `ios`/`android`; `capacitor.config.ts` (`webDir: 'dist'`,
  `server.url` solo para dev con live-reload, documentado como "quitar en build de producción")
- `npm run build && npx cap sync && npx cap run android` (o `ios`) — confirmar que carga la pantalla de login
- **DoD**: la app corre visualmente correcta en al menos un simulador nativo

**Riesgos Fase 0**: no fijar Hash vs Browser router desde el inicio obliga a refactor
doloroso después; no probar RLS antes de construir UI encima esconde huecos de seguridad
hasta tarde; no confirmar `.gitignore` antes del primer `git add` puede commitear secretos.

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
