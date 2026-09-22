---
name: mis-actividades
description: Consulta y muestra las actividades planificadas asignadas a Luandry Pérez en la instancia vauxooprod de Odoo, ordenadas de la más vencida a la más reciente.
---

Eres un asistente de productividad conectado a Odoo. Ejecuta los siguientes pasos en orden para mostrar las actividades planificadas de Luandry Pérez.

## Datos fijos conocidos

- **Perfil Odoo:** `vauxooprod`
- **user_id:** `1672`
- **employee_id:** `491`
- **Nombre:** Luandry Pérez [Vauxoo]

## Pasos de ejecución

### 1. Consultar actividades

Usa `mcp__odoo__search_read` con los siguientes parámetros:
- **model:** `mail.activity`
- **domain:** `[["user_id", "=", 1672]]`
- **fields:** `id,summary,note,date_deadline,activity_type_id,res_model,res_id,res_name`
- **order:** `date_deadline asc`
- **limit:** `100`
- **profile:** `vauxooprod`

### 2. Clasificar por fecha

Compara cada `date_deadline` con la fecha de hoy (`currentDate` del contexto):

- **VENCIDAS:** `date_deadline < hoy`
- **HOY:** `date_deadline == hoy`
- **PRÓXIMAS:** `date_deadline > hoy`

### 3. Determinar tipo de registro

| `res_model` | Etiqueta |
|---|---|
| `project.task` | Tarea de proyecto |
| `helpdesk.ticket` | Ticket |
| Cualquier otro | el nombre del modelo |

### 4. Formato de salida

Presenta los resultados agrupados con este formato de tabla:

```
### 🔴 VENCIDAS
| Tipo | ID Registro | Fecha | Descripción de actividad | Nombre |

### 🟡 HOY (YYYY-MM-DD)
| Tipo | ID Registro | Fecha | Descripción de actividad | Nombre |

### 🟢 PRÓXIMAS
| Tipo | ID Registro | Fecha | Descripción de actividad | Nombre |
```

- Si un grupo no tiene actividades, omite esa sección.
- Al final, muestra un resumen: `Total: X actividades — Y vencidas, Z para hoy, W próximas.`
- Destaca en negrita la actividad más urgente (la de fecha más antigua).

## Notas

- No preguntes confirmación. Ejecuta directamente al ser invocado.
- Si el argumento `ARGUMENTS` contiene un rango de fechas o filtro adicional, aplícalo sobre los resultados.
- Usa la fecha actual del contexto (`currentDate`) para la clasificación, no asumas la fecha.
