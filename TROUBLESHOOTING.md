# Troubleshooting — El Burrito

## 1. Propósito

Guía de diagnóstico operativo del MVP. Responde a: **tengo un problema,
¿qué compruebo y qué hago?**

No es un historial de incidencias ni un changelog. El historial vivido está
en `BUGS_RESUELTOS/`; los contratos, en `ARCHITECTURE.md`,
`FIREBASE_SCHEMA.md` y `DECISIONS.md`.

## 2. Cómo usar este documento

No se lee completo. Se consulta por síntoma. Cada caso indica **qué
comprobar**, **cómo interpretarlo** y **qué hacer**.

| Si el problema es… | Empezar en |
|--------------------|------------|
| No puedo entrar o crear una cuenta | §5 Acceso y autenticación |
| No tengo asignación o no puedo iniciar | §4 Asignaciones e inicio |
| No hay GPS o el servicio se reinicia | §6 Servicio Android / §7 Tracking |
| Faltan permisos o notificaciones | §6 Foreground service |
| El bus no aparece, está congelado o desactualizado | §7 Bus y tracking |
| DETENER no confirma el cierre remoto | §7 Parada local |
| El panel AdminWeb no carga, no autoriza o falla el foco | §3 Configuración / §5 Acceso / §9 UI |
| El proyecto no compila o no arranca | §3 Configuración |
| El mapa no carga o la cámara se sale del campus | §8 Mapbox |
| La app se queda en splash, va lenta o el tema no se conserva | §9 Renderizado y tema |
| Quiero saber qué está aceptado o pendiente | §10 Limitaciones y estado |

**Ruta rápida del tracking:** servicio Android → fix GPS → intento de
publicación → confirmación RTDB → recepción en El Burrito → validación y
estado visual. Un heartbeat o un pulso no prueban por sí solos un fix GPS
reciente ni una entrega remota.

## 3. Configuración y compilación

### `google-services.json` faltante

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | La app Android falla al inicializar Firebase. |
| Qué comprobar | Que exista `android/app/google-services.json` en `User` o `Driver`. |
| Cómo interpretar | Afecta a las apps Android; no es la configuración de AdminWeb. |
| Acción | Colocar el archivo del proyecto Firebase correspondiente. Los README documentan la ubicación. |

### JDK incompatible

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Gradle falla por la versión de Java. |
| Qué comprobar | `java --version` y `JAVA_HOME`. |
| Cómo interpretar | Los wrappers usan Gradle 9.0.0, que requiere una JVM compatible (17–24). |
| Acción | Instalar un JDK compatible y apuntar `JAVA_HOME`. |

### `.env` faltante o incompleto en El Burrito

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Mapbox en blanco o Google Sign-In no disponible. |
| Qué comprobar | Que exista `.env` con `MAPBOX_PUBLIC_TOKEN` y `GOOGLE_WEB_CLIENT_ID`. |
| Cómo interpretar | Son dos variables distintas; corregir una no arregla el otro síntoma. |
| Acción | Crear/completar `.env` según el README y `env.d.ts`. |

### AdminWeb: «Configuración de Firebase incompleta»

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | AdminWeb no monta y la consola nombra variables `VITE_FIREBASE_*` faltantes. |
| Qué comprobar | `.env` contra `.env.example`; que el deploy se haya construido con esas variables. |
| Cómo interpretar | La validación es de configuración cliente, no de secretos. Un build sin variables reproduce el fallo aunque el código sea correcto. |
| Acción | Completar las variables y reconstruir. No versionar `.env`. |

## 4. Firebase, asignaciones e inicio

### «No tienes un bus asignado para hoy»

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El conductor inicia sesión, pero INICIAR informa que no hay asignación. |
| Qué comprobar | En `/asignaciones`, que exista una entrada con `choferId` igual al DNI autenticado, `fecha` igual a la fecha `America/Lima` actual y `activo: true`; que `busId` sea válido. |
| Cómo interpretar | La consulta filtra por conductor y fecha al resolver. Una fecha de ayer/mañana, una asignación inactiva, un `busId` vacío o datos inválidos bloquean el inicio. Los turnos son informativos: no arrancan ni detienen el tracking. |
| Acción | Crear o corregir la asignación en AdminWeb con la fecha y el conductor correctos. |

### La consulta de asignación falla o devuelve vacío

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | DriverApp informa error al consultar `/asignaciones`, o la consulta devuelve vacío sin motivo visible. |
| Qué comprobar | Sesión del conductor, datos del nodo y reglas RTDB. En las reglas, si existe el índice; en la consola de la app, si aparece una advertencia por índice. |
| Cómo interpretar | La falta de `.indexOn: "choferId"` afecta al **rendimiento** y puede generar una advertencia; **no deniega por sí sola** ni explica automáticamente un resultado vacío. La autorización depende de las reglas y de que la consulta cumpla el contrato. Un resultado vacío también puede deberse a datos ausentes o a la fecha resuelta. |
| Acción | Corregir primero reglas, sesión o datos. Agregar el índice solo si la advertencia o el rendimiento lo justifican. Ver índices y reglas en `FIREBASE_SCHEMA.md`. |

### `PERMISSION_DENIED` en lecturas o feedback

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Una lectura permitida falla con `PERMISSION_DENIED`, o el feedback no se puede crear, editar ni borrar. |
| Qué comprobar | Sesión y UID; path y operación exactos; que la consulta cumpla la regla; en feedback, que se cree un hijo nuevo con UID propio y rating entero 1–5. |
| Cómo interpretar | La regla distingue lectura, creación, edición y borrado. Un estudiante no debe leer `/choferes` ni `/asignaciones` completas; el conductor consulta su perfil y sus asignaciones; AdminWeb conserva acceso admin. Editar o borrar feedback ajeno se deniega. |
| Acción | Contrastar sesión, consulta y contrato antes de ampliar permisos. No usar `PERMISSION_DENIED` como evidencia de un índice faltante. Ver `FIREBASE_SCHEMA.md` §9. |

### INICIAR sin red rechaza la asignación (I08)

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Sin red, INICIAR muestra «No se pudo validar la asignación…» aunque la pantalla ya mostraba la asignación del día. |
| Qué comprobar | Si la desconexión está confirmada por `.info/connected == false`; si existe una asignación válida del mismo conductor y día en memoria; y si la consulta falló por timeout aislado. |
| Cómo interpretar | El respaldo solo se habilita con desconexión confirmada, no con cualquier timeout. Vive en memoria: no sobrevive al proceso ni detecta altas, cancelaciones o cambios administrativos hechos offline. |
| Acción | Recuperar la conexión y reintentar. Si hay PARADA LOCAL NO CONFIRMADA, resolverla antes de volver a INICIAR. Ver ADR-028 y CASO 021 de `BUGS_RESUELTOS/driverapp.md`. |

### Hay más de una asignación para el conductor y la fecha

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El inicio bloquea por asignaciones contradictorias. |
| Qué comprobar | Si existen dos o más entradas activas del mismo `choferId` para la misma `fecha`, o si alguna contradice explícitamente el conductor consultado. |
| Cómo interpretar | La validación de exclusividad es previa, no atómica: dos altas concurrentes podrían duplicar. No valida que el bus esté libre; varios conductores pueden compartir bus. |
| Acción | Cancelar o corregir duplicados en AdminWeb. Un turno relevo no debe crear otra asignación. |

## 5. Acceso y autenticación

### AdminWeb: «No autorizado» o «No se pudo verificar»

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Un usuario autenticado no entra al panel, o el panel queda comprobando el rol. |
| Qué comprobar | Si el mensaje es **«No autorizado»** o **«No se pudo verificar»**. En Firebase Auth, que la sesión corresponda al UID esperado. En RTDB, si existe `/administradores/{uid}`. |
| Cómo interpretar | `PERMISSION_DENIED` al leer el nodo se trata como **denegado**: la cuenta no es admin o la regla no permite esa lectura. Un timeout de 10 s o un fallo de red se tratan como **estado desconocido**, no como denegación. |
| Acción | Denegado: crear o corregir el administrador y volver a iniciar sesión. Error: revisar conexión y usar **Reintentar**. Nunca conceder acceso por defecto ni convertir un fallo de red en denegación. Ver `admin_check.ts` y `App.tsx`. |

El flujo vive en AdminWeb. La reproducción original en DriverApp es
histórica y no describe el flujo actual de esa app (`BUGS_RESUELTOS/`,
CASO 014).

### Login de conductor: `auth/user-not-found`

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El conductor no inicia sesión; Auth devuelve `auth/user-not-found`. |
| Qué comprobar | Que el DNI sea el correcto y que exista la cuenta `${dni}@conductor.com` en Firebase Authentication. |
| Cómo interpretar | La cuenta no existe o el identificador introducido no coincide. No confundir con contraseña incorrecta ni con un fallo de red. |
| Acción | Crear o corregir el conductor desde AdminWeb. La contraseña inicial es el DNI; cambiarla antes de producción real. |

### Error al crear un conductor

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | AdminWeb muestra «Error de autenticación» al crear un conductor. |
| Qué comprobar | Si la cuenta de Auth llegó a crearse; si existe `/choferes/{dni}` y el índice correspondiente; y el error concreto de Auth o RTDB en la consola. |
| Cómo interpretar | Auth y RTDB **no se crean de forma atómica**: primero se crea la cuenta y después se escriben los nodos. Un fallo posterior de RTDB puede mostrarse con el mismo mensaje genérico. La app temporal de Auth debe usar `inMemoryPersistence`; AdminWeb no depende de `google-services.json`. |
| Acción | Si Auth existe pero falta RTDB, corregir o reintentar la creación sin duplicar la cuenta. Si falla la app temporal, revisar la inicialización y la configuración web de Firebase. Ver ADR-011/ADR-025 y `admin_service.ts`. |

### Google Sign-In no completa el acceso

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | «Continuar con Google» no hace nada, se cancela o Firebase rechaza la credencial. |
| Qué comprobar | Que `.env` tenga `GOOGLE_WEB_CLIENT_ID`; que el cliente sea de **tipo web**; que el SHA-1 de la build instalada esté registrado en Firebase; y que se haya obtenido un ID token antes de llamar a Firebase Auth. |
| Cómo interpretar | Son causas posibles, no una causa única. Un cliente ausente o incorrecto, un SHA-1 no registrado —incluido el de release—, la selección/cancelación de cuenta y el fallo al obtener el token producen síntomas distintos. No atribuirlo automáticamente a `.env`. |
| Acción | Corregir el cliente OAuth web y registrar el SHA-1 correspondiente; repetir con una build del mismo keystore. Ver CASO 007 de `BUGS_RESUELTOS/userapp.md`. |

### El auth gate queda comprobando sin red

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Con sesión guardada y sin red, AdminWeb se queda comprobando el rol; no aparece «No autorizado» ni el contenido. |
| Qué comprobar | Si AdminWeb alcanza su timeout de 10 s y muestra el estado recuperable; en los logs de red/consola, si la lectura queda pendiente. |
| Cómo interpretar | Una lectura remota sin red puede tardar o quedar pendiente según el SDK y la caché. El código actual acota la espera; al vencer pasa a **error recuperable**, no a denegación. |
| Acción | Restablecer la conexión y usar **Reintentar** o **Cerrar sesión**. No dejar lecturas de rol sin timeout. |

## 6. Foreground service y Android

### El servicio se reinicia cada ~40 s

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El panel muestra ciclos de «Motor sin pulsos» → «Servicio Android reiniciado» sin intervención del conductor. |
| Qué comprobar | Si existió alguna vez un fix; `dumpsys location` y visibilidad de satélites; si los pulsos llegan al proceso; si el servicio nativo sigue vivo. |
| Cómo interpretar | El watchdog reinicia tras 30 s sin pulso. En interiores o sin satélites puede no haber fix; tras un fix previo, en cambio, el heartbeat puede seguir emitiendo con la última posición. Un bucle persistente con GPS exterior y pulsos presentes apunta a un fallo real de tracking. |
| Acción | Probar al aire libre y esperar el primer fix. Si persiste con GPS válido, revisar el estado del servicio y el wrapper 4.1.0. Ver `ARCHITECTURE.md` §8.3 y CASO 020. |

### Excepción al iniciar el foreground service (Android 14+)

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | La DriverApp falla al iniciar tracking; Logcat muestra una excepción de foreground service. |
| Qué comprobar | Tipo declarado en `AndroidManifest.xml` (`location`); permiso `FOREGROUND_SERVICE_LOCATION`; `foregroundServiceType: ['location']` en las opciones JS; permiso de ubicación en runtime; si el inicio se intenta desde background sin una excepción permitida. |
| Cómo interpretar | No todas las causas producen `InvalidForegroundServiceTypeException`: un tipo ausente, un permiso faltante o una restricción de inicio pueden producir excepciones distintas (`MissingForegroundServiceTypeException`, `SecurityException`, etc.). El checklist de tres lugares sigue siendo necesario, pero no explica por sí solo toda excepción. |
| Acción | Corregir la causa concreta que indique Logcat y mantener sincronizados manifest, permisos y opciones JS. Ver `ARCHITECTURE.md` y `Driver/README.md`. |

### La ubicación se detiene con la pantalla apagada

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El marcador se congela al apagar la pantalla o minimizar la app. |
| Qué comprobar | Que el foreground service esté activo; que exista la notificación del servicio; si `POST_NOTIFICATIONS` está concedido; si el fabricante aplica restricciones adicionales de batería. |
| Cómo interpretar | Android no exige `POST_NOTIFICATIONS` para iniciar un FGS, pero **esta app bloquea INICIAR cuando se deniega**. Sin notificación visible, el servicio puede seguir existiendo y aparecer en el Task Manager. Si el servicio no está activo, la causa es anterior a la notificación. |
| Acción | Conceder el permiso desde el flujo de la app. El código ofrece un botón que abre Ajustes; no lo abre automáticamente. Revisar también las restricciones de batería del fabricante. |

### La notificación del servicio no aparece

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El conductor no ve «Transmitiendo ubicación». |
| Qué comprobar | Ajustes → Apps → BurritoDriverApp → Notificaciones; si el servicio está corriendo; si el permiso fue denegado. |
| Cómo interpretar | La notificación puede estar oculta por permiso denegado o por el canal, sin que ello demuestre que el servicio no existe. La denegación, en esta app, impide iniciar el tracking. |
| Acción | Activar notificaciones y volver a iniciar. Si el servicio no está activo, diagnosticar primero el inicio y los permisos de ubicación. |

## 7. GPS, tracking y parada

### No llegan coordenadas nuevas

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El servicio figura activo, pero no aparecen nuevas coordenadas en RTDB. |
| Qué comprobar | GPS del dispositivo encendido; permisos `ACCESS_FINE_LOCATION` y `ACCESS_BACKGROUND_LOCATION`; primer fix disponible; errores del `locationTask` en Logcat; escrituras a `/ubicacion_buses/{busId}`. |
| Cómo interpretar | Son etapas distintas: **no hay fix**, **hay fix pero falla la publicación** o **la publicación llega pero la UserApp la descarta**. Sin primer fix no hay heartbeat; después del primer fix, el heartbeat puede publicar la última posición aunque el GPS no entregue una nueva. |
| Acción | Activar GPS y conceder permisos. Si FINE o BACKGROUND se deniegan, el código aborta el inicio y lo registra; no promete un `Alert` en todos los casos. Ver §6 para servicio y permisos. |

### Coordenadas imprecisas o saltos de ±10–15 m

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El bus salta alrededor de su posición real sin un desplazamiento real. |
| Qué comprobar | Precisión reportada, entorno (interiores/edificios), y si el salto coincide con el fix o con la recepción. |
| Cómo interpretar | El efecto multipath es una explicación posible en el campus, no una causa demostrada por el síntoma. El umbral futuro de geofencing no garantiza absorberlo: sigue siendo calibración pendiente. |
| Acción | Comparar con una posición conocida y validar al aire libre. No prometer mitigación actual. Ver ADR-013/ADR-020. |

### Tracking que no llega a RTDB

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Hay fix GPS, pero no se actualiza `/ubicacion_buses/{busId}`. |
| Qué comprobar | `busId` no vacío; sesión y permisos RTDB de escritura; errores de `writeLocation`; conectividad; que no se esté escribiendo en `/ubicacion_buses/null`. |
| Cómo interpretar | Un error de permisos, una escritura a un bus inexistente o un fallo de red impiden la publicación. Que el servicio esté activo no confirma la escritura. |
| Acción | Corregir la asignación/busId y las reglas de escritura. Revisar el error concreto de RTDB; no ampliar permisos sin contrastar el contrato. |

### El bus no aparece en El Burrito

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | La UserApp no muestra el bus. |
| Qué comprobar | Que exista `/ubicacion_buses/{busId}` con `isActive: true`; el `timestamp`; si el bus está dentro de los límites del campus; si la app recibió el snapshot y lo descartó. |
| Cómo interpretar | La UserApp oculta el bus si `isActive` es `false`, si supera el umbral de antigüedad de 30 s o si el dato no es utilizable. La selección del bus y la caducidad son pasos distintos de la publicación. |
| Acción | Verificar primero la publicación en RTDB; después, la recepción y la caducidad en la UserApp. |

### El bus está congelado o «viaja en el tiempo»

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | La posición no avanza, o al reconectar aparecen posiciones antiguas en rápida sucesión. |
| Qué comprobar | `timestamp` de la última publicación; si el conductor sigue publicando; si `index.js` mantiene desactivada la persistencia en disco; si las escrituras pendientes del SDK se entregan al reconectar; si el timestamp recibido es futuro. |
| Cómo interpretar | Sin persistencia en disco, el SDK **todavía puede conservar escrituras pendientes en memoria** y entregarlas al reconectar. No asumir que todo intento offline se descarta. Un heartbeat reciente tampoco demuestra un fix nuevo. La deduplicación y la tolerancia a timestamps futuros se aplican en la recepción. |
| Acción | No reactivar persistencia en disco. Revisar relojes y orden de los datos; si el bus no avanza, comprobar GPS y publicación. Ver ADR-003, ADR-030 y CASO 019. |

### I06-B: timestamp futuro rechazado

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Una publicación reciente no reemplaza la última posición válida; el bus queda retrasado o caduca. |
| Qué comprobar | Diferencia entre el timestamp publicado y la hora de recepción; si supera los 300 000 ms; reloj del emisor y del receptor. |
| Cómo interpretar | La recepción descarta adelantos mayores a cinco minutos; el límite exacto se acepta. No sustituye fechas inválidas por «ahora» ni garantiza sincronización de relojes. La última posición válida caduca por el mecanismo existente. |
| Acción | Corregir el reloj del dispositivo emisor y volver a publicar. Ver ADR-030 y CASO 019 de `BUGS_RESUELTOS/userapp.md`. |

### Timestamps inconsistentes entre nodos

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Los timestamps de tracking, perfiles o feedback no son comparables. |
| Qué comprobar | Qué método usa cada nodo: tracking y creación de registros con `Date.now()`; perfiles y feedback con `ServerValue.TIMESTAMP`. |
| Cómo interpretar | Ambos métodos coexisten porque los paths y propósitos son distintos. No son intercambiables sin definir el contrato del nodo. |
| Acción | No unificar por suposición. Consultar la representación de cada nodo en `FIREBASE_SCHEMA.md`. |

### Parada local sin confirmación remota

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Al pulsar DETENER, la UI puede quedar en **PARADA LOCAL NO CONFIRMADA** o mostrar que una operación anterior llegó tarde. |
| Qué comprobar | Si la parada local terminó; si el servicio nativo se detuvo; si la UI distingue parada local de parada remota; si INICIAR sigue bloqueado; si el ACK es de la operación vigente. |
| Cómo interpretar | I07 desacopla el cierre local del ACK remoto: DETENER invalida recursos, espera el stop nativo y solicita `isActive: false` antes de liberar INICIAR. Una respuesta antigua solo informa diagnóstico. Un ACK o un heartbeat nuevo no prueban adquisición GPS reciente ni dos motores simultáneos. Sin persistencia en disco aún puede haber pendientes en memoria; no asumir cierre remoto garantizado. |
| Acción | Si aparece PARADA LOCAL NO CONFIRMADA, reintentar DETENER; no iniciar otro recorrido hasta resolverla. Evidencia y residual del wrapper: CASO 020 de `BUGS_RESUELTOS/driverapp.md`. |

## 8. Mapbox

### El mapa no carga

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El mapa se ve gris o blanco. |
| Qué comprobar | `MAPBOX_PUBLIC_TOKEN` presente y válido; que `Mapbox.setAccessToken(...)` se ejecute; errores de red/estilo en la consola. |
| Cómo interpretar | Un token inválido o ausente impide cargar el estilo; un token válido no descarta un fallo de red o de inicialización. |
| Acción | Corregir el token y recargar. Ver el README de UserApp. |

### La cámara se sale del campus

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El usuario puede desplazar la cámara fuera de los límites definidos. |
| Qué comprobar | Que `Camera.maxBounds` reciba `UNMSM_BOUNDS` y se aplique después de cargar el estilo; si el comportamiento se reproduce en dispositivo. |
| Cómo interpretar | `maxBounds` limita el paneo y el zoom de la cámara; **no es una restricción del centro** ni un límite de negocio independiente. Que el código configure el prop no certifica por sí solo el comportamiento observado. |
| Acción | Verificar en dispositivo y ajustar la configuración si el límite no se aplica. Documentar el caso reproducido. |

## 9. Renderizado y tema

### La app se queda en el splash

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | La UserApp queda en el splash y no muestra login ni mapa. |
| Qué comprobar | Errores de hidratación de `userStore` y `themeStore`; que ambos `_hasHydrated` lleguen a `true`; que el splash se oculte en `onReady`. |
| Cómo interpretar | El `NavigationContainer` no se renderiza hasta completar ambas hidrataciones. Un fallo de restauración puede dejar el estado incompleto. |
| Acción | Reiniciar la app y revisar los errores. **Como último recurso**, borrar los datos de la aplicación desde Ajustes; esto elimina todos sus datos locales, no solo el tema. |

### La interfaz va lenta o el mapa pierde FPS

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Baja tasa de FPS o tirones durante el tracking. |
| Qué comprobar | Si el listener de RTDB pasa por `burritoLocationStore` y no actualiza directamente un componente; cuántos renders provoca cada snapshot; perfilado en dispositivo. |
| Cómo interpretar | La arquitectura con Zustand existe, pero su presencia no demuestra que el listener sea la causa de los FPS. Hay que medir antes de atribuirlo. |
| Acción | Instrumentar renders y perfilado; mantener el listener fuera de los componentes. Ver `ARCHITECTURE.md`. |

### El tema oscuro no se conserva como se espera

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | El tema cambia durante el uso, pero al reiniciar parece volver al esquema del sistema. |
| Qué comprobar | Si el tema guardado se restauró; si `DrawerNavigator` ejecutó `setTheme(systemColorScheme === 'dark')` al montar o al cambiar el esquema; si el toggle se hizo después de esa sincronización. |
| Cómo interpretar | `themeStore` persiste el valor y llama a `Appearance.setColorScheme`, pero `DrawerNavigator.tsx:141–147` sincroniza el tema con el esquema del sistema. Un toggle manual puede ser sobrescrito; no está definido que el valor manual prevalezca siempre. |
| Acción | Reproducir con el mismo esquema del sistema y revisar el orden de montaje/hidratación. No asumir que el toggle dura una sesión ni que el sistema siempre prevalece. |

### AdminWeb: el foco de un modal queda atrapado o se pierde

| Paso | Qué comprobar |
|------|---------------|
| Síntoma | Al abrir/cerrar un modal, el foco no vuelve al elemento esperado o queda dentro de un modal cerrado. |
| Qué comprobar | Si el problema ocurre con teclado, con un modal anidado o solo en un navegador concreto; si el foco se restaura al cerrar. |
| Cómo interpretar | La corrección del comportamiento de foco está integrada; la comprobación visual en navegador sigue pendiente. No afirmar que el defecto continúa sin reproducirlo en la build actual. |
| Acción | Documentar navegador y pasos; contrastar con `BUGS_RESUELTOS/adminweb.md`. |

## 10. Limitaciones y estado

Lo siguiente **no son necesariamente errores actuales**. Los riesgos aceptados
no deben presentarse como resueltos.

| Tema | Estado |
|------|--------|
| I06-A: campos ausentes/null | Se conservan los respaldos de coordenadas 0 y timestamp actual; no afirmar que todo registro incompleto se rechaza. Limitación aceptada. |
| I06-B: timestamps futuros | Rechazo >5 min implementado en código integrado; validación física prolongada y publicación de APK no acreditadas. |
| I04: reglas de lectura/feedback | Hubo confirmación de publicación y operación el 03/11; no hay volcado versionado de reglas ni inspección live actual. El contrato vigente está en `FIREBASE_SCHEMA.md`. |
| I07: cierre local/remoto | Implementado e integrado; el residual del wrapper 4.1.0 está en CASO 020. |
| I08: asignación offline | Consulta y respaldo en memoria implementados; no detecta cambios administrativos offline. |
| I11: identidad de diagnóstico | `setUserId(uid)`/`setUserId('')` solicitado por la app; la aplicación nativa/recepción de reportes no está acreditada por las pruebas registradas. |
| Un solo bus visible en UserApp | Límite del MVP; multi-bus está en ROADMAP. |
| Sin geofencing ni cierre automático de vueltas | Límite del MVP; ver ROADMAP y ADR-020. |
| Sin cierre remoto garantizado | No hay `onDisconnect`; si el proceso muere, `isActive` puede quedar en `true`. El heartbeat no garantiza cierre. Ver ADR-026. |
| Respaldo offline de la asignación | En memoria; no sobrevive al proceso ni cubre cambios administrativos offline. Solo se habilita con `.info/connected == false`. |
| Etiqueta «Servicio de ubicación activo» | Describe el servicio Android local; no confirma publicación en RTDB ni conexión. |
| iOS no soportado en DriverApp | Limitación de plataforma. |
| Deriva GPS en interiores | Observada el 24/08/2026; no se afirma que siga ocurriendo. ADR-020 sigue experimental. |
| Corte de red durante la sesión | Firebase puede no disparar `onError`; la UserApp oculta por caducidad y DriverApp depende del heartbeat/watchdog. |
| Deuda de lint de UserApp | Post-MVP; ver `LINT_DEBT.md`. |
| Exposición histórica de la contraseña de firma Android | La contraseña quedó expuesta en el historial de Git. El keystore de release no se encontró en el historial auditado y no hay evidencia identificada de uso indebido. La configuración actual se conserva. **No se ha decidido rotar la clave**; se reevaluará si aparecen evidencias de compromiso o cambian las condiciones de riesgo. `ROADMAP.md` exige rotación antes de Play Store; esta guía no resuelve esa discrepancia ni autoriza una rotación por suposición. |

## 11. Referencias

| Documento | Uso |
|-----------|-----|
| `ARCHITECTURE.md` | Funcionamiento, flujo de datos y límites del sistema. |
| `FIREBASE_SCHEMA.md` | Nodos, permisos, consultas e índices de RTDB. |
| `DECISIONS.md` | Decisiones y riesgos aceptados. |
| `BUGS_RESUELTOS/` | Historial de incidencias y evidencia. |
| `MVP.md` / `ROADMAP.md` | Estado del MVP y planificación posterior. |
| `AGENTS.md` de cada app | Reglas operativas y setup específico. |
