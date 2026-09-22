# Mi Timesheet Diario

Skill personal (disponible en cualquier proyecto) para reconstruir y completar el
timesheet diario de **Luandry Pérez** en Odoo (`vauxooprod`), a partir de evidencia
real de trabajo (Google Calendar, correos enviados, actividad de Google, sesiones de
Claude Code activas y/o lo que el usuario dicte), hasta el total de horas por día que
el usuario indica — **sin inventar horas ni rellenar con actividad genérica**, y sin
duplicar entradas ya cargadas.

El usuario ya sabe cuánto trabajó cada día: el número de horas que da por día es un
**dato, no una estimación a validar**. El trabajo del skill es reconstruir el detalle
(qué tareas, cuánto tiempo cada una) que sustente ese total — no cuestionar el total
en sí. Si la evidencia recolectada no llega a cubrirlo, se lo dice al usuario y le
pregunta directamente en qué más trabajó, en vez de asumir que el total está mal.

## Identidad fija de este skill

- `employee_id`: 491 (Luandry Pérez)
- `user_id`: 1672 (Luandry Pérez [Vauxoo])
- Perfil Odoo por defecto: `vauxooprod` (ahí viven `project.task` y
  `account.analytic.line` para todos los clientes/proyectos, es el tracker interno de
  Vauxoo — no cada instancia de cliente).

Si algún día estos IDs cambian (nuevo `employee_id`, otro perfil), actualizar esta
sección a mano.

## Reglas no negociables (confirmadas explícitamente por el usuario)

1. **Solo horas con evidencia real.** Si la evidencia recolectada no alcanza el
   objetivo de horas del día, se informa el déficit y se detiene ahí — **nunca** se
   rellena el resto con actividad administrativa/gestión genérica inventada para
   cuadrar el número. Un timesheet es un registro financiero/de facturación.
2. **Revisión y confirmación explícita por día, siempre.** Nunca se escribe en Odoo
   sin que el usuario apruebe el borrador de ese día específico (tarea, descripción,
   horas). No hay modo "publicar directo".
3. **Nunca duplicar.** Antes de proponer nada, se consultan las entradas ya
   existentes en Odoo para ese día y ese empleado, y no se recrean conceptos ya
   cargados — solo se completa el faltante.

## Fuentes de evidencia habilitadas

Todas están aprobadas por el usuario, pero cada una tiene límites reales que hay que
respetar y comunicar:

- **Google Calendar (reuniones/sesiones)** — `mcp__claude_ai_Google_Calendar__list_events`
  o `search_events` filtrando por la fecha objetivo (rango `00:00`–`23:59` de ese
  día). Es la fuente más confiable para bloques de tiempo con hora de inicio/fin
  real (reuniones, sesiones de seguimiento) — usarla primero para anclar los bloques
  de tiempo del día antes de rellenar con las demás fuentes. Incluir asistentes/
  título del evento para mapear a cliente/tarea en la Fase 3.
- **Correos enviados (Gmail)** — `mcp__claude_ai_Gmail__search_threads` con
  `in:sent after:YYYY/MM/DD before:YYYY/MM/DD+1` (ajustar formato de fecha al que
  acepte la búsqueda de Gmail). Da destinatario, asunto y hora — buena señal de con
  quién y sobre qué se trabajó.
- **Actividad de Google (My Activity)** — navegar con `claude-in-chrome` a
  `https://myactivity.google.com/myactivity` filtrando por la fecha objetivo, y leer
  la página (`get_page_text` o `read_page`). Requiere que el usuario esté logueado en
  el navegador y que el historial esté habilitado; si la página no carga o no hay
  sesión iniciada, avisar y no bloquear el resto del flujo. **No es un historial de
  navegación completo de Chrome** (esa herramienta no existe) — es solo lo que Google
  decidió registrar.
- **Sesiones de Claude Code activas ahora** — `ListAgents` para ver qué sesiones
  siguen corriendo en este momento, y `SendMessage` a cada una relevante pidiendo un
  resumen breve ("¿en qué tarea/proyecto trabajaste hoy, qué hiciste, cuánto tiempo
  aprox.?"). **Solo sirve para el día de hoy** (o mientras esa sesión siga abierta) —
  no hay forma de recuperar el historial de una sesión ya cerrada de un día pasado.
  Si la fecha objetivo no es hoy, omitir esta fuente automáticamente y decírselo al
  usuario.
- **Dictado manual** — siempre disponible como respaldo/complemento: preguntar
  directamente al usuario qué hizo cuando las fuentes automáticas no alcanzan o no
  aplican (días pasados sin sesión activa, actividad que no deja rastro en correo o
  Google Activity, reuniones, llamadas, etc.).

## Invocación

```
/mi-timesheet-diario --fecha <hoy|ayer|lunes|...|YYYY-MM-DD> --horas <N>
```

Ejemplos:

```
/mi-timesheet-diario --fecha hoy --horas 6
/mi-timesheet-diario --fecha 2026-08-24 --horas 8
```

También acepta varios días en una sola invocación, pero **cada día se procesa y se
confirma por separado** (nunca se agrupa la aprobación de varios días en una sola
pregunta):

```
/mi-timesheet-diario --dias "lunes:6, martes:8, miercoles:6"
```

Si falta la fecha o las horas objetivo de algún día, preguntar antes de continuar —
no asumir.

## Fases

### Fase 0 — Resolver fecha(s) y objetivo de horas

Convertir cada referencia relativa ("hoy", "ayer", "lunes") a fecha absoluta
`YYYY-MM-DD` usando la fecha actual del sistema como ancla. Si el usuario da un día
de la semana sin más contexto, asumir la ocurrencia más reciente (hoy o hacia atrás),
y confirmar la fecha resuelta con el usuario antes de seguir si hay ambigüedad.

### Fase 1 — Consultar lo que ya está cargado ese día (anti-duplicado)

```
mcp__odoo__search_read
  model: account.analytic.line
  domain: [["employee_id", "=", 491], ["date", "=", "<fecha>"]]
  fields: id,name,task_id,project_id,unit_amount
  profile: vauxooprod
```

- Sumar `unit_amount` → `horas_ya_cargadas`.
- Si `horas_ya_cargadas >= horas_objetivo`: informar al usuario que ese día ya está
  completo (mostrar las entradas existentes) y **no crear nada**.
- Si es parcial: mostrar las entradas existentes (para que el usuario vea qué
  conceptos ya están cubiertos) y calcular `horas_faltantes = horas_objetivo -
  horas_ya_cargadas`.

### Fase 2 — Recolectar evidencia para cubrir `horas_faltantes`

Empezar por **Google Calendar** para anclar los bloques de tiempo con hora real
(reuniones/sesiones) — esas horas ya vienen dadas por el evento, no hay que
estimarlas. Completar el resto del día con las demás fuentes según aplique a la
fecha (recordar: sesiones de Claude Code activas solo si la fecha es hoy). Para cada
pieza de evidencia, extraer: qué se hizo, con quién/sobre qué cliente o tarea, y a
qué hora aprox. (para poder estimar duración por espaciado entre eventos cuando no
hay duración explícita, como en correos o actividad de Google).

No es necesario agotar todas las fuentes si una ya deja claro el trabajo del día;
priorizar señal clara sobre volumen.

### Fase 3 — Mapear evidencia a tareas reales de Odoo

Para cada bloque de evidencia, buscar la `project.task` correspondiente en
`vauxooprod` (por cliente/nombre mencionado, o preguntando al usuario si no es
obvio). **No inventar ni asumir una tarea al azar** — si no se puede mapear con
confianza razonable, dejarlo pendiente y preguntar al usuario a qué tarea va, en vez
de forzar una.

### Fase 4 — Redactar cada entrada

Aplicar las reglas de redacción del skill `/timesheet` (primera persona, verbos de
acción fuertes, estructura **qué / sobre qué / para qué / resultado**, **mínimo 100
palabras** por entrada, sin lenguaje administrativo pasivo). El cliente audita
actividad por actividad, así que cada descripción debe decir explícitamente qué se
hizo, sobre qué objeto concreto (registro, empleado, factura, ticket, flujo…) y con
qué resultado, con detalle suficiente para que un auditor externo la entienda sin
más contexto. Si algún dato no se conoce, asumir lo más probable, marcarlo y
preguntar al usuario antes de registrar. El campo `name` de cada línea es directamente
esa descripción redactada (sin el prefijo "[Actividad] |" del formato de salida de
`/timesheet` — así quedan consistentes con las entradas ya existentes en este
tracker).

Repartir `horas_faltantes` entre los bloques de evidencia reales — nunca hacer que la
suma exceda `horas_faltantes` ni el objetivo del día. Si sobra evidencia una vez
cubierto el objetivo, priorizar lo más significativo y descartar el resto (no hace
falta registrar absolutamente todo lo que se hizo, solo llegar al objetivo con lo más
relevante).

### Fase 5 — Presentar el borrador y ajustar de forma iterativa

Mostrar una tabla por día, **numerada** para que sea fácil referirse a una fila
específica:

| # | Tarea (Odoo) | Descripción | Horas | Evidencia |
|---|---|---|---|---|

Incluir también: horas ya cargadas + horas del borrador = total vs. horas indicadas
por el usuario. Si hay déficit (evidencia insuficiente para llegar al total), decirlo
explícitamente y preguntar en qué más trabajó, en vez de disimularlo o inventar.

Este es un ciclo de ajuste, no una pregunta de sí/no: el usuario puede pedir cosas
como "a la 2 súbele 1h y a la 3 bájale lo mismo", "quita la 4", "la reunión duró más,
ponle 1.5h", etc. Ante cualquier ajuste:

1. Aplicar el cambio a la fila indicada.
2. Recalcular el total y mostrar la tabla actualizada completa (no solo la fila
   cambiada, para que siempre vea el cuadre contra el total del día).
3. Repetir hasta que el usuario confirme explícitamente ese día ("así queda bien",
   "confirmado", etc.).

**Nunca escribir nada en Odoo sin esa confirmación explícita**, y solo para el día
que fue confirmado — los demás días de una invocación multi-día se procesan y
confirman por separado.

### Fase 6 — Publicar en Odoo (solo tras confirmación)

Por cada bloque aprobado:

```
mcp__odoo__create
  model: account.analytic.line
  values: {
    "name": "<descripción redactada>",
    "project_id": <id>,
    "task_id": <id>,
    "employee_id": 491,
    "date": "<fecha>",
    "unit_amount": <horas>
  }
  profile: vauxooprod
```

Reportar los IDs creados y el total final de horas de ese día.

## Reglas adicionales

- Si una fuente de evidencia (Gmail, My Activity, sesión activa) no está disponible o
  falla, avisar brevemente y continuar con las demás — no bloquear todo el flujo por
  una fuente caída.
- Nunca exponer en el borrador contenido personal/sensible de más (ej. asuntos de
  correo personales, resultados de búsqueda no laborales) — filtrar a lo
  estrictamente relacionado con trabajo antes de mostrarlo o registrarlo.
- Este skill no reemplaza al skill `/timesheet` (que sigue existiendo para redactar
  una entrada puntual a partir de la resolución de analista/redactor) — lo reutiliza
  como estándar de redacción.

## Limitaciones conocidas

- No existe una herramienta de historial de navegación de Chrome multi-día; "Google
  My Activity" es un sustituto parcial y depende de sesión iniciada y de que Google
  haya registrado esa actividad.
- Las sesiones de Claude Code activas solo cubren el momento en que se ejecuta el
  skill — no hay forma de recuperar conversaciones de sesiones ya cerradas de días
  anteriores. Para días pasados, la fuente principal termina siendo Gmail + dictado
  manual del usuario.
