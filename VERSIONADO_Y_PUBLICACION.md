# Versionado y publicación del Ecosistema El Burrito

Guía operativa de Juan y Jose para el MVP, acordada el **19/11/2026**.
**Versionamos entregas, no commits.** Para Android: preparar → compilar → verificar
→ respaldar → crear tag y release → distribuir → confirmar el resultado.

## 1. Reglas y fuentes de verdad

| Componente | Identificación de la entrega | Distribución |
|------------|-----------------------------|--------------|
| El Burrito (UserApp, `User/`) | SemVer, `versionCode`, commit, tag anotado y GitHub Release | Google Play |
| BurritoDriverApp (`Driver/`) | SemVer, `versionCode`, commit, tag anotado y GitHub Release | APK privada |
| Central y servicio PULSO (`AdminWeb/`) | Commits e IDs de Hosting/Workers; sin nuevo SemVer ni tags obligatorios | Firebase Hosting / Cloudflare |
| `docs/` | Historial Git | Documentación compartida |

- Este documento contiene el procedimiento; **GitHub Releases**, el historial Android.
- **Play Console** indica el estado de User; **`C:\Movil\Releases`** conserva los binarios privados.
- Las versiones y los contadores son propios de cada app. No mover tags, reemplazar
  artefactos publicados ni atribuir origen a binarios históricos sin evidencia.
- No incorporar CI/CD de publicación, Fastlane, generadores de changelog ni otras herramientas.

## 2. Elegir versión y código

Modificar manualmente `versionName` y `versionCode` en `android/app/build.gradle`.
No automatizar su sincronización con `package.json` mediante Gradle antes del MVP.

| Valor | Regla |
|-------|-------|
| `versionName` | SemVer visible: PATCH para correcciones compatibles; MINOR para funciones compatibles; MAJOR para cambios incompatibles |
| `versionCode` | Entero mayor que todos los utilizados por esa app; no reiniciarlo ni incrementarlo por cada commit o build local |
| `package.json.version` | Versión del paquete JavaScript, separada de Android; sincronización no obligatoria durante el MVP |
| `package-lock.json` | Dependencias resueltas y metadatos npm; no identifica el binario. Si cambia la versión npm, mantener el lock coherente |

Ejemplos desde `1.2.0`: PATCH `1.2.1`, MINOR `1.3.0`, MAJOR `2.0.0`.
No exigir sufijos `alpha`, `beta` o `rc`: el canal y la clasificación pre-release
de GitHub indican la etapa. Cada nuevo binario entregado/subido necesita un código
nuevo; si modifica una entrega ya registrada, también una nueva versión de entrega.
Promover **el mismo artefacto** entre canales conserva versión, código, hash y tag.

## 3. Preparar el candidato y fijar el commit

Los comandos son para **futuras entregas autorizadas**, no para recrear `v1.2.0`.
Usar el entorno y las herramientas existentes, PowerShell 5.1 y la misma sesión
para conservar las variables. Ejecutar paso a paso y detenerse ante cualquier error.

1. Acordar alcance, responsable, versión y canal. Consultar las pruebas del
   `AGENTS.md` de la app y [condiciones de publicación](ROADMAP.md#4-condiciones-de-pre-release-y-publicación).
2. Comprobar la rama candidata; no asumir que es `main`. Integrar los cambios
   funcionales y completar sus pruebas antes de preparar la versión.
3. Consultar el máximo código en todos los canales de Play (User) o en entregas
   e instalaciones privadas (Driver). Cambiar solo los dos valores aprobados;
   no mezclar dependencias, Firebase, `applicationId` ni firma.

**Desde la raíz de la app**, seleccionar `User` o `Driver` y comprobar el candidato:

```powershell
$app = 'User' # Cambiar a 'Driver' para su entrega.
$repoPath = "C:\Movil\$app"
Set-Location -LiteralPath $repoPath
git status --short           # Identificar cambios propios o ajenos.
git branch --show-current    # Confirmar la rama acordada.
git remote -v                # Confirmar el repositorio remoto.
$version = Read-Host 'SemVer acordada'
$code = [int](Read-Host 'versionCode acordado')
git diff -- android/app/build.gradle
git diff --cached            # Comprobar que no haya otros cambios preparados.
git add -- android/app/build.gradle
git diff --cached            # Revisar exactamente lo que se va a registrar.
git commit -m "chore(android): bump version to $version"
```

Tras el commit exitoso, **en la misma raíz**, sincronizar y conservar su SHA:

```powershell
git status --short           # Debe estar vacío antes de compilar.
$branch = git branch --show-current
$commit = git rev-parse HEAD
git push origin "$branch"
git ls-remote origin "refs/heads/$branch" # Debe anunciar $commit.
```

Mantener el código y la configuración estables al compilar. Un commit y un lock
identifican fuentes, pero no garantizan reconstrucciones idénticas byte a byte.

## 4. Compilar y verificar el artefacto

Ejecutar **solo el bloque de la app elegida**. Exigir `BUILD SUCCESSFUL`; si falla,
no reutilizar una salida anterior. No regenerar ni rotar claves para publicar.

### User: AAB para Play

**Directorio: `C:\Movil\User\android`.** Generar el bundle y verificar su firma:

```powershell
Set-Location -LiteralPath 'C:\Movil\User\android'
.\gradlew.bat bundleRelease
if ($LASTEXITCODE -ne 0) { throw 'Falló la compilación.' }
$source = 'C:\Movil\User\android\app\build\outputs\bundle\release\app-release.aab'
$fileName = "el-burrito-v$version-vc$code.aab"
jarsigner -verify -verbose -certs "$source"
if ($LASTEXITCODE -ne 0) { throw 'Falló la verificación de firma.' }
```

Comprobar `jar verified`, revisar advertencias y contrastar el certificado con
el de carga esperado. Ese resultado no significa «sin advertencias» ni aceptación
de Play. Play App Signing distingue la firma de carga del AAB de la firma de los
APK distribuidos. `apksigner` verifica APK, no AAB; `npm run android` no genera
el AAB de publicación.

### Driver: APK privada

**Procedimiento previsto, pendiente de validación como nueva entrega formal.**
**Directorio: `C:\Movil\Driver\android`.** Ajustar Build Tools al SDK instalado:

```powershell
Set-Location -LiteralPath 'C:\Movil\Driver\android'
.\gradlew.bat assembleRelease
if ($LASTEXITCODE -ne 0) { throw 'Falló la compilación.' }
$source = 'C:\Movil\Driver\android\app\build\outputs\apk\release\app-release.apk'
$fileName = "burrito-driver-v$version-vc$code.apk"
$buildTools = "$env:LOCALAPPDATA\Android\Sdk\build-tools\36.0.0"
& "$buildTools\aapt.exe" dump badging "$source" # Ver paquete y versión/código.
if ($LASTEXITCODE -ne 0) { throw 'No se pudo inspeccionar el APK.' }
& "$buildTools\apksigner.bat" verify --verbose --print-certs "$source"
if ($LASTEXITCODE -ne 0) { throw 'Falló la verificación de firma.' }
```

Conservar `com.burritodriverapp` y la firma release de la instalación existente.
Contrastar la huella del certificado con la entrega instalada conocida; no
confundirla con el hash del archivo. Resolver discrepancias antes de instalar,
sin desinstalar para eludirlas. Elegir SemVer de tres componentes en la próxima entrega.

## 5. Calcular el hash y respaldar la copia exacta

**Desde el directorio Android de la app elegida**, con las variables anteriores:

```powershell
$hash = (Get-FileHash -LiteralPath $source -Algorithm SHA256).Hash
$hash # Registrar este valor en la GitHub Release.
$backupDir = "C:\Movil\Releases\$app"
if (-not (Test-Path -LiteralPath 'C:\Movil')) { throw 'Revisar la ruta base.' }
New-Item -ItemType Directory -Path $backupDir -Force | Out-Null
$artifact = Join-Path $backupDir $fileName
if (Test-Path -LiteralPath $artifact) { throw 'El destino ya existe; no sobrescribir.' }
[System.IO.File]::Copy($source, $artifact, $false)
if ((Get-FileHash -LiteralPath $artifact -Algorithm SHA256).Hash -ne $hash) {
    throw 'La copia no coincide con el original.'
}
```

Usar esa copia para distribuir. Mantener `Releases` fuera de los repositorios;
guardar una segunda copia en el destino privado acordado y comprobar su hash.
Una copia en el mismo disco no cubre la pérdida del equipo. Conservar mappings
y sourcemaps de esa compilación cuando sean necesarios para diagnosticarla.
No publicar artefactos privados por defecto; nunca incluir tokens, contraseñas
ni keystores en releases o paquetes. Respaldar las claves separadamente.

## 6. Crear el tag anotado y la GitHub Release

**Desde la raíz de la app**, fijar el commit que produjo el artefacto:

```powershell
Set-Location -LiteralPath $repoPath
git status --short            # Debe estar vacío.
git rev-parse HEAD            # Debe coincidir con $commit.
$tag = "v$version"
git tag --list "$tag"
git ls-remote --tags origin "refs/tags/$tag"
```

**Ambas consultas del tag deben estar vacías.** Si existe, inspeccionar la entrega
existente y detener su creación: no eliminar, mover ni forzar el tag.

```powershell
git tag -a "$tag" "$commit" -m "Release $tag - MVP"
git cat-file -t "$tag"        # Debe devolver tag.
git rev-parse "$tag^{}"       # Debe devolver $commit.
git push origin "refs/tags/$tag"
git ls-remote --tags origin "refs/tags/$tag" "refs/tags/$tag^{}"
```

En **GitHub web → Releases → nueva release**, seleccionar ese tag existente.
Usar títulos como `El Burrito vX.Y.Z - Pruebas cerradas MVP` o
`BurritoDriverApp vX.Y.Z - Piloto privado`. Marcar **pre-release** para pruebas;
clasificar como estable solo cuando se acuerde. Incluir:

- Commit completo, versión/código y cambios de la entrega.
- Artefacto, SHA-256, firma y advertencias, copia privada y responsable/fecha.
- Pruebas, compatibilidad, limitaciones y vía de recuperación.
- Canal y estado fechado: inicialmente, pendiente de envío o instalación.

Usar Conventional Commits: `fix`, `feat`, `chore(android)` o `docs` según el cambio.
Los tags `vMAJOR.MINOR.PATCH` son propios de cada repositorio Android.

## 7. Distribuir y confirmar el resultado

### User: Google Play Console

1. Abrir `com.sythor.burritoapp` y el canal acordado. Para el piloto se utilizó
   **Closed testing - El Burrito - App Beta**. Subir la copia privada verificada.
2. Contrastar versión/código reconocidos por Play; completar notas, testers,
   países y rollout. Revisar exclusiones/advertencias y registrar las decisiones.
   **100 % del canal cerrado** significa testers elegibles, no lanzamiento público.
3. Enviar a revisión y registrar el estado en la release. «En revisión» no es
   «disponible»: Play Console es la fuente de verdad.
4. Tras aprobación, confirmar publicación efectiva e instalación/actualización
   desde Play con un tester autorizado; verificar versión y funcionamiento.
5. Actualizar el estado fechado. Ante rechazo/demora, conservar el binario y el
   diagnóstico. Para producción, revisar las condiciones vigentes; promover el
   mismo AAB no requiere reconstruirlo ni cambiar sus identificadores.

### Driver: instalar sin interrumpir la jornada

**Procedimiento previsto; no acredita una actualización ya validada.** Distribuir
el APK privadamente y comprobar la visibilidad antes de adjuntarlo a GitHub.
Actualizar **después de que el último conductor finalice la jornada o antes de
que el primero la inicie**. Nunca durante un relevo ni interrumpiendo un recorrido
activo. Confirmar servicio detenido, estado remoto esperado y versión previa.

**Desde `C:\Movil\Driver`**, con ADB configurado y `$artifact` de esta entrega:

```powershell
Set-Location -LiteralPath 'C:\Movil\Driver'
adb devices # Elegir el teléfono autorizado.
$serial = Read-Host 'Serial del teléfono'
adb -s "$serial" install -r "$artifact" # Actualizar conservando los datos.
if ($LASTEXITCODE -ne 0) { throw 'Falló la instalación; registrar el error.' }
adb -s "$serial" shell dumpsys package com.burritodriverapp |
    Select-String -Pattern 'versionCode=|versionName='
```

Confirmar versión y probar acceso, asignación, INICIAR/publicación y DETENER en
condiciones controladas antes del servicio. Registrar fecha, dispositivo sin
datos sensibles, versiones previa/nueva y resultado. Compilar no acredita instalar.

## 8. Central y PULSO

Antes de desplegar, ejecutar `git status --short` y `git rev-parse HEAD` desde
`C:\Movil\AdminWeb`: exigir árbol limpio y registrar el SHA completo.
Central incluye la pantalla PULSO; el Worker se publica aparte e incluye código
de `monitor/src/` y `monitor/worker/`. No requieren nuevo SemVer ni GitHub Releases.

| Destino | Registro asociado al commit fuente |
|---------|-------------------------------------|
| Hosting | Proyecto/sitio/canal, version y release ID, fecha y resultado |
| Workers | Entorno/Worker, version y deployment ID, fecha y resultado |
| D1 | Migraciones aplicadas y compatibilidad requerida |

Asociar el SHA a la descripción del proveedor cuando sea posible; no inferirlo
por fechas. Seguir los comandos y verificaciones del
[README de AdminWeb](https://github.com/ElRobaCuentas/AdminWeb/blob/main/README.md#migraciones-y-publicación-en-producción):
**migración compatible → Worker → verificación → Hosting**. Conservar `STATE_VERSION`,
validaciones y migraciones; `compatibility_date` identifica compatibilidad del
runtime, no del producto. Validar combinaciones con [FIREBASE_SCHEMA.md](FIREBASE_SCHEMA.md)
y la API vigente. Evidencia histórica: [PULSO §8](PULSO.md#8-operación-en-producción).

## 9. Hotfixes y recuperación

- **Android:** partir del tag de producción afectado si `main` contiene avances no destinados
  a producción; corregir, incrementar PATCH y usar un código superior a todos los
  utilizados. Repetir el procedimiento completo. **Incorporar después el hotfix
  a `main` y verificar sus pruebas**, para no reintroducir el error en otra entrega.
- **User:** frenar una distribución no revierte instalaciones. Recuperar el
  comportamiento anterior mediante una nueva entrega con código superior.
- **Driver:** suspender instalaciones y corregir en la ventana indicada en §7;
  no usar downgrade forzado ni desinstalación como recuperación habitual.
- **Central:** volver a una versión compatible en Hosting, registrar el nuevo
  release de rollback y comprobar el contenido servido.
- **PULSO:** revisar D1, estado y configuración antes de volver a otro Worker.
  Su rollback no revierte D1; el Worker anterior a `0003` no mantiene el inicio
  observado. No borrar columnas, memoria ni incidentes para forzar compatibilidad.

Conservar artefactos fallidos, hashes y evidencias; registrar qué entrega los
sustituye y el resultado de recuperación, sin mover tags ni reemplazar backups.

## 10. Checklist breve

- [ ] Alcance/canal acordados, condiciones revisadas y pruebas completadas.
- [ ] Versión/código correctos; commit sincronizado y árbol limpio al compilar.
- [ ] Identidad/firma verificadas, advertencias revisadas y SHA-256 registrado.
- [ ] Copia privada y respaldo coinciden; tag anotado y GitHub Release completos.
- [ ] Distribución realizada en el canal o ventana acordados; resultado comprobado.
- [ ] Estado real fechado y recuperación registrada; hotfix incorporado a `main` si aplica.

## 11. Primera aplicación y validación pendiente

El Burrito `1.2.0 (3)` fue la primera aplicación del procedimiento; consultar su
[GitHub Release v1.2.0](https://github.com/ElRobaCuentas/BurritoUserApp/releases/tag/v1.2.0)
para artefacto, firma, hash y evidencias. Al corte del **19/11/2026**, los
desarrolladores reportaron envío a revisión; aprobación/disponibilidad para
testers seguían sin confirmar. El estado actual se consulta en Play Console.
Los pasos Driver de esta guía siguen pendientes de validación como entrega formal.
