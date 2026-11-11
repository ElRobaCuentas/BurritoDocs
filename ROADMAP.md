# Roadmap — El Burrito

## 1. Propósito y límites

Fuente principal de planificación posterior al MVP del Ecosistema El Burrito.
`../MVP.md` conserva el alcance, estado, pendientes de cierre, criterios de
aceptación y exclusiones del MVP. Aquí viven las dependencias y estados del
trabajo futuro, las tareas postergadas y las condiciones de pre-release.

La planificación post-MVP, las postergaciones, las obligaciones de publicación
y las ideas sin aprobación de ejecución son categorías distintas. El historial
y la evidencia de tareas completadas están en `DECISIONS.md`, `BUGS_RESUELTOS/`
y Git; no se repiten como backlog.

## 2. Planificación post-MVP

Los identificadores y el orden de las filas no establecen prioridades ni una
secuencia numérica obligatoria: cada tarea espera solo sus dependencias.
Se conserva la condición de ejecución ya documentada: cada fase se ejecuta una
a la vez; la deuda técnica puede avanzar en paralelo.

**Prerrequisitos completados:** T3.1 implementó el listener multi-bus; T3.2
validó el pipeline con un Motorola, caminata y bus, Firebase Console y El Burrito.
T4.1 mantiene el timestamp fresco cada 8 s (ADR-019), dependía de T3.2;
C4.6 oculta buses sin actualización a los 30 s y dependía de T4.1.
La propuesta histórica de caducidad a los 60 s no es trabajo pendiente.

| Tarea | Proyecto | Estado | Depende de | Alcance y validación |
|-------|----------|--------|------------|---------------------|
| T4.2 — Control de turnos y creación de `/recorridos` | BurritoDriverApp | Parcial | T4.1 | INICIAR/DETENER ya existen; falta registrar `/recorridos` (FIREBASE_SCHEMA §7). No equivale a los turnos informativos de I08 (ADR-027). |
| T4.4 — Render multi-bus completo | El Burrito | Pendiente | T3.2 | ShapeSource + SymbolLayer por cada bus activo. |
| T4.5 — Monitor de flota | AdminWeb | Pendiente | T4.4 | Sección «Flota en servicio ahora». |
| T4.6 — Estadísticas de recorridos | El Burrito | Pendiente | T4.2 | StatsScreen con métricas de recorridos. |
| T4.3 — Geofencing | BurritoDriverApp | Pendiente | T4.2 | Punto Cero Dinámico (ADR-013), máquina de estados con Haversine e histéresis 40/80 m. La validación real depende de la implementación; ajustar radios contra multipath de edificios en campus. |
| T5.2 — Smoothing | El Burrito | Pendiente | T4.4 | Interpolación (moving average / Kalman ligero). El ajuste fino depende de la implementación y de datos reales del circuito en campus. |
| T5.1 — Calibración de paraderos | El Burrito | Pendiente | Presencia en Campus UNMSM | Coordenadas GPS reales de los 10 paraderos. Sin otra dependencia funcional documentada. |

**Entornos de trabajo:** Bloque A (Casa/Calle) = implementación y pruebas GPS
locales con un Motorola, BurritoDriverApp en segundo plano y El Burrito en primer
plano. Bloque B (Campus UNMSM) = validación física en universidad y bus para
T4.3, T5.1 y ajuste fino de T5.2. Esta distinción no añade dependencias entre
tareas independientes.

**calculateDistance():** Haversine está en `User/src/features/map/utils/geo.ts`,
con 6 tests unitarios (T2.2), disponible para T4.3 y T5.2.

## 3. Tareas postergadas

| Tarea | Decisión vigente | Referencia |
|-------|------------------|------------|
| I09 — Autorización GPS por asignación | Postergada; no condiciona I03/I08. | FIREBASE_SCHEMA §9 |
| I10 — Heartbeat/antigüedad del fix GPS | Postergada; conservar comportamiento aprobado. | ARCHITECTURE §8.3 |
| P06 — Perfil incompatible | Postergada; no reabrir automáticamente. La fuente original `correciones_piloto_pruebas.md` fue retirada; no es un enlace vigente. | Decisión preservada del registro anterior de ROADMAP.md (Git). |
| Mantenimiento — Eliminar registro legacy `com.burritouserapp` (`b9b8149c`) en Firebase Console | No bloqueante; limpieza post-MVP. No afecta T1. | Firebase Console |

## 4. Condiciones de pre-release y publicación

Estas entradas conservan sus alcances y condiciones originales. Relacionarlas
no las convierte en requisitos idénticos ni permite dar una por cumplida con la otra.

| Tarea | Alcance | Condición original |
|-------|---------|--------------------|
| T1.2 — Eliminar Debug Panel | Panel concreto de El Burrito: `User/src/features/map/components/Map.tsx`, `SHOW_DEBUG_PANEL` y bloque condicionado por `__DEV__` (líneas 32 y 231 al corte). Está desactivado, no eliminado. | Congelada por orden del desarrollador hasta pre-producción. |
| C2 — Ocultar debug panels en producción | Paneles de diagnóstico de producción; no exige por sí sola eliminar el código de T1.2. | Antes de la release candidate. Queda fuera de las dependencias de T1. |
| T2.3 — Asegurar credenciales de firma | Configuración de `gradle.properties` en ambas apps; no equivale por sí sola a rotar el keystore. | Diferida hasta empaquetar v1.2.0. |
| S3 — Rotar keystore de release y contraseña de `gradle.properties` | Rotación de clave y contraseña, distinta de asegurar su configuración (T2.3). | Antes de publicar en Play Store. |
| W12 — `toggleChoferStatus` no limpia `choferes_uids` | Hoy `activo:false` NO revoca la autorización de escritura RTDB. La semántica aceptada del MVP es elegibilidad futura, no revocación GPS en caliente (ADR-024). | **Revisar antes de producción**; no implica aprobar una revocación inmediata descartada para el MVP. |

**Discrepancia previa de S3:** se conserva la exigencia de rotación antes de
Play Store, pero `TROUBLESHOOTING.md` §10 registra que no se ha decidido rotar
la clave y que se reevaluará ante evidencia de compromiso o cambios de riesgo.
Esta redistribución no resuelve esa contradicción ni da por autorizada o
cumplida la rotación; conserva la condición de Play Store y deja la discrepancia
pendiente de decisión expresa.

## 5. Deuda técnica sin condición de publicación documentada

| Ítem | Nota original |
|------|---------------|
| App Check | Reduce riesgo de abuso. |
| Typecheck en DriverApp | Configurar `tsc` en Driver; no bloquea la operación del MVP. |
| Limpieza de `console.log`, `@ts-ignore` y `as any` | Deuda técnica. |
| Backoff para reconexión RTDB (`goOffline`/`goOnline` por AppState) | Tolerancia adicional a gaps. |

No se les asigna una fecha, obligación de publicación o prioridad nueva.
El inventario de lint y sus condiciones de revisión están en `LINT_DEBT.md`.

## 6. Visión futura sin aprobación de ejecución

No forma parte del backlog activo. Se distinguen las ADR planeadas de las
ideas sin ADR; ninguna tiene prioridad ni fecha asignada.

| Iniciativa con ADR planeada | Alcance | Dependencia registrada |
|----------------------------|---------|------------------------|
| ADR-014 — Backend propio | API REST/WebSocket y lógica centralizada (Node.js o similar). | — |
| ADR-015 — Dashboard web de flota | Métricas en vivo y analítica ampliada. AdminWeb ya existe; T4.5 es su monitor de flota, no todo el alcance de ADR-015. | ADR-014. ADR-015 contempla RTDB o el backend cuando exista. |

**Ideas sin ADR:**

- API pública para integración con servicios externos.
- Analytics avanzados: tiempos de espera, ocupación y rutas más usadas.
- Notificaciones push a estudiantes sobre llegada de buses.
- Soporte iOS completo del ecosistema; BurritoDriverApp no está implementada
  ni probada para iOS. No se acredita soporte completo por la configuración
  nativa existente de El Burrito.
