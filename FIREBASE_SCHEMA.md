# Firebase Schema — El Burrito

## 1. Visión General

Firebase Realtime Database es el bus de datos compartido del dominio. No hay
backend intermedio ni base SQL. Firebase Auth mantiene las identidades y las
apps pueden conservar estado local; RTDB no es el único almacenamiento del
ecosistema.

- **Proyecto documentado**: `burritounmsm`
- **URL de RTDB documentada**: `https://burritounmsm-default-rtdb.firebaseio.com`
- **SDK**: `@react-native-firebase/database` v23.8.x en User/Driver;
  Firebase JS SDK v12.x en AdminWeb.

Los nodos son independientes. Las relaciones se resuelven por claves
compartidas (DNI, placa, pushId, uid). Este archivo describe el contrato de
datos y permisos; no certifica por sí solo las reglas desplegadas.

Al verificar permisos, distinguir tres niveles de evidencia: el contrato que
usa el código cliente, una captura local de reglas (si existe) y las reglas
efectivamente desplegadas. Una captura local no prueba los permisos actuales
del servidor. Sin evidencia del despliegue, indicar `[NO VERIFICABLE]` para
los permisos efectivos, sin confundirlos con el contrato documentado.

## 2. Convenciones de Datos

### Timestamps, fechas y turnos

| Uso | Representación |
|-----|----------------|
| Tracking, creación de registros, `ultimaConexion`, feedback | Unix epoch en milisegundos (entero) |
| `fecha` de asignación | String `YYYY-MM-DD`, día calendario `America/Lima` |
| `turnos[].inicio` / `fin` | Entero en minutos desde medianoche (0–1439) |

Los clientes usan `Date.now()` al intentar publicar. `ServerValue.TIMESTAMP`
se usa en perfiles y feedback. El `timestamp` de tracking es el de la
publicación, no necesariamente el instante del fix GPS. No se usan strings
ISO ni fechas localizadas dentro de RTDB.

### Coordenadas y magnitudes

`latitude` y `longitude` son números flotantes sin truncamiento. `speed` se
expresa en **m/s** y `heading` en grados. Los valores ausentes o inválidos se
normalizan en la recepción según las reglas I06 descritas en §3.

### Booleanos de estado

| Campo | Nodos | Significado |
|-------|-------|-------------|
| `activo` | Administración (`choferes`, `buses`, `asignaciones`) | Registro habilitado/vigente. No revoca permisos ni detiene tracking. |
| `isActive` | Tracking (`ubicacion_buses`) | Último estado publicado por la DriverApp. `true` no certifica que el bus siga transmitiendo. |

### Claves primarias

| Nodo | Clave | Ejemplo |
|------|-------|---------|
| `/ubicacion_buses/{placa}` | Placa | `ABC-123` |
| `/choferes/{dni}` | DNI | `12345678` |
| `/choferes_uids/{uid}` | Auth UID | `1u7VkXlh0ZaqE2ysbf024i2QXPo1` |
| `/buses/{placa}` | Placa | `ABC-123` |
| `/asignaciones/{pushId}` | Push ID | `-Nx9aBcDeFgHiJkLmNoP` |
| `/usuarios/{uid}` | Auth UID | `abc123def456uid` |
| `/comentarios/{pushId}` | Push ID | `-Ny8GhIjKlMnOpQrStUv` |
| `/administradores/{auth.uid}` | Auth UID | `jDhveoruZpgX5ZksfrWd5qcrDcp1` |

El nodo legacy `/ubicacion_burrito` dejó de usarse en T3.1 y fue eliminado de
las reglas el 23/08/2026. El dato residual en RTDB no es verificable desde el
cliente; su borrado en Consola quedó pendiente.

## 3. Nodos de Tracking

### `/ubicacion_buses/{placa}`

Nodo único de tracking multi-bus. DriverApp escribe el estado; AdminWeb crea
un marcador al dar de alta un bus; UserApp escucha la colección completa y
mantiene un `Record<string, BurritoLocation>` indexado por placa.

```json
{
  "ubicacion_buses": {
    "ABC-123": {
      "latitude": -12.056,
      "longitude": -77.084,
      "heading": 180,
      "speed": 4.2,
      "timestamp": 1718000000000,
      "isActive": true
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `latitude` | number | Latitud GPS, rango válido −90 a 90 |
| `longitude` | number | Longitud GPS, rango válido −180 a 180 |
| `heading` | number | Rumbo en grados; la recepción usa `0` si falta |
| `speed` | number | Velocidad en m/s; la recepción usa `0` si falta |
| `timestamp` | number | Unix ms del intento de publicación (GPS o heartbeat) |
| `isActive` | boolean | Último estado publicado; `false` indica detención explícita |

**Publicación y ciclo de vida.** `updateBusLocation()` actualiza coordenadas,
rumbo, velocidad, `timestamp` e `isActive: true`. El GPS publica cada ~3 s y
un heartbeat cada 8 s reutiliza la última ubicación conocida si el GPS no
emitió. `stopBusService()` escribe únicamente `isActive: false`: no borra
coordenadas ni renueva `timestamp`. Al crear un bus, AdminWeb escribe
`{ "isActive": false }` como marcador de existencia.

No existe `onDisconnect`. Una terminación anómala puede dejar `isActive: true`.
UserApp además oculta el bus cuando el `timestamp` alcanza el umbral de
frescura (30 s en la implementación actual). Ninguna de estas señales demuestra
que el proceso de tracking siga vivo.

**Recepción I06 (cliente, no reglas RTDB).** UserApp valida el snapshot
completo antes de actualizar el store:

- Rechaza tipos incorrectos, coordenadas fuera de rango, `timestamp` no
  entero o `<= 0` y timestamps futuros más allá de `receivedAt + 300 000 ms`.
  El límite exacto es aceptado.
- Acepta campos ausentes/null. I06-A normaliza latitud/longitud a `0` y
  `timestamp` a `Date.now()`; es una limitación aceptada desde el 30/10.
  `heading` y `speed` no se validan y usan `0` por defecto.
- Si `isActive: false`, no exige coordenadas válidas. MapService asigna
  `coordinatesValid: false` como metadato local; el store consume ese metadato
  y, si había una posición previa, conserva las coordenadas anteriores.
- El filtro temporal acepta una transición activo→inactivo con el mismo
  `timestamp`; fuera de ese caso rechaza timestamps anteriores o iguales.
  Un dato rechazado no renueva la posición ni la caducidad previa.
- I06-B no sincroniza relojes ni corrige adelantos dentro del margen
  (ADR-030). El commit `011a4b6` está integrado en `User/main` y sincronizado
  con `origin/main`. La validación del APK actualizado no está acreditada;
  esta referencia solo acredita la integración Git, no pruebas físicas.

## 4. Nodos de Administración

AdminWeb es el único productor de estos nodos. DriverApp lee su perfil y su
asignación; UserApp no los consulta. Las escrituras descritas son operaciones
de cliente: las reglas y sus límites se detallan en §9.

### `/choferes/{dni}`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `nombre` | string | Nombre del conductor |
| `apellidos` | string | Apellidos |
| `activo` | boolean | Registro habilitado; no bloquea tracking ni permisos |
| `uid` | string | UID de Auth; puede faltar en registros anteriores al 23/08/2026 |
| `creadoEn` | number | Unix ms de creación |
| `creadoPor` | string | UID del admin creador |

`createChofer()` crea la cuenta Auth `${dni}@conductor.com` (contraseña = DNI)
con una app temporal e `inMemoryPersistence`, escribe el perfil y por último
el índice uid→DNI. **No es una transacción multi-ruta**: no hay rollback
conjunto. La escritura del índice es la última y es fail-closed: si falla, el
conductor existe pero no puede publicar tracking hasta repararlo.

Lectura: AdminWeb escucha la colección; el conductor indexado puede leer solo
su propio `/choferes/{dni}`. No existe `deleteChofer`; desactivar solo cambia
`activo`.

### `/choferes_uids/{uid}`

Índice inverso `{ "<authUid>": "<dni>" }`, por ejemplo
`"1u7VkXlh0ZaqE2ysbf024i2QXPo1": "12345678"`. Permite a las reglas verificar
que un UID es conductor sin exponer el catálogo.
`.read: false`; solo las reglas lo consultan internamente. Se escribió por
backfill para conductores preexistentes y lo escribe `createChofer()`.

**La existencia en el índice significa “conductor registrado”, no “conductor
activo”.** `toggleChoferStatus()` no elimina el índice; por tanto desactivar a
un conductor no le revoca la autorización de escritura en tracking.

### `/buses/{placa}`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `activo` | boolean | Bus habilitado; no borra tracking ni asignaciones |
| `creadoEn` | number | Unix ms de creación |
| `creadoPor` | string | UID del admin creador |

La placa se normaliza a mayúsculas. `createBus()` escribe primero el bus y
después el marcador `/ubicacion_buses/{placa}` con `isActive: false`; no es
atómico. `deleteBus()` ejecuta dos borrados RTDB separados. La protección
contra borrar un bus con asignación activa/futura está en la pantalla
(`hasActiveOrFutureBusAssignment`), no en las reglas ni dentro de `deleteBus()`.

### `/asignaciones/{pushId}`

Relación diaria conductor↔bus. AdminWeb escucha la colección y filtra por
fecha; crea, cancela y edita turnos. DriverApp solo lee. La lectura del
conductor está limitada a su propio `choferId` mediante consulta indexada.
El alta exige una sesión activa en el servicio; `updateTurnos()` no repite esa
comprobación. La autorización real de escritura la impone la regla admin.

```json
{
  "asignaciones": {
    "-Nx9aBcDeFgHiJkLmNoP": {
      "choferId": "12345678",
      "busId": "ABC-123",
      "fecha": "2025-06-09",
      "activo": true,
      "turnos": [
        { "inicio": 420, "fin": 480 },
        { "inicio": 1140, "fin": 1260 }
      ],
      "horaInicio": 420,
      "horaFin": 480,
      "createdAt": 1776948195000,
      "createdBy": "jDhveoruZpgX5ZksfrWd5qcrDcp1"
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `choferId` | string | DNI; coincide con `/choferes` |
| `busId` | string | Placa; coincide con `/buses` |
| `fecha` | string | `YYYY-MM-DD` en `America/Lima`; alta solo hoy o futuro |
| `activo` | boolean | Asignación vigente; cancelar solo lo pone en `false` |
| `turnos` | array | Fuente de verdad; intervalos `{ inicio, fin }` en minutos |
| `horaInicio` | number | Legacy: espejo de `turnos[0].inicio` |
| `horaFin` | number | Legacy: espejo de `turnos[0].fin` |
| `createdAt` | number | Unix ms de creación; se muestra en la UI |
| `createdBy` | string | UID del admin creador (coincide con `/administradores`); auditoría, no autorización |

**Turnos.** Mínimo uno; `inicio < fin`; sin cruzar medianoche; sin
solapamientos; ordenados; consecutivos o separados permitidos. Editar turnos
actualiza `turnos`, `horaInicio` y `horaFin`, y solo se permite en asignaciones
existentes y activas. Los turnos son informativos y no inician ni detienen el
tracking.

**Exclusividad.** `createAsignacion()` lee la colección y rechaza si el mismo
conductor ya tiene una asignación activa para la fecha. Es una validación
previa del servicio, **no una garantía atómica ni una restricción de las
reglas**: dos altas concurrentes podrían duplicar. No valida que el bus esté
libre; varios conductores pueden compartir bus. `hasActiveOrFutureBusAssignment`
solo protege el borrado de buses desde la pantalla.

**Compatibilidad.** Registros anteriores al multi-turno pueden carecer de
`turnos`; `getTurnos()` usa `horaInicio`/`horaFin` como un único turno si
forman un intervalo válido. Pueden faltar `createdAt`/`createdBy`; la UI
muestra `createdAt` condicionalmente y hoy no renderiza `createdBy`. No hay
migración masiva.

**I08.** DriverApp valida al montar y al pulsar INICIAR con
`orderByChild('choferId').equalTo(dni)` y la fecha Lima del momento. Una
respuesta ausente, cancelada, duplicada, de otro conductor o inválida bloquea
el inicio. Si la consulta falla y `.info/connected` indica desconexión, puede
continuar con un respaldo **en memoria** del mismo conductor/día; no persiste
ni cubre cambios administrativos offline (ADR-028). `.info/connected` es una
ruta especial del SDK, no un nodo de datos ni una confirmación de escritura.

### `/administradores/{auth.uid}`

Fuente de verdad de autorización administrativa. La existencia de la clave
define el rol; **no se lee ningún campo `rol`**. `createAdmin()` crea la cuenta
Auth `${dni}@admin.com` con el mismo mecanismo de app temporal y luego el
nodo. El primer administrador se crea manualmente en Consola (bootstrap).
`deleteAdmin()` borra solo el nodo RTDB; la cuenta Auth se elimina manualmente.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `dni` | string | DNI del administrador |
| `nombres` | string | Nombres del administrador |
| `apellidos` | string | Apellidos del administrador |
| `correo` | string | Email de Auth (`${dni}@admin.com`) |
| `creadoEn` | number | Unix ms de creación |
| `creadoPor` | string \| null | UID del admin creador; `null` en bootstrap por Consola |

Se aceptan formatos legacy para el valor del nodo: `true` o un objeto con
`nombre`/`dni`. `creadoPor` puede ser `null` en el bootstrap por Consola. La
autorización solo comprueba existencia. Un administrador existente puede crear
otros; no hay validación de campos hijos en las reglas.

## 5. Perfiles de Estudiantes

### `/usuarios/{uid}`

Perfil de la app estudiantil, separado de la identidad de Firebase Auth. Se
crea durante el registro o al completar el perfil tras Google Sign-In.
`SignUpScreen` y `AvatarPickerScreen` escriben el perfil; `SignInScreen` y
`SignUpScreen` actualizan `ultimaConexion`. La lectura es puntual desde esas
pantallas.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `nombre` | string | Nombre del estudiante |
| `avatar` | string | Facultad: `ingeniero`, `salud`, `economista`, `humanidades` |
| `email` | string | Email de Auth |
| `rol` | string | Siempre `estudiante`; valor `admin` deprecado |
| `ultimaConexion` | number | `ServerValue.TIMESTAMP` del último login |

`ultimaConexion` es la marca del último inicio de sesión, no presencia
continua. El cambio de avatar desde el drawer es una preferencia local del
store; no implica una escritura RTDB. El campo `rol` fue deprecado como
autorización porque el dueño puede escribir su propio nodo.

## 6. Nodos de Feedback

### `/comentarios/{pushId}`

`MapService.sendFeedback()` crea un hijo nuevo; ninguna pantalla lo lee. El
servicio exige que `uid` coincida con `auth.uid`. Campos: `username`, `avatar`,
`rating`, `mensaje`, `uid`, `email` y `timestamp` (`ServerValue.TIMESTAMP`).

**Contrato I04.** La colección no admite escritura directa; solo creación de
un hijo inexistente con `uid` string igual a `auth.uid` y `rating` numérico
entero entre 1 y 5. No se puede actualizar ni borrar, tampoco desde admin.
`username`, `avatar`, `mensaje` y `email`, si existen, deben ser strings;
`timestamp`, si existe, debe ser numérico positivo. No se impone lista
cerrada de campos, límites de longitud, moderación ni rate limiting. La
lectura permanece denegada.

## 7. Consistencia, Compatibilidad y Límites

### Funcionalidades no implementadas

- `/recorridos` no tiene productores ni consumidores en el código inspeccionado.
  Depende de geofencing/control de turnos; no debe tratarse como nodo activo.
- I09 (restringir tracking al bus asignado) está postergada en `ROADMAP.md` §3.
- `/ubicacion_burrito` es legacy: fuera de reglas y pendiente de limpieza en
  Consola. No se usa para tracking.

### Compatibilidad

| Caso | Tratamiento actual |
|------|--------------------|
| Conductor anterior sin `uid` | Requiere backfill manual del índice para publicar tracking |
| Administrador legacy (`true` u objeto) | Válido para autorización por existencia |
| Asignación sin `turnos` | Se lee como un turno desde `horaInicio`/`horaFin` |
| `turnos` como objeto legacy | Se normaliza a lista; si no hay turnos válidos, se usa el espejo |
| Asignación sin `createdAt`/`createdBy` | La UI tolera su ausencia; no hay migración masiva |
| Perfil local persistido | Puede sobrevivir offline; no sustituye la fuente RTDB |

### Operaciones parciales y concurrencia

- Alta de conductor: Auth → perfil → índice. Si falla después de Auth puede
  quedar una cuenta huérfana; si falla el índice, el conductor no publica.
- Alta de admin: Auth → nodo RTDB. Si falla el nodo, la cuenta Auth queda
  huérfana y debe limpiarse en Consola.
- Alta de bus: nodo `/buses` → marcador de tracking. Borrado: dos `remove()`
  separados. No hay transacción.
- `deleteAdmin()` no borra la cuenta Auth. `toggleChoferStatus()` y
  `toggleBusStatus()` solo cambian `activo`; no cancelan asignaciones,
  no borran índices y no detienen GPS.
- La exclusividad conductor/fecha se comprueba antes de escribir. No hay
  transacción ni índice único; las reglas no validan campos hijos de
  `/asignaciones`. La concurrencia puede producir duplicados.
- `createdBy` es auditoría, no seguridad: un cliente con permiso podría
  escribir un UID ajeno si las reglas lo admitieran.

### Persistencia y estado offline

- DriverApp mantiene la persistencia de disco desactivada (ADR-003). Las
  escrituras se entregan al SDK y pueden quedar pendientes en memoria; no se
  encola un reintento propio. El `timestamp` se toma al intentar publicar y
  el heartbeat puede reutilizar coordenadas anteriores.
- UserApp mantiene el tracking en memoria y persiste el perfil local con
  Zustand. Ese perfil no refleja cambios remotos.
- El respaldo de asignación de DriverApp es en memoria, por conductor/día, y
  no cubre cambios administrativos hechos durante la desconexión (ADR-028).

## 8. Índices Requeridos

| Path | Campo índice | Consulta |
|------|--------------|----------|
| `/asignaciones` | `choferId` | DriverApp: `orderByChild('choferId').equalTo(dni)` |

```json
{
  "rules": {
    "asignaciones": {
      ".indexOn": "choferId"
    }
  }
}
```

Un índice ausente no impide por sí solo una consulta de cliente, pero degrada
el rendimiento y produce advertencia del SDK. La denegación por permisos se
debe a la regla basada en consulta, no al índice: el conductor solo puede
leer si `orderByChild` y `equalTo` coinciden con su DNI indexado. La candidata
local declara `.indexOn: ["choferId"]`; su publicación no está certificada por
esta auditoría.

## 9. Reglas de Seguridad y Mínimo Privilegio

### Niveles de evidencia

| Nivel | Fuente | Alcance |
|-------|--------|---------|
| **a. Contrato de datos** | Código de las tres apps | Rutas, campos, operaciones y validaciones de cliente |
| **b. Candidata local** | `../FirebaseLab/rules/i03-i04.rules.json` | Reglas de laboratorio; no es un espejo versionado de producción |
| **c. Permisos efectivos** | Firebase Console | Fuente autoritativa; no inspeccionada en esta auditoría |

La candidata local tiene SHA-256
`0BFAC5C2D900F6900B3E975D6CEFA7A34DBC38DCF6929A4D5C4FD2BA283359C2`.
FirebaseLab está fuera de un repositorio Git. `../FirebaseLab/firebase.json`
apunta inicialmente a `database.rules.json`; la candidata completa es un
archivo separado que debe seleccionarse explícitamente en las pruebas. Por tanto, **no se puede afirmar que esa candidata sea la
política activa**.

Corte histórico I03/I04 (03/11/2026): el usuario informó la publicación y
validó operaciones reales; se habían aprobado 72/72 pruebas de reglas y la
integración B2 en emuladores. Es evidencia histórica, no una lectura live de
Firebase en esta fase. No hay volcado versionado de las reglas desplegadas.

### Autorización administrativa

Predicado central:

```
root.child('administradores').child(auth.uid).exists()
```

La existencia de la clave concede acceso. El modelo anterior por
`/usuarios/{uid}/rol` fue descartado porque el dueño podía autoasignarse
`admin`. El primer administrador se crea en Consola; después, un admin
existente puede crear otros desde AdminWeb.

### Autorización de conductores

Predicado de tracking:

```
root.child('choferes_uids').child(auth.uid).exists()
```

La regla de escritura de `/ubicacion_buses` admite conductores indexados o
administradores. `choferes_uids` solo lo escribe un admin, así que un usuario
no puede autoindexarse. El índice se escribe al final de `createChofer()`;
si falla, la postura es fail-closed. La existencia en el índice no comprueba
que el conductor esté activo ni asignado al bus.

### Permisos por nodo

| Nodo | `.read` | `.write` | Notas |
|------|---------|----------|-------|
| `/usuarios/{uid}` | Solo dueño | Solo dueño | Perfil personal |
| `/comentarios` | Denegada | Colección denegada; creación de hijo propio con rating 1–5 | I04; sin editar ni borrar |
| `/administradores` | Admin existente | Admin existente | Primer admin por Consola; `PERMISSION_DENIED` se trata como sin rol |
| `/asignaciones` | Admin; conductor indexado solo con consulta `choferId` propia | Solo admin | Sin listado general para conductor; `.indexOn: choferId` |
| `/ubicacion_buses` | `true` | Conductor indexado o admin | Lectura pública; escritura no ligada a asignación |
| `/choferes` | Colección solo admin; conductor indexado solo su DNI | Solo admin | El índice se consulta internamente |
| `/choferes_uids` | `false` | Solo admin | Índice uid→DNI; sin lectura de cliente |
| `/buses` | Solo admin | Solo admin | Gestión exclusiva de administradores |

### Límites verificados

1. **Tracking no ligado a asignación.** Cualquier conductor indexado o admin
   puede escribir cualquier placa. Restringirlo a la asignación del día es
   I09, postergada; no se agrega como tarea nueva.
2. **Lectura pública de ubicaciones.** Cualquiera con la URL puede leer
   `/ubicacion_buses`.
3. **Feedback abierto.** No hay rate limiting, moderación, límite de longitud
   ni lista cerrada de campos.
4. **Desactivación sin cascada.** `activo: false` no cancela asignaciones, no
   detiene tracking, no borra `/choferes_uids` y no revoca permisos.
5. **Asignaciones no atómicas.** La exclusividad se valida antes de escribir;
   no hay transacción ni restricción de unicidad en reglas.
6. **Administración parcial.** `deleteAdmin()` deja la cuenta Auth; crear un
   admin o conductor puede dejar cuentas huérfanas si falla una escritura
   posterior.
7. **Auditoría limitada.** `createdBy` no es seguridad y la pantalla actual no
   lo muestra.
8. **Reglas locales no certificadas.** La candidata de FirebaseLab no prueba
   el estado desplegado; cambios remotos requieren nueva comprobación en
   Console.

### Antecedente S2

La captura local retirada documentó diferencias en `/choferes`, el nodo legacy
y `/administradores`. I03/I04, ADR-023 y ADR-025 resolvieron o alinearon esos
puntos. No se conserva como contrato vigente; sirve solo como antecedente.

## 10. Referencias Cruzadas

| Documento | Relación |
|-----------|----------|
| `ARCHITECTURE.md` | Propósito, flujo de datos, ciclo de tracking y topología |
| `TROUBLESHOOTING.md` | Incidentes de persistencia, paths y tracking |
| `DECISIONS.md` | ADR-003, ADR-019, ADR-023 a ADR-028 y ADR-030 |
| `../AdminWeb/README.md` | Panel, auth admin y convenciones de correo |
| `../Driver/README.md` | Setup, permisos Android y tracking |
| `../User/README.md` | Setup, entorno y lectura de tracking |
| `ROADMAP.md` | Fases y tareas post-MVP; I09 es una referencia, no un requisito actual |
| `BUGS_RESUELTOS/` | Historial de bugs de RTDB y tracking |
| `../FirebaseLab/rules/i03-i04.rules.json` | Candidata local; no certifica producción |

Firebase Console es la fuente autoritativa de las reglas efectivas. No existe
un volcado versionado de producción en este repositorio. Un cambio de path o
de contrato debe sincronizar `ARCHITECTURE.md` y los READMEs/AGENTS de las
apps afectadas.
