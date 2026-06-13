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
Revisión continua durante la evolución del proyecto.


## FIREBASE_SCHEMA

### [ ] Eliminar `/ubicacion_burrito`

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
Pendiente.

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