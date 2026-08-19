# CharroApp — Resumen para cliente / patrocinador

> Este documento resume el proyecto en términos de alcance, tiempos, costos y riesgos, sin
> detalle técnico. La versión para el equipo de desarrollo (decisiones y su porqué) es
> `plan.md`; el plan operativo detallado (para ejecución paso a paso) es `plan_llm.md`.

## Qué es CharroApp

Una aplicación para calificar charreadas en tiempo real. Los jueces capturan las
calificaciones desde tablet o celular directamente en el lienzo, aunque no haya internet en
ese momento — el sistema sincroniza automáticamente en cuanto vuelve la conexión. Los
resultados se muestran en una pantalla pública para el público, actualizándose al instante
conforme los jueces califican. Cubre las 9 suertes oficiales reconocidas por la Federación
Mexicana de Charrería.

---

## Alcance por etapas

El proyecto se construye en 4 etapas, cada una con un resultado concreto y demostrable —
no se espera hasta el final para ver algo funcionando:

| Etapa | Qué se puede hacer al terminarla |
|---|---|
| **Etapa 0 — Base del sistema** | El sistema existe y funciona: un administrador y los jueces pueden entrar a la aplicación (incluso ya en un dispositivo real). Todavía no se califica nada — es la cimentación técnica. |
| **Etapa 1 — Primera suerte funcionando de punta a punta** | Se puede correr un torneo de prueba real calificando **una suerte** (`cala de caballo`) completa: los jueces califican sin internet, el resultado sincroniza solo, y el tablero público y el panel de administración se actualizan en vivo. Este es el primer punto donde el sistema es útil en un evento real, aunque limitado a una sola suerte. |
| **Etapa 2 — Reglamento completo** | Las 9 suertes oficiales quedan calificables, incluyendo las que requieren cronómetro y la suerte de equipo (`terna en el ruedo`). El sistema queda listo para operar cualquier charreada completa. |
| **Etapa 3 — Mejoras posteriores** | Funciones adicionales que no son necesarias para operar un torneo, pero suman valor: exportar resultados, historial de charros entre torneos, notificaciones a jueces, y la escaramuza charra (que tiene reglas propias). Se prioriza después de tener el sistema base en uso real. |

---

## Cronograma estimado

Estimación inicial en **semanas relativas** desde el arranque del proyecto — no se fijan
fechas de calendario porque todavía no hay fecha de inicio confirmada, y el ritmo real puede
ajustar estos números una vez que se complete la Etapa 0 (primer punto donde se tiene
evidencia real de la velocidad de avance):

```
Semana:        1    2    3    4    5    6    7    8    9    10   11+
Etapa 0        ███████████
Etapa 1                    ███████████
Etapa 2                                ████████████████████
Etapa 3                                                     (backlog, sin fecha fija)
```

- **Etapa 0**: semanas 1–3
- **Etapa 1**: semanas 4–6 — punto en el que el sistema ya es útil para un torneo real (una sola suerte)
- **Etapa 2**: semanas 7–10 — reglamento completo
- **Etapa 3**: posterior al lanzamiento, priorizada según lo que pida el uso real

Estos números son un punto de partida razonable dado el alcance, no un compromiso contractual
de fechas — se recomienda revisarlos con el equipo de desarrollo al cerrar la Etapa 0.

---

## Costos estimados

Este documento cubre únicamente **costos de infraestructura** (no incluye el costo de las
horas de desarrollo, por no tener aún definida la tarifa/tamaño del equipo — se puede agregar
esa sección cuando se defina):

| Concepto | Costo aproximado | Cuándo aplica |
|---|---|---|
| Backend (Supabase) | US$0/mes | Durante desarrollo y pruebas (Etapas 0–2) |
| Backend (Supabase, plan pagado) | ~US$25/mes | Obligatorio antes del primer torneo real — el plan gratuito no incluye respaldos automáticos ni evita que el sistema se pause por inactividad |
| Hosting del panel web / tablero público | US$0/mes | Cubierto por el plan gratuito de los proveedores típicos (Vercel/Netlify) para este volumen de tráfico |
| Publicación en App Store (si se decide app nativa iOS) | ~US$99/año | Solo si se publica en la tienda de Apple — decisión pendiente, no forma parte del alcance actual |
| Publicación en Google Play (si se decide app nativa Android) | ~US$25 pago único | Solo si se publica en la tienda de Google — decisión pendiente |

El costo mensual recurrente relevante para operar el sistema en producción es de
aproximadamente **US$25/mes** (backend). Los costos de tienda de apps solo aplican si se
decide distribuir la app fuera de un navegador — el sistema funciona igual de bien como
aplicación web instalable sin pasar por ninguna tienda.

---

## Riesgos principales

- **Conectividad inestable en el lienzo**: es la razón de ser del diseño "sin internet
  primero" del sistema — los jueces pueden calificar sin conexión y el sistema sincroniza
  solo apenas hay señal. Este riesgo está mitigado por diseño, no es un pendiente.
- **Dependencia de un solo proveedor de backend** (Supabase): decisión deliberada para no
  duplicar esfuerzo manteniendo un servidor propio — el costo de cambiar de proveedor más
  adelante existe, pero es bajo comparado con el ahorro de tiempo de desarrollo.
- **Pruebas en dispositivos reales antes de un torneo real**: el sistema debe probarse en
  tablets/celulares reales, con conectividad intermitente real, no solo en simulador —
  parte obligatoria antes de considerar cualquier etapa lista para producción.
- **Ventana de preparación antes de producción**: hay un paso obligatorio (pasar el backend
  a plan pagado, confirmar respaldos automáticos activos) que debe completarse antes del
  primer torneo real con jueces y resultados reales — no es opcional ni se puede saltar para
  ahorrar costo en el primer evento.

---

## Qué no cubre este documento

Aquí no se detalla cómo está construido el sistema (tecnologías, base de datos, seguridad) —
esa información vive en `plan.md`, pensado para el equipo de desarrollo. Este documento se
actualiza cada vez que cambie el alcance, el cronograma o los costos estimados.
