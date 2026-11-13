# Architecture — El Burrito

Documento de la arquitectura implementada del Ecosistema El Burrito. Describe
componentes, responsabilidades, comunicación, estado local, ciclo de vida del
tracking y límites observables. Los contratos completos de datos viven en
`FIREBASE_SCHEMA.md`; las decisiones y su razonamiento, en `DECISIONS.md`; la
evidencia histórica, en `BUGS_RESUELTOS/`.

## Índice

- [§1. Visión General](#1-visión-general)
- [§2. Arquitectura del Ecosistema](#2-arquitectura-del-ecosistema)
- [§3. Arquitectura de BurritoDriverApp](#3-arquitectura-de-burritodriverapp-conductor)
- [§3.1. Arquitectura de AdminWeb](#31-arquitectura-de-adminweb-administrador)
- [§3.2. PULSO (monitor de seguimiento)](#32-pulso-monitor-de-seguimiento)
- [§4. Arquitectura de BurritoUserApp](#4-arquitectura-de-burritouserapp)
- [§5. Comunicación y Flujo de Datos](#5-comunicación-y-flujo-de-datos)
- [§6. Componentes Principales](#6-componentes-principales)
- [§7. Servicios Externos](#7-servicios-externos)
- [§8. Ciclo de Vida del Tracking](#8-ciclo-de-vida-del-tracking)
- [§9. Consideraciones de Implementación y Límites](#9-consideraciones-de-implementación-y-límites)
- [§10. Referencias](#10-referencias)

## 1. Visión General

El Burrito es un sistema de seguimiento en tiempo real para los autobuses
universitarios de la Universidad Nacional Mayor de San Marcos (UNMSM). Resuelve
una **asimetría de información**: los estudiantes esperan en paraderos sin saber
cuándo llegará el próximo bus; la aplicación muestra la posición en vivo de cada
unidad sobre un mapa accesible desde el celular.

El ecosistema usa un patrón **productor-consumidor para el tracking**: la
BurritoDriverApp produce posiciones GPS y El Burrito las consume. Esa parte del
sistema es **unidireccional** y no tiene una API propia ni un backend intermedio
mantenido por el proyecto. Firebase proporciona servicios backend administrados
(Auth, Realtime Database, Hosting, Analytics y Crashlytics), que se consumen
directamente desde las aplicaciones.

El resto del ecosistema no es estrictamente unidireccional: AdminWeb administra
configuración y BurritoDriverApp lee datos necesarios para operar.

```
BurritoDriverApp ── escribe tracking ──▶ RTDB /ubicacion_buses/{busId}
        │                                        │
        └── lee perfil y asignaciones ◀──────────┘

AdminWeb ── escribe/lee configuración ──▶ RTDB /choferes, /buses,
        │                                   /asignaciones, /administradores
        └── inicializa/elimina /ubicacion_buses/{placa} al crear/eliminar bus

El Burrito ── escucha /ubicacion_buses ──▶ mapa
        ├── lee /usuarios/{uid}
        └── escribe /usuarios/{uid} y /comentarios
```

Firebase RTDB es el bus de sincronización del tracking y también el almacén de
configuración administrativa. Firebase Auth es la fuente de identidad. No hay
un servidor de aplicación propio que opaque estas operaciones.

## 2. Arquitectura del Ecosistema

### Separación de responsabilidades

| Aplicación | Rol | Función principal | Identidad | Autorización |
|-----------|-----|-------------------|-----------|--------------|
| BurritoDriverApp | **Conductor** | Capturar GPS, mantener presencia y escribir tracking en RTDB | DNI + contraseña | Sesión de conductor y reglas RTDB |
| El Burrito | **Consumidor** (estudiante) | Leer tracking y renderizar el mapa; gestionar perfil y feedback | Email/contraseña o Google Sign-In | Sesión de estudiante y reglas por UID |
| AdminWeb | **Administración** | Gestionar conductores, buses, asignaciones y administradores | DNI + contraseña | Nodo `/administradores/{uid}` |
| Firebase RTDB | **Bus y almacén** | Sincronizar tracking y configuración | — | Reglas de seguridad |
| Firebase Auth | **Identidad** | Autenticar las tres aplicaciones | — | — |

La autenticación demuestra una identidad; la autorización decide qué puede
hacer esa identidad. El rol no se deduce del email ni del UID: se deriva del
nodo administrativo, del índice de conductores o de las reglas de cada path.

### Operaciones entre componentes

| Origen | Operación | Destino |
|--------|-----------|---------|
| BurritoDriverApp | Escribe posición y `isActive` del recorrido | `/ubicacion_buses/{busId}` |
| BurritoDriverApp | Lee perfil y asignación diaria | `/choferes/{dni}`, `/asignaciones` |
| BurritoDriverApp | Consulta estado de conexión para el respaldo I08 | `.info/connected` |
| El Burrito | Escucha la colección completa | `/ubicacion_buses` |
| El Burrito | Lee y actualiza perfil | `/usuarios/{uid}` |
| El Burrito | Crea feedback | `/comentarios` |
| AdminWeb | CRUD de configuración y lectura de listas | `/choferes`, `/buses`, `/asignaciones`, `/administradores` |
| AdminWeb | Inicializa o elimina el nodo de un bus | `/ubicacion_buses/{placa}` |
| AdminWeb (PULSO) | Lee el tracking por HTTP público para evaluar el seguimiento (consulta manual) | REST `/ubicacion_buses/{busId}.json` |

### Autenticación y autorización

- **BurritoDriverApp (conductores)**: Firebase Auth con email compuesto
  `${dni}@conductor.com`. La contraseña inicial es el DNI, establecida al crear
  el conductor desde AdminWeb. El gate de la app comprueba sesión, no rol.
- **AdminWeb (administradores)**: Firebase Auth con email `${dni}@admin.com`.
  Tras el login, el auth gate consulta `/administradores/{uid}` para autorizar
  el acceso al panel.
- **El Burrito (estudiantes)**: Firebase Auth con email/contraseña o Google
  Sign-In. El perfil y el feedback se vinculan al UID autenticado.
- **Autorización RTDB**: las reglas limitan la escritura de tracking a
  conductores indexados en `/choferes_uids` o administradores, y las escrituras
  de perfil/feedback al UID propio. La UI no sustituye esas reglas.

### Proyectos y repositorios

Las tres aplicaciones comparten el proyecto Firebase `burritounmsm`, pero son
proyectos independientes con su propio `package.json`, entry point y
configuración. El workspace `C:\Movil` contiene repositorios Git separados para
`User/`, `Driver/`, `AdminWeb/` y `docs/`; no es un único repositorio con un
solo historial.

## 3. Arquitectura de BurritoDriverApp (conductor)

### Entry point y enrutador

```
index.js → src/DriverApp.tsx
              ├── sin sesión → LoginDriverScreen
              └── con sesión → SendCoordinates
```

`DriverApp.tsx` monta `onAuthStateChanged` y decide entre login y tracking. La
app está destinada exclusivamente a conductores y no incluye panel de
administración ni consulta `/administradores`. El gate solo comprueba que
exista una sesión; no verifica por sí mismo el rol ni el flag `activo` del
conductor. La separación operativa se apoya en la cuenta convencional de
conductor y en las reglas de RTDB (ADR-023, ADR-025).

### Login y sesión

`LoginDriverScreen` construye `${dni}@conductor.com` y llama a
`auth().signInWithEmailAndPassword`. Maneja errores de red, credenciales
inválidas y cuentas deshabilitadas.

`DriverApp.tsx` escucha la sesión y deriva `driverDni` de la parte local del
email. En cada cambio de usuario sincroniza Crashlytics con `setUserId(uid)` o
`setUserId('')`. `SendCoordinates` contiene el resto de la lógica de tracking;
el ciclo detallado está en §8.

CERRAR SESIÓN es una acción separada de DETENER RECORRIDO: no escribe
`isActive:false` ni detiene el servicio. La pantalla la ofrece en estado
detenido o en la tarjeta de error de asignación; durante un recorrido activo se
muestra DETENER, no CERRAR SESIÓN. La separación de responsabilidades está en
ADR-026.

La estructura de archivos relevante está centralizada en §6.

## 3.1 Arquitectura de AdminWeb (administrador)

### Entry point y auth gate

```
src/main.tsx → src/App.tsx
                   ├── sin sesión → LoginScreen
                   ├── checking → verificación de rol
                   ├── admin → AdminNavigator
                   ├── denied → "No autorizado"
                   └── error → "No se pudo verificar" (Reintentar / Cerrar Sesión)
```

`App.tsx` escucha `onAuthStateChanged` y mantiene un estado explícito de rol
(`checking` → `admin` / `denied` / `error`). Con sesión, consulta
`existeAdministrador(uid)` sobre `/administradores/{uid}` con timeout de 10 s.
Mientras está en `checking` no muestra "No autorizado". Un `PERMISSION_DENIED`
equivale a `denied`; un timeout o fallo de red va a `error` con reintento o
cierre de sesión.

`existeAdministrador` usa `get()` y retorna `snapshot.exists()`. Es la fuente
de verdad del auth gate y no se reemplaza por un claim local ni por la UI.

### Interfaz

La interfaz usa CSS propio con **tokens** (sin dependencias de UI). `index.css`
importa las hojas de `src/styles/`; usa cian `#00AEEF`, gris `#F5F5F5`, fondo
blanco y texto `#222222` en tema claro. `AppShell` aporta sidebar, topbar y
drawer móvil, muestra la identidad del admin y la expone vía
`SessionContext`/`useSession`. Las pantallas usan las primitivas de
`shared/ui/`; las confirmaciones usan `ConfirmDialog` y los avisos, toasts.

El efecto de apertura de `Modal.tsx` depende solo de `open`: enfoca el diálogo
y bloquea el scroll hasta cerrar o desmontar. Escape se registra en otro efecto
con `[open, onClose]` para usar el callback vigente sin robar el foco al editar
inputs (corrección 03/11/2026; ver `BUGS_RESUELTOS/adminweb.md`). Los cambios B2
de emuladores se preservan en `test/b2-adminweb-emulator` (`dc58daf`), fuera de
`main`.

### Módulo Admin

Accesible solo para usuarios cuyo `auth.uid` existe en `/administradores/`.
Gestión contra RTDB:

| Entidad | Path | Operaciones |
|---------|------|-------------|
| Choferes | `/choferes/{dni}` | Crear (Auth + RTDB + índice `/choferes_uids`), listar, editar, activar/desactivar (no se eliminan) |
| Buses | `/buses/{placa}` | Crear e inicializar `/ubicacion_buses/{placa}`, listar, toggle, eliminar |
| Asignaciones | `/asignaciones/{pushId}` | Crear para hoy/futuro con uno o varios turnos (valida mínimo 1, inicio < fin, mismo día, sin solape y los ordena), validar exclusividad conductor/fecha, listar, editar turnos (con la misma validación/orden) y cancelar |
| Administradores | `/administradores/{uid}` | Crear (Auth + RTDB), listar, editar, eliminar (nodo) |

Al crear cuentas de Auth, `admin_service.ts` usa una **app temporal por
operación** con `inMemoryPersistence` (`initializeApp(config, 'reg_...' /
'admin_...')`) para no sobrescribir la sesión del admin actual (ADR-011,
ADR-025).

Límites de la operación administrativa:

- `createAsignacion` valida la exclusividad conductor/fecha en el cliente con
  una lectura seguida de una escritura; no es una transacción atómica y dos
  administradores podrían competir. No valida exclusividad del bus: varios
  conductores pueden usar la misma placa.
- `hasActiveOrFutureBusAssignment` se usa para bloquear la eliminación de un bus
  con asignación activa o futura; no participa del alta de asignaciones.
- Desactivar un conductor no cancela sus asignaciones ni detiene su tracking.
  No existe eliminación física habitual de conductores.
- Eliminar un administrador borra su nodo, no su cuenta de Authentication.
- Crear un bus normaliza la placa y crea `/ubicacion_buses/{placa}` con
  `isActive:false`. Eliminarlo también elimina ese nodo.
- Los turnos son datos administrativos dentro de una asignación; no habilitan
  ni bloquean el tracking.

La estructura de archivos relevante está centralizada en §6.

### 3.2 PULSO (monitor de seguimiento)

Worker local de Cloudflare en `AdminWeb/monitor/` que observa el tracking
publicado en RTDB y comprueba su estado. No escribe en RTDB y no modifica
aplicaciones. Su acceso es **manual**: `GET /evaluate` solo responde en loopback
con la cabecera `X-Pulso-Local: 1` y el modo operativo no habilita cron
(`monitor/worker/src/index.ts`); cada comprobación se ejecuta a mano.

| Componente | Archivo | Rol |
|------------|---------|-----|
| Evaluador | `monitor/src/domain/evaluateTracking.ts` | Clasifica la publicación (fresca, desactualizada, desconocida), la interrupción y la recuperación |
| Cliente RTDB | `monitor/worker/src/firebaseRtdb.ts` | GET con timeout; distingue errores sin atribuir causa |
| Adaptador | `monitor/worker/src/toTrackingObservation.ts` | Acepta solo los seis campos publicados hoy; no inventa telemetría futura |
| Memoria | `monitor/worker/src/trackingStore.ts` + `migrations/` | Snapshot, historial e incidentes en D1 local |
| Incidentes | `monitor/worker/src/trackingIncidents.ts` | Apertura, actualización y cierre transaccional |

Umbrales vigentes (`monitor/worker/wrangler.jsonc`, entorno `operative`): una
publicación está desactualizada con antigüedad **mayor** a 180000 ms (3 min) y
la recuperación se confirma con **2** publicaciones válidas y distintas.

#### Incidentes del MVP (`migrations/0002_tracking_incidents.sql`)

| Tipo | Se abre | Se cierra |
|------|---------|-----------|
| `tracking-interrupted` (seguimiento interrumpido) | Durante vigilancia activa, cuando la antigüedad de publicación supera el umbral | `publications-recovered` al confirmar la recuperación del evaluador, o `tracking-inactive` al observar `isActive:false` (sin registrar recuperación ficticia) |
| `source-unavailable` (fuente inaccesible) | Durante vigilancia activa, cuando falla la consulta a Firebase | `source-accessible` al volver a poder leer, aunque la publicación siga desactualizada |

Reglas:

- La vigilancia (`monitoring_active`) comienza al observar `isActive:true` y se
  desactiva solo al observar `isActive:false`. Datos ausentes, inválidos o una
  fuente inaccesible la conservan. No hay horarios de jornada.
- Un problema continuo es un único incidente por fuente, bus y tipo (índice
  único parcial sobre incidentes abiertos); ambos tipos pueden coexistir. Cada
  comprobación continua incrementa `check_count` y `updated_at`.
- Estado, historial, poda e incidentes se escriben en un mismo batch D1 con CAS
  (`revision` + `last_check_id`); una comprobación perdedora revierte todo
  (`memory-conflict`). El historial de observaciones se poda a los 7 días; los
  incidentes no se podan.

#### Límites aceptados para el MVP

- Las consultas son manuales; no existe vigilancia automática.
- La duración guardada es el intervalo **observado** entre apertura y cierre; no
  se infiere la hora exacta de la caída.
- No identifica la causa de una interrupción (internet, GPS, batería, Android).
- Sin observar `isActive:false` no puede deducir por sí mismo el fin de la
  jornada.
- La migración `0002` conserva la memoria existente y no fabrica incidentes
  históricos.

Pendiente: vigilancia automática, notificaciones e integración con Central
(ROADMAP §2).

## 4. Arquitectura de BurritoUserApp

### Entry point, hidratación y splash

```
index.js → src/app/App.tsx → StackNavigator → DrawerNavigator → MapScreen
```

`App.tsx` configura Google Sign-In, cambia Firebase RTDB entre offline/online
según `AppState`, espera la hidratación de `userStore` y `themeStore`, y monta
`NavigationContainer`. El splash nativo de `react-native-bootsplash` se oculta
en `onReady` mediante `requestAnimationFrame`. Esa secuencia reduce la ventana
de splash sobre una UI vacía, pero no demuestra que todo trabajo asíncrono
posterior haya terminado ni que la primera pantalla esté completamente
pintada. Un watchdog de sesión limpia el estado local si el store indica sesión
pero Firebase Auth ya no entrega usuario.

### Navegación

```
StackNavigator (root)
├── logged-out: Welcome → SignIn/SignUp → ForgotPassword → AvatarPicker
└── logged-in:
    └── DrawerNavigator
        └── MapScreen (única pantalla del drawer; sin rutas admin)
```

El gating de autenticación es declarativo según `isLoggedIn`. No existen rutas
administrativas ni gating por `rol === 'admin'`. El módulo admin fue eliminado
de la app; la evolución está documentada en ADR-025 y en el historial de
`BUGS_RESUELTOS/adminweb.md`.

### Stores de Zustand

| Store | Persistencia | Propósito |
|-------|-------------|-----------|
| `userStore` | Zustand persist + AsyncStorage | uuid, username, avatar, nickname, email, rol (`'estudiante' \| null`), isLoggedIn |
| `themeStore` | AsyncStorage manual | isDarkMode |
| `mapStore` | Efímera | command (center) |
| `burritoLocationStore` | Efímera | locations, busMovementStates, movementDisplacements, movementMemory, isConnecting, connectionError, startTracking/stopTracking |
| `drawerStore` | Efímera | isOpen, open/close |

El listener RTDB vive en `burritoLocationStore`, no en componentes React. Esto
evita re-renders masivos del canvas y mantiene una única fuente de estado para
`Map.tsx` y `MapBranding.tsx`.

### Logout estudiantil

```
CustomDrawer → logoutStudent()
    ↓
firebaseAuth.signOut()
    ├── falla → no se confirma el cierre local
    └── confirma → onConfirmed() limpia la sesión local
                    ↓
          Google Sign-Out y Analytics se intentan sin bloquear el resultado
```

El cierre local solo se confirma después de que Firebase acepta `signOut`. La
limpieza de Google y el evento `sesion_cerrada` son best-effort: un fallo en
esos pasos no revierte el logout ya confirmado.

### Limpieza de recursos

`MapScreen` inicia `startTracking()` al montar y llama `stopTracking()` al
desmontar. `stopTracking()` libera el listener RTDB y los intervalos/timeout
del store. Esto acredita la limpieza de esos recursos, no la ausencia global de
cualquier listener o consumo en toda la aplicación.

La estructura de archivos relevante está centralizada en §6.

## 5. Comunicación y Flujo de Datos

Esta sección describe **qué componentes se comunican y con qué contrato**. El
ciclo de vida, las guardas y la recuperación del tracking se detallan en §8.

### 5.1 Pipeline de tracking

```
MapScreen.mount()
    ↓
burritoLocationStore.startTracking()
    ↓
MapService.subscribeToBusLocations() → ref.on('value')
    ↓
Recepción de Record<string, BurritoLocation>
    ↓
Validación I06 + deduplicación I05 + estrategia C experimental
    ↓
store.locations / busMovementStates
    ↓
Map.tsx / MapBranding.tsx → interpolación RNAnimated (~2 s) → render del marcador y estado
```

En el productor, `locationTask` recibe los fixes y los publica:

```
GPS watchPosition ─┐
                   ├─→ writeLocation() ─→ updateBusLocation() ─→ RTDB
Heartbeat (8 s) ───┘
```

`writeLocation()` es la única ruta para publicar posiciones GPS y heartbeat;
no es la única escritura de RTDB del ecosistema. `stopBusService()` escribe
`isActive:false`, y AdminWeb inicializa o elimina `/ubicacion_buses/{placa}` al
gestionar un bus.

Señales distintas que no deben confundirse:

| Señal | Origen | Significado |
|-------|--------|-------------|
| Fix GPS | `watchPosition` | Coordenadas, heading y speed entregados por el dispositivo |
| Intento de publicación | `writeLocation` | Se fija antes del `await`; el `timestamp` se toma al intentar publicar, no en el fix ni en el servidor |
| Pulso local | `PRO_LOCATION_PULSE` | Señal para la UI y el watchdog; se emite antes de intentar escribir y aunque la escritura falle |
| Resultado/ACK de la promesa | `updateBusLocation` / `stopBusService` | Éxito o fallo local de la operación; no certifica que otro cliente lo haya leído ni la frescura remota |
| Recepción | `on('value')` en El Burrito | Snapshot completo del nodo, con una hora de recepción por snapshot |

El heartbeat de 8 s solo publica si ya existe una `lastLocation` y si no hubo
un intento de escritura reciente. Un fallo de escritura reinicia el contador
para permitir el siguiente intento y evitar una cola propia. La lógica completa
está en §8.3.

### 5.2 Recepción RTDB: snapshots completos e I06

`MapService` se suscribe a `/ubicacion_buses` y recibe snapshots completos del
nodo; no recibe deltas independientes por bus. Por cada snapshot toma una hora
de recepción y valida cada entrada. La regla general I06 rechaza valores
explícitamente inválidos de coordenadas, timestamp e `isActive`; las excepciones
y límites aprobados son:

- **I06-A (limitación aceptada)**: los campos ausentes o `null` conservan los
  respaldos de coordenadas `0` y timestamp `Date.now()`. No se rechazan todos
  los registros incompletos. Una detención sin coordenadas válidas conserva la
  posición previa y pasa a offline; se mantiene I05.
- **I06-B**: un timestamp explícito mayor que la hora de recepción + 300 000 ms
  se rechaza y no se entrega al store. El límite de +300 000 ms es inclusivo; no
  se recorta. La última ubicación válida y su caducidad se conservan. El margen
  tolera diferencias de reloj, no deriva del heartbeat ni prueba sincronización
  de relojes.
- **Deduplicación I05**: se acepta un timestamp más nuevo; también se acepta la
  transición activo → inactivo con el mismo timestamp. Los datos viejos y las
  reactivaciones con timestamp igual se rechazan.

El mecanismo de recepción no borra una entrada por el hecho de que falte en un
snapshot: el store mezcla las entradas aceptadas con su estado anterior.

### 5.3 Estado visual, multi-bus y caducidad

La fuente de datos es multi-bus: el store conserva
`Record<string, BurritoLocation>`. Sin embargo, la UI actual no renderiza toda
la flota. `Map.tsx` y `MapBranding.tsx`:

1. Filtran entradas con `isActive !== false`.
2. Ordenan por `timestamp` descendente.
3. Seleccionan la primera entrada.

No buscan una alternativa fresca ni eligen un bus arbitrario. El render
multi-marcador completo sigue pendiente (ROADMAP T4.4).

La caducidad de 30 s se evalúa cada 2 s y cambia el estado del bus a `offline`
cuando `!isActive` o cuando la edad del timestamp alcanza el umbral. Ese estado
oculta el marcador y el radar en `Map.tsx`, pero no elimina necesariamente la
entrada de `locations` ni escribe `isActive:false` en RTDB. Un bus activo pero
viejo puede seguir siendo el seleccionado y mostrarse como `SIN SEÑAL`.

La clasificación en vivo usa la **estrategia C experimental** (distancia, Δt,
speed condicional y memoria con `STOP_CONFIRM_PAIRS=2`). `MOVEMENT_THRESHOLD_M`
(8 m) pertenece a la estrategia A histórica y no decide el estado en vivo. Los
parámetros de `movement.ts` son calibración pendiente, no valores definitivos
(ADR-020).

### 5.4 Feedback de El Burrito

`CustomDrawer` obtiene el UID de Firebase Auth y `MapService.sendFeedback()`
lo contrasta con la sesión vigente antes de crear un hijo mediante
`push().set()`, con `ServerValue.TIMESTAMP`. Las reglas publicadas exigen
creación con UID propio y rating entero 1–5; deniegan editar o borrar
comentarios. Los permisos de conductores y asignaciones se detallan en
`FIREBASE_SCHEMA.md` §9.

### 5.5 Autenticación de El Burrito

```
SignInScreen.handleEmailLogin()
    ↓
firebaseAuth.signInWithEmailAndPassword(email, password)
    ↓
Lee /usuarios/{uid}
    ↓
Actualiza /usuarios/{uid}/ultimaConexion = ServerValue.TIMESTAMP
    ↓
userStore.login(uid, username, avatar, email, 'estudiante')
    ↓
StackNavigator detecta isLoggedIn=true → DrawerNavigator → MapScreen
```

La API correcta es `signInWithEmailAndPassword`. El login con Google usa
`@react-native-google-signin/google-signin`, configurado con `webClientId` y
`offlineAccess:true`; `AvatarPickerScreen` completa el registro cuando
corresponde.

### 5.6 Flujo administrativo

```
createChofer:
    Auth temporal (${dni}@conductor.com)
    → /choferes/{dni}
    → /choferes_uids/{uid}

createBus:
    → /buses/{placa}
    → /ubicacion_buses/{placa} = { isActive:false }

createAsignacion:
    → /asignaciones/{pushId} con choferId, busId, fecha, turnos,
      createdAt y createdBy

updateTurnos:
    → actualiza turnos y espejo horaInicio/horaFin del mismo nodo

createAdmin:
    Auth temporal (${dni}@admin.com)
    → /administradores/{uid}
```

Los payloads completos, reglas e índices están en `FIREBASE_SCHEMA.md`. Las
validaciones y limitaciones del panel están en §3.1.

## 6. Componentes Principales

Inventario central de archivos relevantes. Los árboles completos de directorios
pertenecen al README de cada aplicación; esta tabla es el mapa de arquitectura.

### BurritoDriverApp

| Archivo | Responsabilidad |
|---------|----------------|
| `index.js` | Registro de la app; comentario sobre persistencia desactivada; import de gesture-handler |
| `src/DriverApp.tsx` | Auth gate (login ↔ tracking conductor); sincronización de Crashlytics |
| `src/screen/LoginDriverScreen.tsx` | Login con DNI, construcción del email y manejo de errores |
| `src/screen/SendCoordinates.tsx` | Asignación diaria, permisos, foreground service, GPS, heartbeat, watchdog, debug |
| `src/services/firebase_service.ts` | `updateBusLocation()` y `stopBusService()` contra RTDB |
| `src/shared/utils/timeout.ts` | Helper con timeout para lecturas RTDB |

### AdminWeb

| Archivo | Responsabilidad |
|---------|----------------|
| `src/main.tsx` | Entry point de la SPA (BrowserRouter + ToastProvider) |
| `src/App.tsx` | Auth gate: sesión → rol (`checking/admin/denied/error`) |
| `src/index.css` + `src/styles/*` | Sistema de diseño por tokens |
| `src/screen/LoginScreen.tsx` | Login admin (DNI → `{dni}@admin.com`) |
| `src/navigation/AdminNavigator.tsx` | Rutas del panel dentro de `AppShell` |
| `src/shared/layout/AppShell.tsx` | Sidebar, topbar y drawer móvil; provee la sesión |
| `src/shared/layout/SessionContext.tsx` | Datos del admin en sesión (`useSession`) |
| `src/shared/ui/*` | Primitivas de UI y overlays (Button, Field, Card, Modal/ConfirmDialog, Toast, Icon) |
| `src/features/admin/services/admin_service.ts` | CRUD con Auth temporal + RTDB |
| `src/features/admin/services/admin_check.ts` | `existeAdministrador(uid)` con timeout |
| `src/features/admin/screen/ChoferesScreen.tsx` | Gestión de conductores |
| `src/features/admin/screen/BusesScreen.tsx` | CRUD de buses |
| `src/features/admin/screen/AsignacionesScreen.tsx` | Asignaciones por fecha y edición de turnos |
| `src/features/admin/screen/AdminsScreen.tsx` | Gestión de administradores |
| `src/features/admin/components/TurnosEditor.tsx` | Editor reutilizable de turnos |
| `src/features/admin/utils/turno.ts` | Modelo, validación y formato de turnos |
| `src/features/admin/utils/turnoDraft.ts` | Estado de edición de turnos |
| `src/shared/config/env.ts` | Validación de variables `VITE_FIREBASE_*` |
| `src/shared/config/firebase.ts` | Instancia de Firebase (Firebase JS SDK) |
| `src/shared/utils/dni.ts` | Normalización y validación de DNI |
| `src/shared/utils/date.ts` | Fechas de calendario America/Lima |
| `src/shared/utils/format.ts` | Formateo de fechas y nombres de admin |

### El Burrito

| Archivo | Responsabilidad |
|---------|----------------|
| `src/app/App.tsx` | Montaje global, hidratación, Google Sign-In, Firebase offline/online, splash |
| `src/app/navigations/StackNavigator.tsx` | Auth gate declarativo |
| `src/app/navigations/DrawerNavigator.tsx` | Drawer con MapScreen |
| `src/features/map/screen/MapScreen.tsx` | Orquestador del mapa y ciclo start/stop del store |
| `src/features/map/components/Map.tsx` | Canvas Mapbox, selección y render del marcador, ruta, radar |
| `src/features/map/components/MapBranding.tsx` | Estado moving/stopped/offline |
| `src/features/map/components/CustomDrawer.tsx` | Perfil, tema, feedback y logout |
| `src/features/map/services/map_service.ts` | Listener RTDB, validación I06 y feedback |
| `src/features/map/utils/geo.ts` | Haversine (`calculateDistance`) |
| `src/features/map/utils/movement.ts` | Estrategia C experimental y su memoria |
| `src/features/auth/services/studentLogout.ts` | Logout estudiantil y tareas best-effort |
| `src/store/burritoLocationStore.ts` | Tracking, dedup, caducidad y clasificación de movimiento |
| `src/store/userStore.ts` | Sesión persistente del estudiante |
| `src/store/themeStore.ts` | Dark mode con AsyncStorage |
| `src/shared/config/firebase.ts` | Instancia compartida de Firebase |

## 7. Servicios Externos

Firebase es el backend administrado del ecosistema. El proyecto no mantiene un
servidor propio ni una API intermedia.

### Firebase

| Servicio | BurritoDriverApp | El Burrito | AdminWeb |
|----------|------------------|------------|----------|
| Firebase Auth | Login de conductor | Login email/password + Google Sign-In | Login de admin y creación de cuentas con app temporal |
| Firebase RTDB | `/ubicacion_buses/{busId}` (write), `/asignaciones` y `/choferes` (read) | `/ubicacion_buses` (listen), `/usuarios`, `/comentarios` | CRUD de configuración y lectura de listas |
| Firebase Analytics | No | Eventos de usuario (mapa abierto, mapa centrado) | No documentado como uso activo |
| Firebase Crashlytics | Sí (`setUserId` con UID o cadena vacía) | Configuración presente; no se acreditó telemetría activa en esta revisión | No documentado como uso activo |
| `ServerValue.TIMESTAMP` | No | Sí (`ultimaConexion`, feedback) | No; los registros administrativos usan `Date.now()` |
| Firebase Hosting | No | No | Sirve `dist/` con rewrites SPA y cache de assets |

> **AdminWeb** usa Firebase Auth para login y alta de cuentas, y RTDB para
> `/choferes`, `/buses`, `/asignaciones`, `/choferes_uids` y
> `/administradores`.

> **Identidad y diagnóstico (I11):** `DriverApp.tsx` solicita
> `setUserId(uid)` cuando Firebase Auth entrega un usuario y `setUserId('')`
> cuando entrega `null`, incluido el arranque sin sesión. La limpieza no
> elimina reportes históricos. El panel deja de imprimir nombre, DNI y prefijo
> del DNI; conserva GPS, heartbeat, Firebase, bus, asignaciones, permisos,
> watchdog, parada y errores con stack trace.

### Mapbox

- SDK: `@rnmapbox/maps` v10.2.10.
- Token público vía `@env` en El Burrito.
- Renderizado: ruta LineString, paraderos como puntos, marcador del bus con
  heading, radar animado y dark mode opcional.

### Google Sign-In

- SDK: `@react-native-google-signin/google-signin`.
- Configuración: `webClientId` desde `@env`.
- Inicializado en `App.tsx` con `offlineAccess:true`.
- Flujo: `SignInScreen` maneja el login Google; `AvatarPickerScreen` completa
  el registro con nick de facultad.

### react-native-dotenv

- El Burrito usa `module:react-native-dotenv` para inyectar variables desde
  `.env` vía `@env`.
- BurritoDriverApp no usa variables de entorno.

## 8. Ciclo de Vida del Tracking

### 8.1 Carga inicial de la asignación

`SendCoordinates` monta `loadAssignment()` al entrar con sesión. El ciclo hace
**dos intentos totales** de consulta (`ASSIGNMENT_RETRIES=2`), no dos reintentos
adicionales. Cada intento tiene un timeout de 10 s que cubre la lectura
opcional del nombre del conductor y la consulta de asignación. Entre el primero
y el segundo hay una pausa de 1.5 s.

La consulta usa `queryDailyAssignment()`: `orderByChild('choferId')` con
`equalTo(dni)` y resuelve la fecha en `America/Lima` al recibir la respuesta, no
al montar. Descarta registros cancelados, duplicados, de otro conductor o con
datos inválidos. El nombre del conductor es opcional: su fallo no invalida el
intento.

Estados visibles:

| Condición | Acción disponible |
|-----------|-------------------|
| Cargando asignación, sin controles de tracking | Spinner |
| Dos intentos agotados, sin controles | REINTENTAR y CERRAR SESIÓN |
| Asignación válida, detenido | INICIAR RECORRIDO y CERRAR SESIÓN |
| Sin bus asignado, detenido | INICIAR RECORRIDO y CERRAR SESIÓN (la revalidación al iniciar vuelve a bloquear si no hay asignación) |
| Iniciando, enviando, deteniendo o parada fallida | DETENER RECORRIDO; no se muestra CERRAR SESIÓN |

INICIAR no está disponible durante la carga inicial ni en la tarjeta de error.
La validación de asignación no ocurre solo al montar: se repite al presionar
INICIAR.

### 8.2 Secuencia de INICIAR

1. Se comprueba que no haya inicio, reinicio, parada o parada fallida en curso,
   y que el servicio no esté ya activo.
2. Se incrementa la generación, se marca `shouldRun=true` y `starting` con esa
   generación.
3. Se solicitan permisos en este orden exacto: `POST_NOTIFICATIONS` (API 33+),
   `ACCESS_FINE_LOCATION` y `ACCESS_BACKGROUND_LOCATION`. Cada denegación
   devuelve temprano.
4. Se revalida la asignación diaria con timeout de 10 s.
5. Si la respuesta conocida es inválida, ausente, cancelada, contradictoria o de
   otro conductor, se revoca el respaldo y se bloquea el inicio.
6. Si la consulta falla, solo se permite continuar cuando `.info/connected`
   reporta `false` y existe un respaldo en memoria del mismo conductor y día.
   Un timeout aislado no habilita el respaldo.
7. Se vuelve a comprobar que la fecha siga siendo la de Lima. Si cambió el día,
   se revoca el respaldo y se pide presionar INICIAR de nuevo.
8. Se fija `trackingIntent.run = { driverDni, busId }` y se inicia el
   `BackgroundJob` con las opciones de §9.1. La generación se revalida después
   de cada espera.

La revalidación ocurre **después** de los permisos y **antes** de arrancar el
servicio. No se reordena esa secuencia.

### 8.3 GPS, heartbeat y watchdog

`locationTask` ejecuta `Geolocation.watchPosition` con `enableHighAccuracy`,
`distanceFilter:2`, `interval:3000` y `fastestInterval:2000`. El callback
guarda la última posición, emite el pulso local y llama a `writeLocation()`.

`writeLocation()` usa `updateBusLocation(busId, {...coords, timestamp:
Date.now()})`. Fija `lastWriteAt` antes del `await`, para que el heartbeat no se
salte fuegos ni duplique escrituras; en fallo lo reinicia para permitir el
siguiente intento. `timestamp` es el instante del intento de publicación, no el
del fix GPS. Con bus quieto, el heartbeat refresca ese timestamp.

El heartbeat corre cada 8 s dentro del mismo `locationTask`. Solo escribe si ya
hay una `lastLocation` y si pasaron al menos 8 s desde el último intento. Emite
el pulso antes de escribir, incluso si la escritura falla, y no abre una segunda
ruta de escritura. Un `keepAlive` de 1 s verifica que el servicio siga en
ejecución y limpia la ejecución si dejó de estarlo.

El watchdog es un efecto de la pantalla, no del servicio. Se activa mientras
`isSending` es verdadero, revisa cada 10 s y reinicia si pasaron más de 30 s sin
pulso. Antes de reiniciar exige intención/generación vigente y ausencia de
inicio, reinicio o parada en curso. Si el reinicio falla, detiene la UI, emite
un Alert y deja un banner persistente.

El watchdog solo puede actuar mientras la pantalla y el proceso JavaScript
sigan vivos. Al desmontar la pantalla se elimina su intervalo y sus listeners,
pero el motor GPS puede continuar en el foreground service. Al remontar, la UI
restaura el estado si `shouldRun`, `sending` y `BackgroundJob.isRunning()`
siguen vigentes; el watchdog nuevo empieza con un reloj nuevo. No es una
garantía de supervivencia del proceso ni cubre un kill del sistema operativo.

### 8.4 DETENER

DETENER captura la identidad del recorrido antes del primer `await`, invalida
la intención/generación, limpia los recursos locales y serializa la parada
nativa con las demás transiciones.

- Si `BackgroundJob.stop()` falla, la parada local no está confirmada: se
  bloquea INICIAR y se permite reintentar DETENER.
- Si la parada nativa tiene éxito, se solicita `stopBusService(run.busId)`
  (`isActive:false`) antes de liberar INICIAR, sin esperar la confirmación
  remota.
- El resultado de la operación remota solo informa diagnóstico. Un ACK antiguo
  no reescribe el estado local ni cambia la identidad del recorrido nuevo.
- `trackingIntent.run` se libera cuando la parada local se confirma; el
  resultado remoto no decide esa liberación.

`isActive:false` se escribe en la parada funcional. Un kill anormal del proceso,
un crash nativo o una pérdida de ejecución pueden dejar `isActive:true`; el
watchdog dentro del proceso no cubre ese caso. No hay `onDisconnect`.

### 8.5 Cerrar sesión y relevos

CERRAR SESIÓN ejecuta `auth().signOut()` y vuelve a `LoginDriverScreen`. No
detiene el servicio ni escribe `isActive:false`; es una acción distinta de
DETENER. Su visibilidad en la UI no es simultánea con el recorrido activo.

El piloto usa un teléfono compartido: la sesión se mantiene durante los
relevos y el conductor no necesariamente cierra sesión al terminar su turno. La
cuenta autenticada identifica la sesión del teléfono, no necesariamente a la
persona física que lleva el bus en ese momento. Esta es una condición operativa
del MVP y una limitación de identidad, no una función de cambio automático de
conductor.

### 8.6 Identidad I08 y respaldo offline

Al validar INICIAR, `trackingIntent.run` fija `{driverDni, busId}`. Esa
identidad la usan GPS y la parada aunque la pantalla se remonte o la asignación
cambie después. Los `turnos` son informativos: no habilitan ni bloquean el
tracking (ADR-028).

El respaldo de asignación vive en un `useRef` de la pantalla, no en el objeto
compartido de intención. Solo sirve para el mismo conductor y la misma fecha.
No persiste entre procesos, se revoca ante una respuesta conocida como inválida
y no permite adivinar altas, cancelaciones ni cambios administrativos ocurridos
durante la desconexión. Un timeout aislado no equivale a offline.

`.info/connected` debe consultarse sin barra inicial (`ref('.info/connected')`)
porque el SDK instalado lanza excepción con `/`. Que reporte `false` habilita el
respaldo, pero no prueba entrega ni confirmación de escrituras remotas. Ver
`BUGS_RESUELTOS/driverapp.md` (CASO 021).

### 8.7 I07: idempotencia, restauración y HeadlessJS

El foreground service se implementa con `react-native-background-actions` 4.1.0
y `RNBackgroundActionsTask`. En la configuración actual, la Activity y el Service
comparten el proceso Android, pero tienen ciclos de vida distintos.

`trackingIntent` comparte intención, generación y transiciones entre montajes.
DETENER invalida antes del primer `await`. Cada ejecución de GPS tiene identidad
propia y un `dispose` idempotente para su watch, heartbeat y keepAlive. Las
guardas de callbacks y continuaciones impiden que una ejecución cancelada vuelva
a publicar o altere a su sucesora.

Un fallo de `stop()` no es éxito: conserva el error, bloquea INICIAR y permite
reintentar DETENER. Tras el éxito nativo, la solicitud remota se emite antes de
liberar INICIAR, sin esperar confirmación. Los ACK antiguos solo informan
diagnóstico como operación anterior.

Al re-montar, `SendCoordinates` recupera el estado compartido y escucha
`PRO_TRACKING_STATE` para operaciones iniciadas antes del remontaje. No basta
`isRunning()`: se exige intención vigente y envío activo, sin inicio o
recuperación en curso. Una parada cancelada no revive; un recorrido legítimo se
restaura sin crear otro motor. El desmontaje no detiene por sí mismo el
servicio.

La promesa de usuario de `locationTask` queda pendiente al limpiar para evitar
el `stop()` global automático del wrapper al resolverla. En esa versión, entregas
tardías pueden sobrescribir el resolver global y dejar envolturas pendientes. Los
tests caracterizan ese residual; no hay dos motores GPS
completos simultáneos. Ver `BUGS_RESUELTOS/driverapp.md` (CASO 020).

### 8.8 Persistencia y estado de la UI

La persistencia en disco no está activada: no se llama a
`database().setPersistenceEnabled(true)` (ADR-003). Desactivar la persistencia
en disco no elimina las escrituras pendientes en memoria del SDK durante una
desconexión: pueden confirmarse al reconectar. No se garantiza que cada intento
offline se descarte. I07 no agrega cola propia ni cancela escrituras ya
solicitadas; separa su confirmación del cierre local.

La etiqueta «Servicio de ubicación activo» describe el servicio Android local en
ejecución. No afirma que la última posición se haya publicado en RTDB ni
distingue conectado/desconectado. La UI no agrega listeners de conectividad en
esta fase (ADR-028).

## 9. Consideraciones de Implementación y Límites

### 9.1 Android 14 y foreground service

`foregroundServiceType` debe declararse en tres lugares coherentes:

1. `AndroidManifest.xml`: `<service android:foregroundServiceType="location" />`.
2. Permiso: `FOREGROUND_SERVICE_LOCATION` (además de `FOREGROUND_SERVICE`,
   `ACCESS_BACKGROUND_LOCATION` y `POST_NOTIFICATIONS`).
3. Opciones JS de `getBackgroundOptions()`:
   `foregroundServiceType: ['location' as const]`.

La configuración actual también usa `android:stopWithTask="false"` para que
cerrar la tarea no detenga el servicio. Eso no cubre un kill del proceso ni una
restricción del fabricante.

Un faltante de configuración puede producir excepciones específicas como
`MissingForegroundServiceTypeException` o `SecurityException`; no todas las
causas producen la misma `InvalidForegroundServiceTypeException`. Un inicio
desde background puede ser rechazado con
`ForegroundServiceStartNotAllowedException`, que la app informa al conductor.

El foreground service reduce la suspensión mientras está correctamente
configurado y autorizado, pero no garantiza ejecución ilimitada, inmunidad a
Doze, una cadencia exacta ni supervivencia a la terminación del proceso. Sin el
servicio, `watchPosition` puede detenerse al minimizar la app. No se debe
atribuir al sistema una garantía que el código no controla.

### 9.2 Multi-bus, caducidad y deuda pendiente

La selección de un solo bus, la caducidad y el estado `offline` se explican en
§5.3. El render multi-marcador completo sigue pendiente (ROADMAP T4.4) y no debe
presentarse como implementado.

### 9.3 Geofencing y Haversine

`calculateDistance()` está en `User/src/features/map/utils/geo.ts` y lo usa
`burritoLocationStore.ts` para la clasificación de movimiento. No está en
`Map.tsx`.

La arquitectura actual no incorpora geofencing, cierre automático de vueltas ni
escritura a `/recorridos`. Su planificación está en ROADMAP T4.3. Haversine ya
se utiliza; geofencing sigue pendiente.

### 9.4 Índices de RTDB

La consulta I08 usa `orderByChild('choferId')` sobre `/asignaciones`. El índice
`.indexOn` debe existir en la configuración desplegada para evitar advertencias
y escaneos costosos:

```json
{
  "asignaciones": {
    ".indexOn": "choferId"
  }
}
```

El índice es una optimización de rendimiento, no un mecanismo de autorización.
Las bibliotecas cliente pueden ejecutar consultas ad hoc sin él; la ausencia de
índice no convierte por sí sola la consulta en una denegación de permisos. La
autorización se define en las reglas de seguridad, que tampoco actúan como
filtros de resultados. La consulta debe cumplir las reglas y, por separado,
conviene que tenga el índice para operar eficientemente.

### 9.5 Compatibilidad iOS

BurritoDriverApp no está probada ni soportada en iOS: el foreground service
depende de APIs nativas de Android.

En El Burrito existe configuración iOS, pero el soporte funcional **no está
acreditado**. La inspección encontró un `Podfile` con target
`BurritoFeatureTest`, un `AppDelegate.swift` que registra el módulo
`BurritoFeatureTest`, un `app.json` con nombre `BurritoUserApp` y referencias
heredadas a `MapsApp` en el proyecto Xcode. No se encontró un
`GoogleService-Info.plist` versionado ni la inicialización de Firebase en el
AppDelegate inspeccionado. Esto no autoriza a declarar soporte iOS completo ni a
reparar esa configuración desde este documento.

### 9.6 Productores y simulación

El productor actual acreditado en el código es BurritoDriverApp, que escribe en
`/ubicacion_buses/{busId}`. No se acreditó en esta revisión un script Python
versionado, una carpeta de simulación ni una integración actual que escriba a
otro path como segundo productor. Si existe una herramienta externa de
simulación, debe documentarse por separado con su ruta, propósito y estado; no
se presenta aquí como parte de la topología productiva.

## 10. Referencias

| Documento | Relación |
|-----------|----------|
| `FIREBASE_SCHEMA.md` | Estructura detallada de nodos, payloads, reglas e índices de RTDB. |
| `DECISIONS.md` | Decisiones de arquitectura (ADR) con razonamiento técnico. |
| `ROADMAP.md` | Planificación posterior al MVP y dependencias. |
| `../MVP.md` | Operación del piloto, aceptación de jornada y supuestos de relevo. |
| `TROUBLESHOOTING.md` | Incidentes, diagnóstico y soluciones operativas. |
| `BUGS_RESUELTOS/` | Historial de bugs resueltos y evidencia de validación. |
| `../Driver/README.md` | Setup, requisitos y funcionalidad de BurritoDriverApp. |
| `../User/README.md` | Setup, requisitos y funcionalidad de El Burrito. |
| `../AdminWeb/README.md` | Setup, requisitos y funcionalidad del panel administrativo. |
