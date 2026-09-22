---
name: client-test-instructive
category: "Client"
description: >
  Use this skill when the client has to validate a PoC or a development and needs a UAT
  test instructive built from the real Merge Request or Pull Request: it reads the actual
  changes (models, record rules, views, menus, security), translates them into business
  language test scenarios, and renders a self-contained interactive HTML where the client
  marks Pass or Fail, writes observations and returns it as PDF or as downloaded answers.
  Triggers on: "instructivo de pruebas", "guia de pruebas para el cliente", "checklist de
  aceptacion", "UAT", "pruebas funcionales de la PoC", "que el cliente valide", "test
  instructive", "guia de validacion", "manual de pruebas", "como prueba el cliente".

  Invocation modes:
    /client-test-instructive --mr <url>            → desde un MR de GitLab
    /client-test-instructive --pr <url>            → desde un PR de GitHub
    /client-test-instructive --mr <url> --pr <url> → PoC con módulo (MR) + instalación (PR)
metadata:
  source: "adapted from git.vauxoo.com/NicolasOT/hive-mind-agents (skills/client-test-instructive), original ecosystem by Nhomar Hernandez / Vauxoo — dropped the --task/vauxoo.json entry point (no client-portfolio config in this environment); GitLab/GitHub entry points unchanged"
---

# Client Test Instructive Skill

Genera un instructivo de pruebas funcionales (UAT) interactivo para el cliente a
partir del MR/PR de un desarrollo. Output: HTML con marca Vauxoo → archivo local
listo para compartir.

---

## Setup

- **Template**: `~/.claude/skills/_vx-shared/templates/checklist-interactivo.html` —
  variante rellenable (ya trae los `@media` responsivos; **no quitarlos** — debe verse
  bien en celular y computadora).
- **Patrón y gotchas**: `~/.claude/skills/_vx-shared/knowledge/client_html_deliverables.md`
  (consultar antes de tocar el HTML — explica el patrón de radios/checkboxes/campos
  editables y los gotchas de `name` único, comentarios anidados, `@media print`).
- **GitLab**: `glab` autenticado a `git.vauxoo.com`. Lectura de MR:
  `glab mr view <n> --repo <host>/<grupo>/<repo>` y `glab mr diff <n> --repo …`.
- **GitHub**: `gh` autenticado. Lectura de PR: `gh pr view <n> --repo <owner>/<repo>` y
  `gh pr diff <n> --repo …`.
- **Marca (fuentes + logo reales)**: después de rellenar los placeholders, correr
  `python3 ~/.claude/skills/_vx-shared/tools/vx_brand.py inline <archivo.html>` para
  embeber la tipografía Sora/Manrope real (base64, sin CDN) y el SVG oficial del logo.
  Requiere el plugin `agents-brand-guide-vauxoo@vauxoo-brand` instalado.
- **QA antes de entregar** (opcional pero recomendado): `python3
  ~/.claude/skills/_vx-shared/tools/html_report_lint.py <archivo.html>` y `python3
  ~/.claude/skills/_vx-shared/tools/es_ortografia_lint.py <archivo.html>`.

---

## Phase 1 — Reunir las fuentes de verdad

1. Resolver el/los origen(es): MR de GitLab (código del módulo) y/o PR de GitHub
   (instalación / submódulo).
2. Leer **título + descripción + diff completo** de cada fuente. Del diff extraer
   lo que cambia el comportamiento observable por el usuario:
   - Modelos y campos nuevos (qué dato gestiona el usuario).
   - **Record rules / security** → qué ve y qué NO ve cada perfil (aislamientos).
   - **Vistas / menús** → botones ocultos, campos readonly, apps disponibles.
   - **Constraints / validaciones** → mensajes de error que el cliente debe provocar.
   - Tests (`--test-tags`) → confirman qué escenarios ya están cubiertos y son fiables.
3. Anotar requisitos de entorno (extensiones, módulos dependientes, datos demo).

> No inventar comportamiento: cada escenario debe poder rastrearse a una línea del diff,
> un test o la descripción del MR/PR.

---

## Phase 2 — Traducir a escenarios de negocio

Para cada comportamiento, redactar un escenario con:

- **Objetivo** en lenguaje de negocio (sin "record rule", "xpath", etc. → glosario).
- **Pasos** clic-a-clic, en segunda persona, como los haría el usuario final.
- **Resultado esperado** concreto y verificable.

Traducir tecnicismos: `record rule` → "regla de visibilidad"; `timesheet` →
"parte de horas"; `constraint` → "validación que bloquea"; etc. Mantener un
**glosario** al inicio.

Definir los **datos de prueba** (usuarios/roles, registros demo) necesarios para
ejecutar los escenarios, alineados con la demo data del módulo cuando exista.

Marcar como **"VALIDAR EN STAGING"** todo lo que el código señale como dependiente
del entorno (xpaths de vistas, nombres de botones, lock-down de menús).

---

## Phase 3 — Renderizar el HTML interactivo

1. Leer `~/.claude/skills/_vx-shared/templates/checklist-interactivo.html`.
2. Reemplazar los `{{PLACEHOLDERS}}` del meta-box (Cliente, Referencia = MR/PR,
   Entorno = URL staging, Fecha, Elaborado por, Destinado a) y del KPI banner
   (# escenarios, # roles/perfiles, # apps, # puntos a validar en staging).
3. Duplicar el bloque `<!-- repeat scenario -->` una vez por escenario, con un
   **`name=""` ÚNICO por escenario** (`esc01`, `esc02`, …) — ver gotcha en el knowledge.
4. Construir las secciones de texto: Propósito (con "Origen del requerimiento" si
   aplica), Prerrequisitos (con checkboxes), Datos de prueba, Cómo reportar,
   Bitácora de firma.
5. Setear `{{DOWNLOAD_FILENAME}}` (p.ej. `Instructivo-<Cliente>-respuestas.html`).
6. **No modificar la capa de marca** (`:root` + clases); solo rellenar contenido.
7. Correr `vx_brand.py inline <archivo.html>` (ver Setup) para embeber fuentes y logo
   reales.

---

## Phase 4 — Guardar y entregar

Guardar en: `~/Downloads/INSTRUCTIVO_PRUEBAS_<MODULO>.html`.

Dejar como `[COMPLETAR]` los datos que aporta el cliente/admin (credenciales de
staging) salvo que el usuario los provea. Avisar al usuario de los `[COMPLETAR]`
pendientes y ofrecer abrir el HTML en el navegador para revisión antes de enviar.

---

## Reglas de negocio

- **Lenguaje de cliente**: nada de jerga interna (roles PM/BA/DEV, nombres de modelos).
- **Trazabilidad**: cada escenario rastreable al diff / test / descripción del MR/PR.
- **No fabricar**: si falta información para un escenario, marcarlo `[CONFIRMAR]` —
  nunca inventar pasos o resultados.
- **Usar la plantilla**: partir siempre de `checklist-interactivo.html`; no construir
  el HTML desde cero.
- **Interactivo y autocontenido**: un solo `.html`, sin dependencias; verificar que
  Pass/Fail, observaciones y los botones Guardar PDF / Descargar respuestas funcionan.

---

## Usage Examples

### Instructivo desde un MR de GitLab

```text
/client-test-instructive --mr https://git.vauxoo.com/vauxoo/cliente/-/merge_requests/51
→ lee los cambios reales del MR y genera el HTML interactivo con los
  escenarios de prueba en lenguaje de negocio.
```

### PoC con módulo y con instalación

```text
/client-test-instructive --mr <url-gitlab> --pr <url-github>
```
