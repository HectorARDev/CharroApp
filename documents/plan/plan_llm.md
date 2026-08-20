# CharroApp — Scoring System: Implementation Plan

> Este documento es el plan operativo detallado, para consumo de agentes/LLM (o para
> ejecutar sin ambigüedad): sub-fases, archivos exactos, SQL literal, criterios de
> aceptación. La versión para lectura humana (decisiones y su porqué, sin el detalle de
> ejecución) es `plan.md`. Al editar uno, sincronizar el otro. Para cliente/patrocinador
> (alcance por hitos, cronograma y costos, sin jerga técnica) ver `plan_cliente.md`.

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
torneo_id   uuid REFERENCES torneos(id)  -- NULL para rol='admin' (un admin no pertenece a un solo torneo); NOT NULL para rol='juez'
nombre      text NOT NULL
rol         text CHECK (rol IN ('admin','juez')) DEFAULT 'juez'
activo      boolean NOT NULL DEFAULT true  -- desactivar en vez de borrar (ver ON DELETE RESTRICT en calificaciones.juez_id, Fase 0.3.2); false = no aparece en SelectJuez.tsx ni puede seguir calificando vía Supabase

-- calificaciones
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
participacion_id  uuid REFERENCES participaciones(id)
juez_id     uuid REFERENCES jueces(id)
puntaje     numeric(5,1) NOT NULL CHECK (puntaje >= 0 AND puntaje <= 30)  -- 30 = techo global (terna_en_ruedo); el máximo exacto por suerte (0-10 individual) se valida en cliente (zod), ver 0.3.2
detalle     jsonb                 -- NULL salvo terna_en_ruedo: {"piales": n, "derribo": n, "floreo": n}
notas       text                  -- privado del juez; la policy pública de lectura (0.3.3) NO se extiende a authenticated para esta tabla — solo el propio juez y el admin la leen por policy, aunque un anon que llame la API REST directo aún podría leerla (ver mitigación en 0.3.3)
updated_at  timestamptz NOT NULL  -- used for conflict resolution
synced_at   timestamptz           -- NULL = not synced to server yet
-- RLS completo (las 6 tablas, incluye equipos/charros y la función is_admin()): ver Fase 0.3.3
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
  activo (lectura pública, sin auth); 2) `LoginJuez.tsx` — PIN pad, PIN de 6 dígitos (mínimo
  de password que exige Supabase Auth por default, ver 0.4.3) → con el `juez.id` ya
  elegido, `supabase.auth.signInWithPassword` (email derivado de `juez.id@charroapp.internal`,
  password = PIN). El PIN es el password real de `auth.users`, no se guarda un hash aparte.
- Admin login: email + password
- Session persisted in `@capacitor/preferences` for offline access
- Hook: `useAuth()` → `{ user, role, isLoading, signInJuez(juezId, pin), signInAdmin(email, pass), signOut }`
- Alta de jueces: Edge Function `create-juez` (service role) crea `auth.users` + fila en `jueces` en una sola operación (ver Fase 1.1) — nunca se crea un juez insertando directo en la tabla pública.
- Rate limiting de Supabase Auth contra fuerza bruta del PIN: ver 0.4.3 (verificación obligatoria, no solo confiar en el default).

### 2. torneos
- CRUD for torneos, equipos, charros (admin only)
- Bulk import charros via CSV (Fase 3 — ver 3.2, no hay sub-fase de Fase 2 que lo construya)
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
- `useSyncStatus()` hook: `{ isOnline, pendingCount, lastSyncedAt }` — `isOnline` reutiliza
  `useNetworkStatus()` (0.5.2) internamente, no reimplementa su propia detección de
  conectividad, para no terminar con dos fuentes de verdad de "¿hay red?" que puedan
  desincronizarse
- Offline indicator banner in layout when `!isOnline || pendingCount > 0`
- **División de responsabilidad con TanStack Query**: el camino crítico de escritura
  (calificaciones de jueces) pasa exclusivamente por Dexie + este sync engine — nunca por
  TanStack Query. TanStack Query se usa solo para lecturas no-críticas que sí toleran un
  modelo red-primero-cache-después (listados de torneos, datos del dashboard admin). No
  duplicar esta responsabilidad entre ambas capas.

### 5. dashboard (admin)
- Real-time table of all calificaciones for active torneo
- Supabase Realtime subscription: `postgres_changes` on `calificaciones`
- Edit/override with audit log (separate `auditoria` table, Fase 3 — ver 3.3, no Fase 2)
- Export: `json → csv` client-side via `papaparse` (Fase 3 — ver 3.1)

### 6. tablero (public display)
- Route: `/display/:torneoId` — no auth required, read-only
- Full-screen, large text, auto-updates via Realtime
- Shows: current suerte, last 5 scores, ranking table
- No user controls
- Query a `calificaciones` con columnas explícitas (`.select('id, participacion_id, puntaje, detalle, updated_at')`), nunca `select('*')` — evita traer `notas` a la pantalla pública (ver mitigación en 0.3.3)

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
│   ├── seed.sql                    # torneo/equipos/charros de prueba, versionado (Fase 0.3.4)
│   └── functions/
│       ├── create-juez/
│       │   └── index.ts           # crea auth.users + fila en jueces (Fase 1.1)
│       └── reset-pin/
│           └── index.ts           # resetea el PIN de un juez existente (Fase 1.1)
├── scripts/
│   └── seed-auth.ts               # crea admin+juez de prueba vía Admin API (Fase 0.3.4)
├── e2e/
│   └── login-offline.spec.ts      # Playwright: sesión offline con token vencido (Fase 0.4.3)
├── .github/
│   └── workflows/
│       └── ci.yml                 # lint + test + build en cada push/PR (Fase 0.6)
├── capacitor.config.ts
├── vite.config.ts
├── playwright.config.ts
├── tsconfig.json
├── .nvmrc
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
  jueces: 'id, torneo_id',   // cache de solo-lectura para que SelectJuez.tsx muestre algo razonable sin red (ver nota en Fase 0.4.3 — no permite COMPLETAR un login sin red, eso sigue requiriendo conectividad)
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
npx supabase db push                 # apply migrations (001_init.sql + 002_rls.sql)
npm run db:seed                      # seed.sql + scripts/seed-auth.ts (admin+juez de prueba)
npx supabase gen types typescript --project-id <ref> --schema public > src/lib/database.types.ts

# Tests
npm run test                         # Vitest
npx playwright test                  # E2E, incluye simulación de offline
```

---

## VERIFICATION CHECKLIST

### Fase 0 (Fundación)
```
[ ] npm install → .env.local propio (proyecto Supabase propio, 0.3.1) → npx supabase db push → npm run db:seed → npm run dev levanta la app sin más pasos manuales que esos 4 comandos
[ ] npm run lint, npm run test, npm run build pasan
[ ] CI bloquea un push con lint/test roto a propósito (0.6); CI usa la misma versión de Node que .nvmrc
[ ] CI corre npx playwright test contra el proyecto Supabase dedicado a CI (con sus secrets configurados) y pasa, no solo lint/test/build
[ ] Admin de prueba (seed) puede loguearse con email+password
[ ] Juez de prueba (seed) puede loguearse vía selección de nombre + PIN
[ ] Sesión persiste tras cerrar/reabrir la app (sin pedir credenciales de nuevo)
[ ] Con access token vencido y sin red, la app sigue permitiendo operar localmente (no bloquea por sesión)
[ ] RLS está habilitado (ENABLE ROW LEVEL SECURITY) en las 6 tablas, no solo las policies creadas
[ ] RLS: un juez no puede leer calificaciones de otro juez (probado manualmente)
[ ] RLS: un juez SÍ puede leer torneos/participaciones del torneo activo ya autenticado (no solo como anónimo)
[ ] RLS: un anónimo puede leer /display del torneo activo pero no escribir nada
[ ] RLS: el admin puede crear/editar torneos, equipos, charros y participaciones (policy de escritura admin, no solo lectura)
[ ] RLS: un anónimo puede leer nombres de equipos/charros del torneo activo (necesario para que /display muestre la tabla de ranking con nombres, no solo IDs)
[ ] RLS: un juez con jueces.activo=false desaparece de SelectJuez.tsx y no puede insertar/actualizar calificaciones, aunque su sesión de Supabase siga siendo válida
[ ] /calificar/:participacionId sin sesión redirige a /login, igual que /admin/*
[ ] Insertar un segundo torneo con estado='activo' falla (índice único de torneo activo)
[ ] La app corre visualmente correcta en al menos un simulador nativo (iOS o Android)
[ ] @capacitor/network reporta el estado de conexión correctamente en el simulador/dispositivo
[ ] Ningún archivo de credenciales (.env.local) está commiteado
[ ] `SUPABASE_SERVICE_ROLE_KEY` no tiene prefijo `VITE_`; un `grep` sobre `dist/` tras `npm run build` no encuentra ninguna ocurrencia de esa key
[ ] Rate limiting de Supabase Auth para `signInWithPassword` confirmado activo (Dashboard → Auth → Rate Limits) — mitiga fuerza bruta contra el PIN de 6 dígitos, dado que el email de login es predecible y el UUID del juez es público
[ ] `calificaciones.puntaje` rechaza un insert manual con valor negativo o mayor a 30 (CHECK constraint)
[ ] Autenticado como un juez, un insert de prueba en `calificaciones` con `participacion_id` de un torneo ajeno al suyo falla
[ ] Autenticado como un juez, una lectura de `calificaciones` filtrando por `juez_id` de otro juez devuelve 0 filas (la policy pública de `calificaciones` es solo `anon`, no `authenticated`)
[ ] npm run build genera un Service Worker (vite-plugin-pwa) que precachea solo el app-shell, sin entradas de supabase.co
[ ] /admin y /display/:torneoId cargan desde una URL pública del hosting web elegido, no solo localhost
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

**Objetivo global**: cualquiera clona el repo, crea su propio proyecto Supabase, aplica
migraciones y seed (ver 0.3.1), y levanta la app con `npm run dev`; puede loguearse como
admin o como juez (contra Supabase real), la sesión persiste entre reinicios, la app corre
al menos una vez en un simulador nativo, y CI corre lint+test+build en cada push. Cero
lógica de calificación todavía.

**Cobertura por área** (sigue siendo un solo codebase — no son 3 proyectos separados; ver
`## ALTERNATIVAS CONSIDERADAS` si en algún momento se reconsidera):

| Área | Sub-fases que la cubren en Fase 0 | Qué queda para Fase 1+ |
|---|---|---|
| Backend (Supabase) | 0.3.1–0.3.4 (proyecto, schema, RLS, tooling+seed) | Edge Functions `create-juez` + `reset-pin` (1.1), schema de terna (2.4) |
| App de calificación (juez) | 0.4.1 + 0.4.3 (login juez + modelo de sesión offline), 0.5.1 (Dexie), 0.5.2 (Capacitor + `@capacitor/network`) | Suertes (Fase 1/2), sync engine (1.2–1.3) |
| Dashboard admin (incluye `/display` — ambos son "web", sin Capacitor) | 0.4.2 + 0.4.4 (login admin + rutas protegidas) | CRUD de torneos (1.1), `AdminDashboard.tsx`/`TablPublico.tsx` (1.4) |
| Compartido por las 3 | 0.1 (scaffolding + PWA), 0.2 (routing+layout), 0.6 (CI + hosting web) | — |

**Dependencias reales** (la numeración 0.1→0.6 es agrupación temática, no obliga secuencia
estricta salvo donde se indica): 0.1 es el único prerrequisito universal. A partir de ahí,
**0.2, 0.3.1, 0.5.1 y 0.5.2 no dependen entre sí** y pueden hacerse en cualquier orden o en
paralelo. **0.6 (CI) solo depende de 0.1** — conviene activarlo apenas terminen los scripts
de lint/test/build, no esperar hasta el final, para que proteja el resto de Fase 0 mientras
se construye. Las únicas cadenas estrictas son: `0.3.1 → 0.3.2 → 0.3.3 → 0.3.4` (cada una
depende del schema/RLS anterior) y, dentro de 0.4: **0.4.1 solo depende de 0.2** (es un
stub sin red); **0.4.2 y 0.4.3 dependen de 0.3.4** (necesitan el admin/juez del seed para
poder loguearse de verdad, no solo de 0.3.3 — 0.3.3 deja las policies listas pero sin datos
no hay con qué probar el login); **0.4.4 depende de 0.4.1 + 0.2** (conecta el hook real a
`ProtectedRoute`).

**0.1 — Scaffolding (sin backend)**
- `npm create vite@latest . -- --template react-ts --force` — el directorio ya tiene
  `documents/` (con `plan.md`/`plan_llm.md` ya organizados ahí) y `.git`; usar `--force` para
  que el CLI no rechace el directorio no vacío. No mover ni renombrar `documents/`.
- `tsconfig.json` con alias `@/*`, `strict: true`; **`vite.config.ts` debe declarar el mismo
  alias en `resolve.alias` (o usar el plugin `vite-tsconfig-paths`)** — el alias en
  `tsconfig.json` por sí solo afecta al type-checker, no a cómo Vite resuelve imports en
  runtime; olvidarlo produce un error de import que solo aparece al correr, no al compilar
  tipos. Fijar versión de Node con `.nvmrc`/`engines` en `package.json` (evita "en mi máquina
  sí funciona").
- ESLint + Prettier configurados
- Tailwind v4 + `npx shadcn@latest init` en **modo variables CSS** (no utility classes) — define un tono base único en `:root`/`.dark` vía `@theme`. **Decisión de alcance**: solo se deja la arquitectura lista para swapear el tono editando esas variables; NO se construye un selector de color visible para el usuario en Fase 0 (candidato a Fase 3 si hace falta)
- Estructura de carpetas completa (`src/features/*`, `src/db`, `src/lib`, `src/hooks`) con stubs mínimos
- Vitest + Testing Library con un smoke test de `<App />`
- `@playwright/test` instalado (`npx playwright install`) con `playwright.config.ts` mínimo,
  incluyendo `webServer` configurado para levantar `npm run preview` (o `dev`) automáticamente
  antes de correr tests — así ni un developer ni CI (0.6) necesitan levantar el servidor a
  mano por separado. Sin tests todavía en esta sub-fase, solo la herramienta lista. Su primer
  uso real es la prueba de sesión offline de 0.4.3.
- `vite-plugin-pwa` registrado en `vite.config.ts` con manifest mínimo (nombre, ícono base,
  `theme_color`) y modo `generateSW`/`injectManifest` acotado a precachear **solo el
  app-shell** (JS/CSS/HTML/estáticos) — implementa aquí la decisión ya tomada en
  `## ARCHITECTURE` de que el Service Worker nunca debe interceptar llamadas a Supabase (eso
  ya lo hace Dexie); dejarlo sin configurar en esta sub-fase es fácil de olvidar más adelante
  porque no bloquea nada visualmente
- Archivos clave: `vite.config.ts`, `tsconfig.json`, `components.json`, `playwright.config.ts`, `.nvmrc`, `src/main.tsx`, `src/App.tsx`
- **DoD**: `npm run dev` muestra una página con un botón de shadcn/ui estilado con el tono base; `npm run test` y `npm run lint` pasan; `npm run build` genera un Service Worker cuyo manifest de precache contiene únicamente assets estáticos del app-shell — cero entradas apuntando a `supabase.co`; `npx playwright test` corre (aunque sin tests todavía) sin error de configuración.

**0.2 — Routing base + layout**
- Instalar `react-router` **v7, en modo Declarative** (API equivalente a v6:
  `<HashRouter>`/`<Routes>`/`<Route>`) — se descarta a propósito el modo Framework/Data de
  v7 (loaders/actions), que está pensado para apps full-stack con su propio servidor, no
  para esta SPA. **Decisión: `HashRouter`, no `BrowserRouter`** — el WebView de Capacitor
  sirve assets desde un scheme local (`capacitor://`/`file://`); routing basado en History
  API requiere config de servidor que no aplica ahí, mientras que hash routing funciona
  igual en web y nativo sin tocar nada.
- Rutas: `/login`, `/login/admin`, `/admin/*` (layout con nested routes),
  `/calificar/:participacionId`, `/display/:torneoId`. **Estructura del login** (para no
  dejarlo ambiguo entre esta sub-fase y 0.4.2/0.4.3): `/login` renderiza `SelectJuez.tsx` por
  defecto y el paso de PIN (`LoginJuez.tsx`) es un **estado interno del mismo flujo**, no una
  ruta aparte (evita perder el juez ya elegido si el usuario refresca) — `/login/admin`
  renderiza `LoginAdmin.tsx` por separado.
- `ProtectedRoute.tsx`: se crea aquí como wrapper "pass-through" (deja pasar a cualquiera,
  todavía no hay `useAuth()`). La lógica real de bloqueo por rol se conecta en **0.4.4**, no
  aquí — evita ambigüedad sobre en qué sub-fase vive la protección de rutas. **Envuelve
  `/admin/*` y `/calificar/:participacionId`** (ambas requieren sesión — admin o juez según
  corresponda); `/display/:torneoId` es la única ruta intencionalmente pública, nunca se
  envuelve en `ProtectedRoute`.
- Ruta catch-all `*` → `NotFound.tsx` + error boundary básico en `AppLayout.tsx` — evita pantalla en blanco silenciosa ante una ruta o error inesperado
- **Code-splitting por área**: `/admin/*` y `/display/:torneoId` se cargan con `React.lazy()`
  (chunks separados de `/calificar/:id`) — así el bundle que se empaqueta en Capacitor para
  el juez no descarga ni ejecuta el JS del dashboard admin, aunque comparten el mismo
  codebase/repo. Da buena parte del beneficio de "separar" sin la complejidad de un monorepo.
- Placeholders de texto por ruta; `AppLayout.tsx` con header/nav mínimos
- Archivos: `src/routes/index.tsx`, `src/routes/NotFound.tsx`, `src/layouts/AppLayout.tsx`, `src/features/auth/ProtectedRoute.tsx`
- **DoD**: navegar manualmente entre las 5 rutas listadas arriba funciona en `npm run dev`; una ruta inexistente muestra `NotFound`, no pantalla en blanco.

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
- **Convención de nombres obligatoria para las variables de entorno** (Vite empaqueta
  automáticamente en el JS servido al navegador cualquier variable con prefijo `VITE_` —
  usarlo por error en la equivocada equivale a publicarla): `VITE_SUPABASE_URL` y
  `VITE_SUPABASE_ANON_KEY` (consumidas por `src/lib/supabase.ts`, cliente) sí llevan el
  prefijo; `SUPABASE_SERVICE_ROLE_KEY` (usada solo por `scripts/seed-auth.ts` y, en Fase 1, por
  las Edge Functions) **nunca** lleva prefijo `VITE_` ni se importa desde ningún archivo bajo
  `src/` — vive únicamente en contextos server-side/CLI. `.env.example` debe documentar ambas
  con este mismo criterio.
- **Región del proyecto**: elegir explícitamente la región disponible más cercana a México
  (al momento de escribir esto, `us-east-1` — no dejarlo en el default que asigne el
  dashboard) — importa para la latencia de Realtime durante un evento en vivo; ninguna región
  de Supabase está físicamente en México, así que esto es una elección deliberada, no una
  optimización perfecta.
- **Historia de onboarding (resuelve UC-0.1)**: cada developer crea su **propio** proyecto
  Supabase (free tier) — no se comparte un proyecto dev entre el equipo. Onboarding real
  entonces es: `npm install` → llenar `.env.local` con las credenciales del proyecto propio →
  `npx supabase db push` (aplica 0.3.2+0.3.3) → `npm run db:seed` (aplica 0.3.4) → `npm run dev`.
  Esto es más que ".env.example" a secas, así que el DoD de esta sub-fase y UC-0.1 deben
  reflejar esos 4 comandos como el onboarding real, no solo `npm install && npm run dev`.
- Archivos: `.env.example`
- **DoD**: proyecto creado en la región elegida; `.env.local`/`.env.example` existen y `.gitignore` cubre `.env.local` (el cliente `supabase.ts` todavía no existe — eso se verifica hasta 0.3.4, no aquí).

**0.3.2 — Schema**
- `supabase/migrations/001_init.sql` con las 6 tablas (schema de arriba, sin columna `pin`), incluyendo el **índice único parcial que garantiza un solo torneo activo a la vez** (ver `## DATABASE SCHEMA`) — sin esto, dos torneos con `estado='activo'` simultáneos dejan `/display` y el login de jueces en un estado indeterminado
- **Nota de traducción**: la lista de columnas en `## DATABASE SCHEMA` es notación compacta,
  no `CREATE TABLE` literal — hay que envolver cada tabla en `create table <nombre> ( ... );`
  con comas entre columnas. Al hacerlo, resolver explícitamente el `ON DELETE` de cada FK que
  el schema de referencia no especifica (las demás ya lo indican como `ON DELETE CASCADE`):
  - `participaciones.torneo_id` → `ON DELETE CASCADE` (si se borra el torneo, sus participaciones no tienen sentido)
  - `participaciones.charro_id`, `participaciones.equipo_id` → `ON DELETE CASCADE`
  - `calificaciones.participacion_id` → `ON DELETE CASCADE`
  - `calificaciones.juez_id` → `ON DELETE RESTRICT` (evita borrar un juez con historial de puntajes y perder el registro; para desactivar un juez se usa `jueces.activo = false`, ver más abajo, no un DELETE)
  - `jueces.torneo_id` → `ON DELETE CASCADE`
  - `calificaciones.puntaje` agrega `CHECK (puntaje >= 0 AND puntaje <= 30)` — sin esto, el
    schema no tiene ningún resguardo servidor para el campo más importante de la app (el
    resultado de la competencia), pese a que el patrón `CHECK` ya se usa en otras columnas
    (`torneos.estado`, `participaciones.estado`, `jueces.rol`); 30 es el techo global
    (`terna_en_ruedo`, la única suerte de equipo con 3 sub-puntajes); el máximo exacto por
    suerte (0-10 en las individuales) sigue validándose en cliente (zod) — un `CHECK`
    dependiente de la suerte de la participación referenciada requeriría un trigger, evaluado
    como sobre-ingeniería para Fase 0.
  - **Limitación aceptada**: `jueces.id = auth.users.id`, pero borrar una fila de `jueces` en
    cascada (por ejemplo al borrar su torneo) **no** borra el `auth.users` asociado — queda una
    cuenta de auth huérfana. No se resuelve en Fase 0 (no hay flujo de borrado de torneos
    todavía); documentarlo aquí evita que se descubra como "bug" más adelante.
- `npx supabase db push`
- Archivos: `supabase/migrations/001_init.sql`
- **DoD**: la migración aplica sin errores y las 6 tablas existen; probar el índice único con un
  insert manual por SQL (dos `insert into torneos (...) values (..., 'activo')`, el segundo debe
  fallar) — esta prueba NO depende del seed (que llega hasta 0.3.4); probar el `CHECK` de
  `puntaje` con un insert manual que use un valor negativo o mayor a 30 (debe fallar; no depende
  de FKs válidas porque `participacion_id`/`juez_id` son nullable).
- **Nota de flujo (no bloquea el DoD de esta sub-fase)**: el índice único de torneo activo
  implica que activar un torneo nuevo requiere desactivar el anterior en la misma operación —
  ese flujo de escritura (transacción o RPC "activar torneo") se construye en **Fase 1.1**
  (`TorneoForm.tsx`), no aquí; se deja anotado para que no quede sin dueño.

**0.3.3 — RLS (gate de seguridad — checkpoint obligatorio antes de 0.4)**
- `supabase/migrations/002_rls.sql`, en este orden:

  **1) Habilitar RLS en las 6 tablas primero** — sin esto ninguna policy de abajo tiene
  efecto y, por default, cualquiera con la anon key lee/escribe todo. Es el paso que más
  fácil se olvida porque no da error si falta, solo deja todo abierto en silencio:
  ```sql
  alter table torneos enable row level security;
  alter table equipos enable row level security;
  alter table charros enable row level security;
  alter table participaciones enable row level security;
  alter table jueces enable row level security;
  alter table calificaciones enable row level security;
  ```

  **2) Lectura pública** para `torneos`/`participaciones`/`jueces` del torneo activo
  (requisito de `/display` y de `SelectJuez.tsx`), y una policy **separada y más restringida**
  para `calificaciones` (ver justificación abajo). Predicado exacto (no basta `USING (true)` —
  hay que filtrar por torneo activo vía join). **`torneos`/`participaciones`/`jueces` aplican a
  `anon` Y a `authenticated`** — un juez ya logueado usa el rol Postgres `authenticated`, no
  `anon`; si estas policies fueran solo `to anon`, un juez logueado perdería la lectura de
  `torneos`/`participaciones` que `useTorneo()` necesita en Fase 1:
  ```sql
  create policy "public read active torneo" on torneos
    for select to anon, authenticated using (estado = 'activo');

  create policy "public read participaciones of active torneo" on participaciones
    for select to anon, authenticated using (
      exists (select 1 from torneos t where t.id = participaciones.torneo_id and t.estado = 'activo')
    );

  create policy "public read jueces of active torneo" on jueces
    for select to anon, authenticated using (
      rol = 'juez'
      and activo = true
      and exists (select 1 from torneos t where t.id = jueces.torneo_id and t.estado = 'activo')
    );
  ```
  La policy sobre `jueces` es la que hace posible `SelectJuez.tsx` (0.4.3) — sin ella, la
  pantalla de selección de juez no tendría nada que leer. Se filtra por `rol='juez'` para
  no exponer cuentas admin en una lista pública sin auth, y por `activo = true` para que un
  juez desactivado deje de aparecer en la lista de selección.

  **`calificaciones`: policy pública SOLO para `anon`, no `authenticated`** — a diferencia de
  las tres de arriba:
  ```sql
  create policy "public read calificaciones of active torneo" on calificaciones
    for select to anon using (
      exists (
        select 1 from participaciones p join torneos t on t.id = p.torneo_id
        where p.id = calificaciones.participacion_id and t.estado = 'activo'
      )
    );
  ```
  **Por qué solo `anon` (a diferencia de las otras tres)**: si esta policy también aplicara a
  `authenticated`, cualquier juez logueado podría leer las calificaciones de **todos los demás
  jueces** vía la API directa — viola UC-0.6 ("un juez no puede leer calificaciones de otro
  juez") y su ítem correspondiente en `## VERIFICATION CHECKLIST`. Acotada a `anon`, un juez
  autenticado solo lee las suyas (punto 3 más abajo) o, si es admin, todas (punto 4);
  `/display` sigue funcionando porque nunca requiere sesión (siempre corre como `anon`).
  **Limitación aceptada**: si un admin o un juez abre `/display` desde un navegador donde ya
  tiene sesión iniciada, el cliente de Supabase reutiliza esa sesión y las requests salen como
  `authenticated`, no `anon` — un admin lo sigue viendo todo (cubierto por la policy de admin,
  punto 4), pero un juez normal vería la pantalla vacía en ese caso puntual. No se resuelve en
  Fase 0 (implicaría que `TablPublico.tsx`, Fase 1.4, use un cliente de Supabase sin sesión
  persistida); se documenta aquí para no descubrirlo como "bug" más adelante.

  **`notas` sigue siendo legible por cualquier `anon` en esta policy** — RLS filtra filas, no
  columnas, así que no hay forma de excluir `notas` de esta policy sin mover la columna a una
  tabla aparte (cambio que toca también el motor de sync de Fase 1.3, fuera de alcance de Fase
  0). **Mitigación de Fase 0**: `TablPublico.tsx` (Fase 1.4) debe seleccionar columnas
  explícitas (`.select('id, participacion_id, puntaje, detalle, updated_at')`), nunca
  `select('*')` — reduce la exposición práctica sin cerrar el hueco a nivel de API directa.
  Queda como candidato de hardening (tabla `notas` separada con su propia RLS) si más adelante
  se necesita garantía real a nivel de base de datos, no solo de aplicación.

  **También `equipos`/`charros` necesitan lectura pública del torneo activo** — sin esta
  policy, `/display/:torneoId` no podría mostrar nombres de charros/equipos en la tabla de
  ranking (solo IDs vacíos):
  ```sql
  create policy "public read equipos of active torneo" on equipos
    for select to anon, authenticated using (
      exists (select 1 from torneos t where t.id = equipos.torneo_id and t.estado = 'activo')
    );

  create policy "public read charros of active torneo" on charros
    for select to anon, authenticated using (
      exists (
        select 1 from equipos e join torneos t on t.id = e.torneo_id
        where e.id = charros.equipo_id and t.estado = 'activo'
      )
    );
  ```

  **3) Juez: solo sus propias `calificaciones`** (select/insert/update — sin delete, un juez
  nunca borra una calificación ya enviada). Las 3 exigen además `activo = true` en su propia
  fila de `jueces` — un juez desactivado con sesión de Supabase todavía válida no debe poder
  seguir leyendo ni escribiendo vía red. Esta subconsulta **no** tiene el problema circular de
  `is_admin()` (punto 4 más abajo): vive en una policy de `calificaciones`, no de `jueces`
  misma, y la policy pública del punto 2 ya permite que un juez lea su propia fila (matchea
  `rol='juez'` + `activo=true` + torneo activo), así que se resuelve sin necesitar una función
  `security definer`. **`insert`/`update` exigen además que `participacion_id` pertenezca al
  `torneo_id` del propio juez y que ese torneo esté `activo`** — sin este join, `juez_id =
  auth.uid()` por sí solo no dice nada sobre a qué torneo pertenece la fila: un juez válido
  (incluso de un torneo ya `finalizado`, mientras su sesión de Supabase no haya expirado)
  podría escribir calificaciones de **cualquier `participacion_id` de cualquier torneo**, no
  solo del suyo:
  ```sql
  create policy "juez read own calificaciones" on calificaciones
    for select to authenticated using (
      juez_id = auth.uid()
      and exists (select 1 from jueces j where j.id = auth.uid() and j.activo = true)
    );

  create policy "juez insert own calificaciones" on calificaciones
    for insert to authenticated with check (
      juez_id = auth.uid()
      and exists (
        select 1 from jueces j
        join participaciones p on p.torneo_id = j.torneo_id
        join torneos t on t.id = j.torneo_id
        where j.id = auth.uid()
          and j.activo = true
          and t.estado = 'activo'
          and p.id = calificaciones.participacion_id
      )
    );

  create policy "juez update own calificaciones" on calificaciones
    for update to authenticated using (juez_id = auth.uid()) with check (
      juez_id = auth.uid()
      and exists (
        select 1 from jueces j
        join participaciones p on p.torneo_id = j.torneo_id
        join torneos t on t.id = j.torneo_id
        where j.id = auth.uid()
          and j.activo = true
          and t.estado = 'activo'
          and p.id = calificaciones.participacion_id
      )
    );
  ```
  La lectura (`select`) se deja sin esta restricción adicional a propósito — un juez debe poder
  seguir leyendo su propio historial aunque su torneo ya haya terminado (revisar lo que
  calificó); solo la escritura queda atada al torneo activo.

  **Limitación aceptada**: esto bloquea el *sync* hacia Supabase, no la escritura local en
  Dexie — el modelo de sesión offline (0.4.1) ya establece que calificar nunca depende de una
  sesión válida, así que un juez desactivado (o de un torneo ya finalizado) que ya estaba
  calificando sin red puede seguir escribiendo localmente; su intento de sync fallará
  silenciosamente hasta que se investigue el motivo. No se agrega manejo especial de este caso
  en Fase 0/1.

  **4) Admin: acceso total en las 6 tablas** — sin esto, `TorneoForm.tsx`/`ParticipacionForm.tsx`
  (Fase 1.1) fallan por RLS en cuanto el paso 1 quede aplicado, porque ninguna policy cubre
  todavía escritura de admin sobre `torneos`/`equipos`/`charros`/`participaciones`/`jueces`.
  **No usar un `exists (select 1 from jueces ...)` inline dentro de la policy de `jueces`
  misma** — sería una policy sobre `jueces` que, para evaluarse, necesita leer `jueces`, y la
  única fila que probaría que el usuario es admin (la suya propia, con `rol='admin'`) no
  matchea la policy pública (que solo expone `rol='juez'`) ni la propia policy admin
  (todavía no evaluada) → el admin quedaría bloqueado de su propia tabla. Se resuelve con una
  función `security definer`, que al ejecutar con los privilegios del dueño de la función
  evita ese problema de RLS circular (patrón estándar de Supabase para este caso):
  ```sql
  create or replace function is_admin()
  returns boolean
  language sql
  security definer
  set search_path = public
  stable
  as $$
    select exists (select 1 from jueces where id = auth.uid() and rol = 'admin' and activo = true);
  $$;

  create policy "admin full access torneos" on torneos
    for all to authenticated using (is_admin());
  create policy "admin full access equipos" on equipos
    for all to authenticated using (is_admin());
  create policy "admin full access charros" on charros
    for all to authenticated using (is_admin());
  create policy "admin full access participaciones" on participaciones
    for all to authenticated using (is_admin());
  create policy "admin full access jueces" on jueces
    for all to authenticated using (is_admin());
  create policy "admin full access calificaciones" on calificaciones
    for all to authenticated using (is_admin());
  ```
- Archivos: `supabase/migrations/002_rls.sql`
- **Prueba manual obligatoria antes de seguir a 0.4**: confirmar que (a) autenticado como un
  juez, una query a `calificaciones` filtrando por el `juez_id` de otro juez devuelve 0 filas
  (la policy pública ahora es solo `anon`, y la propia policy de juez no la cubre), (b) un
  anónimo puede leer `/display` pero no escribir nada — incluyendo nombres de charros/equipos,
  no solo torneo/participaciones, (c) con RLS habilitado (paso 1 aplicado) una tabla sin policy
  explícita deniega todo por default — confirmar esto con una query de un rol sin policy
  aplicable y esperar 0 filas, no un error, para no confundir "denegado" con "tabla rota", (d)
  autenticado como el admin del seed (0.3.4), `select is_admin()` devuelve `true` y un
  `insert`/`update` de prueba en `torneos` funciona — valida que la función `security definer`
  resuelve el caso circular y no deja al admin bloqueado de su propia tabla, (e) con el juez
  del seed puesto en `activo = false` manualmente, ya no aparece en la policy pública de
  `jueces` y un `insert` de prueba en `calificaciones` con su `juez_id` falla — confirma que el
  mecanismo de desactivación (0.3.2) realmente bloquea, no solo existe la columna, (f)
  autenticado como un juez, un `insert` de prueba en `calificaciones` con un `participacion_id`
  que pertenece a un torneo **distinto** al `torneo_id` del juez falla — confirma que el join
  agregado en el punto 3 realmente ata la escritura al torneo propio, no solo al `juez_id`.
- **DoD**: las 6 pruebas de acceso documentadas y pasando — este DoD debe cumplirse ANTES de empezar 0.4, no en paralelo.

**0.3.4 — Tooling + seed**
- `npx supabase gen types typescript --project-id <ref> --schema public > src/lib/database.types.ts`
  (`<ref>` es el ID del proyecto Supabase creado en 0.3.1, visible en su URL/dashboard) — el
  comando sin `--project-id` ni la redirección de salida no genera nada usable.
- `src/lib/supabase.ts` (`createClient<Database>()`)
- **Seed reproducible, en dos piezas** (una sola no alcanza — ver por qué abajo):
  1. `supabase/seed.sql` versionado en el repo: **1 torneo con `estado='activo'`** + equipos/
     charros de prueba (sin el torneo activo, `SelectJuez.tsx` no tendría nada que listar).
     Aplicado automáticamente por `supabase db push` o corrido manual con `psql`/SQL editor.
  2. `scripts/seed-auth.ts`: crea **1 admin + 1 juez de prueba** usando
     `supabase.auth.admin.createUser({ email, password, email_confirm: true })` (Admin API,
     requiere la **service role key** — nunca se commitea, se lee de `.env.local`) y luego
     inserta su fila correspondiente en `jueces`. **`email_confirm: true` es obligatorio, no
     opcional**: los emails son sintéticos (`{id}@charroapp.internal`) que nunca reciben ni
     pueden confirmar un correo real — sin este flag, Supabase Auth bloquea
     `signInWithPassword` con "Email not confirmed" y nadie puede loguearse, sin importar que
     RLS/schema estén perfectos. **Por qué no basta con `seed.sql`**: `auth.users` no se puede poblar de forma
     confiable con un `INSERT` SQL directo (requiere hash de password correcto y columnas
     internas del sistema de auth) — es el mismo problema que resuelve la Edge Function
     `create-juez` en Fase 1.1, y usar aquí el mismo patrón (Admin API) lo valida temprano en
     vez de descubrirlo recién en Fase 1. **El PIN del juez de prueba debe ser un valor fijo y
     documentado** (por ejemplo, una constante en el propio script o una variable
     `SEED_JUEZ_PIN` en `.env.example` con un default de desarrollo) — no generado al azar.
     `e2e/login-offline.spec.ts` (0.4.3) necesita loguear a este juez de forma automatizada
     tanto localmente como en CI (0.6); un PIN aleatorio en cada corrida del seed rompería ese
     test sin forma de saber qué credencial usar.
  3. Exponer ambos pasos como un solo comando: `npm run db:seed` (corre `seed.sql` vía CLI y
     luego `scripts/seed-auth.ts` vía `tsx`/`ts-node`).
  4. **No es idempotente — a propósito**: el seed asume una base de datos recién migrada.
     Correrlo dos veces falla de forma explícita, no duplica datos en silencio: el segundo
     `insert` de torneo activo choca con el índice único (0.3.2) y `auth.admin.createUser()`
     falla con "usuario ya existe" si el email ya está tomado. Para reiniciar, resetear el
     proyecto Supabase o borrar las filas de prueba antes de volver a correr `npm run db:seed`.
- Archivos: `src/lib/supabase.ts`, `src/lib/database.types.ts`, `supabase/seed.sql`, `scripts/seed-auth.ts`
- **DoD**: `npx supabase db push && npm run db:seed` desde cero deja el proyecto en un estado reproducible e idéntico para cualquiera que lo intente, incluyendo el admin y el juez de prueba ya logueables.

**0.4 — Auth (selección de juez + PIN, admin)**

Se divide en 4 porque login-admin (trivial) y login-juez (el flujo con más riesgo real de
todo Fase 0) tenían un solo DoD compartido, ocultando que el modelo de sesión offline
necesita su propia verificación explícita.

**0.4.1 — Contrato de `useAuth()` + modelo de sesión offline**
- Forma del hook: `useAuth()` → `{ user, role, isLoading, signInJuez(juezId, pin), signInAdmin(email, pass), signOut() }`.
  `signOut()` limpia tanto la sesión de Supabase como los valores planos de `@capacitor/preferences`
  (`juez_id`/`role`) — dejar cualquiera de los dos no invalida la sesión visible, pero sí
  puede confundir un siguiente login en el mismo dispositivo compartido entre jueces.
- **Dispositivo compartido entre jueces — registros pendientes de sincronizar**:
  `signInJuez(juezId, pin)` debe verificar, antes de completar el login, si existen registros
  en `Dexie.calificaciones` con `synced: 0` y un `juez_id` **distinto** al que está por iniciar
  sesión. Si los hay, el login debe advertir claramente (o bloquear hasta que haya red y esos
  registros sincronicen) — sin esto, las calificaciones pendientes del juez anterior quedan
  huérfanas: al reconectar, el sync intentaría subirlas bajo la sesión del juez nuevo, y la
  policy de `calificaciones` (0.3.3, punto 3) las rechazaría porque su `juez_id` no coincide
  con `auth.uid()`, sin ningún mensaje de error visible para nadie.
- **Contrato de error**: `signInJuez`/`signInAdmin` **retornan** `{ error }` (mismo estilo que
  el SDK de Supabase) en vez de lanzar excepción — evita que cada formulario tenga que
  envolver la llamada en `try/catch`; un PIN o password incorrecto es un resultado esperado
  del flujo, no una excepción.
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
- `SelectJuez.tsx`: al montar, si hay red, hace `select` de la policy pública de `jueces`
  (0.3.3) y hace `bulkPut` en `Dexie.jueces`; siempre renderiza desde Dexie (no directo de la
  respuesta de red) para que abrir la pantalla sin red muestre la última lista conocida en
  vez de una pantalla vacía. **Esto no habilita completar un login sin red** — 0.4.1 ya
  estableció que el primer login siempre requiere conectividad (no hay forma de resolver
  PIN→identidad offline); el cache solo evita una UI rota, el botón de login sigue
  necesitando red y debe mostrar un error claro si se intenta sin ella.
- `LoginJuez.tsx`: PIN pad grande (≥60px, sin teclado), **PIN de exactamente 6 dígitos** — no
  4 (el PIN tradicional de cajero), porque Supabase Auth exige un mínimo de 6 caracteres de
  password por default; usar menos de 6 rompe `createUser()`/`signInWithPassword` en cuanto
  se pruebe contra Supabase real. Sigue siendo solo numérico, mismo pad, un dígito más → con
  el `juezId` ya elegido, arma `email = {juezId}@charroapp.internal` y llama
  `signInWithPassword` con el PIN como password real
- **Mitigación de fuerza bruta (obligatoria antes de dar por cerrado este checkpoint)**: el
  PIN de 6 dígitos (10^6 combinaciones) es el password real de la cuenta, el email de login es
  100% predecible (`{juezId}@charroapp.internal`) y el `id` de cada juez ya es público vía la
  policy que alimenta `SelectJuez.tsx` — sin protección adicional, es un objetivo de fuerza
  bruta trivial. Confirmar explícitamente en el Dashboard de Supabase (Auth → Rate Limits) que
  el límite de intentos de `signInWithPassword` está activo (viene activado por default, pero
  debe **verificarse**, no asumirse). Si el volumen de intentos fallidos en producción lo
  justifica más adelante, evaluar CAPTCHA (hCaptcha/Turnstile, soportado nativamente por
  Supabase Auth) — no se implementa en Fase 0, solo se confirma que el rate limit por default
  está activo.
- Archivos: `SelectJuez.tsx`, `LoginJuez.tsx`, `e2e/login-offline.spec.ts`
- **DoD**: el juez del seed puede loguearse; y el test que realmente importa — forzar un
  access token vencido y desconectar la red, y confirmar que escribir en Dexie sigue
  funcionando sin error (valida en la práctica la decisión de 0.4.1). **Automatizado con
  Playwright** (`context.setOffline(true)`, instalado en 0.1) en vez de quedar solo como
  prueba manual — es exactamente el caso para el que el stack ya incluye Playwright, y deja
  este checkpoint crítico protegido en CI, no solo verificado una vez a mano.

**0.4.4 — Integración: rutas protegidas end-to-end**
- Conectar la lógica real en `ProtectedRoute.tsx` (creado como pass-through en 0.2, ya
  envolviendo `/admin/*` y `/calificar/:participacionId`): ahora sí lee `role` (de
  Preferences, no del JWT) y redirige si no corresponde
- Archivos: `src/features/auth/ProtectedRoute.tsx`
- **DoD**: acceder a `/admin/*` sin sesión redirige a `/login`; un juez autenticado no puede
  entrar a `/admin/*`; acceder a `/calificar/:participacionId` sin sesión también redirige a
  `/login` (no solo `/admin/*`); `/display/:torneoId` sigue siendo accesible sin sesión.

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
- **Versión mínima de OS/SDK**: fijar explícitamente `minSdkVersion` en Android (proyecto
  nativo generado por Capacitor) e iOS deployment target — dado que el equipo de un lienzo
  charro es exactamente el tipo de dispositivo que no se renueva seguido (tablets de varios
  años, conectividad ya de por sí pobre), no dejarlo en el default del template sin revisarlo;
  confirmar contra los dispositivos reales que el equipo tenga disponibles antes de fijar el
  número.
- `npm run build && npx cap sync && npx cap run android` (o `ios`) — confirmar que carga la pantalla de login
- Archivos: `capacitor.config.ts`, `src/hooks/useNetworkStatus.ts`
- **DoD**: la app corre visualmente correcta en al menos un simulador nativo; activar/desactivar modo avión en el simulador/dispositivo hace que `@capacitor/network` reporte el cambio correctamente.

**0.6 — CI básico + hosting web**
- GitHub Actions (u equivalente): workflow que corre en cada push/PR: `npm install`, `npm run lint`, `npm run test`, `npm run build`, `npx playwright install --with-deps`, `npx playwright test`. **El step `actions/setup-node` debe leer la
  misma versión fijada en `.nvmrc` (0.1)** (`node-version-file: '.nvmrc'`) en vez de un
  número hardcodeado aparte — evita que CI y desarrollo local diverjan silenciosamente.
- **Playwright en CI necesita un Supabase real que hablar** — el test de sesión offline
  (0.4.3) se conecta a un proyecto Supabase de verdad, ya migrado y sembrado. La filosofía de
  "cada developer crea su propio proyecto" (0.3.1) no aplica a un job de CI: se crea **un
  proyecto Supabase dedicado a CI** (excepción explícita, CI no es un developer), con
  migraciones+seed ya aplicados, y sus credenciales (`SUPABASE_URL`, `SUPABASE_ANON_KEY`)
  guardadas como secrets de GitHub Actions — sin esto, el step de Playwright falla por no
  tener con qué autenticar al juez de prueba, no por un bug real.
  **Tarea explícita de esta sub-fase, no un hecho consumado**: alguien del equipo (no CI mismo)
  crea ese proyecto una sola vez, corre `npx supabase db push` + `npm run db:seed` contra él
  (usando el `SEED_JUEZ_PIN` fijo de 0.3.4) y carga `SUPABASE_URL`/`SUPABASE_ANON_KEY` como
  secrets del repo — sin este paso manual documentado, "el proyecto de CI" queda como algo que
  se asume existente sin que ninguna sub-fase lo haya creado.
- **Decisión de hosting web** ("Web browser" es una de las 3 plataformas del alcance del
  proyecto — admin y `/display` deben ser accesibles sin pasar por Capacitor): desplegar el
  build web (`npm run build` → `dist/`) en un host estático con preview deploys por PR
  (Vercel/Netlify/Cloudflare Pages — cualquiera sirve, no hay requisito técnico que
  desempate). El build nativo (Capacitor) sigue empaquetando assets localmente y no depende
  de este hosting. **Configurar en el panel del host las mismas variables de `.env.example`**
  apuntando al proyecto Supabase dev (0.3.1) — sin esto el build desplegado carga pero no
  puede hablar con Supabase, un fallo que no aparece en `npm run build` local porque ahí sí
  existe `.env.local`.
- Archivos: `.github/workflows/ci.yml`
- **DoD**: un push con un error de lint o de test introducido a propósito hace fallar el
  workflow — prueba que el gate realmente bloquea, no solo que "existe"; `/admin` y
  `/display/:torneoId` cargan correctamente desde una URL pública del host elegido (con datos
  reales del torneo activo, no una pantalla en blanco por falta de env vars), no solo desde
  `localhost`.

---

### Casos de uso — Fase 0

A diferencia de Fase 1+ (que tendrá casos de uso de negocio: calificar, sincronizar, ver
el tablero), los de Fase 0 son de **infraestructura y acceso** — validan que las
decisiones de arquitectura de arriba resuelven escenarios reales, no solo que "el comando
corrió sin error". Cada uno referencia qué sub-fase(s) verifica.

- **UC-0.1 Onboarding de desarrollador**: alguien nuevo clona el repo, crea su propio
  proyecto Supabase (0.3.1), y corre `npm install` → `npx supabase db push` → `npm run
  db:seed` → `npm run dev` — sin más pasos manuales que esos, y sin necesitar credenciales de
  nadie más del equipo. → 0.1, 0.3.1, 0.3.4
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
simultáneos rompen `/display` y el login de jueces de forma silenciosa; configurar
`vite-plugin-pwa` sin acotar el precache al app-shell puede terminar cacheando respuestas de
Supabase y reintroducir el mismo bug de "datos viejos" que Dexie ya resuelve; crear usuarios
vía Admin API sin `email_confirm: true` bloquea todo login con "Email not confirmed" pese a
que RLS/schema estén perfectos; usar un PIN de menos de 6 caracteres choca con el mínimo de
password que exige Supabase Auth por default — ambos son fallas que solo aparecen al probar
contra Supabase real, no en ninguna revisión de código; la policy de admin sobre
`calificaciones` (`for all`) incluye DELETE sin ningún log de auditoría hasta Fase 3 — riesgo
aceptado y conocido (documentado también en `plan.md`), no un descuido, pero vale tenerlo
presente antes de dar acceso de admin a alguien fuera del equipo técnico; la columna `notas`
de `calificaciones` sigue siendo legible por cualquier `anon` que llame a la API REST
directamente (ver mitigación de aplicación en 0.3.3) — aceptado en Fase 0, candidato a tabla
separada con RLS propia si se necesita garantía real a nivel de base de datos.

---

### FASE 1 — MVP: `cala_de_caballo` end-to-end

**Objetivo global**: probar el pipeline completo (Dexie → sync → Supabase → Realtime →
Admin/Display) con el menor riesgo, calificando una sola suerte.

**1.1 — Gestión mínima de torneos**
- `TorneoForm.tsx` (nombre, fecha, lugar); alta de charros con participación en `cala_de_caballo`
- **Activar torneo**: acción de admin que desactiva el torneo `activo` actual (si existe,
  pasa a `estado='finalizado'` o el que corresponda) y activa el nuevo, en una sola
  transacción/RPC — el índice único parcial de 0.3.2 exige que esto sea atómico, no dos
  `update` sueltos desde el cliente (una falla a medio camino dejaría cero o dos torneos
  activos).
- Edge Function `create-juez`: crea `auth.users` (admin API, PIN como password, **`email_confirm: true`
  obligatorio** — mismo motivo que `scripts/seed-auth.ts` en 0.3.4, el email sintético nunca se confirma solo) + fila en
  `jueces` en una sola operación — implementa la decisión de auth de Fase 0
- **Reseteo de PIN**: hasta ahora ninguna fase del plan cubre qué pasa cuando un juez olvida
  su PIN en cancha (escenario probable en un evento real) — se resuelve aquí con el mismo
  patrón que `create-juez`: Edge Function `reset-pin` (solo admin, actualiza el password de
  `auth.users` del juez) en vez de dejarlo como reseteo manual ad-hoc vía dashboard de
  Supabase
- **Desactivar juez** (candidato de esta sub-fase, junto a `reset-pin` — mismo tipo de
  operación administrativa): UI mínima para que el admin alterne `jueces.activo`. La columna
  y las policies ya existen desde Fase 0 (0.3.2/0.3.3); aquí solo falta el botón. Si no
  alcanza el tiempo de esta sub-fase, se puede resolver manualmente vía SQL Editor de
  Supabase hasta que se construya — no bloquea nada más de Fase 1. **Importante al
  implementarlo**: solo voltear `activo = false` no impide el login — Supabase Auth no
  conoce esa columna, así que `signInWithPassword` seguiría funcionando y el juez recién
  ahí chocaría con RLS al leer/escribir (UX confusa: "entré pero no puedo hacer nada"). La
  implementación real debe además deshabilitar la cuenta en `auth.users` vía Admin API
  (mismo patrón que `create-juez`/`reset-pin`, no un simple `UPDATE`), para que el rechazo
  ocurra limpio en el login, no después.
- `useTorneo(id)`: torneo + participaciones, cacheadas en Dexie
- Archivos: `TorneoForm.tsx`, `ParticipacionForm.tsx` (nuevo), `useTorneo.ts`, `supabase/functions/create-juez/index.ts`, `supabase/functions/reset-pin/index.ts`
- **DoD**: admin crea un torneo con 3+ charros con participación en `cala_de_caballo`; admin resetea el PIN de un juez existente y el juez puede loguearse con el PIN nuevo

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

- 3.1 Export CSV/PDF/Excel del dashboard admin (client-side; CSV vía `papaparse`, agregar librería de PDF si aplica) — ninguna fase anterior construye export, se hace todo aquí
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
