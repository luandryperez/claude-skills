# Knowledge Base — Odoo Guru

Memoria de aprendizaje técnico acumulada por el skill `odoo-guru`.
Cada hallazgo se registra aquí con referencia exacta al código fuente para consultas futuras.

## Formato de entrada

```
### [MODULO] — Tema / Lógica estudiada
- **Versión:** 17 / 18 / 19
- **Archivo(s):** ruta/al/archivo.py:línea
- **Pregunta o contexto:** ¿Qué se quería entender?
- **Hallazgo:** Explicación técnica del comportamiento
- **Señal de reconocimiento:** Cómo identificar este patrón en el futuro
```

---

## Odoo 17

<!-- Los hallazgos de V17 se agregan aquí -->

---

## Odoo 18

<!-- Los hallazgos de V18 se agregan aquí -->

---

## Odoo 19

### [ACCOUNT_ACCOUNTANT] — Botón "Establecer Cuenta" en líneas de extracto bancario
- **Versión:** 19
- **Archivo(s):**
  - `enterprise/19.0/account_accountant/models/account_bank_statement.py:944` → método servidor
  - `enterprise/19.0/account_accountant/static/src/components/bank_reconciliation/button_list/button_list.js:546` → condición de visibilidad
  - `button_list.js:170` → llamada JS `_setAccountOnReconcileLine`
- **Pregunta o contexto:** ¿Para qué sirve el botón "Establecer Cuenta" en las líneas de los extractos bancarios? ¿Qué hace internamente?
- **Hallazgo:**
  El botón aparece cuando `!statementLineData.account_id` (la línea aún tiene la cuenta puente/suspense como contrapartida y no se ha definido cuenta definitiva). Al hacer clic se llama `set_account_bank_statement_line(aml_id, account_id)` en el servidor. Lo que hace:
  1. Recibe el ID de la `account.move.line` contrapartida (la que está en la cuenta de suspense) y el `account_id` destino elegido por el usuario.
  2. Llama `_create_account_model_fee` — si la línea es un cobro casi conciliado con saldo residual pequeño, crea/actualiza un modelo de conciliación automático para "comisiones bancarias" del diario.
  3. Cambia `base_line.account_id = account` — reemplaza la cuenta puente (suspense) por la cuenta destino real.
  4. Si la cuenta tiene impuestos, recalcula las líneas de impuestos.
  5. **Rama clave:** Si la cuenta destino es `asset_receivable`, `liability_payable`, o es la cuenta de liquidez/suspense del diario → sólo llama `_post_matching_done_confirmation()` y termina (no crea regla automática).
  6. **Para cualquier otra cuenta** (gastos, ingresos, etc.) → llama `_create_automatic_reconciliation_model` que busca si hay ≥2 líneas anteriores del mismo diario con el mismo patrón en `payment_ref`; si las hay, crea automáticamente un `account.reconcile.model` con regex para aplicar esa cuenta en el futuro.
  
  **En resumen:** el botón no toca la cuenta del diario bancario (`default_account_id`). Sólo reemplaza la cuenta puente (suspense) de la contrapartida con la cuenta real de destino. Es el atajo para conciliar sin vincular a una factura.
- **Señal de reconocimiento:** Línea de extracto abierta, con la cuenta de suspense como contrapartida, sin factura/pago vinculado. El botón aparece junto a "Cobrar", "Pagar", "Conciliar".

---

## Transversal (aplica a múltiples versiones)

<!-- Patrones del ORM, decoradores, herencia, etc. que aplican en general -->
