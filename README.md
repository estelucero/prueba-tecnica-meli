# Sistema multiagente de investigación de contracargos

[Arquitectura](README.md) · [Diagrama](DIAGRAMA_SOLUCION.md) · [RAG](RAG.md) · [Consultas SQL](QUERIES_SQL.md)

> En este documento cuento cómo está pensado el sistema y cómo ponerlo en marcha. En los documentos vinculados dejo más detalle sobre cada parte.

## 1. Objetivo

La idea es poder investigar una transacción a partir de su `transaction_id`: reunir la información de la operación, revisar las políticas internas que podrían aplicar y tener en cuenta los antecedentes disponibles. Con esa información, los agentes evalúan el caso y el sistema genera un informe HTML que explica **qué se investigó, qué políticas fueron importantes, qué decisión recomienda la IA y por qué**. También incluye las críticas y mejoras que se pueden aplicar al proceso.

El informe tiene un estado principal, `report_status`, que puede ser `ACCEPT`, `REJECT` o `PENDING`. Ese valor representa **la decisión recomendada por la IA**, no necesariamente el estado que ya figura en los sistemas. Por ejemplo, si el análisis recomienda rechazar un contracargo, el informe lo muestra como **rechazo recomendado**, pero eso no significa que el rechazo se haya ejecutado.

El sistema sirve como herramienta de apoyo para tomar decisiones: **no ejecuta** reembolsos, bloqueos ni cambios de estado.

## 2. Arquitectura

Separé el trabajo en dos partes: por un lado, la búsqueda de **datos concretos**; por otro, la interpretación de las políticas y demás información escrita en lenguaje natural. Después, cada agente tiene una responsabilidad específica para evitar que todos vuelvan a analizar o decidir lo mismo.

| Componente            | Función                                                                                                                                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **n8n**               | Recibe el webhook, valida la solicitud, consulta las bases, prepara el contexto, coordina los agentes y devuelve el informe HTML.                                                                              |
| **PostgreSQL**        | Guarda los datos estructurados de transacciones, eventos, casos y métricas. Las consultas se hacen **antes** de llamar a los agentes.                                                                          |
| **Qdrant**            | Permite recuperar políticas internas y casos históricos por similitud de significado (RAG).                                                                                                                    |
| **Embeddings**        | Representan consultas y documentos como vectores para poder compararlos. El export anterior usa `text-embedding-3-small`.                                                                                      |
| **Siete agentes LLM** | Tres agentes investigan desde distintas perspectivas; el Analizador evalúa las políticas y recomienda una decisión; el Crítico propone mejoras; Summarize prepara el JSON final; y el Redactor genera el HTML. |

**Una decisión importante:** ningún agente consulta SQL directamente. n8n obtiene los registros y calcula las métricas mediante sus nodos de PostgreSQL antes de llamar a los agentes. Los agentes especializados pueden trabajar con las políticas y los antecedentes recuperados desde Qdrant; los agentes posteriores reciben esos resultados y no necesitan repetir la búsqueda.

## 3. Puesta en marcha

### Levantar los servicios

Desde la carpeta principal del proyecto, levantá los servicios con Docker Compose:

```bash
docker compose up -d
```

El comando inicia los servicios definidos en `compose.yml`, entre ellos n8n, PostgreSQL y Qdrant. Después podés entrar a n8n desde [http://localhost:5678](http://localhost:5678).

### Importar y ejecutar los workflows

Importá en n8n los archivos JSON de `workflows/` y ejecutalos en este orden:

1. **`meli-ingesta-postgresql.json`** (`meli-ingesta-postgresql`): carga los datos estructurados en PostgreSQL.
2. **`meli-database-vector.json`** (`meli-database-vector`): prepara la base vectorial en Qdrant.
3. **`meli-workflow.json`** (`meli-workflow`): ejecuta el flujo completo de investigación.

Primero corré los dos workflows que cargan los datos. Recién después ejecutá el flujo completo, que necesita información tanto de PostgreSQL como de Qdrant.

## 4. Qué hace cada agente

La idea de dividir el trabajo es que cada agente resuelva una parte concreta de la investigación. Así evitamos repetir evaluaciones, generar conclusiones diferentes sobre el mismo punto o trasladar decisiones al agente que solamente redacta el informe.

1. **Comercial:** revisa las políticas, condiciones, obligaciones, restricciones y beneficios vinculados al comercio. Para eso utiliza su historial y las métricas que ya preparó n8n. Su trabajo es investigar desde el lado comercial y dejar documentadas las políticas evaluadas, con su evidencia.
2. **Operativo:** analiza los controles antifraude, el procesamiento de la operación, las autorizaciones, los eventos, los plazos y la documentación disponible. Identifica qué condiciones operativas se cumplen y cuáles necesitan verificación, sin dar por hecho que la falta de registros demuestra un incumplimiento.
3. **Cliente:** revisa los derechos, beneficios, requisitos, plazos y restricciones que afectan al titular y al contracargo. Su objetivo es aportar la perspectiva del cliente y las políticas que podrían determinar la elegibilidad o el tratamiento del reclamo.
4. **Analizador:** recibe las evaluaciones de los tres especialistas y revisa **todas las políticas identificadas**. Determina cuáles se cumplen, cuáles se incumplen, cuáles no aplican y cuáles no se pueden verificar. Lo más importante es que **interpreta cómo se relacionan las políticas entre sí**: una exclusión de elegibilidad no tiene el mismo efecto que un beneficio o un control operativo. A partir de ese razonamiento establece una decisión recomendada: `ACCEPT`, `REJECT` o `PENDING`, explicando las políticas y la evidencia que la justifican.
5. **Crítico y mejoras:** recibe el resultado del Analizador y la información de la investigación. Revisa si la recomendación está bien fundamentada, detecta posibles errores de interpretación, fallas operativas y riesgos del proceso. No vuelve a decidir el contracargo: propone mejoras concretas, explica por qué servirían y aporta la evidencia y las políticas relacionadas. Puede señalar que una recomendación es cuestionable sin reemplazarla automáticamente por otra.
6. **Summarize:** toma la transacción original, la decisión del Analizador y las observaciones del Crítico. No vuelve a investigar ni a evaluar políticas. Su función es **armar un único JSON final** con la información del caso, el resumen, los hallazgos relevantes, los antecedentes comparables que ya estén disponibles, la resolución recomendada, las mejoras y los próximos pasos. El campo principal es `report_status`, cuyo valor debe coincidir con la decisión del Analizador.
7. **Redactor HTML:** recibe el JSON de Summarize y lo convierte en un informe HTML5 ordenado y fácil de leer. No cambia la decisión ni realiza análisis nuevos. Utiliza `report_status` para destacar si la IA recomienda aceptar, rechazar o mantener pendiente el contracargo, y presenta las demás secciones de manera clara.

### Cómo se pasa la información entre ellos

Los tres especialistas entregan sus evaluaciones, incluyendo políticas que aplican, que no aplican o cuya situación no se puede verificar. El Analizador consolida esas políticas en `policy_evaluations`, con un estado como `COMPLIANT`, `NON_COMPLIANT`, `NOT_APPLICABLE` o `UNDETERMINED`, y genera `chargeback_decision` con su razonamiento y las políticas determinantes.

Esa decisión pasa al Crítico, que devuelve una evaluación de la recomendación y una lista breve de `improvements`. Finalmente, Summarize reúne lo necesario en el JSON que recibe el Redactor HTML. **La decisión no se vuelve a calcular en las últimas etapas:** el estado destacado del informe sale de `analysis.chargeback_decision.decision` y se copia a `report_status`.

El JSON final se organiza en estas partes:

| Campo              | Qué contiene                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| `report_status`    | La decisión recomendada por la IA: `ACCEPT`, `REJECT` o `PENDING`. Es el estado principal del informe.      |
| `case_information` | Datos de la transacción, cliente, comercio y estado originalmente registrado.                               |
| `case_summary`     | Una explicación breve de qué pasó y qué motivó la investigación, cuando esa información está documentada.   |
| `investigation`    | Qué se analizó y cuáles fueron los hallazgos más importantes.                                               |
| `similar_cases`    | Casos históricos comparables que ya estén identificados en la información recibida. Si no hay, queda vacío. |
| `resolution`       | Por qué se recomienda esa decisión, qué políticas son determinantes y qué evidencia la respalda.            |
| `improvements`     | Críticas y mejoras propuestas para corregir o prevenir problemas del proceso.                               |
| `next_actions`     | Pasos sugeridos para revisar, autorizar o dar seguimiento a la recomendación y a las mejoras.               |

## Documentación complementaria

- **[Diagrama de la solución](DIAGRAMA_SOLUCION.md):** cómo avanza una investigación y qué pasa en cada etapa.
- **[RAG](RAG.md):** cómo se buscan políticas y casos parecidos, y qué límites tiene esa búsqueda.
- **[Consultas SQL y métricas](QUERIES_SQL.md):** qué datos se consultan, cómo se calculan algunas métricas y qué supuestos hay que validar.

## 5. Integridad y límites de la investigación

- **La decisión de la IA no es un cambio ejecutado en el sistema.** `report_status: "REJECT"` significa que el análisis recomienda el rechazo; no demuestra que el contracargo ya haya sido rechazado en la base de datos.
- **El estado de la transacción no determina la recomendación.** Una transacción con estado `Contracargo iniciado` puede terminar con una recomendación `REJECT`, `ACCEPT` o `PENDING`, dependiendo de las políticas y la evidencia.
- **Las políticas no se deciden por cantidad.** El Analizador tiene que comprender su función y su relación lógica: una condición que determina la elegibilidad puede ser decisiva, mientras un beneficio, un SLA o un control operativo puede generar obligaciones independientes sin modificar el resultado del contracargo.
- **Que una condición se active no significa que se ejecutó una acción.** Por ejemplo, superar un umbral puede exigir una revisión, pero sin registros suficientes no corresponde afirmar que la revisión se realizó o que se omitió.
- **Que no haya logs no prueba que no haya habido una acción.** Cuando falta evidencia para evaluar una política, puede quedar `UNDETERMINED`; eso no obliga a poner la decisión completa en `PENDING` si otra política ya determina el resultado.
- **Un caso parecido no es evidencia del caso actual.** Los antecedentes sirven para comparar y entender posibles patrones, no para copiar una resolución. Summarize solo los incluye si ya llegan identificados y con información suficiente.
- **Si hay una crítica importante, se conserva su observación.** El Crítico puede cuestionar el razonamiento del Analizador. Summarize debe reflejar esa reserva, pero no cambiar silenciosamente `report_status`.
- **Todos los montos del conjunto de datos están expresados en USD.**
- **La búsqueda RAG no es exhaustiva.** `topK=5` devuelve hasta cinco resultados por consulta; para una auditoría de cumplimiento convendría sumar una búsqueda determinística de todo el catálogo.

## 6. Alcance y deuda técnica

A lo largo del desarrollo pude implementar el flujo principal de investigación de contracargos, desde la ingesta de información hasta la generación del informe final. La idea fue integrar distintas herramientas para que el sistema no dependa únicamente de un modelo de lenguaje, sino que pueda trabajar con datos estructurados, consultar documentación relevante y justificar las decisiones que toma.

En cuanto a lo que llegué a implementar, el sistema cuenta con los siguientes componentes:

**Ingesta de datos**: desarrollé workflows en n8n para cargar información estructurada en PostgreSQL y preparar las políticas internas y los casos históricos que después se utilizan en el RAG. También incorporé un webhook para iniciar una investigación a partir del identificador de una transacción.

**RAG**: implementé la búsqueda de políticas y antecedentes mediante Qdrant y embeddings. Esto permite recuperar documentación relevante para cada investigación sin tener que incorporar todas las políticas directamente en los prompts.

**Sistema multiagente**: desarrollé agentes especializados para analizar la información desde las perspectivas operativa, comercial y del cliente. A partir de sus resultados, el Analizador evalúa las políticas, determina cuáles se cumplen y cuáles no, e identifica las condiciones que realmente influyen en la decisión del contracargo. Después, el Crítico revisa ese razonamiento y propone mejoras, mientras que Summarize consolida la información para que el Redactor genere el informe HTML.

**Automatización**: utilicé n8n para coordinar todo el proceso, desde las consultas SQL y la preparación de métricas hasta la ejecución de los agentes y la generación del resultado. Una decisión importante fue realizar las consultas a PostgreSQL antes de ejecutar los agentes, de manera que trabajen con información previamente preparada y no tengan que consultar la base de datos por su cuenta.

**Observabilidad y trazabilidad**: implementé el almacenamiento de las salidas de los agentes en PostgreSQL, lo que permite conservar los resultados de cada etapa de la investigación. También incorporé tablas para guardar y versionar los prompts, junto con una separación entre sus versiones y las utilizadas en producción. De esta manera, puedo conocer qué información generó cada agente y mantener un registro de las instrucciones utilizadas, facilitando el seguimiento y la revisión de las investigaciones.

**Identificación de problemas y propuestas de mejora**: mediante el Agente Crítico, el sistema puede identificar inconsistencias, posibles incumplimientos y riesgos operativos a partir de los resultados del análisis. Además de señalar esos problemas, propone medidas correctivas o preventivas y explica por qué podrían mejorar el proceso.

### ¿Qué quedó pendiente?

La principal funcionalidad que no llegué a implementar fue el circuito de mejora continua.

Si bien el sistema ya guarda los prompts y las salidas de los agentes, todavía no desarrollé un proceso que utilice automáticamente esa información para evaluar la calidad de las respuestas y mejorar el comportamiento de los agentes.

La idea era aprovechar los resultados almacenados para identificar errores recurrentes, detectar decisiones mal fundamentadas, comparar diferentes versiones de prompts y utilizar las correcciones para mejorar las siguientes investigaciones.

También quedó pendiente implementar un mecanismo para actualizar el RAG a partir de los resultados y del feedback obtenido. Por ejemplo, si durante una investigación se detecta una interpretación incorrecta de una política o se incorpora un nuevo antecedente validado, el sistema podría utilizar esa información para enriquecer la base de conocimiento y evitar repetir errores similares.

Considero que haber incorporado la observabilidad y el versionado de prompts fue un primer paso importante, porque ahora la información necesaria para analizar el comportamiento del sistema queda registrada. Sin embargo, **tener esa información almacenada no significa que el sistema ya aprenda de sus errores**. Para eso todavía sería necesario desarrollar un workflow que evalúe los resultados, incorpore feedback, valide los cambios y actualice los prompts o la base de conocimiento cuando corresponda.

En definitiva, pude completar el proceso de investigación, análisis y generación de informes, junto con el registro de sus resultados. Lo que quedó como una siguiente etapa fue cerrar el ciclo de mejora continua, para que el sistema pueda mejorar progresivamente a partir de las investigaciones realizadas.

---

**En pocas palabras:** PostgreSQL aporta los datos y las métricas; Qdrant ayuda a encontrar políticas y antecedentes relevantes. Tres agentes especializados investigan el caso, el Analizador interpreta las políticas y recomienda una decisión, el Crítico revisa esa conclusión y propone mejoras, Summarize prepara el JSON final con `report_status` como estado principal, y el Redactor genera el informe HTML sin volver a decidir nada.

**Fuente:** documentación del proyecto proporcionada por el usuario y ajustes de responsabilidades definidos para los agentes el 20/11/2026. La implementación efectiva de los cambios debe verificarse contra el workflow actualizado.
