# Milton Orquestador

Este flujo orquesta todas las consultas dirigidas a Milton desde Slack: recibe el mensaje, limpia el texto, clasifica la intención (definiciones, revenue, scoring o desconocido) y deriva la conversación al sub-workflow correspondiente. 

---

## Nodo 1 – MT_SlackTrigger

- **Nombre:** `MT_SlackTrigger`  
- **Tipo:** Slack Trigger (`n8n-nodes-base.slackTrigger`)  
- **Qué hace:**  
  Escucha eventos `app_mention` en todo el workspace de Slack. Cada vez que alguien menciona a Milton, dispara el workflow y pasa el payload completo del evento (texto, usuario, canal, timestamps, etc.).

- **Configuración clave:**
  - `trigger`: `["app_mention"]`
  - `watchWorkspace`: `true`
  - Credenciales: cuenta de Slack `Milton Account`

- **Código / expresiones:**  
  Este nodo no ejecuta JavaScript propio; solo define el trigger. Su configuración esencial en JSON es:

  ```json
  {
    "type": "n8n-nodes-base.slackTrigger",
    "parameters": {
      "trigger": ["app_mention"],
      "watchWorkspace": true,
      "options": {}
    },
    "credentials": {
      "slackApi": {
        "name": "Milton Account"
      }
    }
  }

---

## Nodo 2 – MT_SetFields

- **Nombre:** `MT_SetFields`  
- **Tipo:** Set (`n8n-nodes-base.set`)  
- **Qué hace:**  
  Extrae y normaliza los campos básicos del evento de Slack. Asegura que siempre existan `text`, `channel`, `user`, `ts` y `thread_ts`, tomando valores desde cualquier variante del payload (`json`, `event`, `body.event`). Prepara un objeto limpio y plano para el resto del flujo.

- **Configuración clave:**  
  Usa asignaciones individuales para cada campo. No crea items adicionales.

- **Código exacto (expresiones del nodo):**

```js
text = {{$json.text || $json.event?.text || $json.body?.event?.text || ''}}

channel = {{ 
  ($json.channel || $json.event?.channel || $json.body?.event?.channel || '')
    .replace(/[\u000A\u000D\u202F\u00A0\s]+$/g, '')
    .trim()
}}

user = {{$json.user || $json.event?.user || $json.body?.event?.user || ''}}

ts = {{$json.ts || $json.event?.ts || $json.body?.event?.ts || ''}}

thread_ts = {{
  $json.thread_ts ||
  $json.event?.thread_ts ||
  $json.body?.event?.thread_ts ||
  $json.ts ||
  $json.event?.ts ||
  $json.body?.event?.ts ||
  ''
}}
```

---
## Nodo 3 – MT_CleanNormalize

- **Nombre:** `MT_CleanNormalize`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Limpia y normaliza el texto recibido desde Slack.  
  Elimina menciones `<@user>`, colapsa espacios, pasa todo a minúsculas y remueve acentos.  
  Devuelve el texto original (`text`) y una versión normalizada (`clean_text`) junto con los metadatos del mensaje.

- **Configuración clave:**  
  • No usa inputs adicionales.  
  • Produce **un solo item** con el texto limpio.  

- **Código exacto del nodo:**

```js
function deacc(s) {
  return s.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
}

const text = ($json.text || '').trim();

// quita menciones tipo <@UXXXXXX> y colapsa espacios
let t = text.replace(/<@[^>]+>/g, ' ').replace(/\s+/g, ' ').trim();

// lower + sin acentos
t = deacc(t.toLowerCase());

return [{
  text: $json.text || '',
  clean_text: t,
  channel: $json.channel || '',
  user: $json.user || '',
  ts: $json.ts || '',
  thread_ts: $json.thread_ts || $json.ts
}];
```

---
## Nodo 4 – MT_LLMClassifier

- **Nombre:** `MT_LLMClassifier`  
- **Tipo:** Message a Model (OpenAI) (`@n8n/n8n-nodes-langchain.openAi`)  
- **Qué hace:**  
  Envía el texto original y el texto normalizado al modelo **gpt-4o** para clasificar la intención del mensaje.  
  El modelo no responde la pregunta: únicamente devuelve un JSON con un campo `"intent"` que puede ser `definitions`, `revenue`, `scoring` o `unknown`.

- **Configuración clave:**  
  • Modelo: `gpt-4o`  
  • Temperature: `0.1`  
  • Max Tokens: `300`  
  • Envía dos valores: `text` (original) y `clean_text` (normalizado).  
  • El *system prompt* instruye al modelo a responder exclusivamente con un JSON de un solo nivel.

- **System Prompt exacto del nodo:**

```text
SYSTEM PROMPT — Intent Classifier for Milton (FP&A Assistant)

You are an intent classifier for an internal financial assistant called Milton, used by the FP&A team at AstroPay.
Users write short English messages to Milton on Slack asking either for definitions of financial concepts or for revenue-related figures.
They may also ask for fraud or scoring-related model information.
Your job is to analyze the user message and return only the intent of the question.

You must not provide an answer, extract keywords, or include any explanation.
Output only one JSON object in a single line, with this format:

{"intent":"definitions"}
or
{"intent":"revenue"}
or
{"intent":"scoring"}
or
{"intent":"unknown"}

INTENTS
definitions
The user is asking for the meaning, explanation, or definition of a financial term, KPI, or concept (for example, a revenue component, COGS, or OPEX item).
These messages are conceptual, not numerical.
They often include trigger patterns such as:

"what is ..."
"define ..."
"explain ..."
"meaning of ..."
"how do we calculate ..."
"can you clarify ..."
"what does ... mean"
"Index"

These questions typically refer to understanding what something means, not how much something is.

revenue
The user is asking for a numeric or factual value related to revenue or income.
The presence of words like revenue, sales, income, billing, turnover, or GMV is a strong indicator of this intent.
These queries often include a time period, country, merchant, or activity grouping (for example: group activity, activity, or type stored in the Notion database used in the Financial Definitions flow).
Common trigger patterns include:

"what was the revenue in ..."
"show revenue by ..."
"how much did we make ..."
"total sales for ..."
"GMV for ..."
"income by ..."
"billing by ..."

Any question that explicitly refers to revenue or income values should be classified as "revenue".

scoring
The user is asking about fraud-risk, scoring models, transaction risk, or any kind of model-based evaluation.
These queries often include terms like fraud, score, scoring, risk, model, prediction, or classification.
Common trigger patterns include:

"provide info for user id ..."
"what is the fraud score for user ..."
"evaluate risk for user id ..."
"fraud score user ..."
"fraud scoring for transaction ..."
"risk evaluation for merchant ..."
"fraud risk prediction for user ..."

Any question that refers to fraud, risk scoring, or predictive model output should be classified as "scoring".

unknown
Anything that does not clearly fit either definitions or revenue or scoring.
This includes unrelated questions, greetings, or ambiguous prompts.

CONFIDENCE

If the message contains strong signals like "what is" or "define" → confidence is high for "definitions".
If it contains "revenue", "sales", "income", "GMV", "how much", or similar → confidence is high for "revenue".
If it contains "score", "scoring", "fraud", "risk", "model", or "predict" → confidence is high for "scoring".
If neither pattern is clear → classify as "unknown".

OUTPUT FORMAT

Output must be a single-line JSON object with only one field: "intent".
No markdown, no explanations, no additional fields.

Examples of valid outputs:
{"intent":"definitions"}
{"intent":"revenue"}
{"intent":"scoring"}
{"intent":"unknown"}

EXAMPLES
Definitions intent
Input: @Milton what is merchant settlement?
Output: {"intent":"definitions"}

Input: explain what wallet use means
Output: {"intent":"definitions"}

Input: can you define take rate?
Output: {"intent":"definitions"}

Input: what does processing cost mean?
Output: {"intent":"definitions"}

Input: how do we calculate gross profit?
Output: {"intent":"definitions"}

Input: what is the meaning of transfer gateway fee?
Output: {"intent":"definitions"}

Input: can you clarify what FX surcharge refers to?
Output: {"intent":"definitions"}

Input: Index
Output: {"intent":"definitions"}

Revenue intent

Input: what was the revenue in Argentina for July 2025?
Output: {"intent":"revenue"}

Input: show me the total revenue for Brazil last quarter
Output: {"intent":"revenue"}

Input: how much did we make in August?
Output: {"intent":"revenue"}

Input: revenue by merchant 10166 for Q3 2024
Output: {"intent":"revenue"}

Input: total sales for Argentina year to date
Output: {"intent":"revenue"}

Input: income by merchant 20218 last month
Output: {"intent":"revenue"}

Input: GMV for India this month
Output: {"intent":"revenue"}

Input: billing by country for June
Output: {"intent":"revenue"}

Input: turnover by merchant last year
Output: {"intent":"revenue"}

Input: revenue for group activity Wallet Payment this quarter
Output: {"intent":"revenue"}

Input: what was the income for merchant Stake in Argentina last month?
Output: {"intent":"revenue"}

Scoring intent

Input: provide info for user id 4324
Output: {"intent":"scoring"}

Input: what is the fraud score for user 4326?
Output: {"intent":"scoring"}

Input: evaluate risk for user id 2345
Output: {"intent":"scoring"}

Input: fraud score user 4322
Output: {"intent":"scoring"}

Input: fraud risk prediction for user 5462
Output: {"intent":"scoring"}

Unknown intent

Input: hello Milton
Output: {"intent":"unknown"}

Input: can you update the dashboard?
Output: {"intent":"unknown"}

Input: who is the CEO?
Output: {"intent":"unknown"}

Input: good morning
Output: {"intent":"unknown"}

```

---
## Nodo 5 – MT_CombineOriginalAndLLM

- **Nombre:** `MT_CombineOriginalAndLLM`  
- **Tipo:** Merge (`n8n-nodes-base.merge`)  
- **Qué hace:**  
  Combina en un solo item los dos flujos provenientes de los nodos anteriores:  
  1) el mensaje normalizado (`MT_CleanNormalize`)  
  2) la respuesta del clasificador LLM (`MT_LLMClassifier`)  
  El resultado es un item unificado que contiene tanto el texto limpio como la salida del modelo, para poder ser parseado correctamente en el siguiente nodo.

- **Configuración clave:**  
  • Modo de merge: **Append**  
  • Número de items combinados: **1 a 1**  
  • No realiza procesamiento adicional; solo agrupa los outputs en un array que luego será consumido por `MT_ParseLLM`.

---
## Nodo 6 – MT_ParseLLM

- **Nombre:** `MT_ParseLLM`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Toma los dos items combinados por el merge (texto original/normalizado + respuesta del LLM) y produce **un único objeto final**.  
  Identifica cuál item pertenece al LLM y cuál al mensaje original, parsea el JSON devuelto por el modelo y reconstruye todos los campos necesarios (`intent`, `entities`, `channel`, `thread_ts`, `user`, etc.).  
  Su función es dejar el input listo y homogéneo para el router de intención.

- **Configuración clave:**  
  • Entrada múltiple desde el Merge (2 items).  
  • Produce una sola salida consolidada.  
  • Valida y normaliza campos críticos (especialmente `channel` y `intent`).  
  • Captura errores de parseo del JSON del modelo y asigna `unknown` si corresponde.

```js
// === Utilities ===
const S = v => (v == null ? '' : String(v)).trim();
const tryParse = s => { try { return JSON.parse(s); } catch { return null; } };

// === Items entrantes (del Merge append) ===
const items = $input.all();

// 1) Item del LLM (assistant)
const llmItem =
  items.find(i => i.json?.message?.role === 'assistant') ||
  items.find(i => typeof i.json?.content === 'string') || null;

// 2) Item original (texto/canales)
const origItem =
  items.find(i => i.json?.text || i.json?.clean_text || i.json?.channel) ||
  items[0] || { json: {} };

// 3) Slack trigger por si falta algo
const trig = $item(0, 'Slack Trigger')?.json || {};

// === Campos de contexto (limpios) ===
const text       = S(origItem.json.text) || S(trig.text) || S(trig.event?.text);
const clean_text = S(origItem.json.clean_text) || S(text).toLowerCase();

let channel = S(origItem.json.channel) || S(trig.event?.channel) || S(trig.channel);
channel = channel.replace(/\s+/g, ''); // quita espacios/saltos

let thread_ts =
  S(origItem.json.thread_ts) ||
  S(trig.event?.thread_ts) ||
  S(origItem.json.ts) ||
  S(trig.event?.ts) ||
  S(trig.ts);

// === Parsear JSON del LLM ===
const raw = S(llmItem?.json?.message?.content || llmItem?.json?.content || '');
const obj = tryParse(raw) || tryParse(raw.replace(/```json|```/g, '')) || null;
const intent = (S(obj?.intent).toLowerCase()) || 'unknown';

// === Salida para el router ===
return [{
  text,
  clean_text,
  channel,
  user: S(origItem.json.user) || S(trig.event?.user) || S(trig.user),
  ts:   S(origItem.json.ts)   || S(trig.event?.ts)   || S(trig.ts),
  thread_ts,
  intent,
  entities: {
    term:        obj?.entities?.term ?? null,
    country:     obj?.entities?.country ?? null,
    merchant_id: obj?.entities?.merchant_id ?? null,
    month:       obj?.entities?.month ?? null,
    year:        obj?.entities?.year ?? null,
    period:      obj?.entities?.period ?? null,
  },
  confidence: typeof obj?.confidence === 'number' ? obj.confidence : 0,
}];

```

---

