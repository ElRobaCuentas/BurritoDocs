# Decisiones de Arquitectura — El Burrito

## 1. Propósito

Este documento registra las decisiones de arquitectura (ADR) relevantes
del ecosistema. Cada entrada explica el problema, las alternativas
consideradas y el razonamiento detrás de la solución adoptada.

No es un manual técnico ni un changelog. Las decisiones aquí documentadas
responden a la pregunta: **¿por qué está hecho así?**

## 2. Convenciones del Documento

Cada ADR sigue esta estructura:

- **Estado**: Aceptada, Planeada o Reemplazada.
- **Contexto**: situación que originó la decisión.
- **Decisión**: qué se decidió y cómo se implementó.
- **Alternativas consideradas**: opciones que fueron evaluadas y
  descartadas.
- **Consecuencias**: impacto positivo y negativo de la decisión.
- **Referencias**: documentos relacionados.

## ADR Aceptadas

### ADR-003: Desactivación de persistencia local en Firebase

**Estado:** Aceptada.

**Contexto:**

En pruebas iniciales, cuando la UserApp experimentaba micro-cortes de
red en el campus, el mapa renderizaba "saltos" anómalos o el bus
aparecía en posiciones desactualizadas al recuperar la conexión.

**Decisión:**

Desactivar explícitamente la persistencia en disco de Firebase
(`setPersistenceEnabled(false)`). El código actual de DriverApp
`index.js` contiene la explicación como comentario.

**Alternativas consideradas:**

- Mantener persistencia con filtrado por marcas de tiempo en el cliente:
  descartado por sobreingeniería. Requería lógica de limpieza en el
  store de Zustand para descartar deltas con latencia acumulada.

**Consecuencias:**

- La ubicación en tiempo real es un dato efímero. Si el dispositivo
  pierde conexión 30 segundos, al recuperar la señal no tiene valor
  operativo procesar coordenadas históricas.
- La app no retiene la última posición conocida en modo offline. Es
  aceptable dado el propósito del sistema.

**Referencias:** ARCHITECTURE.md (sección 8), index.js de DriverApp.

---

### ADR-004: Arquitectura serverless directa (sin backend intermedio)

**Estado:** Aceptada.

**Contexto:**

Definir el canal de comunicación en tiempo real entre DriverApp y
UserApp minimizando latencia (<500ms) y sobrecarga de infraestructura.

**Decisión:**

Conexión directa desde los clientes React Native hacia Firebase
Realtime Database mediante websockets nativos del SDK, sin capas
intermedias.

**Alternativas consideradas:**

- Backend Node.js/Express + Socket.io: descartado por requerir
  despliegue, mantenimiento y escalabilidad, aumentando los puntos de
  fallo.
- Protocolos IoT directos (MQTT/TCP raw): descartado por complejidad
  en React Native.

**Consecuencias:**

- Firebase RTDB resuelve la sincronización mediante sockets sin
  infraestructura intermedia.
- Acoplamiento al ecosistema Firebase. Cambiar de base de datos en el
  futuro requeriría reescribir la capa de comunicación.

**Referencias:** ARCHITECTURE.md (sección 1 y 2), ROADMAP.md (Fase 5).

---

### ADR-005: Manejo de estado con Zustand en lugar de Context o Redux

**Estado:** Aceptada.

**Contexto:**

Los latidos de ubicación desde Firebase RTDB ocurren cada pocos
segundos. Inyectar estos deltas directamente en el árbol de
componentes de React provocaba caídas de rendimiento y retrasos en
Mapbox.

**Decisión:**

Adoptar **Zustand** para aislar los listeners de red del flujo de
renderizado de la UI.

**Alternativas consideradas:**

- React Context API: descartado porque Context no tiene selectores
  atómicos. Cualquier cambio forzaba el re-renderizado de todo el
  Provider.
- Redux Toolkit: descartado por excesivo boilerplate para un stack
  enfocado en simplicidad.

**Consecuencias:**

- El listener de Firebase muta el store fuera del flujo de React.
- `Map.tsx` usa selectores estrictos para escuchar únicamente los
  cambios de coordenadas.
- Mapbox recibe el dato refinado e interpola la posición en su canvas
  nativo, manteniendo 60 FPS estables.
- Dependencia externa de gestión de estado, justificada por el control
  de rendimiento que ofrece.

**Referencias:** ARCHITECTURE.md (sección 4, stores de Zustand).

---

### ADR-006: Separación del ecosistema en dos aplicaciones independientes

**Estado:** Aceptada.

**Contexto:**

El sistema requiere dos perfiles de usuario completamente distintos:
conductores que emiten GPS y estudiantes que consumen ubicaciones.
Cada perfil tiene requisitos de plataforma, autenticación y ciclo de
vida diferentes.

**Decisión:**

Crear dos proyectos React Native independientes (BurritoDriverApp y
BurritoUserApp) dentro del mismo repositorio. Comparten la misma base
de datos Firebase pero tienen su propio `package.json`, entry point y
configuración nativa.

**Alternativas consideradas:**

- Una sola app con roles y pantallas condicionales: descartado porque
  la DriverApp necesita foreground services nativos de Android y un
  ciclo de vida de tracking que la UserApp no debe tener. Además, una
  sola app obligaría a los conductores a tener una interfaz de
  estudiante que no necesitan.

**Consecuencias:**

- Cada app puede evolucionar de forma independiente.
- La DriverApp puede enfocarse en Android sin arrastrar dependencias
  iOS de la UserApp.
- El código compartido (como la referencia a Firebase) debe mantenerse
  sincronizado manualmente entre proyectos.

**Referencias:** PROJECT_CONTEXT.md (sección 2), ARCHITECTURE.md
(sección 2).

---

### ADR-007: Comunicación desacoplada mediante Firebase Realtime Database

**Estado:** Aceptada.

**Contexto:**

DriverApp y UserApp necesitan intercambiar datos de ubicación en
tiempo real. Se requería un mecanismo que no dependiera de
comunicación directa entre dispositivos ni de un backend propio.

**Decisión:**

Toda la comunicación entre aplicaciones ocurre exclusivamente a través
de Firebase Realtime Database. La DriverApp escribe; la UserApp lee.
No existe comunicación directa entre dispositivos ni llamadas HTTP
entre apps.

**Alternativas consideradas:**

- Sockets directos (TCP/UDP) entre dispositivos: descartado por
  requerir direccionamiento IP, NAT traversal y gestionar
  desconexiones en redes móviles.
- Backend intermediario con API REST + WebSockets: descartado en
  ADR-004 por la sobrecarga de infraestructura.

**Consecuencias:**

- La base de datos actúa como único bus de mensajes. No hay
  coordinación entre pares.
- Si una app falla, la otra no se ve afectada.
- Toda la lógica de comunicación se reduce a operaciones de
  lectura/escritura sobre RTDB.
- La base de datos se convierte en un punto único. Si Firebase está
  caído, el sistema completo deja de funcionar.

**Referencias:** ARCHITECTURE.md (sección 2 y 5).

---

### ADR-008: Foreground Service como requisito obligatorio de plataforma

**Estado:** Aceptada.

**Contexto:**

El `watchPosition` de React Native se suspende cuando la app pasa a
segundo plano por las políticas de ahorro de energía de Android (Doze
Mode). La DriverApp necesita transmitir coordenadas incluso con la
pantalla apagada.

**Decisión:**

Implementar un Foreground Service nativo declarado en
`AndroidManifest.xml` con tipo `location`, vinculado a una notificación
persistente. Para Android 14, debe declararse en tres lugares:
manifest, permisos y opciones de JS.

**Alternativas consideradas:**

- No hacer nada y aceptar que Android detenga el GPS: descartado. El
  propósito del sistema es tracking continuo.
- Usar `setInterval` en segundo plano: descartado. React Native no
  garantiza ejecución en segundo plano sin servicio nativo.

**Consecuencias:**

- La transmisión se mantiene activa incluso con la app minimizada o la
  pantalla apagada.
- Dependencia de APIs nativas de Android. La DriverApp no puede
  ejecutarse en iOS con la misma funcionalidad.
- Android 14 exige `foregroundServiceType: "location"`. Sin los tres
  lugares sincronizados, se produce
  `InvalidForegroundServiceTypeException`.

**Referencias:** ARCHITECTURE.md (sección 8), TROUBLESHOOTING.md
(sección 6).

---

### ADR-009: Autenticación segmentada por roles

**Estado:** Aceptada. Parcialmente reemplazada por ADR-017.

**Contexto:**

El sistema tiene tres perfiles: conductores (DriverApp), estudiantes
(UserApp) y administradores (UserApp con rol admin). Cada perfil
necesita un flujo de autenticación y un alcance de datos distinto.

**Decisión:**

- **Conductores**: login con email compuesto como
  `${dni}@burritodriver.com` sobre Firebase Auth. La contraseña es el
  DNI, establecida al crear el chofer desde el panel admin. Sin
  registro público.
- **Estudiantes**: registro con email/contraseña o Google Sign-In.
  Sesión persistente con AsyncStorage + Zustand persist.
- **Administradores**: mismo flujo que estudiantes pero con
  `rol: "admin"` en RTDB. El enlace al panel de gestión se muestra
  condicionalmente en el menú lateral.

**Alternativas consideradas:**

- Usar Firebase Auth UID como identificador en asignaciones: descartado
  porque el UID se genera automáticamente y no es conocido por el
  admin al crear la asignación. Se optó por usar el DNI como
  identificador transversal (choferId en asignaciones, key en
  `/choferes`, parte del email de Auth).

**Consecuencias:**

- El DNI es el identificador único a través de todo el flujo:
  autenticación, asignación y tracking.
- El panel admin usa una instancia secundaria de Firebase Auth para
  crear cuentas de conductores sin cerrar la sesión del admin (ver
  ADR-011).
- Los roles se almacenan en RTDB y el gating se implementa en dos
  niveles: gating visual (menú admin condicional en CustomDrawer) y
  gating de rutas (pantallas admin envueltas en `rol === 'admin'`
  en StackNavigator).
- **Nota**: este modelo de autorización por `/usuarios/{uid}/rol` fue
  reemplazado por el modelo de membresía en `/administradores/{auth.uid}`
  (ver ADR-017). El campo `rol` en `/usuarios` ahora solo almacena
  `'estudiante'` y ya no participa en autorización.

**Referencias:** ARCHITECTURE.md (sección 3 y 4), FIREBASE_SCHEMA.md
(sección 5 y 9).

---

### ADR-010: Gestión manual del ciclo de vida del tracking

**Estado:** Aceptada.

**Contexto:**

Cuando la DriverApp se cierra abruptamente (el conductor cierra la app
desde el administrador de tareas o el teléfono se apaga), Firebase no
recibe `isActive: false`. El bus aparece como "fantasma" en el mapa
de la UserApp.

**Decisión:**

El ciclo de vida del tracking se gestiona mediante acciones manuales
explícitas. El conductor debe presionar "INICIAR RECORRIDO" para
empezar y "DETENER TODO" para finalizar. Al detener, se escribe
`isActive: false` en RTDB y se cierra sesión de Firebase Auth.

**Alternativas consideradas:**

- `onDisconnect` hooks de Firebase: evaluado pero postergado. En redes
  móviles inestables, Firebase puede tardar varios minutos en detectar
  la desconexión, dejando el bus como activo durante ese período.

**Consecuencias:**

- La única fuente confiable de fin de turno es la pulsación del botón
  "DETENER TODO".
- Si el conductor cierra la app abruptamente, el bus queda como activo
  en RTDB. La UserApp mitiga esto clasificando ubicaciones como
  "offline" cuando el timestamp expira (30s, C4.6) o "stopped" cuando
  no hay desplazamiento entre publicaciones (burritoLocationStore).
- Timeout check automático está planificado en Fase 4 del roadmap.

**Referencias:** ARCHITECTURE.md (sección 8), ROADMAP.md (Fase 4).

---

### ADR-011: Instancia secundaria de Firebase Auth para CRUD admin

**Estado:** Aceptada.

**Contexto:**

Al crear un chofer desde el panel administrativo, el admin debe crear
una cuenta de Firebase Auth para el conductor (email
`${dni}@burritodriver.com`). Si se usa la misma instancia de Auth,
el admin se desconecta y se autentica como el nuevo conductor.

**Decisión:**

Usar una instancia secundaria de Firebase Auth (`SecondaryApp`) para
crear cuentas de conductores. La instancia principal del admin
permanece intacta.

**Alternativas consideradas:**

- Usar Admin SDK desde un backend: descartado porque no hay backend
  propio (ADR-004).
- Crear cuentas manualmente desde Firebase Console: inviable para
  gestión diaria.

**Consecuencias:**

- El admin puede crear conductores sin interrupción de su sesión.
- El código usa inicialización lazy: si la app secundaria no existe,
  la crea con `firebase.initializeApp(config, 'SecondaryApp')`.
- El service account key (`serviceAccountKey.json`) es gitignored.
- **Riesgo identificado y corregido (ADR-018)**: `initializeApp()`
  es asíncrono y retorna `Promise<FirebaseApp>`. El código original
  no usaba `await`, por lo que `secondaryApp` recibía una Promise en
  lugar del objeto FirebaseApp. Se corrigió con `await` en
  `admin_service.ts:79`.

**Referencias:** ARCHITECTURE.md (sección 4, módulo admin),
FIREBASE_SCHEMA.md (sección 4).

---

### ADR-012: DriverApp implementada únicamente para Android

**Estado:** Aceptada.

**Contexto:**

La DriverApp necesita un Foreground Service nativo para mantener la
transmisión GPS activa en segundo plano. Este servicio depende de APIs
exclusivas de Android.

**Decisión:**

La DriverApp está diseñada y probada exclusivamente para Android. No
se invierte tiempo en compatibilidad iOS. La estructura del proyecto
incluye configuración iOS por herencia del template de React Native,
pero no está soportada ni probada.

**Alternativas consideradas:**

- Invertir en iOS con equivalente de Background Modes: descartado por
  el costo de desarrollo y mantenimiento para un MVP universitario.

**Consecuencias:**

- Los conductores deben usar dispositivos Android. La universidad ya
  dispone de teléfonos Android dedicados en los buses.
- El proyecto iOS en DriverApp existe pero no debe considerarse
  funcional.
- Si en el futuro se requiere iOS, deberá implementarse con
  `react-native-background-fetch` u otra estrategia de background
  modes.

**Referencias:** ARCHITECTURE.md (sección 9), BurritoDriverApp/README.md.

---

### ADR-016: Migración a listener multi-bus con fallback al primer bus activo

**Estado:** Aceptada.

**Contexto:**

El sistema inició con un único bus de prueba. La DriverApp escribía a
`/ubicacion_burrito` (legacy) y la UserApp escuchaba ese mismo path.
Al expandir la flota, se necesitaba que la UserApp recibiera ubicaciones
de todos los buses simultáneamente, no solo de uno.

**Decisión:**

Migrar el listener de RTDB de `/ubicacion_burrito` a `/ubicacion_buses`.
El store de Zustand cambió de `location: BurritoLocation | null` a
`locations: Record<string, BurritoLocation>`, indexado por placa.
Cada bus tiene su propio filtro de deduplicación y clasificación
moving/stopped/offline independiente.

Para no afectar la UI antes del render multi-marcador (T4.4), los
componentes de mapa (`Map.tsx`, `MapBranding.tsx`) derivan la ubicación
a mostrar seleccionando el primer bus activo con timestamp más reciente.
Este fallback es temporal y será reemplazado por marcadores individuales
por bus en T4.4.

**Alternativas consideradas:**

- Mantener `/ubicacion_burrito` legacy y agregar un segundo listener:
  descartado. Habría duplicación de suscripciones y el store tendría dos
  fuentes de verdad parcialmente solapadas.
- Migrar directamente a render multi-marcador en el mismo cambio:
  descartado por §5.6 (no expandir alcance). El cambio se dividió en
  lógica de datos (T3.1) y renderizado (T4.4).

**Consecuencias:**

- La UserApp ahora escucha un solo path (`/ubicacion_buses`) que contiene
  todos los buses activos.
- El nodo legacy `/ubicacion_burrito` queda huérfano. La DriverApp ya no
  escribe allí. Puede eliminarse en una limpieza futura de la RTDB.
- El filtro de deduplicación ahora itera por cada placa individual, con
  su propia referencia de timestamp.
- La UI sigue mostrando un solo marcador (fallback al primer bus activo)
  hasta T4.4.
- Se actualizaron 5 archivos: `map_service.ts` (path + callback),
  `burritoLocationStore.ts` (estructura), `MapScreen.tsx` (prop),
  `Map.tsx`, `MapBranding.tsx` (derivación de primer bus activo).

**Referencias:** ARCHITECTURE.md (sección 4 y 5), FIREBASE_SCHEMA.md
(secciones 3 y 7), MVP.md (T3.1, T4.4).

---

### ADR-017: Migración del panel admin a DriverApp + autorización por auth.uid

**Estado:** Aceptada.

**Contexto:**

La UserApp (El Burrito) se atasca en el spinner de Mapbox cuando la
base de datos está limpia (sin asignaciones activas). El spinner es
intencional y no se modifica. El deadlock real es de arquitectura: el
Panel de Gestión vivía en la UserApp y solo se alcanzaba por el drawer
que el spinner bloqueaba — sin asignaciones → la DriverApp no transmite
→ la UserApp nunca recibe datos → spinner infinito.

Además, el modelo de autorización admin basado en `rol` dentro de
`/usuarios/{uid}` era vulnerable: la regla de escritura del propio
usuario sobre su nodo permitía auto-asignarse `rol: 'admin'`.

**Decisión:**

1. Migrar el módulo admin (`features/admin/`) completo de la UserApp a
   la DriverApp. La UserApp queda como cliente exclusivo de estudiantes.
2. Crear un nuevo nodo `/administradores/{auth.uid}` como fuente única
   de verdad de autorización admin. La mera existencia de la clave
   (`snapshot.exists()`) define que el UID es administrador. No se usa
   ni se lee un campo `rol`.
3. El enrutador de DriverApp consulta `existeAdministrador(user.uid)`
   post-login para decidir si renderiza el AdminNavigator o envía al
   conductor a SendCoordinates.
4. Las reglas de RTDB se reescriben para usar el predicado
   `root.child('administradores').child(auth.uid).exists()` en lugar de
   `root.child('usuarios').child(auth.uid).child('rol').val() === 'admin'`.
5. El nodo `/administradores` tiene `.write: false` — solo se puebla
   manualmente desde Firebase Console.

**Alternativas consideradas:**

- Modelo por email (`auth.token.email.replace('@burritodriver.com', '')`
  como clave en `/administradores`): descartado por ser explotable
  mediante registro público con email arbitrario.
- Mantener `/usuarios/{uid}/rol` como fuente admin: descartado por ser
  auto-escribible.
- Custom Claims de Firebase Auth: descartado por requerir Admin SDK o
  Cloud Functions (no hay backend propio).

**Consecuencias:**

- La UserApp ya no tiene dependencia del panel admin. Su único propósito
  es mostrar el mapa al estudiante.
- La DriverApp ahora tiene dos modos de operación: admin (panel de
  gestión) y conductor (tracking).
- El administrador se identifica por su Auth UID, no por DNI ni email.
  El bootstrap requiere crear la cuenta Auth → copiar UID → poblar
  `/administradores/{uid}` manualmente.
- El campo `rol` en `/usuarios/{uid}` queda inerte como legado, siempre
  con valor `'estudiante'`.
- Se agregaron 6 archivos nuevos en DriverApp, se modificaron 3 archivos
  en DriverApp, se eliminaron 5 archivos en UserApp, se modificaron 3
  archivos en UserApp.

**Referencias:** ARCHITECTURE.md (sección 3), FIREBASE_SCHEMA.md
(secciones 4, 9), MVP.md (TA.1, TA.2, TA.3, TA.4).

---

### ADR-018: Corrección de inicialización asíncrona de SecondaryApp

**Estado:** Aceptada.

**Contexto:**

Durante la migración del panel admin a la DriverApp (ADR-017), la
función `createChofer()` en `admin_service.ts` fallaba con el error:
`"firebase.auth(app)" arg expects a FirebaseApp instance or undefined`.

La causa era que `firebase.initializeApp()` en `@react-native-firebase/app`
v23.8.8 retorna `Promise<ReactNativeFirebase.FirebaseApp>`, no el objeto
`FirebaseApp` directamente. El código original asignaba el resultado sin
`await`:

```typescript
secondaryApp = firebase.initializeApp(config, 'SecondaryApp');
// secondaryApp = Promise ❌, no FirebaseApp
```

Posteriormente, `auth(secondaryApp)` ejecutaba un duck-type check interno
del SDK (`namespace.ts:163-168`) que verificaba `'name' in _app`. Una
Promise no tiene propiedad `name`, por lo que el check fallaba.

**Decisión:**

Agregar `await` a la llamada de `initializeApp()` y tipar correctamente
la variable `secondaryApp`:

```typescript
let secondaryApp: ReactNativeFirebase.FirebaseApp;
try {
  secondaryApp = firebase.app('SecondaryApp');
} catch (e) {
  secondaryApp = await firebase.initializeApp(config, 'SecondaryApp');
}
```

**Alternativas consideradas:**

- Mantener `any` y confiar en el try/catch para reintentar: descartado
  porque enmascara el error de tipos.
- Usar `.then()` en lugar de `await`: equivalente funcional pero menos
  legible en un flujo ya async.

**Consecuencias:**

- `secondaryApp` recibe correctamente un `FirebaseApp` en lugar de una
  Promise.
- El duck-type check de `auth()` pasa exitosamente.
- La creación de choferes funciona en el primer intento (ya no requiere
  un segundo intento gracias al registro síncrono en `APP_REGISTRY`).
- TypeScript ahora detectaría el error si se omitiera el `await` (la
  variable está tipada como `FirebaseApp`, no como `any`).
- Se modificó 1 archivo (`admin_service.ts`), 3 líneas en total.

**Referencias:** ARCHITECTURE.md (sección 3, módulo admin),
TROUBLESHOOTING.md (sección 10), admin_service.ts.

---

### ADR-021: Splash nativo único (react-native-bootsplash) en lugar de AnimatedSplash

**Estado:** Aceptada.

**Contexto:**

La UserApp tenía una doble capa de splash: el tema nativo de arranque
(`BootTheme` de `react-native-bootsplash`) y un overlay JS
(`AnimatedSplash.tsx`, animación Lottie) montado en `App.tsx` con
`zIndex: 999` sobre `StyleSheet.absoluteFill`. Al terminar la animación
JS, `onFinish` desmontaba el overlay y el fondo `#00AEEF` (cian) del
`GestureHandlerRootView` quedaba expuesto en el instante de la
transición hacia `WelcomeScreen`, provocando un "flash cian". El
`StatusBar` además dependía de `showAnimatedSplash`, forzando un cambio
de color abrupto.

**Decisión:**

Eliminar `AnimatedSplash.tsx` por completo y dejar el splash nativo de
`react-native-bootsplash` (dependencia ya presente, `^7.1.0`) como única
capa. El splash se oculta en `onReady` del `NavigationContainer`:

```typescript
<NavigationContainer
  onReady={() => {
    requestAnimationFrame(() => BootSplash.hide({ fade: true }));
  }}
>
```

De esta forma el splash nativo cubre el arranque, la hidratación de los
stores y el primer paint del árbol de navegación; al ocultarse con fade,
la primera pantalla ya está lista y no hay flash. El `StatusBar` deja de
depender del estado del splash.

**Alternativas consideradas:**

- Mantener el overlay JS ajustando solo los colores: descartado, no
  elimina la doble capa ni el flash de transición.
- Ocultar el splash con un temporizador fijo: descartado, no garantiza
  que la navegación ya esté pintada (mismo problema de flash o splash
  recortado).
- Ocultar apenas completada la hidratación (`_hasHydrated`): descartado,
  podía revelar la pantalla antes del primer paint de React Navigation.

**Consecuencias:**

- Una sola capa de splash, controlada por el ciclo de vida real del
  árbol de navegación (`onReady`).
- El arranque muestra el logo oficial, regenerado desde
  `assets/splash_logo.png` en `drawable-*/bootsplash_logo.png` y
  `assets/bootsplash/*`. El launcher usa `mipmap-*/ic_launcher*.png`.
- Se eliminó `src/app/screen/AnimatedSplash.tsx`; `App.tsx` ya no usa
  `useState` para el splash ni el overlay absoluto.
- No se agregó ninguna dependencia nueva: `react-native-bootsplash`
  ya estaba en `package.json` (`^7.1.0`).

**Referencias:** ARCHITECTURE.md (sección 4), BUGS_RESUELTOS/userapp.md
(CASO 018).

---

## ADR Planificadas

### ADR-013: Geofencing con Punto Cero Dinámico

**Estado:** Planeada.

**Contexto:**

Se requiere automatizar el conteo de vueltas y el ciclo de vida de los
recorridos sin forzar al conductor a interactuar con la pantalla
mientras conduce.

**Decisión definida:**

Implementar un Punto Cero Dinámico calculado en el dispositivo mediante
la fórmula de Haversine con un umbral de detección de 40 metros. La
función `calculateDistance()` ya existe en `Map.tsx` pero no está
activa.

**Por qué esta dirección:**

- El campus de la UNMSM sufre cambios estructurales constantes
  (obras, eventos), lo que invalida coordenadas fijas.
- El umbral de 40 metros absorbe el error de ±10–15m del efecto
  multipath en el campus.
- Un radio menor (20m) provocaba cierres falsos por imprecisión del
  GPS. Un radio mayor podría causar cierres prematuros.

**Dependencia:** requiere tracking consolidado (Fase 1 del roadmap).

**Referencias:** ROADMAP.md (Fase 2).

---

### ADR-014: Backend propio

**Estado:** Planeada.

**Contexto:**

A medida que el sistema crezca, Firebase RTDB puede volverse
insuficiente para lógica de negocio compleja, reglas de seguridad
dinámicas o integración con servicios externos.

**Decisión definida:**

Incorporar un backend propio (Node.js o similar) para centralizar
lógica, reglas de negocio y servir como intermediario entre las apps
y la base de datos.

**Dependencia:** sin fecha estimada. Visión de largo plazo.

**Referencias:** ROADMAP.md (Visión de Largo Plazo).

---

### ADR-015: Dashboard Web de Monitoreo y Analítica

**Estado:** Planeada.

**Contexto:**

Actualmente no existe una interfaz web para visualizar la flota
completa, acceder a estadísticas operativas, reportes diarios,
métricas de recorridos ni administración avanzada.

**Decisión definida:**

Desarrollar una aplicación web independiente que consuma los datos de
Firebase RTDB (o del backend propio, cuando exista) para proporcionar:

- Mapa de flota completa con todos los buses activos.
- Reportes diarios y semanales de recorridos.
- Métricas de tiempos de espera, ocupación, rutas más usadas.
- Administración avanzada de conductores, buses y asignaciones.
- Analítica histórica.

**Dependencia:** sin fecha estimada. Visión de largo plazo.

**Referencias:** ROADMAP.md (Visión de Largo Plazo).

---

### ADR-019: Heartbeat de presencia en la DriverApp (T4.1)

**Estado:** Aceptada.

**Contexto:**

El GPS publica vía `watchPosition` con `distanceFilter: 2` e
`interval: 3000`. Cuando el bus está quieto (semáforo, paradero), el
GPS no emite callbacks con la misma frecuencia: se observaron pausas
reales de 13–26 s y una de ~36 s. En la UserApp, una pausa de ese
tamaño hace que el bus cruce el umbral de expiración (C4.6,
`BUS_STALE_AFTER_MS = 30000`) y desaparezca del mapa aunque el
servicio siga vivo. Además, el watchdog C3 usaba los pulsos GPS como
señal de vida: con el bus estacionado podía reiniciar el servicio en
falso.

El problema no era el umbral de la UserApp sino la falta de garantía
de publicación periódica desde el Driver.

**Decisión:**

Implementar un heartbeat de presencia de **8 s**
(`HEARTBEAT_INTERVAL_MS = 8000`) dentro de `locationTask`
(`SendCoordinates.tsx`). El heartbeat escribe en RTDB la **última
posición conocida** solo si no hubo otra escritura reciente, y emite
`PRO_LOCATION_PULSE` (pre-write) igual que el GPS, de modo que el
watchdog C3 mide "servicio vivo" y no "GPS activo".

Una única función `writeLocation(coords)` es responsable de escribir
en RTDB; tanto el callback del GPS como el heartbeat la usan. Se
evita así que dos caminos de escritura diverjan con el tiempo.

**Separación conceptual adoptada:**

```
Presencia  → heartbeat del Driver (publica periódico)
Movimiento → desplazamiento entre posiciones publicadas
Expiración → C4.6 en la UserApp (timestamp demasiado viejo)
```

El `timestamp` mantiene su nombre y pasa a significar "última
actualización publicada por el Driver", no "momento del fix GPS".

**Alternativas consideradas:**

- Subir `BUS_STALE_AFTER_MS` en la UserApp: descartado, no ataca la
  causa. El problema está aguas arriba, en la periodicidad del Driver.
- Heartbeat con segunda ruta de escritura separada: descartado, dos
  caminos de escritura divergen; se impone un único `writeLocation`.
- `onDisconnect` de Firebase: postergado (ver ADR-010).

**Consecuencias:**

- Un bus estacionado se mantiene visible (timestamp fresco) y pasa a
  `stopped` por movimiento (C4.6) en vez de `offline` por expiración.
- El watchdog C3 ya no reinicia el servicio con bus quieto.
- Costo de batería mínimo: una escritura extra cada 8 s solo cuando el
  GPS no escribió.
- Pendiente de validación en campo con el Motorola (Grupo A): verificar
  que el heartbeat publica realmente con el GPS quieto.

**Referencias:** ARCHITECTURE.md (sección 5 y 8), FIREBASE_SCHEMA.md
(sección 3), HISTORIAL.md (T4.1 — heartbeat), ROADMAP.md (Bloque A).

---

### ADR-020: C4.6 — Estrategia C experimental para MOVIMIENTO/PARADERO

**Estado:** EXPERIMENTAL — pendiente de validación en campo. NO es una
decisión definitiva.

**Contexto:**

La medición interna de calibración (20/08/2026, ~5 min, 61 parejas)
reveló dos hechos que invalidan los enfoques simples:

1. **Speed congelada:** durante una parada real de 73 s, el GPS del
   Motorola mantuvo `speed = 2.78 m/s` en RTDB (nunca bajó a 0), porque
   el heartbeat escribe la última posición conocida. La estrategia B
   (speed sola) fracasa: en simulación dejó 58 de 60 lecturas como
   EN MOVIMIENTO durante la parada.
2. **Coordenadas congeladas:** con el bus quieto, la posición publicada
   no varió (jitter 0), pero el timestamp seguía fresco por el
   heartbeat (cadencia moda 3 s).

Además, la cadencia GPS real es variable (moda 3 s, con lag gaps de
hasta 21 s), por lo que un umbral fijo de desplazamiento (estrategia A,
8 m) no puede distinguir "avanza 2 m en 3 s" (casi detenido) de "jitter
GPS en parada".

**Decisión (provisional, sujeta a la prueba de campo):**

- **Definición de producto:** EN PARADERO = vehículo físicamente
  detenido. Un vehículo lento (incluso 1 km/h con desplazamiento real)
  es EN MOVIMIENTO. Prohibido usar umbrales semánticos de velocidad
  ("menos de X km/h = parado"). Sin esperas artificiales largas
  (10/15/30 s).
- **Estrategia C** (implementada en UserApp, módulo
  `src/features/map/utils/movement.ts`):

  ```
  vCalc = dist / (dtMs/1000)
  evidence = vCalc > 1 m/s  OR  (speed > 2 m/s AND dist > 2 m)
  ```

  - `vCalc` combina distancia y tiempo: un bus a 10 km/h entre fixes
    de 3 s produce vCalc ~2.8 > 1 → evidencia.
  - `speed` es señal complementaria y **condicional**: solo cuenta si
    hubo desplazamiento real (dist > 2 m), descartando la speed
    congelada del heartbeat.
  - **Memoria/histeresis mínima:** evidencia → EN MOVIMIENTO inmediato;
    sin evidencia se cuentan parejas consecutivas y solo tras 2
    (`STOP_CONFIRM_PAIRS = 2`) se pasa a EN PARADERO.
- **Parámetros EXPERIMENTALES (CALIBRACIÓN PENDIENTE):** 1 m/s, 2 m/s,
  2 m, 2 parejas. Ninguno es valor definitivo; se ajustarán con los
  datos de la prueba de campo. No documentarlos como definitivos.
- **8 m (`MOVEMENT_THRESHOLD_M`, estrategia A) queda fuera de la
  decisión en vivo.** Se conserva únicamente para comparación offline
  A vs C, tests y simulación sobre los mismos datos.
- **Plumbing de speed:** la UserApp ahora recibe `speed` del snapshot
  de RTDB (`speed?: number`, `speed: entry.speed ?? 0`). Sin cambios
  en DriverApp ni en el esquema Firebase (el campo ya existía).
- **Offline independiente:** la expiración (30 s, `isActive`) no se
  mezcla con MOVIMIENTO/PARADERO; la estrategia C solo decide entre
  esos dos estados con datos utilizables.
- **Diagnóstico temporal:** `CALIB_LOG_ENABLED` registra por cada
  actualización `{placa, dist, dtMs, vCalc, speed, evidence,
  previousStatus, status, transition, weakCount}`. Se elimina al
  terminar la evaluación; no es logging de producción.

**Alternativas consideradas y descartadas (con evidencia de la
calibración):**

- Estrategia A (umbral fijo 8 m): 5 transiciones en la prueba interna
  contra 2 reales (falsos PARADERO/MOVIMIENTO por cadencia variable).
- Estrategia B (speed sola): 58/60 EN MOVIMIENTO durante la parada de
  73 s por speed congelada.

**Limitación conocida y aceptada (a medir en campo):**

- A ~1 km/h entre fixes de 3 s (dist ~0.83 m), vCalc ~0.28 < 1 y
  speed < 2 m/s → la estrategia C NO detecta movimiento. Es el caso
  que la prueba de campo debe responder si se puede o no resolver.

**Consecuencias:**

- La UserApp distingue semánticamente vehículo detenido (EN PARADERO)
  de vehículo lento (EN MOVIMIENTO) solo si la señal GPS lo permite;
  el resultado final depende de la prueba de campo.
- Cualquier ajuste de parámetros posterior a la prueba se documentará
  aquí y se marcará como VALIDADO solo con evidencia de campo.

**Referencias:** ARCHITECTURE.md (sección 5), FIREBASE_SCHEMA.md
(sección 3), TROUBLESHOOTING.md (Ubicación congelada), ROADMAP.md,
AGENTS.md (UserApp, sección 9).

---

### ADR-022: Desactivación de la funcionalidad "Seguir Bus" en la UserApp

**Estado:** Aceptada.

**Contexto:**

La UserApp incluía un botón flotante (FAB con icono `crosshairs`) que
centraba la cámara sobre el bus (zoom 17.5) y la mantenía anclada a su
posición mediante un seguimiento continuo (`isFollowing` en `mapStore`,
rama `'follow'` del comando y efecto de cámara en `Map.tsx`). La vista
general por defecto del campus (zoom 15.1, centrada en la UNMSM) ya
muestra el circuito completo de la ruta, por lo que el seguimiento
cercano resultaba redundante para el caso de uso principal.

**Decisión:**

Funcionalidad dejada de lado por decisión del desarrollador. Se eliminó
por completo el botón y toda su maquinaria asociada: props del `FAB`
(`isFollowingBus`, `onFollowBus`, `isBusActive`), evento de analytics
`bus_seguido`, rama `'follow'` del manejador de comandos, efecto de
seguimiento continuo de cámara en `Map.tsx`, cancelación por gesto del
usuario (`onRegionWillChange`) y el estado `isFollowing`/`setIsFollowing`
de `mapStore`. Este ADR es el único registro histórico de que esta
funcionalidad existió; el código anterior permanece recuperable en el
historial de Git si algún día se decide reintroducirla.

**Alternativas consideradas:**

- Mantener la maquinaria dormida (solo ocultar el botón): descartado
  porque deja estado muerto y un comando inalcanzable en un store
  compartido, dificultando el mantenimiento.
- Reutilizar el botón restante para centrar en el bus en lugar del
  campus: descartado; se preserva el comportamiento actual de recentrado
  a la vista general (comando `'center'`) sin cambios.

**Consecuencias:**

- El FAB reduce a un solo botón ("centrar mapa"). No existe forma de
  anclar la cámara al bus; el usuario navega manualmente o usa la vista
  general, donde todo el circuito es visible.
- Desaparece el evento de analytics `bus_seguido`; cualquier tablero o
  embudo que lo use deja de recibir datos desde esta versión.
- `mapStore` queda reducido al comando `'center'`.

**Referencias:** ARCHITECTURE.md (sección 4, tabla de stores;
apéndice de archivos UserApp), Mockups §7c, MVP.md (flujo manual User).

---

### ADR-023: Autorización de escritura en `/ubicacion_buses` por vínculo uid→DNI (`/choferes_uids`)

**Estado:** Aceptada e implementada (23/08/2026).

**Contexto:**

Hasta esta fecha, la escritura del nodo de tracking era
`.write: "auth != null"` (decisión aceptada para el MVP, ADR-017):
cualquier usuario autenticado podía sobrescribir la posición de
cualquier bus, suplantando al conductor real. La auditoría de seguridad
del 23/08 identificó además que la alternativa "barata" de restringir
por dominio de email (`@burritodriver.com`) era inviable: `SignUpScreen.tsx:86`
acepta emails arbitrarios sin verificación (`emailVerified` no se
consulta ni exige en ninguno de los tres repos), por lo que un atacante
podía registrarse como `atacante@burritodriver.com` y pasar el filtro.
El catálogo `/choferes/{dni}` tampoco sirve directamente a las reglas:
sus claves son DNI, no UID, y las Security Rules solo conocen el
`auth.uid` del solicitante.

**Decisión:**

Introducir un índice inverso `/choferes_uids/{uid} = dni`, escrito
automáticamente por `createChofer()` (DriverApp `e09ec2f`, AdminWeb
`9c5c288`) inmediatamente después de crear la cuenta Auth y el registro
en `/choferes`. Las reglas publicadas quedan así:

- `/ubicacion_buses` `.write`: `auth != null && (root.child('administradores').child(auth.uid).exists() || root.child('choferes_uids').child(auth.uid).exists())`
- Nuevo bloque `/choferes_uids`: `.read: false`, `.write` solo admin.
- Eliminado el bloque legacy `/ubicacion_burrito` (denegado por defecto).

Los choferes preexistentes se poblaron con backfill manual desde
Consola (2 entradas, verificadas). La lectura pública del tracking se
mantiene intacta (requisito funcional de UserApp).

**Alternativas consideradas:**

- Autorizar por dominio de email: descartada; sin verificación de
  email es trivialmente burlable (ver Contexto).
- Autorizar conductor→bus específico (que cada uid solo pueda escribir
  SU bus asignado del día): descartada para MVP porque exige evaluar
  `/asignaciones` con condición de fecha actual dentro de las rules,
  algo no expresable de forma fiable (el roll-over diario de
  `fecha = hoy` requiere escritura diaria que solo puede hacer un
  admin). El password=DNI sigue siendo la debilidad real previa; se
  deja anotada como mejora post-MVP.

**Consecuencias:**

- Un usuario autenticado ya no puede publicar ubicación salvo que su
  uid esté indexado en `/choferes_uids` o sea admin. Suplantación de
  bus desde cuentas de estudiante: cerrada.
- Postura fail-closed: si falla la última escritura de `createChofer()`
  (el índice), el chofer existe pero no puede transmitir hasta reparar
  el índice manualmente desde Consola.
- Todo chofer creado desde una build vieja de AdminWeb desplegada
  nacerá SIN índice (no podrá trackear): obligatorio redeployar
  Firebase Hosting tras el merge.
- `/choferes_uids` duplica parcialmente información de `/choferes`;
  mantener ambos sincronizados es responsabilidad de `createChofer()`.
- La matriz de validación quedó probada end-to-end el 23/08: simulador
  de reglas (estudiante denegado, conductor permitido, admin permitido)
  + pruebas reales web y móvil con choferes nuevo (99999999) y
  backfilleado (44332211).

**Referencias:** FIREBASE_SCHEMA.md (§2 claves primarias, §4 nodo
`/choferes_uids`, §9 modelo de autorización), MVP.md (§3 Seguridad,
Bloque 2 S3), HISTORIAL.md.

---

### ADR-024: Auditoría mínima en `/asignaciones` (`createdAt` epoch + `createdBy` UID) y semántica de "desactivar"

**Estado:** Aceptada e implementada (24/08/2026).

**Contexto:**

El modelo de gestión del MVP permitía crear asignaciones sin rastro de
quién las creó ni cuándo. Ante un dato inconsistente (asignación
inesperada, turno duplicado) no había forma de reconstruir qué ocurrió.
Se evaluó añadir campos de auditoría y se definieron tres decisiones de
diseño:

1. **Formato del timestamp**: se descartó guardar la hora como string
   localizado (`"24/08/2026, 07:43:15 a. m."`). Un string así impide
   ordenar cronológicamente, consultar por rangos o calcular duraciones
   sin parsear. Se respeta la convención de FIREBASE_SCHEMA.md §2:
   en RTDB se guarda Unix timestamp en milisegundos (`Date.now()`); el
   formato legible es responsabilidad exclusiva de la UI
   (`toLocaleString('es-PE', { timeZone: 'America/Lima' })`), que
   convierte al mostrar. Datos estructurados en DB, presentación
   localizada en pantalla.
2. **Autoría sin fallback falso**: `createdBy` toma siempre el UID real
   de la sesión admin (`auth.currentUser.uid`, vía `getAuth()` en
   AdminWeb y `auth()` en DriverApp). Si no hay administrador
   autenticado, `createAsignacion()` falla con error explícito ("No hay
   administrador autenticado.") y no se escribe nada. Se descartó el
   fallback `|| 'unknown'` porque introduciría un dato falso en la base.
   No se creó `/administradores_uids`: ya existe `/administradores/{uid}`
   como índice por UID (a diferencia de `/choferes`, indexado por DNI,
   que sí necesitó `/choferes_uids`).
3. **Alcance limitado a asignaciones**: `choferes` y `buses` NO reciben
   campos de auditoría por ahora. Criterio: solo se guarda un dato si
   cambia una decisión operativa del MVP. La fecha de alta de un
   chofer/bus no condiciona nada hoy; agregarla sería construir el
   post-MVP antes de validar el piloto.

**Decisión complementaria — semántica de "desactivar" (Opción A):**

Se documenta formalmente que desactivar conductor o bus
(`activo: false` en `/choferes/{dni}` o `/buses/{placa}`) afecta solo la
**elegibilidad futura**: el elemento deja de aparecer en los selectores
de nueva asignación, pero NO revoca la autorización de escritura GPS ni
interrumpe un recorrido ya iniciado. El tracking continúa hasta que el
conductor presione DETENER TODO. Razones:

- Desactivar es decisión administrativa; cancelar asignación es acción
  operativa. Son conceptos distintos y usan herramientas distintas.
- Revocar el GPS en caliente (Opción B) dejaría al bus desaparecido del
  mapa sin aviso al conductor ni al estudiante: peor escenario de
  producto que el falso positivo administrativo.
- Con un único admin, el caso de uso real ("sacar a alguien AHORA") no
  existe aún; si existiera, post-MVP: botón "Forzar detención" que
  escriba `isActive: false` directo en `/ubicacion_buses/{placa}`.

**Alternativas consideradas:**

- String localizado en RTDB: descartada (ver punto 1).
- `createdBy: 'unknown'` como fallback: descartada (dato falso).
- Auditoría también en choferes/buses: diferida al post-MVP.
- Revocación GPS inmediata al desactivar: descartada para MVP (Opción B,
  ver arriba).

**Consecuencias:**

- Payload de `/asignaciones` gana `createdAt` (number, epoch ms) y
  `createdBy` (string, UID). Ambos opcionales: registros previos siguen
  siendo válidos y las pantallas renderizan condicionalmente.
- `fecha` conserva su significado propio (día del turno); `createdAt`
  registra el momento de creación. Puede darse `fecha: 2026-08-25` con
  `createdAt` del 24/08 23:50 si el admin prepara turnos con antelación.
- `createdBy` es auditoría, no seguridad: un cliente malicioso podría
  escribir un UID ajeno en ese campo si las reglas lo permitieran. La
  autorización real sigue determinada por `auth.uid` en
  `/administradores` vía Security Rules (ADR-017/023).
- Congelamiento: tras este cambio no se agregan más funcionalidades
  administrativas al MVP; el siguiente hito es E2E de campo.

**Referencias:** FIREBASE_SCHEMA.md (§4 nodo `/asignaciones`),
HISTORIAL.md (S5 — semántica de desactivación), MVP.md (§4 W12).

---

## Referencias

| Documento | Relación |
|-----------|----------|
| `PROJECT_CONTEXT.md` | Visión general y propósito del sistema. |
| `ARCHITECTURE.md` | Flujo de datos, ciclo de vida y componentes. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos impactada por estas decisiones. |
| `TROUBLESHOOTING.md` | Problemas derivados de estas decisiones. |
| `ROADMAP.md` | Fases donde se ejecutarán las ADR planificadas. |
| `BUGS_RESUELTOS/` | Bugs resueltos que originaron decisiones arquitectónicas. |
| BurritoUserApp/README.md | Setup y funcionalidad de UserApp. |
| BurritoDriverApp/README.md | Setup y funcionalidad de DriverApp. |
| BurritoUserApp/AGENTS.md | Reglas de trabajo para IA en UserApp. |
| BurritoDriverApp/AGENTS.md | Reglas de trabajo para IA en DriverApp. |
