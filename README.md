# Sistema multiagente de investigación de contracargos

[Arquitectura](README.md) · [Diagrama](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

> Documento principal de arquitectura. Para profundizar, consultá los documentos vinculados.

## 1. Objetivo

Investigar una transacción a partir de su `transaction_id`, reunir evidencia operativa y comercial, identificar políticas internas relevantes, contrastarlas con antecedentes y generar una **recomendación fundamentada** en un informe HTML. El sistema asiste a la investigación: **no ejecuta** reembolsos, bloqueos ni cambios de estado.

## 2. Arquitectura

La arquitectura separa deliberadamente **datos determinísticos** de **razonamiento semántico**:

| Componente            | Función                                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **n8n**               | Recibe el webhook, valida la solicitud, ejecuta consultas, compone contextos, orquesta agentes y devuelve HTML.                      |
| **PostgreSQL**        | Fuente estructurada para transacciones, eventos, casos y métricas. Todas las consultas se ejecutan **antes** de invocar los agentes. |
| **Qdrant**            | Recupera políticas internas y casos históricos por similitud semántica mediante RAG.                                                 |
| **Embeddings**        | Convierte consultas y documentos a vectores comparables. El export anterior utiliza `text-embedding-3-small`.                        |
| **Siete agentes LLM** | Investigan por especialidad, validan políticas, critican, recomiendan una resolución y redactan el informe.                          |

**Regla de diseño:** ningún agente tiene herramientas SQL. Las métricas se calculan en nodos PostgreSQL de n8n y se suministran como hechos. Los agentes investigadores sí disponen de búsquedas semánticas de políticas y precedentes.

## 3. Puesta en marcha

### Levantar los servicios

Desde la raíz del proyecto, iniciá los servicios con Docker Compose:

```bash
docker compose up -d
```

Esto inicia n8n, PostgreSQL y Qdrant. La interfaz de n8n queda disponible en [http://localhost:5678](http://localhost:5678).

### Importar y ejecutar los workflows

Importá en n8n los archivos JSON ubicados en `workflows/`. Ejecutalos en este orden:

1. **`meli-ingesta-postgresql.json`** (`meli-ingesta-postgresql`): carga los datos estructurados en PostgreSQL.
2. **`meli-database-vector.json`** (`meli-database-vector`): prepara la base vectorial en Qdrant.
3. **`meli-workflow.json`** (`meli-workflow`): ejecuta el flujo completo de investigación.

Ejecutá primero los dos workflows de carga y configuración; el workflow completo depende de los datos disponibles en PostgreSQL y Qdrant.

### Responsabilidades de los agentes

1. **Comercial:** identifica obligaciones, excepciones y condiciones aplicables al comercio, con contexto de su historial y métricas mensuales.
2. **Operativo:** revisa controles antifraude, autorizaciones, eventos, plazos, documentación y acciones del equipo o sistema.
3. **Cliente:** evalúa derechos, beneficios, elegibilidad, restricciones y obligaciones del titular.
4. **Analista integrador:** valida **todas** las políticas recuperadas por los especialistas, distingue activación, ejecución y cumplimiento, y separa el estado observado del tratamiento esperado.
5. **Crítico y mejoras:** cuestiona decisiones y actuaciones pasadas; propone medidas preventivas para expedientes abiertos.
6. **Resolutor:** produce una recomendación final concreta, justificada con evidencia, políticas y críticas.
7. **Redactor HTML:** transforma el resultado en un documento HTML5 válido sin alterar los hechos.

Los tres investigadores entregan un JSON con `applicable_policies`, que incluye **todas las políticas evaluadas de su ámbito**, incluso `not_applicable` y `undetermined`. El analista conserva esa cobertura en `policy_decisions`, distinguiendo `activation_decision`, `execution_status`, `compliance_assessment` y `case_impact`.

## Documentación complementaria

- **[Diagrama de la solución](DIAGRAMA_SOLUCION.md):** flujo end-to-end, bifurcaciones y responsabilidades de cada etapa.
- **[RAG](RAG.md):** recuperación semántica, embeddings, filtros, chunking y limitaciones.
- **[Consultas SQL y métricas](QUERIES_SQL.md):** queries parametrizadas, propósito de cada consulta y supuestos de datos.

## 5. Integridad y límites de la investigación

- **Estado de transacción ≠ estado de contracargo.** Una operación aprobada no prueba que el expediente esté abierto o cerrado.
- **Activación ≠ ejecución ≠ cumplimiento.** Superar un umbral no demuestra que el sistema haya ejecutado —o omitido— una acción.
- **Ausencia de logs ≠ ausencia de acción.** Cuando faltan pruebas, registrar `not_determined`.
- **Casos similares ≠ evidencia actual.** Los antecedentes sirven para comparar, no para copiar veredictos.
- **Cronologías inconsistentes:** señalar contradicciones de fechas; no convertirlas automáticamente en fraude.
- **Cobertura RAG:** `topK=5` por consulta no garantiza exhaustividad; una recuperación determinística del catálogo sería una mejora para auditorías de cumplimiento.

---

**Resumen:** PostgreSQL suministra hechos y métricas calculadas previamente; Qdrant permite recuperar políticas y precedentes por significado; tres investigadores producen evaluaciones trazables; el analista integra todas las políticas; el crítico revisa y mejora; el resolutor recomienda; el redactor entrega el informe HTML.

**Fuente:** documentación del proyecto proporcionada por el usuario, _Sistema multiagente de investigación de contracargos_, 20/11/2026. Los apartados señalados como ilustrativos o pendientes no están certificados por un workflow actualizado.
