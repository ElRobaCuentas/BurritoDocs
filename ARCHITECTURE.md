# Architecture — El Burrito

## 1. Visión General

El sistema sigue el patrón **productor-consumidor** sobre una arquitectura
**serverless** y **unidireccional**:

```
DriverApp (productor GPS) ──escribe──→ Firebase RTDB (bus) ──escucha──→ UserApp (consumidor)
AdminWeb (gestión admin) ──escribe──→ Firebase RTDB (config)
```

No hay backend intermedio. Firebase Realtime Database actúa como único bus
de mensajes, sincronizando los datos entre ambas aplicaciones mediante
websockets nativos del SDK.

## 2. Arquitectura del Ecosistema

### Separación de responsabilidades (post-migración)

| Aplicación | Rol | Función principal | Autenticación |
|-----------|-----|-------------------|---------------|
| BurritoDriverApp | **Conductor** | Capturar GPS y escribir en RTDB (conductor-only) | DNI + contraseña |
| BurritoUserApp | **Consumidor** (estudiante) | Leer desde RTDB y renderizar mapa | Email/contraseña o Google Sign-In |
| AdminWeb | **Administración** | Gestionar conductores, buses y asignaciones | Auth UID (admin) |
| Firebase RTDB | **Bus de datos** | Almacenar y sincronizar estado en vivo | — |
| Firebase Auth | **Identidad** | Autenticar conductores, estudiantes y administradores | — |

### Flujo de datos

```
┌────────────────────┐  write   ┌────────────────────┐
│ BurritoDriverApp    │ ───────→ │  Firebase RTDB     │
│ (Conductor)         │ubicación │  /ubicacion_buses  │
└────────────────────┘          │  /asignaciones     │
┌────────────────────┐  write   │  /choferes         │
│ AdminWeb            │ ───────→ │  /buses            │
│ (Administrador)     │ gestión  │                    │
└────────────────────┘          │  /choferes_uids    │
┌────────────────────┐  listen  │  /administradores  │
│ BurritoUserApp      │ ←─────── │  /usuarios         │
│ (Estudiante)        │          │  /comentarios      │
└────────────────────┘          └────────────────────┘
```

### Autenticación

- **DriverApp (conductores)**: Firebase Auth con email compuesto como
  `${dni}@conductor.com`. Contraseña = DNI del conductor, establecida al
  crear el chofer desde AdminWeb.
- **AdminWeb (administradores)**: Firebase Auth con email
  `${dni}@admin.com`. Tras el login, el auth gate consulta
  `/administradores/{auth.uid}` para autorizar el acceso al panel.
- **UserApp (estudiantes)**: Firebase Auth con email/contraseña (registro
  estándar) o Google Sign-In.

### Separación de proyectos

El ecosistema tiene tres aplicaciones independientes dentro del mismo
repositorio: dos proyectos React Native (**BurritoDriverApp** y
**BurritoUserApp**) y una SPA web (**AdminWeb**). Comparten la misma base
de datos Firebase y el mismo proyecto `burritounmsm`, pero tienen su
propio `package.json`, entry point y configuración.

## 3. Arquitectura de BurritoDriverApp (conductor)

### Entry point y enrutador

```
index.js → src/DriverApp.tsx
```

`DriverApp.tsx` monta `onAuthStateChanged` y decide entre login y tracking.
La DriverApp es **exclusiva de conductores**: no incluye panel de
administración ni consulta `/administradores` (el panel vive en AdminWeb,
ADR-025).

```
Login exitoso
    ↓
onAuthStateChanged → user detectado
    ├── user → SendCoordinates(driverDni = email.split('@')[0])
    └── null → LoginDriverScreen
```

### Estructura de archivos

```
src/
├── DriverApp.tsx                    # Auth gate (login ↔ tracking)
├── screen/
│   ├── LoginDriverScreen.tsx        # Login con DNI + contraseña
│   └── SendCoordinates.tsx          # Tracking, permisos, GPS
├── services/
│   └── firebase_service.ts          # Escritura a RTDB (updateBusLocation)
└── shared/
    ├── theme/
    │   ├── colors.ts
    │   └── typography.ts
    └── utils/timeout.ts             # Helper con timeout para lecturas RTDB
```

### LoginDriverScreen

- Input de DNI y contraseña.
- Construye email como `${dni}@conductor.com` y llama a
  `auth().signInWithEmailAndPassword`.
- Manejo de errores: red, credenciales inválidas, cuenta deshabilitada.

### SendCoordinates (pantalla principal del conductor)

Contiene la lógica completa de tracking. Ver sección 8 para el ciclo
detallado.

### Firebase Auth (DriverApp)

- `auth().onAuthStateChanged()` en `DriverApp.tsx` maneja la sesión.
- `auth().signOut()` disponible desde SendCoordinates (acción "Cerrar
  sesión", **independiente** de detener el recorrido — ADR-026).
- La escritura de ubicación está restringida por reglas a conductores
  indexados en `/choferes_uids` o a administradores (ADR-023).

---

## 3.1 Arquitectura de AdminWeb (administrador)

### Entry point y enrutador

```
src/main.tsx → src/App.tsx
```

`App.tsx` es el auth gate: escucha `onAuthStateChanged`; con sesión,
consulta `existeAdministrador(uid)` sobre `/administradores/{uid}` (con
timeout de 10 s) y decide entre el panel, la pantalla "No autorizado" o un
estado de error con reintento / cierre de sesión.

### Estructura de archivos

```
src/
├── main.tsx                         # Entry point
├── App.tsx                          # Auth gate: sesión → admin check → role error
├── screen/
│   └── LoginScreen.tsx              # Login admin (DNI + contraseña → {dni}@admin.com)
├── navigation/
│   └── AdminNavigator.tsx           # Rutas del panel
├── features/admin/
│   ├── screen/
│   │   ├── AdminPanelScreen.tsx     # Menú principal del admin
│   │   ├── ChoferesScreen.tsx       # CRUD de conductores
│   │   ├── BusesScreen.tsx          # CRUD de buses
│   │   ├── AsignacionesScreen.tsx   # Asignaciones diarias conductor ↔ bus
│   │   └── AdminsScreen.tsx         # Gestión de administradores
│   └── services/
│       ├── admin_service.ts         # CRUD (Firebase Auth + RTDB)
│       └── admin_check.ts           # existeAdministrador(uid): Promise<boolean>
└── shared/
    ├── config/firebase.ts           # Firebase Auth + RTDB (Firebase JS SDK)
    └── utils/timeout.ts             # Helper con timeout para reads de RTDB
```

### Módulo Admin

Accesible solo para usuarios cuyo `auth.uid` existe en
`/administradores/`. CRUD completo contra RTDB:

| Entidad | Path | Operaciones |
|---------|------|-------------|
| Choferes | `/choferes/{dni}` | Crear (Auth + RTDB + índice `/choferes_uids`), listar, editar, toggle activo, eliminar |
| Buses | `/buses/{placa}` | Crear + init `/ubicacion_buses/{placa}`, listar, toggle, eliminar |
| Asignaciones | `/asignaciones/{pushId}` | Crear con validación de exclusividad, listar, cancelar |
| Administradores | `/administradores/{uid}` | Crear (Auth + RTDB), listar, editar, eliminar (nodo) |

Al crear o eliminar cuentas de Auth, `admin_service.ts` usa una **app
temporal por operación** con `inMemoryPersistence`
(`initializeApp(config, 'reg_...' / 'del_...' / 'admin_...')`) para no
sobrescribir la sesión del admin actual (ver ADR-011 y ADR-025).

### admin_check.ts

`existeAdministrador(uid: string): Promise<boolean>` consulta
`/administradores/{uid}` con `get()` y retorna `snapshot.exists()`, acotado
por un timeout de 10 s (C4.AUTH). Es la fuente de verdad del auth gate de
AdminWeb.

### Navegación

```
Login → (sesión) → existeAdministrador(uid)?
    ├── Sí    → AdminNavigator (AdminPanelScreen → Choferes / Buses / Asignaciones / Admins)
    ├── No    → "No autorizado" (Cerrar Sesión)
    └── Error → "No se pudo verificar tu cuenta" (Reintentar / Cerrar Sesión)
```

---

## 4. Arquitectura de BurritoUserApp

### Entry point

```
index.js → src/app/App.tsx
```

`App.tsx` orquesta: gestor de navegación, stores de Zustand, Firebase
offline/online según estado de la app, Google Sign-In config, splash
nativo (react-native-bootsplash) y gating de hidratación. El splash se
oculta en `onReady` del `NavigationContainer`, garantizando que la
primera pantalla ya está pintada e hidratada (ADR-021).

### Navegación (post-poda)

```
StackNavigator (root)
├── logged-out: Welcome → SignIn/SignUp → ForgotPassword → AvatarPicker
└── logged-in:
    └── DrawerNavigator
        └── MapScreen (única pantalla en el drawer — sin rutas admin)
```

El gating de autenticación es declarativo. Ya no existen rutas
administrativas ni gating por `rol === 'admin'`. El módulo admin fue
eliminado completamente (ver FASE 4 de la migración).

### Estructura de archivos (post-poda)

```
src/
├── app/
│   ├── App.tsx                        # Entry point, Firebase offline/online
│   └── navigations/
│       ├── StackNavigator.tsx         # Root navigator con auth gate (sin admin)
│       └── DrawerNavigator.tsx        # Menú lateral con MapScreen
├── features/
│   ├── auth/screen/                   # Welcome, SignIn, SignUp, ForgotPassword, AvatarPicker
│   └── map/
│       ├── screen/MapScreen.tsx       # Contenedor del mapa + overlay UI
│       ├── components/
│       │   ├── Map.tsx                # Mapbox canvas con marcadores, ruta y paraderos
│       │   ├── CustomDrawer.tsx       # Menú lateral animado (sin botón admin)
│       │   ├── FAB.tsx                # Botón flotante
│       │   ├── StopCard.tsx           # Modal de información de paradero
│       │   └── MapBranding.tsx        # Marca de agua del mapa
│       ├── constants/
│       │   └── map_route.ts           # GeoJSON de ruta y paraderos
│       ├── utils/
│       │   ├── movement.ts            # C4.6 estrategia C (MOVIMIENTO/PARADERO)
│       │   └── geo.ts                 # Utilidades geográficas (Haversine)
│       ├── services/map_service.ts    # Suscripción RTDB + feedback
│       └── types.ts
├── shared/
│   ├── config/firebase.ts             # Instancia compartida de RTDB y Auth
│   └── theme/
│       ├── colors.ts
│       └── typography.ts
└── store/
    ├── userStore.ts                   # Persistida, solo rol 'estudiante' | null
    ├── themeStore.ts                  # Persistencia manual (AsyncStorage)
    ├── mapStore.ts                    # Estado efímero del mapa
    ├── burritoLocationStore.ts        # Estado efímero de ubicación del bus
    └── drawerStore.ts                 # Estado efímero del menú lateral
```

### Stores de Zustand

| Store | Persistencia | Propósito |
|-------|-------------|-----------|
| `userStore` | Zustand persist + AsyncStorage | uuid, username, avatar, rol (solo 'estudiante'), isLoggedIn |
| `themeStore` | AsyncStorage manual | isDarkMode |
| `mapStore` | Efímera | command (center) |
| `burritoLocationStore` | Efímera | locations (Record<string,BurritoLocation>), busMovementStates, movementDisplacements, movementMemory, isConnecting, connectionError, startTracking/stopTracking |
| `drawerStore` | Efímera | isOpen, open/close |

### Módulo Admin (eliminado)

El módulo `features/admin/` fue eliminado de la UserApp. El panel de
gestión reside exclusivamente en **AdminWeb** (ADR-025). El botón "Panel
de Gestión" ya no existe en el CustomDrawer. El campo `rol` en `userStore`
se redujo a `'estudiante' | null`.

## 5. Flujo de Datos

### Flujo de tracking (inicio a fin)

```
Conductor presiona INICIAR
    ↓
requestAllPermissions()
    ↓ (permisos OK)
BackgroundJob.start(locationTask, options)
    ↓
locationTask:

  GPS callback (watchPosition)
      │
      ├──────────────► writeLocation()          ← única ruta de escritura
      │
  Heartbeat (8 s, T4.1)
      │
      └──────────────► writeLocation()

                    │
                    ▼
        updateBusLocation(busId, { latitude, longitude,
                                   heading, speed,
                                   timestamp, isActive: true })
                    │
                    ▼
                Firebase RTDB         ← /ubicacion_buses/{busId}
                    │
                    ▼
     UserApp recibe el delta vía subscribeToBusLocations()
```

Notas sobre el ciclo de tracking:

- `writeLocation()` es la **única** función que escribe en RTDB. Tanto el
  callback del GPS como el heartbeat pasan por ella para no divergir
  (T4.1).
- El heartbeat (cada 8 s) solo escribe si no hubo otra escritura reciente
  (no duplica el trabajo del GPS) y si ya existe una `lastLocation` de la
  primera lectura GPS.
- `timestamp` es la última actualización publicada, no el instante del fix
  GPS. Con bus quieto lo refresca el heartbeat.

### Flujo de visualización (UserApp — multi-bus)

```
UserApp inicia
    ↓
MapScreen.mount()
    ↓
burritoLocationStore.startTracking()
    ↓
MapService.subscribeToBusLocations()  ← se suscribe a /ubicacion_buses
    ↓
ref.on('value', callback)            ← Firebase empuja deltas
    ↓
Callback recibe Record<string, BurritoLocation> { placa1: {lat, lng, speed, ...}, placa2: {...}, ... }
    ↓
Filtro dedup por placa (timestamp > actual por cada bus)
    ↓
C4.6 EXPERIMENTAL — estrategia C (decide EN MOVIMIENTO/EN PARADERO):
    dist = Haversine(pos previa, pos nueva)
    dtMs = timestamp nuevo − timestamp previo
    evidence = vCalc > 1 m/s  OR  (speed > 2 m/s AND dist > 2 m)
    vCalc = dist / (dtMs/1000)
    Memoria por placa: evidencia → moving; 2 parejas consecutivas sin
    evidencia → stopped (STOP_CONFIRM_PAIRS=2). El desplazamiento de 8 m
    (estrategia A) NO participa en esta decisión.
    Diagnóstico temporal: CALIB_LOG_ENABLED → console.log [C4.6] por
    actualización (placa, dist, dtMs, vCalc, speed, evidence, transición,
    weakCount). Se elimina tras la validación en campo.
    ↓
Set en burritoLocationStore.locations + busMovementStates + movementMemory
    ↓
Map.tsx deriva burritoLocation del primer bus activo (fallback)
    ↓
RNAnimated interpola posición del marcador (~2s)
    ↓
Mapbox actualiza canvas

Desmontaje (C4.4):
MapScreen.unmount() → cleanup → burritoLocationStore.stopTracking()
    ↓
unsubscribe() (ref.off('value', callback))  ← se libera el listener RTDB
    ↓
clearInterval(onlineInterval de 2s)          ← se libera el ciclo de vida
    ↓
0 listeners / 0 intervalos activos (sin consumo residual)
```

### Flujo de autenticación (DriverApp — conductor)

```
LoginDriverScreen.handleLogin()
    ↓
auth().signInWithEmailAndPassword(
    `${dni}@conductor.com`, password
)
    ↓
Firebase Auth valida credenciales
    ↓
onAuthStateChanged en DriverApp.tsx detecta user
    ↓
SendCoordinates(driverDni = email.split('@')[0])
    ↓
SendCoordinates monta useEffect
    ↓
loadAssignment() (C4.3): lee /choferes/{dni} y /asignaciones con
timeout de 10s por intento y hasta 2 reintentos acotados
    ↓
Filtra por fecha===today && activo===true
    ├── Sí: guarda busId en estado local
    └── No / agota reintentos:
        ├── Sin asignación válida → "No tienes un bus asignado"
        └── Error de red → errorCard (REINTENTAR / CERRAR SESIÓN)
```

### Flujo de autenticación (UserApp)

```
SignInScreen.handleLogin()
    ↓
auth().signInWithEmailPassword(email, password)
    ↓
Lee /usuarios/{uid}.once('value')  → obtiene nombre, avatar
    ↓
Actualiza /usuarios/{uid}/ultimaConexion = ServerValue.TIMESTAMP
    ↓
userStore.login(uuid, username, avatar, email, 'estudiante')
    ↓
StackNavigator detecta isLoggedIn=true → renderiza DrawerNavigator (MapScreen)
```

### Flujo admin (AdminWeb)

```
Admin crea chofer:
    ChoferesScreen → createChofer({ dni, nombre, apellidos })
        → app temporal: createUserWithEmailAndPassword(`${dni}@conductor.com`, dni)
        → /choferes/{dni}.set({ nombre, apellidos, activo: true, uid, creadoEn, creadoPor })
        → /choferes_uids/{uid}.set(dni)

Admin crea bus:
    BusesScreen → createBus({ placa, modelo, marca, anio })
        → /buses/{placa}.set({ modelo, marca, anio, activo: true })
        → /ubicacion_buses/{placa}.set({ isActive: false })

Admin asigna:
    AsignacionesScreen → createAsignacion(choferId, busId)
        → Valida que chofer y bus no tengan asignación activa hoy
        → /asignaciones.push().set({ choferId, busId, fecha, activo: true, createdAt, createdBy })

Admin crea admin:
    AdminsScreen → createAdmin(dni, nombres, apellidos, password)
        → app temporal: createUserWithEmailAndPassword(`${dni}@admin.com`, password)
        → /administradores/{uid}.set({ dni, nombres, apellidos, correo })
```

## 6. Componentes Principales

### BurritoDriverApp

| Archivo | Responsabilidad |
|---------|----------------|
| `index.js` | Registro de la app, comentario sobre persistence desactivada, import gesture-handler |
| `src/DriverApp.tsx` | Auth gate (login ↔ tracking conductor) |
| `src/screen/LoginDriverScreen.tsx` | Login con DNI, manejo de errores de autenticación |
| `src/screen/SendCoordinates.tsx` | Asignación de bus, permisos, foreground service, GPS, debug console |
| `src/services/firebase_service.ts` | `updateBusLocation()` y `stopBusService()` contra RTDB |
| `src/shared/utils/timeout.ts` | Helper con timeout para lecturas RTDB |

### AdminWeb

| Archivo | Responsabilidad |
|---------|----------------|
| `src/main.tsx` | Entry point de la SPA |
| `src/App.tsx` | Auth gate: sesión → admin check → pantalla de rol/error |
| `src/screen/LoginScreen.tsx` | Login admin (DNI → `{dni}@admin.com`) |
| `src/navigation/AdminNavigator.tsx` | Rutas de las pantallas del panel |
| `src/features/admin/services/admin_service.ts` | CRUD (Auth temporal + RTDB) |
| `src/features/admin/services/admin_check.ts` | `existeAdministrador(uid)` con timeout |
| `src/features/admin/screen/AdminPanelScreen.tsx` | Menú principal del admin |
| `src/features/admin/screen/ChoferesScreen.tsx` | CRUD de conductores |
| `src/features/admin/screen/BusesScreen.tsx` | CRUD de buses |
| `src/features/admin/screen/AsignacionesScreen.tsx` | Asignaciones diarias |
| `src/features/admin/screen/AdminsScreen.tsx` | Gestión de administradores |
| `src/shared/config/firebase.ts` | Instancia de Firebase (Firebase JS SDK) |

### BurritoUserApp

| Archivo | Responsabilidad |
|---------|----------------|
| `src/app/App.tsx` | Montaje global, splash nativo (bootsplash), hidratación, Firebase offline/online |
| `src/features/map/screen/MapScreen.tsx` | Orquestador del mapa, inicializa tracking |
| `src/features/map/components/Map.tsx` | Mapbox canvas, marcadores, ruta, radar |
| `src/features/map/components/CustomDrawer.tsx` | Drawer animado con perfil, tema, feedback (sin admin) |
| `src/features/map/services/map_service.ts` | Listener RTDB a `/ubicacion_buses`, envío de feedback |
| `src/store/burritoLocationStore.ts` | Estado del tracking, dedup, clasificación moving/stopped/offline |
| `src/store/userStore.ts` | Sesión persistente del usuario (solo 'estudiante') |
| `src/store/themeStore.ts` | Dark mode con persistencia AsyncStorage |
| `src/shared/config/firebase.ts` | Instancia compartida de Firebase |

## 7. Servicios Externos

### Firebase

| Servicio | Uso en DriverApp | Uso en UserApp |
|----------|-----------------|----------------|
| Firebase Auth | Login de conductor (App móvil). *AdminWeb:* login de admin y creación de cuentas con app temporal | Login email/password + Google Sign-In |
| Firebase RTDB | `/ubicacion_buses/{busId}` (write), `/asignaciones` (read) | `/ubicacion_buses`, `/usuarios`, `/comentarios` |
| Firebase Analytics | No | Eventos de usuario (mapa abierto, mapa centrado) |
| Firebase Crashlytics | Sí (`setUserId` con el uid del conductor) | Configurado en firebase.json |
| ServerValue.TIMESTAMP | No | Sí (6 ocurrencias: ultimaConexion, feedback) |

> **AdminWeb** usa Firebase Auth (login de admin + creación/eliminación de
> cuentas vía app temporal) y RTDB (gestión de `/choferes`, `/buses`,
> `/asignaciones`, `/choferes_uids` y `/administradores`).

### Mapbox

- SDK: `@rnmapbox/maps` v10.2.10
- Token público vía `@env` (UserApp)
- Renderizado: ruta LineString, paraderos como puntos, marcador del bus
  con heading, radar animado, dark mode opcional

### Google Sign-In

- SDK: `@react-native-google-signin/google-signin`
- Config: `webClientId` desde `@env`
- Inicializado en `App.tsx` con `offlineAccess: true`
- Flujo: SignInScreen maneja login Google, AvatarPickerScreen completa
  registro con nick de facultad

### react-native-dotenv

- UserApp usa `module:react-native-dotenv` para inyectar variables de
  entorno desde `.env` vía `@env`
- DriverApp no usa variables de entorno

## 8. Ciclo de Vida del Tracking

### Secuencia completa

```
Estado inicial: app cerrada, sin sesión
    ↓
1. Abrir DriverApp
    ↓
2. LoginDriverScreen: ingresa DNI + contraseña
    ↓
3. Firebase Auth valida, onAuthStateChanged → user detectado
    ↓
4. DriverApp renderiza SendCoordinates(driverDni)
    ↓
5. useEffect: loadAssignment() consulta /choferes/{dni} y /asignaciones
   con timeout de 10s por intento y hasta 2 reintentos acotados (C4.3)
    ↓
6. ¿Asignación activa encontrada?
    ├── Sí: guarda busId, habilita botón INICIAR
    ├── No: muestra "No tienes un bus asignado"
    └── Error (se agotan los reintentos): errorCard
        "No se pudo cargar tu asignación" con REINTENTAR y CERRAR SESIÓN
    ↓
7. Usuario presiona INICIAR RECORRIDO
    ↓
8. requestAllPermissions():
    ├── POST_NOTIFICATIONS (API 33+)
    ├── ACCESS_FINE_LOCATION
    └── ACCESS_BACKGROUND_LOCATION
    ↓
9. BackgroundJob.start(locationTask, {
       foregroundServiceType: ['location'],
       taskTitle: "El Bus {busId} está en ruta",
       ...
   })
    ↓
10. Sistema Android muestra notificación persistente
    ↓
11. locationTask ejecuta Geolocation.watchPosition()
    ↓
12. GPS callback o heartbeat (T4.1, cada 8 s) → writeLocation()
    → updateBusLocation(busId, { lat, lng, heading, speed, timestamp, isActive: true })
    ↓
13. Firebase RTDB recibe escritura en /ubicacion_buses/{busId}
    ↓
13a. Cada escritura (GPS o heartbeat) emite un pulso (PRO_LOCATION_PULSE)
    hacia la UI antes de escribir
    ↓
13b. Watchdog (C3): intervalo 10s, timeout 30s sin pulso
    ├── Pulso presente: no hace nada
    └── Sin pulso: restartBackgroundJob()
         ├── BackgroundJob.stop() (try/catch) + BackgroundJob.start()
         ├── Éxito: log "🔄 SERVICIO REINICIADO"
         └── Falla: isSending=false + Alert + banner persistente
    ↓
14. [Usuario presiona DETENER RECORRIDO]
    ↓
15. BackgroundJob.stop()
    ↓
16. stopBusService(busId) → /ubicacion_buses/{busId}/isActive = false
    ↓
17. (UI) vuelve a estado idle; el conductor puede reanudar el recorrido

Cerrar sesión (acción independiente, ADR-026):
18. auth().signOut() → onAuthStateChanged → renderiza LoginDriverScreen
```

### Notas sobre el ciclo de vida

- **Foreground Service**: utiliza `react-native-background-actions` que
  envuelve la API nativa `android.app.Service`. El servicio
  (`RNBackgroundActionsTask`) corre en el mismo proceso que la actividad.
- **Android 14**: requiere `foregroundServiceType="location"` en el
  manifest, `FOREGROUND_SERVICE_LOCATION` en permisos, y
  `foregroundServiceType: ['location']` en las opciones de JS.
  La ausencia de cualquiera de los tres provoca
  `InvalidForegroundServiceTypeException` en runtime.
- **Watchdog (C3)**: `isRunning()` de la librería no detecta la muerte
  nativa del servicio (flag JS que solo cambia con `stop()` explícito).
  Por eso el watchdog usa pulsos reales de GPS. `BackgroundJob.start()`
  es idempotente (nativo: `stopService` + `startForegroundService`), lo
  que hace seguro el ciclo stop/start del reinicio. Un FGS no puede
  iniciarse desde background (Android 14 lanza
  `ForegroundServiceStartNotAllowedException`); en ese escenario el
  reinicio falla y se avisa al conductor.
- **Cierre idempotente (C4.2)**: `stopProcess` protege el apagado del
  tracking ante doble pulsación, reinicio del watchdog o servicio ya
  detenido. `isStoppingRef` impide ejecuciones en paralelo; el cuerpo
  envuelve `BackgroundJob.stop()` en try/catch y verifica el resultado
  de `stopBusService`; el `finally` garantiza `setIsSending(false)` y
  libera el guard. `DETENER RECORRIDO` sigue siendo la única fuente de
  verdad para `isActive: false`.
- **Restauración del estado al re-montar (C4.7)**: al cerrar la app desde
  recientes con el tracking activo, el servicio nativo sigue vivo
  (`stopWithTask="false"`, mismo proceso), por lo que al re-montar
  `SendCoordinates`, `BackgroundJob.isRunning()` es `true` (el singleton JS
  sobrevivió). En el `useEffect` de montaje, si es `true` se restaura
  `isSending=true`, se reinicia `lastPulseRef`, se limpia `recoveryFailed` y se
  loguea "Recorrido restaurado", para que la UI muestre "Compartiendo
  ubicación" + DETENER RECORRIDO. `isActive` de Firebase sigue sin ser fuente
  de verdad: el conductor recupera el control del servicio real.
- **Doze Mode**: el foreground service evita que el SO suspenda el hilo
  de GPS cuando la pantalla se apaga. Sin este servicio,
  `watchPosition` se detiene al minimizar la app.
- **Persistencia desactivada**: `database().setPersistenceEnabled(true)`
  fue eliminado explícitamente. Si estuviera activo, Firebase acumularía
  ubicaciones durante pérdidas de señal y las reenviaría en ráfaga al
  reconectar, causando que el bus "viaje en el tiempo" en el mapa.
  Sin persistencia, cada envío fallido se descarta y el siguiente pulso
  (3s después) refleja la posición real.
- **Haversine / geofencing**: la función `calculateDistance()` existe
  en `Map.tsx` pero la arquitectura actual **no incorpora geofencing**.
  No hay cierre automático de vueltas ni escritura a `/recorridos`.

### Ramas de navegación (post-migración)

```
LoginDriverScreen
    ↓
onAuthStateChanged
    ├── user → SendCoordinates (tracking conductor)
    │         ├── Botón "INICIAR RECORRIDO" → BackgroundJob.start()
    │         ├── Botón "DETENER RECORRIDO" → BackgroundJob.stop() + stopBusService (isActive:false)
    │         └── Botón "Cerrar sesión" (independiente) → auth().signOut() → LoginDriverScreen
    └── null → LoginDriverScreen
```

El panel de administración **no** forma parte de las ramas de la DriverApp;
vive en AdminWeb (ver §3.1 y ADR-025).

## 9. Consideraciones de Implementación

### Android 14 (API 34)

El `foregroundServiceType` debe declararse en tres lugares para que
Android 14 acepte el servicio:

1. `AndroidManifest.xml`:
   `<service android:foregroundServiceType="location" />`
2. Permisos Android:
   `<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />`
3. Opciones JS en `getBackgroundOptions()`:
   `foregroundServiceType: ['location' as const]`

### Estado del tracking

- La arquitectura actual utiliza **múltiples buses como fuente** de datos.
  La UserApp escucha `/ubicacion_buses` y recibe un `Record<string, BurritoLocation>`
  con todas las placas activas. Los componentes de UI (`Map.tsx`, `MapBranding.tsx`)
  derivan el estado del primer bus activo como fallback hasta que se implemente
  el render multi-marcador completo (T4.4).

### Geofencing

- Actualmente la arquitectura no incorpora geofencing. Su incorporación
  está contemplada en futuras iteraciones del roadmap.

### Índices de RTDB

- DriverApp usa `orderByChild('choferId')` sobre `/asignaciones`. Esto
  requiere un índice definido en Firebase Rules:
  ```json
  {
    "asignaciones": {
      ".indexOn": "choferId"
    }
  }
  ```
  Sin este índice, Firebase rechazará la consulta en producción o la
  ejecutará ineficientemente en clientes.

### Compatibilidad iOS

- DriverApp no está probada ni soportada en iOS. El Foreground Service
  depende de APIs nativas de Android no disponibles en iOS.
- UserApp sí soporta iOS (configuración nativa con CocoaPods,
  GoogleService-Info.plist).

### Separación de productores

- El sistema actual recibe datos de tracking desde dos fuentes: la
  DriverApp física y un script Python de simulación. Ambos escriben a
  paths distintos. La UserApp consume de una sola fuente. La unificación
  de fuentes está planificada como parte de la evolución del sistema.

## 10. Referencias

| Documento | Relación |
|-----------|----------|
| `PROJECT_CONTEXT.md` | Visión general del ecosistema, propósito y estado. |
| `FIREBASE_SCHEMA.md` | Estructura detallada de nodos, payloads e índices de la RTDB. |
| `ROADMAP.md` | Tareas pendientes, fases y prioridades. |
| `TROUBLESHOOTING.md` | Incidentes, bugs históricos y soluciones. |
| `DECISIONS.md` | Decisiones de arquitectura (ADR) con razonamiento técnico. |
| Driver/README.md | Setup, requisitos y funcionalidad de DriverApp. |
| User/README.md | Setup, requisitos y funcionalidad de UserApp. |
| AdminWeb/README.md | Setup, requisitos y funcionalidad del panel de administración. |
| `BUGS_RESUELTOS/` | Historial de bugs resueltos durante el desarrollo. |
