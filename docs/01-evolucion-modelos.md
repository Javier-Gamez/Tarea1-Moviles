# 1. Evolución de los modelos

## 1.1 ¿Qué es un modelo de lenguaje (LM)?

Un **modelo de lenguaje** (*Language Model*, LM) es un modelo estadístico que aprende la
distribución de probabilidad de secuencias de texto: dado un fragmento de texto,
predice qué token (palabra o sub-palabra) es más probable que venga a continuación. Los
modelos de lenguaje clásicos (n-gramas, modelos ocultos de Markov, y luego redes
neuronales recurrentes como LSTM) ya hacían esto, pero con capacidad limitada: contexto
corto, vocabulario reducido y poca capacidad de generalizar a tareas para las que no
fueron entrenados explícitamente.

## 1.2 De LM a LLM

El salto a **modelo de lenguaje grande** (*Large Language Model*, LLM) no es solo una
cuestión de tamaño, aunque el tamaño es parte central del fenómeno. Tres factores
convergieron:

1. **Arquitectura Transformer** (Vaswani et al., 2017): el mecanismo de atención permite
   que el modelo relacione cualquier token de una secuencia con cualquier otro,
   independientemente de la distancia entre ellos, y hacerlo en paralelo (a diferencia
   de las RNN, que procesan token por token de forma secuencial). Esto hizo viable
   entrenar modelos mucho más grandes en tiempos razonables.
2. **Escala**: entrenar con miles de millones de parámetros sobre corpus de texto del
   orden de billones de tokens produjo un salto cualitativo, no solo cuantitativo. A
   partir de cierta escala aparecen capacidades emergentes: seguir instrucciones,
   resolver problemas de razonamiento simple, traducir, programar, sin haber sido
   entrenados específicamente para cada una de esas tareas por separado.
3. **Ajuste posterior al preentrenamiento** (*fine-tuning*, RLHF — aprendizaje por
   refuerzo con retroalimentación humana, y más recientemente RLAIF/DPO): el
   preentrenamiento produce un modelo que predice texto plausible, pero no
   necesariamente útil, seguro o alineado con lo que un usuario pide. El ajuste
   posterior es lo que convierte un "modelo que completa texto" en un asistente que
   responde instrucciones.

En resumen: un LLM es un LM llevado a una escala (parámetros + datos + cómputo) en la que
emergen capacidades generales, entrenado con arquitectura Transformer y refinado con
técnicas de alineación para ser útil como asistente conversacional o agente.

## 1.3 Modelos con razonamiento explícito

Un modelo con **razonamiento explícito** (a veces llamado *reasoning model*) es aquel
que, antes de dar la respuesta final, genera una cadena de pasos intermedios de
razonamiento (*chain of thought*) — ya sea visible o interna — que usa como andamiaje
para llegar a una respuesta más confiable en tareas que requieren varios pasos lógicos,
matemáticos o de planeación (por ejemplo, resolver un problema matemático, depurar
código o planear una secuencia de llamadas a herramientas).

**Punto importante y frecuentemente malentendido**: esta capacidad **no aparece
espontáneamente solo por aumentar el número de parámetros**. Un modelo grande sin ningún
tratamiento adicional no necesariamente "razona mejor" que uno mediano; de hecho, en
igualdad de entrenamiento, un modelo más grande puede seguir fallando en tareas de
varios pasos si nunca fue expuesto ni optimizado para producir ese tipo de traza. La
capacidad de razonamiento explícito proviene de dos fuentes concretas, no del tamaño en
sí:

- **Técnicas de entrenamiento específicas**: afinar el modelo con ejemplos de cadenas de
  razonamiento correctas, o entrenarlo con aprendizaje por refuerzo donde la recompensa
  premia llegar a la respuesta correcta a través de una traza de razonamiento coherente
  (y no solo la respuesta final). Esto enseña al modelo *a producir* pasos intermedios
  útiles, no solo a memorizar patrones.
- **Cómputo adicional en tiempo de inferencia**: permitir que el modelo "piense más"
  antes de responder — generando más tokens de razonamiento interno, explorando varias
  rutas de solución y descartando las que no funcionan, o iterando sobre su propia
  respuesta — mejora el resultado en tareas complejas incluso sin cambiar los parámetros
  del modelo. Es la diferencia entre pedirle a alguien que responda de inmediato o que
  se tome un tiempo para pensarlo en un borrador antes de contestar.

En otras palabras: el razonamiento explícito es el resultado de *cómo* se entrena y
*cuánto* cómputo se le permite usar al modelo al momento de responder, no una
consecuencia automática de tener más parámetros.

## Referencias de esta sección

Ver lista completa de referencias en formato APA en el [`README.md`](../README.md) del
repositorio.
