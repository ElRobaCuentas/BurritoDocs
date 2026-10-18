# Project Context — El Burrito

## 1. Propósito del Sistema

"El Burrito" es un sistema de seguimiento en tiempo real para los autobuses universitarios de la Universidad Nacional Mayor de San Marcos (UNMSM).

El problema que resuelve es de **asimetría de información**: los estudiantes esperan en paraderos sin saber cuándo llegará el próximo bus. El sistema elimina esa incertidumbre mostrando la posición en vivo de cada unidad sobre un mapa accesible desde el celular.

## 2. Aplicaciones del Ecosistema

Estado del proyecto

El proyecto se encuentra en la fase de desarrollo Casa/Calle (Bloque A).
La infraestructura principal (Firebase, autenticación, tracking GPS,
Background Service, reglas de seguridad RBAC, testing unitario,
listener multi-bus, **panel de gestión en AdminWeb** y **autorización por
auth.uid**) ya está implementada y operativa.

La migración del panel de gestión resolvió el deadlock arquitectónico
entre el spinner de Mapbox de la UserApp y el panel admin (ADR-017). La
UserApp quedó 100% estudiante; el panel admin se separó de la app móvil
en **AdminWeb** (SPA web), y la DriverApp quedó exclusiva de conductores
(ADR-025).

Las siguientes iteraciones se enfocan en funcionalidades de flota
(heartbeat, timeout, turnos, render multi-marcador, monitor, estadísticas,
geofencing y smoothing) descritas en MVP.md y ROADMAP.md.

### BurritoDriverApp

- Dispositivo: teléfono Android instalado en cada bus universitario.
- Función: capturar coordenadas GPS y transmitirlas a Firebase en tiempo
  real mediante un Foreground Service.
- Usuarios: conductores de la Oficina de Transportes.
- Autenticación: login con DNI + contraseña sobre Firebase Auth, con email
  compuesto `${dni}@conductor.com`. La app es exclusiva de conductores.

### BurritoUserApp

- Dispositivo: celular personal de cualquier estudiante (Android/iOS).
- Función: consumir las coordenadas de los buses y mostrarlas sobre un
  mapa de Mapbox.
- Usuarios: estudiantes de la UNMSM.
- Autenticación: email/contraseña o Google Sign-In.

### AdminWeb

- Dispositivo: navegador (SPA web).
- Función: panel de gestión — CRUD de conductores, buses, asignaciones y
  administradores.
- Usuarios: administradores de la Oficina de Transportes.
- Autenticación: login con DNI + contraseña sobre Firebase Auth, con email
  compuesto `${dni}@admin.com`; acceso autorizado por membresía en
  `/administradores/{auth.uid}`.

### Relación entre aplicaciones

```
DriverApp ──escribe──→ Firebase RTDB ──escucha──→ UserApp
AdminWeb  ──escribe──→ Firebase RTDB
```

Firebase Realtime Database actúa como el bus de datos. La UserApp nunca
escribe coordenadas; solo lee. No existe backend propio ni API intermedia.
La DriverApp escribe el tracking (`/ubicacion_buses`) y AdminWeb los nodos
de administración (conductores, buses, asignaciones).

## 3. Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | React Native CLI + TypeScript |
| Estado | Zustand |
| Base de datos | Firebase Realtime Database |
| Autenticación | Firebase Auth |
| Mapas | Mapbox (`@rnmapbox/maps`) |
| Tracking GPS | `react-native-background-actions` + `@react-native-community/geolocation` |
| Navegación | React Navigation (ambas apps) |

## 4. Estado actual del proyecto

### Funcionalidades implementadas

- Autenticación de conductores (login con DNI).
- Autenticación de estudiantes (email/contraseña y Google Sign-In).
- Asignación de conductores a buses mediante panel administrativo.
- Captura de coordenadas GPS y transmisión continua a Firebase, incluso con la app en segundo plano.
- Presencia del bus (heartbeat) cada 8 s, independiente del GPS, para mantener el timestamp fresco cuando el bus está detenido (T4.1).
- Recepción en tiempo real de la ubicación de todos los buses activos en
  la aplicación del estudiante mediante Mapbox. El backend de tracking
  multi-bus está implementado (T3.1): el store de Zustand almacena las
  posiciones como `Record<string, BurritoLocation>` indexado por placa,
  con filtro de deduplicación y clasificación moving/stopped/offline
  independiente por cada bus. La UI actualmente muestra un solo marcador
  (seleccionando el bus activo más reciente); el render multi-marcador
  completo está planificado en T4.4.
- **Interfaz administrativa**: en AdminWeb (SPA web). CRUD completo de
  conductores, buses y asignaciones. Acceso por membresía en
  `/administradores/{auth.uid}`.
- Dark mode en la app de estudiantes.
- Compatibilidad con Android 14 (API 34), incluyendo soporte para Foreground Services de tipo location.
- Feedback de estudiantes mediante formulario de comentarios.
- **Migración del panel admin**: completada. La UserApp fue podada de todo
  código administrativo y la DriverApp quedó exclusiva de conductores; el
  panel reside en AdminWeb (ADR-025).
- **Modelo de autorización**: reemplazado de `/usuarios/{uid}/rol` a
  `/administradores/{auth.uid}`, que el cliente no puede autoasignarse
  (la escritura exige ser un administrador ya existente).
- **Bug de inicialización asíncrona**: corregido. `initializeApp()` del
  SDK ahora se invoca con `await`.

### Funcionalidades pendientes (ver MVP.md y ROADMAP.md)

- Timeout check (ocultar buses fantasma tras inactividad — T5.3).
- Control de turnos (inicio/fin de recorrido — T4.2).
- Render multi-marcador completo en el mapa (T4.4).
- Monitor de flota en panel admin (T4.5).
- Estadísticas de recorridos (T4.6).
- Geofencing con cierre automático de vueltas (T4.3).
- Smoothing de interpolación visual (T5.2).
- Calibración de paraderos en campus (T5.1).

## 5. Limitaciones Actuales

- **Precisión GPS**: el campus de la UNMSM produce interferencia
  estructural (efecto multipath) con márgenes de error de ±10–15 metros.
- **Dependencia de red móvil**: la sincronización en tiempo real requiere
  cobertura de datos en la ruta del bus.
- **Solo Android para conducción**: el Foreground Service que mantiene
  la transmisión activa está implementado sobre APIs nativas de Android.
  DriverApp no está probada ni soportada en iOS.
- **Render de un solo marcador**: el backend multi-bus ya está implementado (T3.1), pero la UI del mapa aún muestra un único marcador (seleccionando el bus activo más reciente). El render multi-marcador completo está planificado en T4.4.
- **Bootstrap del primer admin manual**: crear el **primer** administrador
  requiere ir a Firebase Console, crear la cuenta Auth y registrar su UID en
  `/administradores/{uid}`. Los siguientes administradores se crean desde el
  panel (AdminWeb → AdminsScreen). Aceptable por el número reducido de admins.

## 6. Documentación Relacionada

| Documento | Propósito |
|-----------|-----------|
| `ARCHITECTURE.md` | Flujo de datos, componentes y ciclo de vida. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos, payloads e índices de la RTDB. |
| `ROADMAP.md` | Tareas pendientes, fases y prioridades. |
| `TROUBLESHOOTING.md` | Historial de bugs, síntomas y soluciones. |
| `DECISIONS.md` | Registro de decisiones de arquitectura (ADR). |
| Driver/README.md | Setup y funcionalidad de la app del conductor. |
| User/README.md | Setup y funcionalidad de la app del estudiante. |
| AdminWeb/README.md | Setup y funcionalidad del panel de administración. |
| `BUGS_RESUELTOS/` | Historial de bugs resueltos por aplicación. |
