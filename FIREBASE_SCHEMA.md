# Firebase Schema — El Burrito

## 1. Visión General

El ecosistema utiliza **Firebase Realtime Database** como único bus de
datos y almacenamiento persistente. No existe backend intermedio, base
SQL ni otra fuente de verdad.

- **Proyecto Firebase**: `burritounmsm`
- **URL de RTDB**: `https://burritounmsm-default-rtdb.firebaseio.com`
- **SDK**: `@react-native-firebase/database` v23.8.x en las apps móviles
  (Driver y User); Firebase JS SDK v12.x en AdminWeb (web).

La base de datos se organiza en nodos independientes, cada uno con un
propósito específico dentro del sistema. No hay relaciones foráneas
formales; las asociaciones entre nodos se resuelven mediante claves
compartidas (DNI, placa, pushId, uid).

## 2. Convenciones de Datos

### Timestamps

Todos los registros de tiempo se almacenan como **Unix epoch en
milisegundos** (enteros). No se usan strings ISO, fechas locales ni
zonas horarias.

| Origen | Método |
|--------|--------|
| JavaScript (DriverApp) | `Date.now()` |
| UserApp (server-side) | `database.ServerValue.TIMESTAMP` |

Las fechas de asignación (turno diario) se almacenan como string
`YYYY-MM-DD` para permitir filtros client-side.

### Coordenadas

Latitud y longitud se almacenan como **números flotantes** con precisión
directa desde el hardware GPS. Sin truncamiento, redondeo ni
transformación.

### Booleanos de estado

- `isActive`: presente en nodos de tracking. Indica si el bus está
  transmitiendo activamente.
- `activo`: presente en nodos de administración (conductores, buses,
  asignaciones). Indica si el registro está vigente.

### Claves primarias

| Nodo | Tipo de clave | Ejemplo |
|------|--------------|---------|
| `/ubicacion_buses/{placa}` | Placa del bus | `ABC-123` |
| `/choferes/{dni}` | DNI del conductor | `12345678` |
| `/choferes_uids/{uid}` | Auth UID de Firebase (índice uid→DNI) | `abc123...` |
| `/buses/{placa}` | Placa del bus | `ABC-123` |
| `/asignaciones/{pushId}` | Push ID generado por Firebase | `-Nx9...` |
| `/usuarios/{uid}` | Auth UID de Firebase | `abc123...` |
| `/comentarios/{pushId}` | Push ID generado por Firebase | `-Ny8...` |
| `/administradores/{auth.uid}` | Auth UID de Firebase | `def456...` |

> Nota: el nodo legacy `/ubicacion_burrito` fue eliminado de las reglas de
> seguridad el 23/08/2026 (queda denegado por defecto). El borrado del dato
> residual en RTDB está **pendiente en Consola** — no es verificable desde
> el cliente.

## 3. Nodos de Tracking

Actualmente el sistema utiliza un único nodo de tracking consolidado
(`/ubicacion_buses/{placa}`). El nodo heredado `/ubicacion_burrito` dejó
de usarse desde la migración multi-bus (T3.1) y el 23/08/2026 fue
eliminado de las reglas de seguridad (queda denegado por defecto;
borrado del dato pendiente en Consola).

### `/ubicacion_buses/{placa}` (activo)

- **Propósito**: nodo único de tracking. Contiene la posición en vivo
  de todos los buses activos, cada uno identificado por su placa.
- **Escritura**: DriverApp mediante `firebase_service.ts`
  (`updateBusLocation`/`stopBusService`). También se inicializa desde
  `admin_service.ts` al crear un bus nuevo.
- **Lectura**: UserApp, mediante listener continuo en
  `map_service.ts` (`subscribeToBusLocations()`) sobre el path
  `/ubicacion_buses`, que alimenta `burritoLocationStore` con un
  `Record<string, BurritoLocation>` indexado por placa (T3.1).
- **Estructura** (payload de tracking):

```json
{
  "ubicacion_buses": {
    "ABC-123": {
      "latitude": -12.056,
      "longitude": -77.084,
      "heading": 180,
      "speed": 15,
      "timestamp": 1718000000000,
      "isActive": true
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `latitude` | number | Coordenada del GPS |
| `longitude` | number | Coordenada del GPS |
| `heading` | number | Rumbo en grados |
| `speed` | number | Velocidad |
| `timestamp` | number | Unix epoch ms — última actualización **publicada** por la DriverApp. Puede provenir del fix GPS o del heartbeat de presencia (T4.1). No debe leerse como "el momento exacto del GPS". |
| `isActive` | boolean | `true` durante tracking, `false` al detener |

**Inicialización desde admin**: al crear un bus en el panel
administrativo, se escribe `{ isActive: false }` en
`/ubicacion_buses/{placa}` como marcador de existencia.

```json
{
  "ubicacion_buses": {
    "ABC-123": {
      "isActive": false
    }
  }
}
```

## 4. Nodos de Administración

### `/choferes/{dni}`

- **Propósito**: catálogo de conductores. Cada clave es el DNI del
  conductor.
- **Escritura**: `admin_service.createChofer()` (desde AdminWeb).
- **Lectura**: `admin_service.subscribeToChoferes()` (listener continuo
  desde AdminWeb, panel admin) y DriverApp, que lee `/choferes/{dni}`
  (`SendCoordinates.tsx`) para mostrar el nombre del conductor logueado.
- **Estructura**:

```json
{
  "choferes": {
    "12345678": {
      "nombre": "Juan",
      "apellidos": "Pérez López",
      "activo": true,
      "uid": "1u7VkXlh0ZaqE2ysbf024i2QXPo1",
      "creadoEn": 1776948195000,
      "creadoPor": "jDhveoruZpgX5ZksfrWd5qcrDcp1"
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `nombre` | string | Nombre del conductor |
| `apellidos` | string | Apellidos del conductor |
| `activo` | boolean | `true` mientras el registro está vigente |
| `uid` | string | UID de Firebase Auth del conductor; base del índice `/choferes_uids`. Puede faltar en conductores creados antes del 23/08/2026 |
| `creadoEn` | number | Unix epoch ms de creación (`Date.now()`) |
| `creadoPor` | string | UID del administrador que creó el registro |

**Creación**: además de escribir en RTDB, `createChofer()` crea una
cuenta de Firebase Auth con email `${dni}@conductor.com` y contraseña
igual al DNI. Usa una **app temporal por operación** con
`inMemoryPersistence` para no cerrar la sesión del admin (ver ADR-011 y
ADR-025). El mismo mecanismo usa `createAdmin()` con `{dni}@admin.com`.

### `/choferes_uids/{uid}` (nuevo, 23/08/2026)

- **Propósito**: índice inverso uid→DNI que permite a las Security Rules
  verificar que un usuario autenticado es un conductor registrado, sin
  exponer el catálogo `/choferes` (que tiene `.read` amplio). Es la base
  de la autorización de escritura sobre `/ubicacion_buses` (ADR-023).
- **Escritura**: `admin_service.createChofer()` (AdminWeb)
  escribe `{uid: dni}` inmediatamente después de crear la cuenta Auth y
  el registro en `/choferes`. Postura **fail-closed**: es la última
  escritura de la transacción; si falla, el conductor existe pero no podrá
  publicar ubicación hasta reparar el índice manualmente.
- **Lectura**: ninguna desde cliente (`.read: false`). Solo las reglas
  lo consultan internamente vía `root.child('choferes_uids')`.
- **Población inicial**: backfill manual desde Consola para conductores
  preexistentes (hecho el 23/08/2026).
- **Estructura**:

```json
{
  "choferes_uids": {
    "1u7VkXlh0ZaqE2ysbf024i2QXPo1": "11223344"
  }
}
```

### `/buses/{placa}`

- **Propósito**: catálogo de buses. Cada clave es la placa del vehículo
  en mayúsculas.
- **Escritura**: `admin_service.createBus()` (desde AdminWeb).
- **Lectura**: `admin_service.subscribeToBuses()` (listener continuo
  desde AdminWeb, panel admin).
- **Estructura**:

```json
{
  "buses": {
    "ABC-123": {
      "activo": true,
      "creadoEn": 1776948195000,
      "creadoPor": "jDhveoruZpgX5ZksfrWd5qcrDcp1"
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `activo` | boolean | `true` mientras el bus está vigente |
| `creadoEn` | number | Unix epoch ms de creación (`Date.now()`) |
| `creadoPor` | string | UID del administrador que creó el registro |

**Efecto secundario**: al crear un bus, se inicializa automáticamente
`/ubicacion_buses/{placa}` con `{ isActive: false }`.

### `/asignaciones/{pushId}`

- **Propósito**: relación temporal entre un conductor y un bus para un
  turno diario. Clave generada por `push()` de Firebase.
- **Escritura**: `admin_service.createAsignacion()` (desde AdminWeb,
  panel admin). Requiere administrador autenticado: si no existe sesión,
  la operación falla sin escribir en RTDB. La DriverApp solo lee
  `/asignaciones`; no escribe.
- **Lectura**: `admin_service.subscribeToAsignacionesHoy()` (listener
  continuo) y DriverApp (consulta puntual con `orderByChild('choferId')`).
- **Estructura**:

```json
{
  "asignaciones": {
    "-Nx9aBcDeFgHiJkLmNoP": {
      "choferId": "12345678",
      "busId": "ABC-123",
      "fecha": "2025-06-09",
      "activo": true,
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
| `choferId` | string | DNI del conductor (coincide con clave en `/choferes`) |
| `busId` | string | Placa del bus (coincide con clave en `/buses`) |
| `fecha` | string | Fecha del turno en formato `YYYY-MM-DD`, calculada con `getTodayDateString()` (TZ local del navegador). El panel solo crea asignaciones para el día de hoy |
| `activo` | boolean | `true` mientras el turno está vigente |
| `horaInicio` | number | **Opcional**. Hora de inicio como minutos desde medianoche (0–1439). Se omite si el admin no la ingresa |
| `horaFin` | number | **Opcional**. Hora de fin en el mismo formato que `horaInicio` |
| `createdAt` | number | Unix timestamp en milisegundos del momento de creación (`Date.now()`). Formato legible solo en la UI (`toLocaleString('es-PE', { timeZone: 'America/Lima' })`) |
| `createdBy` | string | UID de Firebase Auth del administrador que creó la asignación. Coincide con una clave de `/administradores`. Es auditoría, no seguridad: la autorización real sigue siendo `auth.uid` en `/administradores` vía reglas RTDB |

**Validación de exclusividad**: al crear una asignación, el servicio
verifica que el **conductor** no tenga otra asignación activa para hoy.
La validación del bus (`hasActiveBusAssignment`) existe en el servicio
pero **no está conectada** al flujo de creación.

**Cancelación**: al cancelar, se establece `activo: false`. El registro
no se elimina.

**Nota**: asignaciones creadas antes de esta mejora no tienen
`createdAt` ni `createdBy`; ambos campos son opcionales y las pantallas
los renderizan condicionalmente.

### `/administradores/{auth.uid}` (nuevo)

- **Propósito**: fuente única de verdad para autorización administrativa.
  Cada clave es el UID de Firebase Auth del administrador. La mera
  existencia de la clave (`snapshot.exists()`) define que el usuario es
  administrador. **No existe ni se lee un campo `rol`**.
- **Escritura**: la gestión se realiza desde **AdminWeb** (`AdminsScreen`
  → `createAdmin()`), que crea la cuenta Auth `${dni}@admin.com` y
  escribe el nodo. El **primer administrador** debe crearse manualmente
  desde Firebase Console (bootstrap): es el único que no puede crearse
  desde el panel.
- **Lectura**: AdminWeb — `admin_check.ts` (`existeAdministrador(uid)`)
  para el auth gate post-login. RTDB Security Rules — para el predicado
  admin en todos los nodos administrativos.
- **Estructura**:

```json
{
  "administradores": {
    "abcDef123456UidAdmin": {
      "dni": "77777777",
      "nombres": "Admin",
      "apellidos": "UNMSM",
      "correo": "77777777@admin.com",
      "creadoEn": 1776948195000,
      "creadoPor": "jDhveoruZpgX5ZksfrWd5qcrDcp1"
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `dni` | string | DNI del administrador |
| `nombres` | string | Nombres del administrador |
| `apellidos` | string | Apellidos del administrador |
| `correo` | string | Email de Auth (`${dni}@admin.com`) |
| `creadoEn` | number | Unix epoch ms de creación (`Date.now()`) |
| `creadoPor` | string | UID del administrador que creó la cuenta (o `null` si se creó desde Consola) |

Se aceptan formatos **legacy** para el valor del nodo: `true` (mínimo, RTDB
no permite valores nulos) o `{ "nombre", "dni" }`. La verificación de rol
solo comprueba que la clave exista, por lo que ambos siguen siendo válidos.

**Propiedad del nodo**: AdminWeb (admin) escribe/lee los nodos de gestión
(`/choferes`, `/buses`, `/asignaciones`). La UserApp ya no consulta ni
escribe en estos nodos. La DriverApp solo lee `/asignaciones` para obtener
su turno. El administrador accede al panel exclusivamente desde AdminWeb.

## 5. Nodos de Autenticación

### `/usuarios/{uid}`

- **Propósito**: perfil de usuarios estudiantes. Creado durante el
  registro o al completar el perfil tras Google Sign-In.
- **Escritura**: `SignUpScreen.tsx` y `AvatarPickerScreen.tsx`.
- **Lectura**: `SignInScreen.tsx` y `SignUpScreen.tsx` (consulta
  puntual con `once('value')`).
- **Estructura**:

```json
{
  "usuarios": {
    "abc123def456uid": {
      "nombre": "Carlos",
      "avatar": "ingeniero",
      "email": "carlos@example.com",
      "rol": "estudiante",
      "ultimaConexion": 1718000000000
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `nombre` | string | Nombre del estudiante |
| `avatar` | string | Facultad: `ingeniero`, `salud`, `economista`, `humanidades` |
| `email` | string | Email registrado en Firebase Auth |
| `rol` | string | Siempre `estudiante`. El valor `admin` fue deprecado. |
| `ultimaConexion` | number | ServerValue.TIMESTAMP del último login |

**Nota sobre roles**: el campo `rol` en `/usuarios/{uid}` fue deprecado
como fuente de autorización administrativa por ser auto-escribible por el
propio usuario (regla `.write` del dueño sobre su propio nodo). La nueva
fuente única de verdad es `/administradores/{auth.uid}`: la existencia de
la clave define el rol admin y un usuario no puede autoasignárselo, porque
la escritura sobre ese nodo exige ser un administrador ya existente.

## 6. Nodos de Feedback

### `/comentarios/{pushId}`

- **Propósito**: almacenar opiniones y calificaciones de estudiantes
  enviadas desde el modal de feedback en el CustomDrawer.
- **Escritura**: `MapService.sendFeedback()`.
- **Lectura**: no se consume actualmente en ninguna pantalla.
- **Estructura**:

```json
{
  "comentarios": {
    "-Ny8GhIjKlMnOpQrStUv": {
      "username": "Carlos",
      "avatar": "ingeniero",
      "rating": 4,
      "mensaje": "Buen servicio",
      "uid": "abc123def456uid",
      "email": "carlos@example.com",
      "timestamp": 1718000000000
    }
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `username` | string | Nombre del estudiante |
| `avatar` | string | Facultad seleccionada |
| `rating` | number | Calificación (1–5) |
| `mensaje` | string | Texto libre |
| `uid` | string | Auth UID del remitente |
| `email` | string | Email del remitente |
| `timestamp` | number | ServerValue.TIMESTAMP (generado por Firebase) |

## 7. Estado del Esquema

El esquema se encuentra consolidado tras la migración multi-bus (T3.1),
la migración del panel admin a la aplicación web **AdminWeb** (ADR-025) y
el endurecimiento de las reglas de seguridad por UID (23/08/2026,
ADR-023). El nodo heredado `/ubicacion_burrito` fue eliminado de las
reglas; todo el tracking fluye a través de `/ubicacion_buses/{placa}`.

### Nodos activos (flujo actual)

| Nodo | Estado | Propietario | Evolución prevista |
|------|--------|-------------|-------------------|
| `/asignaciones` | Definitivo | AdminWeb (admin) | Se mantiene. |
| `/choferes` | Definitivo | AdminWeb (admin) | Se mantiene. |
| `/choferes_uids` | Definitivo | AdminWeb (createChofer) | Índice uid→DNI para reglas (ADR-023). Backfill manual para preexistentes. |
| `/buses` | Definitivo | AdminWeb (admin) | Se mantiene. |
| `/administradores` | Definitivo | AdminWeb (Admin) | Alta desde AdminWeb; el primer administrador se crea en Consola. |
| `/usuarios` | Definitivo | UserApp | Se mantiene. |
| `/comentarios` | Provisional | UserApp | Sin cambios previstos inmediatos. |
| `/ubicacion_buses` | Definitivo | DriverApp (conductor) | Nodo único de tracking. Escritura restringida a conductores indexados y admins (ADR-023). |
| `/ubicacion_burrito` | Eliminado de reglas | — | Denegado por defecto desde el 23/08/2026. Borrado del dato pendiente en Consola. |

### Nodos planificados (no existen en RTDB)

| Nodo | Dependencia |
|------|-------------|
| `/recorridos` | Requiere implementación de geofencing y control de turnos |

### Recomendaciones para cambios futuros

Cuando el esquema evolucione, los siguientes documentos deben
actualizarse en conjunto:

- **PROJECT_CONTEXT.md**: sección de estado actual y limitaciones.
- **ARCHITECTURE.md**: secciones de flujo de datos y consideraciones.
- **README.md** de cada app: paths de RTDB relevantes.
- **AGENTS.md** de cada app: paths que los asistentes deben conocer.

## 8. Índices Requeridos

Firebase Realtime Database requiere reglas de índice (`.indexOn`) para
consultas con `orderByChild`. Sin el índice apropiado, la consulta
falla o se ejecuta ineficientemente.

### Índice actualmente necesario

| Path | Campo índice | Origen de consulta |
|------|-------------|-------------------|
| `/asignaciones` | `choferId` | DriverApp: `orderByChild('choferId').equalTo(driverDni)` en `SendCoordinates.tsx` |

Regla Firebase requerida:

```json
{
  "rules": {
    "asignaciones": {
      ".indexOn": "choferId"
    }
  }
}
```

### Nota sobre la creación de índices

Los índices deben configurarse en la pestaña **Rules** de Firebase
Console, que es la **única fuente autoritativa** de las reglas (ver la
nota de niveles en §9). El repositorio **ya no mantiene un volcado local**
de reglas (ver §10). Si la regla no está definida, la
consulta desde DriverApp fallará en producción con un error de permiso o
advertencia de cliente no optimizado.

## 9. Reglas de Seguridad y Mínimo Privilegio (RBAC)

### Niveles de información sobre RTDB (no confundir)

Esta sección describe **tres niveles distintos** que no deben mezclarse:

1. **Contrato de datos (a)**: rutas y campos que las aplicaciones
   efectivamente leen/escriben. Verificable en el código de cada app.
   Es lo que documenta el resto de este archivo (§3–§6).
2. **Reglas locales (b)**: la versión de reglas que en algún momento se
   capturó en el repositorio (hoy **retirada**).
   Era una captura histórica (S2), nunca una representación garantizada de
   lo desplegado.
3. **Permisos efectivos (c)**: las reglas realmente desplegadas en la
   instancia de Firebase. **Solo verificables desde Firebase Console.**
   Marcadas como **[NO VERIFICABLE]** desde el repositorio.

Las afirmaciones de seguridad de este documento describen (a) y (b). No
certifican (c): hasta contrastar con Console, los permisos efectivos
permanecen **[NO VERIFICABLE]**.

### Modelo de autorización admin (post-migración)

El acceso a cada nodo de RTDB se controla mediante **Firebase Security
Rules** basadas en la membresía del auth.uid en el nodo
`/administradores`. El predicado central de autorización admin es:

```
root.child('administradores').child(auth.uid).exists()
```

Esta expresión verifica que el UID del usuario autenticado exista como
clave en `/administradores/`. A diferencia del modelo anterior
(`/usuarios/{uid}/rol === 'admin'`), este predicado no es auto-escribible
por el cliente: para escribir en `/administradores` hay que ser ya un
administrador existente.

**Fundamento**: el modelo por `rol` en `/usuarios/{uid}` fue descartado
porque la regla de escritura del propio usuario sobre su nodo permitía
auto-asignarse `rol: 'admin'`. `/administradores` es una fuente de
verdad que el cliente no puede autoasignarse; se puebla desde AdminWeb
(y el primer administrador desde Consola).

### Modelo de autorización conductor (23/08/2026, ADR-023)

La escritura sobre `/ubicacion_buses` exige que el usuario autenticado
sea un conductor registrado. El predicado central de autorización
conductor es:

```
root.child('choferes_uids').child(auth.uid).exists()
```

Esta expresión verifica que el UID del usuario autenticado exista como
clave en `/choferes_uids/`. El índice se escribe automáticamente al
crear un conductor (`createChofer()`) y tiene `.write` exclusivo de admin,
por lo que un usuario cualquiera no puede auto-indexarse. La regla
completa de escritura del tracking admite conductores indexados **o**
admins:

```json
".write": "auth != null && (root.child('administradores').child(auth.uid).exists() || root.child('choferes_uids').child(auth.uid).exists())"
```

**Alternativa descartada**: autorizar por dominio de email
(`auth.token.email.matches('.*@<dominio>')`). Se rechazó porque
`SignUpScreen.tsx:83` acepta emails arbitrarios sin verificación
(`emailVerified` no se consulta ni exige en ningún repo), por lo que un
atacante podía registrarse con un correo del dominio permitido.

**Postura fail-closed**: `/choferes_uids/{uid}` se escribe al final de
`createChofer()`; si esa escritura falla, el conductor existe pero no
puede publicar ubicación hasta reparar el índice manualmente desde
Consola.

### Tabla de permisos por nodo

| Nodo | .read | .write | Notas |
|------|-------|--------|-------|
| `/usuarios/{uid}` | Solo dueño | Solo dueño | Perfil personal |
| `/comentarios` | `false` | `auth != null` | Solo escritura, lectura desde Consola |
| `/administradores` | `auth != null` | Admin existente | Lectura autenticada. Escritura de un administrador ya existente; el primer admin se crea en Consola |
| `/asignaciones` | `auth != null` | Solo admin | DriverApp necesita lectura, admin escribe |
| `/ubicacion_buses` | `true` | Conductor indexado o admin | Lectura pública; escritura vía `/choferes_uids` o `/administradores` (ADR-023) |
| `/choferes` | `auth != null` | Solo admin | Catálogo legible por usuarios autenticados; escritura exclusiva de admin |
| `/choferes_uids` | `false` | Solo admin | Índice uid→DNI para reglas (ADR-023); sin lectura de cliente |
| `/buses` | Solo admin | Solo admin | Gestión exclusiva de administradores |

### Discrepancias detectadas (S2) y resolución

Comparación entre la captura local de reglas (nivel b, hoy retirada)
y la tabla de permisos anterior. Los permisos
**desplegados** (nivel c) no son verificables desde el repositorio:

1. **`/choferes` (.read)**: la tabla documentaba `Solo admin`, pero la
   captura local de reglas permite `.read: "auth != null"` — cualquier
   usuario autenticado puede leer el directorio de conductores. El
   `.write` sí coincide (solo admin). **Estado: divergencia vigente**;
   la tabla fue actualizada el 23/08/2026 para reflejar la captura.
   Restringir la lectura a solo-admin queda como decisión futura si el
   catálogo deja de ser necesario para clientes.
2. **Nodo legacy `/ubicacion_burrito`**: estaba presente en las reglas
   con `.read: true`. **RESUELTO el 23/08/2026**: el bloque fue
   eliminado de las reglas publicadas (denegado por defecto) y retirado
   del espejo de reglas local (que ya no existe). Borrado del dato residual
   en RTDB pendiente de Consola.
3. **`/administradores` (.write)**: la tabla documentaba `.write: false`
   (solo Consola), pero el permiso vigente permite que un administrador
   ya existente cree otros administradores — que es exactamente lo que
   hace AdminWeb (`createAdmin()`), con el primer administrador creado
   por Consola. **RESUELTO**: la tabla se alineó al comportamiento real
   del panel.

### Predicado admin en reglas

```json
"admin": "auth != null && root.child('administradores').child(auth.uid).exists()"
```

Este predicado reemplaza al anterior:
```
// ELIMINADO — vulnerable a auto-asignación:
"auth != null && root.child('usuarios').child(auth.uid).child('rol').val() === 'admin'"
```

### Riesgos residuales aceptados (MVP)

1. **Escritura en `/ubicacion_buses`**: tras ADR-023 (23/08/2026) la
   escritura está restringida a conductores indexados en `/choferes_uids`
   o a administradores; ya no la puede hacer "cualquier usuario
   autenticado". Sí persiste que **no** se verifica que el conductor esté
   asignado al bus del día. Restringir la escritura al conductor asignado
   sigue planificado (ver "Mejora planificada" en esta misma sección).
2. **Lectura en `/ubicacion_buses`**: es pública (`true`). Cualquier
   persona con la URL de la base de datos puede leer ubicaciones en
   vivo.
3. **Escritura en `/comentarios`**: solo se exige autenticación. No hay
   rate limiting ni validación del contenido.

### Mejora planificada (Fase 4/5)

Restringir la escritura en `/ubicacion_buses/{placa}` para que solo el
conductor asignado al bus (según `/asignaciones`) pueda escribir
coordenadas. Esto requiere una regla que cruce
`root.child('asignaciones')` con el UID o DNI del conductor
autenticado.

## 10. Referencias Cruzadas

| Documento | Relación |
|-----------|----------|
| `PROJECT_CONTEXT.md` | Visión general del sistema, propósito y limitaciones. |
| `ARCHITECTURE.md` | Flujo de datos, ciclo de vida del tracking y topología del ecosistema. |
| Reglas desplegadas | Única fuente autoritativa: Firebase Console (nivel c). No hay volcado local de reglas en el repositorio. |
| `TROUBLESHOOTING.md` | Incidentes relacionados con Firebase: persistence, path mismatch, regresiones, async initializeApp. |
| `DECISIONS.md` | ADR sobre persistence desactivada, arquitectura serverless, migración del panel admin, dominios por rol y fix async initializeApp. |
| `AdminWeb/README.md` | Setup del panel web, auth admin y convenciones de correo. |
| `Driver/README.md` | Setup de Firebase, permisos Android y google-services.json. |
| `User/README.md` | Setup de Firebase, .env y google-services.json. |
| `ROADMAP.md` | Fases y prioridades para la evolución del esquema. |
| `BUGS_RESUELTOS/` | Historial de bugs relacionados con Firebase RTDB. |
