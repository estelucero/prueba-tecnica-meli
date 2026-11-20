# Recuperación de información con RAG

[Arquitectura](README.md) · [Diagrama de la solución](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

## 1. Por qué utilizo RAG

Incorporé RAG (_Retrieval-Augmented Generation_, o generación aumentada por recuperación) para que los investigadores puedan consultar documentación relevante durante el análisis de un contracargo. La idea es que no dependan únicamente del conocimiento del modelo ni de políticas copiadas dentro de un prompt.

En este proyecto, RAG se utiliza principalmente para recuperar **políticas internas** y **casos históricos similares**. Es importante aclarar que esta búsqueda complementa los datos de PostgreSQL, pero no los reemplaza: una política puede encontrarse por similitud semántica, mientras que el estado de una transacción o la cantidad de operaciones de un cliente se obtienen mediante consultas exactas.

## 2. Cómo se recupera la información

```mermaid
flowchart LR
    I[Hechos y métricas SQL] --> A[Investigador]
    A --> S[Consulta en lenguaje natural]
    S --> E[Embedding de la consulta]
    E --> Q[(Qdrant)]
    Q --> D[Fragmentos y metadatos de origen]
    D --> A
    A --> O[Políticas evaluadas, evidencia y dudas]
```

Cada investigador parte de los datos y las métricas que ya preparó n8n. Cuando necesita consultar una política o buscar antecedentes, realiza una búsqueda en lenguaje natural. Esa consulta se convierte en un embedding, que es una representación numérica de su significado.

Qdrant compara ese vector con los documentos indexados y devuelve los resultados más cercanos, junto con sus metadatos. Después, el agente utiliza esa información para evaluar qué políticas podrían aplicar y qué antecedentes vale la pena considerar.

No alcanza con que dos documentos sean parecidos para afirmar que una política aplica o que un caso anterior sirve como prueba. El agente tiene que revisar las condiciones concretas y relacionarlas con la evidencia de la transacción investigada.

## 3. Configuración documentada

En un export anterior del workflow quedaron registrados los siguientes parámetros:

| Parámetro                     | Valor documentado                                          |
| ----------------------------- | ---------------------------------------------------------- |
| Colección                     | `chargeback_knowledge`                                     |
| Modelo de embeddings          | `text-embedding-3-small`                                   |
| Dimensión habitual del modelo | 1536, pendiente de confirmar en la configuración efectiva  |
| Filtro para políticas         | `metadata.source_type = internal_policy`                   |
| Filtro para casos históricos  | `metadata.source_type = historical_case`                   |
| Resultados por búsqueda       | `topK = 5`                                                 |
| Distancia vectorial           | Cosine, planteada en el diseño y pendiente de verificación |

Estos valores corresponden a la documentación disponible del export anterior. Antes de tomarlos como la configuración actual, habría que comprobarlos en el workflow de ingesta y en la colección activa de Qdrant.

## 4. Cómo organizo los documentos

Para las políticas, el criterio de diseño es conservar cada norma completa cuando su tamaño lo permita. De esa manera, el agente puede recuperar en un mismo fragmento el identificador, los requisitos, las excepciones y las condiciones que necesita evaluar.

Con los casos históricos, busco mantener juntos los hechos relevantes y su resolución, siempre que el tamaño del documento lo permita. La idea es evitar que una fragmentación excesiva haga perder información necesaria para interpretar el antecedente.

Utilice estos metodos debido a que optimice la base de datos vectorial a partir de las consultas que iba a realizar en un futuro el agente.

Durante la búsqueda, los investigadores pueden utilizar distintas expresiones y sinónimos para encontrar políticas relacionadas. También deben evitar repetir documentos y buscar antecedentes por las condiciones relevantes del caso, no solamente por el motivo del reclamo.

## 5. Por qué utilizo PostgreSQL y Qdrant

Al principio puede parecer que ambas bases permiten recuperar la misma información, porque Qdrant también admite filtros exactos sobre los metadatos. Sin embargo, las utilizo para resolver necesidades diferentes.

| Necesidad                                         | Herramienta | Motivo                                                                           |
| ------------------------------------------------- | ----------- | -------------------------------------------------------------------------------- |
| Buscar una transacción mediante `transaction_id`  | PostgreSQL  | Necesito identificar un registro exacto.                                         |
| Recuperar logs y casos asociados                  | PostgreSQL  | Es importante respetar las relaciones entre los datos.                           |
| Calcular cantidades, acumulados, tasas y períodos | PostgreSQL  | SQL permite hacer agregaciones de forma controlada.                              |
| Encontrar una política escrita con otras palabras | Qdrant      | Me interesa la similitud de significado, no solo una coincidencia textual.       |
| Buscar antecedentes narrativamente parecidos      | Qdrant      | Los documentos pueden describir situaciones similares con expresiones distintas. |

Qdrant permite utilizar `/points/scroll` para filtrar registros por metadatos, como un identificador o un tipo de documento. En ese caso, la búsqueda es exacta sobre el payload y **no utiliza la similitud vectorial**.

Eso no significa que Qdrant reemplace a PostgreSQL. Para este proyecto, PostgreSQL sigue siendo la fuente de los datos estructurados, las relaciones y los cálculos. Qdrant se utiliza como una herramienta complementaria para buscar documentación por significado.

## 6. Limitaciones y mejoras pendientes

Una búsqueda vectorial no garantiza recuperar todas las políticas relevantes. Por ejemplo, `topK = 5` devuelve hasta cinco resultados en cada consulta, pero una investigación podría necesitar evaluar una norma que no haya quedado entre esos resultados. Si el objetivo fuera hacer una auditoría exhaustiva de cumplimiento, sería conveniente complementar la búsqueda semántica con una recuperación determinística del catálogo de políticas.

También quedó pendiente integrar el RAG a un proceso de mejora continua. El sistema ya conserva las salidas de los agentes y las versiones de los prompts, pero todavía no cuenta con un workflow que revise automáticamente los resultados validados y actualice la base de conocimiento. Cualquier incorporación de políticas corregidas o nuevos antecedentes debería pasar por controles para evitar que una conclusión incorrecta generada por la IA termine utilizándose como referencia en investigaciones futuras.

Para entender cómo se integra RAG en el proceso completo, se puede consultar el [diagrama de la solución](DIAGRAMA_SOLUCION.md). La preparación de los datos estructurados se explica en [consultas SQL](QUERIES_SQL.md).
