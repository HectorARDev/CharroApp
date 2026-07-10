# Plan: Sistema de Calificación en Tiempo Real para Charrería

## Contexto

Los jueces de charrería califican manualmente durante eventos deportivos con conectividad intermitente en lienzos charros. El objetivo es digitalizar ese proceso con una app que funcione sin internet (offline-first), sincronice los puntajes en tiempo real cuando haya conexión, y muestre resultados en un tablero público. Cubre las 9 suertes oficiales de la Federación Mexicana de Charrería.

---

## Stack Tecnológico Recomendado

### Frontend / App
| Tecnología | Rol | Por qué |
|---|---|---|
| **React 18 + TypeScript** | UI única para web y app | Un solo codebase, ecosistema maduro |
| **Vite** | Build tool | Rápido, soporte nativo de PWA con `vite-plugin-pwa` |
| **Capacitor 6** | Wrapper nativo iOS/Android | Convierte la web app en app nativa sin reescritura |
| **Tailwind CSS + shadcn/ui** | Estilos | Componentes accesibles, adaptables a tablet táctil |
| **Zustand** | Estado global | Ligero, funciona bien offline-first |
| **TanStack Query v5** | Data fetching + cache | Manejo de estados loading/error/offline |

### Almacenamiento Offline
| Tecnología | Rol |
|---|---|
| **Dexie.js** | Wrapper TypeScript de IndexedDB — almacén local principal |
| **@capacitor/preferences** | Configuración pequeña (token, usuario activo) |

### Backend
| Tecnología | Rol |
|---|---|
| **Supabase** | PostgreSQL + Auth + Realtime + Storage |
| **Supabase Realtime** | Canales de sincronización entre jueces y tablero |
| **Row Level Security (RLS)** | Cada juez solo edita sus propias calificaciones |

### PWA / Service Worker
| Tecnología | Rol |
|---|---|
| **Workbox (via vite-plugin-pwa)** | Cacheo de assets y API responses |

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
1. Juez abre app → carga datos del evento desde Supabase (si hay red) o Dexie (si no hay red)
2. Juez califica → registro guardado en Dexie inmediatamente (optimistic)
3. `navigator.onLine` event + Supabase Realtime `onConnect` → dispara sync
4. Sync: registros con `synced: false` se upsert a Supabase; conflictos resueltos por `last_write_wins` (timestamp)
5. Admin y tablero reciben update via Supabase Realtime

---

## Módulos del Proyecto

### 1. Auth
- Login por PIN (4-6 dígitos) para jueces en campo — evita teclado complejo
- Login email/contraseña para admin
- Roles: `admin`, `juez`, `espectador`

### 2. Gestión de Torneos
- Crear torneo, agregar equipos/charros
- Programar orden de participación por suerte

### 3. Motor de Calificación (core)
- Interfaz específica por suerte con sus criterios oficiales
- Botones grandes (>60px) para uso en tablet con guantes
- Validación de rango de puntos por suerte

**Las 9 suertes y sus esquemas de puntaje:**
| Suerte | Rango | Unidad | Notas |
|---|---|---|---|
| Cala de caballo | 0–10 | 0.5 | Puntos por calidad de trabajo |
| Piales en el lienzo | 0–10 | 0.5 | Por cada pial logrado, tiempo |
| Coleadero | 0–10 | 0.5 | Derribo, estilo, tiempo |
| Jineteada de toro | 0–10 | 0.5 | Tiempo + estilo |
| Terna en el ruedo | 0–30 | 0.5 | 3 maniobras: piales+derribo+floreo |
| Jineteo de yegua | 0–10 | 0.5 | Similar a jineteada |
| Manganas a pie | 0–10 | 0.5 | Por mangana lograda |
| Manganas a caballo | 0–10 | 0.5 | Por mangana lograda |
| Paso de la muerte | 0–10 | 0.5 | Tiempo y ejecución |

### 4. Sync Engine
- Capa de abstracción sobre Dexie que marca registros `synced: false`
- Worker en background que sincroniza cuando `online`
- `useNetworkStatus()` hook React para UI de "modo offline"

### 5. Dashboard Admin
- Vista en tiempo real de todas las calificaciones activas
- Edición/corrección con log de auditoría
- Exportar resultados (CSV/PDF)

### 6. Tablero Público
- Ruta `/display` — pantalla grande, sin controles
- Auto-refresh via Supabase Realtime
- Muestra ranking en vivo, suerte actual, últimas puntuaciones

---

## Estructura de Carpetas

```
/
├── src/
│   ├── features/
│   │   ├── auth/
│   │   ├── torneos/
│   │   ├── calificacion/
│   │   │   ├── suertes/        # componente por suerte
│   │   │   └── SuerteRouter.tsx
│   │   ├── sync/
│   │   ├── dashboard/
│   │   └── tablero/
│   ├── db/                     # Dexie schema + migrations
│   ├── lib/
│   │   ├── supabase.ts
│   │   └── sync.ts
│   └── hooks/
├── capacitor.config.ts
├── vite.config.ts
└── supabase/
    └── migrations/             # SQL migrations
```

---

## Base de Datos (Supabase / PostgreSQL)

Tablas principales:
- `torneos` — eventos
- `equipos` — equipos participantes
- `charros` — participantes individuales
- `participaciones` — charro × suerte × torneo
- `calificaciones` — puntuación, `juez_id`, `participacion_id`, `created_at`, `synced_at`
- `jueces` — perfil + rol + torneo asignado

---

## Verificación (Plan de pruebas)

1. **Offline**: Desactivar red → calificar una suerte → verificar en Dexie DevTools → reconectar → verificar que aparece en Supabase dashboard
2. **Tiempo real**: Admin dashboard + tablero abiertos → juez envía calificación → ambas pantallas se actualizan < 1s
3. **Conflictos**: Dos jueces envían puntaje diferente para el mismo participante → last_write_wins resuelve correctamente
4. **App nativa**: `npx cap run android` / `npx cap run ios` → flujo completo funcional
5. **Accesibilidad**: Botones mínimo 60px, funciona con dedos gruesos/guantes

---

## MVP Recomendado (Fase 1)

Para primera entrega funcional:
1. Auth (PIN + admin)
2. Crear torneo con participantes
3. Calificación de las 9 suertes (offline-first + sync)
4. Dashboard admin en tiempo real
5. Tablero público

Dejar para Fase 2: exportación PDF, historial de torneos anteriores, estadísticas por charro.
