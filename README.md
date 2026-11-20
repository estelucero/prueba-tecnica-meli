# Sistema multiagente para la investigación de contracargos

[Arquitectura](README.md) · [Diagrama de la solución](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

## 1. Objetivo del proyecto

El objetivo de este proyecto es automatizar la investigación de contracargos a partir del identificador de una transacción (`transaction_id`). Para hacerlo, desarrollé un sistema que reúne los datos de la operación, consulta las políticas internas, busca antecedentes similares y analiza toda esa información mediante agentes de inteligencia artificial con responsabilidades diferentes.

Como resultado, el sistema genera un informe HTML que explica qué se investigó, cuáles fueron los hallazgos más importantes, qué decisión recomienda la IA y qué evidencia respalda esa recomendación. También incluye observaciones sobre el proceso y propuestas de mejora.

Una distinción importante es que la recomendación de la IA no representa necesariamente el estado registrado en la base de datos. En el JSON final utilizo `report_status` para indicar la conclusión del análisis: `ACCEPT`, `REJECT` o `PENDING`. Por ejemplo, si el informe muestra `REJECT`, significa que la IA recomienda rechazar el contracargo, no que ya se haya ejecutado ese rechazo.

Por eso, el sistema funciona como una herramienta de apoyo a la toma de decisiones. No realiza reembolsos, bloqueos ni modificaciones de estado en los sistemas operativos.

## 2. Arquitectura y decisiones de diseño

Decidí separar la recuperación de datos de su interpretación. Por un lado, PostgreSQL permite consultar información concreta y calcular métricas. Por otro, Qdrant permite encontrar políticas y casos históricos relacionados por su significado, incluso cuando están redactados con palabras distintas. n8n se encarga de coordinar todo el proceso.

| Componente | Para qué lo utilizo |
|---|---|
| **n8n** | Recibe la solicitud, valida el identificador, ejecuta las consultas, prepara los datos para cada agente y devuelve el informe. |
| **PostgreSQL** | Almacena información estructurada, como transacciones, eventos y casos históricos. También guarda las salidas de los agentes y las versiones de sus prompts. |
| **Qdrant** | Permite recuperar políticas internas y antecedentes mediante búsquedas semánticas. |
| **Embeddings** | Convierten los documentos y las consultas en vectores para comparar su significado. En la configuración documentada anteriormente se utilizó `text-embedding-3-small`. |
| **Agentes LLM** | Investigan el caso desde distintas perspectivas, integran los resultados, revisan la recomendación y preparan el informe final. |

Una de las principales decisiones fue evitar que los agentes consultaran PostgreSQL directamente. Las consultas SQL y los cálculos se ejecutan desde n8n antes de invocar a los investigadores, que reciben un contexto preparado para su especialidad. Esto permite mantener un mayor control sobre las consultas, reducir la duplicación de trabajo y separar los datos obtenidos de las interpretaciones que hace la IA.

También dividí las responsabilidades entre siete agentes. La intención es que cada uno tenga una función concreta y que las últimas etapas no vuelvan a decidir algo que ya resolvió el Analizador.

Para ver el recorrido completo de una investigación, se puede consultar el [diagrama de la solución](DIAGRAMA_SOLUCION.md). La documentación del [RAG](RAG.md) y las [consultas SQL](QUERIES_SQL.md) explica con más detalle cómo se obtiene la información.

## 3. Puesta en marcha

### Levantar los servicios

Desde la carpeta principal del proyecto se pueden iniciar los servicios definidos en `compose.yml` con Docker Compose:

```bash
docker compose up -d
```

Entre esos servicios se encuentran n8n, PostgreSQL y Qdrant. Una vez iniciado n8n, se puede acceder desde [http://localhost:5678](http://localhost:5678).

### Importar los workflows

Los archivos JSON de la carpeta `workflows/` se importan en n8n y se ejecutan en el siguiente orden:

1. **`meli-ingesta-postgresql.json`**: carga los datos estructurados en PostgreSQL.
2. **`meli-database-vector.json`**: prepara los documentos que se utilizarán en Qdrant.
3. **`meli-workflow.json`**: ejecuta la investigación completa de un contracargo.

Es necesario ejecutar primero las ingestas porque el flujo principal depende tanto de los registros almacenados en PostgreSQL como de la información disponible en Qdrant.

## 4. Responsabilidades de los agentes

Para organizar la investigación, separé el análisis en distintas etapas. Los tres primeros agentes trabajan sobre perspectivas diferentes del mismo caso. Después, los agentes restantes integran la información, determinan una recomendación, revisan el análisis y generan el informe.

### Investigador comercial

Revisa las políticas y condiciones asociadas al comercio, como obligaciones, restricciones, beneficios y criterios de revisión. Utiliza el historial del comercio y las métricas calculadas previamente por n8n. Su objetivo es identificar qué reglas pueden resultar relevantes y dejar documentada la evidencia encontrada, sin determinar por su cuenta la decisión final del contracargo.

### Investigador operativo

Analiza el procesamiento de la operación, los controles antifraude, las autorizaciones, los eventos registrados, los plazos y la documentación disponible. Busca establecer qué condiciones operativas se cumplen y cuáles requieren una verificación adicional. Si falta un registro, no asume automáticamente que una acción no se realizó.

### Investigador del cliente

Evalúa las políticas que afectan al titular de la transacción, incluyendo derechos, beneficios, requisitos, plazos y restricciones. Su función es aportar la perspectiva del cliente y reconocer las condiciones que pueden influir en el tratamiento del reclamo.

### Analizador

Recibe los resultados de los tres investigadores y evalúa todas las políticas que identificaron. Determina cuáles se cumplen, cuáles se incumplen, cuáles no aplican y cuáles no pueden verificarse con la información disponible.

Lo más importante de este agente es que no decide simplemente por la cantidad de reglas cumplidas o incumplidas. Tiene que interpretar el propósito de cada política y cómo se relaciona con las demás. Por ejemplo, una condición de elegibilidad puede ser determinante para resolver el contracargo, mientras que un control operativo o un beneficio puede generar una obligación independiente.

A partir de esa evaluación, recomienda `ACCEPT`, `REJECT` o `PENDING` y explica qué políticas y evidencias justifican su conclusión.

### Crítico y mejoras

El Crítico revisa la recomendación del Analizador para detectar problemas de interpretación, riesgos, inconsistencias o debilidades en la justificación. También propone medidas correctivas y preventivas que podrían mejorar el proceso.

No vuelve a resolver el contracargo ni reemplaza automáticamente la decisión anterior. Si encuentra una observación relevante, la deja documentada y explica por qué debería tenerse en cuenta. De esta manera, sus propuestas complementan el análisis sin mezclar la evaluación del caso con la mejora del proceso.

### Summarize

Este agente reúne los datos originales de la transacción, el resultado del Analizador y las observaciones del Crítico para construir un único JSON final. No vuelve a buscar información ni a evaluar políticas.

Una de sus responsabilidades principales es dejar explícita la decisión recomendada por la IA mediante `report_status`. Ese valor debe salir de `analysis.chargeback_decision.decision`, sin volver a calcularse ni modificarse durante el resumen.

### Redactor HTML

Toma el JSON generado por Summarize y lo transforma en un informe HTML5 que se pueda visualizar desde n8n. Organiza la información, destaca el estado recomendado y explica los hallazgos, la evidencia, las mejoras y los próximos pasos. Su tarea es presentar los resultados, no realizar un análisis adicional ni cambiar la decisión.

## 5. Cómo se construye la decisión final

Los investigadores entregan las políticas identificadas y sus evidencias. El Analizador consolida esas evaluaciones en `policy_evaluations`, utilizando estados como `COMPLIANT`, `NON_COMPLIANT`, `NOT_APPLICABLE` y `UNDETERMINED`. Después genera `chargeback_decision`, donde queda registrada la decisión recomendada junto con su justificación.

El Crítico puede señalar debilidades o proponer mejoras sobre esa conclusión, pero no la reemplaza. Finalmente, Summarize reúne toda la información y copia la decisión del Analizador en `report_status`. Así, el informe mantiene una única conclusión identificable y permite distinguirla de las observaciones posteriores.

El JSON final contiene estas secciones:

| Campo | Contenido |
|---|---|
| `report_status` | Recomendación principal de la IA: `ACCEPT`, `REJECT` o `PENDING`. |
| `case_information` | Información de la transacción, el cliente, el comercio y el estado original. |
| `case_summary` | Resumen del caso y de los motivos documentados de la investigación. |
| `investigation` | Evaluaciones realizadas y hallazgos relevantes. |
| `similar_cases` | Antecedentes comparables que ya fueron identificados. Si no existen, se deja vacío. |
| `resolution` | Justificación de la recomendación y políticas y evidencias determinantes. |
| `improvements` | Observaciones del Crítico y propuestas para mejorar el proceso. |
| `next_actions` | Acciones sugeridas para revisar, autorizar o dar seguimiento al caso. |

## 6. Criterios y límites de la investigación

Hay algunas decisiones que tomé para evitar interpretaciones incorrectas de los resultados:

- **La recomendación no equivale a una acción ejecutada.** Un `report_status` con valor `REJECT` indica que la IA recomienda rechazar el contracargo, pero no confirma que el sistema ya lo haya rechazado.
- **El estado registrado de la transacción no define por sí solo la decisión.** Una transacción con un contracargo iniciado puede terminar con una recomendación de aceptación, rechazo o revisión pendiente.
- **Las políticas deben interpretarse según su función.** No todas tienen el mismo peso ni determinan el mismo tipo de consecuencia. Una restricción de elegibilidad no es equivalente a un beneficio o a un plazo operativo.
- **Una condición cumplida no demuestra que se haya realizado la acción correspondiente.** Si se supera un umbral que exige revisión, todavía hace falta evidencia para confirmar si esa revisión se efectuó.
- **La ausencia de logs no demuestra que algo no haya ocurrido.** Cuando no hay información suficiente, una política puede quedar como `UNDETERMINED`. Eso no obliga a que toda la decisión sea `PENDING` si otra condición ya permite resolverla.
- **Los antecedentes similares sirven como referencia, no como prueba del caso actual.** No se debe copiar la resolución de otro contracargo sin evaluar la evidencia propia.
- **Las críticas deben conservarse.** Si el Crítico cuestiona una conclusión, Summarize tiene que incluir esa observación sin modificar silenciosamente la decisión del Analizador.
- **Los importes del conjunto de datos están expresados en USD.** Aun así, los cálculos deben respetar la definición de cada métrica y su período.
- **El RAG tiene límites de cobertura.** Una búsqueda con `topK = 5` puede recuperar documentos relevantes, pero no garantiza revisar todas las políticas existentes.

## 7. Alcance y deuda técnica

### Qué llegué a implementar

Durante el desarrollo pude completar el flujo principal de investigación de contracargos, desde la carga de datos hasta la generación del informe HTML. Busqué que el sistema no dependiera solamente de las respuestas de un modelo de lenguaje, sino que pudiera trabajar con registros concretos, documentación de respaldo y una división clara de responsabilidades.

Implementé los workflows de ingesta para cargar información estructurada en PostgreSQL y preparar las políticas y los casos históricos utilizados por el RAG. También incorporé un webhook que permite iniciar una investigación mediante el identificador de una transacción.

Para recuperar documentación relevante, integré Qdrant y embeddings. Esto permite que los investigadores consulten políticas y antecedentes relacionados con el caso sin incluir toda la documentación dentro de los prompts.

En la parte de análisis desarrollé los agentes comercial, operativo y del cliente, junto con el Analizador, el Crítico, Summarize y el Redactor HTML. n8n coordina sus ejecuciones y prepara los datos y las métricas antes de invocar a los agentes, evitando que cada uno tenga que consultar PostgreSQL por separado.

También avancé en la **observabilidad y trazabilidad**. Ahora las salidas de los agentes se guardan en PostgreSQL, lo que permite revisar qué produjo cada etapa de una investigación. Además, incorporé tablas para almacenar y versionar los prompts y distinguir las versiones disponibles de las que se utilizan en producción. Esto deja una base para analizar el comportamiento del sistema y seguir los cambios en sus instrucciones.

Por último, el Agente Crítico ya permite identificar inconsistencias y proponer mejoras concretas. Sin embargo, esas recomendaciones todavía no se convierten automáticamente en modificaciones del sistema.

### Qué quedó pendiente

La principal funcionalidad que no llegué a desarrollar fue el **circuito de mejora continua**.

Aunque ahora se almacenan los resultados de los agentes y sus prompts, todavía no implementé un workflow que tome esa información, evalúe la calidad de las respuestas e identifique de manera sistemática los errores que se repiten. Tampoco desarrollé un mecanismo para comparar versiones de prompts con criterios de evaluación definidos, validar los cambios y promover una nueva versión a producción.

También quedó pendiente la actualización controlada del RAG a partir de investigaciones y correcciones validadas. Por ejemplo, cuando se identifica una interpretación incorrecta de una política o se incorpora un nuevo antecedente confiable, sería útil contar con un proceso que revise esa información antes de incorporarla a la base de conocimiento. No sería conveniente que el sistema agregara automáticamente cualquier conclusión generada por la IA, porque podría terminar reforzando errores.

Considero que la observabilidad y el versionado de prompts son avances importantes porque permiten conservar información que antes se perdía al finalizar cada ejecución. Sin embargo, **guardar resultados no significa que el sistema esté aprendiendo de ellos**. Para cerrar ese ciclo todavía hace falta incorporar evaluaciones, feedback, validaciones y mecanismos controlados para actualizar tanto los prompts como la documentación recuperada por RAG.

En resumen, pude implementar la investigación multiagente, la generación del informe y el registro de los resultados. La mejora continua quedó planteada como una siguiente etapa de desarrollo, no como una funcionalidad que ya esté funcionando.

## Documentación complementaria

- [Diagrama de la solución](DIAGRAMA_SOLUCION.md): muestra cómo circula la información desde el webhook hasta el informe.
- [RAG](RAG.md): explica cómo se recuperan las políticas y los antecedentes y por qué se utiliza una base vectorial.
- [Consultas SQL](QUERIES_SQL.md): documenta los datos y las métricas que se preparan antes del análisis.
