# Consultas SQL y métricas de negocio

[Arquitectura](README.md) · [Diagrama](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

## 4. Consultas SQL y métricas

### Estrategia

n8n ejecuta consultas parametrizadas de **solo lectura** y prepara con sus resultados el contexto para los agentes. Salvo que indique lo contrario, `$1` representa el `transaction_id`. Siempre que estén disponibles, las métricas deberían incluir el período, la moneda, la fecha de referencia y la calidad del dato. `NULL` significa **desconocido o no calculado**, no cero.

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

**Historial del cliente y del comercio:** para estas consultas, reemplazá `$1` por `transaction.client` o `transaction.merchant`, según corresponda.

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

La consulta siguiente sigue el criterio de referencia de la documentación. **Ojo:** si `transaction_date` es de tipo `DATE`, no incluye la hora y no alcanza para comprobar una ventana móvil de 24 horas. Para aplicar la política estrictamente hace falta un timestamp.

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

Para **IVR**, se compara el monto de cada operación con USD 500. Para **App Móvil**, se compara el acumulado diario del cliente con USD 3.000. Antes de usar estos valores, hay que validar la moneda, el huso horario y qué estados de transacción se incluyen.

#### C. Reincidencia de contracargos en seis meses

**Esta consulta es ilustrativa: no la ejecutes sin revisar el esquema.** No está confirmado que exista la columna `historical_cases.opened_at`.

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

La consulta cuenta **casos de todas las transacciones del cliente**, no solo los asociados a la transacción investigada. Si hay varios reclamos, primero hay que identificar cuál se está analizando y confirmar cuál es la fuente oficial de la fecha de apertura.

#### D. Días desde la compra hasta el reclamo

**Esta consulta es ilustrativa: no la ejecutes sin confirmar que existe `opened_at`.**

```sql
SELECT t.transaction_id,
       hc.case_id,
       hc.opened_at::date - t.transaction_date::date AS days_to_claim
FROM transactions t
JOIN historical_cases hc ON hc.transaction_id = t.transaction_id
WHERE t.transaction_id = $1;
```

No uses `CURRENT_DATE`: el plazo se calcula **hasta la fecha en que se abrió el reclamo**, no hasta hoy. Si hay más de un caso, hay que tomar el `case_id` que corresponde.

### 4.3. Métricas comerciales: criterios de cálculo

La documentación indica que estas consultas se hacen antes de llamar al agente:

- **Tasa mensual de contracargos:** agrupar las transacciones del comercio por mes calendario, contarlas e identificar los contracargos a partir de los casos relacionados o de los estados de disputa. Hay que evitar contar más de una vez una transacción con varios registros. La tasa es **preliminar** hasta confirmar la definición oficial de POL-CB-004.
- **Volumen mensual:** agrupar por comercio y mes, calcular `COUNT(*)` y `SUM(amount)`, y separar los montos según el estado. No llamar al resultado `monthly_volume_usd` hasta confirmar que todos los importes estén expresados o convertidos a USD.

No incluyo una consulta comercial exacta porque la documentación explica la lógica, pero **no trae el SQL completo**. Tampoco supongo que exista una tabla de contracargos ni agrego columnas de fecha que no estén verificadas.

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

Este JSON es un **ejemplo del formato esperado**, no una respuesta real. n8n solo completa los campos que puede respaldar con datos confiables.

## Regla de integración

Las consultas se ejecutan en los nodos de PostgreSQL de n8n **antes** de llamar a los agentes. Ningún agente tiene acceso directo a SQL. `NULL` indica que el dato se desconoce o no está validado; no significa cero. Antes de usar las consultas que dependen de `opened_at`, hay que confirmar que esa columna exista y que tenga el significado esperado.

Ver [arquitectura](README.md) y [RAG](RAG.md).
