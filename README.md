# Proyecto de IA Conversacional por Voz para Soporte IT

## 1. Descripción general

Este proyecto implementa un asistente conversacional técnico orientado a soporte IT. El sistema permite realizar consultas por voz o texto sobre errores, procedimientos y documentación técnica. La solución integra Dialogflow ES para la gestión conversacional, FastAPI como backend, una base vectorial Qdrant para recuperación documental, embeddings generados con Sentence Transformers, un modelo LLM local servido con Ollama y servicios externos de STT/TTS para entrada y salida por voz.

El objetivo principal es demostrar un flujo completo de IA conversacional:

```text
Audio del usuario → STT → Dialogflow ES → Webhook FastAPI → RAG/Qdrant/Ollama → Dialogflow → TTS → Audio de respuesta
```

## 2. Componentes principales

### Dialogflow ES

Dialogflow ES se utiliza para detectar intenciones, extraer entidades y gestionar el flujo conversacional. El agente incluye intents personalizados para saludo, ayuda, consulta de errores técnicos, consulta de procedimientos, consulta documental, corrección de datos, cancelación y despedida.

La entidad personalizada `codigo_error` permite detectar códigos técnicos como:

```text
BKP-302
BKP302
SRV-1001
SRV1001
DR-2024
DR2024
```

Los intents relacionados con consultas técnicas tienen activado el webhook para delegar la respuesta dinámica al backend FastAPI.

### FastAPI

FastAPI actúa como backend principal del sistema. Expone endpoints para:

- Consultas directas al RAG.
- Webhook de Dialogflow ES.
- Entrada de audio con depuración.
- Entrada de audio y salida de respuesta hablada.

El backend también normaliza transcripciones de voz, por ejemplo convirtiendo `BKP302` en `BKP-302`, para mejorar la extracción de códigos técnicos.

### Qdrant

Qdrant se utiliza como base de datos vectorial. Almacena fragmentos de documentación técnica previamente ingerida. Cada fragmento queda asociado a su texto, fuente y metadatos.

### Embeddings

Los embeddings se generan mediante:

```text
sentence-transformers/all-MiniLM-L6-v2
```

Este modelo transforma las consultas y los documentos en vectores comparables para realizar búsqueda semántica.

### Ollama y Qwen

El LLM se ejecuta localmente mediante Ollama. Durante las pruebas se utilizaron modelos Llama, pero finalmente se adoptó:

```text
qwen2.5:0.5b
```

El cambio se realizó para reducir la latencia del webhook y hacer que el flujo con Dialogflow ES fuera más estable.

### STT y TTS

El sistema usa STT externo para transcribir el audio del usuario y TTS externo para generar la respuesta hablada. De esta forma, Dialogflow se utiliza como motor conversacional, pero no como sistema de reconocimiento o síntesis de voz.

## 3. Flujo funcional

### Consulta por texto desde Dialogflow

1. El usuario escribe una consulta en Dialogflow, por ejemplo: `¿Qué significa el error BKP-302?`.
2. Dialogflow detecta el intent `consultar_error_tecnico`.
3. Dialogflow extrae la entidad `codigo_error`.
4. Dialogflow llama al webhook configurado en FastAPI.
5. FastAPI formula una consulta RAG usando el código técnico extraído.
6. Qdrant recupera los fragmentos documentales más relevantes.
7. Ollama genera una respuesta breve usando el contexto recuperado.
8. FastAPI devuelve a Dialogflow un `fulfillmentText`.
9. Dialogflow muestra la respuesta al usuario.

### Consulta completa por voz

1. El usuario envía un audio.
2. FastAPI transcribe el audio mediante STT.
3. Se normaliza la transcripción para corregir códigos técnicos sin guion.
4. FastAPI envía el texto normalizado a Dialogflow mediante API.
5. Dialogflow detecta intent y entidades.
6. Dialogflow llama al webhook de FastAPI.
7. FastAPI ejecuta el flujo RAG.
8. Dialogflow devuelve el resultado a FastAPI.
9. FastAPI genera una respuesta hablada mediante TTS.
10. El usuario recibe un archivo de audio con la respuesta.

## 4. Optimización aplicada

Durante las pruebas se detectó que modelos LLM más grandes, como `llama3.2:3b` y `llama3.2:1b`, respondían correctamente en pruebas locales, pero podían superar el tiempo práctico de espera del webhook de Dialogflow ES.

Para estabilizar el flujo completo se realizaron los siguientes ajustes:

- Cambio del modelo LLM a `qwen2.5:0.5b`.
- Limitación de la recuperación RAG a 2 fragmentos.
- Limitación del contexto enviado al LLM a 700 caracteres por fragmento.
- Uso de la entidad `codigo_error` para formular consultas RAG más precisas.
- Normalización de transcripciones para transformar códigos como `BKP302` en `BKP-302`.
- Configuración de Uvicorn con 2 workers para permitir que FastAPI atienda simultáneamente la petición de voz y la llamada entrante del webhook.

## 5. Endpoints principales

### Webhook de Dialogflow

```text
POST /dialogflow-webhook
```

Recibe el payload de Dialogflow, identifica intent y parámetros, ejecuta RAG si corresponde y devuelve `fulfillmentText`.

### Prueba de voz en modo debug

```text
POST /voice-dialogflow-debug
```

Devuelve JSON con:

- Transcripción original.
- Transcripción normalizada.
- Intent detectado.
- Respuesta generada.
- ID de sesión.

### Prueba de voz completa

```text
POST /voice-dialogflow
```

Recibe un archivo de audio y devuelve un archivo de audio con la respuesta hablada.

## 6. Comandos de prueba

### Probar webhook local

```bash
curl -X POST http://localhost:8000/dialogflow-webhook \
  -H "Content-Type: application/json" \
  -d '{"queryResult":{"queryText":"Qué significa el error BKP-302?","intent":{"displayName":"consultar_error_tecnico"},"parameters":{"codigo_error":"BKP-302"}}}'
```

### Probar voz en modo debug

```bash
curl -X POST http://localhost:8000/voice-dialogflow-debug \
  -F "file=@/ruta/al/audio/pregunta1.m4a"
```

### Probar voz con salida hablada

```bash
curl -X POST http://localhost:8000/voice-dialogflow \
  -F "file=@/ruta/al/audio/pregunta1.m4a" \
  --output respuesta_dialogflow.mp3
```

## 7. Entregables

La entrega debe incluir:

- Código fuente del proyecto.
- Fichero `.env.example` sin credenciales reales.
- Exportación ZIP del agente Dialogflow ES.
- Documentos de prueba usados en el RAG.
- Informe PDF.
- Vídeo demo de máximo 5 minutos.
- Documentación de instalación, ejecución y pruebas.

## 8. Consulta recomendada para la demo

Para la demostración se recomienda usar una consulta validada contra la base documental:

```text
¿Qué significa el error BKP-302?
```

Esta consulta permite mostrar extracción de entidad, llamada al webhook, búsqueda RAG, generación con LLM y respuesta por voz.
