# Justificación técnica del proyecto

## 1. Objetivo técnico

El proyecto implementa un asistente conversacional por voz orientado a soporte IT. La solución integra reconocimiento de voz, comprensión conversacional, recuperación documental, generación de respuesta y síntesis de voz.

El sistema no se limita a respuestas estáticas, sino que utiliza una arquitectura RAG para consultar documentación técnica y generar respuestas contextualizadas.

## 2. Arquitectura seleccionada

La arquitectura final combina los siguientes componentes:

```text
Usuario
→ Audio
→ STT externo
→ FastAPI
→ Dialogflow ES
→ Webhook FastAPI
→ Embeddings
→ Qdrant
→ Ollama/Qwen
→ Dialogflow
→ FastAPI
→ TTS externo
→ Audio de respuesta
```

Esta separación permite mantener Dialogflow ES como motor de gestión conversacional, mientras que FastAPI asume la lógica dinámica, documental y de integración.

## 3. Uso de Dialogflow ES

Dialogflow ES se emplea para:

- Detectar la intención del usuario.
- Extraer entidades relevantes.
- Gestionar contextos conversacionales.
- Activar webhooks para respuestas dinámicas.

La entidad personalizada `codigo_error` permite detectar códigos técnicos con y sin guion, como `BKP-302` o `BKP302`. Esto es especialmente importante en el flujo de voz, ya que el STT puede omitir signos como el guion.

## 4. Uso de FastAPI

FastAPI centraliza la lógica de backend. Se utiliza para:

- Recibir llamadas de Dialogflow mediante webhook.
- Enviar texto a Dialogflow en el flujo de voz.
- Ejecutar la recuperación documental.
- Generar respuestas usando un LLM local.
- Orquestar la entrada y salida de audio.

Durante las pruebas se detectó que el flujo de voz genera una llamada reentrante: FastAPI llama a Dialogflow y Dialogflow vuelve a llamar al webhook de FastAPI. Con un único worker, este flujo podía bloquearse. Por ello se configuró Uvicorn con 2 workers, permitiendo atender simultáneamente la petición inicial y la llamada entrante del webhook.

## 5. Uso de RAG

Se implementa RAG para responder a consultas técnicas a partir de documentación propia. El flujo RAG es:

1. Se recibe la pregunta.
2. Se genera un embedding de la consulta.
3. Se consulta Qdrant para recuperar fragmentos relevantes.
4. Se construye un contexto reducido.
5. El LLM genera la respuesta usando solo ese contexto.

Este enfoque permite responder sobre documentación técnica sin tener que codificar todas las respuestas de forma manual.

## 6. Modelo de embeddings

Se utiliza:

```text
sentence-transformers/all-MiniLM-L6-v2
```

Este modelo ofrece un equilibrio adecuado entre velocidad, tamaño y calidad para búsquedas semánticas en documentación técnica breve.

Para evitar descargas repetidas del modelo de embeddings, se recomienda persistir la caché de Hugging Face mediante un volumen Docker.

## 7. Base vectorial Qdrant

Qdrant se utiliza como base de datos vectorial por su integración sencilla en contenedores y su capacidad para realizar búsquedas por similitud. Cada documento se fragmenta y se almacena con metadatos como fuente y página.

La colección usada es:

```text
it_knowledge
```

## 8. Elección del LLM

Inicialmente se probaron modelos Llama mediante Ollama:

```text
llama3.2:3b
llama3.2:1b
```

Ambos respondían correctamente en pruebas locales, pero presentaban una latencia demasiado alta para el webhook de Dialogflow ES. Dialogflow no espera indefinidamente a que el webhook responda, por lo que respuestas de 7, 8 u 11 segundos podían provocar errores intermitentes o respuestas `Not available`.

Por este motivo se adoptó:

```text
qwen2.5:0.5b
```

Este modelo permite mantener el flujo completo con LLM local, reduciendo la latencia y mejorando la estabilidad durante la demo.

## 9. Optimización del contexto RAG

Para reducir la latencia y estabilizar la generación, se aplicaron estos ajustes:

- Recuperación limitada a 2 chunks.
- Contexto limitado a 700 caracteres por chunk.
- Generación limitada en número de tokens.
- Temperatura configurada en 0 para respuestas más deterministas.
- Uso de `top_k`, `top_p` y `seed` para reducir variabilidad.

El objetivo de esta configuración es evitar que el LLM reciba demasiado texto y tarde más de lo aceptable para Dialogflow ES.

## 10. Uso de búsqueda por código técnico

En el flujo con Dialogflow, cuando se extrae la entidad `codigo_error`, el backend usa ese valor para formular una consulta RAG específica.

Ejemplo:

```text
Usuario: ¿Qué significa el error BKP-302?
Entidad extraída: BKP-302
Consulta RAG: Explica el código técnico BKP-302, su causa y los pasos de resolución.
```

Esto mejora la recuperación documental porque evita depender de la frase completa del usuario y centra la búsqueda en el identificador técnico.

## 11. Normalización de transcripciones

Durante las pruebas con audio se observó que el STT podía transcribir:

```text
BKP302
```

cuando el documento y la entidad estaban definidos como:

```text
BKP-302
```

Para resolverlo, se añadió una normalización previa al envío a Dialogflow. Esta normalización transforma códigos técnicos sin guion en su forma canónica con guion.

Ejemplo:

```text
que significa el error BKP302
→ que significa el error BKP-302
```

Esta mejora permite que el flujo de voz sea más robusto.

## 12. Uso de STT y TTS externos

El sistema no utiliza el reconocimiento ni la síntesis de voz de Dialogflow. La voz de entrada se procesa mediante STT externo y la respuesta final se convierte a audio mediante TTS externo.

Esto permite cumplir el requisito de usar servicios de voz externos a Dialogflow y mantener Dialogflow centrado en la comprensión conversacional.

## 13. Problemas encontrados y soluciones

### Latencia del LLM

Problema: los modelos Llama respondían correctamente pero tardaban demasiado para Dialogflow ES.

Solución: cambio a `qwen2.5:0.5b`, reducción del contexto y limitación de generación.

### Variabilidad en respuestas

Problema: el LLM podía dar respuestas distintas ante la misma pregunta.

Solución: temperatura 0, `top_k` bajo, `top_p` bajo, `seed` fijo y prompt más cerrado.

### Transcripción de códigos sin guion

Problema: el STT transcribía `BKP-302` como `BKP302`.

Solución: normalización de códigos técnicos antes de enviar el texto a Dialogflow.

### Bloqueo en flujo de voz

Problema: FastAPI llamaba a Dialogflow y Dialogflow llamaba de vuelta al webhook del mismo FastAPI.

Solución: ejecutar Uvicorn con 2 workers para atender ambas peticiones simultáneamente.

## 14. Conclusión

La solución final mantiene el flujo completo solicitado en la práctica: entrada por voz, detección conversacional con Dialogflow ES, webhook en FastAPI, recuperación documental con Qdrant, generación con LLM local vía Ollama y respuesta final por voz.

Las optimizaciones aplicadas no eliminan componentes del flujo, sino que lo hacen viable dentro de las restricciones prácticas de latencia de Dialogflow ES y del entorno local de ejecución.
