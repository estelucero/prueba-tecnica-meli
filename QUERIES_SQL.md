# Consultas SQL y métricas de negocio

[Arquitectura](README.md) · [Diagrama de la solución](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

## 1. Para qué utilizo las consultas SQL

Antes de ejecutar los agentes, n8n consulta PostgreSQL para recuperar la información de la transacción y calcular las métricas necesarias para la investigación. Elegí hacerlo de esta manera para que los agentes reciban datos preparados y no tengan que generar ni ejecutar sus propias consultas.

Las consultas utilizadas en esta etapa son de **solo lectura** y se ejecutan con parámetros. Salvo que indique lo contrario, `$1` representa el `transaction_id` que llega al webhook. Cuando se calculan métricas, también es importante identificar el período, la moneda y la fecha de referencia.

Hay una diferencia que considero fundamental: un valor `NULL` indica que el dato es desconocido, no está disponible o todavía no pudo calcularse. No debe interpretarse como cero, porque eso podría llevar a conclusiones incorrectas.

## 2. Recuperación de los datos del caso

### Transacción investigada

Primero recupero la transacción a partir de su identificador. En este caso, `row_to_json` permite devolver el registro como un objeto que luego se puede incorporar al payload de n8n.

```sql
SELECT row_to_json(t) AS transaction
FROM (
  SELECT * FROM public.transactions
  WHERE transaction_id = $1
  LIMIT 1
) t;
```

### Eventos técnicos

Los eventos registrados permiten revisar qué ocurrió durante el procesamiento de la operación y qué evidencia técnica existe.

```sql
SELECT * FROM event_logs
WHERE transaction_id = $1;
```

### Casos vinculados

También recupero los casos históricos asociados directamente con la transacción investigada.

```sql
SELECT * FROM historical_cases
WHERE transaction_id = $1;
```

### Historial del cliente y del comercio

Para analizar antecedentes más amplios, recupero las operaciones relacionadas con el mismo cliente o comercio. En estas consultas `$1` ya no representa el identificador de la transacción: debe recibir el valor de `transaction.client` o `transaction.merchant`, según corresponda.

```sql
SELECT * FROM transactions WHERE client = $1;
SELECT * FROM transactions WHERE merchant = $1;
```

## 3. Métricas utilizadas en las políticas

Además de recuperar registros, calculo indicadores que ayudan a los investigadores a evaluar reglas de negocio. Cada métrica tiene un propósito concreto y debe interpretarse según el período y las condiciones establecidas en su política.

| Métrica | Política | Para qué se utiliza |
|---|---|---|
| `monthly_transactions`, `monthly_chargebacks`, `chargeback_rate` | `POL-CB-004` | Revisar si un comercio supera los umbrales de revisión (más del 1 %) o suspensión preventiva (más del 2 %). |
| `monthly_volume_usd` | `POL-EXC-001` | Identificar comercios con un volumen mensual superior a USD 1 millón y evaluar posibles condiciones especiales. |
| `transactions_24h`, `distinct_countries_24h` | `POL-FRD-002` | Detectar patrones de actividad geográfica que podrían resultar anómalos. |
| `channel_amount` | `POL-FRD-003` | Comprobar límites por operación IVR o acumulados diarios en App Móvil. |
| `chargebacks_6m` | `POL-CB-005` | Evaluar la reincidencia del cliente durante los seis meses anteriores a la apertura del reclamo. |
| `days_since_transaction_to_claim` | `POL-CB-001` | Verificar el plazo de 120 días entre la transacción y la apertura efectiva del reclamo. |

### 3.1. Actividad del cliente durante 24 horas

Esta consulta cuenta las operaciones del cliente y los países distintos asociados con ellas dentro del período definido alrededor de la transacción investigada.

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

**Limitación:** si `transaction_date` es de tipo `DATE`, solamente guarda la fecha, no la hora. Por lo tanto, esta consulta sirve como referencia, pero no permite verificar con precisión una ventana móvil de 24 horas. Para aplicar estrictamente esa política se necesita un campo con fecha y hora (`timestamp`).

### 3.2. Acumulado diario por canal

Esta consulta agrupa las operaciones de un cliente por canal y calcula el importe total y la cantidad de transacciones correspondientes al día de la operación investigada.

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

La política establece que, para **IVR**, se debe comparar cada operación con el límite de USD 500. En **App Móvil**, se analiza el acumulado diario del cliente frente al límite de USD 3.000. Antes de usar estos resultados, es necesario confirmar la moneda, el huso horario y qué estados de transacción deben incluirse.

### 3.3. Reincidencia de contracargos en seis meses

El objetivo es contar los contracargos asociados con el cliente durante los seis meses anteriores a la apertura del reclamo que se está investigando. No alcanza con consultar los casos de una única transacción, porque la reincidencia corresponde al historial completo del cliente.

**Importante:** la siguiente consulta es ilustrativa. Todavía no está confirmado que el esquema de `historical_cases` incluya la columna `opened_at`, por lo que no debería ejecutarse sin verificarla.

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

Si una transacción tiene varios reclamos, primero hay que identificar el `case_id` que corresponde a la investigación. También es necesario confirmar cuál es la fuente oficial de la fecha de apertura. De lo contrario, el intervalo utilizado podría no corresponder al caso que realmente se está evaluando.

### 3.4. Días transcurridos entre la compra y el reclamo

Esta métrica permite comprobar si el reclamo se abrió dentro del plazo establecido por la política.

**Importante:** esta consulta también depende de que exista `historical_cases.opened_at` y de que ese campo represente la fecha real de apertura.

```sql
SELECT t.transaction_id,
       hc.case_id,
       hc.opened_at::date - t.transaction_date::date AS days_to_claim
FROM transactions t
JOIN historical_cases hc ON hc.transaction_id = t.transaction_id
WHERE t.transaction_id = $1;
```

Para calcular ese plazo no corresponde utilizar `CURRENT_DATE`, porque la comparación se hace entre la compra y la **apertura del reclamo**, no entre la compra y el día en que se ejecuta la investigación. Si existen varios casos, debe seleccionarse el que se está analizando.

## 4. Cálculo de métricas comerciales

Las métricas comerciales también se calculan antes de llamar al investigador correspondiente. En este caso, la documentación disponible define el criterio de cálculo, pero no incluye las consultas SQL completas. Por eso, dejo explicada la lógica sin inventar columnas o tablas que no estén confirmadas.

**Tasa mensual de contracargos.** Se agrupan las transacciones de un comercio por mes calendario, se cuenta la cantidad de operaciones y se identifican los contracargos vinculados. Hay que evitar duplicaciones cuando una transacción tiene varios registros asociados. La tasa debe considerarse preliminar hasta confirmar la definición exacta utilizada por `POL-CB-004`.

**Volumen mensual.** Se agrupan las operaciones por comercio y mes para obtener la cantidad de transacciones y la suma de los importes. También es importante distinguir los montos según el estado de las operaciones. El resultado solamente debería denominarse `monthly_volume_usd` cuando se confirme que todos los importes están expresados o convertidos a dólares.

## 5. Cómo se entregan las métricas a los agentes

n8n incorpora las métricas calculadas al payload de cada investigador. El siguiente JSON muestra un ejemplo de la estructura esperada; los valores `null` son ilustrativos y no representan el resultado de una consulta real.

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

El objetivo es completar únicamente los campos que puedan respaldarse con información confiable. Si no existe evidencia suficiente para calcular una métrica, es preferible conservar `null` antes que utilizar un cero que podría interpretarse como un resultado válido.

## 6. Decisión de integración

Todas estas consultas se ejecutan en los nodos de PostgreSQL de n8n **antes** de que comiencen a trabajar los agentes. Ningún agente necesita conectarse directamente a SQL. Esto permite controlar las consultas, preparar indicadores consistentes y entregar a cada investigador la información que le corresponde.

Queda pendiente validar los campos de fecha de apertura de reclamos y las definiciones exactas de algunas métricas para que las políticas se puedan aplicar sin supuestos. Esta limitación no debe ocultarse en el análisis: cuando un dato no se puede confirmar, el agente debe poder señalarlo expresamente.

Para ver cómo se utilizan estos resultados en el flujo completo, se puede consultar la [arquitectura principal](README.md) y el [diagrama de la solución](DIAGRAMA_SOLUCION.md).
