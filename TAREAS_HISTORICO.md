> [!IMPORTANT]
> DOCUMENTO HISTÓRICO — Este archivo fue el backlog oficial del proyecto
> hasta la definición del MVP (julio 2026). Fue reemplazado por
> MVP.md, que es el único backlog operativo vigente. Este documento
> se conserva únicamente como referencia histórica y no forma parte
> del flujo de trabajo diario.
═══════════════════════════════════════════════════════════════════
        BACKLOG OFICIAL DE DESARROLLO — EL BURRITO (UNIFICADO)
═══════════════════════════════════════════════════════════════════

═══════════════════════════════════════════════════════════════════
BLOQUE 0 — HISTÓRICO (solo referencia)
═══════════════════════════════════════════════════════════════════
Tareas completadas durante el desarrollo base del proyecto. Se
archivan aquí para no mezclarlas con las pendientes. No se
volverán a ejecutar.

───────────────────────────────────────────────────────────────────
0.1 — Seguridad y Configuración Inicial
───────────────────────────────────────────────────────────────────

[✅] T1.1: Route Guard por Rol
  * Rama: fix/userapp/route-guard-rol
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Protege las pantallas de administración
    (AdminPanelScreen, ChoferesScreen, BusesScreen, AsignacionesScreen)
    validando rol === 'admin' en StackNavigator.tsx.
  * Commits:
    - fix(security): implementar bloqueo de rutas por rol en StackNavigator
    - docs(security): actualizar ARCHITECTURE y DECISIONS

[✅] T2.1: Firebase Rules
  * Proyecto: Consola Firebase (Realtime Database)
  * ¿Qué es/Qué resuelve?: Configura .indexOn para choferId sobre
    /asignaciones y reglas de mínimo privilegio por nodo.

[✅] T2.2: Testing Unitario
  * Rama: test/both/unit-testing
  * Proyecto: BurritoUserApp + BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Corrige test roto de DriverApp y añade
    cobertura a calculateDistance() (Haversine), getMovementStatus(),
    y filtro dedup por timestamp.
  * Commits:
    - test(core): corregir test de inicializacion y añadir cobertura a funciones Haversine
    - docs(testing): actualizar estado de cobertura de pruebas

───────────────────────────────────────────────────────────────────
0.2 — Multi-bus (infraestructura + validación)
───────────────────────────────────────────────────────────────────

[✅] T3.1: Multi-bus listener (Código)
  * Rama: refactor/userapp/multi-bus-listener
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Cambia listener de /ubicacion_burrito a
    /ubicacion_buses. Refactoriza store de Zustand a
    Record<string, BurritoLocation>.
  * Commits:
    - refactor(store): migrar path de firebase y cambiar estructura a Record multibus
    - docs(store): actualizar FIREBASE_SCHEMA y ROADMAP

[✅] T3.2: Multi-bus integración (Validación en campo)
  * Rama: test/userapp/multi-bus-integration
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Valida el pipeline completo con datos
    reales (Motorola, caminata + bus, Firebase Console + UserApp).
    Pipeline operativo y probado en condiciones reales.
  * Criterio de éxito cumplido:
    - 1 bus visible en Firebase Console + 1 placa en locations del store
    - 2 buses visibles en Firebase Console + 2 placas en Object.keys(locations)
  * Instrumentación temporal eliminada post-validación.

───────────────────────────────────────────────────────────────────
0.3 — Migración del Panel de Gestión a DriverApp (FASE 1–5)
───────────────────────────────────────────────────────────────────

Plan completo documentado en sección histórica al final de este
archivo. Todo completado: trasplante del módulo admin, router
tripartito, poda de UserApp, reglas RTDB, documentación.

[✅] Bug Async initializeApp: Corregido admin_service.ts (línea 79)
  con await + tipo FirebaseApp. Verificado con typecheck 0 errores
  y lint 0 errores.

───────────────────────────────────────────────────────────────────
0.4 — UX/UI Polishing (DriverApp) — Completado
───────────────────────────────────────────────────────────────────

[✅] UX-01: Ciclo 3 estados en SendCoordinates (separar stopProcess
  de signOut), FloatingBackButton, FloatingBackButton en CRUDs.

[✅] UX-02: Native-stack en AdminNavigator, gestureEnabled,
  headers eliminados.

[✅] UX-03: Dependencias react-native-vector-icons, native-stack.

[✅] T0: Standardize Navigation — unificar navegación bajo
  @react-navigation/stack con animation: 'none'.
  Commit: chore(nav): standardize on @react-navigation/stack

[✅] T1: Fix Admin Routing State Leak — eliminar flash
  Login→Tracking→Admin por verificación asíncrona de rol.
  Commit: fix(auth): prevent admin routing flash during role verification

[✅] T2: Android Vector Icons — configurar fonts.gradle para
  MaterialCommunityIcons.
  Commit: fix(android): link vector-icons fonts for Android

[✅] T3: Rediseño Login — branding, perfiles Admin/Conductor,
  diseño moderno.
  Commit: feat(ui): redesign driver login screen

───────────────────────────────────────────────────────────────────
0.5 — Congeladas (pasan a BLOQUE 7 — DIFERIDAS)
───────────────────────────────────────────────────────────────────

[⏸️] T1.2: Eliminar Debug Panel — Congelada por orden del
  desarrollador. Se mantiene visible hasta pre-producción.

[⏸️] T2.3: Asegurar Credenciales de Firma — Diferida hasta
  empaquetar v1.2.0.

═══════════════════════════════════════════════════════════════════
BLOQUE 1 — PRIMERA VALIDACIÓN EN CAMPO (PRIORIDAD ABSOLUTA)
═══════════════════════════════════════════════════════════════════
Objetivo: dejar el ecosistema listo para una prueba controlada con
1 bus real en el campus. La prioridad es funcionalidad y
confiabilidad, no estética.

Orden de ejecución dentro del bloque:
  1 → 2 → 3 → 4 (secuencial, cada uno depende del anterior)

───────────────────────────────────────────────────────────────────
[✅] P1 — DriverApp: Pantalla de conductor usable
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/tracking-screen-prod
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Reemplaza el terminal debug de
    SendCoordinates.tsx por una pantalla limpia y profesional.
    Muestra: DNI del conductor, bus asignado, estado del servicio
    (Compartiendo ubicación / Detenido), botones INICIAR/DETENER/
    CERRAR SESIÓN. Consola de logs colapsada bajo toggle para
    diagnóstico en campo. Sin emojis, sin hardcoded colors.
  * Por qué es bloqueante: el chofer podría operar el teléfono
    solo; no puede depender de una consola de eventos con emojis.
  * Versión lean (no branding completo todavía).
  * Origen: ROADMAP UX-4 (nunca entró en TAREAS.txt).
  * Commits:
    - feat(ui): redesign tracking screen for production
    - docs(ui): sincronizar documentacion post-rediseno
  * Depende de: nada (código UI puro, lógica intacta)

───────────────────────────────────────────────────────────────────
P2 — T4.1: Heartbeat (DriverApp)
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/heartbeat
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Agrega setInterval(8000) que mantiene
    fresco el timestamp en /ubicacion_buses/{busId} cuando el GPS
    está quieto. Evita que buses detenidos legítimamente (semáforo,
    paradero) se marquen como "SIN SEÑAL".
  * Por qué es bloqueante: en una ruta real el bus para varias
    veces por minuto. Sin heartbeat, cada parada genera un falso
    "desconectado". Base de la confiabilidad de la señal.
  * Commits:
    - feat(tracking): implementar intervalo de refresco de timestamp cada 8 segundos
    - docs(tracking): documentar watchdog y tiempos del ciclo en ARCHITECTURE

───────────────────────────────────────────────────────────────────
P3 — T5.3: Timeout Check (UserApp)
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/timeout-check
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Oculta buses cuyo timestamp no se
    actualiza en >60s (conductor cerró la app abruptamente o perdió
    señal). Depende de P2 (heartbeat) para distinguir un bus
    detenido legítimamente de uno fantasma.
  * Por qué es bloqueante: un bus fantasma en el mapa invalida
    cualquier conclusión de la prueba de campo.
  * Commits:
    - feat(tracking): ocultar de forma automatica buses sin señal tras inactividad
    - docs(tracking): registrar politicas de desconexion y expiracion de tokens
  * Depende de: P2 (Heartbeat)

───────────────────────────────────────────────────────────────────
P0 — Checklist de validación integral del ecosistema (NUEVA)
───────────────────────────────────────────────────────────────────
  * Rama: (sin rama — ejecución manual)
  * Proyecto: BurritoDriverApp + BurritoUserApp
  * ¿Qué es/Qué resuelve?: Lista de verificación sistemática para
    validar que el ecosistema completo funciona antes de la primera
    prueba con un bus real. No es código: es una ejecución QA.
  * Casos de prueba obligatorios:
    a) Login DriverApp (DNI + contraseña) → pantalla de tracking
    b) Login UserApp (email + contraseña) → mapa con bus
    c) INICIAR recorrido → permisos Android → foreground service
    d) GPS tracking continuo → Firebase Console escribe ubicaciones
    e) UserApp recibe actualizaciones → marcador se mueve
    f) Pantalla apagada → tracking continúa (15 min mínimo)
    g) App minimizada → tracking continúa (foreground service)
    h) Cerrar app sin DETENER → timeout (P3) oculta bus a los 60s
    i) DETENER recorrido → isActive: false en Firebase
    j) CERRAR SESIÓN → vuelve a LoginDriverScreen
    k) Asignación desde panel admin → DriverApp muestra el bus
    l) Sin asignación → mensaje "Sin bus asignado" + botón disabled
  * Criterio de éxito: los 12 casos (a-l) pasan sin errores.
  * Commits:
    - (ninguno — es checklist de validación, no genera código)
  * Depende de: P1 + P2 + P3 implementados
  * Nota: los resultados de esta validación definen si se pasa a
    Bloque 2 o se corrige algo primero.

═══════════════════════════════════════════════════════════════════
BLOQUE 2 — ROBUSTEZ (no bloquea el arranque, pero es deseable)
═══════════════════════════════════════════════════════════════════
Mejoras de confiabilidad y resistencia. No son obligatorias para
la primera prueba con 1 bus, pero aumentan la calidad de los
resultados.

───────────────────────────────────────────────────────────────────
R1 — Prueba de resistencia de turno completo (NUEVA)
───────────────────────────────────────────────────────────────────
  * Rama: (sin rama — ejecución manual)
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Validar batería + servicio en foreground
    con pantalla apagada durante 2-3 horas continuas (duración de
    un turno real). Verificar que Android no mata el proceso, la
    batería no se agota críticamente, y el timestamp se mantiene
    fresco durante toda la sesión.
  * Justificación: el piloto real corre un turno completo. Hoy no
    existe ninguna tarea que valide que el teléfono aguanta la
    sesión sin que Android mate el proceso o se agote la batería.
    Es un riesgo directo de fracaso en campo.
  * Criterio de éxito:
    - GPS activo y escribiendo a Firebase durante 120+ minutos
    - Batería restante > 30% al finalizar
    - Sin crashes ni cierres inesperados del foreground service
    - Al abrir la app después de 2h, el tracking sigue activo
  * Depende de: P1 + P2

───────────────────────────────────────────────────────────────────
R2 — Validación de pérdida de señal / reconexión (NUEVA)
───────────────────────────────────────────────────────────────────
  * Rama: (sin rama — ejecución manual)
  * Proyecto: BurritoDriverApp + BurritoUserApp
  * ¿Qué es/Qué resuelve?: Verificar que al perder cobertura
    (modo avión, túnel) y recuperarla, el tracking se reanuda
    automáticamente y no dispara posiciones atrasadas en ráfaga.
  * Justificación: las rutas universitarias tienen zonas sin datos.
    Es un escenario garantizado del piloto y hoy no está cubierto
    por ninguna tarea. Coherente con ADR-003 (sin persistence).
  * Criterio de éxito:
    - Desactivar datos → tracking se detiene (timestamp se congela)
    - Reactivar datos → tracking se reanuda en ≤10s
    - La posición enviada post-reconexión es la actual, no una
      acumulada durante la desconexión
    - UserApp muestra el bus correctamente post-reconexión
  * Depende de: P1 + P2 + P3

───────────────────────────────────────────────────────────────────
T5.2: Smoothing (implementación)
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/smoothing
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Algoritmo de interpolación (moving
    average / Kalman ligero) para suavizar saltos del marcador
    en el mapa del estudiante. Hoy solo hay un tween lineal de 2s.
  * Commits:
    - feat(map): aplicar interpolacion para suavizado del movimiento del bus
    - docs(map): documentar el algoritmo de transicion de coordenadas aplicado
  * Depende de: P1 + P2 + P3

═══════════════════════════════════════════════════════════════════
BLOQUE 3 — PRE-ESCALADO A 2 BUSES
═══════════════════════════════════════════════════════════════════
Requisitos antes de agregar un segundo bus a las pruebas.

───────────────────────────────────────────────────────────────────
T4.4: Multi-bus render completo
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/multi-bus-render
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: ShapeSource + SymbolLayer + CircleLayer
    independiente por cada bus activo en Map.tsx. Hoy solo se
    dibuja el primer bus activo (fallback del store). Con 1 bus no
    es bloqueante; con 2 buses es obligatorio.
  * Commits:
    - feat(map): renderizar multiples marcadores con animaciones independientes por busId
    - docs(map): documentar ciclo de vida del renderizado de ShapeSources en ARCHITECTURE

───────────────────────────────────────────────────────────────────
T1.2: Eliminar Debug Panel (sale de congelada)
───────────────────────────────────────────────────────────────────
  * Rama: chore/userapp/remove-debug-panel
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Remueve el panel "RADAR DE DATOS RAW"
    (verde/cyan) de Map.tsx que no debe verse cuando la UserApp
    se exponga a estudiantes reales.
  * Nota: se ejecuta solo cuando se decide exponer UserApp a
    estudiantes. Durante el piloto interno (solo equipo dev) puede
    permanecer visible.
  * Commits:
    - chore(ui): extirpar panel de debug radar de datos raw en Map
    - docs(ui): actualizar PROJECT_CONTEXT reflejando la limpieza de la interfaz

═══════════════════════════════════════════════════════════════════
BLOQUE 4 — MEJORAS FUNCIONALES (DriverApp)
═══════════════════════════════════════════════════════════════════
Mejoras en la experiencia del conductor que requieren lógica ligera
pero NO modifican el tracking, GPS ni BackgroundJob. Se implementan
post-piloto porque ninguna bloquea la validación en campo.

───────────────────────────────────────────────────────────────────
F1 — Conductor Info: mostrar nombre del conductor
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/conductor-info
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Agrega el nombre del conductor a la
    pantalla de tracking. Requiere consulta a /choferes/{dni} en
    fetchAssignment().
  * Commits:
    - feat(tracking): mostrar nombre del conductor en pantalla de tracking
    - docs(tracking): actualizar documentacion con nueva consulta
  * Depende de: P1 (pantalla de tracking con diseño de producción)

───────────────────────────────────────────────────────────────────
F2 — Fecha de Asignación
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/fecha-asignacion
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Muestra la fecha de la asignación actual
    en la pantalla de tracking. Ya existe en el snapshot de Firebase
    (val.fecha) pero se descarta. Solo requiere persistir el valor.
  * Commits:
    - feat(tracking): mostrar fecha de asignacion en pantalla de tracking
  * Depende de: P1

───────────────────────────────────────────────────────────────────
F3 — Hora de Inicio del Servicio
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/hora-inicio
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Registra y muestra la hora exacta en
    que el conductor presionó INICIAR. Requiere agregar timestamp
    en startProcess() y reset en stopProcess().
  * Commits:
    - feat(tracking): registrar y mostrar hora de inicio del servicio
  * Depende de: P1

───────────────────────────────────────────────────────────────────
F4 — Información adicional del Bus
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/info-bus
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Muestra marca y modelo del bus asignado
    (ej. "Mercedes Benz — 2022"). Requiere consulta a /buses/{busId}.
  * Commits:
    - feat(tracking): mostrar marca y modelo del bus asignado
  * Depende de: P1

═══════════════════════════════════════════════════════════════════
BLOQUE 5 — FUNCIONALIDADES GRANDES
═══════════════════════════════════════════════════════════════════
Funcionalidades que cambian el producto significativamente. Se
implementan después de validar el tracking base en campo.

───────────────────────────────────────────────────────────────────
T4.2: Control de Turnos
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/control-turnos
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Botones INICIAR/FINALIZAR TURNO. Guarda
    Punto Cero (GPS actual) y crea /recorridos/{pushId} con
    activo:true.
  * Commits:
    - feat(turnos): crear ciclo de recorridos con botones de inicio y fin de jornada
    - docs(turnos): mapear nuevo nodo de base de datos en FIREBASE_SCHEMA

───────────────────────────────────────────────────────────────────
T4.3: Geofencing (implementación + validación real)
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/geofencing
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Máquina de estados con Haversine (ya
    testeada en T2.2), histéresis de 40m/80m y cierre automático
    de vueltas. La validación contra multipath real va durante el
    Bloque 6.
  * Commits:
    - feat(geo): implementar maquina de estados con histeresis para cierre automatico
    - docs(geo): documentar tolerancias metricas y radio de captura en DECISIONS
  * Depende de: T4.2 (Control de Turnos)

───────────────────────────────────────────────────────────────────
T4.5: Monitor de Flota
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/monitor-flota
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Sección "Flota en servicio ahora" en
    AdminPanelScreen. Cruza /ubicacion_buses con /asignaciones.
  * Commits:
    - feat(admin): añadir panel de monitoreo en tiempo real de buses activos
    - docs(admin): actualizar guia del panel de control en el manual del sistema

───────────────────────────────────────────────────────────────────
T4.6: Estadísticas
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/estadisticas
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: StatsScreen con total de vueltas,
    duración promedio, vuelta más rápida/lenta. Lee de /recorridos
    (creados en T4.2).
  * Commits:
    - feat(stats): crear modulo de calculo de tiempos promedio y duracion de vueltas
    - docs(stats): definir formulas de promedio aplicadas en la documentacion tecnica
  * Depende de: T4.2 (Control de Turnos)

═══════════════════════════════════════════════════════════════════
BLOQUE 6 — VALIDACIÓN CAMPUS UNMSM
═══════════════════════════════════════════════════════════════════
Requiere estar físicamente en la universidad y dentro del bus
universitario.

───────────────────────────────────────────────────────────────────
T4.3: Geofencing (validación real)
───────────────────────────────────────────────────────────────────
  * Rama: feat/driverapp/geofencing
  * Proyecto: BurritoDriverApp
  * ¿Qué es/Qué resuelve?: Recorrer el circuito universitario con
    el bus real. Ajustar radio de 40m y ventana de histéresis de
    80m contra el multipath de los edificios. La implementación
    está en Bloque 5.
  * Depende de: T4.3 implementación (Bloque 5)

───────────────────────────────────────────────────────────────────
T5.1: Calibración de Paraderos
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/calibracion-paraderos
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Registrar coordenadas GPS reales en
    cada uno de los 10 paraderos con el Motorola en el campus.
  * Commits:
    - chore(geo): calibrar coordenadas reales de los diez paraderos desde campo
    - docs(geo): actualizar listado oficial de paraderos con precision de hardware

───────────────────────────────────────────────────────────────────
T5.2: Smoothing (ajuste fino)
───────────────────────────────────────────────────────────────────
  * Rama: feat/userapp/smoothing
  * Proyecto: BurritoUserApp
  * ¿Qué es/Qué resuelve?: Ajustar coeficientes del suavizado
    contra datos reales capturados en el circuito del campus.
    La implementación está en Bloque 2.
  * Depende de: T5.2 implementación (Bloque 2)

═══════════════════════════════════════════════════════════════════
BLOQUE 7 — DIFERIDAS
═══════════════════════════════════════════════════════════════════
Tareas sin fecha asignada. No bloquean nada.

───────────────────────────────────────────────────────────────────
[⏸️] T1.2: Eliminar Debug Panel (reenviada desde Bloque 0)
  * Estado: Congelada hasta pre-producción
  * Disparador: cuando la UserApp vaya a exponerse a estudiantes
    reales. No antes.

[⏸️] T2.3: Asegurar Credenciales de Firma
  * Estado: Diferida
  * Disparador: al empaquetar el build de distribución para
    Play Store / Closed Testing. Para piloto con build sideloaded
    sigue diferida.

═══════════════════════════════════════════════════════════════════
FIN DEL BACKLOG OFICIAL
═════════════════════════════════════════════════════════════════
