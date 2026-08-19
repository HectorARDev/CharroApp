# Plan: Sistema de Calificación en Tiempo Real para Charrería

> Este documento es la versión para lectura humana: decisiones y su porqué, sin el detalle
> de ejecución paso a paso. El plan operativo completo (sub-fases, archivos exactos,
> migraciones SQL literales, criterios de aceptación) vive en `plan_llm.md`, pensado para
> que un agente/LLM lo siga sin ambigüedad. Cuando algo cambie, se actualiza primero
> `plan_llm.md` y este documento se resincroniza después — evitar editarlos por separado.
> Para cliente/patrocinador (alcance por hitos, cronograma y costos, sin jerga técnica) ver
> `plan_cliente.md`.

## Contexto

Los jueces de charrería califican manualmente durante eventos deportivos con conectividad intermitente en lienzos charros. El objetivo es digitalizar ese proceso con una app que funcione sin internet (offline-first), sincronice los puntajes en tiempo real cuando haya conexión, y muestre resultados en un tablero público. Cubre las 9 suertes oficiales de la Federación Mexicana de Charrería.

---

## Alcance por fases

El plan original agrupaba "calificar las 9 suertes" entero en una sola fase de MVP — en la práctica eso no es un MVP, es la app completa. Se reestructuró en 4 fases para aislar el riesgo técnico antes de invertir en cobertura completa del reglamento:

| Fase | Qué incluye | Objetivo |
|---|---|---|
| **0 — Fundación** | Scaffolding, Supabase (schema+RLS), auth, Dexie, Capacitor, CI. Cero lógica de calificación. | El esqueleto compila, despliega, y un admin/juez pueden loguearse. |
| **1 — MVP** | Auth completo + torneo mínimo + **una sola suerte** (`cala_de_caballo`) calificable offline-first+sync+realtime + dashboard admin básico + `/display` básico | Probar que TODO el pipeline (Dexie → sync → Supabase → Realtime → Admin/Display) funciona de punta a punta en dispositivo real, con el menor riesgo posible |
| **2 — Cobertura completa** | Las 8 suertes restantes (incluye cronómetro compartido y el caso especial de equipo `terna_en_ruedo`) | Cubrir el reglamento completo de la FMCH |
| **3 — Extras** | Export, CSV import, auditoría, históricos, push notifications, escaramuza | Todo lo que no es crítico para operar un torneo |

**¿Por qué `cala_de_caballo` para el MVP?** Es la suerte que tradicionalmente abre toda charreada (alta relevancia simbólica) y, bajo el esquema de puntaje ya definido, no requiere cronómetro ni lógica de equipo — es la suerte más simple que aún valida el pipeline completo con el menor riesgo posible. Se descartó elegir por popularidad pura (`coleadero`, `jineteada_de_toro`) porque ambas necesitan un componente de cronómetro que hubiera inflado el alcance del MVP sin aportar validación adicional de la arquitectura.

Fase 0 tiene, además, su propio desglose interno en 6 sub-fases (scaffolding, routing, Supabase, auth, Dexie+Capacitor, CI) con checkpoints y criterios de aceptación propios — ese nivel de detalle, junto con 8 casos de uso de infraestructura que lo validan (login, seguridad RLS, sesión offline, primer arranque nativo, etc.), vive en `plan_llm.md`.

---

## Stack Tecnológico

Cada tecnología fue revisada explícitamente contra alternativas antes de fijarse — no es solo "lo que se ocurrió primero".

### Frontend / App
| Tecnología | Rol | Por qué |
|---|---|---|
| **React 18/19 + TypeScript** | UI única para web y app | Mejor encaje con el resto del stack ya elegido: shadcn/ui nació para React, TanStack Query/Router y react-hook-form son React-first. Solid/Svelte/Vue no aportan ventaja suficiente para perder ese ecosistema. |
| **Vite** | Build tool | Estándar actual para SPAs: HMR rápido, mejor plugin de PWA/Workbox, soporte amplio para Capacitor. |
| **Capacitor** | Wrapper nativo iOS/Android | Única opción madura que cumple "un solo codebase" para web+iOS+Android sin reescritura. React Native forzaría un codebase nativo separado; Tauri Mobile aún es inmaduro para producción en un evento en vivo sin margen de error. |
| **React Router** (modo Declarative, `HashRouter`) | Routing | Se prefirió sobre TanStack Router por más ejemplos específicos para apps híbridas Capacitor (hash routing en WebView nativo, deep-linking). `HashRouter` (no `BrowserRouter`) porque el WebView de Capacitor sirve assets desde un scheme local, donde el routing por History API necesita configuración de servidor que no aplica ahí. |
| **Tailwind CSS v4 + shadcn/ui** (modo variables CSS) | Estilos | Componentes accesibles (Radix) y totalmente editables — clave para el requisito de botones ≥60px/alto contraste. El modo de variables CSS deja un tono base único definido desde el día 1, con la arquitectura ya lista para cambiarlo después sin tocar componentes (no se construye un selector de color visible al usuario todavía — eso queda como candidato de Fase 3). |
| **Zustand** | Estado global | API mínima, sin boilerplate, funciona bien offline-first. Redux Toolkit sería sobre-ingeniería para el tamaño real del estado de esta app. |
| **TanStack Query** | Data fetching + cache | Su rol es exclusivamente para lecturas no-críticas que toleran un modelo red-primero-cache-después (listados de torneos, dashboard admin) — **nunca** para el camino crítico de escritura de calificaciones, que pasa por Dexie + el sync engine propio. Mezclar ambas responsabilidades es la forma más común de introducir bugs de datos duplicados/perdidos. |
| **Vitest + Testing Library + Playwright** | Testing | Ausente del plan original, pese a ser una app donde un bug de sync/conflicto en vivo es costoso (evento en curso, sin segunda oportunidad). Playwright permite simular offline para probar el flujo completo sin depender solo de pruebas manuales. |
| **ESLint + Prettier** | Calidad de código | Hueco menor del plan original, cerrado junto con el resto del scaffolding en Fase 0. |

### Almacenamiento Offline
| Tecnología | Rol | Por qué |
|---|---|---|
| **Dexie.js** | Wrapper TypeScript de IndexedDB — almacén local principal | TS-first, API madura y simple, suficiente para el modelo de conflicto elegido (last-write-wins por fila, cada juez dueño de sus propias filas vía RLS). RxDB se descartó por resolución de conflictos más sofisticada de la que hace falta, con plugins clave bajo licencia comercial de pago. |
| **@capacitor/preferences** | Configuración pequeña (token, juez_id, rol) | Bien acotado a datos pequeños — el volumen real de datos (calificaciones) vive en Dexie. |
| **@capacitor/network** | Detección de conectividad en nativo | `navigator.onLine`/evento `online` es conocido por no ser confiable en WebView de iOS/Android al recuperar señal real — riesgo directo para la promesa de offline-first. Con fallback a la API del navegador en el build web. |

### Backend
| Tecnología | Rol | Por qué |
|---|---|---|
| **Supabase** | PostgreSQL + Auth + Realtime + Storage | El schema es relacional (FKs entre torneos/equipos/charros/participaciones/calificaciones) y Fase 3 pide reportes/export — eso pide Postgres, no un modelo de documentos (se descartó Firebase/Firestore por esa razón). Se eligió deliberadamente para *evitar* mantener un backend propio — no hay plan de introducir una API custom (Node/Express) además de Supabase; la única pieza de "servidor" necesaria (crear jueces con service role) se resuelve con una Edge Function. |
| **Hosting: free tier desde el día 1** (Fase 0–2) | — | Los límites duros (500MB DB, 5GB bandwidth/mes, 200 conexiones Realtime) están muy por encima de lo que esta app necesitará. El riesgo real no son las cuotas: auto-pausa tras 7 días de inactividad (trampa para una app de uso periódico) y sin backups automáticos. Por eso hay un gate explícito: **subir a plan Pro antes del primer torneo real en producción**. |
| **Row Level Security (RLS)** | Cada juez solo edita sus propias calificaciones; lectura pública sin auth para `/display` | Políticas verificadas manualmente antes de construir UI encima — un hueco de RLS descubierto tarde obliga a re-probar todo lo construido sobre él. |

### PWA / Service Worker
| Tecnología | Rol | Por qué |
|---|---|---|
| **Workbox (vía vite-plugin-pwa)** | Cacheo de app-shell | Sirve únicamente al despliegue *web* (el build de Capacitor ya carga assets empaquetados localmente). Debe limitarse a precachear JS/CSS/HTML — **nunca** interceptar llamadas a Supabase, ese trabajo ya lo hace Dexie. Mezclar ambas capas de cacheo es la causa más común de bugs de "datos viejos" en offline-first. |

### Alternativas descartadas (resumen — detalle completo en `plan_llm.md`)
Firebase/Firestore, Dexie Cloud, RxDB, PowerSync/ElectricSQL, React Native, Tauri Mobile, Redux Toolkit y TanStack Router se evaluaron y descartaron explícitamente — cada una por una razón puntual documentada, no por default.

---

## Arquitectura

```
Juez (tablet/phone)
  └─ React app (Capacitor)
       ├─ Dexie.js (IndexedDB) ← califica aquí offline
       └─ Sync layer (background)
            ├─ online → push a Supabase
            └─ offline → cola de cambios pendientes en Dexie

Supabase (servidor)
  ├─ PostgreSQL (fuente de verdad)
  └─ Realtime channels
       ├─ Panel Admin (web)
       └─ Tablero Público (pantalla del evento)
```

**Flujo offline-first:**
1. Juez abre app → selecciona su nombre de una lista (jueces del torneo activo, lectura pública sin auth) → ingresa su PIN, que **es** el password real de su cuenta en `auth.users` (no se guarda un hash de PIN por separado — evita una segunda fuente de verdad que se puede desincronizar)
2. Juez califica → registro guardado en Dexie inmediatamente (optimistic)
3. `navigator.onLine`/`@capacitor/network` + Supabase Realtime `onConnect` → dispara sync
4. Sync: registros con `synced: false` se upsert a Supabase; conflictos resueltos por `last_write_wins` (timestamp) — suficiente porque cada juez solo escribe sus propias filas
5. Admin y tablero reciben update vía Supabase Realtime

**Decisión de arquitectura crítica — modelo de sesión offline**: escribir en Dexie **nunca depende de tener una sesión de Supabase válida**, solo del `juez_id` ya cacheado localmente desde el último login exitoso. El access token de Supabase expira (~1h) y en un evento de varias horas sin red no se puede refrescar; si calificar dependiera de una sesión válida, la promesa de "offline-first" se rompería justo cuando más se necesita. La sesión solo importa al momento de sincronizar — si no hay red para refrescarla, el sync reintenta cuando vuelva la conexión, sin bloquear al juez. El primer login sí requiere conectividad siempre (no hay forma de resolver PIN→identidad sin red la primera vez); esto es una restricción aceptada, no un defecto.

**El proyecto sigue siendo un solo codebase**, no 3 apps separadas — admin/juez/backend son áreas funcionales dentro del mismo repo, con `React.lazy()` por ruta para que el bundle nativo del juez no cargue el JS del dashboard admin, sin la complejidad de un monorepo.

---

## Módulos del Proyecto

### 1. Auth
- Login de juez en 2 pasos: selección de nombre (lista del torneo activo) → PIN pad. Evita depender de un identificador críptico y resuelve el problema de que un PIN por sí solo no identifica a la persona.
- Login email/contraseña para admin.
- Roles: `admin`, `juez`. La alta de jueces siempre pasa por una función de servidor (Edge Function con permisos elevados) que crea la cuenta y la fila del juez en una sola operación — nunca se inserta un juez directo en la tabla pública.
- Reseteo de PIN (Fase 1): un juez olvidando su PIN en cancha es un escenario probable en un evento real; se resuelve con una Edge Function `reset-pin` (mismo patrón que la de alta), en vez de dejarlo como reseteo manual ad-hoc.

### 2. Gestión de Torneos
- Crear torneo, agregar equipos/charros.
- Programar orden de participación por suerte.
- Solo puede existir **un torneo activo a la vez** (garantizado por constraint de base de datos) — de eso depende que `/display` y la selección de juez no queden ambiguos.

### 3. Motor de Calificación (core)
- Interfaz específica por suerte con sus criterios oficiales.
- Botones grandes (>60px) para uso en tablet con guantes.
- El router de suertes se diseña desde el inicio contemplando las 9, aunque en el MVP (Fase 1) solo una tiene lógica real y el resto son placeholders — evita re-arquitecturar en Fase 2.

**Las 9 suertes y sus esquemas de puntaje:**
| Suerte | Rango | Unidad | Cronómetro | Fase |
|---|---|---|---|---|
| Cala de caballo | 0–10 | 0.5 | No | **1 (MVP)** |
| Piales en el lienzo | 0–10 | 0.5 | Sí | 2 |
| Coleadero | 0–10 | 0.5 | Sí | 2 |
| Jineteada de toro | 0–10 | 0.5 | Sí | 2 |
| Terna en el ruedo (equipo) | 0–30 | 0.5 | No | 2 |
| Jineteo de yegua | 0–10 | 0.5 | Sí | 2 |
| Manganas a pie | 0–10 | 0.5 | No | 2 |
| Manganas a caballo | 0–10 | 0.5 | No | 2 |
| Paso de la muerte | 0–10 | 0.5 | Sí | 2 |

5 de las 9 suertes comparten la necesidad de un componente de cronómetro (se construye una sola vez en Fase 2, no 5 veces). `terna_en_ruedo` es la única de equipo y la única con 3 sub-puntajes en vez de 1 (se guardan en una columna `detalle` además del total, para que el desglose sea auditable y no se confíe ciegamente en un total calculado en el cliente).

### 4. Sync Engine
- Capa de abstracción sobre Dexie que marca registros `synced: false`.
- Worker en background que sincroniza cuando `online`, con protección contra sincronizar dos veces en paralelo.
- `useNetworkStatus()` / `useSyncStatus()` para UI de "modo offline".

### 5. Dashboard Admin
- Vista en tiempo real de todas las calificaciones activas.
- Edición/corrección con log de auditoría (Fase 3).
- Exportar resultados (Fase 3).

### 6. Tablero Público
- Ruta `/display` — pantalla grande, sin controles, sin autenticación (protegido por una política de solo-lectura, no por ausencia de rutas).
- Auto-refresh vía Supabase Realtime.
- Muestra ranking en vivo, suerte actual, últimas puntuaciones.

---

## Estructura de Carpetas

```
CharroApp/
├── src/
│   ├── routes/              # HashRouter + NotFound
│   ├── layouts/
│   ├── features/
│   │   ├── auth/            # selección de juez, PIN, login admin, rutas protegidas
│   │   ├── torneos/
│   │   ├── calificacion/
│   │   │   └── suertes/     # un componente por suerte
│   │   ├── sync/
│   │   ├── dashboard/
│   │   └── tablero/
│   ├── db/                  # Dexie schema + instancia
│   ├── lib/                 # cliente Supabase + sync engine
│   └── hooks/
├── supabase/
│   ├── migrations/          # SQL versionado
│   ├── seed.sql             # datos de prueba reproducibles
│   └── functions/           # Edge Functions (ej. alta de jueces)
├── .github/workflows/       # CI: lint + test + build en cada push
├── capacitor.config.ts
└── vite.config.ts
```

---

## Base de Datos (Supabase / PostgreSQL)

Tablas principales:
- `torneos` — eventos. Solo uno puede tener `estado='activo'` a la vez (constraint de base de datos, no solo convención).
- `equipos` — equipos participantes.
- `charros` — participantes individuales.
- `participaciones` — charro (o equipo) × suerte × torneo.
- `calificaciones` — puntuación, `juez_id`, `participacion_id`, `updated_at`/`synced_at` (control de sync), `detalle` (sub-puntajes para suertes de equipo).
- `jueces` — perfil + rol + torneo asignado. Sin columna de PIN propia: el PIN es directamente el password de la cuenta de autenticación asociada.

RLS: cada juez solo lee/escribe sus propias calificaciones; el admin tiene acceso completo (verificado con una función `security definer` — comprobar el rol admin con una subconsulta directa sobre `jueces` deja al propio admin bloqueado de su tabla, porque esa subconsulta también queda sujeta a RLS); existe una política de **lectura pública** (torneos, participaciones, calificaciones y jueces del torneo activo) que es lo que hace posible tanto `/display` como la pantalla de selección de juez antes de loguearse — esa política aplica tanto a usuarios anónimos como a jueces ya autenticados (son roles de Postgres distintos; limitarla solo a anónimos deja a un juez logueado sin poder leer el torneo que necesita calificar). RLS debe habilitarse explícitamente tabla por tabla antes de que cualquier política tenga efecto — es el paso que más fácil se olvida porque, si falta, no da error: simplemente deja todo abierto.

---

## Verificación (Plan de pruebas)

1. **Offline**: Desactivar red → calificar → verificar en Dexie → reconectar → verificar que aparece en Supabase.
2. **Sesión offline**: con el access token ya vencido y sin red, calificar debe seguir funcionando sin error — es la prueba que valida la promesa central del proyecto.
3. **Tiempo real**: Admin dashboard + tablero abiertos → un juez envía calificación → ambas pantallas se actualizan en menos de 1 segundo.
4. **Seguridad**: un juez no puede leer calificaciones de otro juez; un anónimo puede leer `/display` pero no escribir nada — probado manualmente antes de construir UI encima, no después.
5. **Conflictos**: dos jueces envían puntaje diferente para el mismo participante → last_write_wins resuelve correctamente.
6. **App nativa**: `npx cap run android` / `npx cap run ios` → flujo completo funcional, incluida la detección de conectividad real.
7. **Accesibilidad**: botones mínimo 60px, funciona con dedos gruesos/guantes.
8. **CI**: un error de lint o de test introducido a propósito bloquea el push antes de llegar a `main`.

**Gate antes de producción real**: subir Supabase de free tier a plan Pro (backups + sin auto-pausa) antes del primer torneo real con jueces reales — no antes de eso, para no pagar de más durante desarrollo.

---

## Fase 0 en detalle (resumen — ver `plan_llm.md` para el desglose completo)

Fase 0 se dividió en 6 sub-fases, cada una con su propio criterio de aceptación:

1. **Scaffolding** — proyecto, lint/format, tema visual base, estructura de carpetas, testing, `vite-plugin-pwa` configurado para precachear solo el app-shell (nunca llamadas a Supabase).
2. **Routing + layout** — rutas de las 4 áreas, protección de rutas (esqueleto), code-splitting. `/login` resuelve selección de juez + PIN como un solo flujo (el PIN es un estado interno, no una ruta aparte); `/login/admin` es la ruta separada para el login de administrador. La protección de rutas cubre tanto `/admin/*` como `/calificar/:participacionId` — solo `/display` es intencionalmente pública.
3. **Supabase** — proyecto (en la región disponible más cercana a México, por latencia de Realtime en un evento en vivo), schema, políticas de seguridad (con su propio checkpoint de verificación antes de continuar — ver la nota de RLS más abajo), datos de prueba reproducibles. El seed de prueba se arma en dos piezas: los datos de negocio (torneo/equipos/charros) vía SQL, y el admin+juez de prueba vía un script que usa la Admin API de Supabase — igual que `auth.users` no se puede poblar de forma confiable con un `INSERT` directo, se necesita el mismo mecanismo que después usa la función de alta de jueces en Fase 1.
4. **Auth** — el hook de autenticación y su contrato de sesión offline (login/registro retornan `{ error }`, no lanzan excepción — un PIN equivocado es un resultado esperado, no algo excepcional), login admin, login juez (el checkpoint más crítico de toda la fase), integración de rutas protegidas.
5. **Dexie + Capacitor nativo** — almacén local y primera corrida en simulador. Se fija explícitamente una versión mínima de OS/SDK (Android/iOS) acorde al hardware real que suele usarse en un lienzo charro — no es un dominio con dispositivos de último modelo.
6. **CI básico + hosting web** — CI que realmente bloquee, no solo que exista, usando la misma versión de Node que el resto del equipo (fijada en `.nvmrc`); además la decisión de dónde se despliega el build web (`dist/`) para que admin y `/display` sean accesibles desde una URL real, no solo `localhost`, con sus propias variables de entorno configuradas en el host (apuntando al proyecto Supabase de cada developer).

**Onboarding**: cada developer crea su **propio** proyecto Supabase (gratis) — no se comparte un proyecto de equipo entre todos. El flujo real de arranque es clonar el repo, crear el proyecto, aplicar migraciones y correr el seed, no solo `npm install && npm run dev`.

Se validó con 8 casos de uso de infraestructura (no de negocio todavía) — el más importante es "un juez reabre la app sin red, horas después de su último login, con el token ya vencido, y puede seguir calificando sin error": ese es el caso de uso que demuestra que la arquitectura cumple su promesa central antes de construir nada más encima.
