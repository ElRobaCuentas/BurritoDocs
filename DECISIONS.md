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

**Estado:** Aceptada.

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
- En **T2.1** se añadió un tercer nivel: **gating a nivel de
  servidor** mediante Firebase Security Rules con RBAC. La regla
  `root.child('usuarios').child(auth.uid).child('rol').val() === 'admin'`
  protege los nodos `/choferes`, `/buses` y la escritura en
  `/asignaciones`, reemplazando el UID de administrador hardcodeado
  que se usaba anteriormente.

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
  en RTDB. La UserApp mitiga esto clasificando ubicaciones con
  timestamp antiguo como "stopped" (burritoLocationStore).
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
(secciones 3 y 7), TAREAS.txt (T3.1, T4.4).

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

## Referencias

| Documento | Relación |
|-----------|----------|
| `PROJECT_CONTEXT.md` | Visión general y propósito del sistema. |
| `ARCHITECTURE.md` | Flujo de datos, ciclo de vida y componentes. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos impactada por estas decisiones. |
| `TROUBLESHOOTING.md` | Problemas derivados de estas decisiones. |
| `ROADMAP.md` | Fases donde se ejecutarán las ADR planificadas. |
| `BUGS_RESUELTOS/` | Bugs resueltos que originaron decisiones arquitectónicas. |
| `ReviewNotes.md` | Notas de revisión futura sobre decisiones arquitectónicas. |
| BurritoUserApp/README.md | Setup y funcionalidad de UserApp. |
| BurritoDriverApp/README.md | Setup y funcionalidad de DriverApp. |
| BurritoUserApp/AGENTS.md | Reglas de trabajo para IA en UserApp. |
| BurritoDriverApp/AGENTS.md | Reglas de trabajo para IA en DriverApp. |
