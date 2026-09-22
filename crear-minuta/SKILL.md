---
name: crear-minuta
description: >
  Genera una minuta formal de una llamada de seguimiento Vauxoo-Cliente a partir de
  la transcripción y la publica como Google Doc nativo (marca Vauxoo) en una carpeta
  de Drive indicada. Antes de generar, verifica automáticamente que el prompt local
  siga sincronizado con el artículo fuente de Knowledge (vauxooprod, id 160) y avisa
  si hay diferencias. Triggers: "crear minuta", "generar minuta", "minuta de la
  llamada", "acta de reunión con el cliente", "CrearMinuta".

  Invocación:
    /crear-minuta --transcript <link o ID de Google Doc con la transcripción> --folder <link o ID de carpeta de Drive> [--proyecto "Nombre del proyecto/cliente"] [--grabacion <link del video de la reunión>]
    /crear-minuta --check-prompt   → solo compara el prompt local contra Knowledge #160, no genera minuta

  El link de grabación es opcional como argumento: si no se da, el skill intenta
  detectarlo automáticamente dentro del propio Google Doc de transcripción (las notas
  de Gemini/Meet suelen traer su propio link "Grabación").
metadata:
  source: "prompt original: https://www.vauxoo.com/odoo/knowledge/160 (artículo 'Prompt para generar una minuta')"
---

# Crear Minuta

Skill personal (disponible en cualquier proyecto) que convierte la transcripción de
una llamada de seguimiento Vauxoo-Cliente en una minuta formal, y la entrega como
**Google Doc nativo** con la marca Vauxoo, creado directamente en la carpeta de Drive
que el usuario indique.

## Archivos de este skill

- `prompt.md` — copia local del prompt fuente (estructura de las 6 secciones de la
  minuta). Es la fuente de verdad que se usa para redactar cada minuta.
- `prompt_meta.json` — `last_synced_write_date` del artículo Knowledge #160 al momento
  de la última sincronización. Se usa para detectar si el artículo cambió.
- `template.html` — plantilla HTML con la marca Vauxoo (rojo `#AC0340`, tipografías
  Sora/Manrope, tabla de pendientes) y placeholders `{{...}}`.

## Fase 0 — Verificar que el prompt sigue vigente (automático, cada corrida)

1. Leer `prompt_meta.json` de este skill para obtener `last_synced_write_date`.
2. Consultar el artículo fuente vía Odoo MCP:
   ```
   mcp__odoo__search_read
     model: knowledge.article
     domain: [["id", "=", 160]]
     fields: id,name,body,write_date
     profile: vauxooprod
   ```
3. Comparar el `write_date` recibido contra `last_synced_write_date`:
   - **Igual** → seguir sin avisar nada, continuar a la Fase 1 (o terminar aquí si el
     modo es `--check-prompt`, informando que está sincronizado).
   - **Distinto** → el artículo cambió desde la última sincronización. Convertir el
     `body` (HTML) a texto plano y mostrarle al usuario un resumen breve de qué
     secciones parecen nuevas/modificadas respecto a `prompt.md`. Preguntar si quiere
     que actualices `prompt.md` y `prompt_meta.json` ahora con el contenido nuevo
     antes de continuar. **No sobreescribir `prompt.md` sin confirmación** — es un
     archivo que define el comportamiento futuro del skill.
   - Si el modo invocado es `--check-prompt`, detenerse aquí (no generar ninguna
     minuta): solo reportar si está sincronizado o no, y aplicar la actualización si
     el usuario la confirma.
4. Si `mcp__odoo` no está disponible o la consulta falla (ej. sin VPN/sesión), avisar
   brevemente y continuar con la copia local de `prompt.md` (no bloquear la
   generación de la minuta por esto).

## Fase 1 — Resolver los inputs

Requiere dos datos; si falta alguno, preguntar al usuario en vez de adivinar:

- `--transcript`: link o ID de un **Google Doc** que contiene la transcripción de la
  llamada. Extraer el ID con la regex `/d/([a-zA-Z0-9_-]+)` si viene como URL
  completa; si el usuario ya pasó solo el ID, usarlo tal cual.
- `--folder`: link o ID de la **carpeta de Drive** destino. Extraer el ID con la
  regex `/folders/([a-zA-Z0-9_-]+)` si viene como URL; si no, usar el valor tal cual
  como ID.
- `--proyecto` (opcional): nombre del proyecto/cliente para el título del documento y
  el encabezado. Si no se da, inferirlo de la transcripción o del nombre del
  workspace/proyecto actual; si tampoco es claro, usar "[Proyecto no especificado]".
- `--grabacion` (opcional): link al video de la grabación de la reunión.
  - Si el usuario lo da explícitamente al invocar, usar ese link tal cual (tiene
    prioridad sobre cualquier detección automática).
  - Si no lo da: **detectarlo automáticamente** del propio Google Doc de transcripción
    — las notas de Google Meet/Gemini casi siempre incluyen su propio link de
    grabación en una línea tipo `Registros de la reunión [Transcripción](...)
    [Grabación](<url>)`. Buscar ese patrón (markdown `[Grabación](url)` o `[Recording](url)`,
    case-insensitive) en el `contentSnippet` de `get_file_metadata` o en el contenido
    leído en el Paso 2 de esta fase, y si aparece, usar esa URL.
  - Si ni se dio por argumento ni se detectó en el documento, se omite sin más (no
    preguntar por él, no dejar la fila ni el placeholder vacíos).

Pasos:
1. `mcp__claude_ai_Google_Drive__get_file_metadata` sobre el ID de transcripción para
   confirmar que existe y ver su `mimeType`.
2. Leer el contenido con `mcp__claude_ai_Google_Drive__read_file_content` (soporta
   Google Docs, PDF, Word, ODT directamente). Si el mimeType no está soportado por esa
   herramienta, usar `download_file_content` y decodificar el base64 como texto.
3. `mcp__claude_ai_Google_Drive__get_file_metadata` sobre el ID de carpeta para
   confirmar que existe (mimeType `application/vnd.google-apps.folder`) y que es
   accesible. Si no es una carpeta o no es accesible, avisar y preguntar.

## Fase 2 — Redactar la minuta

Seguir **estrictamente** la estructura de `prompt.md` (Encabezado, Introducción,
Asistentes, Orden del Día, Acuerdos Logrados, Pendientes) usando como única fuente el
texto de la transcripción obtenida en la Fase 1. No inventar nombres, fechas ni
acuerdos que no estén en el texto — usar los placeholders `[...]` indicados en
`prompt.md` cuando falte información.

## Fase 3 — Renderizar el HTML de marca

1. Leer `template.html` de este skill.
2. Reemplazar cada `{{PLACEHOLDER}}` con el contenido redactado en la Fase 2:
   - `{{TITULO_DOC}}`, `{{PROYECTO}}`, `{{FECHA_REUNION}}`, `{{OBJETIVO_GENERAL}}`,
     `{{CAMBIOS_AGENDA}}` → texto plano.
   - `{{RESUMEN_EJECUTIVO}}` → uno o más `<p>`.
   - `{{LISTA_ASISTENTES}}` → `<ul><li>Nombre - Afiliación</li>...</ul>`.
   - `{{ORDEN_DEL_DIA}}` y `{{ACUERDOS_LOGRADOS}}` → `<ol><li>...</li></ol>`.
   - `{{FILAS_PENDIENTES}}` → filas `<tr><td>#</td><td>Tarea</td><td>Responsable</td><td>Fecha</td></tr>`,
     una por pendiente (si no hay pendientes, una sola fila con "Sin pendientes" en
     las 4 columnas fusionado con `colspan="4"`).
   - `{{FILA_GRABACION}}` → **condicional**:
     - Si el usuario dio `--grabacion`: `<tr><td class="etiqueta">Grabación</td><td><a href="<link>"><link></a></td></tr>`.
     - Si no lo dio: cadena vacía (no dejar la fila ni el placeholder, el recuadro de
       encabezado queda con las mismas 4 filas de siempre).
3. No modificar el `<style>` de la plantilla (marca Vauxoo ya definida); solo llenar
   contenido.

## Fase 4 — Crear el Google Doc en Drive

```
mcp__claude_ai_Google_Drive__create_file
  title: "Minuta - {{PROYECTO}} - {{FECHA_REUNION o fecha de hoy si no hay fecha}}"
  parentId: <ID de la carpeta resuelto en Fase 1>
  textContent: <HTML completo de la Fase 3>
  contentMimeType: "text/html"
```

Esto crea un **Google Doc nativo** (Drive convierte `text/html` automáticamente a
`application/vnd.google-apps.document`; no pasar `disableConversionToGoogleType`).

## Fase 5 — Entregar

Responder al usuario con:
- El link del Doc creado (del resultado de `create_file`, o construido como
  `https://docs.google.com/document/d/<id>/edit` si el resultado solo trae el `id`).
- Un recordatorio breve de que es un borrador generado por IA y debe revisarse antes
  de compartirlo con el cliente (ver "Consideraciones Adicionales" en `prompt.md`).
- Si en la Fase 0 se detectó que el prompt fuente cambió y el usuario no confirmó
  actualizarlo, recordárselo aquí también.

## Reglas

- **No compartir el Doc automáticamente** (no llamar `share_file`) — el usuario decide
  con quién compartirlo. Solo mencionar que quedó creado en la carpeta indicada.
- **No inventar** transcripción, asistentes, acuerdos ni fechas: todo debe rastrearse
  al texto de la transcripción.
- **No sobreescribir `prompt.md`** por una corrida normal — solo se actualiza tras
  confirmación explícita del usuario cuando la Fase 0 detecta drift.
- Si el usuario no da `--transcript` o `--folder`, preguntar antes de continuar; no
  usar una carpeta o documento "parecido" encontrado por búsqueda.

## Limitación conocida

Probado en la práctica (2026-08-25): la conversión HTML→Google Docs de Drive respeta
la **estructura** (encabezados, negritas, listas, tablas) pero **no** el `<style>`
inline — ni el color rojo de marca (`#AC0340`) ni las tipografías Sora/Manrope llegan
al documento final; sale con el estilo por defecto de Docs (texto negro, fuente Arial).
El documento queda correcto en contenido y organización, pero sin la marca visual
Vauxoo aplicada automáticamente. Ajustarlo a mano en Docs (seleccionar títulos y
aplicar color/fuente) toma un minuto. Si se necesita que la marca quede aplicada sin
intervención manual, la alternativa es cambiar este skill para usar la Google Docs API
(`documents.batchUpdate`) en vez de la conversión HTML de Drive — no implementado
todavía.

## Ejemplo de uso

```
/crear-minuta --transcript https://docs.google.com/document/d/1AbCdEf.../edit --folder https://drive.google.com/drive/folders/1XyZ... --proyecto "Implementación Odoo - Cliente ABC" --grabacion https://drive.google.com/file/d/1GrAbCdEf.../view
```

```
/crear-minuta --check-prompt
→ Solo compara prompt.md contra el artículo Knowledge #160 y reporta si hay drift.
```
