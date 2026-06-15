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

