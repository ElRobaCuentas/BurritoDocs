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

La prioridad actual es estabilizar el MVP y validar una jornada de
aproximadamente ocho horas con un bus y teléfono controlados. Heartbeat,
caducidad y asignaciones diarias ya existen; las ampliaciones de flota,
monitor, estadísticas, geofencing y smoothing permanecen Post-MVP.

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
- **Interfaz administrativa**: en AdminWeb (SPA web). Gestión de
  conductores (crear, editar, activar/desactivar — no se eliminan),
  buses y asignaciones. Acceso por membresía en
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
- **Validación diaria de asignación en DriverApp (I08)**: al montar y al
  presionar INICIAR se consulta `/asignaciones` por `choferId` con la fecha
  `America/Lima`; una respuesta válida fija el bus del recorrido y una
  conocida como inválida lo bloquea. Con desconexión confirmada por
  `.info/connected`, INICIAR puede usar un respaldo en memoria del mismo
  conductor/día. La etiqueta «Servicio de ubicación activo» describe el
  servicio Android local (ADR-028).
- **Privacidad del diagnóstico (I11)**: el panel del conductor ya no imprime
  nombre, DNI ni prefijo del DNI. Crashlytics recibe el UID con sesión y
  `setUserId('')` cuando Auth informa usuario nulo. Los diagnósticos
  operativos y los errores con stack trace se conservan.
- **I03/I04 (03/11/2026)**: reglas publicadas y operación real aprobada por
  el usuario: AdminWeb administra; BurritoDriverApp autentica, carga la
  asignación e inicia/detiene GPS; El Burrito muestra el bus y envía feedback
  con Auth UID. Laboratorio previo: 72/72 pruebas y B2 funcional. No se
  repitió Firebase real en esta fase. El código I04 está en User `5bc3a83`,
  con push aún pendiente al corte previo a integración.
- **I06-B (03/11/2026)**: recepción rechaza timestamps explícitos mayores
  que hora local de recepción + 300 000 ms. User: 124/124 tests en diez
  suites, incluidos límites y pipeline real servicio → store. Corrección
  local verificada en `fix/i06-future-timestamps` (`011a4b6`), aún sin
  integración/publicación ni APK nuevo.
- **Foco AdminWeb (03/11/2026)**: efectos de apertura y Escape separados en
  `Modal.tsx`; lint, TypeScript, bundle en memoria y 12 comprobaciones de
  efectos React con DOM simulado aprobados. Verificación visual en navegador
  pendiente; commit local `5e66d46` en `fix/modal-focus`, aún sin
  integrar/publicar/desplegar.
- **Caducidad C4.6:** los buses sin actualización válida se ocultan a los
  30 s; no falta implementar otro timeout para cerrar el MVP.

### Planificación posterior al MVP

Las tareas, estados y dependencias viven únicamente en `ROADMAP.md`.
El alcance y cierre del MVP se consultan en `../MVP.md`.

## 5. Limitaciones Actuales

- **I06-A (decisión 30/10, ratificada 03/11/2026):** campos ausentes/null
  conservan coordenadas `0` y timestamp `Date.now()`, incluso en activos.
  Es una limitación aceptada del MVP con emisor conocido; las pruebas GPS
  normales recibieron coordenadas completas. No equivale a validar todo
  registro incompleto ni a un defecto nuevo descubierto por la auditoría.
- **Relojes I06-B:** cinco minutos son una tolerancia autorizada, no una
  medición de precisión ni sincronización de relojes. Adelantos dentro del
  margen pueden retrasar datos posteriores y caducidad; un desfase mayor
  persistente hace que se descarten publicaciones (también detenciones).
  La posición previa caduca con el mecanismo existente. No hay corrección
  retroactiva de datos aceptados por un proceso anterior a este cambio.
- **Cierre operativo:** falta la jornada de ~8 h con pantalla bloqueada,
  movimiento/paradas, relevos y pérdida/recuperación de red. Las pruebas
  unitarias y las comprobaciones físicas cortas no la acreditan. Para I09/I10
  y P06, ver `ROADMAP.md` (Tareas postergadas); I07 y el logout aceptado no se reabren.
- **Estado de esta fase:** correcciones I06-B/foco y documentación pendientes
  de revisión previa a integrar/push. B2 aislado en `dc58daf`; test protegido
  de Driver intacto y sin versionar. Graphify diferido por decisión expresa
  hasta después de sincronizar GitHub, con autorización posterior propia.
- **Ciclo de vida GPS (I07, 01/11/2026)**: corregido funcionalmente y
  validado e integrado/publicado en `Driver/main` (`9faaaf9`); documentación
  técnica integrada/publicada mediante `docs/main` (`d5c4d48`). H1 A/B y la regresión de
  64/64 tests respaldan cierre local offline, ausencia de reinicio voluntario
  indebido y nuevo inicio antes del ACK de parada anterior; no se requiere H2
  adicional. La confirmación remota no gobierna el cierre local y pueden
  quedar escrituras SDK pendientes en memoria aun sin persistencia en disco.
  D3 se cubre con pruebas deterministas, no con recuperación en vuelo aislada
  físicamente; tampoco se forzaron fallos nativos. El wrapper 4.1.0 conserva
  el residual de resolver global sobrescrito por entregas HeadlessJS tardías:
  promesa JS pendiente no equivale a cierre HeadlessJS, ni prueba dos motores
  GPS completos simultáneos en Android. No se extrapola a una jornada de 8 h.
  Ver `BUGS_RESUELTOS/driverapp.md` (CASO 020) y `ARCHITECTURE.md` §8.
- **Respaldo offline de la asignación (I08, 01/11/2026)**: implementado y
  validado físicamente; integrado/publicado en `Driver/main` (`67fa362`,
  documentación `5735fa8`). AdminWeb I08 integrado/publicado y desplegado
  desde `80a52bd`, según la validación previa de Hosting. El respaldo vive solo en memoria
  (no sobrevive al proceso ni al cambio de día) y no detecta altas,
  cancelaciones ni reasignaciones administrativas ocurridas offline. Solo
  se habilita con `.info/connected == false`, no con un timeout aislado.
  La ruta offline se probó sin desplazamiento físico; el remontaje durante
  el recorrido se cubre con Jest. El APK validado es anterior al cambio de
  etiqueta «Servicio de ubicación activo»; no se generó otro. Ver
  `BUGS_RESUELTOS/driverapp.md` (CASO 021) y ADR-028.
- **Identidad de Crashlytics (I11, 02/11/2026)**: las pruebas verifican las
  solicitudes al SDK, no la aplicación nativa del UID ni la recepción de
  reportes. La limpieza no elimina reportes históricos ni sanitiza objetos de
  error. No hay una cola que garantice el orden nativo de `setUserId`. Ver
  `BUGS_RESUELTOS/driverapp.md` (CASO 022) y ADR-029.
  Integrado/publicado en Driver `4a05e41` y docs `1689a7a`; no implica una
  prueba de recepción nativa de telemetría.
- **Precisión GPS**: el campus de la UNMSM produce interferencia
  estructural (efecto multipath) con márgenes de error de ±10–15 metros.
- **Dependencia de red móvil**: la sincronización en tiempo real requiere
  cobertura de datos en la ruta del bus.
- **Solo Android para conducción**: el Foreground Service que mantiene
  la transmisión activa está implementado sobre APIs nativas de Android.
  DriverApp no está probada ni soportada en iOS.
- **Render de un solo marcador**: el backend multi-bus ya está implementado (T3.1), pero la UI del mapa aún muestra un único marcador (seleccionando el bus activo más reciente). La tarea de render multi-marcador es T4.4; su planificación vive en `ROADMAP.md`.
- **Bootstrap del primer admin manual**: crear el **primer** administrador
  requiere ir a Firebase Console, crear la cuenta Auth y registrar su UID en
  `/administradores/{uid}`. Los siguientes administradores se crean desde el
  panel (AdminWeb → AdminsScreen). Aceptable por el número reducido de admins.

## 6. Documentación Relacionada

| Documento | Propósito |
|-----------|-----------|
| `ARCHITECTURE.md` | Flujo de datos, componentes y ciclo de vida. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos, payloads e índices de la RTDB. |
| `ROADMAP.md` | Planificación posterior al MVP, postergaciones y condiciones de pre-release. |
| `TROUBLESHOOTING.md` | Historial de bugs, síntomas y soluciones. |
| `DECISIONS.md` | Registro de decisiones de arquitectura (ADR). |
| Driver/README.md | Setup y funcionalidad de la app del conductor. |
| User/README.md | Setup y funcionalidad de la app del estudiante. |
| AdminWeb/README.md | Setup y funcionalidad del panel de administración. |
| `BUGS_RESUELTOS/` | Historial de bugs resueltos por aplicación. |
