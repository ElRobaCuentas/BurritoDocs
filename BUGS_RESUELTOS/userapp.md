## CASO 006 — Sesión de Google Auth sin persistencia real ni validación de token
Síntoma
Al cerrar y reabrir la app, un usuario autenticado con Google podía quedar en un estado inconsistente: el teléfono recordaba que estaba logueado (o no), pero nunca se validaba si el token de Firebase seguía activo. Adicionalmente, si un administrador eliminaba la cuenta desde la consola de Firebase, el usuario seguía entrando al mapa con normalidad porque el teléfono no sabía que su sesión había muerto.
Diagnóstico
El problema tenía dos capas. Primera: se confundió "dónde guardar la sesión" — un Context de React no sirve porque muere cuando la app se cierra; se necesita almacenamiento permanente en el teléfono. Segunda: aunque se usara almacenamiento permanente, nunca se cruzaba ese dato local con el estado real de Firebase Auth, creando un "usuario fantasma" que el teléfono creía activo pero Firebase ya no reconocía. El idToken de Google (JWT de 1 hora) no necesita guardarse manualmente — Firebase Auth lo gestiona internamente. Lo que sí necesita guardarse es el estado de sesión del usuario (nombre, avatar, uuid) para no pedirlo en cada arranque.
Solución
Dos piezas complementarias. Primera: userStore.ts con Zustand + middleware persist conectado a AsyncStorage — guarda el estado del usuario en el teléfono de forma permanente y lo recupera instantáneamente al abrir la app, sin depender de red. Segunda: un Watchdog silencioso en App.tsx usando auth().onAuthStateChanged con dependencias vacías [] — lee el estado de Firebase Auth desde la memoria local del dispositivo (sin llamada de red) y, si detecta que Firebase ya no reconoce la sesión pero Zustand todavía dice isLoggedIn: true, ejecuta logout() automáticamente antes de que el usuario llegue al mapa.
Archivo
userStore.ts, App.tsx
Estado
Resuelto
Lección
Un Context de React vive mientras la app está abierta — es memoria volátil. AsyncStorage es el disco duro del teléfono — persiste entre sesiones. Firebase Auth ya gestiona la persistencia del token de Google internamente; no hay que guardarlo manualmente. La arquitectura correcta es: AsyncStorage como caché de arranque rápido (sin red) + Firebase Auth como árbitro final de validez de sesión (local, sin red). Nunca bloquear el splash con llamadas a la base de datos — el watchdog debe correr en segundo plano sin detener la experiencia del usuario.


## CASO 007 — Google Sign-In falla en APK release por SHA-1 no registrado
Síntoma
El inicio de sesión con Google funcionaba perfectamente en el emulador pero fallaba en el APK release instalado en un celular físico. El usuario completaba el flujo de verificación de Google (elegía cuenta, confirmaba identidad), pero al regresar a la app aparecía el mensaje "Error con Google. No se pudo iniciar sesión."
Diagnóstico
Firebase verifica la identidad de la app que solicita autenticación mediante el certificado SHA-1 con el que fue firmada. El emulador usa el debug.keystore de Android — cuyo SHA-1 ya estaba registrado en Firebase. El APK release usa un keystore propio creado para producción — cuyo SHA-1 nunca fue registrado. Para Firebase, el APK release era un desconocido intentando usar su sistema de autenticación, por lo que rechazaba el token aunque Google lo hubiera generado correctamente. El error ocurre en el paso firebaseAuth.signInWithCredential(), no en el flujo de Google.
Solución
Obtener el SHA-1 del keystore de release con keytool -list -v -keystore android/app/burrito-release.keystore -alias burrito, registrarlo en Firebase Console → Project Settings → com.burritouserapp → Add fingerprint, y descargar el google-services.json actualizado para reemplazarlo en android/app/ y recompilar el APK.
Archivo
android/app/google-services.json
Estado
Resuelto
Lección
Existen 3 entornos con SHA-1 distintos que deben registrarse por separado en Firebase: (1) Debug — debug.keystore automático de Android, obtenido con ./gradlew signingReport; (2) Release — el keystore propio del proyecto, obtenido con keytool; (3) Play Store — Google re-firma el APK con su propio certificado al publicarlo, el SHA-1 se obtiene desde Google Play Console → App signing. Los tres deben estar registrados en Firebase antes del lanzamiento. El google-services.json se compila dentro del APK, por lo que cualquier cambio en Firebase que lo afecte requiere recompilar.




## CASO 011 — Colisión de Animaciones (Overlap) 

Síntoma
El movimiento del bus se veía "entrecortado" o pesado. A veces el ícono parecía vibrar intensamente en lugar de deslizarse suavemente.  
Diagnóstico
El intervalo de envío (3s) era igual a la duración de la animación (3s). Si un dato llegaba con un ligero retraso de red, la nueva animación comenzaba antes de que la anterior terminara. Al haber dos animaciones peleando por el mismo valor (latAnim y lngAnim), el bus intentaba ir a dos lugares a la vez. 
Solución
(1) Uso de latAnim.stopAnimation() para matar cualquier movimiento previo antes de iniciar el nuevo. (2) Desincronización de tiempos: Animación de 2000ms para un intervalo de datos de 3000ms, dejando 1 segundo de "respiro" para el motor gráfico. 
Archivos
Map.tsx 
Estado
Resuelto
Lección
La duración de la animación siempre debe ser menor al intervalo de llegada de los datos para evitar la acumulación de frames en el buffer. 











## CASO 012 — SnapToRoute "Ciego" (Vértice vs Segmento) 

Síntoma
El bus se teletransportaba a paraderos anteriores o se deslizaba por la polyline de forma errática cuando el bus real giraba en una esquina.   
Diagnóstico
La función snapToRoute original solo comparaba la posición del bus contra los vértices (puntos fijos) del GeoJSON. Si el bus estaba a mitad de una calle larga, el vértice más cercano podía ser el de atrás, forzando al bus a retroceder visualmente aunque el GPS estuviera bien. 
Solución
Se desactivó temporalmente para depuración de GPS crudo y se determinó que la solución final requiere una proyección geométrica sobre el segmento de línea (polyline), no solo una búsqueda del punto más cercano en el array. 
Archivos
Map.tsx 
Estado
Desactivado / Pendiente
Lección
El "ajuste a la ruta" es peligroso si los datos de entrada tienen ruido. Primero estabiliza el GPS, luego aplica la restricción geométrica.

## CASO 013 — Crash inmediato al iniciar recorrido en Android 14  

Síntoma
Al presionar ‘INICIAR RECORRIDO’, la aplicación se cerraba inmediatamente. Luego de 1-2 segundos me aparecia un popup de que la app no responde (me daba opción para cerrar la app).
  
Diagnóstico
Inicialmente se sospechó de un ANR, pero la instrumentación completa del flujo demostró que el proceso avanzaba correctamente hasta startForeground(). Android 14 rechazaba el servicio porque react-native-background-actions iniciaba el Foreground Service con tipo "none", aunque el Manifest ya declaraba "location". Esto provocaba una InvalidForegroundServiceTypeException y Android cerraba inmediatamente la aplicación. 
Solución
Se agregó foregroundServiceType: ['location' as const] dentro de getBackgroundOptions() para que el tipo enviado en tiempo de ejecución coincidiera con el declarado en AndroidManifest.xml. 
Archivos
SendCoordinates.tsx 
Estado
En Observación 
Lección
En Android 14 no basta con declarar foregroundServiceType en el Manifest. El mismo tipo debe enviarse también al iniciar el Foreground Service; de lo contrario, Android finaliza el proceso aunque todos los permisos estén correctamente configurados. 





