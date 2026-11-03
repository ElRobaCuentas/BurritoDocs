# Incidentes corregidos — AdminWeb

## CASO 001 — El modal robaba el foco al escribir

**Fecha:** 03/11/2026.

**Síntoma:** edición de choferes perdía foco tras actualizar un input,
observado en B2. Administradores y turnos compartían el patrón.

**Causa:** el mismo efecto gestionaba apertura/foco/scroll y Escape con
dependencias `[open, onClose]`. Los callbacks nuevos en cada render
reiniciaban el efecto y ejecutaban `focus()` otra vez.

**Solución:** únicamente `AdminWeb/src/shared/ui/Modal.tsx`. Efecto de
apertura/scroll con `[open]`; listener de Escape separado con
`[open, onClose]`. Utiliza el callback vigente sin reenfocar en cada edición.
Los consumidores, CRUD, Auth y configuración Firebase se conservan.

**Verificación:** Oxlint y ambos proyectos TypeScript aprobados; bundle
Vite de producción generado en memoria con `envDir:false`, sin leer `.env`
ni escribir `dist`. Doce comprobaciones con efectos reales de React y
DOM/portal simulados: foco inicial único, cinco rerenders con nuevos
callbacks, Escape vigente, overlay, scroll, cierre/reapertura y desmontaje.
Se utilizó el renderer ya instalado en User, sin instalar dependencias;
emitió su advertencia de deprecación. Script de evidencia local:
`C:/Users/matia/AppData/Local/Temp/opencode/verify-modal-focus.cjs`.
No existe suite automatizada propia de AdminWeb; no se agregó framework.

**Pendiente visual:** en navegador, comprobar escritura continua en choferes,
administradores y turnos, Escape, overlay, reapertura y scroll. No se afirma
que esta prueba de navegador haya ocurrido ni que el bundle valide Firebase.

**Estado:** corregido/verificado localmente en `fix/modal-focus` (`5e66d46`), pendiente
de revisión e integración/push; sin deploy. B2 preservado exclusivamente
en `test/b2-adminweb-emulator`, commit `dc58daf`. Sus tres archivos conservan
los hashes Git previos y no forman parte de esta corrección:

| Archivo B2 | Hash Git |
|---|---|
| `src/shared/config/env.ts` | `f5859a1d7f7bc8c2a559dcf482037497fa5da746` |
| `src/shared/config/firebase.ts` | `592c3f3749f279b2589910d521b37529cc6c7931` |
| `src/features/admin/services/admin_service.ts` | `13059246b6e7c7769858873db54fd8152e66fa43` |

**Lección:** foco de apertura y cambios del callback de teclado tienen
ciclos diferentes. Sus efectos independientes evitan repetir el enfoque
sin fijar callbacks obsoletos ni retocar cada pantalla.
