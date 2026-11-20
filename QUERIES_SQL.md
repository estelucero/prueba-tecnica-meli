# Consultas SQL y métricas de negocio

[Arquitectura](README.md) · [Diagrama](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

## 4. Consultas SQL y métricas

### Estrategia

n8n ejecuta consultas parametrizadas de **solo lectura** y construye payloads especializados. El parámetro `$1` representa `transaction_id` salvo indicación contraria. Las métricas deben llevar período, moneda, fecha de referencia y estado de calidad cuando estén disponibles. `NULL` significa **desconocido/no calculado**, no cero.

### 4.1. Recuperación de hechos

**Transacción investigada:**

```sql
SELECT row_to_json(t) AS transaction
FROM (
  SELECT * FROM public.transactions
  WHERE transaction_id = $1
  LIMIT 1
) t;
```

**Eventos técnicos:**

```sql
SELECT * FROM event_logs
WHERE transaction_id = $1;
```

**Casos vinculados a la transacción:**

```sql
SELECT * FROM historical_cases
WHERE transaction_id = $1;
```

**Historial del cliente y del comercio:** en estas dos consultas `$1` se reemplaza por el valor `transaction.client` o `transaction.merchant`, respectivamente.

```sql
SELECT * FROM transactions WHERE client = $1;
SELECT * FROM transactions WHERE merchant = $1;
```

### 4.2. Métricas relevantes

| Métrica | Política relacionada | Propósito |
|---|---|---|
| `monthly_transactions`, `monthly_chargebacks`, `chargeback_rate` | POL-CB-004 | Identificar comercios que superan umbrales de revisión (>1 %) o suspensión preventiva (>2 %). |
| `monthly_volume_usd` | POL-EXC-001 | Identificar comercios con volumen mensual >USD 1 millón y posibles condiciones especiales. |
| `transactions_24h`, `distinct_countries_24h` | POL-FRD-002 | Detectar actividad geográfica potencialmente anómala. |
| `channel_amount` | POL-FRD-003 | Revisar límites por operación IVR o acumulado diario en App Móvil. |
| `chargebacks_6m` | POL-CB-005 | Evaluar reincidencia del **cliente completo** en seis meses hasta la apertura del reclamo. |
| `days_since_transaction_to_claim` | POL-CB-001 | Evaluar el plazo de 120 días desde la transacción hasta la **apertura real** del reclamo. |

#### A. Actividad del cliente en 24 horas

La siguiente consulta reproduce el criterio de referencia de la documentación. **Advertencia:** si `transaction_date` es `DATE`, no tiene hora y no permite verificar una ventana móvil de 24 horas; se necesita un timestamp real para aplicar la política estrictamente.

```sql
SELECT COUNT(*) AS transactions_24h,
       COUNT(DISTINCT t.country) AS distinct_countries_24h
FROM transactions t
WHERE t.client = (
  SELECT client FROM transactions WHERE transaction_id = $1
)
AND t.transaction_date BETWEEN
  (SELECT transaction_date - INTERVAL '24 hours'
   FROM transactions WHERE transaction_id = $1)
  AND
  (SELECT transaction_date
   FROM transactions WHERE transaction_id = $1);
```

#### B. Acumulado diario por canal

```sql
SELECT t.channel,
       SUM(t.amount) AS total_amount,
       COUNT(*) AS total_transactions
FROM transactions t
WHERE t.client = (
  SELECT client FROM transactions WHERE transaction_id = $1
)
AND DATE(t.transaction_date) = (
  SELECT DATE(transaction_date)
  FROM transactions WHERE transaction_id = $1
)
GROUP BY t.channel;
```

Para **IVR** se contrasta el monto **individual** con USD 500; para **App Móvil**, el acumulado diario del cliente con USD 3.000. Se deben validar moneda, huso horario y estados elegibles.

#### C. Reincidencia de contracargos en seis meses

**SQL ilustrativo, no ejecutar sin verificar el esquema.** La columna `historical_cases.opened_at` no está confirmada.

```sql
SELECT COUNT(DISTINCT hc.case_id) AS chargebacks_6m
FROM historical_cases hc
JOIN transactions t ON t.transaction_id = hc.transaction_id
WHERE t.client = (
  SELECT client FROM transactions WHERE transaction_id = $1
)
AND hc.opened_at > (
  SELECT MIN(opened_at) - INTERVAL '6 months'
  FROM historical_cases WHERE transaction_id = $1
)
AND hc.opened_at <= (
  SELECT MIN(opened_at)
  FROM historical_cases WHERE transaction_id = $1
);
```

La consulta cuenta **casos de todas las transacciones del cliente**, no únicamente los de la transacción investigada. Debe definirse qué reclamo es el investigado si hay varios y cuál es la fuente oficial de aperturas.

#### D. Días desde la compra hasta el reclamo

**SQL ilustrativo, no ejecutar sin verificar `opened_at`.**

```sql
SELECT t.transaction_id,
       hc.case_id,
       hc.opened_at::date - t.transaction_date::date AS days_to_claim
FROM transactions t
JOIN historical_cases hc ON hc.transaction_id = t.transaction_id
WHERE t.transaction_id = $1;
```

No usar `CURRENT_DATE`: el período se calcula **hasta la fecha de apertura del reclamo**, no hasta hoy. Si hay varios casos, se evalúa el `case_id` correcto.

### 4.3. Métricas comerciales: criterios de cálculo

La documentación confirma consultas anteriores al agente para:

- **Tasa mensual de contracargos:** agrupar las transacciones del comercio por mes calendario, contar transacciones y detectar CB mediante casos asociados o estados de disputa. Evitar contar dos veces una operación con varios registros. Esta tasa es **preliminar** hasta validar la definición oficial de POL-CB-004.
- **Volumen mensual:** agrupar por comercio y mes, calcular `COUNT(*)` y `SUM(amount)` y separar montos por estado. No llamar `monthly_volume_usd` al resultado sin confirmar que todos los importes están denominados o convertidos a USD.

No se reproduce una query comercial exacta porque la documentación describe su lógica, pero **no incluye su SQL íntegro**. Tampoco se inventa una tabla de contracargos o columnas de fecha no verificadas.

### 4.4. Payload de métricas

```json
{
  "metrics": {
    "merchant": {
      "monthly_transactions": null,
      "monthly_chargebacks": null,
      "chargeback_rate": null,
      "monthly_volume_usd": null
    },
    "client": {
      "transactions_24h": null,
      "distinct_countries_24h": null,
      "chargebacks_6m": null,
      "app_daily_amount_usd": null
    },
    "case": {
      "days_from_transaction_to_claim": null
    }
  }
}
```

Este JSON es un **contrato ilustrativo**, no una respuesta real. n8n completa únicamente los campos que puede sustentar con datos confiables.

## Regla de integración

Las consultas se ejecutan en nodos PostgreSQL de n8n **antes** de los agentes. Ningún agente tiene herramientas SQL. `NULL` indica dato desconocido o no validado, no cero. Las queries que dependen de `opened_at` son ejemplos condicionados a verificar el esquema.

Ver [arquitectura](README.md) y [RAG](RAG.md).
