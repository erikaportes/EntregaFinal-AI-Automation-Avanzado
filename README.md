# Asistente Inteligente Ni@ · Sistema Agéntico Empresarial

Sistema multiagente construido en **n8n** para **Ni@ Digital Brand**, agencia de publicidad, marketing digital, branding, diseño web y automatización. El asistente atiende por WhatsApp y Telegram (texto o notas de voz), asigna cada solicitud a un especialista de IA y **valida cada respuesta con un supervisor AI-as-a-Judge** antes de entregarla, corregirla o escalarla a revisión humana.

> Proyecto Final Integrador · AI Automation Avanzado (AI Experto) · Cohorte 2026 Autora: **Erika Portes**

---

## Qué resuelve

El equipo de Ni@ resolvía a mano cientos de solicitudes al mes (naming, copys, estrategias, seguimiento de proyectos, incidencias) y los clientes preguntaban una y otra vez por las mismas políticas de contratación. El asistente:

- Atiende al **equipo interno** con seis especialistas: Estrategia, Branding, Contenido, Proyectos, Automatización y Problemas (tickets).  
- Atiende a **clientes externos** con un asistente de preguntas frecuentes (FAQ) que responde **solo** con el Manual de Políticas indexado, cita la fuente y responde "No sé" cuando el manual no cubre la duda.  
- Revisa la calidad de **cada** respuesta antes de que llegue a una persona.  
- Registra cada interacción (pregunta, respuesta, crítica, nota y veredicto) para auditoría.

| Indicador | Resultado estimado |
| :---- | :---- |
| Horas liberadas al mes | ≈ 358 h (de 417 h a 58 h) |
| Costo mensual | US\$1,435 con el asistente vs. US\$5,000 manual |
| Ahorro neto | ≈ US\$3,565 al mes (−71 %) |
| ROI primer año | ≈ 206 % |
| Costo de IA por solicitud | US\$0.00676 (US\$6.76 por 1,000 solicitudes) |

---

## Arquitectura

flowchart TD

    A\[WhatsApp · Whapi webhook\] \--\> N\[Unificar mensaje\]

    B\[Telegram trigger\] \--\> N

    N \--\>|nota de voz| W\[Whisper · Groq\<br/\>audio a texto\]

    W \--\> G

    N \--\>|texto| G\[Guardrails\<br/\>datos privados, jailbreak, URL, tópico\]

    G \--\>|bloqueado| R0\[Respuesta fija por canal\]

    G \--\>|permitido| M\[(Airtable\<br/\>memoria de largo plazo)\]

    M \--\> O\[AI Agent Orquestador\<br/\>clasifica y asigna\]

    O \--\> S{Switch Workers}

    S \--\> E\[Estrategia\]

    S \--\> BR\[Branding\]

    S \--\> C\[Contenido\]

    S \--\> P\[Proyectos\]

    S \--\> AU\[Automatización\]

    S \--\> PR\[Problema · tickets Airtable\]

    S \--\> F\[FAQ · RAG Pinecone\]

    E & BR & C & P & AU & PR & F \--\> J\[AI Agent Juez\<br/\>exactitud\_factual 1–5\]

    J \--\>|4–5 ACEPTADO| OUT\[Respuesta texto o audio\<br/\>ElevenLabs\]

    J \--\>|3 CORREGIR| CO\[AI Agent Corrector\] \--\> OUT

    J \--\>|1–2 RECHAZADO| H\[Gmail · Send and wait\<br/\>aprobación humana\]

    H \--\> OUT

    OUT \--\> L\[(Google Sheets \+ Airtable\<br/\>registro auditable)\]

Cada especialista tiene su propio bloque completo dentro del workflow: agente, memoria de conversación, base de conocimiento, juez, corrector, ramas de error, registro y respuesta por canal.

---

## Componentes

| Capa | Qué hace | Nodos principales |
| :---- | :---- | :---- |
| Entrada | Recibe mensajes de WhatsApp (webhook POST /entrada-mensaje, responde de inmediato) y Telegram | Webhook, Telegram Trigger, Code "Mensaje unificado" |
| Voz | Convierte notas de voz a texto y respuestas a audio | HTTP Request (Whisper vía Groq), Code (oga → ogg), ElevenLabs |
| Seguridad | Bloquea datos privados (contraseñas, tarjetas, INE, CURP), jailbreak, URLs y temas ajenos | Guardrails (usuario y empresa), Switch por canal |
| Coordinación | Clasifica la solicitud con la memoria del cliente y la envía al especialista | AI Agent Orquestador, Structured Output Parser, Airtable |
| Especialistas | Responden con su base de conocimiento estática y dinámica | 7 AI Agents, Google Docs, Simple Memory |
| FAQ (RAG) | Responde solo con el Manual de Políticas y cita la fuente | Pinecone (retrieve-as-tool), Cohere embeddings |
| Supervisión | Califica, corrige o escala cada respuesta | AI Agent Juez, AI Agent Corrector, Gmail Send and Wait |
| Registro | Guarda cada interacción para auditoría y tablero | Google Sheets, Airtable |

---

## Control de calidad (AI-as-a-Judge)

Cada respuesta pasa por un agente juez que devuelve un JSON plano con tres campos:

{

  "type": "object",

  "properties": {

    "critica": { "type": "string", "minLength": 1, "description": "Crítica breve y accionable, máximo 2 oraciones" },

    "exactitud\_factual": { "type": "integer", "minimum": 1, "maximum": 5, "description": "Nota entera de 1 a 5" },

    "estado\_veredicto": { "type": "string", "enum": \["ACEPTADO", "CORREGIR", "RECHAZADO"\] }

  },

  "required": \["critica", "exactitud\_factual", "estado\_veredicto"\],

  "additionalProperties": false

}

| Nota | Veredicto | Ruta |
| :---- | :---- | :---- |
| 4 o 5 | ACEPTADO | Se entrega al usuario y se registra |
| 3 | CORREGIR | El agente corrector la reescribe con la crítica del juez |
| 1 o 2 | RECHAZADO | Se congela y un supervisor la aprueba o edita por Gmail (Human-in-the-loop) |

El juez usa dos rúbricas: una para el equipo interno (entregar exactamente lo pedido, sin hechos inventados) y otra para clientes externos (sin promesas de descuentos, excepciones, plazos personalizados ni información interna).

---

## Asistente FAQ con RAG

| Paso | Configuración |
| :---- | :---- |
| Ingesta | Formulario n8n → LlamaParse (PDF a Markdown) → chunking 1,000 / 150 → Cohere embeddings |
| Almacenamiento | Pinecone, índice nia, namespace manual (se limpia en cada recarga para no mezclar versiones) |
| Consulta | Pinecone como herramienta del agente, hasta 4 fragmentos por búsqueda |
| Sin cobertura | Respuesta fija "No sé" y aviso al equipo por Gmail |
| Cita | Cada respuesta termina con Fuente: Manual\_Politicas\_NiaDigitalBrand\_2026 |

---

## Manejo de errores

- Los nodos de IA y de servicios externos reintentan hasta **5 veces con 5 segundos** de espera.  
- Si un agente o el juez fallan, la rama de error envía una alerta por Gmail al supervisor y el usuario recibe un mensaje de respaldo.  
- Ninguna solicitud queda sin respuesta.

---

## Stack

| Servicio | Uso | Plan |
| :---- | :---- | :---- |
| n8n | Orquestación de todo el sistema | Cloud de prueba / comunitario |
| Google Gemini (gemini-3.5-flash y variantes flash) | Orquestador, especialistas, juez y corrector | Pago por uso, modelos económicos |
| Groq · Whisper | Transcripción de voz | Freemium |
| ElevenLabs | Respuestas en audio | Freemium |
| LlamaParse | Conversión del manual PDF | Freemium |
| Cohere | Embeddings | Freemium |
| Pinecone | Base vectorial | Freemium |
| Airtable | Memoria de largo plazo, tickets y registro | Freemium |
| Google Sheets / Docs | Registro auditable y bases de conocimiento | Gratuito |
| Gmail | Alertas y aprobación humana | Gratuito |
| Whapi (WhatsApp) / Telegram | Canales de mensajería | Freemium / gratuito |

---

## Instalación

1. **Importa el workflow.** En n8n: *Workflows → Import from file* y selecciona Entrega\_Final\_-\_Erika\_Portes\_-\_AI\_Automation\_Avanz.json.  
     
2. **Crea las credenciales** en n8n (el JSON solo guarda referencias, no claves):  
   

| Credencial | Tipo en n8n |
| :---- | :---- |
| Google Gemini | Google Gemini (PaLM) API |
| Gmail, Google Docs, Google Sheets | OAuth2 de Google |
| Airtable | Airtable OAuth2 |
| Telegram | Telegram API (token del bot) |
| Whapi (WhatsApp) y Groq | HTTP Bearer Auth |
| ElevenLabs | ElevenLabs API |
| Cohere | Cohere API |
| Pinecone | Pinecone API |
| LlamaParse | LlamaParse API |

   

3. **Reemplaza los recursos propios** por los tuyos:  
     
   - Base y tablas de Airtable (memoria, tickets y registro).  
   - Hoja de Google Sheets del registro de respuestas.  
   - Documentos de Google Docs con las bases de conocimiento de cada especialista.  
   - Índice de Pinecone (nia) y namespace (manual).  
   - Voz de ElevenLabs.  
   - Correo del supervisor en los nodos de Gmail.

   

4. **Carga el Manual de Políticas** con el formulario "Manual políticas" para indexarlo en Pinecone.  
     
5. **Configura los webhooks:** apunta Whapi a la URL de producción de /entrada-mensaje y conecta el bot de Telegram.  
     
6. **Activa el workflow** y prueba con un mensaje de texto y una nota de voz.

---

## Estructura del registro (Google Sheets)

| Campo | Contenido |
| :---- | :---- |
| Correo, Nombre, Chat Id, Tel | Identificación de quien pregunta y del canal |
| Agente | Especialista que respondió |
| Pregunta | Solicitud original |
| Respuesta x corregir | Primera respuesta del especialista |
| Respuesta | Respuesta final enviada |
| Estado veredicto | ACEPTADO, CORREGIR o RECHAZADO |
| Crítica | Observación del juez |
| Exactitud factual | Nota del juez (1–5) |
| Fecha | Fecha y hora |
| ¿Audio? | Si llegó como nota de voz |
| Error | Detalle técnico si falló algún servicio |
| Tiempo | Duración del procesamiento |

---

## Resultados de la prueba A/B

Se compararon dos modelos con las mismas 5 preguntas (10 corridas):

| Métrica | Modelo A (gemini-3.5-flash) | Modelo B (gemini-3.8-flash) |
| :---- | :---- | :---- |
| Precisión media (1–5) | 3.40 | 2.60 |
| Aceptadas al primer intento | 40 % | 0 % |
| Escaladas a humano | 0 | 2 |
| Datos inventados | 0 | 1 |
| Costo por 1,000 ejecuciones | US\$6.76 | US\$12.69 |

Se opera con el modelo A: más preciso y a casi la mitad del costo.

---

## Seguridad y gobernanza

- El workflow exportado **no contiene claves**: las credenciales viven cifradas en n8n.  
- Los guardrails filtran datos personales sensibles antes de que lleguen a la IA.  
- Solo la responsable aprueba nuevas versiones del Manual de Políticas; cada recarga reemplaza el namespace completo.  
- Cada ejecución queda en el historial de n8n y en el registro de respuestas.

---

## Próximas mejoras

- Agregar el canal Slack como medio de comunicación  
- Integrar Airtable a todos los Workers

---

## Documentación

- Documentación Maestra del Sistema Agéntico (DMSA): Portes\_Erika\_ProyectoFinal\_AI\_Experto.pdf  
- Entregas previas : [Entrega 1](https://github.com/erikaportes/Entrega1-Automation-Avanz)  [Entrega 2](https://github.com/erikaportes/entrega2-AI-Aut-Avanz) · [Entrega 3](https://github.com/erikaportes/Entrega3-AI-Automation-Avanzado) · [Entrega 4](https://github.com/erikaportes/Entrega4-AI-Automation-Avanzado) · [Entrega 5](https://github.com/erikaportes/Entrega5-AI-Automation-Avanzado) · [Entrega 6](https://github.com/erikaportes/Entrega6-AI-Automation-Avanzado) · [Entrega 8](https://github.com/erikaportes/Entrega8-AI-Automation-Avanzado)

---

**Erika Portes** · AI Automation Avanzado · 2026