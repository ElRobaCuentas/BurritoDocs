# Project Context — El Burrito

## 1. Propósito del Sistema

"El Burrito" es un sistema de seguimiento en tiempo real para los autobuses universitarios de la Universidad Nacional Mayor de San Marcos (UNMSM).

El problema que resuelve es de **asimetría de información**: los estudiantes esperan en paraderos sin saber cuándo llegará el próximo bus. El sistema elimina esa incertidumbre mostrando la posición en vivo de cada unidad sobre un mapa accesible desde el celular.

## 2. Aplicaciones del Ecosistema

Estado del proyecto

El proyecto se encuentra actualmente en una fase de consolidación documental.

La infraestructura principal (Firebase, autenticación, tracking GPS y
Background Service) ya funciona de forma estable.

Las siguientes iteraciones se enfocan en funcionalidades de flota
(multi-bus, geofencing, dashboard y estadísticas) descritas en ROADMAP.md.


El sistema está compuesto por dos aplicaciones móviles independientes:

### BurritoDriverApp

- Dispositivo: teléfono Android instalado en cada bus universitario.
- Función: capturar coordenadas GPS y transmitirlas a Firebase en tiempo
  real mediante un Foreground Service que mantiene el envío activo incluso
  con la pantalla apagada.
- Usuario: conductores de la Oficina de Transportes.
- Autenticación: login con DNI + contraseña sobre Firebase Auth.

### BurritoUserApp

- Dispositivo: celular personal de cualquier estudiante que use Android o iOS (próximamente).
- Función: consumir las coordenadas de los buses y mostrarlas sobre un
  mapa de Mapbox.
- Usuario: estudiantes de la UNMSM.
- Autenticación: email/contraseña o Google Sign-In.

### Relación entre aplicaciones

```
DriverApp ──escribe──→ Firebase RTDB ──escucha──→ UserApp

```

Firebase Realtime Database actúa como el bus de datos. La UserApp nunca escribe coordenadas; solo lee. No existe backend propio ni API intermedia.

## 3. Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | React Native CLI + TypeScript |
| Estado | Zustand |
| Base de datos | Firebase Realtime Database |
| Autenticación | Firebase Auth |
| Mapas | Mapbox (`@rnmapbox/maps`) |
| Tracking GPS | `react-native-background-actions` + `@react-native-community/geolocation` |
| Navegación | React Navigation |

## 4. Estado actual del proyecto

### Funcionalidades implementadas

- Autenticación de conductores (login con DNI).
- Autenticación de estudiantes (email/contraseña y Google Sign-In).
- Asignación de conductores a buses mediante panel administrativo.
- Captura de coordenadas GPS y transmisión continua a Firebase, incluso con la app en segundo plano.
- Recepción en tiempo real de la ubicación del bus en la aplicación del estudiante mediante Mapbox. Actualmente esta funcionalidad utiliza el flujo de tracking legado; la migración al listener multi-bus forma parte de la Tarea 4.1 del roadmap (TOMAR EN CUENTA PARA FUTURAS REVISIONES)
- Interfaz administrativa para gestionar conductores, buses y asignaciones.
- Dark mode en la app de estudiantes.
- Compatibilidad con Android 14 (API 34), incluyendo soporte para Foreground Services de tipo location. Este soporte utiliza foregroundServiceType: location requerido por Android para servicios de ubicación en primer plano.
  `foregroundServiceType: location` requerido por el sistema operativo.
- Feedback de estudiantes mediante formulario de comentarios.

### Funcionalidades pendientes (ver ROADMAP.md)

- Listener multi-bus (soportar múltiples buses simultáneos en el mapa).
- Geofencing y cierre automático de vueltas (varias tareas del roadmap).
- Control de turnos (inicio/fin de recorrido con registro en base de datos).
- Dashboard y estadísticas de flota.
- Indicadores de estado en el mapa (activo, detenido, sin conexión).

## 5. Limitaciones Actuales

- **Precisión GPS**: el campus de la UNMSM produce interferencia
  estructural (efecto multipath) con márgenes de error de ±10–15 metros.
- **Dependencia de red móvil**: la sincronización en tiempo real requiere
  cobertura de datos en la ruta del bus.
- **Solo Android para conducción**: el Foreground Service que mantiene
  la transmisión activa está implementado sobre APIs nativas de Android.
  DriverApp no está probada ni soportada en iOS.
- **Seguimiento de un único bus**: La UserApp utiliza actualmente el listener legado para una única fuente de ubicación. El soporte para múltiples buses simultáneos forma parte de la Tarea 4.1 del roadmap (VER A FUTURO)

## 6. Documentación Relacionada

| Documento | Propósito |
|-----------|-----------|
| `ARCHITECTURE.md` | Flujo de datos, componentes y ciclo de vida. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos, payloads e índices de la RTDB. |
| `ROADMAP.md` | Tareas pendientes, fases y prioridades. |
| `TROUBLESHOOTING.md` | Historial de bugs, síntomas y soluciones. |
| `DECISIONS.md` | Registro de decisiones de arquitectura (ADR). |
| BurritoDriverApp/README.md | Setup y funcionalidad de la app del conductor. |
| BurritoUserApp/README.md | Setup y funcionalidad de la app del estudiante. |
