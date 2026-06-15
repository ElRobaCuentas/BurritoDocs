# Architecture — El Burrito

## 1. Visión General

El sistema sigue el patrón **productor-consumidor** sobre una arquitectura
**serverless** y **unidireccional**:

```
DriverApp (productor) ──escribe──→ Firebase RTDB (bus) ──escucha──→ UserApp (consumidor)
```

No hay backend intermedio. Firebase Realtime Database actúa como único bus
de mensajes, sincronizando los datos entre ambas aplicaciones mediante
websockets nativos del SDK.

## 2. Arquitectura del Ecosistema

### Roles

| Aplicación | Rol | Función principal |
|-----------|-----|-------------------|
| BurritoDriverApp | **Productor** | Capturar GPS y escribir a RTDB |
| BurritoUserApp | **Consumidor** | Leer desde RTDB y renderizar en mapa |
| Firebase RTDB | **Bus de datos** | Almacenar y sincronizar estado en vivo |
| Firebase Auth | **Identidad** | Autenticar conductores y estudiantes |

### Flujo de datos

```
┌──────────────────┐    write     ┌──────────────────┐
│   BurritoDriverApp │ ──────────→ │  Firebase RTDB   │
│   (Android)        │             │                  │
└──────────────────┘             │  /ubicacion_buses │
                                  │  /asignaciones    │
┌──────────────────┐             │  /choferes        │
│   BurritoUserApp  │ ←────────── │  /buses           │
│   (Android/iOS)   │    listen   │  /usuarios        │
└──────────────────┘             │  /comentarios     │
                                  │  /ubicacion_burrito │
      
      /ubicacion_burrito (LO VAMOS A QUITAR MAS ADELANTE)
      
                                  └──────────────────┘
```



### Autenticación

- **DriverApp**: Firebase Auth con email compuesto como `${dni}@burritodriver.com`.
  Contraseña = DNI del conductor, establecida al crear el chofer desde el panel admin.
- **UserApp**: Firebase Auth con email/contraseña (registro estándar) o Google Sign-In.
  Detecta y redirige automáticamente según sesión activa.

### Separación de proyectos

Ambas aplicaciones son proyectos React Native independientes dentro del
mismo repositorio. Comparten la misma base de datos Firebase y el mismo
proyecto `burritounmsm`, pero tienen su propio `package.json`, entry point
y configuración nativa.

## 3. Arquitectura de BurritoDriverApp

### Entry point

```
index.js → src/DriverApp.tsx
```

`DriverApp.tsx` monta un listener de `onAuthStateChanged`. Cuando hay
sesión activa renderiza `SendCoordinates`, caso contrario `LoginDriverScreen`.
No usa React Navigation.

### Estructura de archivos

```
src/
├── DriverApp.tsx              # Entrypoint, auth gate
├── screen/
│   ├── LoginDriverScreen.tsx  # Login con DNI + contraseña
│   └── SendCoordinates.tsx    # Tracking, permisos, GPS
└── services/
    └── firebase_service.ts    # Escritura a RTDB
```

### LoginDriverScreen

- Input de DNI y contraseña.
- Construye email como `${dni}@burritodriver.com` y llama a
  `auth().signInWithEmailAndPassword`.
- Manejo de errores: red, credenciales inválidas, cuenta deshabilitada.

### SendCoordinates (pantalla principal)

Contiene la lógica completa de tracking:

1. **Al montar**: consulta `/asignaciones` con filtro `orderByChild('choferId').equalTo(driverDni)`
   y valida client-side `fecha === today && activo === true`. Si encuentra asignación,
   guarda `busId` en estado local.

2. **Al presionar INICIAR**: solicita permisos Android (POST_NOTIFICATIONS,
   ACCESS_FINE_LOCATION, ACCESS_BACKGROUND_LOCATION), luego inicia el
   Foreground Service mediante `BackgroundJob.start()`.

3. **Foreground Service**: ejecuta `locationTask`, que conecta
   `Geolocation.watchPosition()` con callback cada ~3 segundos. En cada
   pulso escribe a `/ubicacion_buses/{busId}` vía `updateBusLocation()`.

4. **Al presionar DETENER**: detiene `BackgroundJob.stop()`, escribe
   `{ isActive: false }` a `/ubicacion_buses/{busId}`, y cierra sesión
   de Firebase Auth.

Payload escrito a RTDB en cada pulso:
```
/ubicacion_buses/{busId} {
  latitude,    // float
  longitude,   // float
  heading,     // float
  speed,       // float
  timestamp,   // Date.now()
  isActive: true
}
```

### Sistema de debug

SendCoordinates incluye una consola visual de eventos mediante
`DeviceEventEmitter` + `sendLog()`. Todos los eventos del ciclo de
vida del tracking se emiten con timestamp y tipo (info, error, success)
y se muestran en un ScrollView dentro de la misma pantalla. No tiene
efecto en la lógica de negocio.

### Firebase Auth

- `auth().onAuthStateChanged()` en DriverApp.tsx maneja la sesión global.
- `auth().signOut()` al detener tracking.

## 4. Arquitectura de BurritoUserApp

### Entry point

```
index.js → src/app/App.tsx
```

`App.tsx` orquesta: gestor de navegación, stores de Zustand, Firebase
offline/online según estado de la app, Google Sign-In config, splash
animado y gating de hidratación.

### Navegación

```
StackNavigator (root)
├── logged-out: Welcome → SignIn/SignUp → ForgotPassword → AvatarPicker
└── logged-in:
    ├── DrawerNavigator
    │   └── MapScreen (única pantalla en el drawer)
    ├── AdminPanelScreen (stack, role-gated)
    ├── ChoferesScreen
    ├── BusesScreen
    └── AsignacionesScreen
```

El gating de autenticación es declarativo: `StackNavigator.tsx` renderiza
distintos grupos de pantallas según `isLoggedIn` del store. No hay route
guards manuales.

### Estructura de archivos

```
src/
├── app/
│   ├── App.tsx                        # Entry point, Firebase offline/online
│   ├── screen/AnimatedSplash.tsx      # Splash animado
│   └── navigations/
│       ├── StackNavigator.tsx         # Root navigator con auth gate
│       └── DrawerNavigator.tsx        # Menú lateral con MapScreen
├── features/
│   ├── auth/screen/                   # Welcome, SignIn, SignUp, ForgotPassword, AvatarPicker
│   ├── map/
│   │   ├── screen/MapScreen.tsx       # Contenedor del mapa + overlay UI
│   │   ├── components/
│   │   │   ├── Map.tsx                # Mapbox canvas con marcadores, ruta y paraderos
│   │   │   ├── CustomDrawer.tsx       # Menú lateral animado
│   │   │   ├── FAB.tsx                # Botón flotante
│   │   │   ├── StopCard.tsx           # Modal de información de paradero
│   │   │   └── MapBranding.tsx        # Marca de agua del mapa
│   │   ├── constants/
│   │   │   ├── map_route.ts           # GeoJSON de ruta y paraderos
│   │   │   └── mapTheme.ts            # Estilos oscuro/claro para Mapbox
│   │   ├── services/map_service.ts    # Suscripción RTDB + feedback
│   │   └── types.ts
│   └── admin/
│       ├── screen/                    # AdminPanel, Choferes, Buses, Asignaciones
│       └── services/admin_service.ts  # CRUD contra RTDB con Auth secundario
├── shared/
│   ├── config/firebase.ts             # Instancia compartida de RTDB y Auth
│   └── theme/
│       ├── colors.ts
│       └── typography.ts
└── store/
    ├── userStore.ts                   # Persistida (AsyncStorage), maneja sesión local
    ├── themeStore.ts                  # Persistencia manual (AsyncStorage)
    ├── mapStore.ts                    # Estado efímero del mapa
    ├── burritoLocationStore.ts        # Estado efímero de ubicación del bus
    └── drawerStore.ts                 # Estado efímero del menú lateral
```

### Stores de Zustand

| Store | Persistencia | Propósito |
|-------|-------------|-----------|
| `userStore` | Zustand persist + AsyncStorage | uuid, username, avatar, rol, isLoggedIn |
| `themeStore` | AsyncStorage manual | isDarkMode |
| `mapStore` | Efímera | isFollowing, command (center/follow) |
| `burritoLocationStore` | Efímera | location, isConnecting, busMovementStatus, startTracking/stopTracking |
| `drawerStore` | Efímera | isOpen, open/close |

### burritoLocationStore (tracking)

Inicia/para el listener de RTDB. Recibe actualizaciones vía
`MapService.subscribeToBusLocation()` sobre el path `/ubicacion_burrito`.

Aplica un filtro de deduplicación: rechaza timestamps no más nuevos que
el actual. Clasifica el estado del bus según antigüedad del timestamp:

| Condición | Estado |
|-----------|--------|
| `isActive === false` | `offline` |
| timestamp age < 12s | `moving` |
| timestamp age >= 12s | `stopped` |

Un intervalo de 2s refresca periódicamente la clasificación.

### Map.tsx (renderizado)

Componente principal del mapa. Integra `@rnmapbox/maps` con:

- Cámara con centro fijo en UNMSM y bounds que restringen el desplazamiento.
- Capa de ruta (LineString GeoJSON) con 60+ pares de coordenadas.
- Capa de paraderos (10 puntos con nombres).
- Marcador del bus (punto animado con heading).
- Círculo de radar animado alrededor del bus.
- Lógica de interpolación: anima la transición entre posiciones GPS
  usando `RNAnimated.Value` con duración de ~2s.
- Botón de seguimiento (follow) y centrado.
- Función Haversine (`calculateDistance`) disponible pero **no activa**
  en el flujo de tracking (originalmente diseñada para `snapToRoute`,
  que está comentado por causar saltos con GPS impreciso).

### Módulo Admin

Accesible desde el CustomDrawer cuando `rol === 'admin'`.
CRUD completo contra RTDB:

| Entidad | Path | Operaciones |
|---------|------|-------------|
| Choferes | `/choferes/{dni}` | Crear (Auth + RTDB), listar, toggle activo |
| Buses | `/buses/{placa}` | Crear + init `/ubicacion_buses/{placa}`, listar, toggle |
| Asignaciones | `/asignaciones/{pushId}` | Crear con validación de exclusividad, listar, cancelar |

Al crear un chofer, `admin_service.ts` usa una **instancia secundaria de
Firebase Auth** para no sobrescribir la sesión del admin actual.

### Dark mode

`themeStore` persiste el estado en AsyncStorage. `CustomDrawerContent`
tiene un toggle manual. En cada montaje del drawer, el estado del sistema
(`systemColorScheme`) se usa como valor por defecto, pero el toggle manual
prevalece durante la sesión.

### Hydration gating

`App.tsx` espera que `userStore._hasHydrated` y `themeStore._hasHydrated`
sean `true` antes de renderizar `NavigationContainer`. Esto evita que
la navegación se monte con estado vacío antes de que AsyncStorage restaure
la sesión y el tema.

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
UserApp NO recibe esto todavía
    (UserApp escucha /ubicacion_burrito, no /ubicacion_buses)
```

### Flujo de visualización (UserApp)

```
UserApp inicia
    ↓
MapScreen.mount()
    ↓
burritoLocationStore.startTracking()
    ↓
MapService.subscribeToBusLocation()  ← se suscribe a /ubicacion_burrito
    ↓
ref.on('value', callback)            ← Firebase empuja deltas
    ↓
Callback recibe {latitude, longitude, heading, isActive, timestamp}
    ↓
Filtro dedup (timestamp > actual)
    ↓
Set en burritoLocationStore.location
    ↓
Map.tsx re-lectura selectiva del store
    ↓
RNAnimated interpola posición del marcador (~2s)
    ↓
Mapbox actualiza canvas
```

### Flujo de autenticación (DriverApp)

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
Renderiza SendCoordinates con driverDni = email.split('@')[0]
    ↓
SendCoordinates monta useEffect
    ↓
database().ref('/asignaciones').orderByChild('choferId').equalTo(driverDni).once('value')
    ↓
Filtra resultado por fecha===today && activo===true
    ↓
Guarda busId en estado local
```

### Flujo de autenticación (UserApp)

```
SignInScreen.handleLogin()
    ↓
auth().signInWithEmailPassword(email, password)
    ↓
Lee /usuarios/{uid}.once('value')  → obtiene nombre, avatar, rol
    ↓
Actualiza /usuarios/{uid}/ultimaConexion = ServerValue.TIMESTAMP
    ↓
userStore.login(uuid, username, avatar, email, rol)
    ↓
StackNavigator detecta isLoggedIn=true → renderiza DrawerNavigator
```

### Flujo admin

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
| `index.js` | Registro de la app, comentario sobre persistence desactivada |
| `src/DriverApp.tsx` | Auth gate, renderiza LoginDriverScreen o SendCoordinates |
| `src/screen/LoginDriverScreen.tsx` | Login con DNI, manejo de errores de autenticación |
| `src/screen/SendCoordinates.tsx` | Asignación de bus, permisos, foreground service, GPS, debug console |
| `src/services/firebase_service.ts` | `updateBusLocation()` y `stopBusService()` contra RTDB |

### BurritoUserApp

| Archivo | Responsabilidad |
|---------|----------------|
| `src/app/App.tsx` | Montaje global, splash, hidratación, Firebase offline/online |
| `src/app/navigations/StackNavigator.tsx` | Auth gating declarativo |
| `src/app/navigations/DrawerNavigator.tsx` | Menú lateral con MapScreen |
| `src/features/app/screen/AnimatedSplash.tsx` | Splash animado (Lottie) |
| `src/features/map/screen/MapScreen.tsx` | Orquestador del mapa, inicializa tracking |
| `src/features/map/components/Map.tsx` | Mapbox canvas, marcadores, ruta, radar, follow |
| `src/features/map/components/CustomDrawer.tsx` | Drawer animado con perfil, tema, feedback, admin |
| `src/features/map/services/map_service.ts` | Listener RTDB a `/ubicacion_burrito`, envío de feedback |
| `src/features/admin/services/admin_service.ts` | CRUD choferes, buses, asignaciones |
| `src/store/burritoLocationStore.ts` | Estado del tracking, dedup, clasificación moving/stopped/offline |
| `src/store/userStore.ts` | Sesión persistente del usuario |
| `src/store/themeStore.ts` | Dark mode con persistencia AsyncStorage |
| `src/shared/config/firebase.ts` | Instancia compartida de Firebase |

## 7. Servicios Externos

### Firebase

| Servicio | Uso en DriverApp | Uso en UserApp |
|----------|-----------------|----------------|
| Firebase Auth | Login con email/password | Login email/password + Google Sign-In |
| Firebase RTDB | `/ubicacion_buses/{busId}`, `/asignaciones` | `/ubicacion_burrito`, `/choferes`, `/buses`, `/asignaciones`, `/usuarios`, `/comentarios` |
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
4. DriverApp renderiza SendCoordinates(driverDni)
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

- La arquitectura actual utiliza un **único bus como fuente** de datos.
  El path de tracking que la UserApp escucha actualmente corresponde a la
  fuente de datos vigente. La evolución hacia un listener multi-bus está
  planificada y será documentada cuando forme parte de la arquitectura
  oficial.

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
