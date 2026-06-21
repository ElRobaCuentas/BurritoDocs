# Roadmap — El Burrito

## 1. Visión General

El proyecto se encuentra en el **Bloque A (Casa/Calle)** de desarrollo,
con la infraestructura base completa (T1.1, T2.1, T2.2, T3.1) y el
tracking multi-bus operativo. El backlog oficial detallado vive en
`TAREAS.txt`.

A partir de aquí, el roadmap se organiza en dos bloques con
dependencias funcionales.

## 2. Tareas Completadas

| Tarea | Descripción | Proyecto |
|-------|-------------|---------|
| T1.1 | Route Guard por Rol: gating de rutas admin via `rol === 'admin'` en StackNavigator | BurritoUserApp |
| T2.1 | Firebase Rules: RBAC, `.indexOn` en `/asignaciones/choferId`, mínimo privilegio | Consola Firebase |
| T2.2 | Testing Unitario: Haversine (6 tests), getMovementStatus (6 tests), filtro dedup (7 tests), App.test.tsx corregido | BurritoUserApp + BurritoDriverApp |
| T3.1 | Multi-bus listener: migración de `/ubicacion_burrito` a `/ubicacion_buses`, store `Record<string, BurritoLocation>` | BurritoUserApp |

## 3. Bloque A: Casa / Calle

Implementación y pruebas con GPS local (casa/calle del desarrollador).
Hardware: 1 Motorola (DriverApp background + UserApp foreground).
No requiere estar en el campus universitario.

| Orden | Tarea | Proyecto | Depende de |
|-------|-------|----------|------------|
| 1 | T3.2: Multi-bus integración — validación del pipeline completo con datos reales | UserApp | T3.1 |
| 2 | T4.1: Heartbeat — setInterval(8000) para mantener timestamp fresco | DriverApp | T3.2 |
| 3 | T5.3: Timeout Check — ocultar buses sin actualización >60s | UserApp | T4.1 |
| 4 | T4.2: Control de Turnos — botones INICIAR/FINALIZAR, creación de `/recorridos` | DriverApp | T4.1 |
| 5 | T4.4: Multi-bus render completo — ShapeSource + SymbolLayer por cada bus activo | UserApp | T3.2 |
| 6 | T4.5: Monitor de Flota — sección "Flota en servicio ahora" en AdminPanelScreen | UserApp | T4.4 |
| 7 | T4.6: Estadísticas — StatsScreen con métricas de recorridos | UserApp | T4.2 |
| 8 | T4.3: Geofencing (implementación) — máquina de estados con Haversine, histéresis 40m/80m | DriverApp | T4.2 |
| 9 | T5.2: Smoothing (implementación) — algoritmo de interpolación (moving average / Kalman ligero) | UserApp | T4.4 |

**Nota sobre calculateDistance():** la función Haversine se extrajo a
`BurritoUserApp/src/features/map/utils/geo.ts` y está cubierta por
6 tests unitarios (T2.2). Está disponible para su uso en geofencing
(T4.3) y smoothing (T5.2).

## 4. Bloque B: Campus UNMSM

Validación física in-situ dentro de la universidad y dentro del bus
universitario. Requiere presencia física en el campus.

| Orden | Tarea | Proyecto | Depende de |
|-------|-------|----------|------------|
| 1 | T4.3: Geofencing (validación real) — ajustar radios contra multipath de edificios | DriverApp | T4.3 impl (Bloque A) |
| 2 | T5.1: Calibración de Paraderos — coordenadas GPS reales de los 10 paraderos | UserApp | — |
| 3 | T5.2: Smoothing (ajuste fino) — coeficientes contra datos reales del circuito | UserApp | T5.2 impl (Bloque A) |

## 5. Tareas Diferidas

| Tarea | Referencia | Motivo |
|-------|-----------|--------|
| T1.2: Eliminar Debug Panel | `Map.tsx:255-262` | Congelada por orden del desarrollador hasta pre-producción |
| T2.3: Asegurar Credenciales de Firma | `gradle.properties` en ambas apps | Diferida hasta empaquetar v1.2.0 |

## 6. Visión de Largo Plazo

Iniciativas identificadas que no forman parte del roadmap activo pero
que representan la dirección futura del proyecto.

- Dashboard web para administración de flota.
- Backend propio (Node.js o similar) para centralizar lógica.
- API pública para integración con servicios externos.
- Analytics avanzados (tiempos de espera, ocupación, rutas más usadas).
- Notificaciones push a estudiantes sobre llegada de buses.
- App multiplataforma con soporte iOS completo.

**Estado:** ideas registradas. Sin prioridad ni fecha asignada.

## 7. Dependencias entre Bloques

```
Tareas Completadas (T1.1, T2.1, T2.2, T3.1)
    ↓
Bloque A (Casa/Calle) ← implementación desde casa
    ↓
Bloque B (Campus UNMSM) ← validación física en campus

Tareas Diferidas ← sin fecha
Visión Largo Plazo ← sin prioridad
```
