# Flujo de Ingresos 

Este sub-workflow procesa las consultas de revenue que el orquestador le deriva a Milton. Recibe el texto de la pregunta y los metadatos de Slack, normaliza el mensaje, extrae filtros (fechas, países, taxonomía, merchant), construye y ejecuta una consulta SQL sobre el data warehouse y finalmente devuelve una respuesta resumida y formateada para Slack.

---

## Nodo 1 – RC_Trigger

- **Nombre:** `RC_Trigger`  
- **Tipo:** Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
- **Qué hace:**  
  Es el punto de entrada del flujo de ingresos cuando es invocado desde el orquestador.  
  Recibe como inputs el texto de la consulta y los metadatos mínimos necesarios para poder responder luego en el mismo canal e hilo de Slack.

- **Configuración clave:**  
  • Define cuatro `workflowInputs` obligatorios:  
    - `text`  
    - `channel`  
    - `user`  
    - `thread_ts`  
  • No ejecuta lógica; solo expone estos campos al siguiente nodo (`RC_Normalize`).

- **Fragmento representativo (estructura):**
```json
{
  "name": "RC_Trigger",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "parameters": {
    "workflowInputs": {
      "values": [
        { "name": "text", "type": "any" },
        { "name": "channel", "type": "any" },
        { "name": "user", "type": "any" },
        { "name": "thread_ts", "type": "any" }
      ]
    }
  }
```

---
## Nodo 2 – RC_Normalize

- **Nombre:** `RC_Normalize`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Limpia y normaliza el texto de la consulta de revenue para facilitar la extracción de entidades.  
  Elimina acentos, pasa todo a minúsculas, reduce espacios, quita caracteres irrelevantes y genera dos campos:  
  - `text_raw` → texto original  
  - `text_clean` → texto normalizado  
  Además, mantiene `channel`, `user` y `thread_ts` para el resto del flujo.

- **Configuración clave:**  
  • Estandariza la cadena antes de cualquier búsqueda.  
  • Produce un único item con los campos normalizados.  
  • Prepara la entrada para el nodo extractor de parámetros (`RC_ExtractParams`).  

- **Código representativo del nodo:**

```js
// RC_Normalize — Function node
// Modo: Run Once for All Items

const toStr = v => (v == null ? '' : String(v));

function stripAccents(s){
  return toStr(s)
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, ''); // quita diacríticos
}

function normalizeSpaces(s){
  return toStr(s)
    .replace(/[\u00A0\u202F]/g, ' ')   // NBSP / narrow NBSP -> espacio normal
    .replace(/\s+/g, ' ')              // colapsa espacios
    .trim();
}

function pickText(j){
  const cands = [
    j.text,
    j.message,
    j.query,
    j.prompt,
    j.input?.text,
    j.payload?.text,
    j.body?.text,
  ];
  for (const v of cands){
    const s = toStr(v);
    if (s.trim()) return s;
  }
  return '';
}

// --- obtener texto de entrada (robusto) ---
const raw = pickText($json);

if (!raw.trim()) {
  const keys = Object.keys($json || {});
  throw new Error('Revenue Consult: missing required "text" input. Keys present: ' + keys.join(', '));
}

// --- normalizaciones útiles ---
const noAccents = stripAccents(raw);
const text_norm = normalizeSpaces(noAccents);
const text_lower = text_norm.toLowerCase();

// --- salida para los siguientes nodos ---
return [{
  json: {
    // metadatos opcionales que pueden venir del orquestador
    channel: $json.channel ?? null,
    user: $json.user ?? null,
    thread_ts: $json.thread_ts ?? null,

    // texto normalizado
    text_raw: raw,
    text_norm,
    text_lower
  }
}];
```

---
## Nodo 3 – RC_FetchTaxonomy

- **Nombre:** `RC_FetchTaxonomy`  
- **Tipo:** Notion → Get Many (`n8n-nodes-base.notion`)  
- **Qué hace:**  
  Consulta la base de Notion que contiene la **taxonomía de revenue** y trae todas las filas donde el campo `Area` es igual a `Revenue`.  
  Este nodo alimenta al siguiente (`RC_BuildLexicon`) con la lista completa de Activity Group / Activity / Type que Milton puede usar como taxonomía canónica. :contentReference[oaicite:0]{index=0}  

- **Configuración clave:**  
  • Recurso: `databasePage`  
  • Operación: `getAll`  
  • `databaseId`: base de Notion de taxonomía (enlazada por URL)  
  • `returnAll`: `true` (trae todas las filas)  
  • `simple`: `false` (se mantiene el objeto completo de Notion)  
  • Filtro en JSON sobre la columna `Area`:

  ```json
  {
    "property": "Area",
    "title": { "equals": "Revenue" }
  }
  ```

---
## Nodo 4 – RC_BuildLexicon

- **Nombre:** `RC_BuildLexicon`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Construye el **léxico de términos de revenue** a partir de la taxonomía obtenida desde Notion en el nodo anterior.  
  Para cada fila de la tabla de taxonomía, extrae y normaliza:  
  - `Activity Group`  
  - `Activity`  
  - `Type`  
  - `Aliases` (si existen)  

  Genera una lista unificada con todas las variantes posibles que Milton puede reconocer en una consulta.  
  Este léxico luego se usa para mapear palabras del usuario a categorías canónicas (group/activity/type).

- **Configuración clave:**  
  • Recibe múltiples items desde `RC_FetchTaxonomy`.  
  • Normaliza texto (toLowerCase, remove accents).  
  • Construye un array de objetos con estructura consistente.  
  • La salida alimenta directamente al nodo siguiente (`RC_Canonize`).

- **Código representativo del nodo:**

```js
// RC_BuildLexicon
// Lee todas las páginas que llegan de RC_FetchTaxonomy (Area = Revenue)
// y arma listas canónicas + mapas jerárquicos para activity_group, activity, type.

// 1) Helpers
function deaccent(s){
  return String(s || '')
    .normalize('NFD').replace(/[\u0300-\u036f]/g,'');
}
function normKey(s){
  return deaccent(s).toLowerCase().replace(/\s+/g,' ').trim();
}
function readPropValue(prop){
  if (!prop || !prop.type) return null;
  const t = prop.type;
  if (t === 'select')       return prop.select?.name ?? null;
  if (t === 'multi_select') return prop.multi_select?.[0]?.name ?? null; // usamos el primero
  if (t === 'title')        return (prop.title?.[0]?.plain_text ?? '').trim() || null;
  if (t === 'rich_text')    return (prop.rich_text?.[0]?.plain_text ?? '').trim() || null;
  if (typeof prop[t] === 'string') return prop[t].trim() || null; // fallback genérico
  return null;
}
function pick(props, candidates){
  for (const k of candidates){
    if (props[k]) return readPropValue(props[k]);
  }
  return null;
}

// 2) Configura aquí los nombres de columnas (por si difieren)
const GROUP_KEYS    = ['Activity Group','Group','activity_group','activity group'];
const ACTIVITY_KEYS = ['Activity','activity'];
const TYPE_KEYS     = ['Type','type'];

// 3) Recorremos items
const items = $items(); // cada item es una "page"
const groupsSet = new Set();
const activitiesSet = new Set();
const typesSet = new Set();

// Relaciones jerárquicas canónicas (usamos los nombres tal cual Notion)
const typeToActivity = {};   // Type -> Activity
const activityToGroup = {};  // Activity -> Group

for (const it of items){
  const page = it.json || {};
  const props = page.properties || {};

  const Graw = pick(props, GROUP_KEYS);
  const Araw = pick(props, ACTIVITY_KEYS);
  const Traw = pick(props, TYPE_KEYS);

  const G = Graw ? Graw.trim() : null;
  const A = Araw ? Araw.trim() : null;
  const T = Traw ? Traw.trim() : null;

  if (G) groupsSet.add(G);
  if (A) activitiesSet.add(A);
  if (T) typesSet.add(T);

  if (T && A) typeToActivity[T] = A;
  if (A && G) activityToGroup[A] = G;
}

// 4) También armamos mapas de normalización (para tolerancia a mayúsculas/acentos)
const groups = Array.from(groupsSet).sort();
const activities = Array.from(activitiesSet).sort();
const types = Array.from(typesSet).sort();

const normGroups = {};
for (const g of groups) normGroups[normKey(g)] = g;

const normActivities = {};
for (const a of activities) normActivities[normKey(a)] = a;

const normTypes = {};
for (const t of types) normTypes[normKey(t)] = t;

// 5) Salida
return [{
  json: {
    groups,
    activities,
    types,
    type_to_activity: typeToActivity,
    activity_to_group: activityToGroup,
    // Mapas de normalización (útiles si querés canónicas a partir de texto libre)
    norm_maps: {
      group_by_norm: normGroups,
      activity_by_norm: normActivities,
      type_by_norm: normTypes,
    }
  }
}];
```

---
## Nodo 5 – RC_LLMExtract

- **Nombre:** `RC_LLMExtract`  
- **Tipo:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.openAi`)  
- **Qué hace:**  
  Extrae filtros avanzados desde el texto del usuario usando un modelo LLM.  
  El modelo recibe:
  - el texto original del usuario  
  - la lista canónica de `activity_groups`, `activities` y `types` proveniente de `RC_BuildLexicon`  

  Y devuelve un **único JSON** con:
  - activity_groups  
  - activities  
  - types  
  - countries  
  - merchant_id  
  - date_start  
  - date_end  
  - date_granularity  

  Este nodo es clave porque detecta expresiones complejas como:
  - “last 3 months”  
  - “between August and October 2025”  
  - “revenue for Cards Payment in Argentina”  
  - “Cross Border Transfer in Mexico last year”  
  Y asigna la taxonomía *solo* usando los valores oficiales del lexicón.

- **Configuración clave:**  
  • Modelo: **gpt-4.1-mini**  
  • `jsonOutput: true` (el modelo devuelve JSON estructurado)  
  • `temperature: 0` para respuestas determinísticas  
  • El `system` prompt es extenso e incluye reglas exactas para fechas, país y taxonomía.  
  • El mensaje `user` incluye:  
    - texto original  
    - listas canónicas generadas por `RC_BuildLexicon`  
    - ejemplos few-shot  

- **Código representativo del nodo (estructura):**

```text
System Prompt:

You extract filters for a revenue SQL query.
Return ONLY a single JSON object. No prose, no code blocks.

If a parameter is not specified by the user, set it to null or an empty array as indicated below.

DATES:
- If a single month is mentioned (e.g., "August 2025"), return date_start=first day of that month and date_end=last day (inclusive). date_granularity="month".
- If a range like "from Aug 2025 to Oct 2025" or "between 2025-08-01 and 2025-10-31" is mentioned, use inclusive boundaries. Granularity: "month" for month names, "day" for explicit dates.
- If "last/past N months" (1..24) is mentioned, set start to the FIRST day of the month N−1 months before the end of THIS month, and end to the LAST day of THIS month. Granularity="month".
- "in the last year"/"past year": last 12 months up to the end of THIS month. Granularity="month".
- "last year" (without “in the”): previous calendar year (Jan 1–Dec 31). Granularity="month".

COUNTRY:
- Return COUNTRY NAMES, not ISO codes. Multiple countries allowed.

CANONICAL TAXONOMY (VERY IMPORTANT):
- You MUST choose values ONLY from the canonical lists provided in the user message.
- Case-insensitive and accent-insensitive matching is allowed. If the wording is close (e.g., "cards payment" ~ "Cards Payment"), choose the closest canonical item.
- If the text clearly refers to a GROUP only, fill `activity_groups` with that item and leave `activities`/`types` empty. Do NOT invent items.

OUTPUT (JSON only):
{
  "activity_groups": string[] | [],
  "activities": string[] | [],
  "types": string[] | [],
  "countries": string[] | [],
  "merchant_id": null | integer,
  "date_start": null | "YYYY-MM-DD",
  "date_end": null | "YYYY-MM-DD",
  "date_granularity": null | "month" | "day"
}
```

```text
User Prompt:
Text: {{$items("RC_Normalize")[0].json.text_raw}}

Canonical lists (use exact strings if you output them):
activity_groups: {{$jsonGet($items("RC_BuildLexicon")[0].json,"groups")}}
activities: {{$jsonGet($items("RC_BuildLexicon")[0].json,"activities")}}
types: {{$jsonGet($items("RC_BuildLexicon")[0].json,"types")}}

Few-shot examples (follow strictly):
- "revenue in argentina for cards payment" ⇒ activity_groups=["Cards Payment"], countries=["Argentina"]
- "credit card payment in brazil" ⇒ activity_groups=["Cards Payment"], countries=["Brazil"]
- "cross border transfer in mexico" ⇒ activities=["Cross-Border Transfer"], countries=["Mexico"]

Now return ONLY the JSON.
```

---

