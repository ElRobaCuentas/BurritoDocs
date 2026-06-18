# DOCUMENTACIÓN OFICIAL — EL BURRITO

> **Guion maestro para la elaboración del documento institucional del proyecto.**
> Este archivo contiene el contenido estructurado, listo para ser transferido a un documento Word con diseño, imágenes, tablas y diagramas.

---

## PORTADA

```
────────────────────────────────────────────
                                          (logo)
            EL BURRITO
   Sistema de Seguimiento en Tiempo Real
  para el Transporte Universitario — UNMSM

          Versión 1.0 — MVP Funcional

     Universidad Nacional Mayor de San Marcos
   Facultad de Ciencias Matemáticas — Ciencias
               de la Computación

               Autores:
          [Nombre del autor / equipo]

                Tutor:
           [Nombre del tutor]

           Fecha: Junio 2026
────────────────────────────────────────────
```

---

## EQUIPO DE DESARROLLO

| Campo | Detalle |
|-------|---------|
| **Proyecto** | El Burrito |
| **Versión** | 1.0 — MVP Funcional |
| **Universidad** | Universidad Nacional Mayor de San Marcos (UNMSM) |
| **Facultad** | Ciencias Matemáticas |
| **Carrera** | Ciencias de la Computación |
| **Autores** | [Nombre del autor principal / equipo de desarrollo] |
| **Tutor** | [Nombre del tutor académico] |
| **Fecha** | Junio 2026 |
| **Estado** | MVP Funcional — Tracking operativo, documentación completa |
| **Repositorio** | `C:\ProyectosMovil` |

---

## ÍNDICE

*(Para generar automáticamente en Word con la estructura de títulos)*

1. Introducción
2. Contexto
3. Objetivos
4. El problema
5. Propuesta de solución
6. Ecosistema del sistema
7. Paraderos oficiales
8. Tecnologías
9. Versión actual del proyecto
10. Funcionalidades
11. Arquitectura conceptual
12. Evolución del proyecto
13. Lecciones aprendidas
14. Roadmap
15. Resultados
16. Conclusiones
17. Referencias

---

## 1. INTRODUCCIÓN

### 1.1 Qué es El Burrito

**El Burrito** es un sistema de seguimiento en tiempo real diseñado para los autobuses universitarios de la Universidad Nacional Mayor de San Marcos (UNMSM). Permite a los estudiantes conocer la ubicación exacta de las unidades de transporte interno ("Burritos") desde cualquier dispositivo móvil, eliminando la incertidumbre del tiempo de espera en los paraderos.

El sistema se compone de dos aplicaciones móviles independientes —una para el conductor y otra para el estudiante— que se comunican a través de Firebase Realtime Database, sin necesidad de un backend intermedio.

### 1.2 Por qué existe

El transporte universitario interno de la UNMSM recorre un circuito cerrado dentro de la Ciudad Universitaria, conectando las facultades, puertas de acceso y puntos estratégicos del campus. Sin embargo, los estudiantes no disponían de información sobre la ubicación de los buses ni el tiempo estimado de llegada. Esto generaba:

- Esperas prolongadas sin información.
- Acumulación de estudiantes en ciertos paraderos.
- Desaprovechamiento de la flota por desconocimiento de rutas.
- Incertidumbre generalizada sobre el servicio.

El Burrito nace para resolver esa asimetría de información.

### 1.3 Quiénes participaron

El proyecto fue desarrollado por estudiantes de Ciencias de la Computación de la UNMSM como parte de un proyecto de innovación en movilidad universitaria. Participaron:

- **Desarrollo de software:** [Nombres]
- **Tutoría académica:** [Nombre del tutor]
- **Pruebas en campo:** Conductores de la Oficina de Transportes de la UNMSM
- **Colaboración operativa:** Oficina de Transportes de la UNMSM

### 1.4 Qué problema resuelve

El sistema resuelve el problema de **asimetría de información** en el transporte universitario. Antes del proyecto, los estudiantes esperaban en los paraderos sin saber:

- Dónde estaba el bus en ese momento.
- Cuánto tiempo faltaba para que llegara.
- Si el servicio estaba operativo o suspendido.

El Burrito muestra la posición en vivo de cada unidad sobre un mapa accesible desde el celular, reduciendo la incertidumbre y optimizando los tiempos de espera.

### 1.5 Qué necesidad detectaron

Se detectó que la comunidad estudiantil de la UNMSM (más de 30,000 estudiantes) depende del transporte interno para desplazarse entre facultades, pero la falta de información generaba:

- Pérdida de tiempo por esperas innecesarias.
- Descontento generalizado con el servicio.
- Baja eficiencia en el uso de la flota disponible.
- Dificultad para la toma de decisiones (esperar o caminar).

La necesidad era clara: **una herramienta que muestre dónde está el bus en tiempo real**.

---

## 2. CONTEXTO

### 2.1 La Universidad Nacional Mayor de San Marcos

La UNMSM, fundada en 1551, es la universidad más antigua de América. Su Ciudad Universitaria, ubicada en el distrito de Lima, alberga a más de 30,000 estudiantes distribuidos en 20 facultades. El campus tiene una extensión significativa, con pabellones, clínicas, residencias y áreas verdes distribuidas en un circuito cerrado.

### 2.2 El transporte universitario interno

La universidad cuenta con una flota de autobuses internos, conocidos popularmente como **"Burritos"**, que recorren un circuito circular dentro del campus. Estos buses conectan:

- Las facultades principales (Odontología, Química, Ingeniería, Derecho, etc.).
- Puertas de acceso (Puerta 2, Puerta 7, Puerta de Sistemas).
- Puntos estratégicos (Plaza Cívica, Gimnasio, Clínica Universitaria, Comedor).

El recorrido es continuo durante el horario de atención (aproximadamente 7:00 a 22:00 horas), con conductores que se turnan para mantener el servicio operativo.

### 2.3 Los estudiantes y el problema

Los estudiantes de la UNMSM utilizan el Burrito para trasladarse entre facultades, asistir a clases, ir al comedor o dirigirse a las puertas de salida. El problema principal era:

> **"No saber cuándo va a llegar el próximo bus."**

Un estudiante podía esperar entre 10 y 30 minutos sin tener idea de si el bus estaba por llegar o si acababa de pasar. Esto generaba:

- Llegadas tarde a clases.
- Decisiones apresuradas (caminar vs. esperar).
- Acumulación excesiva en los paraderos más concurridos.
- Frustración y quejas recurrentes.

### 2.4 La incertidumbre como problema de diseño

La incertidumbre no era solo una molestia: era un problema de **asimetría de información**. El conductor sabía dónde estaba, pero el estudiante no. Resolver esa asimetría requería:

- Capturar la ubicación del bus en tiempo real.
- Transmitirla a un servidor central.
- Distribuirla a todos los estudiantes conectados.
- Mostrarla en un mapa intuitivo y accesible.

### 2.5 La motivación

El proyecto surge de la experiencia vivida por los propios estudiantes desarrolladores, que enfrentaban a diario la incertidumbre del transporte universitario. La motivación fue doble:

- **Personal:** resolver un problema real que afecta a decenas de miles de estudiantes todos los días.
- **Técnica:** demostrar que es posible construir un sistema de tracking en tiempo real con tecnologías accesibles (serverless, Firebase, React Native) y hardware reciclado (teléfonos Android dedicados en los buses).

---

## 3. OBJETIVOS

### 3.1 Objetivo general

Desarrollar un sistema de seguimiento en tiempo real para los autobuses universitarios de la UNMSM que permita a los estudiantes conocer la ubicación de las unidades desde sus dispositivos móviles, reduciendo la incertidumbre del tiempo de espera en los paraderos.

### 3.2 Objetivos específicos

1. Implementar una aplicación móvil para conductores que capture y transmita coordenadas GPS en tiempo real, incluso con la pantalla apagada o la aplicación en segundo plano.

2. Implementar una aplicación móvil para estudiantes que consuma las coordenadas de los buses y las muestre sobre un mapa interactivo con los paraderos oficiales.

3. Diseñar un panel administrativo para gestionar conductores, buses y asignaciones diarias sin intervención técnica.

4. Documentar todo el ecosistema para garantizar la continuidad del desarrollo por parte de futuros equipos.

5. Asegurar la compatibilidad con Android 14 y las políticas actuales de Google Play.

### 3.3 Alcance

**Incluye:**
- Aplicación de tracking para conductores (DriverApp) — solo Android.
- Aplicación de visualización para estudiantes (UserApp) — Android e iOS.
- Panel administrativo para gestión de flota (dentro de UserApp).
- Base de datos en tiempo real (Firebase RTDB) como bus de comunicación.
- Mapa interactivo con ruta, paraderos y marcador del bus en vivo.
- Autenticación segmentada (conductores con DNI, estudiantes con email/Google).
- Documentación técnica completa del sistema.

**No incluye (en esta versión):**
- Soporte para múltiples buses simultáneos en el mapa (planificado).
- Dashboard web de monitoreo de flota (planificado).
- Aplicación iOS para conductores (limitación de plataforma).
- Sistema de geofencing automatizado para cierre de vueltas (planificado).
- Notificaciones push (visión futura).

### 3.4 Restricciones

- **Plataforma de conducción:** solo Android. El Foreground Service requerido para mantener el tracking activo depende de APIs nativas de Android no disponibles en iOS.
- **Conectividad:** el sistema requiere cobertura de datos móviles en la ruta del bus. Sin conexión, las coordenadas no se transmiten.
- **Precisión GPS:** el campus de la UNMSM produce interferencia estructural (efecto multipath) que genera un margen de error de ±10–15 metros. Esto limita la precisión del geofencing.
- **Costo:** el proyecto opera bajo el Plan Spark de Firebase (gratuito), con límites de 100 conexiones simultáneas y 10 GB de descarga mensual.
- **Hardware:** los dispositivos emisores son teléfonos Android reciclados instalados en los buses, no hardware GPS especializado.

---

## 4. EL PROBLEMA

### 4.1 El escenario antes de El Burrito

**Un día típico de un estudiante de la UNMSM:**

1. Termina su clase en la Facultad de Ingeniería Industrial.
2. Necesita ir a la Facultad de Derecho para la siguiente clase.
3. Camina al paradero más cercano.
4. **Espera. Sin saber cuánto falta.**

El estudiante no tiene forma de saber:
- Si el bus ya pasó hace 2 minutos.
- Si está por llegar en 1 minuto.
- Si el servicio está suspendido por algún motivo.
- Si es mejor esperar o caminar los 15 minutos hasta el otro pabellón.

El resultado: **incertidumbre total**. El estudiante puede esperar 5 minutos o 30, sin diferencia visual. No hay información.

### 4.2 El impacto del problema

- **Tiempo perdido:** los estudiantes pierden entre 10 y 30 minutos por espera, varias veces al día.
- **Llegadas tarde:** la incertidumbre provoca que los estudiantes lleguen tarde a clases o salgan antes para "no arriesgarse".
- **Sobrecarga de paraderos:** sin información, los estudiantes se acumulan en los paraderos más conocidos, generando aglomeraciones.
- **Descontento:** el servicio es gratuito y necesario, pero la falta de información lo hace impredecible y frustrante.
- **Ineficiencia:** la flota podría aprovecharse mejor si los estudiantes distribuyeran su uso a lo largo del día.

### 4.3 ¿Por qué no existía una solución antes?

- **Costo:** implementar un sistema de tracking GPS con backend propio requiere inversión en servidores, desarrollo y mantenimiento.
- **Complejidad:** el tracking en tiempo real con teléfonos Android tiene desafíos técnicos (permisos, foreground services, Doze Mode).
- **Coordinación:** involucra a la universidad, los conductores, los estudiantes y los desarrolladores.
- **Viabilidad:** hasta hace pocos años, Firebase y React Native no estaban lo suficientemente maduros para este tipo de solución.

---

## 5. PROPUESTA DE SOLUCIÓN

### 5.1 La idea central

**Poner un teléfono Android en cada bus y una aplicación en el celular de cada estudiante.**

El teléfono del bus captura su ubicación GPS constantemente. Esa ubicación se envía a un servidor en la nube. Los estudiantes se conectan a ese servidor y ven la posición del bus en un mapa.

### 5.2 Cómo funciona (visión conceptual)

```
Conductor en el bus
        │
        ▼
Teléfono Android captura GPS
        │
        ▼
La ubicación se envía a la nube
        │
        ▼
El estudiante consulta desde su celular
        │
        ▼
Vea el bus moviéndose en el mapa
```

No hay complicación técnica visible para el usuario. El conductor solo debe iniciar y detener su turno. El estudiante solo abre la aplicación y ve el mapa.

### 5.3 Principios de diseño

- **Simplicidad para el conductor:** la DriverApp tiene una sola pantalla. El conductor inicia sesión con su DNI y presiona "INICIAR". No necesita configurar nada más.
- **Claridad para el estudiante:** la UserApp muestra un mapa con el bus, la ruta y los paraderos. El estudiante sabe al instante dónde está el bus.
- **Sin backend propio:** Firebase Realtime Database actúa como servidor, eliminando la necesidad de mantener infraestructura propia.
- **Costo mínimo:** se usan teléfonos Android reciclados como emisores GPS. Firebase Spark Plan es gratuito para el volumen actual de usuarios.
- **Escalable:** cuando el sistema crezca, se puede migrar a planes de pago de Firebase o a un backend propio sin cambiar la arquitectura de las aplicaciones.

---

## 6. ECOSISTEMA DEL SISTEMA

### 6.1 Diagrama general

```
                 ┌──────────────────────────────────────┐
                 │           EL BURRITO                  │
                 │   Sistema de Tracking Universitario   │
                 └──────────────────────────────────────┘

                      ┌──────────────────┐
                      │                  │
                      │   BURRITO        │
                      │   DRIVER APP     │
                      │   (Conductor)    │
                      │                  │
                      └────────┬─────────┘
                               │
                          Envía GPS
                          cada ~3s
                               │
                               ▼
                    ┌─────────────────────┐
                    │                     │
                    │   FIREBASE RTDB     │
                    │   (Servidor en      │
                    │    la nube)          │
                    │                     │
                    └─────────────────────┘
                      ▲               ▲
                      │               │
                      │               │
            ┌─────────┴──┐      ┌─────┴──────────┐
            │             │      │                │
            │  BURRITO    │      │  PANEL         │
            │  USER APP   │      │  ADMINISTRATIVO│
            │  (Estudiante)│     │  (Oficina)     │
            │             │      │                │
            └─────────────┘      └────────────────┘
```

### 6.2 Los actores del sistema

| Actor | Aplicación | Dispositivo | Rol |
|-------|-----------|-------------|-----|
| **Conductor** | BurritoDriverApp | Teléfono Android en el bus | Captura y envía GPS |
| **Estudiante** | BurritoUserApp | Celular personal | Consulta ubicación en mapa |
| **Administrador** | BurritoUserApp (panel) | Celular personal | Gestiona conductores, buses y asignaciones |
| **Servidor** | Firebase RTDB | Nube (Google) | Almacena y distribuye datos en tiempo real |

### 6.3 Flujo de información

1. El **conductor** inicia sesión con su DNI en la DriverApp.
2. La DriverApp consulta qué bus tiene asignado para hoy.
3. Al presionar "INICIAR", el teléfono comienza a capturar la ubicación GPS.
4. Cada 3 segundos, la ubicación se envía a **Firebase RTDB**.
5. La **UserApp** del estudiante escucha los cambios en Firebase.
6. Cuando llega una nueva ubicación, el mapa actualiza la posición del bus.
7. El **administrador** puede crear conductores, registrar buses y asignar turnos desde el panel de gestión.

### 6.4 Separación de responsabilidades

- **DriverApp:** solo escribe. No muestra mapa ni información de estudiantes.
- **UserApp:** solo lee (tracking). No captura GPS ni escribe ubicaciones.
- **Firebase:** actúa como intermediario. No ejecuta lógica de negocio.
- **Administrador:** gestiona los datos estáticos (choferes, buses, turnos).

---

## 7. PARADEROS OFICIALES

### 7.1 La ruta del Burrito

El circuito del transporte universitario interno de la UNMSM recorre la Ciudad Universitaria en un trazado circular que conecta las facultades, puertas de acceso y puntos estratégicos. El recorrido completo tiene aproximadamente 2.5 km y cuenta con **10 paraderos oficiales**.

*(Aquí debe insertarse un mapa/imagen del recorrido con los paraderos marcados)*

### 7.2 Tabla de paraderos oficiales

| # | Paradero | Facultad / Ubicación | Descripción |
|---|----------|---------------------|-------------|
| 01 | **Paradero 01** | Odontología | Punto de inicio de ruta y relevo de conductores |
| 02 | **Paradero 02** | Plaza Cívica / Residencia | Cerca de la residencia universitaria |
| 03 | **Paradero 03** | Gimnasio | Acceso al complejo deportivo |
| 04 | **Paradero 04** | Facultad de Química / Comedor | Zona de alta concurrencia estudiantil |
| 05 | **Paradero 05** | Facultad de Ingeniería Industrial | Facultad con mayor densidad de estudiantes |
| 06 | **Paradero 06** | Puerta 2 | Acceso vehicular principal |
| 07 | **Paradero 07** | Facultad de Derecho / Economía | Zona de facultades de ciencias sociales |
| 08 | **Paradero 08** | Clínica Universitaria | Acceso a la clínica y bienestar universitario |
| 09 | **Paradero 09** | Puerta 7 | Acceso secundario |
| 10 | **Paradero 10** | Puerta Sistemas | Acceso a la Facultad de Ciencias Matemáticas |

### 7.3 Recorrido

*(Aquí debe insertarse una imagen con la ruta dibujada sobre el mapa de la UNMSM, con los 10 paraderos numerados)*

El recorrido es circular y continuo. El bus inicia en Odontología (Paradero 01) y recorre los 10 paraderos en orden numérico. Al completar el paradero 10, el bus regresa al paradero 01 y reinicia el circuito. No hay un punto de destino final — el servicio es un loop continuo.

### 7.4 Datos de la ruta

| Aspecto | Valor |
|---------|-------|
| Longitud aproximada | 2.5 km |
| Número de paraderos | 10 |
| Tiempo estimado por vuelta | 15–25 min (dependiendo del tráfico interno) |
| Horario de servicio | 7:00 – 22:00 |
| Tipo de recorrido | Circular continuo |

---

## 8. TECNOLOGÍAS

### 8.1 Stack tecnológico

| Tecnología | Versión | Uso en el proyecto |
|------------|---------|---------------------|
| **React Native** | 0.83.x | Framework de desarrollo móvil para ambas aplicaciones |
| **TypeScript** | 5.x | Lenguaje de programación con tipado estricto |
| **Firebase Realtime Database** | — | Base de datos en tiempo real y bus de comunicación |
| **Firebase Auth** | — | Autenticación de conductores y estudiantes |
| **Mapbox** | v10 (rnmapbox) | Motor de mapas nativos para renderizado geoespacial |
| **Zustand** | 4.x | Manejo de estado global en ambas aplicaciones |
| **React Navigation** | 6.x | Navegación entre pantallas (UserApp) |
| **react-native-background-actions** | — | Foreground Service para tracking GPS en segundo plano |
| **@react-native-community/geolocation** | — | API de geolocalización nativa |

### 8.2 Arquitectura de infraestructura

| Componente | Tecnología | Propósito |
|------------|-----------|-----------|
| Frontend móvil | React Native CLI | Aplicaciones nativas para Android (y iOS en UserApp) |
| Backend | Firebase (serverless) | Sin servidores propios. Firebase RTDB + Auth |
| Base de datos | Firebase RTDB | Almacenamiento NoSQL en tiempo real |
| Mapas | Mapbox | Renderizado nativo de mapas, ruta y marcadores |
| Estado | Zustand | Gestión de estado global desacoplada de la UI |

### 8.3 ¿Por qué estas tecnologías?

- **React Native:** permite desarrollar para Android e iOS con una sola base de código, acelerando el desarrollo.
- **Firebase:** elimina la necesidad de gestionar servidores propios. La Realtime Database ofrece sincronización nativa mediante websockets, ideal para tracking en tiempo real.
- **Mapbox:** renderizado nativo de mapas con alto rendimiento (60 FPS), soporte para GeoJSON y personalización completa de capas.
- **Zustand:** liviano, sin boilerplate, con actualizaciones fuera del flujo de React para evitar re-renders masivos.
- **TypeScript:** tipado estricto que reduce errores en tiempo de ejecución y mejora la mantenibilidad del código.

---

## 9. VERSIÓN ACTUAL DEL PROYECTO

### 9.1 Estado general

El proyecto se encuentra en su **versión 1.0 — MVP Funcional**. La infraestructura principal está operativa y documentada. Las aplicaciones están listas para pruebas en entorno real con conductores y estudiantes.

### 9.2 Qué está funcionando

| Funcionalidad | Estado | Detalle |
|---------------|--------|---------|
| Tracking GPS en tiempo real | ✅ Completo | Captura y envío de coordenadas cada ~3s con Foreground Service |
| Login de conductores | ✅ Completo | Autenticación con DNI + contraseña sobre Firebase Auth |
| Login de estudiantes | ✅ Completo | Email/contraseña y Google Sign-In |
| Panel administrativo | ✅ Completo | CRUD de conductores, buses y asignaciones diarias |
| Mapa en vivo (UserApp) | ✅ Completo | Bus animado sobre ruta con paraderos y modo oscuro |
| Asignación automática de bus | ✅ Completo | La DriverApp consulta el bus asignado para hoy |
| Compatibilidad Android 14 | ✅ Completo | Foreground Service con tipo location, 3 permisos en runtime |
| Dark mode | ✅ Completo | Modo oscuro/claro con persistencia en AsyncStorage |
| Documentación técnica | ✅ Completo | 8 documentos + READMEs + AGENTS en `docs/` |
| Firebase Auth + RTDB | ✅ Completo | Backend serverless operativo |
| Feedback de estudiantes | ✅ Completo | Formulario de comentarios con rating |

### 9.3 Qué quedó fuera del MVP

| Funcionalidad | Estado | Planificado en |
|---------------|--------|----------------|
| Múltiples buses en el mapa | ❌ Pendiente | Roadmap — Fase Multi-bus |
| Geofencing y cierre automático de vueltas | ❌ Pendiente | Roadmap — Fase Geofencing |
| Dashboard web de monitoreo | ❌ Pendiente | Visión a futuro |
| Notificaciones push | ❌ Pendiente | Visión a futuro |
| Soporte iOS para DriverApp | ❌ Pendiente | Sin fecha |
| Estadísticas de recorridos | ❌ Pendiente | Roadmap — Fase Monitoreo |
| Calibración GPS de paraderos en campo | ❌ Pendiente | Roadmap — Fase Calibración |

### 9.4 Limitaciones conocidas

- **Un solo bus activo:** actualmente la aplicación del estudiante solo puede mostrar un bus a la vez. La arquitectura está preparada para múltiples buses, pero la funcionalidad no está implementada.
- **Dependencia de red móvil:** el tracking requiere conexión a internet. Sin datos móviles, el bus no transmite su ubicación.
- **Precisión GPS limitada:** el efecto multipath en el campus genera un margen de error de ±10–15 metros.
- **DriverApp solo Android:** el Foreground Service depende de APIs nativas de Android. No hay soporte para iOS en la aplicación del conductor.
- **Cobertura Firebase Spark:** el plan gratuito tiene límite de 100 conexiones simultáneas, suficiente para pruebas pero restrictivo para producción masiva.

---

## 10. FUNCIONALIDADES

### 10.1 Funcionalidades del estudiante (UserApp)

| Funcionalidad | Descripción |
|---------------|-------------|
| **Ver mapa en vivo** | Mapa interactivo con la ruta del bus, los 10 paraderos y el marcador del bus moviéndose en tiempo real |
| **Iniciar sesión** | Registro e inicio de sesión con email/contraseña o Google Sign-In |
| **Seleccionar avatar de facultad** | Avatar personalizado según la facultad (Ingeniero, Salud, Economista, Humanidades) |
| **Modo oscuro / claro** | Alternar entre tema oscuro y claro, con persistencia entre sesiones |
| **Seguir al bus** | Botón que centra la cámara en el bus y la mantiene siguiéndolo |
| **Ver información de paraderos** | Al tocar un paradero, ver su nombre, número y distancia estimada al bus |
| **Indicador de señal** | Indicador visual del estado de la conexión: estable, débil o perdida |
| **Enviar feedback** | Formulario de comentarios con calificación y mensaje |
| **Cerrar sesión** | Botón de cierre de sesión en el menú lateral |

### 10.2 Funcionalidades del conductor (DriverApp)

| Funcionalidad | Descripción |
|---------------|-------------|
| **Iniciar sesión con DNI** | Login simple con número de DNI y contraseña |
| **Asignación automática** | La app detecta automáticamente qué bus tiene asignado el conductor para hoy |
| **Iniciar tracking** | Botón que activa la captura y transmisión de coordenadas GPS |
| **Foreground Service** | El tracking continúa aunque la pantalla esté apagada o la app minimizada |
| **Consola de eventos** | Interfaz de diagnóstico que muestra los últimos eventos del motor de tracking |
| **Detener tracking** | Botón que finaliza el turno, detiene el GPS y marca el bus como fuera de servicio |
| **Heartbeat** | Señal periódica que indica que el bus sigue operativo incluso si no se mueve |

### 10.3 Funcionalidades del administrador

| Funcionalidad | Descripción |
|---------------|-------------|
| **Gestionar conductores** | Crear, listar, activar/desactivar conductores. Cada conductor tiene su cuenta de Firebase Auth |
| **Gestionar buses** | Registrar buses con placa, modelo, marca y año |
| **Crear asignaciones diarias** | Vincular un conductor con un bus para un día específico |
| **Validación de exclusividad** | El sistema evita asignar un conductor o un bus a dos turnos el mismo día |
| **Cancelar asignaciones** | Desactivar una asignación sin eliminar el registro histórico |

---

## 11. ARQUITECTURA CONCEPTUAL

### 11.1 Flujo general del sistema

```
CONDUCTOR
    │
    ▼
UBICACIÓN GPS (cada ~3 segundos)
    │
    ▼
DRIVERAPP (teléfono en el bus)
    │
    ▼
FIREBASE (servidor en la nube)
    │
    ▼
USERAPP (celular del estudiante)
    │
    ▼
ESTUDIANTE (ve el bus en el mapa)
```

### 11.2 Descripción del flujo

1. **El conductor** inicia sesión con su DNI en la DriverApp. La app identifica automáticamente qué bus le fue asignado para hoy.

2. **Al presionar "INICIAR"**, el teléfono activa el GPS y comienza a capturar la ubicación cada 3 segundos. Un servicio especial (Foreground Service) mantiene el GPS activo incluso si el conductor apaga la pantalla o abre otra aplicación.

3. **Cada coordenada** (latitud, longitud, dirección, velocidad) se envía inmediatamente a Firebase Realtime Database, un servidor en la nube de Google.

4. **La aplicación del estudiante** está permanentemente conectada a Firebase. Cuando llega una nueva coordenada, Firebase la "empuja" automáticamente a todos los estudiantes conectados.

5. **El estudiante** ve el bus moviéndose en tiempo real sobre un mapa de Mapbox. También puede ver la ruta, los paraderos y el estado del servicio.

### 11.3 Principios arquitectónicos

- **Flujo unidireccional:** la información viaja en una sola dirección: conductor → servidor → estudiante. No hay comunicación directa entre dispositivos.
- **Serverless:** no hay un backend propio. Firebase actúa como servidor sin necesidad de administrar infraestructura.
- **Desacoplamiento:** las aplicaciones del conductor y del estudiante son proyectos independientes. Cada una puede evolucionar sin afectar a la otra.
- **Productor-Consumidor:** la DriverApp escribe datos; la UserApp los lee. Ninguna aplicación hace ambas cosas.

---

## 12. EVOLUCIÓN DEL PROYECTO

### 12.1 Línea de tiempo

```
2025 ──── v0: Idea
           │
           ├── Identificación del problema
           ├── Primeros bocetos de la solución
           └── Elección del stack tecnológico

2025 ──── v1: Primer mapa
           │
           ├── App básica con mapa estático
           ├── Paraderos dibujados manualmente
           └── Sin conexión a Firebase

2025 ──── v2: Tracking en tiempo real
           │
           ├── Firebase integrado
           ├── DriverApp envía GPS
           ├── UserApp recibe y muestra coordenadas
           └── Primeras pruebas con un bus real

2025 ──── v3: Panel administrativo
           │
           ├── CRUD de conductores
           ├── CRUD de buses
           ├── Sistema de asignaciones diarias
           └── Login con DNI para conductores

2025-2026 ─ v4: Documentación
           │
           ├── Arquitectura documentada
           ├── Esquema Firebase documentado
           ├── Roadmap definido
           ├── ADR registrados
           ├── Bugs documentados
           └── Guías para IA (AGENTS)

2026 ──── v5: Versión actual (MVP)
           │
           ├── Tracking estable (Android 14)
           ├── Dark mode
           ├── Documentación completa
           ├── Pruebas en campo
           └── Preparación para Play Store
```

### 12.2 Versiones del proyecto

| Versión | Fecha | Hito | Estado |
|---------|-------|------|--------|
| **v0 — Idea** | 2025 | Concepto inicial, identificación del problema | Completado |
| **v1 — Mapa** | 2025 | Aplicación con mapa y paraderos estáticos | Completado |
| **v2 — Tracking** | 2025 | Transmisión y recepción de GPS en tiempo real | Completado |
| **v3 — Admin** | 2025 | Panel de gestión con CRUD y asignaciones | Completado |
| **v4 — Documentación** | 2025–2026 | Documentación técnica completa del ecosistema | Completado |
| **v5 — MVP Actual** | 2026 | Tracking estable, Android 14+, dark mode, documentación | **Versión actual** |
| **v6 — Multi-bus** | Próximo | Múltiples buses simultáneos en el mapa | Planificado |
| **v7 — Geofencing** | Futuro | Cierre automático de vueltas con Punto Cero | Planificado |
| **v8 — Dashboard** | Futuro | Monitoreo web y estadísticas de flota | Visión |

### 12.3 Aprendizaje por versión

- **v0 → v1:** aprendimos que identificar el problema correcto (incertidumbre, no transporte) define la solución.
- **v1 → v2:** aprendimos que el tracking en segundo plano requiere soluciones nativas (Foreground Service), no solo código JavaScript.
- **v2 → v3:** aprendimos que los conductores no deben configurar nada — la asignación debe ser automática.
- **v3 → v4:** aprendimos que la documentación no es un extra, es parte del producto. Sin ella, el conocimiento se pierde.
- **v4 → v5:** aprendimos que la IA acelera el desarrollo, pero necesita documentación precisa para ser efectiva.

---

## 13. LECCIONES APRENDIDAS

### 13.1 Lecciones técnicas

**1. Android 14 cambió completamente los Foreground Services**

Con Android 14 (API 34), declarar `foregroundServiceType="location"` en el Manifest ya no es suficiente. El mismo tipo debe declararse también al iniciar el servicio desde JavaScript. Si los tres lugares (Manifest, permisos, opciones JS) no están sincronizados, Android rechaza el servicio y la aplicación se cierra inmediatamente.

*Aprendizaje:* en Android moderno, los permisos y servicios deben declararse en múltiples capas. Una sola declaración ya no garantiza el funcionamiento.

---

**2. Firebase Persistence no es adecuado para tracking en tiempo real**

La persistencia local de Firebase (`setPersistenceEnabled(true)`) está diseñada para aplicaciones offline como listas de tareas o notas. En un tracker en tiempo real, acumula todas las coordenadas fallidas durante una pérdida de señal y las reenvía en ráfaga al reconectar. Esto hace que el bus "viaje en el tiempo" en el mapa.

*Aprendizaje:* no todas las funcionalidades de Firebase son compatibles con todos los casos de uso. La persistencia offline es dañina para tracking en tiempo real.

---

**3. `watchPosition` (streaming) es superior a `getCurrentPositionAsync` (polling)**

El uso de `getCurrentPositionAsync` dentro de un bucle while provocaba que el chip GPS del teléfono se "apagara y encendiera" constantemente, impidiendo que aplicara sus filtros internos de suavizado (Kalman). Esto generaba lecturas erráticas que hacían que el bus pareciera avanzar y retroceder (efecto yo-yo).

*Aprendizaje:* para seguimiento continuo, el GPS necesita permanecer activo. Usar `watchPosition` como streaming continuo, no como peticiones discretas.

---

**4. Un Foreground Service y una Activity no sobreviven independientemente por defecto**

Si no se declara `android:stopWithTask="false"` en el Manifest, Android asume que el servicio debe morir cuando la actividad principal es destruida. Esto significaba que si el conductor deslizaba la aplicación fuera del administrador de tareas, el tracking se detenía aunque el Foreground Service estuviera activo.

*Aprendizaje:* en Android, un Foreground Service debe declararse explícitamente como independiente de la UI para sobrevivir al cierre de la app.

---

**5. El GPS tiene un margen de error que debe ser absorbido por el diseño**

En el campus de la UNMSM, los edificios causan rebote de señal GPS (efecto multipath) con un margen de error de ±10–15 metros. Si el sistema usa radios de detección pequeños (ej. 20 metros), el geofencing genera falsos positivos. Con 40 metros de radio, el error se absorbe sin afectar la funcionalidad.

*Aprendizaje:* en sistemas geoespaciales con hardware móvil estándar, el diseño debe contemplar el error del sensor como parte de la solución, no como una limitación a resolver después.

---

**6. Existen tres entornos con SHA-1 distintos y deben registrarse por separado**

Firebase verifica la identidad de la aplicación mediante el certificado SHA-1 con el que fue firmada. El emulador usa un keystore de depuración, el APK release usa el keystore de producción, y Google Play Store vuelve a firmar el APK con su propio certificado. Si alguno de estos SHA-1 no está registrado en Firebase, la autenticación falla silenciosamente.

*Aprendizaje:* los tres entornos (debug, release y Play Store) deben registrarse por separado en Firebase Console antes del lanzamiento.

### 13.2 Lecciones del proyecto

**7. Documentar desde el inicio reduce la deuda técnica**

Al principio del proyecto, la documentación era mínima. Esto provocaba que retomar el desarrollo después de una pausa requiriera leer todo el código nuevamente. La creación de documentos como PROJECT_CONTEXT, ARCHITECTURE y DECISIONS redujo drásticamente el tiempo de reinicio.

*Aprendizaje:* la documentación no es un lujo. Es una inversión que se amortiza cada vez que alguien (incluyéndote a ti mismo en el futuro) necesita entender el proyecto.

---

**8. Separar DriverApp y UserApp simplifica la arquitectura**

Tener dos aplicaciones independientes en lugar de una app con dos modos permite que cada una evolucione a su propio ritmo. La DriverApp puede enfocarse exclusivamente en Android y tracking, mientras que la UserApp puede soportar iOS y tener una interfaz más compleja.

*Aprendizaje:* la separación de responsabilidades no es solo para el código. Aplica también a nivel de productos y aplicaciones.

---

**9. Firebase RTDB funciona muy bien para tiempo real si se usa correctamente**

Firebase Realtime Database demostró ser una plataforma sólida para tracking en tiempo real, siempre que se respeten sus reglas: desactivar persistencia, usar timestamps del cliente con validación de orden, y mantener la estructura de datos plana.

*Aprendizaje:* Firebase no es mágico. Funciona bien si se entienden sus límites y se diseña la arquitectura en torno a ellos.

---

**10. La IA acelera el desarrollo pero necesita documentación precisa**

El uso de asistentes de IA (OpenCode, Claude, Gemini, Cursor) para generar y mantener código demostró ser altamente productivo, pero requiere que el proyecto tenga documentación precisa. Sin AGENTS.md que especifique reglas críticas y sin documentos de referencia (ARCHITECTURE, FIREBASE_SCHEMA), la IA genera código inconsistente o incorrecto.

*Aprendizaje:* la documentación es el puente entre el desarrollador humano y la IA. Sin documentación, la IA es impredecible. Con documentación, la IA se convierte en un miembro productivo del equipo.

---

## 14. ROADMAP

### 14.1 Visión general

El proyecto se encuentra en su **MVP Funcional (v5)**. Las siguientes versiones están planificadas en tres fases, cada una con dependencias claras.

### 14.2 Versiones planificadas

| Versión | Nombre | Estado | Objetivo |
|---------|--------|--------|----------|
| **v5** | MVP Actual | ✅ Completado | Tracking funcional, documentación completa |
| **v6** | Multi-bus | 🔄 Siguiente | Soportar múltiples buses simultáneos en el mapa |
| **v7** | Geofencing | ⏳ Futuro | Cierre automático de vueltas con Punto Cero dinámico |
| **v8** | Dashboard | 🔭 Visión | Monitoreo web y estadísticas de flota |

### 14.3 Fase actual y siguiente

**Versión actual (v5 — MVP):**
- Tracking GPS en tiempo real ✅
- Autenticación segmentada ✅
- Panel administrativo ✅
- Documentación técnica completa ✅
- Dark mode ✅
- Android 14 compatible ✅

**Próxima versión (v6 — Multi-bus):**
- Migrar la aplicación del estudiante al nodo multi-bus
- Renderizar múltiples buses en el mapa
- Unificar las fuentes de datos de tracking

### 14.4 Visión a largo plazo

- Dashboard web para administración de flota
- Backend propio (Node.js) para lógica de negocio compleja
- API pública para integraciones externas
- Analítica de tiempos de espera, ocupación y rutas
- Notificaciones push a estudiantes
- Aplicación iOS para conductores

---

## 15. RESULTADOS

### 15.1 Indicadores del proyecto

| Indicador | Valor |
|-----------|-------|
| **Aplicaciones desarrolladas** | 2 (DriverApp + UserApp) |
| **Documentos técnicos** | 14 (docs/ + READMEs + AGENTS) |
| **Bugs documentados** | 13 (en BUGS_RESUELTOS/) |
| **Decisiones de arquitectura (ADR)** | 12 (en DECISIONS.md) |
| **Paraderos mapeados** | 10 oficiales |
| **Plataformas soportadas** | Android (ambas), iOS (UserApp) |
| **Versión de Android compatible** | API 34 (Android 14) |
| **Tecnologías integradas** | 6 (React Native, Firebase, Mapbox, Zustand, TypeScript, React Navigation) |
| **Permisos Android gestionados** | 6 (POST_NOTIFICATIONS, 2x Location, Foreground Service, etc.) |
| **Líneas de documentación** | ~4,000+ (incluyendo todos los documentos) |

### 15.2 Logros del proyecto

- ✅ **Tracking en tiempo real** funcionando con Foreground Service, incluso con la pantalla apagada.
- ✅ **Dos aplicaciones móviles independientes** con autenticación segmentada (conductores con DNI, estudiantes con email/Google).
- ✅ **Panel administrativo completo** para gestionar conductores, buses y asignaciones diarias.
- ✅ **Mapbox integrado** con ruta, paraderos y marcador del bus animado a 60 FPS.
- ✅ **Modo oscuro** con persistencia entre sesiones.
- ✅ **Android 14 compatible**, incluyendo los nuevos requisitos de Foreground Service tipo location.
- ✅ **Firebase Realtime Database** como bus de comunicación serverless.
- ✅ **Documentación técnica completa** del ecosistema (arquitectura, esquema, roadmap, ADR, troubleshooting).
- ✅ **Arquitectura preparada para escalar** a múltiples buses, backend propio y dashboard web.

### 15.3 Impacto

El Burrito demuestra que es posible construir un sistema de tracking universitario en tiempo real con:

- **Costo cero en infraestructura** (Firebase Spark Plan + hardware reciclado).
- **Tecnología accesible** (React Native, TypeScript, Firebase).
- **Arquitectura profesional** (serverless, desacoplada, documentada).
- **Colaboración humano-IA** (documentación como puente entre desarrolladores y asistentes de IA).

---

## 16. CONCLUSIONES

### 16.1 Lo que se logró

El proyecto **"El Burrito"** alcanzó su objetivo principal: construir un sistema de seguimiento en tiempo real para los autobuses universitarios de la UNMSM que reduce la incertidumbre del tiempo de espera de los estudiantes.

Se logró:
- **Reducir la asimetría de información** en el transporte universitario. Ahora los estudiantes pueden ver dónde está el bus en tiempo real, desde cualquier dispositivo móvil.
- **Construir un ecosistema completo de aplicaciones** (DriverApp, UserApp, panel administrativo) que cubre las necesidades de todos los actores: conductores, estudiantes y administradores.
- **Demostrar que es posible hacer tracking profesional con tecnologías serverless y hardware reciclado**, sin necesidad de inversión en infraestructura propia.
- **Documentar todo el sistema** de forma que cualquier desarrollador (humano o IA) pueda entenderlo y continuar su desarrollo.

### 16.2 El estado del proyecto

El proyecto se encuentra en un **MVP funcional y estable**. La infraestructura principal (tracking, autenticación, mapa, administración) está operativa y probada. La documentación técnica está completa y permite la continuidad del desarrollo.

El proyecto está preparado para:
- **Escalar** a múltiples buses (planificado en la siguiente fase).
- **Publicarse** en Google Play (requiere completar los pasos de preparación para producción).
- **Ser mantenido** por nuevos desarrolladores gracias a la documentación existente.

### 16.3 Reflexión final

El Burrito no es solo un proyecto de software. Es una intervención operativa en el transporte universitario que demuestra cómo la tecnología móvil, el cómputo serverless y las metodologías modernas de desarrollo (incluyendo la colaboración con IA) pueden resolver problemas reales de la comunidad universitaria.

El proyecto queda abierto para futuras iteraciones, mejoras y expansiones. La base está construida, documentada y lista para evolucionar.

---

## 17. REFERENCIAS

### 17.1 Documentación técnica

La documentación técnica detallada del proyecto se encuentra en la carpeta `docs/` del repositorio. Cada documento cubre un aspecto específico del sistema:

| Documento | Propósito |
|-----------|-----------|
| `docs/PROJECT_CONTEXT.md` | Visión general del sistema, propósito, estado actual y limitaciones |
| `docs/ARCHITECTURE.md` | Flujo de datos, ciclo de vida, componentes y servicios |
| `docs/FIREBASE_SCHEMA.md` | Estructura de nodos, payloads, índices y reglas de RTDB |
| `docs/ROADMAP.md` | Fases, tareas, dependencias y visión a largo plazo |
| `docs/DECISIONS.md` | Registro de decisiones de arquitectura (ADR) con razonamiento técnico |
| `docs/TROUBLESHOOTING.md` | Problemas conocidos, síntomas, causas y soluciones |
| `docs/BUGS_RESUELTOS/` | Historial detallado de bugs resueltos por aplicación |
| `docs/ReviewNotes.md` | Notas de revisión futura para mantener la documentación actualizada |
| `BurritoUserApp/README.md` | Setup, stack y funcionalidad de la aplicación del estudiante |
| `BurritoDriverApp/README.md` | Setup, requisitos de plataforma y flujo de tracking |
| `BurritoUserApp/AGENTS.md` | Reglas de trabajo para IA en la aplicación del estudiante |
| `BurritoDriverApp/AGENTS.md` | Reglas de trabajo para IA en la aplicación del conductor |

### 17.2 Herramientas de desarrollo

| Herramienta | Uso |
|-------------|-----|
| React Native CLI | Framework de desarrollo móvil |
| Firebase Console | Gestión de base de datos, autenticación y reglas |
| Mapbox Studio | Configuración de estilos de mapa |
| Android Studio | Depuración y compilación nativa |
| Visual Studio Code | Editor de código |
| OpenCode / Claude / Gemini | Asistentes de IA para desarrollo y documentación |
| Git / GitHub | Control de versiones (local) |

### 17.3 Recursos externos

- Firebase Realtime Database: https://firebase.google.com/docs/database
- React Native: https://reactnative.dev
- Mapbox (rnmapbox): https://github.com/rnmapbox/maps
- Zustand: https://github.com/pmndrs/zustand

---

> **Fin del guion maestro.**
>
> Este documento será transformado en un documento Word institucional con portada, imágenes, diagramas, tablas y formato profesional.
