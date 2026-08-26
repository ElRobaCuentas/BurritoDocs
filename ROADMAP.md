# Roadmap — El Burrito

## 1. Visión General

El proyecto se encuentra en el **Bloque A (Casa/Calle)** de desarrollo,
con la infraestructura base completa (T1.1, T2.1, T2.2, T3.1) y el
tracking multi-bus operativo. La migración del panel de gestión a la
DriverApp (FASE 1-5) fue completada, cerrando el deadlock arquitectónico
entre el spinner de Mapbox y el panel admin. El backlog oficial detallado
vive en `TAREAS.txt`.

A partir de aquí, el roadmap se organiza en dos bloques con
dependencias funcionales.

## 2. Tareas Completadas

| Tarea | Descripción | Proyecto |
|-------|-------------|---------|
| T1.1 | Route Guard por Rol (pre-migración): gating de rutas admin via `rol === 'admin'` en StackNavigator. Reemplazado por ADR-017 (autorización por `/administradores/{auth.uid}`). | BurritoUserApp |
| T2.1 | Firebase Rules: RBAC, `.indexOn` en `/asignaciones/choferId`, mínimo privilegio | Consola Firebase |
| T2.2 | Testing Unitario: Haversine (6 tests), getMovementStatus (6 tests), filtro dedup (7 tests), App.test.tsx corregido | BurritoUserApp + BurritoDriverApp |
| T3.1 | Multi-bus listener: migración de `/ubicacion_burrito` a `/ubicacion_buses`, store `Record<string, BurritoLocation>` | BurritoUserApp |
| T3.2 | Multi-bus integración — validación del pipeline completo con datos reales (Motorola, caminata + bus, Firebase Console + UserApp). | UserApp |
| T5.4 | Migración del Panel de Gestión a DriverApp: módulo admin trasplantado, enrutador tripartito por auth.uid, poda de UserApp, reglas RTDB reescritas | BurritoDriverApp + BurritoUserApp |
| T5.5 | Seguridad — autorización por `/administradores/{auth.uid}` con `.write: false`, deprecación de `rol` como fuente admin | Consola Firebase |
| ADR-018 | Bug async `initializeApp` en `admin_service.ts` — corregido con `await` + tipo `FirebaseApp` | BurritoDriverApp |
| UX-01 | Ciclo 3 estados en SendCoordinates (separar stopProcess de signOut), FloatingBackButton, FloatingBackButton en CRUDs | BurritoDriverApp |
| UX-02 | Native-stack en AdminNavigator, gestureEnabled, headers eliminados | BurritoDriverApp |
| UX-03 | Dependencias: react-native-vector-icons, native-stack | BurritoDriverApp |

## 3. UX / UI Polishing — BurritoDriverApp

Fase de pulido visual y corrección de fugas de estado en la interfaz de
conductor y administración. No se modifica lógica de negocio, Firebase,
GPS, BackgroundJob, servicios, modelos, ni flujo de tracking.

Cada tarea es un commit atómico: se implementa, se prueba en emulador,
se commitea y se sube a GitHub antes de comenzar la siguiente.

| Orden | Tarea | Commit | Archivos clave |
|-------|-------|--------|---------------|
| 0 | **Standardize Navigation** — unificar navegación bajo `@react-navigation/stack` con `animation: 'none'` | `chore(nav): standardize on @react-navigation/stack` | `AdminNavigator.tsx`, `AdminPanelScreen.tsx`, `package.json` |
| 1 | **Fix Admin Routing State Leak** — eliminar flash Login→Tracking→Admin por verificación asíncrona de rol | `fix(auth): prevent admin routing flash during role verification` | `DriverApp.tsx` |
| 2 | **Android Vector Icons** — configurar fonts.gradle y react-native.config.js para MaterialCommunityIcons | `fix(android): link vector-icons fonts for Android` | `android/app/build.gradle`, `react-native.config.js` (nuevo) |
| 3 | **Rediseño Login** — branding, perfiles Admin/Conductor, diseño moderno | `feat(ui): redesign driver login screen` | `LoginDriverScreen.tsx` |
| 4 | ✅ **Tracking Screen Producción** — reemplazar interfaz provisional (emojis, terminal debug) por versión profesional | `f79b99e` | `SendCoordinates.tsx` |
| 5 | **Admin Dashboard** — reemplazar emojis del menú admin por iconos vectoriales | `feat(ui): replace admin panel emojis with vector icons` | `AdminPanelScreen.tsx` |
| 6 | **Consistencia CRUD** — unificar estilos de formularios, inputs, botones, tarjetas en Choferes/Buses/Asignaciones | `style(admin): unify CRUD screen visual consistency` | `ChoferesScreen.tsx`, `BusesScreen.tsx`, `AsignacionesScreen.tsx` |
| 7 | **Sistema Tipográfico** — auditar y migrar todos los textos a `TYPOGRAPHY.primary.*` eliminando fontWeight hardcodeado | `style(ui): enforce typography system across DriverApp` | Todos los screens |
| 8 | **Branding Final** — paleta completa en COLORS, bordes, sombras, radios consistentes | `style(brand): apply DriverApp branding and color system` | `theme/colors.ts` + retoques |
| 9 | **Documentación** — sincronizar docs tras refactor UI | `docs(driverapp): synchronize documentation after UI refactor` | `AGENTS.md`, `ARCHITECTURE.md`, `DECISIONS.md`, `PROJECT_CONTEXT.md` |

## 4. Bloque A: Casa / Calle

Implementación y pruebas con GPS local (casa/calle del desarrollador).
Hardware: 1 Motorola (DriverApp background + UserApp foreground).
No requiere estar en el campus universitario.

| Orden | Tarea | Proyecto | Depende de |
|-------|-------|----------|------------|
| 1 | ✅ T4.1: Heartbeat — setInterval(8000) para mantener timestamp fresco | DriverApp | T3.2 |
| 2 | T5.3: Timeout Check — ocultar buses sin actualización >60s | UserApp | T4.1 |
| 3 | T4.2: Control de Turnos — botones INICIAR/FINALIZAR, creación de `/recorridos` | DriverApp | T4.1 |
| 4 | T4.4: Multi-bus render completo — ShapeSource + SymbolLayer por cada bus activo | UserApp | T3.2 |
| 5 | T4.5: Monitor de Flota — sección "Flota en servicio ahora" en AdminPanelScreen | DriverApp | T4.4 |
| 6 | T4.6: Estadísticas — StatsScreen con métricas de recorridos | UserApp | T4.2 |
| 7 | T4.3: Geofencing (implementación) — máquina de estados con Haversine, histéresis 40m/80m | DriverApp | T4.2 |
| 8 | T5.2: Smoothing (implementación) — algoritmo de interpolación (moving average / Kalman ligero) | UserApp | T4.4 |

**Nota sobre calculateDistance():** la función Haversine se extrajo a
`BurritoUserApp/src/features/map/utils/geo.ts` y está cubierta por
6 tests unitarios (T2.2). Está disponible para su uso en geofencing
(T4.3) y smoothing (T5.2).

## 5. Bloque B: Campus UNMSM

Validación física in-situ dentro de la universidad y dentro del bus
universitario. Requiere presencia física en el campus.

| Orden | Tarea | Proyecto | Depende de |
|-------|-------|----------|------------|
| 1 | T4.3: Geofencing (validación real) — ajustar radios contra multipath de edificios | DriverApp | T4.3 impl (Bloque A) |
| 2 | T5.1: Calibración de Paraderos — coordenadas GPS reales de los 10 paraderos | UserApp | — |
| 3 | T5.2: Smoothing (ajuste fino) — coeficientes contra datos reales del circuito | UserApp | T5.2 impl (Bloque A) |

## 6. Tareas Diferidas

| Tarea | Referencia | Motivo |
|-------|-----------|--------|
| T1.2: Eliminar Debug Panel | `Map.tsx:255-262` | Congelada por orden del desarrollador hasta pre-producción |
| T2.3: Asegurar Credenciales de Firma | `gradle.properties` en ambas apps | Diferida hasta empaquetar v1.2.0 |

## 7. Visión de Largo Plazo

Iniciativas identificadas que no forman parte del roadmap activo pero
que representan la dirección futura del proyecto.

- Dashboard web para administración de flota.
- Backend propio (Node.js o similar) para centralizar lógica.
- API pública para integración con servicios externos.
- Analytics avanzados (tiempos de espera, ocupación, rutas más usadas).
- Notificaciones push a estudiantes sobre llegada de buses.
- App multiplataforma con soporte iOS completo.

**Estado:** ideas registradas. Sin prioridad ni fecha asignada.

## 8. Dependencias entre Bloques

```
Completadas (T1.1, T2.1, T2.2, T3.1, T3.2, T5.4, T5.5, UX-01, UX-02, UX-03)
    ↓
UX/UI Polishing (Tareas 0–9) ← commits atómicos, cada uno probado en emulador
    ↓
Bloque A (Casa/Calle) ← implementación desde casa
    ↓
Bloque B (Campus UNMSM) ← validación física en campus

Tareas Diferidas ← sin fecha
Visión Largo Plazo ← sin prioridad
```
