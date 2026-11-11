# Deuda técnica de lint — UserApp (El Burrito)

## 1. Propósito y estado

Este documento registra la **deuda técnica de ESLint de la UserApp**: hallazgos
conocidos, **no corregidos y no olvidados**. No es un listado de bugs resueltos
ni un plan de refactor: es el inventario de referencia y las condiciones bajo
las cuales cada hallazgo debe revisarse.

Estado general al **07/11/2026**:

- **71 hallazgos pendientes**: 21 errores y 50 advertencias.
- **Ningún fallo funcional confirmado** en la auditoría.
- **4 efectos auditados** a nivel de código, con riesgos **aceptados
  provisionalmente para el MVP**.
- **67 hallazgos restantes** pendientes de análisis individual, programados
  para después del MVP.
- En esta auditoría **no se modificó código ni se ejecutó `eslint --fix`**.

> Este documento es la **fuente de verdad** del inventario. Otros documentos
> deben enlazarlo, no duplicar sus tablas.

---

## 2. Alcance

- Proyecto: **UserApp** (`C:\Movil\User`).
- Herramienta: **ESLint 8.57.1** con la configuración de React Native del
  proyecto (`npm run lint` → `eslint .`).
- Cubre los hallazgos del árbol de trabajo al 07/11/2026.
- **Fuera de alcance:** DriverApp y AdminWeb. El error preexistente de
  DriverApp (`asignacionId` sin usar en `SendCoordinates.tsx`) y el resultado
  limpio de AdminWeb no forman parte de este registro.

---

## 3. Línea base (07/11/2026)

| Regla | Errores | Advertencias | Total |
|-------|--------:|-------------:|------:|
| `@typescript-eslint/no-unused-vars` | 9 | 0 | 9 |
| `react-hooks/exhaustive-deps` | 12 | 0 | 12 |
| `react-native/no-inline-styles` | 0 | 47 | 47 |
| `react/no-unstable-nested-components` | 0 | 3 | 3 |
| **Total** | **21** | **50** | **71** |

Aclaraciones de trazabilidad:

- Los hallazgos se inventariaron en la auditoría del 07/11/2026; el mismo
  conteo ya figuraba como deuda previa en el corte del 03/11/2026
  (`TROUBLESHOOTING.md` §10).
- La antigüedad del código afectado, según el historial del repositorio, va del
  **21/02/2026** al **22/08/2026**.
- Las cifras son una **fotografía**: pueden variar si se actualizan ESLint, la
  configuración de React Native o `eslint-plugin-react-hooks`. En ese caso se
  debe re-baselinear y anotar en el registro de cambios (§11).

---

## 4. Convenciones del registro

### 4.1 Identificadores

- `USER-LINT-001` … `USER-LINT-021`: errores, ordenados por regla y archivo.
- `USER-LINT-W01` … `USER-LINT-W11`: advertencias **agrupadas por regla y
  archivo**. Cada línea listada dentro de un grupo es un hallazgo individual.

Los IDs son estables: pueden citarse en commits, incidencias o tests.

### 4.2 Decisiones

| Código | Significado |
|--------|-------------|
| **A** | Corregir antes del piloto. |
| **B** | Corregir después del MVP. |
| **C** | Mantener y documentar; si aplica, suprimir la regla con justificación explícita. |

### 4.3 Estados

| Estado | Significado |
|--------|-------------|
| `Aceptado provisional` | Auditado; se decidió no intervenir antes del MVP. |
| `Pendiente post-MVP` | Conocido y registrado; sin análisis individual todavía. |
| `Resuelto` | Corregido y verificado; se enlaza el caso en `BUGS_RESUELTOS/userapp.md`. |

### 4.4 Severidad

La columna "Grav." refleja la **severidad de ESLint**, no la gravedad en
runtime. Un `error` de `exhaustive-deps` no implica un fallo de la aplicación.

---

## 5. Inventario de errores (21)

### 5.1 `@typescript-eslint/no-unused-vars` (9)

| ID | Archivo:línea | Detalle |
|----|---------------|---------|
| USER-LINT-001 | `src/features/auth/screen/AvatarPickerScreen.tsx:44` | `navigation` asignado y no usado. |
| USER-LINT-002 | `src/features/auth/screen/SignInScreen.tsx:17` | `FadeInDown` definido y no usado. |
| USER-LINT-003 | `src/features/auth/screen/SignInScreen.tsx:17` | `FadeInUp` definido y no usado. |
| USER-LINT-004 | `src/features/auth/screen/SignInScreen.tsx:24` | `crashlytics` definido y no usado. |
| USER-LINT-005 | `src/features/auth/screen/SignUpScreen.tsx:19` | `FadeInDown` definido y no usado. |
| USER-LINT-006 | `src/features/auth/screen/SignUpScreen.tsx:19` | `FadeInUp` definido y no usado. |
| USER-LINT-007 | `src/features/auth/screen/WelcomeScreen.tsx:12` | `FadeInDown` definido y no usado. |
| USER-LINT-008 | `src/features/auth/screen/WelcomeScreen.tsx:12` | `FadeInUp` definido y no usado. |
| USER-LINT-009 | `src/features/map/components/CustomDrawer.tsx:44` | `navigation` asignado y no usado. |

### 5.2 `react-hooks/exhaustive-deps` (12)

| ID | Archivo:línea | Dependencias señaladas por ESLint |
|----|---------------|-----------------------------------|
| USER-LINT-010 | `src/app/App.tsx:63` | `loadThemeFromStorage` |
| USER-LINT-011 | `src/app/navigations/DrawerNavigator.tsx:147` | `setTheme` |
| USER-LINT-012 | `src/features/auth/screen/AvatarPickerScreen.tsx:68` | `breathing` |
| USER-LINT-013 | `src/features/auth/screen/SignUpScreen.tsx:71` | `breathing` |
| USER-LINT-014 | `src/features/map/components/CustomDrawer.tsx:82` | `backdropOpacity`, `translateX` |
| USER-LINT-015 | `src/features/map/components/Map.tsx:103` | `radarAnimValue` |
| USER-LINT-016 | `src/features/map/components/Map.tsx:147` | `radarAnimValue` |
| USER-LINT-017 | `src/features/map/components/Map.tsx:182` | `isFirstBusLoad`, `latAnim`, `lngAnim` |
| USER-LINT-018 | `src/features/map/components/Map.tsx:190` | `latAnim._value`, `lngAnim` |
| USER-LINT-019 | `src/features/map/components/Map.tsx:210` | `setCommand` |
| USER-LINT-020 | `src/features/map/components/StopCard.tsx:25` | `shimmerValue` |
| USER-LINT-021 | `src/features/map/screen/MapScreen.tsx:74` | `actions` |

Los cuatro últimos hallazgos auditados en detalle son **017, 018, 019 y 021**
(§7).

---

## 6. Inventario de advertencias (50)

### 6.1 `react-native/no-inline-styles` (47)

| ID | Archivo | Líneas | Nº |
|----|---------|--------|---:|
| USER-LINT-W01 | `src/app/App.tsx` | 77 | 1 |
| USER-LINT-W02 | `src/app/navigations/DrawerNavigator.tsx` | 78, 82, 86, 109, 125, 131, 133 | 7 |
| USER-LINT-W03 | `src/features/auth/screen/AvatarPickerScreen.tsx` | 109, 170, 175 | 3 |
| USER-LINT-W04 | `src/features/auth/screen/ForgotPasswordScreen.tsx` | 109 | 1 |
| USER-LINT-W05 | `src/features/auth/screen/SignInScreen.tsx` | 180, 226 | 2 |
| USER-LINT-W06 | `src/features/auth/screen/SignUpScreen.tsx` | 180, 226, 240, 266, 271, 332 | 6 |
| USER-LINT-W07 | `src/features/map/components/CustomDrawer.tsx` | 163, 198, 203, 218, 222, 226, 266, 270, 283, 286, 317, 326 | 12 |
| USER-LINT-W08 | `src/features/map/components/Map.tsx` | 276, 284, 316, 325, 334, 352 | 6 |
| USER-LINT-W09 | `src/features/map/components/StopCard.tsx` | 41, 42 | 2 |
| USER-LINT-W10 | `src/features/map/screen/MapScreen.tsx` | 124, 125, 128, 166, 180, 182, 185 | 7 |

Subtotal: **47 advertencias**.

### 6.2 `react/no-unstable-nested-components` (3)

| ID | Archivo | Líneas | Nº |
|----|---------|--------|---:|
| USER-LINT-W11 | `src/app/navigations/DrawerNavigator.tsx` | 127, 151, 170 | 3 |

Subtotal: **3 advertencias**. Total de advertencias: **50**.

> Las líneas se conservan textualmente del reporte JSON de ESLint. En los
> grupos, cada línea listada cuenta como un hallazgo independiente; se agrupan
> solo para evitar 50 filas idénticas.

---

## 7. Efectos auditados en `Map.tsx` y `MapScreen.tsx` (4)

Solo estos cuatro hallazgos se analizaron a nivel de código. Los 67 restantes
no tienen análisis individual y permanecen como deuda pendiente.

### 7.1 USER-LINT-017 — `Map.tsx:182` — Decisión **B**

| Campo | Valor |
|-------|-------|
| Ubicación | `src/features/map/components/Map.tsx:182` |
| Regla | `react-hooks/exhaustive-deps` |
| Dependencias señaladas | `isFirstBusLoad`, `latAnim`, `lngAnim` |
| Decisión | **B** (post-MVP) |
| Estado | Aceptado provisional |

**Comportamiento actual.** El efecto reacciona a `burritoLocation`: en la
primera carga fija las animaciones y el estado inicial; en las siguientes
interpola la posición con `RNAnimated.parallel` y actualiza el rumbo.

**Causa del hallazgo.** El efecto lee `isFirstBusLoad` (estado) y usa
`latAnim`/`lngAnim` (referencias animadas) sin declararlos.

**Impacto/riesgo.** `latAnim` y `lngAnim` son referencias estables
(`useRef(...).current`). `isFirstBusLoad` se actualiza en el mismo flujo que
dispara el efecto, por lo que en la práctica no se observó un cierre obsoleto
ni un fallo funcional. El riesgo es bajo y teórico.

**Decisión.** No intervenir antes del MVP. Al corregir, evitar añadir
`isFirstBusLoad` ingenuamente al arreglo de dependencias: al pasar a `false`
provocaría una segunda ejecución con la misma ubicación y una animación
duplicada. Preferir una referencia (`isFirstBusLoadRef`) o reestructurar el
efecto.

**Condición de revisión.** Al modificar la animación del bus, antes de
producción, o al actualizar `eslint-plugin-react-hooks`.

### 7.2 USER-LINT-018 — `Map.tsx:190` — Decisión **C**

| Campo | Valor |
|-------|-------|
| Ubicación | `src/features/map/components/Map.tsx:190` |
| Regla | `react-hooks/exhaustive-deps` |
| Dependencias señaladas | `latAnim._value`, `lngAnim` |
| Decisión | **C** (mantener y documentar) |
| Estado | Aceptado provisional |

**Comportamiento actual.** Un listener sobre `lngAnim` actualiza `currentPos`
leyendo `lngAnim._value` y `latAnim._value` (API privada de React Native, con
`@ts-ignore` en la línea 186).

**Causa del hallazgo.** `_value` es una propiedad privada, no una dependencia
reactiva; ESLint la señala porque se lee dentro del efecto con arreglo vacío.

**Impacto/riesgo.** El listener permanece activo y actualiza la posición sin
fallos observados. El riesgo real es de **mantenibilidad**: `_value` puede
cambiar entre versiones de React Native.

**Decisión.** Mantener y documentar. No añadir `_value` a las dependencias
(no tiene sentido reactivo). Cuando se refactorice la animación, sustituir la
lectura de `_value` por el payload del listener o por referencias compartidas,
y retirar el `@ts-ignore`.

**Condición de revisión.** Al refactorizar la animación del mapa o al
actualizar React Native.

### 7.3 USER-LINT-019 — `Map.tsx:210` — Decisión **B**

| Campo | Valor |
|-------|-------|
| Ubicación | `src/features/map/components/Map.tsx:210` |
| Regla | `react-hooks/exhaustive-deps` |
| Dependencias señaladas | `setCommand` |
| Decisión | **B** (post-MVP) |
| Estado | Aceptado provisional |

**Comportamiento actual.** Cuando `command === 'center'`, el efecto centra la
cámara y limpia el comando con `setCommand(null)`.

**Causa del hallazgo.** `setCommand` proviene del store de Zustand y no está
declarado en las dependencias.

**Impacto/riesgo.** `setCommand` es una acción estable del store; no se observó
fallo ni re-render anómalo.

**Decisión.** Añadir `setCommand` a las dependencias después del MVP (cambio
trivial y seguro) o mantener el efecto documentado si se prefiere no tocarlo.

**Condición de revisión.** Al modificar `mapStore` o la lógica de comandos del
mapa.

### 7.4 USER-LINT-021 — `MapScreen.tsx:74` — Decisión **B**

| Campo | Valor |
|-------|-------|
| Ubicación | `src/features/map/screen/MapScreen.tsx:74` |
| Regla | `react-hooks/exhaustive-deps` |
| Dependencias señaladas | `actions` |
| Decisión | **B** (post-MVP) |
| Estado | Aceptado provisional |

**Comportamiento actual.** Al montar la pantalla inicia el tracking, registra
analytics y arma un temporizador; al desmontar detiene el tracking y limpia el
temporizador.

**Causa del hallazgo.** `actions` se obtiene de `useBurritoStore()` y se usa
dentro del efecto sin declararlo.

**Impacto/riesgo.** El efecto corre una vez por montaje; no se observó fallo.
El riesgo es que, si la identidad de `actions` cambiara, añadirlo como
dependencia reiniciaría el tracking en cada cambio. Debe verificarse antes de
modificarlo.

**Decisión.** Después del MVP, preferir `useBurritoStore.getState().actions`
dentro del efecto (o un selector estable) antes que añadir el objeto completo
como dependencia.

**Condición de revisión.** Al modificar el ciclo de vida del tracking o antes
de producción.

---

## 8. Estado y riesgos aceptados

| ID | Hallazgo | Decisión | Estado | Riesgo aceptado |
|----|----------|----------|--------|-----------------|
| USER-LINT-017 | `Map.tsx:182` | B | Aceptado provisional | Cierre potencialmente obsoleto / animación duplicada si se corrige mal. |
| USER-LINT-018 | `Map.tsx:190` | C | Aceptado provisional | Dependencia de API privada (`_value`) y deuda de mantenibilidad. |
| USER-LINT-019 | `Map.tsx:210` | B | Aceptado provisional | Dependencia no declarada de una acción estable. |
| USER-LINT-021 | `MapScreen.tsx:74` | B | Aceptado provisional | Reinicio del tracking si la identidad de `actions` cambia. |
| Resto (67) | Ver §5 y §6 | — | Pendiente post-MVP | Sin análisis individual; riesgo no evaluado todavía. |

**Conclusión de la auditoría:** no hay evidencia de fallo funcional en los
cuatro efectos. Las advertencias restantes son de estilos inline y componentes
anidados; sus posibles efectos sobre rendimiento y comportamiento requieren
evaluación individual.

Riesgos aceptados provisionalmente para el MVP:

- Posibles cierres obsoletos o animaciones duplicadas en el mapa.
- Posibles re-renderizados o remontajes relacionados con estilos inline y
  componentes anidados, pendientes de evaluación.
- Deuda de mantenibilidad por `@ts-ignore` y API privada de animación.

Ninguno de estos riesgos bloquea el piloto. La aceptación es **provisional** y
queda sujeta a las condiciones de §9.

---

## 9. Condiciones de revisión

1. **Durante las pruebas reales (piloto).** Observar específicamente:
   saltos o duplicaciones en la animación del bus, posición congelada,
   radar o tarjeta de paradero con parpadeos, y reinicios inesperados del
   tracking. Si aparece un síntoma, reabrir el ID correspondiente y registrar
   la evidencia.
2. **Antes de producción.** Revisar obligatoriamente USER-LINT-017, 018, 019 y
   021. Decidir por escrito: corregir o re-aceptar el riesgo.
3. **Al tocar un archivo afectado.** Reevaluar los hallazgos de ese archivo en
   la misma tarea; actualizar el estado si cambia.
4. **Después del MVP.** Abrir un bloque de mantenimiento: primero los errores
   (§5), luego las advertencias (§6). Priorizar los archivos del mapa.
5. **Al actualizar React Native, ESLint o `eslint-plugin-react-hooks`.**
   Re-ejecutar la auditoría, comparar con la línea base y actualizar §3 y §11.
6. **Criterio de cierre.** Un hallazgo se considera resuelto cuando ESLint deja
   de reportarlo y la corrección se verificó. Si se opta por una excepción
   permanente, debe existir un `eslint-disable` justificado en el código y una
   fila con decisión **C** en este documento.
7. **Al resolver.** Mover el detalle a `docs/BUGS_RESUELTOS/userapp.md` como
   caso, y dejar aquí el ID con estado `Resuelto` y el enlace.

---

## 10. Cómo reproducir la auditoría

Ejecutar desde `C:\Movil\User`:

```powershell
Set-Location C:\Movil\User

# Versión del linter usada en la línea base
npx eslint --version        # 8.57.1

# Resumen legible (debe dar 21 errores y 50 advertencias)
npx eslint .

# Equivalente mediante el script del proyecto
npm run lint

# Detalle por archivo, regla y línea (formato JSON)
npx eslint . -f json

# Un archivo concreto
npx eslint src/features/map/components/Map.tsx
```

Notas:

- El comando `npm run lint` ejecuta `eslint .` (definido en `package.json`).
- Para esta deuda **no se usa `--fix`**: cualquier corrección es una tarea
  separada y debe registrarse en §11 y, al cerrarse, en
  `BUGS_RESUELTOS/userapp.md`.
- Si el conteo difiere, revisar primero cambios en la configuración de ESLint o
  en las dependencias antes de asumir una regresión.

---

## 11. Registro de cambios

| Fecha | Cambio |
|-------|--------|
| 07/11/2026 | Creación del registro. Línea base: 21 errores y 50 advertencias. Análisis de los cuatro efectos auditados (017, 018, 019, 021) con decisiones B/C/B/B. Sin cambios de código. |

---

## 12. Referencias

| Documento | Relación |
|-----------|----------|
| `TROUBLESHOOTING.md` §10 | Problemas conocidos; enlaza a este registro. |
| `BUGS_RESUELTOS/userapp.md` | Cierre de hallazgos resueltos. |
| `ARCHITECTURE.md` §9 | Limitaciones actuales del sistema. |
| `../MVP.md` | Alcance, cierre del MVP y regla de postergación. |
| `ROADMAP.md` | Planificación posterior al piloto y referencia al inventario de deuda técnica. |
| `../User/AGENTS.md` §7 | Enrutado de cambios documentales. |
