# Firebase Schema — El Burrito

## 1. Visión General

El ecosistema utiliza **Firebase Realtime Database** como único bus de
datos y almacenamiento persistente. No existe backend intermedio, base
SQL ni otra fuente de verdad.

- **Proyecto Firebase**: `burritounmsm`
- **URL de RTDB**: `https://burritounmsm-default-rtdb.firebaseio.com`
- **SDK**: `@react-native-firebase/database` v23.8.x en ambas apps

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
- `activo`: presente en nodos de administración (choferes, buses,
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
> seguridad el 23/08/2026; su dato residual en RTDB está pendiente de
> borrado manual desde Consola (ver `ReviewNotes.md`).

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
- **Escritura**: `admin_service.createChofer()` (desde DriverApp).
- **Lectura**: `admin_service.subscribeToChoferes()` (listener continuo
  desde DriverApp, panel admin).
- **Estructura**:

```json
{
  "choferes": {
    "12345678": {
      "nombre": "Juan",
      "apellidos": "Pérez López",
      "activo": true
    }
  }
}
```

**Creación**: además de escribir en RTDB, `createChofer()` crea una
cuenta de Firebase Auth con email `${dni}@burritodriver.com` y
contraseña igual al DNI. Usa una instancia secundaria de Auth
(`SecondaryApp`) para no cerrar la sesión del admin.

### `/choferes_uids/{uid}` (nuevo, 23/08/2026)

- **Propósito**: índice inverso uid→DNI que permite a las Security Rules
  verificar que un usuario autenticado es un conductor registrado, sin
  exponer el catálogo `/choferes` (que tiene `.read` amplio). Es la base
  de la autorización de escritura sobre `/ubicacion_buses` (ADR-023).
- **Escritura**: `admin_service.createChofer()` (DriverApp y AdminWeb)
  escribe `{uid: dni}` inmediatamente después de crear la cuenta Auth y
  el registro en `/choferes`. Postura **fail-closed**: es la última
  escritura de la transacción; si falla, el chofer existe pero no podrá
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
- **Escritura**: `admin_service.createBus()` (desde DriverApp).
- **Lectura**: `admin_service.subscribeToBuses()` (listener continuo
  desde DriverApp, panel admin).
- **Estructura**:

```json
{
  "buses": {
    "ABC-123": {
      "modelo": "Mercedes-Benz",
      "marca": "OF-1721",
      "anio": "2020",
      "activo": true
    }
  }
}
```

**Efecto secundario**: al crear un bus, se inicializa automáticamente
`/ubicacion_buses/{placa}` con `{ isActive: false }`.

### `/asignaciones/{pushId}`

- **Propósito**: relación temporal entre un conductor y un bus para un
  turno diario. Clave generada por `push()` de Firebase.
- **Escritura**: `admin_service.createAsignacion()` (desde AdminWeb y
  DriverApp). Requiere administrador autenticado: si no existe sesión,
  la operación falla sin escribir en RTDB.
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
| `fecha` | string | Fecha del turno en formato `YYYY-MM-DD`. Es independiente de `createdAt`: el admin puede preparar asignaciones de un día futuro |
| `activo` | boolean | `true` mientras el turno está vigente |
| `createdAt` | number | Unix timestamp en milisegundos del momento de creación (`Date.now()`). Formato legible solo en la UI (`toLocaleString('es-PE', { timeZone: 'America/Lima' })`) |
| `createdBy` | string | UID de Firebase Auth del administrador que creó la asignación. Coincide con una clave de `/administradores`. Es auditoría, no seguridad: la autorización real sigue siendo `auth.uid` en `/administradores` vía reglas RTDB |

**Validación de exclusividad**: al crear una asignación, el servicio
verifica que ni el chofer ni el bus tengan otra asignación activa para
la misma fecha.

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
- **Escritura**: exclusivamente manual desde Firebase Console. Las reglas
  de RTDB tienen `.write: false` sobre este nodo.
- **Lectura**: DriverApp — `admin_check.ts` (`existeAdministrador(uid)`)
  para el enrutador post-login. RTDB Security Rules — para el predicado
  admin en todos los nodos administrativos.
- **Estructura**:

```json
{
  "administradores": {
    "abcDef123456UidAdmin": true
  }
}
```

El valor mínimo es `true` (RTDB no permite valores nulos).
Opcionalmente puede almacenar `{ nombre, dni }` para trazabilidad:

```json
{
  "administradores": {
    "abcDef123456UidAdmin": {
      "nombre": "Admin UNMSM",
      "dni": "77777777"
    }
  }
}
```

**Propiedad del nodo**: la DriverApp (admin) escribe/lee los nodos de
gestión (`/choferes`, `/buses`, `/asignaciones`). La UserApp ya no
consulta ni escribe en estos nodos. El administrador accede al panel
exclusivamente desde la DriverApp.

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
fuente única de verdad es `/administradores/{auth.uid}`, que es inmutable
desde el cliente (`.write: false`).

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
la migración del panel admin a DriverApp (FASE 1-5) y el endurecimiento
de las reglas de seguridad por UID (23/08/2026, ADR-023). El nodo
heredado `/ubicacion_burrito` fue eliminado de las reglas; todo el
tracking fluye a través de `/ubicacion_buses/{placa}`.

### Nodos activos (flujo actual)

| Nodo | Estado | Propietario | Evolución prevista |
|------|--------|-------------|-------------------|
| `/asignaciones` | Definitivo | DriverApp (admin) | Se mantiene. |
| `/choferes` | Definitivo | DriverApp (admin) | Se mantiene. |
| `/choferes_uids` | Definitivo | DriverApp/AdminWeb (createChofer) | Índice uid→DNI para reglas (ADR-023). Backfill manual para preexistentes. |
| `/buses` | Definitivo | DriverApp (admin) | Se mantiene. |
| `/administradores` | Definitivo | Firebase Console | Se mantiene. Solo escritura manual. |
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
Console. No existen archivos de reglas en el repositorio. Si la regla
no está definida, la consulta desde DriverApp fallará en producción con
un error de permiso o advertencia de cliente no optimizado.

## 9. Reglas de Seguridad y Mínimo Privilegio (RBAC)

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
por el cliente porque `/administradores` tiene `.write: false`.

**Fundamento**: el modelo por `rol` en `/usuarios/{uid}` fue descartado
porque la regla de escritura del propio usuario sobre su nodo permitía
auto-asignarse `rol: 'admin'`. `/administradores` es una fuente de
verdad inmutable desde el cliente, poblada exclusivamente por consola
de Firebase.

### Modelo de autorización conductor (23/08/2026, ADR-023)

La escritura sobre `/ubicacion_buses` exige que el usuario autenticado
sea un conductor registrado. El predicado central de autorización
conductor es:

```
root.child('choferes_uids').child(auth.uid).exists()
```

Esta expresión verifica que el UID del usuario autenticado exista como
clave en `/choferes_uids/`. El índice se escribe automáticamente al
crear un chofer (`createChofer()`) y tiene `.write` exclusivo de admin,
por lo que un usuario cualquiera no puede auto-indexarse. La regla
completa de escritura del tracking admite conductores indexados **o**
admins:

```json
".write": "auth != null && (root.child('administradores').child(auth.uid).exists() || root.child('choferes_uids').child(auth.uid).exists())"
```

**Alternativa descartada**: autorizar por dominio de email
(`auth.token.email.matches('.*@burritodriver.com')`). Se rechazó porque
`SignUpScreen.tsx:86` acepta emails arbitrarios sin verificación
(`emailVerified` no se consulta ni exige en ningún repo), por lo que un
atacante podía registrarse con cualquier correo `@burritodriver.com`.

**Postura fail-closed**: `/choferes_uids/{uid}` se escribe al final de
`createChofer()`; si esa escritura falla, el chofer existe pero no
puede publicar ubicación hasta reparar el índice manualmente desde
Consola.

### Tabla de permisos por nodo

| Nodo | .read | .write | Notas |
|------|-------|--------|-------|
| `/usuarios/{uid}` | Solo dueño | Solo dueño | Perfil personal |
| `/comentarios` | `false` | `auth != null` | Solo escritura, lectura desde Consola |
| `/administradores` | `auth != null` | `false` | Solo lectura. Escritura exclusiva desde Consola |
| `/asignaciones` | `auth != null` | Solo admin | DriverApp necesita lectura, admin escribe |
| `/ubicacion_buses` | `true` | Conductor indexado o admin | Lectura pública; escritura vía `/choferes_uids` o `/administradores` (ADR-023) |
| `/choferes` | `auth != null` | Solo admin | Catálogo legible por usuarios autenticados; escritura exclusiva de admin |
| `/choferes_uids` | `false` | Solo admin | Índice uid→DNI para reglas (ADR-023); sin lectura de cliente |
| `/buses` | Solo admin | Solo admin | Gestión exclusiva de administradores |

### Discrepancias detectadas (S2) y resolución

Comparación entre las reglas desplegadas en Firebase (`firebase-rules.json`)
y la tabla de permisos anterior:

1. **`/choferes` (.read)**: la tabla documentaba `Solo admin`, pero las
   reglas desplegadas permiten `.read: "auth != null"` — cualquier
   usuario autenticado puede leer el directorio de conductores. El
   `.write` sí coincide (solo admin). **Estado: divergencia vigente**;
   la tabla fue actualizada el 23/08/2026 para reflejar lo desplegado.
   Restringir la lectura a solo-admin queda como decisión futura si el
   catálogo deja de ser necesario para clientes.
2. **Nodo legacy `/ubicacion_burrito`**: estaba presente en las reglas
   con `.read: true`. **RESUELTO el 23/08/2026**: el bloque fue
   eliminado de las reglas publicadas (denegado por defecto) y retirado
   del espejo `docs/firebase-rules.json`. Borrado del dato residual en
   RTDB pendiente de Consola.

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

1. **Escritura en `/ubicacion_buses`**: cualquier usuario autenticado
   puede escribir coordenadas en cualquier bus. No hay verificación de
   que el conductor esté asignado al bus. La mejora futura (restringir
   escritura solo al conductor asignado) está planificada.
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
| `firebase-rules.json` | Exportación exacta de las reglas RTDB desplegadas (captura de S2). |
| `TROUBLESHOOTING.md` | Incidentes relacionados con Firebase: persistence, path mismatch, regresiones, async initializeApp. |
| `DECISIONS.md` | ADR sobre persistence desactivada, arquitectura serverless, migración admin, fix async initializeApp. |
| BurritoDriverApp/README.md | Setup de Firebase, permisos Android y google-services.json. |
| BurritoUserApp/README.md | Setup de Firebase, .env y google-services.json. |
| `ROADMAP.md` | Fases y prioridades para la evolución del esquema. |
| `ReviewNotes.md` | Notas de revisión futura para cambios en el esquema. |
| `BUGS_RESUELTOS/` | Historial de bugs relacionados con Firebase RTDB. |
