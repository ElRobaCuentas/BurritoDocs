# Troubleshooting — El Burrito

## 1. Propósito

Este documento recoge problemas conocidos, sus síntomas, causas y
soluciones. No es una guía de arquitectura ni un changelog.

Responde a la pregunta: **tengo un problema, ¿cómo lo diagnostico y
cómo lo soluciono?**

## 2. Cómo usar este documento

No se lee completo. Se consulta.

Si encuentras un problema, busca primero la categoría correspondiente:

| Categoría | Cuándo consultar |
|-----------|-----------------|
| Configuración y Compilación | El proyecto no compila o no arranca |
| Firebase y RTDB | La base de datos no responde, consultas fallan |
| Autenticación | No se puede iniciar sesión en ninguna app |
| Foreground Service y Android | El tracking no se mantiene vivo en segundo plano |
| Tracking GPS | Las coordenadas no llegan o son incorrectas |
| Mapbox | El mapa no carga o se comporta mal |
| Renderizado y Estado | La interfaz está lenta o no se actualiza |
| Problemas conocidos | Limitaciones del estado actual del proyecto |

Cada caso documenta: **síntoma**, **causa**, **diagnóstico**,
**solución** y **prevención**.

## 3. Configuración y Compilación

### google-services.json faltante

| Campo | Descripción |
|-------|-------------|
| Síntoma | La app crashea al iniciar con errores de Firebase no inicializado |
| Causa | El archivo `android/app/google-services.json` no existe |
| Diagnóstico | Verificar existencia del archivo en la ruta esperada |
| Solución | Descargar `google-services.json` desde Firebase Console (proyecto `burritounmsm`) y colocarlo en `android/app/` |
| Prevención | El README de ambas apps documenta la ubicación exacta |

### Versión de JDK incompatible

| Campo | Descripción |
|-------|-------------|
| Síntoma | Gradle falla con errores relacionados a la versión de JDK |
| Causa | JDK 8 o 11 instalado en lugar de JDK 17+ |
| Diagnóstico | Ejecutar `java --version` en la terminal |
| Solución | Instalar JDK 17 o superior y configurar `JAVA_HOME` |
| Prevención | El proyecto usa Gradle 9.0, que requiere JDK 17+ |

### .env faltante en UserApp

| Campo | Descripción |
|-------|-------------|
| Síntoma | Mapbox en blanco (`MAPBOX_PUBLIC_TOKEN`) o Google Sign-In no funciona (`GOOGLE_WEB_CLIENT_ID`) |
| Causa | El archivo `.env` en la raíz de BurritoUserApp no existe o está incompleto |
| Diagnóstico | Verificar existencia de `.env` y que contenga ambas variables |
| Solución | Crear `.env` con `MAPBOX_PUBLIC_TOKEN=<token>` y `GOOGLE_WEB_CLIENT_ID=<id>` |
| Prevención | README documenta las variables requeridas. `env.d.ts` las tipa |

## 4. Firebase y RTDB

### Missing index en /asignaciones

| Campo | Descripción |
|-------|-------------|
| Síntoma | DriverApp muestra error al consultar `/asignaciones` o la consulta devuelve vacío sin motivo |
| Causa | Falta el índice `.indexOn: "choferId"` en las reglas de Firebase RTDB |
| Diagnóstico | Ir a Firebase Console → Realtime Database → Rules. Verificar que existe: `"asignaciones": { ".indexOn": "choferId" }` |
| Solución | Agregar la regla de índice manualmente en Firebase Console |
| Prevención | FIREBASE_SCHEMA.md documenta el índice requerido |

### Persistence causando ráfagas de posición

| Campo | Descripción |
|-------|-------------|
| Síntoma | El bus aparece "viajando en el tiempo" en el mapa al recuperar conexión |
| Causa | Firebase persistence habilitada: acumula ubicaciones durante pérdida de señal y las reenvía en ráfaga al reconectar |
| Diagnóstico | Verificar en `index.js` de DriverApp que `setPersistenceEnabled(true)` está comentado |
| Solución | No reactivar persistence. El código actual ya la tiene desactivada (valor por defecto de Firebase) |
| Prevención | DECISIONS.md documenta la decisión. `index.js` tiene un comentario explicativo |

### ServerValue.TIMESTAMP vs Date.now()

| Campo | Descripción |
|-------|-------------|
| Síntoma | Timestamps inconsistentes entre nodos de Firebase |
| Causa | DriverApp usa `Date.now()` (reloj del dispositivo), UserApp usa `database.ServerValue.TIMESTAMP` (reloj del servidor) en `/usuarios` y `/comentarios` |
| Diagnóstico | Comparar timestamps entre `/ubicacion_buses` y `/usuarios/ultimaConexion` |
| Solución | No requiere acción inmediata. Ambos métodos coexisten porque los paths son distintos. Al unificar tracking, debe elegirse uno |
| Prevención | FIREBASE_SCHEMA.md documenta qué método usa cada nodo |

## 5. Autenticación

### Login de conductor falla con "user-not-found"

| Campo | Descripción |
|-------|-------------|
| Síntoma | El conductor no puede iniciar sesión, Firebase Auth devuelve `auth/user-not-found` |
| Causa | La cuenta `${dni}@burritodriver.com` no existe en Firebase Auth. El administrador no creó el chofer desde el panel, o se ingresó un DNI incorrecto |
| Diagnóstico | Verificar en Firebase Console → Authentication que el email existe |
| Solución | Crear el chofer desde el panel administrativo de UserApp (AsignacionesScreen). El servicio `admin_service.ts` crea Auth + RTDB simultáneamente |
| Prevención | El flujo admin siempre crea Auth con email `${dni}@burritodriver.com` y contraseña = DNI |

### Error al crear chofer desde el panel admin

| Campo | Descripción |
|-------|-------------|
| Síntoma | Al crear un chofer aparece "Error de autenticación" |
| Causa | La instancia secundaria de Firebase Auth (`SecondaryApp`) no se inicializa correctamente |
| Diagnóstico | Revisar logs de Firebase en consola para errores de inicialización |
| Solución | El código ya maneja la inicialización lazy con try/catch. Si persiste, verificar que `google-services.json` esté presente |
| Prevención | `admin_service.ts` usa inicialización lazy: `firebase.initializeApp(config, 'SecondaryApp')` |

### Google Sign-In no funciona en UserApp

| Campo | Descripción |
|-------|-------------|
| Síntoma | Al presionar "Continuar con Google" no sucede nada o muestra error |
| Causa | `GOOGLE_WEB_CLIENT_ID` no configurado en `.env` o el valor es incorrecto |
| Diagnóstico | Verificar que `.env` existe y contiene `GOOGLE_WEB_CLIENT_ID` |
| Solución | Agregar o corregir el `webClientId` en `.env`. El valor se obtiene de Firebase Console → Project Settings → General → Web apps |
| Prevención | README de UserApp documenta las variables de entorno |

## 6. Foreground Service y Android

### InvalidForegroundServiceTypeException (Android 14+)

| Campo | Descripción |
|-------|-------------|
| Síntoma | La DriverApp crashea al iniciar el tracking en Android 14 (API 34) o superior. Logcat muestra `InvalidForegroundServiceTypeException` |
| Causa | `foregroundServiceType` no está declarado en uno de los tres lugares requeridos por Android 14 |
| Diagnóstico | Revisar Logcat en Android Studio: `adb logcat \| grep -i foreground` |
| Solución | Verificar los tres lugares: (1) `AndroidManifest.xml`: `android:foregroundServiceType="location"`, (2) Permisos: `FOREGROUND_SERVICE_LOCATION`, (3) Opciones JS: `foregroundServiceType: ['location' as const]` en `getBackgroundOptions()` |
| Prevención | ARCHITECTURE.md documenta este requisito. Los tres lugares deben mantenerse sincronizados |

### GPS se detiene al apagar la pantalla (Doze Mode)

| Campo | Descripción |
|-------|-------------|
| Síntoma | El marcador del bus se congela después de apagar la pantalla o minimizar la app |
| Causa | Android suspende `watchPosition` porque la app pasó a segundo plano sin foreground service |
| Diagnóstico | Verificar que la notificación persistente "Transmitiendo ubicación" esté visible en la barra de estado |
| Solución | Asegurar que `BackgroundJob.start()` se ejecutó correctamente. Si la notificación no aparece, el permiso `POST_NOTIFICATIONS` puede estar denegado |
| Prevención | El código solicita `POST_NOTIFICATIONS` al iniciar tracking. Si se denegó, abre Settings para que el usuario lo active manualmente |

### Notificación del foreground service no aparece

| Campo | Descripción |
|-------|-------------|
| Síntoma | El conductor no ve la notificación "Transmitiendo ubicación" |
| Causa | Permiso `POST_NOTIFICATIONS` denegado (Android 13+) |
| Diagnóstico | Ir a Settings → Apps → BurritoDriverApp → Notificaciones |
| Solución | El código ya abre Settings automáticamente si el permiso fue denegado. El usuario debe activarlo manualmente |
| Prevención | `SendCoordinates.tsx` maneja este caso con `Linking.openSettings()` |

## 7. Tracking GPS

### watchPosition no dispara callbacks

| Campo | Descripción |
|-------|-------------|
| Síntoma | No aparecen nuevas coordenadas en RTDB a pesar de que el foreground service está activo |
| Causa | GPS desactivado en el dispositivo, permiso `ACCESS_FINE_LOCATION` denegado, o el hardware no tiene señal |
| Diagnóstico | Verificar que el GPS del dispositivo está encendido. Verificar permisos en Settings |
| Solución | Activar GPS. Si el permiso fue denegado, el código aborta el inicio con un alert |
| Prevención | `SendCoordinates.tsx` solicita todos los permisos antes de iniciar el service |

### Coordenadas imprecisas por efecto multipath

| Campo | Descripción |
|-------|-------------|
| Síntoma | La ubicación del bus salta ±10–15 metros en el mapa sin motivo aparente |
| Causa | Los edificios del campus UNMSM causan interferencia GPS (efecto multipath) |
| Diagnóstico | Comparar coordenadas registradas con la posición real del bus |
| Solución | No hay mitigación en el tracking actual. Cuando se implemente geofencing, el umbral de 40 metros absorberá este error |
| Prevención | Limitación conocida documentada en PROJECT_CONTEXT.md |

### Ubicación congelada (stale location)

| Campo | Descripción |
|-------|-------------|
| Síntoma | El bus aparece en el mapa en una posición que no se actualiza |
| Causa | La DriverApp perdió conexión de red, o el conductor cerró la app sin presionar "DETENER" |
| Diagnóstico | Verificar el timestamp de la última actualización en Firebase RTDB para `/ubicacion_buses/{busId}` |
| Solución | La UserApp clasifica el bus como `stopped` si el timestamp tiene más de 12 segundos. Si `isActive === false` en RTDB, lo clasifica como `offline` y no lo muestra |
| Prevención | Timeout check automático (Fase 4 del roadmap) proporcionará una solución más robusta |

## 8. Mapbox

### Mapa en blanco (no carga)

| Campo | Descripción |
|-------|-------------|
| Síntoma | El mapa no carga, se ve gris o blanco |
| Causa | Token de Mapbox inválido, vencido o no configurado en `.env` |
| Diagnóstico | Verificar que `MAPBOX_PUBLIC_TOKEN` existe en `.env` y es un token válido |
| Solución | Configurar un token público de Mapbox válido. El token se asigna en `Map.tsx` mediante `Mapbox.setAccessToken(MAPBOX_PUBLIC_TOKEN)` |
| Prevención | README de UserApp documenta la variable de entorno requerida |

### Bounds no restringen el mapa

| Campo | Descripción |
|-------|-------------|
| Síntoma | El usuario puede desplazarse fuera del campus universitario |
| Causa | Los bounds (`UNMSM_BOUNDS`) están definidos pero no bloquean el desplazamiento forzado de la cámara |
| Diagnóstico | Intentar arrastrar el mapa más allá de los límites del campus |
| Solución | No es un bug. Los bounds actualmente limitan el centro de la cámara pero no restringen el desplazamiento del usuario. Se puede reforzar si es necesario |
| Prevención | — |

## 9. Renderizado y Estado

### Firebase listeners causan re-renders masivos

| Campo | Descripción |
|-------|-------------|
| Síntoma | El mapa tiene baja tasa de FPS, la app se siente lenta |
| Causa | El listener de Firebase RTDB está conectado directamente a un componente de React, forzando re-renders completos en cada latido del GPS |
| Diagnóstico | Verificar que el listener pasa por `burritoLocationStore.ts` (Zustand) y no directamente a `Map.tsx` |
| Solución | Mantener Zustand como buffer entre Firebase y la UI. Es la arquitectura actual del proyecto |
| Prevención | Regla arquitectónica documentada en ARCHITECTURE.md: el listener nunca muta el DOM directamente |

### Hydration gating — la app no avanza del splash

| Campo | Descripción |
|-------|-------------|
| Síntoma | La UserApp se queda en el splash animado y no muestra el mapa ni el login |
| Causa | `userStore` o `themeStore` no completaron la hidratación desde AsyncStorage. El `NavigationContainer` no se renderiza hasta que ambos `_hasHydrated` sean `true` |
| Diagnóstico | Revisar la consola por errores de hidratación o corrupción de AsyncStorage |
| Solución | Forzar reinicio de la app. Si persiste, borrar los datos de AsyncStorage desde Settings → Apps → BurritoUserApp → Storage → Clear data |
| Prevención | App.tsx maneja errores de hidratación y setea `_hasHydrated: true` en el callback `onRehydrateStorage` |

### Dark mode no persiste al cerrar la app

| Campo | Descripción |
|-------|-------------|
| Síntoma | Al abrir la app, el tema vuelve al valor por defecto del sistema aunque se haya cambiado manualmente en la sesión anterior |
| Causa | `themeStore` tiene persistencia manual (AsyncStorage), pero el `CustomDrawer` ejecuta `setTheme(systemColorScheme === 'dark')` en cada montaje, sobrescribiendo el valor persistido |
| Diagnóstico | Revisar `themeStore.ts` y `CustomDrawer.tsx` |
| Solución | Comportamiento esperado. El toggle manual solo aplica dentro de la sesión actual. El tema del sistema prevalece al reiniciar la app |
| Prevención | ARCHITECTURE.md documenta este comportamiento |

## 10. Problemas conocidos

Los siguientes items **no son errores**. Son limitaciones del estado
actual del proyecto, planificadas en fases futuras del roadmap.

| Problema | Estado |
|----------|--------|
| La UserApp muestra un solo bus (multi-bus no implementado) | Planificado (Fase 1) |
| No hay geofencing ni cierre automático de vueltas | Planificado (Fase 2) |
| La UserApp y DriverApp usan paths de tracking distintos (`/ubicacion_burrito` vs `/ubicacion_buses`) | Planificado (Fase 1, migración del listener) |
| El simulador Python es la única fuente de datos para la UserApp en ausencia de DriverApp física | Heredado, dejará de usarse tras Fase 1 |
| iOS no soportado para DriverApp | Limitación de plataforma |
| Test unitario de DriverApp (`__tests__/App.test.tsx`) está roto (hereda de archivo `../App` inexistente) | Por resolver |
| El archivo `gradle.properties` contiene credenciales de release en texto plano (`burrito123`) | Requiere rotación de claves |

## 11. Referencias

| Documento | Relación |
|-----------|----------|
| `PROJECT_CONTEXT.md` | Visión general, limitaciones del sistema. |
| `ARCHITECTURE.md` | Flujo de datos, ciclo de vida del tracking. |
| `FIREBASE_SCHEMA.md` | Estructura de nodos, payloads e índices de la RTDB. |
| `ROADMAP.md` | Fases y prioridades para resolver problemas conocidos. |
| `DECISIONS.md` | Decisiones de arquitectura (persistence, serverless, etc.). |
| BurritoDriverApp/README.md | Setup y requisitos de DriverApp. |
| BurritoUserApp/README.md | Setup y requisitos de UserApp. |
| `BUGS_RESUELTOS/` | Historial técnico de bugs resueltos durante el desarrollo. |
