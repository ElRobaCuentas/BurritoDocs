# GRAPHIFY.md — Procedimiento de uso y mantenimiento del grafo

> Documento **detallado** (a demanda). Las reglas breves y siempre cargadas
> están en `../AGENTS.md`. Este archivo desarrolla el **cómo**. No duplica ni
> reemplaza la skill oficial de Graphify: ante conflicto, la skill oficial
> (sección 2) tiene precedencia.

---

## 1. Alcance

Aplica a todo `C:\Movil` (Ecosistema El Burrito). El grafo se construye sobre
el **root del repositorio** y se escribe en `graphify-out/`. Los artefactos
generados (`graph.json`, `GRAPH_REPORT.md`, `graph.html`, `manifest.json`,
`cost.json`, `.graphify_labels.json`) **no se editan a mano**.

Recordatorio (regla 4): el grafo es un **mapa**, no la especificación. Toda
afirmación debe confirmarse en el código. `EXTRACTED` = explícito en la fuente;
`INFERRED` = inferido (pista, no verificación).

---

## 2. Entorno instalado (esta máquina)

| Elemento | Valor |
|----------|-------|
| Versión CLI | `graphify 0.9.91` (verificado con `graphify --version`) |
| Binario CLI | `C:\Users\matia\.local\bin\graphify.exe` |
| Intérprete | `C:\Users\matia\AppData\Roaming\uv\tools\graphifyy\Scripts\python.exe` |
| Skill oficial (OpenCode) | `C:\Users\matia\.config\opencode\skills\graphify\SKILL.md` |
| Runbook incremental | `…\skills\graphify\references\update.md` |
| Spec de extracción semántica | `…\skills\graphify\references\extraction-spec.md` |
| API keys LLM | **Ninguna** (`GEMINI_API_KEY`/`GOOGLE_API_KEY` ausentes; tampoco ollama) |

**Consecuencia:** la extracción semántica (docs/papers/imágenes) la realiza el
**propio agente** vía subagentes. Código se extrae con AST (sin LLM).

Shell de referencia: **PowerShell 5.1** (no `&&`; usar `;` o `if ($?) { … }`).
Los bloques bash de la skill oficial se traducen a llamadas directas al
intérprete.

---

## 3. Mecanismo de actualización incremental (v0.9.91)

**Verificado contra `graphify --help` de la versión instalada.** Hay **dos**
rutas según el tipo de cambio:

### 3.1 Solo código (AST, sin LLM)

El subcomando del CLI cubre únicamente re-extracción de **código**:

```
graphify update <path>            # re-extract code files and update the graph (no LLM needed)
graphify update <path> --force    # sobrescribe aunque el rebuild tenga menos nodos
graphify update <path> --no-cluster
```

Usalo cuando **todos** los archivos cambiados sean de código.

### 3.2 Con documentación/instrucciones (semántico)

`graphify update` (CLI) **no procesa** `.md`/docs/papers/imágenes. Si algún
archivo cambiado es doc/instrucción/configuración textual, seguí el runbook de
la skill oficial: `references/update.md`. Resumen del flujo:

1. `detect_incremental(...)` → `.graphify_incremental.json`
   (nuevos/modificados/**eliminados** y sensibles omitidos).
2. Poblar `.graphify_detect.json` (`files` = subconjunto cambiado,
   `all_files` = corpus completo).
3. Si `code_only` → solo AST; si no → AST + **subagentes semánticos** (no hay
   API key) sobre los archivos no cacheados.
4. `build_merge(...)` con `prune_sources` = **solo** archivos eliminados
   (los modificados se reconcilian por "replace-on-re-extract").
   - Guardar antes el grafo previo (`Copy-Item graph.json .graphify_old.json`).
5. `save_manifest(...)` para que el próximo `--update` compare contra hoy.
6. Steps 4–8 de la skill (build → health check → labels → HTML → manifest/cost).
7. Limpiar temporales.

En OpenCode, la skill oficial indica despachar subagentes con `@agent` en un
único mensaje (paralelo). Si ese mecanismo no está disponible, usar el
equivalente de subagentes del entorno (`task`). **No leer los archivos uno por
uno a mano** (la skill lo prohíbe por lento).

### 3.3 Cómo decidir la ruta

```
¿Todos los archivos cambiados son código?  → sí → 3.1 (CLI)
                                           → no → 3.2 (runbook)
¿Solo hay eliminaciones?                    → 3.2 con extracción vacía (poda)
¿Cambios no reconocidos como code ni doc?   → reportar (regla 5); NO asumir
```

---

## 4. Verificación post-actualización (obligatoria)

No dar por bueno el comando sin comprobarlo (regla 4):

- **Diff:** `graph_diff(G_old, G_new)` (el runbook guarda `.graphify_old.json`).
- **Health check:** `diagnose_extraction(...)` — 0 dangling/missing/collapsed.
- **Manifiesto:** confirmar que los archivos **nuevos** figuran y que los
  **eliminados** ya no aparecen; `manifest.json` con la entrada nueva.
- **Grep selectivo:** buscar por nombre los archivos nuevos/modificados/
  eliminados en `graph.json` / `manifest.json` / `GRAPH_REPORT.md`.
- **Shrink-guard #479:** `to_json()` **rechaza escribir** si el grafo nuevo
  tiene **menos nodos** que el existente. Si el rechazo es por re-extracción
  legítima (ids semánticos que cambian de nombre), se documenta y se repite con
  `force=True`. **Nunca** forzar sin justificar.

---

## 5. Reconstrucción completa (excepcional)

Reservada para: problemas de integridad del grafo, cambios de configuración o
de **alcance** indexado (incl./excl.), cambios de versión mayor, o cuando el
procedimiento oficial lo exija. No es el camino por defecto (regla 6).

---

## 6. Manejo de fallos

Si Graphify falla o un archivo no se incorpora (regla 7):

1. Diagnosticá (versión, rutas, caché, formato del archivo).
2. Si no lo resolvés con seguridad, **informá** que el repositorio cambió pero
   **el grafo quedó pendiente de actualización**, indicando **qué** quedó
   pendiente.
3. No declares la tarea sincronizada si no verificaste el resultado.

---

## 7. Alcance indexado y limitaciones conocidas

Configuración de exclusión: `graphify-out/.graphify_build.json` (21 patrones) +
`.gitignore`. Excluidos relevantes:

- `**/google-services.json`, `**/.env`, `**/.env.*`, `**/*.keystore`,
  `**/keystore.properties`, `**/local.properties`, `**/gradle.properties`,
  `**/.xcode.env`, `**/.firebaserc`, `**/*.pem`, `**/*.key`, `**/*.p12`,
  `**/*.pfx` (sensibles — regla de seguridad).
- `.opencode/` (patrón `".opencode/"`): **el plugin y los agentes/commandos
  bajo `.opencode/` NO están en el grafo.** Cambios ahí **no** requieren
  actualización y, si se mencionan, debe aclararse que están fuera de alcance.
- `CUsersmatiaAppDataLocalTempopencodeadminweb/` (residuo histórico, ya
  eliminado) y **imágenes** (`png/jpg/jpeg/gif/webp/svg`).

**Estado indexado = árbol de trabajo.** El manifiesto (`manifest.json`) registra
`mtime` + hash por archivo; Graphify indexa lo que está **en disco** (incluye
cambios sin commit), no exclusivamente el último commit. No se exige commit
previo para actualizar el grafo.

**`docs/` es un repositorio Git independiente** (remoto `BurritoDocs`), separado
del resto de `C:\Movil`. Puede tener cambios sin commitear; el grafo refleja el
estado de disco de esos archivos igual que cualquier otro.

**Limitaciones operativas conocidas:**

- `BurritoApp_Informe.docx` (`C:\Movil\`) **no está indexado** (requiere el
  extra `graphifyy[office]`, no instalado; no instalar sin evaluar — regla 10).
- Las **reglas RTDB efectivamente desplegadas** no son verificables desde el
  repo → marcar como `[NO VERIFICABLE]` (ver 3 niveles en sección 9).
- Los `.json` se tratan como **código** (categoría `code`, extracción
  estructural/AST, **sin LLM**): verificado con `detect_incremental` (p. ej.
  `opencode.json`, `package.json`, `tsconfig.json`, `firebase.json`).

---

## 8. Restricciones de plataforma (PowerShell 5.1)

- Sin `&&`; encadenar con `;` o `if ($?) { … }`.
- Sin heredocs; escribir scripts a un temporal y llamar al intérprete.
- Leer archivos con `-Encoding UTF8` para evitar mojibake.
- `rg` no está en PATH → usar `Select-String`.
- Reemplazar `python3` de los ejemplos de la skill por la ruta del intérprete
  (sección 2).

---

## 9. Hechos verificados del repositorio (referencia rápida)

Verificar siempre contra el código antes de reutilizar:

- **Dominios:** conductores `{dni}@conductor.com` (`Driver/src/screen/LoginDriverScreen.tsx`,
  `AdminWeb/src/features/admin/services/admin_service.ts`); admins
  `{dni}@admin.com` (`AdminWeb/src/screen/LoginScreen.tsx`,
  `admin_service.ts`); usuarios: email simple.
- **Flags:** `SHOW_DEBUG_PANEL=false` (`User/src/features/map/Map.tsx`);
  `CALIB_LOG_ENABLED=false` (`User/src/store/burritoLocationStore.ts`).
- **Crashlytics:** en uso (`Driver/src/DriverApp.tsx`).
- **ADR:** `docs/DECISIONS.md` contiene ADR-003…ADR-026 (no hay ADR-001/002);
  ADR-017 marcada como *Sustituida* (su razonamiento original se conserva).
- **`DETENER TODO`** no es un concepto vigente (preservar solo como histórico).
  `DETENER RECORRIDO` detiene el envío de ubicación; `Cerrar sesión` es
  independiente (`signOut`).
- **Autoridad RTDB — 3 niveles:** (a) contrato que el código usa; (b) reglas
  locales capturadas (`.rules`/informe); (c) **permisos efectivos desplegados**
  → `[NO VERIFICABLE]`.
- **Faltantes confirmados:** `JEV_INVESTIGATION_REPORT.md`, `TAREAS_HISTORICO.md`,
  `ReviewNotes.md`, `mapTheme.ts` no existen en el repo.

---

## 10. Formato de reporte de cierre (regla 10 de AGENTS.md §5)

Al terminar una tarea que modificó archivos indexables:

```
Graphify: [actualizado y verificado | PENDIENTE — motivo: …]

Cambios de alcance:
  - Nuevos:      <archivos>
  - Modificados: <archivos>
  - Eliminados:  <archivos>
  - Fuera de alcance (no indexados): <archivos + motivo>

Verificación:
  - Nodos/edges/communities: <antes> → <después>
  - Health: <dangling/missing/collapsed>
  - Manifiesto: <N entradas>
```

---

## 11. Cómo comprobar que las instrucciones se cargaron

1. `C:\Movil\opencode.json` debe existir con `"instructions": ["AGENTS.md"]`.
2. **Reiniciar OpenCode** (la config no se recarga en caliente).
3. En sesión nueva, pedir "resumí las reglas de `AGENTS.md`": si OpenCode las
   tiene cargado, las reproduce sin leer el archivo a mano.
4. Prueba determinista: preguntar por el contenido de la sección 5 de
   `AGENTS.md`. Si no aparece, la carga falló.
5. `docs/GRAPHIFY.md` (este archivo) es **a demanda**: no figura en
   `instructions`; se lee solo cuando `AGENTS.md` o la tarea lo referencia.

---

## 12. Referencias

| Fuente | Uso |
|--------|-----|
| `../AGENTS.md` | Reglas breves siempre cargadas |
| Skill oficial Graphify (`…\skills\graphify\SKILL.md`) | Pipeline completo |
| `…\references\update.md` | Runbook incremental / cluster-only |
| `…\references\extraction-spec.md` | Prompt de extracción semántica |
| `https://github.com/Graphify-Labs/graphify` (rama `v8`) | Repositorio oficial |
