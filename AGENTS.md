# Documentación compartida — AGENTS.md

## 1. Alcance

`docs/` es un repositorio Git independiente del Ecosistema El Burrito. Contiene
la documentación técnica compartida por El Burrito (`../User/`), BurritoDriverApp
(`../Driver/`) y AdminWeb (`../AdminWeb/`): arquitectura, esquema de RTDB,
decisiones, diagnóstico, deuda técnica y bugs resueltos.

Este archivo rige solo el trabajo dentro de `docs/`; no reemplaza las reglas de
cada app ni el índice del ecosistema. Los comandos Git, diffs y commits se
ejecutan dentro de este repositorio, nunca desde `../`.

---

## 2. Convenciones

- Escribí en español. Identificadores, paths de RTDB y nombres de símbolos van
  en su forma exacta.
- Documentá el comportamiento vigente, no el deseado. Lo pendiente se marca como
  pendiente.
- Preferí referencias concretas (`archivo:línea`, path de RTDB, nombre de
  función) sobre descripciones vagas.
- No dupliques contenido: si un dato ya vive en otro documento, referencialo.
- Mantené la estructura y el estilo existentes; no reordenes ni renombres
  secciones sin necesidad.

---

## 3. Preservación

- Las decisiones arquitectónicas van en `DECISIONS.md`. No reescribas una
  decisión: agregá una entrada nueva y conservá el historial.
- Los bugs corregidos van en `BUGS_RESUELTOS/`, en el archivo de la app afectada
  (`userapp.md`, `driverapp.md`, `adminweb.md`). Los bugs abiertos o conocidos
  van en `TROUBLESHOOTING.md`.
- No elimines historial, evidencias, contratos técnicos ni documentación
  histórica vigente sin autorización explícita.
- No sobrescribas backups ni evidencias.
- Si un cambio invalida un contrato (path o nodo de RTDB, flujo, variable de
  entorno), actualizá el documento que lo define en la misma tarea.

---

## 4. Verificación contra las fuentes

Confirmá toda afirmación técnica contra el código o la configuración vigente del
repositorio correspondiente. Una inferencia, un resumen previo o una herramienta
de navegación no constituyen verificación. Si no podés comprobarlo, decilo en
vez de afirmarlo.

---

## 5. Reglas comunes del ecosistema

Antes de modificar documentación, consultá `../AGENTS.md` (índice y reglas
comunes) y `../FLUJO_DE_TRABAJO.md` (procedimiento). Si no están disponibles,
informalo y no asumas que fueron cargados automáticamente. `../MVP.md` define el
alcance y el estado del MVP.

---

## 6. Tareas exclusivamente documentales

Durante una tarea de documentación no modifiques código de `../User/`,
`../Driver/` ni `../AdminWeb/` por iniciativa propia. Si detectás una
discrepancia entre código y documentación, documentá la discrepancia o preguntá;
no "corrijas" el código como efecto colateral. Cambiar código requiere una
solicitud explícita.

---

## 7. Mapa de documentos

| Documento | Contenido |
|-----------|-----------|
| `ARCHITECTURE.md` | Arquitectura, flujos y limitaciones del sistema |
| `FIREBASE_SCHEMA.md` | Paths y nodos de Realtime Database |
| `DECISIONS.md` | Decisiones arquitectónicas (ADR) |
| `TROUBLESHOOTING.md` | Diagnóstico y bugs conocidos |
| `BUGS_RESUELTOS/` | Historial de bugs corregidos por app |
| `ROADMAP.md` | Prioridades post-MVP |
| `LINT_DEBT.md` | Deuda de lint de UserApp |
