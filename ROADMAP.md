# Roadmap — El Burrito

## 1. Visión General

El proyecto se encuentra cerrando su **fase de documentación** para
estabilizar la base técnica antes de retomar el desarrollo activo.

A partir de aquí, el roadmap se organiza en fases con dependencias
funcionales: cada fase desbloquea los requisitos de la siguiente.

## 2. Fase Actual: Documentación

Completar la documentación del ecosistema para que cualquier
desarrollador o IA pueda entender el proyecto sin leer todo el código.

**Documentos incluidos:**
- PROJECT_CONTEXT.md ✅
- ARCHITECTURE.md ✅
- FIREBASE_SCHEMA.md ✅
- READMEs (UserApp + DriverApp) ✅
- ROADMAP.md ← Este documento
- TROUBLESHOOTING.md ✅
- DECISIONS.md ✅
- AGENTS.md (UserApp + DriverApp) ✅

**Estado actual:** en progreso.
**Próximo hito:** completar todos los documentos y retomar desarrollo.

## 3. Fase 1: Tracking Multi-Bus

Unificar el listener de ubicación en la UserApp para que consuma
múltiples buses simultáneamente desde el nodo de tracking definitivo.

**Tareas incluidas:**

| Tarea | Depende de |
|-------|-----------|
| Multi-bus listener: migrar UserApp de `/ubicacion_burrito` a `/ubicacion_buses` | — |
| Multi-bus render: mostrar múltiples buses en Mapbox | Multi-bus listener |

**Impacto en la documentación:**
- Al completarse, el nodo `/ubicacion_burrito` deja de utilizarse.
- El simulador Python pierde utilidad como fuente de datos.
- La limitación "un solo bus activo" se elimina de PROJECT_CONTEXT.

**Estado:** pendiente. Primera prioridad post-documentación.

## 4. Fase 2: Automatización de Recorridos

Incorporar lógica de geofencing y control de turnos para automatizar
la apertura y cierre de recorridos sin intervención manual del conductor.

**Tareas incluidas:**

| Tarea | Depende de |
|-------|-----------|
| Control de turnos: crear nodo `/recorridos` con inicio/cierre | Fase 1 |
| Geofencing: activar Haversine con Punto Cero y umbral de 40m | Control de turnos |

**Nota:** la función `calculateDistance()` (Haversine) ya existe en
`Map.tsx` pero no está activa en el flujo de tracking. Su activación
requiere que el flujo de datos esté consolidado (Fase 1).

**Estado:** pendiente. Depende de Fase 1.

## 5. Fase 3: Monitoreo de Flota

Construir un panel de monitoreo para visualizar el estado de todos los
buses en tiempo real y acceder a estadísticas operativas.

**Tareas incluidas:**

| Tarea | Depende de |
|-------|-----------|
| Monitor de flota: mapa con todos los buses activos | Fase 1 |
| Estadísticas: métricas de recorridos, tiempos, distancia | Fase 2 |

**Estado:** pendiente. Depende de Fases 1 y 2.

## 6. Fase 4: Preparación para Producción

Asegurar que el sistema cumpla con los requisitos de seguridad,
rendimiento y empaquetado para un lanzamiento oficial.

**Tareas incluidas:**

| Tarea | Descripción |
|-------|-------------|
| Firebase Rules | Configurar reglas de seguridad en RTDB (`.read`, `.write`, `.indexOn`) |
| Variables de entorno | Migrar tokens a perfiles seguros (revisar `gradle.properties`) |
| Firma AAB | Configurar almacén de llaves de producción y generar Android App Bundle |
| Smoothing | Algoritmo de interpolación para suavizar movimiento del marcador |
| Timeout check | Validador de inactividad para ocultar buses desconectados |
| Testing | Ampliar cobertura de pruebas (Haversine, stores, flujos críticos) |

**Estado:** puede ejecutarse en paralelo con Fases 1-3, pero requiere
que el flujo principal esté estable antes del lanzamiento.

## 7. Tareas Diferidas (sin prioridad actual)

Funcionalidades reconocidas pero que el equipo ha decidido no priorizar
en el corto plazo. Marcadas como "NO TOCAR" en la base de código.

| Tarea | Referencia en código |
|-------|---------------------|
| Debug panel | `Map.tsx`: `setDebugLogs`, `styles.debugPanel` (comentado) |
| Heartbeat independiente | `SendCoordinates.tsx`: watchdog cleanup (rama `task/T04-driver-heartbeat` sin mergear) |
| Calibración de GPS | Sin implementación |

**Estado:** diferido. Sin fecha estimada.

## 8. Visión de Largo Plazo

Iniciativas identificadas que no forman parte del roadmap activo pero
que representan la dirección futura del proyecto.

- Dashboard web para administración de flota.
- Backend propio (Node.js o similar) para centralizar lógica.
- API pública para integración con servicios externos.
- Analytics avanzados (tiempos de espera, ocupación, rutas más usadas).
- Notificaciones push a estudiantes sobre llegada de buses.
- App multiplataforma con soporte iOS completo.

**Estado:** ideas registradas. Sin prioridad ni fecha asignada.

Estas iniciativas representan la dirección futura del ecosistema y
podrán incorporarse al roadmap activo cuando finalicen las fases
actuales.

## 9. Dependencias entre Fases

```
Fase Actual (Documentación)
    ↓
Fase 1 (Multi-bus) ← sin esto, el resto no tiene base
    ↓
Fase 2 (Geofencing + Turnos) ← necesita tracking consolidado
    ↓
Fase 3 (Monitoreo) ← necesita multi-bus y recorridos
    ↓
Fase 4 (Preparación Producción) ← paralelizable, pero requiere estabilidad

Tareas Diferidas ← sin fecha
Visión Largo Plazo ← sin prioridad
```
