# Review Notes

## PROJECT_CONTEXT

### [ ] Revisar compatibilidad Android
Fecha: 09/06/2026

Motivo:
Actualmente se documenta Android 14.
Cuando se migre de targetSdk o aparezca Android 15/16,
verificar si el texto sigue siendo válido.

Estado:
Pendiente de revisión futura.

---

### [ ] Eliminar referencias a Tarea 4.1

Motivo:
Actualmente PROJECT_CONTEXT explica el listener legado.

Cuando se implemente Multi-bus Listener:

- eliminar referencia a T4.1
- actualizar Estado Actual
- actualizar Limitaciones


## ARCHITECTURE

### [ ] Revisar compatibilidad con nuevas versiones de Android

**Fecha:** 09/06/2026

**Motivo:**

La arquitectura documenta el uso de un **Foreground Service** para el envío continuo de coordenadas GPS, incluyendo los requisitos incorporados para Android 14 (`foregroundServiceType="location"` y permisos asociados).

En futuras migraciones de `targetSdkVersion` o cuando Google publique nuevas versiones de Android (15, 16, etc.), será necesario verificar que la arquitectura siga describiendo correctamente el comportamiento real de la aplicación.

No realizar cambios únicamente porque exista una nueva versión de Android. Solo actualizar esta documentación si:

* Android introduce nuevos requisitos para Foreground Services.
* Es necesario modificar el código para mantener la compatibilidad.
* Se incorpora una nueva API relevante para el proyecto.

**Estado:**
Pendiente de revisión cuando exista una migración importante del SDK de Android.

---

### [ ] Actualizar arquitectura cuando se implemente el Multi-bus Listener

**Motivo:**

Actualmente la arquitectura describe el flujo de tracking existente.

La aplicación de estudiantes aún utiliza el flujo legado de seguimiento, mientras que la migración al listener multi-bus forma parte del roadmap del proyecto.

Una vez implementada esa funcionalidad será necesario revisar toda la documentación arquitectónica para reflejar el nuevo flujo de datos.

**Cuando se complete esta funcionalidad:**

* Actualizar los diagramas de arquitectura.
* Modificar el flujo de tracking descrito.
* Eliminar cualquier referencia al flujo legado.
* Documentar la arquitectura definitiva del seguimiento multi-bus.

**Estado:**
Pendiente hasta finalizar la implementación del Multi-bus Listener.

---

### [ ] Incorporar la arquitectura de Geofencing cuando sea implementada

**Motivo:**

Actualmente la documentación describe únicamente el flujo de transmisión de coordenadas.

Las funcionalidades relacionadas con geofencing, cierre automático de recorridos y lógica espacial aún no forman parte de la arquitectura implementada.

Cuando dichas características entren en producción, este documento deberá ampliarse para explicar cómo interactúan con el resto del sistema.

**Cuando se implemente:**

* Documentar el flujo completo del geofencing.
* Explicar la integración con el tracking.
* Incorporar los nuevos componentes involucrados.
* Actualizar los diagramas correspondientes.

**Estado:**
Pendiente de implementación futura.

---

### [ ] Revisar los diagramas de arquitectura cuando cambie el flujo del sistema

**Motivo:**

Los diagramas representan la arquitectura real del proyecto.

Cada vez que se incorpore un componente importante (por ejemplo nuevos servicios, módulos, sincronización adicional, backend propio o nuevos consumidores de datos), deberán actualizarse para mantener la documentación consistente con el código.

**Ejemplos de cambios que requieren revisión:**

* Nuevos servicios externos.
* Nuevos módulos principales.
* Cambios importantes en Firebase.
* Incorporación de backend.
* Cambios en la comunicación entre DriverApp y UserApp.
**Estado:**
Revisión continua durante la evolución
del proyecto.


## AGENTS (BurritoUserApp)

### [ ] Revisar reglas críticas de AGENTS.md

Fecha: 13/06/2026

Motivo:

AGENTS.md contiene 13 reglas críticas (sección 5)
que reflejan la arquitectura actual del proyecto.

Si cambia la arquitectura (nuevo store, nuevo
patrón de tracking, backend propio, nueva forma de
autenticación), las reglas de trabajo para la IA
deben actualizarse para mantener el documento
como fuente de verdad.

Ejemplos de cambios que dispararían revisión:
- Nuevo store de Zustand
- Migración de RTDB a backend propio
- Nuevo método de autenticación
- Agregar soporte iOS a DriverApp
- Reactivar snapToRoute o geofencing

Estado:
Revisión continua — AGENTS evoluciona con la
arquitectura.

---

### [ ] Revisar checklist pre-entrega de AGENTS

Fecha: 13/06/2026

Motivo:

El checklist de sección 9 en AGENTS.md lista 10
verificaciones obligatorias antes de finalizar
una tarea.

Si se agregan nuevas herramientas al pipeline
(nuevo linter, formateador, test runner, typecheck
automatizado), actualizar el checklist.

Estado:
Revisión futura cuando cambie el pipeline.

---

### [ ] Revisar tabla de actualización de documentación

Fecha: 13/06/2026

Motivo:

La tabla en sección 8 de AGENTS.md mapea cambios
del código a documentos que deben actualizarse.

Si se agregan o eliminan documentos del sistema
documental, actualizar esta tabla.

Estado:
Revisión futura cuando cambie el sistema
documental.

---

### [ ] Revisar arquitectura mínima necesaria

Fecha: 13/06/2026

Motivo:

La sección "Arquitectura mínima necesaria"
resume los componentes esenciales que una IA
debe comprender antes de modificar código.

Actualmente incluye:

- Entry point
- Auth gate
- Navegación
- Zustand
- Firebase RTDB

Si la arquitectura incorpora nuevos módulos
principales o cambia el flujo de datos,
actualizar esta sección.

Ejemplos:
- Backend propio
- Dashboard compartido
- Nuevo sistema de estado
- Migración de Firebase

Estado:
Revisión continua.


## DECISIONS

### [ ] Revisar ADR de persistence

Fecha: 09/06/2026

Motivo:

DECISIONS.md documenta la desactivación de
persistencia local (ADR-003).

Si en el futuro se decide reactivarla por
nuevos requisitos offline, actualizar la
decisión y sus consecuencias.

Estado:
Revisión futura.

---

### [ ] Revisar ADR de comunicación mediante RTDB

Fecha: 09/06/2026

Motivo:

Actualmente ambas aplicaciones se comunican
únicamente mediante Firebase Realtime Database
(ADR-007).

Si en el futuro se incorpora un backend propio,
mensajería, colas de eventos o una API
intermedia, actualizar esta decisión
arquitectónica.

Estado:
Pendiente de revisión cuando exista Backend
Propio (ADR-014).

---

### [ ] Revisar ADR de foreground service

Fecha: 09/06/2026

Motivo:

DECISIONS.md documenta el Foreground Service
como requisito obligatorio (ADR-008).

Si Android 15/16 introduce nuevas restricciones
para foreground services, actualizar las
consecuencias de la decisión.

Estado:
Revisión futura.

---

### [ ] Revisar ADR de Android-only DriverApp

Fecha: 09/06/2026

Motivo:

La DriverApp está implementada únicamente para
Android (ADR-012).

Si se decide soportar iOS para la app del
conductor, actualizar la decisión y las
alternativas consideradas.

Estado:
Revisión futura.

---

### [ ] Incorporar ADR de geofencing

Fecha: 09/06/2026

Motivo:

El geofencing está documentado como ADR
planificada (ADR-013).

Cuando se implemente oficialmente (Fase 2 del
roadmap), mover de planificada a aceptada y
completar el detalle de implementación.

Estado:
Pendiente de implementación.

---

### [ ] Revisar ADR de backend propio y dashboard web

Fecha: 09/06/2026

Motivo:

El backend propio (ADR-014) y el dashboard web
(ADR-015) están documentados como ADR
planificadas en la visión de largo plazo.

Cuando alguna de estas iniciativas pase a
desarrollo activo, mover a ADR aceptadas y
actualizar el detalle de implementación.

Estado:
Pendiente de revisión futura.


## FIREBASE_SCHEMA

### [x] Eliminar `/ubicacion_burrito`

Fecha: 09/06/2026

Motivo:

Actualmente la UserApp consume el nodo legado
`/ubicacion_burrito`.

Cuando se implemente el Multi-bus Listener, toda la
lectura deberá realizarse desde
`/ubicacion_buses/{placa}`.

Cuando ocurra:

- eliminar la sección `/ubicacion_burrito`
- actualizar "Nodos de Tracking"
- actualizar "Estado del Esquema"
- revisar referencias en:
  - ARCHITECTURE.md
  - UserApp README
  - DriverApp README
  - AGENTS.md

Estado:
Resuelto el 23/08/2026. El bloque de la regla
`.read: true` fue eliminado de las reglas publicadas
en Firebase Console (el nodo queda denegado por
defecto para todo cliente) y del espejo
`docs/firebase-rules.json`. FIREBASE_SCHEMA.md
actualizado ("Nodos de Tracking", "Estado del
Esquema", tabla de permisos §9). Las referencias en
READMEs/AGENTS.md ya habían sido corregidas durante
la migración de paths (22/08). Pendiente menor:
borrar el dato residual `/ubicacion_burrito` en RTDB
desde Consola (ver item nuevo en este archivo).

---

### [ ] Documentar `/recorridos`

Fecha: 09/06/2026

Motivo:

El nodo `/recorridos`
todavía no existe.

Será creado cuando se implemente:

- Control de Turnos
- Geofencing
- Inicio/Cierre automático

Cuando exista:

- agregar payload
- agregar relaciones
- agregar índices
- agregar referencias cruzadas

Estado:
Pendiente.

---

### [ ] Revisar esquema de Tracking

Fecha: 09/06/2026

Motivo:

Actualmente existen dos productores de ubicación.

DriverApp

↓

/ubicacion_buses

Simulador

↓

/ubicacion_burrito

Cuando se complete el Multi-bus Listener
deberá existir un único flujo oficial.

Estado:
Pendiente.

---

### [ ] Revisar `/usuarios`

Fecha: 09/06/2026

Motivo:

Actualmente el nodo almacena:

- nombre
- avatar
- email
- rol
- ultimaConexion

Si cambia el modelo de autenticación,
los campos deberán revisarse.

Estado:
Revisión futura.

---

### [ ] Revisar índices RTDB

Fecha: 09/06/2026

Motivo:

Actualmente solo se documenta el índice:

/asignaciones
↓
choferId

Cada nueva consulta con
orderByChild()
debe reflejarse aquí.

Estado:
Revisión continua.

---

### [ ] Verificar URL del proyecto Firebase

Fecha: 09/06/2026

Motivo:

Actualmente se documenta:

https://burritounmsm-default-rtdb.firebaseio.com

Si en el futuro el proyecto Firebase cambia
(entorno de producción, staging o migración),
actualizar esta referencia.

Estado:
Revisión futura.

## ROADMAP

### [ ] Revisar el roadmap al finalizar cada fase

**Fecha:** 09/06/2026

**Motivo:**

El roadmap refleja la planificación activa del proyecto.

Cada vez que una fase finalice, será necesario:

- marcarla como completada.
- actualizar el estado general del documento.
- promover la siguiente fase como prioridad actual.
- verificar que las dependencias entre fases continúen siendo válidas.

**Estado:**
Revisión continua durante el desarrollo del proyecto.

---

### [ ] Revisar la Visión de Largo Plazo

**Fecha:** 09/06/2026

**Motivo:**

La sección "Visión de Largo Plazo" almacena iniciativas que actualmente
no forman parte del roadmap activo.

Cuando alguna de esas iniciativas pase a desarrollo real (por ejemplo,
Dashboard Web, Backend propio, API o Analytics), deberá:

- eliminarse de la Visión de Largo Plazo.
- incorporarse a una nueva fase del roadmap.
- actualizar las prioridades del proyecto.

**Estado:**
Pendiente de revisión futura.


## TROUBLESHOOTING

### [ ] Revisar compatibilidad Android 14

Fecha: 09/06/2026

Motivo:

TROUBLESHOOTING documenta el caso
"InvalidForegroundServiceTypeException"
con requisitos para Android 14.

Cuando aparezcan Android 15/16 con nuevos
requisitos para foreground services,
o cuando se migre targetSdkVersion,
verificar que el diagnóstico y la solución
sigan siendo válidos.

Estado:
Revisión futura.

---

### [ ] Revisar Foreground Service cuando cambie Android

Fecha: 09/06/2026

Motivo:

Si Android modifica nuevamente las
restricciones para Foreground Services,
actualizar síntomas, diagnóstico y solución
del caso correspondiente en
TROUBLESHOOTING.md.

Estado:
Revisión futura.

---

### [ ] Revisar caso de missing index

Fecha: 09/06/2026

Motivo:

Se documenta el índice `choferId` sobre
`/asignaciones`. Cuando se agreguen nuevas
consultas con `orderByChild()` en el código,
actualizar esta sección de troubleshooting.

Estado:
Revisión continua.

---

### [ ] Revisar caso de persistence

Fecha: 09/06/2026

Motivo:

TROUBLESHOOTING documenta que persistence
está desactivada intencionalmente.

Si en el futuro alguien propone reactivarla
(por ejemplo, para soporte offline),
actualizar el caso con la nueva decisión.

Estado:
Revisión futura.

---

### [ ] Revisar caso de ubicación congelada

Fecha: 09/06/2026

Motivo:

La solución actual para stale locations
es la clasificación por timestamp en
burritoLocationStore.

Cuando se implemente Timeout Check
automático (Fase 4 del roadmap),
actualizar el diagnóstico y la solución.

Estado:
Pendiente de implementación.

---

### [ ] Documentar nuevos incidentes reales

Fecha: 09/06/2026

Motivo:

Cada vez que aparezca un problema relevante
durante el desarrollo o en producción,
agregar un nuevo caso siguiendo la plantilla
estándar:

- Problema
- Síntoma
- Causa
- Diagnóstico
- Solución
- Prevención

Estado:
Revisión continua.

---

### [ ] Revisar Troubleshooting al completar nuevas fases del roadmap

Fecha: 09/06/2026

Motivo:

Cada nueva funcionalidad importante del
sistema puede introducir nuevos escenarios
de fallo.

Cuando se implemente una fase importante
(Multi-bus, Geofencing, Dashboard, Backend,
etc.), revisar si aparecen nuevos casos que
deban documentarse en TROUBLESHOOTING.md.

Estado:
Revisión continua durante la evolución
del proyecto.


## AGENTS (BurritoDriverApp)

### [ ] Revisar reglas críticas de DriverApp

Fecha: 13/06/2026

Motivo:

El AGENTS de DriverApp contiene 15 reglas
críticas (3 generales y 12 específicas) que reflejan la
arquitectura actual del tracking.

Si cambia la arquitectura (nuevo patrón de
tracking, migración a backend propio, soporte
iOS, nuevo sistema de permisos), las reglas
de trabajo para la IA deben actualizarse.

Ejemplos:
- Agregar geofencing
- Reemplazar react-native-background-actions
- Agregar soporte iOS
- Cambiar el intervalo GPS

Estado:
Revisión continua — AGENTS evoluciona con la
arquitectura.

---

### [ ] Revisar ciclo de vida del tracking

Fecha: 13/06/2026

Motivo:

La sección 5 resume el ciclo de vida del tracking y referencia ARCHITECTURE.md para los detalles de implementación: login → fetchAssignment →
requestAllPermissions → BackgroundJob →
locationTask → stopProcess → signOut.

Si se implementa geofencing, cierre automático
de vueltas (Fase 2/4 del roadmap) o control
de turnos, el ciclo de vida cambiará y debe
actualizarse esta sección.

Estado:
Pendiente de implementación (ver ROADMAP.md).

---

### [ ] Revisar permisos Android

Fecha: 13/06/2026

Motivo:

La sección 5 documenta la secuencia de 3
permisos Android y la declaración del
Foreground Service en 3 lugares.

Si Android 15/16 introduce nuevas restricciones
para foreground services de tipo location,
verificar que los 3 lugares sigan siendo
válidos.

Estado:
Revisión futura cuando cambie Android.

---

### [ ] Revisar checklist pre-entrega

Fecha: 13/06/2026

Motivo:

El checklist de sección 9 incluye
verificaciones obligatorias antes de finalizar
una tarea.

Si se agregan nuevas herramientas al pipeline
(typecheck automatizado, nuevo linter, test
runner), actualizar el checklist y los
comandos de sección 3.

Estado:
Revisión futura cuando cambie el pipeline.

---

### [ ] Revisar estrategia de testing de DriverApp

Fecha: 13/06/2026

Motivo:

Actualmente la aplicación conserva un test
heredado (__tests__/App.test.tsx) que no
representa el estado real del proyecto
(importa ../App que ya no existe).

Cuando se defina una estrategia de pruebas
para DriverApp, actualizar:
- AGENTS.md (sección 3 comandos, sección 9
  checklist)
- README.md
- TROUBLESHOOTING.md (si aplica)

Estado:
Pendiente de definición.

---

## BUGS_RESUELTOS

### [ ] Revisar y actualizar BUGS_RESUELTOS/ al corregir un bug nuevo

**Fecha:** 14/06/2026

**Motivo:**
BUGS_RESUELTOS/ es el historial técnico de bugs del proyecto.
Cada vez que se corrija un bug durante el desarrollo, debe
documentarse en el archivo correspondiente:
- `driverapp.md` para bugs de DriverApp
- `userapp.md` para bugs de UserApp
- `ambasApp.md` para bugs que afectan a ambas

**Estado:**
Pendiente — aplicar al corregir el próximo bug.

---

## BurritoUserApp/README.md

### [ ] Revisar README de UserApp cuando cambie el setup o stack

**Fecha:** 14/06/2026

**Motivo:**
El README de UserApp documenta stack, setup, scripts y
documentación relacionada.

Requiere revisión si:
- Cambia la versión de React Native, Firebase, Zustand o Mapbox.
- Se agregan o eliminan variables de entorno.
- Cambia la estructura del código.
- Se modifica el flujo de tracking descrito.

**Estado:**
Revisión futura cuando cambie el stack o setup.

---

## BurritoDriverApp/README.md

### [ ] Revisar README de DriverApp cuando cambie el setup o stack

**Fecha:** 14/06/2026

**Motivo:**
El README de DriverApp documenta stack, requisitos de
plataforma, setup y flujo de tracking.

Requiere revisión si:
- Cambia la versión de React Native, Firebase o dependencias.
- Se modifican los requisitos de Android (targetSdk, permisos).
- Cambia la estructura del código o entry point.
- Se agrega soporte iOS.

**Estado:**
Revisión futura cuando cambie el stack o plataforma.

---

## Firebase Console (RTDB y Auth)

### [ ] Borrar el dato residual de `/ubicacion_burrito` en RTDB

**Fecha:** 23/08/2026

**Motivo:**
Las reglas publicadas el 23/08/2026 ya no incluyen
el nodo `/ubicacion_burrito`, por lo que queda
denegado por defecto e inaccesible desde clientes.
El dato físico aún existe en la base.

**Cuando ocurra:**
- Firebase Console > Realtime Database > borrar
  `/ubicacion_burrito`
- Verificar que ninguna herramienta interna lo lea

**Estado:**
Pendiente (limpieza; sin impacto funcional).

### [ ] Redesplegar AdminWeb a Firebase Hosting tras el merge de `feat/security/rtdb-uid-authorization`

**Fecha:** 23/08/2026

**Motivo:**
`https://burritounmsm.web.app/` sigue sirviendo la
build vieja SIN la escritura del índice
`/choferes_uids`. Todo chofer creado desde el deploy
viejo nacerá sin índice y no podrá transmitir
ubicación (postura fail-closed, ADR-023).

**Cuando ocurra:**
- Mergear la rama a main
- `npm run build && firebase deploy --only hosting`

**Estado:**
Pendiente (bloquea creación de choferes vía web
desplegada con tracking operativo).

### [ ] Limpieza post-MVP de cuentas Auth de prueba antiguas

**Fecha:** 23/08/2026

**Motivo:**
Quedan cuentas Auth inertes de pruebas tempranas:
DNI 12345678, DNI 87654321 y un posible registro
temprano bajo `chofer@burritounmsm.com`. No tienen
entrada en `/choferes_uids` ni asignaciones activas,
por lo que son inofensivas bajo las reglas nuevas.

**Cuando ocurra:**
- Confirmar con el usuario qué cuentas conservar
- Borrar usuarios desde Console > Authentication
- Borrar sus registros si existen en `/choferes`

**Estado:**
Pendiente (post-MVP).

### [ ] Eliminar registro legacy `com.burritouserapp` de Firebase Console

**Fecha:** 23/08/2026

**Motivo:**
UserApp se registró originalmente como `com.burritouserapp`
(app_id `b9b8149c`). El 19/03/2026 se renombró
applicationId a `com.sythor.burritoapp` pero el
google-services.json fue editado a mano en vez de
re-registrar. Hoy UserApp ya usa el registro canónico
`com.sythor.burritoapp` (`7beb07f9`) con sus 3 SHA-1s.
El registro `b9b8149c` (`com.burritouserapp`) queda como
huérfano legacy.

**Cuando ocurra:**
- Firebase Console > ⚙ > Tus apps > eliminar
  `com.burritouserapp` (app_id `b9b8149c`)
- Verificar que ninguna app o dependencia lo use
  (DriverApp usa su propio registro `810e95c0`)
- Confirmar en Analytics que los streams de UserApp
  apuntan a `com.sythor.burritoapp`

**Estado:**
Pendiente (post-MVP; sin impacto funcional).

---

## Keystore

### Documentar estado del keystore y contraseña histórica

**Fecha:** 23/08/2026

**Motivo:**
La contraseña del keystore release (`burrito123`) aparece
en el historial de git (commit anterior a `a2235a9`,
27/07/2026, en `android/gradle.properties`). El archivo
keystore NO está en Git (`.gitignore` lo cubre).

**Estado actual:**
- `my-release-key.keystore`: existe localmente en ambos
  repos (gitignored)
- `keystore.properties`: existe localmente (gitignored)
- SHA-1 debug: `5E:8F:16:06:2E:A3:CD:2C:4A:0D:54:78:76:BA:A6:F3:8C:AB:F6:25`
- SHA-1 release User: `44:19:4D:4E:A5:22:80:B1:DB:0C:5E:AD:46:44:F2:F2:B6:08:CB:58`
- SHA-1 release Driver: `A3:32:3F:70:FF:34:EE:0A:46:C7:EC:95:3D:9B:ED:87:50:03:E3:29`

**Riesgo:**
Bajo. Sin el archivo keystore, la contraseña es inútil.
Estos keystores solo se usan para builds de desarrollo.

**Cuándo rotar:**
- Solo si se publica en Play Store sin App Signing de Google
- O si hay sospecha de compromiso del archivo local

**Acción:** documentar. No requiere cambios de código ni Console.

**Estado:**
Documentado (23/08/2026). Re-verificado en auditoría completa del
ecosistema (23/08/2026): sigue siendo riesgo bajo y aceptado para el
MVP. Rotación obligatoria antes de publicar en Play Store.

---

## Seguridad

### Contraseña de choferes = su DNI (riesgo aceptado del MVP)

**Fecha:** 23/08/2026

**Motivo:**
La auditoría completa del ecosistema confirmó que `AdminService.createChofer()`
(ambos repos, Driver y AdminWeb) crea cuentas Firebase Auth con
`email = ${dni}@burritodriver.com` y `password = dni`. Un DNI es un
identificador público de 8 dígitos (~27 bits de entropía): cualquiera que
conozca el DNI de un chofer puede suplantarlo.

**Decisión (23/08/2026):**
Riesgo **aceptado** para este MVP:
- Las cuentas actuales son de prueba, no conductores reales.
- Convertir ahora a password temporal + cambio obligatorio introduce lógica
  nueva y riesgo de romper el flujo a días de la presentación.
- La autorización real de escritura NO depende de la contraseña sino de las
  reglas RTDB vía `/choferes_uids` (ADR-023), que siguen intactas.

**Condición dura:**
Antes de registrar conductores reales en producción, este esquema DEBE
reemplazarse (password aleatoria temporal + cambio obligatorio en primer
login, o flujo passwordless). No publicar a producción con DNI como contraseña.

**Estado:**
Documentado como riesgo aceptado. Post-MVP obligatorio.

---

### Resumen de la auditoría completa del ecosistema (23/08/2026)

Auditoría de seguridad sobre DriverApp, UserApp y AdminWeb enmarcada en la
definición funcional del MVP (conductor opera jornada completa; estudiante ve
ubicación en tiempo real).

**Resueltos antes del cierre del MVP:**
| ID | Hallazgo | Commit(s) |
|----|----------|-----------|
| W1 | Panel debug visible en release → `__DEV__` | User `19b118a` |
| W2 | `CALIB_LOG_ENABLED=true` → `false` | User `c133518` |
| W4 | ProGuard/R8 deshabilitado → habilitado + probado en dispositivo | Driver `490349a`, User `d444f37` |
| W5 | Servicio tracking `exported="true"` → `"false"` + verificado | Driver `02ef1da` |
| C7 | Reglas RTDB versionadas — ya resuelto desde S3 (`docs/firebase-rules.json`) | verificado |

**Documentados como riesgo aceptado:**
- C1: contraseña = DNI (ver sección anterior).
- C3: keystore `burrito123` (ver sección Keystore; rotar pre-Play Store).
- "EN MOVIMIENTO" con bus quieto en interiores: deriva GPS interior vence los
  umbrales experimentales de C4.6 (ADR-020, calibración pendiente). Al aire
  libre se atenúa. Post-MVP.

**Post-MVP (no bloquean, listos en backlog):**
C4 (createChofer no atómico), C5 (google-services.json multi-app),
W3 (console.error), W6 (rate limiting login), W7 (admin check client-side —
la seguridad real vive en las reglas RTDB), W8 (AvatarPicker `.set()`),
W9 (ForgotPassword falso éxito), W10 (validación email), W11 (spam feedback),
W12 (toggleChoferStatus no limpia `choferes_uids` — **revisar antes de
producción**: hoy `activo:false` NO revoca la autorización de escritura),
W13 (createBus sobreescribe ubicación), W14 (security headers Hosting),
W15 (race condition secondary app).

**Estado:**
Cierre de seguridad del MVP completo (23/08/2026). Arquitectura congelada;
sigue prueba E2E integral.

### Semántica de "desactivar" (Opción A) + auditoría mínima en asignaciones (24/08/2026)

**Desactivar conductor/bus = elegibilidad futura, no corte inmediato.**
`activo: false` en `/choferes/{dni}` o `/buses/{placa}` solo saca al
elemento de los selectores de nueva asignación. NO revoca la autorización
GPS ni interrumpe un recorrido activo: el tracking continúa hasta DETENER
TODO. Decisión de producto deliberada (ADR-024): desactivar es decisión
administrativa; cancelar asignación es la acción operativa. La revocación
en caliente queda como post-MVP ("Forzar detención" escribiendo
`isActive: false` en `/ubicacion_buses/{placa}`). Esto formaliza el
comportamiento anotado en W12: para el MVP es semántica deseada, no bug.
Pendiente post-MVP real de W12: decidir si desactivar debe limpiar
`choferes_uids` cuando se implemente la revocación.

**Auditoría mínima en `/asignaciones`.** Nuevos campos `createdAt`
(epoch ms en RTDB; formato legible solo en UI con
`toLocaleString('es-PE', { timeZone: 'America/Lima' })`) y `createdBy`
(UID real del admin; sin sesión la creación falla — sin fallback
`'unknown'`). Implementado en AdminWeb y DriverApp (rama
`feat/asignacion-auditoria`). Registros previos sin estos campos siguen
siendo válidos. Choferes/buses NO reciben auditoría en el MVP (no cambia
decisiones operativas hoy). Detalle completo: ADR-024,
FIREBASE_SCHEMA.md §4.

**Congelamiento:** tras este cambio se cierra la fase funcional del MVP;
siguiente hito: E2E de campo.
