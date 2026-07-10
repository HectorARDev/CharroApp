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
Layer              Package                        Version   Purpose
─────────────────────────────────────────────────────────────────────
Build              vite                           ^5        Bundler
Build              vite-plugin-pwa                latest    PWA + Workbox SW
Language           typescript                     ^5        Type safety
UI framework       react                          ^18       Component model
Mobile wrapper     @capacitor/core                ^6        iOS/Android native
Mobile wrapper     @capacitor/cli                 ^6        CLI build tool
Mobile storage     @capacitor/preferences         ^6        Key-value config
Styling            tailwindcss                    ^3        Utility CSS
UI components      shadcn/ui                      latest    Accessible components
State              zustand                        ^4        Global state
Data fetching      @tanstack/react-query          ^5        Server state + cache
Offline DB         dexie                          ^3        IndexedDB ORM (TS-first)
Backend            @supabase/supabase-js          ^2        PostgreSQL + Realtime + Auth
Form               react-hook-form + zod          latest    Validation
```

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

-- jueces
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()  -- = auth.users.id
torneo_id   uuid REFERENCES torneos(id)
nombre      text NOT NULL
pin         text NOT NULL  -- bcrypt hash of 4-6 digit PIN
rol         text CHECK (rol IN ('admin','juez')) DEFAULT 'juez'

-- calificaciones
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
participacion_id  uuid REFERENCES participaciones(id)
juez_id     uuid REFERENCES jueces(id)
puntaje     numeric(5,1) NOT NULL
notas       text
updated_at  timestamptz NOT NULL  -- used for conflict resolution
synced_at   timestamptz           -- NULL = not synced to server yet
-- RLS: juez can only INSERT/UPDATE WHERE juez_id = auth.uid()
-- RLS: admin can SELECT/UPDATE all
```

---

## SUERTES — SCORING RULES

```
Suerte                  Type        Range   Step  Notes
──────────────────────────────────────────────────────────────────────
cala_de_caballo         individual  0–10    0.5   Horse maneuver quality
piales_en_lienzo        individual  0–10    0.5   Successful piales + time
coleadero               individual  0–10    0.5   Knockdown + style + time
jineteada_de_toro       individual  0–10    0.5   Time + style on bull
terna_en_ruedo          team        0–30    0.5   3 sub-scores: piales(0-10) + derribo(0-10) + floreo(0-10)
jineteo_de_yegua        individual  0–10    0.5   Time + style on mare
manganas_a_pie          individual  0–10    0.5   Successful mangana (on foot)
manganas_a_caballo      individual  0–10    0.5   Successful mangana (on horse)
paso_de_la_muerte       individual  0–10    0.5   Time + execution
```

---

## MODULES

### 1. auth
- Judge login: 4–6 digit PIN → `supabase.auth.signInWithPassword` (email derived from `juez.id@charroapp.internal`, password = PIN)
- Admin login: email + password
- Session persisted in `@capacitor/preferences` for offline access
- Hook: `useAuth()` → `{ user, role, isLoading }`

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

### 4. sync
- File: `src/lib/sync.ts`
- `syncPendingCalificaciones()`: reads Dexie dirty records → upsert Supabase → mark synced
- `useSyncStatus()` hook: `{ isOnline, pendingCount, lastSyncedAt }`
- Offline indicator banner in layout when `!isOnline || pendingCount > 0`

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
│   ├── features/
│   │   ├── auth/
│   │   │   ├── LoginJuez.tsx      # PIN pad
│   │   │   ├── LoginAdmin.tsx     # email+pass
│   │   │   └── useAuth.ts
│   │   ├── torneos/
│   │   │   ├── TorneoList.tsx
│   │   │   ├── TorneoForm.tsx
│   │   │   └── useTorneo.ts
│   │   ├── calificacion/
│   │   │   ├── SuerteRouter.tsx
│   │   │   ├── CalificacionForm.tsx
│   │   │   └── suertes/
│   │   │       ├── CalaDeCalaballo.tsx
│   │   │       ├── PialesEnLineo.tsx
│   │   │       ├── Coleadero.tsx
│   │   │       ├── JineteadaDeToro.tsx
│   │   │       ├── TernaEnRuedo.tsx
│   │   │       ├── JineteoDeYegua.tsx
│   │   │       ├── ManganasPie.tsx
│   │   │       ├── ManganasCaballo.tsx
│   │   │       └── PasoDeLaMuerte.tsx
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
│   └── migrations/
│       ├── 001_init.sql
│       └── 002_rls.sql
├── capacitor.config.ts
├── vite.config.ts
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

// db.ts
const db = new Dexie('CharroApp')
db.version(1).stores({
  calificaciones: '++id, participacion_id, juez_id, synced',
  participaciones: 'id, torneo_id, suerte',
  torneos: 'id',
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

```
[ ] Offline: disable network → score suerte → Dexie has record with synced=0
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

---

## MVP SCOPE (Phase 1) — EXCLUDED FROM PHASE 2

```
Phase 1 (MVP):
  ✓ Auth (PIN + admin email)
  ✓ Create torneo + equipos + charros
  ✓ Score all 9 suertes (offline-first + sync)
  ✓ Admin dashboard (realtime)
  ✓ Public display /display

Phase 2 (deferred):
  ✗ PDF/Excel export
  ✗ Bulk CSV import for charros
  ✗ Audit log for score corrections
  ✗ Historical torneo stats per charro
  ✗ Push notifications for judges
  ✗ Escaramuza charra scoring module
```
