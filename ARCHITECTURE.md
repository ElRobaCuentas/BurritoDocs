# Architecture — El Burrito

## 1. Visión General

El sistema sigue el patrón **productor-consumidor** sobre una arquitectura
**serverless** y **unidireccional**:

```
DriverApp (productor + operaciones) ──escribe──→ Firebase RTDB (bus) ──escucha──→ UserApp (consumidor)
```

No hay backend intermedio. Firebase Realtime Database actúa como único bus
de mensajes, sincronizando los datos entre ambas aplicaciones mediante
websockets nativos del SDK.

## 2. Arquitectura del Ecosistema

### Separación de responsabilidades (post-migración)

| Aplicación | Rol | Función principal | Autenticación |
|-----------|-----|-------------------|---------------|
| BurritoDriverApp | **Operador** (admin + conductor) | Capturar GPS, panel de gestión, enrutar según rol | DNI + contraseña (conductores), Auth UID (admin) |
| BurritoUserApp | **Consumidor** (estudiante) | Leer desde RTDB y renderizar mapa | Email/contraseña o Google Sign-In |
| Firebase RTDB | **Bus de datos** | Almacenar y sincronizar estado en vivo | — |
| Firebase Auth | **Identidad** | Autenticar conductores y estudiantes | — |

### Flujo de datos

```
┌──────────────────┐    write     ┌──────────────────┐
│   BurritoDriverApp │ ──────────→ │  Firebase RTDB   │
│   (Operador)       │             │                  │
│   ├─ Admin         │             │  /ubicacion_buses│
│   └─ Conductor     │             │  /asignaciones   │
└──────────────────┘             │  /choferes       │
                                  │  /buses          │
┌──────────────────┐             │  /usuarios       │
│   BurritoUserApp  │ ←────────── │  /comentarios    │
│   (Estudiante)    │    listen   │  /administradores│
└──────────────────┘             └──────────────────┘
```

### Autenticación

- **DriverApp (conductores)**: Firebase Auth con email compuesto como
  `${dni}@burritodriver.com`. Contraseña = DNI del conductor, establecida
  al crear el chofer desde el panel admin.
- **DriverApp (admin)**: misma cuenta Auth (email/contraseña). El enrutador
  post-login consulta `/administradores/{auth.uid}` para decidir si es
  administrador o conductor.
- **UserApp (estudiantes)**: Firebase Auth con email/contraseña (registro
  estándar) o Google Sign-In.

### Separación de proyectos

Ambas aplicaciones son proyectos React Native independientes dentro del
mismo repositorio. Comparten la misma base de datos Firebase y el mismo
proyecto `burritounmsm`, pero tienen su propio `package.json`, entry point
y configuración nativa.

## 3. Arquitectura de BurritoDriverApp

### Entry point y enrutador

```
index.js → src/DriverApp.tsx
```

`DriverApp.tsx` monta un listener de `onAuthStateChanged`. Cuando hay
sesión activa, resuelve el rol del usuario consultando `/administradores/{auth.uid}`:

```
Login exitoso
    ↓
onAuthStateChanged → user detectado
    ↓
existeAdministrador(user.uid)?
    ├── Sí → NavigationContainer + AdminNavigator (AdminPanelScreen)
    └── No → SendCoordinates(driverDni)  ← tracking conductor existente
```

El estado `initializing=true` cubre tanto la carga inicial de Auth como la
consulta a RTDB, eliminando flickers de navegación.

### Estructura de archivos

```
src/
├── DriverApp.tsx                    # Auth gate + enrutador tripartito
├── navigation/
│   └── AdminNavigator.tsx           # Stack navigator del panel admin
├── screen/
│   ├── LoginDriverScreen.tsx        # Login con DNI + contraseña
│   └── SendCoordinates.tsx          # Tracking, permisos, GPS
├── features/
│   └── admin/
│       ├── screen/
│       │   ├── AdminPanelScreen.tsx  # Menú principal del panel (+ botón Cerrar Sesión)
│       │   ├── ChoferesScreen.tsx    # CRUD de conductores
│       │   ├── BusesScreen.tsx       # CRUD de buses
│       │   └── AsignacionesScreen.tsx # CRUD de asignaciones diarias
│       └── services/
│           ├── admin_service.ts      # CRUD contra RTDB con Auth secundario
│           └── admin_check.ts        # existeAdministrador(uid): Promise<boolean>
├── services/
│   └── firebase_service.ts          # Escritura a RTDB (updateBusLocation)
└── shared/
    ├── config/firebase.ts           # firebaseDatabase + firebaseAuth
    └── theme/
        ├── colors.ts
        └── typography.ts
```

### LoginDriverScreen

- Input de DNI y contraseña.
- Construye email como `${dni}@burritodriver.com` y llama a
  `auth().signInWithEmailAndPassword`.
- Manejo de errores: red, credenciales inválidas, cuenta deshabilitada.

### SendCoordinates (pantalla principal del conductor)

Contiene la lógica completa de tracking (sin cambios respecto a la
arquitectura previa). Ver sección 8 para el ciclo detallado.

### Módulo Admin (nuevo en DriverApp)

Accesible solo para usuarios cuyo `auth.uid` existe en `/administradores/`.
CRUD completo contra RTDB:

| Entidad | Path | Operaciones |
|---------|------|-------------|
| Choferes | `/choferes/{dni}` | Crear (Auth + RTDB), listar, toggle activo |
| Buses | `/buses/{placa}` | Crear + init `/ubicacion_buses/{placa}`, listar, toggle |
| Asignaciones | `/asignaciones/{pushId}` | Crear con validación de exclusividad, listar, cancelar |

Al crear un chofer, `admin_service.ts` usa una **instancia secundaria de
Firebase Auth** (`SecondaryApp`) para no sobrescribir la sesión del admin
actual.

### admin_check.ts

Función única exportada:

```typescript
existeAdministrador(uid: string): Promise<boolean>
```

Consulta `/administradores/{uid}` con `once('value')` y retorna
`snapshot.exists()`. Es la fuente de verdad tanto para el enrutador de
DriverApp como para las reglas de RTDB.

### AdminNavigator

Stack navigator simple que registra las 4 pantallas del panel:

```typescript
RootStackParams = {
  AdminPanelScreen: undefined;
  ChoferesScreen: undefined;
  BusesScreen: undefined;
  AsignacionesScreen: undefined;
};
```

Sin sobreingeniería: un solo stack para la rama admin.

### Firebase Auth

- `auth().onAuthStateChanged()` en DriverApp.tsx maneja la sesión global.
- El enrutador post-login deriva el destino de la sesión según la membresía
  en `/administradores/{auth.uid}`.
- `auth().signOut()` disponible desde AdminPanelScreen (botón "Cerrar
  Sesión") y desde SendCoordinates (al detener tracking).

## 4. Arquitectura de BurritoUserApp

### Entry point

```
index.js → src/app/App.tsx
```

`App.tsx` orquesta: gestor de navegación, stores de Zustand, Firebase
offline/online según estado de la app, Google Sign-In config, splash
animado y gating de hidratación.

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
│   ├── screen/AnimatedSplash.tsx      # Splash animado
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
│       │   ├── map_route.ts           # GeoJSON de ruta y paraderos
│       │   └── mapTheme.ts            # Estilos oscuro/claro para Mapbox
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
| `mapStore` | Efímera | isFollowing, command (center/follow) |
| `burritoLocationStore` | Efímera | locations (Record<string,BurritoLocation>), busMovementStates, isConnecting, startTracking/stopTracking |
| `drawerStore` | Efímera | isOpen, open/close |

### Módulo Admin (eliminado)

El módulo `features/admin/` fue eliminado de la UserApp. El panel de
gestión ahora reside exclusivamente en la DriverApp. El botón "Panel de
Gestión" ya no existe en el CustomDrawer. El campo `rol` en `userStore`
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
Geolocation.watchPosition()  ← cada ~3s
    ↓
sendLog()                    ← solo debug
    ↓
updateBusLocation(busId, {   ← firebase_service.ts
    latitude, longitude,
    heading, speed,
    timestamp, isActive: true
})
    ↓
Firebase RTDB                ← /ubicacion_buses/{busId}
    ↓
UserApp recibe el delta vía subscribeToBusLocations()
```

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
Callback recibe Record<string, BurritoLocation> { placa1: {lat, lng, ...}, placa2: {...}, ... }
    ↓
Filtro dedup por placa (timestamp > actual por cada bus)
    ↓
Set en burritoLocationStore.locations
    ↓
Map.tsx deriva burritoLocation del primer bus activo (fallback)
    ↓
RNAnimated interpola posición del marcador (~2s)
    ↓
Mapbox actualiza canvas
```

### Flujo de autenticación (DriverApp — post-migración)

```
LoginDriverScreen.handleLogin()
    ↓
auth().signInWithEmailAndPassword(
    `${dni}@burritodriver.com`, password
)
    ↓
Firebase Auth valida credenciales
    ↓
onAuthStateChanged en DriverApp.tsx detecta user
    ↓
existeAdministrador(user.uid)?
    ├── Sí: → NavigationContainer + AdminPanelScreen
    └── No: → SendCoordinates(driverDni = email.split('@')[0])
              ↓
              SendCoordinates monta useEffect
              ↓
              database().ref('/asignaciones').orderByChild('choferId').equalTo(driverDni).once('value')
              ↓
              Filtra por fecha===today && activo===true
              ↓
              Guarda busId en estado local
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

### Flujo admin (ahora en DriverApp)

```
Admin crea chofer:
    ChoferesScreen → createChofer({ dni, nombre, apellidos })
        → secondaryAuth.createUserWithEmailAndPassword(`${dni}@burritodriver.com`, dni)
        → /choferes/{dni}.set({ nombre, apellidos, activo: true })

Admin crea bus:
    BusesScreen → createBus({ placa, modelo, marca, anio })
        → /buses/{placa}.set({ modelo, marca, anio, activo: true })
        → /ubicacion_buses/{placa}.set({ isActive: false })

Admin asigna:
    AsignacionesScreen → createAsignacion(choferId, busId)
        → Valida que chofer y bus no tengan asignación activa hoy
        → /asignaciones.push().set({ choferId, busId, fecha, activo: true })
```

## 6. Componentes Principales

### BurritoDriverApp

| Archivo | Responsabilidad |
|---------|----------------|
| `index.js` | Registro de la app, comentario sobre persistence desactivada, import gesture-handler |
| `src/DriverApp.tsx` | Auth gate, enrutador tripartito (admin / conductor / no sesión) |
| `src/navigation/AdminNavigator.tsx` | Stack navigator de 4 pantallas admin |
| `src/screen/LoginDriverScreen.tsx` | Login con DNI, manejo de errores de autenticación |
| `src/screen/SendCoordinates.tsx` | Asignación de bus, permisos, foreground service, GPS, debug console |
| `src/features/admin/services/admin_service.ts` | CRUD choferes (Auth+RTDB), buses, asignaciones |
| `src/features/admin/services/admin_check.ts` | `existeAdministrador(uid)` consulta puntual |
| `src/features/admin/screen/AdminPanelScreen.tsx` | Menú principal admin + botón Cerrar Sesión |
| `src/features/admin/screen/ChoferesScreen.tsx` | Listar, crear, toggle choferes |
| `src/features/admin/screen/BusesScreen.tsx` | Listar, crear, toggle buses |
| `src/features/admin/screen/AsignacionesScreen.tsx` | Listar, crear, cancelar asignaciones |
| `src/services/firebase_service.ts` | `updateBusLocation()` y `stopBusService()` contra RTDB |

### BurritoUserApp

| Archivo | Responsabilidad |
|---------|----------------|
| `src/app/App.tsx` | Montaje global, splash, hidratación, Firebase offline/online |
| `src/app/navigations/StackNavigator.tsx` | Auth gating declarativo (sin admin) |
| `src/app/navigations/DrawerNavigator.tsx` | Menú lateral con MapScreen |
| `src/features/app/screen/AnimatedSplash.tsx` | Splash animado (Lottie) |
| `src/features/map/screen/MapScreen.tsx` | Orquestador del mapa, inicializa tracking |
| `src/features/map/components/Map.tsx` | Mapbox canvas, marcadores, ruta, radar, follow |
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
| Firebase Auth | Login admin y conductor, creación de cuentas de conductor (SecondaryApp) | Login email/password + Google Sign-In |
| Firebase RTDB | `/ubicacion_buses/{busId}`, `/asignaciones`, `/administradores/`, `/choferes/`, `/buses/` | `/ubicacion_buses`, `/usuarios`, `/comentarios` |
| Firebase Analytics | No | Eventos de usuario (mapa abierto, bus seguido, mapa centrado) |
| Firebase Crashlytics | No | Configurado en firebase.json |
| ServerValue.TIMESTAMP | No | Sí (6 ocurrencias: ultimaConexion, feedback) |

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
4. DriverApp ejecuta existeAdministrador(user.uid)
    ├── Sí: renderiza AdminNavigator (fin del flujo para admin)
    └── No: renderiza SendCoordinates(driverDni)
         ↓
5. useEffect: consulta /asignaciones filtrando por DNI
    ↓
6. ¿Asignación activa encontrada?
    ├── Sí: guarda busId, habilita botón INICIAR
    └── No: muestra "No tienes un bus asignado"
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
12. Cada ~3s: updateBusLocation(busId, { lat, lng, heading, speed, timestamp, isActive: true })
    ↓
13. Firebase RTDB recibe escritura en /ubicacion_buses/{busId}
    ↓
14. [Usuario presiona DETENER TODO]
    ↓
15. BackgroundJob.stop()
    ↓
16. stopBusService(busId) → /ubicacion_buses/{busId}/isActive = false
    ↓
17. auth().signOut()
    ↓
18. onAuthStateChanged → user null → renderiza LoginDriverScreen
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
    ↓
¿user?.uid existe en /administradores/{uid}?
    ├── Sí → AdminNavigator
    │         ├── AdminPanelScreen
    │         ├── ChoferesScreen
    │         ├── BusesScreen
    │         └── AsignacionesScreen
    │         └── Botón "Cerrar Sesión" → auth().signOut() → LoginDriverScreen
    │
    └── No → SendCoordinates (tracking conductor)
              ├── Botón "INICIAR" → BackgroundJob.start()
              ├── Botón "DETENER TODO" → BackgroundJob.stop() + auth().signOut()
              └── → LoginDriverScreen
```

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
| BurritoDriverApp/README.md | Setup, requisitos y funcionalidad de DriverApp. |
| BurritoUserApp/README.md | Setup, requisitos y funcionalidad de UserApp. |
| `BUGS_RESUELTOS/` | Historial de bugs resueltos durante el desarrollo. |
| `ReviewNotes.md` | Notas de revisión futura sobre la arquitectura. |
