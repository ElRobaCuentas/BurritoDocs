## CASO 010 — Retroceso por Desorden Cronológico (Firebase Latency) 



Síntoma
El bus "salta" repentinamente a una posición en la que estuvo hace 10 o 20 segundos, especialmente después de pasar por zonas con baja cobertura (como los sótanos de las facultades). 
Diagnóstico
Race conditions y entrega de caché de Firebase. Cuando la señal fallaba y volvía, Firebase Realtime Database entregaba paquetes de datos desordenados o servía el último dato de la caché local antes que el dato fresco de la red. La UserApp aceptaba cualquier dato entrante sin verificar si era "más joven" que el que ya estaba mostrando. . 
Solución
Implementación de un "Filtro de Aduana Temporal" en el Store de Zustand. Se agregó un campo timestamp: Date.now() en la DriverApp y una validación en la UserApp: if (newTimestamp <= currentTimestamp) return;. Ahora la aplicación ignora cualquier dato que no pertenezca al futuro. 
Archivos
SendCoordinates.tsx , burritoLocationStore.ts (UserApp) 
Estado
Resuelto
Lección
En sistemas en tiempo real, el tiempo del servidor no es suficiente si hay modo offline; el cliente debe validar la jerarquía temporal de los datos antes de renderizar. 

