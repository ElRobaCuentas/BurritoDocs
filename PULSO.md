# PULSO — Referencia técnica y operativa del MVP

## 1. Presentación y propósito

PULSO es el monitor técnico interno del seguimiento del bus universitario del
Ecosistema El Burrito. Permite saber qué publicaciones se observaron, detectar
interrupciones y conservar incidentes para su consulta administrativa en Central
(AdminWeb), sin depender de que alguien mantenga el panel abierto.

Lo construimos para distinguir un recorrido declarado activo de un seguimiento
que realmente sigue publicando. No sustituye el mapa de El Burrito, el productor
GPS de BurritoDriverApp ni una herramienta de diagnóstico integral del teléfono.

**Estado del MVP:** implementado, desplegado y validado operativamente el
**18/11/2026** para `AHK-459`. La evidencia y sus límites están en §9. Este cierre
no acredita el cierre del MVP general de El Burrito, definido en `../MVP.md`.

Esta es la referencia de funcionamiento, alcance y validación. La integración
general vive en [ARCHITECTURE §3.2](ARCHITECTURE.md#32-pulso-monitor-de-seguimiento),
los procedimientos técnicos en el [README de AdminWeb](https://github.com/ElRobaCuentas/AdminWeb/blob/main/README.md#monitor-pulso-local-y-despliegue-remoto)
y la planificación posterior en [ROADMAP](ROADMAP.md#21-evolución-de-pulso-posterior-al-mvp).

## 2. Alcance del MVP

- Supervisa un bus configurado, actualmente `AHK-459`, leyendo su nodo
  `/ubicacion_buses/{busId}` de Firebase RTDB.
- Observa declaración de actividad, antigüedad de publicación y accesibilidad de
  la fuente; conserva estado, comprobaciones e incidentes en Cloudflare D1.
- Expone los registros mediante una API administrativa y la pantalla `/pulso`.
- Detecta síntomas: una publicación antigua o una lectura fallida. No demuestra
  por sí solo si la causa es red, GPS, batería, Android o un error de aplicación.

El adaptador acepta los seis campos actuales: `isActive`, `timestamp`, `latitude`,
`longitude`, `heading` y `speed`. Distingue campos ausentes e inválidos y no deriva
telemetría adicional. El contrato del productor pertenece a
[FIREBASE_SCHEMA §2](FIREBASE_SCHEMA.md#ubicacion_busesplaca).

El `timestamp` es de publicación, no del fix GPS original. El heartbeat puede
republicar la última posición: publicación reciente no prueba una medición nueva
ni movimiento. Tampoco unas coordenadas repetidas prueban una avería.

## 3. Funcionamiento operativo

La operación usa un Motorola compartido durante la jornada. Los relevos de
conductores no requieren detener el recorrido ni cambiar la sesión; la cuenta
autenticada no identifica necesariamente al conductor físico del momento.

| Momento | Comportamiento de PULSO |
|---------|------------------------|
| Inicio | Al observar `isActive:true`, registra vigilancia activa. Si inicia un período nuevo, guarda la hora de esa comprobación como inicio observado. |
| Seguimiento normal | Comprueba la fuente cada minuto, evalúa la publicación y conserva el resultado. No exige que Central esté abierta. |
| Interrupción | Con actividad declarada y publicación de más de tres minutos, abre un incidente de seguimiento interrumpido. No fecha la caída retrospectivamente. |
| Problema persistente | Actualiza el mismo incidente abierto; no crea uno por cada minuto. |
| Recuperación | La primera publicación reciente inicia confirmación; una segunda válida y con timestamp mayor confirma y cierra el incidente. |
| Detención normal | Al observar `isActive:false`, desactiva la vigilancia del recorrido, limpia su inicio y cierra seguimiento interrumpido por inactividad, sin inventar recuperación. |
| Fuera del recorrido | El cron continúa comprobando y guardando resultados, pero no abre nuevos incidentes mientras la vigilancia registrada esté inactiva. |

Ausencia de nodo, declaración inválida o fuente inaccesible no equivalen a
DETENER: conservan la última vigilancia conocida. Si todavía no se observó
actividad, una fuente inaccesible no abre un incidente de recorrido.

El inicio observado persiste durante interrupciones, recuperación, falta de
acceso a Firebase y reinicios del Worker con la misma memoria compatible. Un
relevo sin `false` observado mantiene el mismo período. No existen horarios de
jornada ni identificación inequívoca de cada recorrido en este MVP.

## 4. Arquitectura técnica

```text
BurritoDriverApp → Firebase RTDB → Cloudflare Worker (cron) → D1
                                                            ↓
                                      API administrativa → Central
```

Son dos procesos independientes, aunque el cron y la API pertenecen al mismo
Worker:

1. **Vigilancia:** el cron carga la memoria anterior, lee RTDB, adapta los campos,
   ejecuta el evaluador puro y persiste estado, observación e incidentes.
2. **Consulta:** Central envía su ID token de Firebase Authentication a
   `GET /api/pulso`. El Worker autoriza mediante Firebase y lee únicamente D1
   para devolver los registros. No ejecuta una evaluación ni consulta el tracking.

La lectura operativa del tracking es HTTP pública, sin token de conductor ni
escrituras en RTDB. La autorización administrativa es distinta: el Worker
prevalida el token y delega su verificación a Firebase al leer
`/administradores/{uid}` con las reglas existentes. La existencia del nodo concede
acceso; la UI, el email y el UID enviado por el cliente no sustituyen esa frontera.
La API depende de Firebase para autorizar, aunque sus datos provengan de D1.
La lectura usa timeout y límite de respuesta, bloquea redirecciones y diferencia
fallos sanitizados; no intenta destinos alternativos ni registra el payload crudo.

| Responsabilidad | Fuente en `AdminWeb/` |
|-----------------|-----------------------|
| Evaluación pura, sin I/O ni temporizadores | `monitor/src/domain/evaluateTracking.ts` |
| Lectura HTTP y adaptación de los seis campos | `monitor/worker/src/firebaseRtdb.ts`, `toTrackingObservation.ts` |
| Ciclo automático y validación de configuración | `monitor/worker/src/index.ts`, `executionConfig.ts` |
| Memoria e incidentes transaccionales | `monitor/worker/src/trackingStore.ts`, `trackingIncidents.ts` |
| API y autorización administrativa | `monitor/worker/src/pulsoRead.ts`, `adminAuthorization.ts` |
| Consulta, presentación y reloj visual | `src/features/pulso/services/pulso_service.ts`, `pulsoPresentation.ts`, `screen/PulsoScreen.tsx` |

En remoto, la API exige HTTPS, `GET` sin query y, cuando se envía `Origin`, el
origen exacto de Central. El preflight autorizado acepta `GET` + `Authorization`
con `OPTIONS` 204; las respuestas no se cachean. Distingue 401 por sesión
ausente/token inválido, 403 por acceso denegado y 503 por configuración o servicio
no disponible. No expone coordenadas, perfiles, SQL ni credenciales.
`/evaluate` es exclusivamente local, limitado a loopback y cabecera
`X-Pulso-Local: 1`; no es una forma de forzar comprobaciones en producción.

La API devuelve `no-data` u `observed`, con `observedState`, `lastCheck`,
`activeIncidents` y `recentIncidents`. Lee estado, observación enlazada por
`last_check_id` e incidentes en un mismo batch de `SELECT`. Los códigos de error
principales son `authentication-required`/`invalid-token` (401), `access-denied`
(403) y `authorization-not-configured`/`authorization-unavailable`/
`read-not-configured`/`read-failed` (503). No confundir autorización inaccesible
con un incidente persistido de acceso a la fuente del tracking.

## 5. Estados e incidentes

### Clasificación del seguimiento

La clasificación resume la última comprobación; no es el estado de un incidente
ni una certificación de la situación actual del bus. Central la presenta con una
etiqueta operativa en «Estado del recorrido» y conserva la etiqueta técnica en
«Detalle del estado registrado».

| Código | Etiqueta operativa | Etiqueta técnica y significado |
|--------|--------------------|--------------------------------|
| `active-publishing` | Seguimiento funcionando | Recorrido activo · Publicación reciente. |
| `active-publication-stale` | Seguimiento interrumpido | Recorrido activo · Publicación interrumpida. |
| `recovering-pending` | Seguimiento en recuperación | Recuperación pendiente de confirmación. |
| `recovering-confirmed` | Recuperación confirmada | Recuperación confirmada; después puede volver a publicación reciente. |
| `inactive-unconfirmed` | Bus fuera de recorrido | Recorrido inactivo · Finalización no confirmada: hay `false`, pero falta evidencia independiente de detención. |
| `source-unavailable` | Estado del recorrido sin confirmar | Fuente inaccesible: no se pudo obtener una respuesta utilizable de RTDB. |
| `insufficient-data` | Información del recorrido incompleta | Información insuficiente: falta una declaración o un timestamp utilizable para decidir. |
| `inactive-finalized` | Bus fuera de recorrido | Recorrido finalizado con evidencia. Existe en el evaluador, pero requiere telemetría que el adaptador actual no proporciona. |

El evaluador admite campos futuros como `tripSessionId`, `gpsFixId`,
`gpsFixTimestamp` y `stopEvidence`; se simulan en pruebas del dominio. No son
capacidades conectadas al productor actual ni evidencia disponible en producción.
La finalización no confirmada tras DETENER es un límite de observabilidad, no una
prueba de que la detención haya fallado.

### Incidentes persistentes

| Tipo | Apertura | Cierre |
|------|----------|--------|
| `tracking-interrupted` — Seguimiento interrumpido | Vigilancia activa y clasificación `active-publication-stale`. | `publications-recovered` al confirmar recuperación, o `tracking-inactive` al observar `false`. |
| `source-unavailable` — Fuente inaccesible | Vigilancia activa y fallo de lectura: red, timeout, HTTP denegado/fallido o respuesta no utilizable. | `source-accessible` al volver a poder leer, incluso si falta el nodo o la publicación sigue antigua. |

- **Umbral:** antigüedad estrictamente mayor a `180000` ms. A los tres minutos
  exactos no se abre interrupción; la detección ocurre en una comprobación posterior.
- **Recuperación de publicaciones:** dos observaciones de publicaciones recientes
  válidas con timestamps crecientes. Repetir el mismo dato no suma confirmación;
  una nueva interrupción durante la recuperación reinicia las confirmaciones.
- **Acceso a la fuente:** no necesita dos publicaciones nuevas para cerrar su
  incidente. Restablecer acceso no equivale a recuperar publicaciones.
- Ambos incidentes pueden coexistir. Hay uno abierto como máximo por fuente,
  bus y tipo; las comprobaciones posteriores actualizan `updated_at`,
  `last_check_id` y `check_count` hasta el cierre.
- `duration_ms = closed_at - opened_at`: duración del incidente observado, no
  duración exacta de la desconexión ni del recorrido. Abierto, no tiene duración
  final persistida. Una recurrencia posterior al cierre puede crear otro incidente.

## 6. Memoria y persistencia

| Tabla D1 | Responsabilidad y conservación |
|----------|--------------------------------|
| `tracking_state` | Snapshot por fuente y bus: revisión, última comprobación, clasificación, configuración, memoria del evaluador, vigilancia e inicio observado. Conserva las últimas coordenadas necesarias para comparar, no un trayecto histórico. |
| `tracking_observations` | Resumen sanitizado de cada comprobación aceptada, declaración, timestamp de publicación y resultado. No almacena un historial de coordenadas precisas. Se podan filas de más de siete días al guardar nuevas comprobaciones, preservando la observación actual. |
| `tracking_incidents` | Identidad, tipo, apertura, última observación, contador, cierre, motivo y duración observada. Sin poda automática implementada. |

La clave combina modo/origen/namespace de la fuente y bus. La memoria conserva
interrupciones y recuperación entre ejecuciones sin depender de variables del
proceso. Su versión y configuración deben ser compatibles; una memoria ilegible
o incompatible falla explícitamente, sin reiniciarse ni consultar la fuente como
si no existiera estado previo.

El estado anterior se carga antes del HTTP. El batch D1 escribe snapshot,
historial, poda e incidentes con control de concurrencia (`revision` y
`last_check_id`); una comprobación perdedora no debe sobrescribir otra aceptada
ni producir actualizaciones parciales (`memory-conflict`).

| Migración en `monitor/worker/migrations/` | Cambio |
|----------------------------------------|--------|
| `0001_tracking_memory.sql` | Crea snapshot, historial, claves e índice de retención. |
| `0002_tracking_incidents.sql` | Agrega vigilancia conservando la última declaración conocida y crea incidentes e índice único parcial; no fabrica incidentes históricos. |
| `0003_tracking_observed_start.sql` | Agrega `observed_trip_started_at` nullable, entero positivo y no posterior a `evaluated_at`; no reconstruye inicios anteriores. |

El inicio se guarda fuera de `prior_state_json`, sin cambiar el evaluador ni su
versión de memoria. Se asigna a `evaluated_at` al iniciar un período observado,
se conserva mientras la vigilancia siga activa y se limpia al observar `false`.
Un período ya activo sin inicio confiable sigue con `null` hasta que termine;
no se fecha desde una migración o un despliegue.

La API lo expone como `observedState.observedTripStartedAt`; el cliente tolera
su ausencia durante una transición desde la API anterior. La duración del
recorrido se calcula en Central, en minutos enteros u horas y minutos. No hay
contador persistente por segundo, tabla de recorridos ni duración final de cada
recorrido archivada por esta mejora.

## 7. Interfaz de Central

`/pulso` agrupa la lectura en tres bloques —«Estado del recorrido», «Estado de
PULSO» e «Incidentes»— y un aviso superior cuando la comprobación es antigua.
Todas las fechas se presentan en `America/Lima`. Los indicadores describen los
registros consultados; no certifican la situación actual del bus, del monitor ni
de la red.

### Estado del recorrido

El encabezado usa la etiqueta operativa de la clasificación (§5). El bloque
muestra la última publicación del bus y, solo mientras la vigilancia está activa
y la clasificación no es de inactividad, el inicio y el tiempo de recorrido.

| Indicador | Qué representa |
|-----------|----------------|
| Encabezado y descripción | Etiqueta operativa de la clasificación en la última comprobación (§5). |
| Inicio observado | Primera comprobación del período activo registrado. No es necesariamente la hora de INICIAR en DriverApp. |
| Tiempo de recorrido | Estimación desde ese inicio, incluyendo interrupciones. No mide transmisión GPS continua. |
| Última publicación del bus | `publicationTimestamp` recibido de RTDB, si es utilizable; no es hora del fix GPS ni hora de lectura de PULSO. |
| Detención | Solo en estados inactivos: «Declarada» o «Con evidencia registrada», con «Sin verificación independiente» cuando falta esa evidencia. |
| Detalle del estado registrado | Desplegable con la etiqueta técnica, `declaredActive` y `monitoringActive`. |

Con vigilancia inactiva o clasificación de inactividad, el bloque omite inicio y
duración en lugar de mostrar «No aplica». Con vigilancia activa y sin inicio
confiable muestra «Inicio no determinado» y omite la duración. Si los datos están
vencidos o la consulta falló, el bloque pasa a tono de advertencia y antepone
«Último estado: » con la aclaración «Según la última revisión guardada; falta
confirmar la situación actual».

**Inicio y duración:** con datos recientes y fuente/declaración activas
confirmadas, el reloj del servidor ancla la estimación y el tiempo local la hace
avanzar. Si los datos envejecen, falla la consulta, la fuente no está `ok` o la
declaración es desconocida, se limita a la última comprobación guardada con la
etiqueta «(hasta la última comprobación guardada)».

### Estado de PULSO

Este bloque describe la revisión del monitor, no el recorrido. Muestra cuándo se
realizó la última revisión, su antigüedad y el resultado de la conexión con
Firebase.

| Indicador | Qué representa |
|-----------|----------------|
| Encabezado | «Revisión reciente», «Sin revisión reciente», «Última revisión guardada», «Revisión sin datos del bus» o «Revisión con fallo de conexión». |
| Última revisión | `evaluatedAt`: cuándo PULSO evaluó el dato guardado. |
| Hace cuánto | Tiempo desde esa revisión, no desde la última publicación del teléfono. |
| Conexión con Firebase | «Consulta completada», «Consulta completada, sin datos del bus» o «No se pudo consultar», según el resultado de esa revisión. |

El encabezado prioriza la antigüedad: una revisión vencida muestra «Sin revisión
reciente» aunque su fuente haya sido accesible. Con datos vencidos o consulta
fallida, la conexión se refiere a la última revisión guardada y no confirma el
acceso actual.

### Incidentes

Un único bloque agrupa los problemas abiertos y el historial reciente, con el
número de incidentes activos en una insignia.

- **Incidentes activos:** los que permanecían abiertos en la última revisión.
  Sin ellos, muestra «Sin incidentes activos».
- **Historial reciente:** los últimos 20 incidentes, abiertos y cerrados, del
  más reciente al más antiguo. No es la totalidad de D1 y puede incluir un
  incidente que también aparezca como activo.
- Cada incidente muestra tipo, estado «Abierto»/«Cerrado», inicio, última
  observación y cierre; al estar cerrado agrega la duración registrada y el
  resultado (§5). Abierto, indica «Sin cierre registrado».

Sin incidentes guardados, el historial lo dice explícitamente y aclara que las
revisiones del bus sí están guardadas.

### Actualización, antigüedad y errores

**Dos ciclos independientes:** el Worker comprueba cada minuto; Central consulta
al entrar y cada 60 s mientras la pantalla permanece montada. Actualizar repite
la consulta, sin evaluar ni escribir D1. No hay sincronización entre ambos ciclos
ni garantía de que la pantalla refleje una comprobación inmediatamente.

Central impide consultas solapadas, cancela al salir y descarta respuestas
tardías. Un error conserva los últimos registros con aviso; una denegación 401/403
retira los datos. Sin comprobaciones guardadas muestra «Sin registros de PULSO»;
no lo confunde con un error ni con ausencia de incidentes.

La antigüedad visual avanza sin reclasificar el seguimiento ni crear incidentes.
Superado el umbral de antigüedad de comprobación, un aviso indica «Datos
antiguos»; una consulta exitosa no rejuvenece una comprobación vieja. La
advertencia significa que el estado y los incidentes guardados no confirman la
situación actual.

## 8. Operación en producción

### Configuración y control

La configuración versionada está en `monitor/worker/wrangler.jsonc`, entorno
`production`: Worker `pulso-monitor`, binding `PULSO_DB` a
`pulso-memory-production`, bus `AHK-459`, modo `cloudflare`, memoria `d1` y fuente
`operative-public`. Cron `* * * * *`, `PULSO_SCHEDULE_ENABLED="true"`, umbral
`180000` ms, recuperación `2`, tolerancia futura `5000` ms y timeout HTTP
`5000` ms. Previews deshabilitadas. Una configuración remota incompleta falla
antes de leer tracking o persistir una comprobación.

- Worker/API: `https://pulso-monitor.elburrito.workers.dev/api/pulso`.
- Central: `https://burritounmsm.web.app/pulso`; origen permitido
  `https://burritounmsm.web.app`.
- La build usa `VITE_PULSO_API_ORIGIN`; la autorización usa las variables
  `PULSO_ADMIN_*` del proyecto `burritounmsm`. No hay login adicional de PULSO.

Para comprobar funcionamiento, abrir Central con una sesión administrativa y
comparar comprobaciones sucesivas, su antigüedad y la fuente. Un estado inactivo
puede ser normal; no equivale a cron detenido. Si hace falta diagnóstico técnico,
contrastar ejecuciones programadas, errores sanitizados y última escritura D1.
Un fallo de D1/configuración falla el handler; un fallo de fuente correctamente
guardado no es el mismo caso. El botón Actualizar no repara ni dispara el cron.

Para suspender vigilancia, el interruptor y el trigger son controles distintos:
apagar el primero evita evaluar/persistir en la entrada programada; retirar el
cron evita futuras invocaciones. La API puede seguir consultando memoria, que
envejecerá. No se limpia D1 ni se simula DETENER. El procedimiento de cambio,
publicación y verificación pertenece al
[README de mantenimiento](https://github.com/ElRobaCuentas/AdminWeb/blob/main/README.md#mantenimiento-de-la-vigilancia).

### Evidencia de publicación conservada

La integración administrativa inicial del 15/11/2026 mostró correctamente
«Sin registros de PULSO» con D1 vacía. Ese resultado fue previo a activar la
vigilancia; no describe el estado posterior a las pruebas del 18/11.

| Hito | Evidencia registrada |
|------|----------------------|
| Activación, 15/11/2026 | Cron e interruptor habilitados; dos ejecuciones programadas reales consecutivas, revisiones 1 y 2, coincidentes con D1 y sin incidentes para el bus inactivo. |
| Migración `0003`, 18/11/2026 | Aplicada antes del nuevo Worker. Conservados el estado y las 4190 observaciones previas mediante comparación de hashes; sin pérdida ni reinicialización, `quick_check` correcto y sin violaciones de claves foráneas. |
| Worker publicado, 18/11/2026 | Versión `0423f706-8ee8-4cec-a3d4-3f09bef83e33`, deployment `e1178ef4-926c-48e9-ad2f-f31a0d4195a5`, `20:16:04.969567Z`. Referencia previa: `add07c85-1cb5-49d8-a24f-4bd72544e2d5`. |
| Cron tras publicación | Revisiones 4193 y 4194, `20:17:12.259Z` y `20:18:12.346Z`, ejecuciones reales correctas y resultados coincidentes en D1. Snapshot previo a la jornada de pruebas: 1 estado, 4194 observaciones, 0 incidentes, bus inactivo e inicio `null`. |
| Central publicada, 18/11/2026 | Release `1795033146333000`, versión `3026576fcc32c269`, `20:19:06.333Z`, finalizada. Referencia previa: `3b69ad434e3d33d7`, release `1794782963690000` (refresco automático). |

Las horas de publicación de esa tabla son UTC (`Z`). Los cuatro archivos de
Hosting coincidieron con `dist/` por SHA-256 y `/pulso` resolvió a `index.html`.
Las verificaciones HTTPS registraron 401 sin token o con token malformado,
preflight 204 con CORS exacto, rechazo de origen ajeno/métodos no permitidos y
403 en `/evaluate`. No se conservaron contraseñas ni tokens en la evidencia.

Estas son referencias históricas, no una consulta remota actual. Un rollback
requiere revisar compatibilidad: el Worker anterior a `0003` no mantiene el
inicio observado; no se debe asumir que la fecha seguirá siendo confiable tras
alternar versiones. No se elimina la columna ni el historial para revertir UI.

## 9. Validación operativa del MVP

**Fecha:** 18/11/2026. **Bus:** `AHK-459`. **Zona:** `America/Lima` (Perú).
Pruebas con Motorola, BurritoDriverApp, Firebase de producción, Cloudflare y
Central. Fuente: registro operativo aportado por el responsable del proyecto.

| Prueba | Resultado y alcance comprobado |
|--------|--------------------------------|
| 1. Recorrido inactivo | Aprobada. Se observó la inactividad. |
| 2. Inicio del recorrido | Aprobada. Inicio del período activo observado a las 15:49:12. |
| 3. Seguimiento normal y duración | Aprobada mediante observación continuada; no se ejecutó por separado todo el procedimiento original. Se comprobó la continuidad del inicio observado durante interrupción y recuperación. |
| 4. Detección de interrupción | Aprobada. Incidente abierto a las 15:54:12 tras dejar de llegar publicaciones recientes. |
| 5. Persistencia sin duplicados | Aprobada. A las 15:59:12 seguía abierto el mismo incidente, sin duplicados. |
| 6. Recuperación y cierre | Aprobada. Recuperación confirmada y cierre a las 16:03:12; duración registrada de nueve minutos. |
| 7. Detención normal | Aprobada. Inactividad observada a las 16:07:12; el inicio observado dejó de aplicar. |
| 8. Ausencia de incidentes tras finalizar | Aprobada. A las 16:15:12 continuaban las comprobaciones sin nuevos incidentes. |

### Cronología comprobada

| Hora en Lima | Hecho registrado |
|--------------|------------------|
| 15:49:12 | PULSO observa el inicio del período activo. |
| 15:50:47 | Última publicación antes de desconectar Internet. |
| Aproximadamente 15:51 | Se desconecta Internet del Motorola. |
| 15:54:12 | PULSO detecta la interrupción y abre el incidente. |
| 15:59:12 | El mismo incidente permanece abierto, sin duplicados. |
| Aproximadamente 16:01 | Se restablece Internet. |
| 16:03:12 | PULSO confirma la recuperación y cierra el incidente. |
| Aproximadamente 16:07 | Se presiona DETENER RECORRIDO en DriverApp. |
| 16:07:04 | Publicación con `isActive:false`, según el registro operativo. |
| 16:07:12 | PULSO observa la inactividad. |
| 16:15:12 | Continúa comprobando Firebase sin generar nuevos incidentes. |

Los **nueve minutos** corresponden a `16:03:12 − 15:54:12`, entre apertura y
cierre observados. No equivalen a la duración exacta del corte de Internet.
El registro de `false` no implica que DETENER renueve el campo `timestamp`:
el contrato del productor conserva su distinción entre declaración y publicación.

El inicio observado se mantuvo durante interrupción y recuperación y dejó de
aplicar al finalizar. No se atribuyen a esta jornada fallos provocados de acceso
a Firebase, reinicios, cambios de conductor ni fallos del propio Worker.

### Verificación automatizada previa a la publicación

Se registraron **237 pruebas aprobadas**: 147 de Worker, 45 de evaluador, 29 de
cliente/presentación/navegador, 9 de integración de pantalla, 6 de API y 1
histórica de RTDB. También aprobaron typecheck, lint, build y ensayo de despliegue;
el build informó un chunk de 521,27 kB, superior al aviso de 500 kB.

La cobertura incluye fuente inaccesible, reinicios, concurrencia, integridad de
migraciones, retención y refresco con reloj controlado. Es evidencia automatizada,
no pruebas físicas adicionales ni una nueva ejecución durante el cierre
documental. No se acredita con estas ocho pruebas una jornada completa de ocho
horas ni una medición independiente del intervalo exacto de refresco en producción.

## 10. Limitaciones y evolución futura

### Límites aceptados del MVP

- Observa publicaciones, no garantiza frescura de cada medición GPS ni identifica
  necesariamente la causa de una interrupción.
- No supervisa integralmente errores internos de BurritoDriverApp ni El Burrito
  (UserApp), y no garantiza detectar fallos del propio Worker sin supervisión externa.
- El muestreo periódico introduce demora entre un hecho y su observación; no es
  telemetría continua ni una alarma instantánea.
- Sin observar un `false` entre períodos puede unir recorridos diferentes.
  El inicio observado no es necesariamente el instante exacto de INICIAR.
- La duración observada incluye cortes de conexión y no se conserva como un
  registro final de recorrido cuando se limpia el inicio.
- `false` permite reconocer inactividad, pero no aporta la evidencia independiente
  necesaria para clasificar finalización confirmada.
- Fuente accesible no significa publicación reciente; una consulta exitosa de
  Central no significa comprobación nueva ni estado actual confirmado.

Son límites de diseño y observabilidad, no fallos encontrados en las ocho pruebas.

### Capacidades no implementadas y posibilidades posteriores

No están conectadas la identidad inequívoca de recorrido/fix ni evidencia
independiente de detención; tampoco hay supervisión externa del monitor,
diagnóstico integral de aplicaciones, gestión multi-bus en Central ni
notificaciones externas. Las últimas permanecen pospuestas.

Más telemetría podría separar mejor publicación y medición; supervisión externa
podría cubrir el fallo del monitor. Ambas requieren diseño, alcance y validación
propios, no se obtienen habilitando únicamente una etiqueta del evaluador.
La clasificación de estas posibilidades, su postergación y cualquier aprobación
futura pertenecen a [ROADMAP §2.1](ROADMAP.md#21-evolución-de-pulso-posterior-al-mvp).
No son compromisos, prioridades ni requisitos adicionales del MVP cerrado.
