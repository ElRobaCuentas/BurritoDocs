## CASO 001 — El motor funcionaba 2-3 veces y se detenía al ir al fondo
Síntoma
El rastreo GPS transmitía correctamente mientras la app estaba en primer plano. Al minimizar o bloquear la pantalla, el motor se detenía después de 2 o 3 ciclos.
Diagnóstico
POST_NOTIFICATIONS estaba declarado en AndroidManifest.xml pero nunca se pedía en runtime. En Android 13+ esto es obligatorio. Sin el permiso en tiempo de ejecución, Android bloquea la notificación silenciosamente. Sin notificación visible → el Foreground Service no se ancla al sistema → Android lo trata como proceso de fondo normal → lo mata al detectar inactividad de UI.
La cadena completa
No se pide POST_NOTIFICATIONS → sin notificación → sin Foreground Service real → Android mata el proceso → GPS para.
Solución
Agregar POST_NOTIFICATIONS como Paso 1/3 en requestAllPermissions(), con validación de versión Platform.Version >= 33. En versiones anteriores se omite porque Android lo concede automáticamente.
Archivo
SendCoordinates.tsx
Estado
Resuelto
Lección
Declarar un permiso en el Manifest no es suficiente en Android moderno. Los permisos sensibles deben pedirse en runtime. Android 13 rompió silenciosamente el comportamiento anterior con las notificaciones.












## CASO 002 — El bus "viajaba en el tiempo" al recuperar señal
Síntoma
Después de perder cobertura durante varios minutos (zonas sin señal del campus), al reconectarse el bus aparecía recorriendo el mapa a velocidad imposible, pasando por posiciones que ya eran del pasado.
Diagnóstico
setPersistenceEnabled(true) estaba activo en index.js. Esta función de Firebase está diseñada para apps de datos offline (listas de tareas, notas). Para un tracker en tiempo real es contraproducente: acumula en memoria todos los intentos de escritura fallidos durante la pérdida de señal (~200 posiciones en 10 minutos). Al reconectarse, los sube todos en ráfaga. La UserApp los recibe en secuencia y el bus recorre retroactivamente toda esa ruta.
Solución
Eliminar setPersistenceEnabled(true) por completo. El comportamiento por defecto de Firebase (descartar el intento fallido, enviar la posición actual en el siguiente ciclo) es exactamente el correcto para un tracker en tiempo real.
Archivo
index.js
Estado
Resuelto
Lección
Las features de Firebase pensadas para apps offline son incompatibles con apps de tracking en tiempo real. "Persistencia" en Firebase significa "acumula y reintenta todo", no "envía solo lo actual".












## CASO 003 — El rastreo se detenía al deslizar la app del administrador de tareas
Síntoma
Si el conductor deslizaba la app fuera del administrador de tareas de Android (el botón de cuadrados), el motor de rastreo se detenía completamente aunque el Foreground Service debería seguir activo.
Diagnóstico
Faltaba android:stopWithTask="false" en la declaración del servicio en AndroidManifest.xml. El valor por defecto de Android es stopWithTask="true", lo que significa: si la actividad principal muere (por cualquier razón), mata también el servicio asociado.
Solución
Agregar android:stopWithTask="false" al bloque <service> del Manifest. Con esto, el servicio de rastreo tiene ciclo de vida independiente de la UI.
Archivo
AndroidManifest.xml
Estado
Resuelto
Lección
En Android, un Foreground Service y una Activity son dos procesos con ciclos de vida distintos. Si no se declara explícitamente que son independientes, el sistema los trata como uno solo.















## CASO 004 — GPS se pausaba al bloquear la pantalla a pesar de tener permiso de ubicación
Síntoma
Con ACCESS_FINE_LOCATION concedido, el GPS se detenía en cuanto el conductor bloqueaba la pantalla o abría otra app. En pantalla activa funcionaba perfectamente.
Diagnóstico
ACCESS_BACKGROUND_LOCATION no se estaba pidiendo como permiso separado en runtime. Android 10 en adelante exige que este permiso se solicite en un paso independiente, después de que ACCESS_FINE_LOCATION ya esté aprobado. Pedirlos juntos resulta en que el sistema ignora o deniega el de segundo plano. Adicionalmente, el usuario debe elegir "Permitir siempre" explícitamente (no "Solo mientras se usa").
Solución
Agregar ACCESS_BACKGROUND_LOCATION como Paso 3/3 en requestAllPermissions(), con un Alert que abre directamente los ajustes del sistema si es denegado, guiando al conductor a elegir "Permitir siempre".
Archivo
SendCoordinates.tsx
Estado
Resuelto
Lección
En Android 10+, "ubicación en primer plano" y "ubicación en segundo plano" son dos permisos completamente distintos con solicitudes separadas. Tener el primero no implica tener el segundo.











## CASO 005 — heading y speed llegaban como null a Firebase
Síntoma
En ciertos dispositivos o con baja precisión GPS, el ícono del bus en el mapa no rotaba correctamente, o aparecía en posición inesperada.
Diagnóstico
position.coords.heading y position.coords.speed pueden ser null en la API de Geolocation cuando el sensor no tiene suficiente información de movimiento (dispositivo quieto, señal débil, primer fix del GPS). Sin manejo, esos null se escribían en Firebase y la UserApp los recibía, causando errores al intentar calcular la rotación del marcador.
Solución
Aplicar nullish coalescing al leer las coordenadas: heading: position.coords.heading ?? 0 y speed: position.coords.speed ?? 0. Si el sensor no entrega el valor, se envía 0 como valor neutral.
Archivo
SendCoordinates.tsx
Estado
Resuelto
Lección
La API de Geolocation de Android no garantiza todos los campos del objeto coords. Los campos opcionales como heading, speed y altitude deben tratarse defensivamente con valores por defecto.



## CASO 008 — RTDB sin reglas de seguridad
Síntoma
Cualquier persona con la URL de Firebase podía leer y escribir en todos los nodos de la base de datos sin autenticación. Un atacante podía enviar coordenadas falsas al mapa o leer los datos de todos los usuarios.
Diagnóstico
Las reglas de RTDB estaban configuradas con ".read": true y ".write": true a nivel raíz — la configuración por defecto de Firebase que no debió dejarse en producción. La DriverApp escribía en /ubicacion_burrito sin ningún token de autenticación, lo que impedía restringir la escritura.
Solución
Tres cambios simultáneos: (1) Crear cuenta de conductor en Firebase Auth (chofer@burritounmsm.com) y agregar login silencioso en DriverApp.tsx. (2) Registrar SHA-1 de debug y release de la DriverApp en Firebase Console. (3) Publicar reglas de RTDB con principio de mínimo privilegio: /ubicacion_burrito lectura pública y escritura solo para el UID del conductor, /usuarios solo accesible por el propio usuario autenticado, /comentarios escritura solo para usuarios autenticados.
Archivos
DriverApp.tsx, Firebase Console (RTDB Rules), Firebase Console (Authentication)
Estado
Resuelto
Lección
Las reglas abiertas de Firebase son el error de seguridad más común en proyectos móviles. Nunca dejar ".write": true a nivel raíz en producción. Si una app necesita escribir sin login visible, usar autenticación silenciosa en segundo plano en lugar de dejar la DB abierta.









## CASO 009 —Efecto "Yo-Yo" por Ruido de GPS y Polling 

Síntoma
El bus en el mapa avanza y retrocede constantemente (efecto chicle/resorte) en un rango de 5 a 15 metros, incluso cuando el vehículo real se desplaza de forma lineal y constante. 
Diagnóstico
Uso de getCurrentPositionAsync dentro de un bucle while. Al "apagar y encender" la petición del GPS cada 3 segundos, el chip de hardware del Motorola no lograba estabilizarse ni aplicar sus filtros de suavizado internos (Kalman). Cada lectura era un "disparo en frío" que incluía el ruido natural del sensor, el cual era interpretado por la UserApp como movimiento real hacia atrás. 
Solución
Migración de la lógica de captura de "Polling" a "Streaming" mediante Geolocation.watchPosition. Esto mantiene el sensor activo y permite que el hardware descarte lecturas erráticas. Además, se redujo el maximumAge de 4000ms a 1000ms para evitar el envío de coordenadas recicladas. 
Archivos
SendCoordinates.tsx
Estado
Resuelto
Lección
Para seguimiento en tiempo real, nunca uses un bucle con peticiones únicas. El GPS necesita continuidad para ser preciso. 


## CASO 010 — Async initializeApp + SecondaryApp

Síntoma:
Al crear un chofer desde el panel admin, la función createChofer lanzaba:
"firebase.auth(app) arg expects a FirebaseApp instance or undefined".

Causa Raíz:
firebase.initializeApp() en @react-native-firebase/app v23.8.8 retorna
Promise<ReactNativeFirebase.FirebaseApp>, no el objeto directamente. El código
asignaba el resultado sin await, por lo que secondaryApp era una Promise.
El duck-type check interno del SDK ('name' in _app en namespace.ts:163-168)
fallaba porque una Promise no tiene propiedad 'name'.

Diagnóstico:
Revisar admin_service.ts en BurritoDriverApp. Si secondaryApp no tiene await
antes de initializeApp(), la variable recibe una Promise en lugar de FirebaseApp.

Solución:
Agregar await a firebase.initializeApp() y tipar secondaryApp como
ReactNativeFirebase.FirebaseApp:

let secondaryApp: ReactNativeFirebase.FirebaseApp;
try { secondaryApp = firebase.app('SecondaryApp'); }
catch (e) { secondaryApp = await firebase.initializeApp(config, 'SecondaryApp'); }

Archivo:
BurritoDriverApp/src/features/admin/services/admin_service.ts

Estado:
Resuelto.

Lección:
initializeApp() en React Native Firebase es asíncrono. Siempre usar await
y tipar explícitamente. El tipo any oculta errores de inicialización que
el SDK no puede manejar.

---

## CASO 011 — El tracking se detenía en silencio sin avisar al conductor

Síntoma:
El servicio que envía la ubicación podía detenerse de forma silenciosa
(por Android, batería o error del sistema) y el conductor no se enteraba:
la app seguía mostrando "Compartiendo ubicación", pero el bus ya no
escribía en RTDB y desaparecía del mapa de los estudiantes.

Causa Raíz:
`BackgroundJob.isRunning()` de react-native-background-actions solo
devuelve un flag JS que cambia con `stop()` explícito; no detecta la
muerte nativa del servicio. Además no existía ningún mecanismo que
intentara recuperar el servicio ni avisar al conductor.

Diagnóstico:
El panel de diagnóstico de SendCoordinates mostraba pulsos "✅ POSICIÓN"
y luego nada: el servicio había muerto pero la UI seguía en estado activo.

Solución (tarea C3):
- Pulso de vida: cada fix de `watchPosition` emite `PRO_LOCATION_PULSE`
  hacia la UI.
- Watchdog: con el recorrido activo, cada 10s verifica si pasaron más de
  30s sin pulso. De ser así, `restartBackgroundJob()`: `BackgroundJob.stop()`
  en try/catch + `BackgroundJob.start()` (idempotente en nativo).
- Si el reinicio falla (p. ej. `ForegroundServiceStartNotAllowedException`
  al intentar arrancar desde background en Android 14): `isSending=false`,
  banner persistente "El envío de ubicación se detuvo y no pudo
  recuperarse automáticamente" y Alert con instrucción de reintentar.

Archivo:
BurritoDriverApp/src/screen/SendCoordinates.tsx

Estado:
Resuelto — validado en hardware real (Motorola G24 Power):
- Reinicio automático: 3 ciclos puros en panel debug sin interacción.
- Fallo de recuperación: reinicio desde background rechazado por el
  sistema (`startForegroundService() not allowed due to
  mAllowStartForeground false`) → banner + botón INICIAR restaurados.
- No-regresión: INICIAR/DETENER verificados con `isActive` en RTDB.

Lección:
`isRunning()` de la librería no es una fuente de verdad confiable para
detectar muerte del servicio; los pulsos de datos reales sí. Un FGS
no puede arrancar desde background en Android 14: la UI debe avisar
cuando la recuperación automática no es posible.

---

## CASO 012 — El cierre del tracking podía dejar la UI en "Compartiendo ubicación" (C4.2)

Síntoma:
Si `BackgroundJob.stop()` fallaba (o había doble pulsación sobre DETENER), la
UI podía quedarse en "Compartiendo ubicación" aunque el servicio ya estuviera
detenido, o lanzar errores al detener en momentos intermedios (servicio ya
detenido, watchdog reiniciando).

Causa Raíz:
`stopProcess` ejecutaba `BackgroundJob.stop()` sin protección: si lanzaba error,
`setIsSending(false)` nunca se alcanzaba y el estado quedaba inconsistente.
Tampoco había guard contra ejecuciones en paralelo (doble toque o colisión con
el reinicio del watchdog).

Solución (tarea C4.2):
- Guard `isStoppingRef` (useRef): impide que `stopProcess` se ejecute en paralelo.
- `BackgroundJob.stop()` envuelto en try/catch: un error al detener no aborta la rutina.
- Verificación del resultado de `stopBusService(busId)`: si no marca
  `isActive:false`, se loguea el fallo.
- `setIsSending(false)` y liberación del guard en `finally`: el estado siempre
  converge a "Detenido".

Archivo:
Driver/src/screen/SendCoordinates.tsx

Estado:
Resuelto — validado en hardware real (Motorola G24 Power):
- Ciclo completo INICIAR → DETENER: UI "Detenido", `isActive=false` en RTDB.
- Doble toque DETENER: sin crash, UI "Detenido" (idempotente).
- DETENER durante reinicio del watchdog (C3): UI consistente "Detenido".
- Sin FATAL EXCEPTION en logcat.

Lección:
El apagado del tracking debe ser idempotente y tolerante a cualquier orden de
llamadas: nunca asumir que `stop()` no falla, y garantizar con `finally` que la
UI siempre converja al estado real.

---

## CASO 013 — Al reabrir la app con el servicio activo, la UI se desincronizaba (C4.7)

Síntoma:
Con el recorrido activo, al deslizar la app desde recientes (el servicio nativo
sigue vivo por `stopWithTask="false"` y sigue transmitiendo) y reabrirla, la UI
mostraba "INICIAR RECORRIDO" aunque el servicio seguía activo; el botón INICIAR
no respondía (el guard `if (BackgroundJob.isRunning()) return` abortaba en
silencio), dejando al conductor sin control sobre el servicio.

Causa Raíz:
`isSending` es `useState(false)`: al re-montar `SendCoordinates` el estado
visual se reinicia sin consultar la realidad del motor. El servicio y el
singleton JS sobreviven (mismo proceso, `stopWithTask="false"`), pero la UI
asumía "detenido".

Solución (tarea C4.7):
En el `useEffect` de montaje, después de registrar los listeners (para que
`sendLog` funcione) y antes de `fetchAssignment()` (solo restaura estado visual,
no depende de Firebase ni modifica la asignación): si `BackgroundJob.isRunning()`
es `true`, restaurar el estado del motor — `setLogs([])`, `lastPulseRef.current
= Date.now()`, `setRecoveryFailed(false)`, `setIsSending(true)` — y loguear
"Recorrido restaurado". No se crea un segundo BackgroundJob ni se altera el
ciclo de vida del servicio; `isActive` de Firebase sigue sin ser fuente de
verdad.

Archivo:
Driver/src/screen/SendCoordinates.tsx

Estado:
Resuelto — validado en hardware real (Motorola G24 Power):
- INICIAR → quitar de recientes (proceso vivo) → reabrir: UI "Compartiendo
  ubicación" + DETENER RECORRIDO con log "Recorrido restaurado".
- DETENER sobre el servicio restaurado: UI "Detenido", servicio detenido,
  `isActive=false` en RTDB.
- Regresión C4.2: ciclo INICIAR→DETENER y doble toque DETENER sin crash.
- Sin FATAL EXCEPTION en logcat. Nota: en sala sin fix GPS no hubo escrituras
  nuevas (timestamp congelado); la restauración y el apagado se validaron por
  UI, proceso y RTDB.

Lección:
La UI no debe asumir el estado del motor desde cero en cada montaje: si el
proceso y el servicio sobrevivieron, `BackgroundJob.isRunning()` (mismo proceso)
es una fuente de verdad fiable para restaurar el estado visual y devolverle el
control al conductor.

---

## CASO 014 — El auth gate colgaba offline: spinner infinito sin CERRAR SESIÓN (C4.AUTH)

Síntoma:
Con sesión guardada y sin red (modo avión), la DriverApp quedaba en un spinner
infinito sin botón CERRAR SESIÓN. `SendCoordinates` nunca se montaba, por lo
que era imposible validar C4.3 offline. Espera >30 min sin cambio de estado.

Causa Raíz:
`existeAdministrador()` (`admin_check.ts`) usaba `once('value')` de rnfirebase.
Sin persistence (ADR-003) y sin red, `once('value')` **ni resuelve ni rechaza**:
cuelga indefinidamente. El efecto de `DriverApp.tsx` quedaba con
`isCheckingRole=true` para siempre; el `.catch` (que sí tenía UI de error con
REINTENTAR y CERRAR SESIÓN) nunca se disparaba.

Solución (tarea C4.AUTH):
Acotar el read con timeout a nivel de servicio: `existeAdministrador` envuelve
el `once('value')` en `withTimeout(..., ROLE_TIMEOUT_MS=10000)` (helper compartido
`src/shared/utils/timeout.ts`, extraído de C4.3). Al vencer el timeout la promesa
rechaza, el `.catch` existente de `DriverApp.tsx` dispara la pantalla de error
(roleError) y el `.finally` libera el spinner. Sin nuevas pantallas: se reutilizó
la UI que ya existía (REINTENTAR incrementa `roleAttempt` y re-ejecuta el efecto;
CERRAR SESIÓN usa `auth().signOut()`, operación local que funciona offline). Sin
reintentos automáticos (decisión de diseño: el gate corre una vez al abrir la
app; 10s + esperar y reintentar empeoraría la UX).

Archivos:
- `Driver/src/shared/utils/timeout.ts` (nuevo: `withTimeout`, `pause`)
- `Driver/src/features/admin/services/admin_check.ts`
- `Driver/src/screen/SendCoordinates.tsx` (reemplaza los helpers locales por el import compartido)
- `Driver/src/DriverApp.tsx` (sin cambios netos: la UI de roleError ya cubría el caso)

Estado:
Resuelto — validado en hardware real (Motorola G24 Power):
- Red normal: el gate resuelve y entra a SendCoordinates (sin regresión).
- Modo avión + WiFi off (sin red real, 100% pérdida y sin DNS): el timeout de
  10s dispara `existeAdministrador REJECTED -> roleError` y
  `isCheckingRole false` (logcat), mostrando "No se pudo verificar tu cuenta"
  con REINTENTAR y Cerrar sesión (dump uiautomator). Antes quedaba en spinner
  eterno sin salida.
- Red restaurada + tap REINTENTAR: resuelve el rol y navega a SendCoordinates
  (Conductor Juan Matias / AHK-124).
- `npx tsc --noEmit` exit 0. Los 2 errores de lint son preexistentes en HEAD
  (`asignacionId` en SendCoordinates:135, `e` en admin_service:79) y están fuera
  de alcance.

Lección:
Todo read con `once()` de rnfirebase sin persistence cuelga offline (ni resuelve
ni rechaza). Nunca debe quedar sin acotación de tiempo; el timeout debe vivir en
la capa de servicio para proteger a todos sus llamadores. Nota: `admin_service.ts`
(l.67, 147, 225) tiene más `once('value')` sin acotar, pero son del panel admin
tras login y quedaron fuera de alcance de C4.AUTH — evaluar como deuda separada.

---

## CASO 015 — La carga de la asignación colgaba sin límite y sin salida ante la pérdida de red (C4.3)

Síntoma:
Con red caída durante el montaje de `SendCoordinates`, el spinner "Cargando
asignación..." podía quedarse indefinidamente: la lectura de `/asignaciones` (y
de `/choferes/{dni}`) no resolvía, no había límite de tiempo, ni reintento
acotado, ni una forma de salir (el CERRAR SESIÓN solo aparecía tras renderizar
la pantalla normal). El conductor quedaba atrapado en el spinner sin informar
del problema.

Causa Raíz:
`fetchAssignment` (dentro del `useEffect` de montaje) usaba dos `once('value')`
de rnfirebase. Sin persistence (ADR-003) y sin red, `once()` **ni resuelve ni
rechaza** (mismo comportamiento empírico que el auth gate de C4.AUTH): la
promesa queda colgada y `setLoadingAssignment(false)` del `finally` jamás se
alcanza. No había timeout, ni reintentos, ni estado de error en la UI.

Solución (tarea C4.3):
Extraer la carga a `loadAssignment` (useCallback) y acotarla con las primitivas
compartidas `withTimeout` + `pause` (`src/shared/utils/timeout.ts`, heredadas de
C4.AUTH):
- Timeout por intento COMPLETO: `ASSIGNMENT_TIMEOUT_MS = 10000` cubre lectura de
  chofer + lectura de asignación con un solo reloj.
- Reintentos acotados: `ASSIGNMENT_RETRIES = 2` con pausa
  `ASSIGNMENT_RETRY_PAUSE_MS = 1500`. La UI muestra "Problemas de conexión.
  Reintentando…" mientras reintenta.
- Guard de generación (`fetchGenRef`): las respuestas tardías de un intento
  vencido no sobrescriben un ciclo más nuevo (Promise.race no cancela la
  consulta, solo deja de esperarla).
- Guard anti-paralelo (`isLoadingAssignmentRef`): REINTENTAR no lanza ciclos
  simultáneos.
- Resultado válido sin asignación: Firebase respondió y no hay bus → NO se
  reintenta (mensaje "No hay asignación para hoy").
- Al agotar los reintentos: errorCard visible con REINTENTAR (nuevo ciclo) y
  CERRAR SESIÓN (siempre accesible). Nunca se atrapa al conductor.
- El guard `isLoadingAssignmentRef` se libera en `finally` en TODOS los caminos
  para que REINTENTAR siga funcionando.

Archivo:
Driver/src/screen/SendCoordinates.tsx

Estado:
Resuelto — validado en hardware real (Motorola G24 Power):
- Offline (corte de red al ver "Cargando asignación..."): log de diagnóstico
  ❌ Intento 1/2 falló → ⚠️ Problemas de conexión, reintentando (2/2) → ❌
  Intento 2/2 falló → errorCard "No se pudo cargar tu asignación / Verifica tu
  conexión a Internet e inténtalo nuevamente." con REINTENTAR y CERRAR SESIÓN.
  Sin spinner infinito. 2 ciclos completos.
- Guard anti-paralelo: 3 taps rápidos a REINTENTAR con red apagada → 1 solo
  ciclo, sin llamadas paralelas ni crash.
- Recuperación: `svc wifi enable` + REINTENTAR → "Conductor identificado: Juan
  Matias" + "Bus asignado: AHK-124", pantalla normal.
- CERRAR SESIÓN desde la errorCard: diálogo de confirmación → pantalla de login.
- Regresión INICIAR→DETENER: tracking real en RTDB (`isActive=true` → `false`),
  UI "Compartiendo ubicación" → "Detenido".
- Sin FATAL EXCEPTION. `npx tsc --noEmit` exit 0.

Lección:
Todo read con `once()` de rnfirebase sin persistence cuelga offline (ni resuelve
ni rechaza). Además de acotar con timeout el gate de autenticación (C4.AUTH),
la carga crítica de la asignación debe tener timeout por intento, reintentos
acotados y una tarjeta de error con salida (REINTENTAR/CERRAR SESIÓN) para que
el conductor nunca quede atrapado en un spinner sin límite.

---

## CASO 019 — El heartbeat escribía cada ~16s en paradero en vez de cada 8s (T4.1)

Síntoma:
Tras implementar el heartbeat T4.1 (`setInterval` de `HEARTBEAT_INTERVAL_MS =
8000`), la Prueba 1 (20/08/2026) mostró que la cadencia real de escrituras en
RTDB durante PARADERO era de **~16.25 s** (huecos deterministas de 16 s entre
timestamps consecutivos), el doble de la cadencia configurada. En MOVIMIENTO la
cadencia era correcta (~8 s). Consecuencia directa: la frescura percibida por
la UserApp se degradaba exactamente cuando el bus está detenido, que es el
escenario crítico para distinguir "bus parado" de "bus desaparecido".

Causa Raíz:
Carrera ε entre el marcador de frescura y el chequeo del intervalo.
`writeLocation` asignaba `lastWriteAt = Date.now()` DESPUÉS del `await` de la
escritura en RTDB, mientras el tick del heartbeat evaluaba
`Date.now() - lastWriteAt < HEARTBEAT_INTERVAL_MS` de forma independiente. Si
el tick vencía mientras una escritura estaba en vuelo (o justo antes de que el
`await` resolviera y actualizara el marcador), la condición daba "vencida",
disparaba una segunda escritura y reiniciaba el ciclo desfasado: dos ciclos de
8 s colisionados producían un hueco efectivo de ~16 s. En PARADERO —sin GPS
updates reales que re-frescaran el marcador— el desfase era determinista y se
acumulaba visible; en MOVIMIENTO cada update GPS enmascaraba el problema.

Solución (refinamiento de T4.1):
- `lastWriteAt = Date.now()` se fija ANTES del `await` de la escritura: el
  instante del INTENTO es la marca de frescura, no el instante de confirmación
  de RTDB. Así ningún tick puede evaluar la condición con el marcador
  "viejo" mientras la escritura está en vuelo.
- En el `catch` de la escritura fallida, `lastWriteAt = 0`: si RTDB rechaza,
  el siguiente tick reintenta de inmediato en vez de esperar otro intervalo
  completo creyendo que los datos están frescos.
- Sin cambios en el intervalo (8000 ms), ni en `writeLocation` como API, ni en
  el esquema de RTDB.

Archivo:
Driver/src/screen/SendCoordinates.tsx

Estado:
Resuelto — verificado empíricamente con captura directa de RTDB (22/08/2026),
dos sesiones independientes (`stat_rtdb.jsonl`, `stat2_rtdb.jsonl`, muestreo a
1 Hz sobre `/ubicacion_buses`):
- Sesión 1 (paradero sostenido): deltas de timestamp 8003–8005 ms, **cero
  valores ≥16 s** en toda la sesión.
- Sesión 2: idem, cadencia estable 8003–8005 ms durante todo el paradero.
- Antes del fix: medias de ~16.25 s documentadas en los reportes de Prueba 1.
- `npx tsc --noEmit` exit 0. Lint sin errores nuevos. Commit `99eafa0`
  (rama `feat/driverapp/heartbeat`). Decisión registrada en ADR-019.

Lección:
Nunca fijar un marcador de frescura DESPUÉS de un `await` cuando un intervalo
independiente evalúa ese mismo marcador: el hueco entre intento y confirmación
es justamente la ventana donde el intervalo decide mal. La marca debe capturar
el instante del intento, y el camino de error debe invalidar el marcador para
no postergar el reintento.

---

## Hallazgos de auditoría pendientes (sin corrección)

Hallazgos detectados durante la validación de C4.2. Se registran por separado
porque no fueron corregidos en esa tarea y requieren evaluación propia.

### Hallazgo A — Cerrar la app desde recientes con tracking activo desincroniza la UI
Si el conductor inicia el recorrido, desliza la app desde recientes (el servicio
sigue vivo por `stopWithTask="false"` y escribe en RTDB) y la reabre, la UI
muestra "INICIAR RECORRIDO" (estado por defecto al re-montar) aunque el servicio
sigue transmitiendo; el botón INICIAR no responde.
→ Convertido en la tarea **C4.7** en el backlog (`MVP.md`). Sonda previa
validada en el Motorola (G24 Power): tras INICIAR → quitar de recientes →
reabrir, `BackgroundJob.isRunning()` devuelve **`true`** con el servicio vivo
(mismo proceso, singleton JS conservado). Fuente de verdad fiable para la
corrección.
→ **Resuelto por C4.7** (ver CASO 013): al re-montar `SendCoordinates`, si
`isRunning()` es `true` se restaura `isSending` y la UI recupera "Compartiendo
ubicación" + DETENER RECORRIDO.

### Hallazgo B — `isActive=true` con timestamp congelado — RESUELTO (22/08)
Puede quedar `isActive=true` en RTDB mientras el servicio está muerto y el
timestamp deja de actualizarse. Validar en C4.6/T1 que BurritoUserApp oculta
el bus por inactividad; si no lo oculta, abrir subtarea bloqueante.
→ **Resuelto por diseño**: el heartbeat T4.1 garantiza timestamp fresco cada
8 s (CASO 019, verificado en campo con deltas de 8003–8005 ms) y la UserApp
oculta el bus por OFFLINE (estrategia C, ADR-020) cuando los datos caducan.
Ambas piezas fusionadas a main (`99eafa0`, `916f123`, merge `aed7bf1`).

### Hallazgo C — Logout/revocación remota con tracking activo
CERRAR SESIÓN (o revocación remota de sesión) con el servicio nativo vivo deja
el servicio corriendo en segundo plano (no hay cleanup en unmount). Riesgo
pendiente de validación; evaluar por separado si se reproduce.
