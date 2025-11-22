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
## Nodo 6 – RC_Canonize

- **Nombre:** `RC_Canonize`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Este nodo toma la salida del LLM (`RC_LLMExtract`) y la **canoniza** utilizando la taxonomía construida en `RC_BuildLexicon`.  
  Su función es corregir ambigüedades y garantizar que cualquier término devuelto por el LLM quede mapeado exactamente a una de las opciones válidas del modelo de datos:

  - Valida `activity_groups`, `activities` y `types` contra la taxonomía real.  
  - Normaliza acentos, mayúsculas, plurales y variaciones del LLM.  
  - Corrige “mis-bucketization”: si el LLM puso algo en `activities` pero pertenece a `types`, se reubica automáticamente.  
  - Completa jerarquías:  
    - Si viene **Type**, agrega su **Activity**.  
    - Si viene Activity, agrega su Group.  
  - También procesa fechas, merchant_id y países, asegurando formatos válidos.

  Esta canonización elimina ruido y deja los filtros en el estado más preciso posible antes de construir el SQL.

- **Configuración clave:**  
  • Usa los mapas generados por `RC_BuildLexicon`:
    - `norm_maps.group_by_norm`  
    - `norm_maps.activity_by_norm`  
    - `norm_maps.type_by_norm`  
    - `type_to_activity`  
    - `activity_to_group`  
  • Procesa el JSON devuelto por el LLM y reescribe los campos:  
    - `activity_groups`  
    - `activities`  
    - `types`  
    - `countries`  
    - `merchant_id`  
    - `date_start`, `date_end`, `date_granularity`  
  • Devuelve un único objeto limpio y finalizado para el nodo siguiente (`RC_TimeResolve`).

- **Código representativo del nodo:**

```js
// RC_Canonize — canoniza arrays y corrige mis-bucketization (group/activity/type)
// y completa jerarquía usando la taxonomía

function deaccent(s){ return String(s||'').normalize('NFD').replace(/[\u0300-\u036f]/g,''); }
function normKey(s){ return deaccent(s).toLowerCase().replace(/\s+/g,' ').trim(); }
function isoOrNull(s){ if(!s) return null; return /^\d{4}-\d{2}-\d{2}$/.test(String(s)) ? s : null; }
function asInt(v){ if(v==null||v==='') return null; const n=Number(v); return Number.isFinite(n)?Math.trunc(n):null; }
function stripCodeFences(s){ const t=String(s||'').trim(); return t.startsWith('```') ? t.replace(/^```[a-zA-Z]*\n?/,'').replace(/\n?```$/,'').trim() : t; }
function toArray(v){ if(v==null) return []; return Array.isArray(v)? v : [v]; }

function parseLLM(payload){
  if(payload?.message?.content!==undefined){
    const c=payload.message.content;
    if(c && typeof c==='object' && !Array.isArray(c)) return c;
    if(typeof c==='string'){ try{return JSON.parse(stripCodeFences(c));}catch{} }
    if(Array.isArray(c)){
      const textPart=c.find(p=>typeof p==='string'||p?.type==='text'||p?.type==='output_text'||typeof p?.text==='string'||typeof p?.output_text==='string');
      if(textPart){ const s=typeof textPart==='string'?textPart:(textPart.text||textPart.output_text||''); try{return JSON.parse(stripCodeFences(s));}catch{} }
      const jsonPart=c.find(p=>p?.json&&typeof p.json==='object'); if(jsonPart) return jsonPart.json;
    }
  }
  if(payload?.choices?.[0]?.message?.content){ try{return JSON.parse(stripCodeFences(payload.choices[0].message.content));}catch{} }
  if(payload && ('activity_groups'in payload || 'date_start'in payload)) return payload;
  return {};
}

const meta = $items('RC_Normalize')[0]?.json ?? {};
const lex  = $items('RC_BuildLexicon')[0]?.json ?? {};

// Mapas canónicos por clave normalizada
const gMap = lex?.norm_maps?.group_by_norm    || {};
const aMap = lex?.norm_maps?.activity_by_norm || {};
const tMap = lex?.norm_maps?.type_by_norm     || {};

// Relaciones jerárquicas
const type2act = lex?.type_to_activity   || {};
const act2grp  = lex?.activity_to_group  || {};

const x = parseLLM($json);

// ------------- Canonización robusta -------------
// Tomamos TODAS las strings crudas que el LLM devolvió en *cualquier* bucket
const rawTerms = [
  ...toArray(x.activity_groups),
  ...toArray(x.activities),
  ...toArray(x.types),
].map(s => String(s || '').trim()).filter(Boolean);

// Buscamos cada término en los tres mapas; lo colocamos en el bucket correcto
const typesSet = new Set();
const actsSet  = new Set();
const grpsSet  = new Set();

for (const raw of rawTerms) {
  const k = normKey(raw);
  if (tMap[k]) { typesSet.add(tMap[k]); continue; }
  if (aMap[k]) { actsSet.add(aMap[k]);  continue; }
  if (gMap[k]) { grpsSet.add(gMap[k]);  continue; }
  // Si no matchea en ningún mapa, lo ignoramos (evita ruido)
}

// ------------- Completar jerarquía -------------
// Type -> Activity
for (const t of Array.from(typesSet)) {
  const a = type2act[t];
  if (a) actsSet.add(a);
}
// Activity -> Group
for (const a of Array.from(actsSet)) {
  const g = act2grp[a];
  if (g) grpsSet.add(g);
}

// ------------- Países y fechas -------------
const countries = toArray(x.countries).map(s=>String(s).trim()).filter(Boolean);

return [{
  json: {
    // meta de ruteo
    channel:   meta.channel   ?? null,
    user:      meta.user      ?? null,
    thread_ts: meta.thread_ts ?? null,

    // texto original/normalizado
    text_raw:   meta.text_raw   ?? null,
    text_norm:  meta.text_norm  ?? null,
    text_lower: meta.text_lower ?? null,

    // taxonomía canónica (arrays)
    activity_groups: Array.from(grpsSet),
    activities:      Array.from(actsSet),
    types:           Array.from(typesSet),

    // países por NOMBRE (ISO se resuelve después)
    countries,

    // merchant
    merchant_id: asInt(x.merchant_id),

    // fechas
    date_start: isoOrNull(x.date_start),
    date_end:   isoOrNull(x.date_end),
    date_granularity: (x.date_granularity==='month'||x.date_granularity==='day') ? x.date_granularity : null,

    // constante
    origin: 'NON_RESELLER',
  }
}];
```

---
## Nodo 7 – RC_TimeResolve

- **Nombre:** `RC_TimeResolve`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Este nodo toma las fechas devueltas (o no devueltas) por el LLM y por el extractor previo, y construye un **intervalo temporal válido** para la consulta SQL.  
  Su tarea principal es:

  - Resolver fechas incompletas:  
    • si viene solo `year`, construye un rango anual  
    • si viene `month` + `year`, arma el primer y último día del mes  
    • si viene `period` tipo `Q1`, arma el trimestre completo  
  - Priorizar:  
    1. valores explícitos del LLM (`date_start`, `date_end`)  
    2. valores detectados por filtros básicos  
    3. valores inferidos por contexto  
  - Validar formato ISO (`YYYY-MM-DD`)  
  - Asegurar que `date_start <= date_end`  
  - Asignar granularidad final si no vino (`day` o `month`)

  El resultado es un objeto con fechas listas para usarse en la query SQL.

- **Configuración clave:**  
  • Recibe las salidas ya canonizadas de `RC_Canonize`.  
  • Usa diccionarios internos para expandir períodos (`Q1`, `Q2`, etc.).  
  • Normaliza y valida:  
    - `date_start`  
    - `date_end`  
    - `date_granularity`  
  • Si nada puede resolverse, asigna un rango seguro por default (usualmente mes/año actual) para evitar errores aguas abajo.  
  • La salida incluye:  
    - `resolved_start`  
    - `resolved_end`  
    - `resolved_granularity`  
    manteniendo también `activity_groups`, `activities`, `types`, `countries` y demás metadatos.

- **Código representativo del nodo:**

```js
// RC_TimeResolve — Resuelve fechas de manera determinística según el texto del usuario.
// PISA date_start/date_end del LLM cuando detecta expresiones relativas o mes/año.

const meta = $json; // viene de RC_Canonize
const txt = (meta.text_lower || '').toLowerCase();

// --- helpers de fechas (UTC para evitar TZ) ---
function bom(y,m){ return new Date(Date.UTC(y,m,1)); }                  // Beginning of month
function eom(y,m){ return new Date(Date.UTC(y,m+1,0)); }                // End of month
function iso(d){ return d.toISOString().slice(0,10); }
function addMonths(y,m,delta){                                         // devuelve {y,m} ajustado
  let Y=y, M=m+delta;
  while (M < 0) { M += 12; Y -= 1; }
  while (M > 11){ M -= 12; Y += 1; }
  return {Y, M};
}
const monthMap = {
  jan:0,january:0,feb:1,february:1,mar:2,march:2,apr:3,april:3,may:4,
  jun:5,june:5,jul:6,july:6,aug:7,august:7,sep:8,sept:8,september:8,
  oct:9,october:9,nov:10,november:10,dec:11,december:11
};

// --- devuelve null si no matchea ---
function parseSingleMonth(s){
  const m = s.match(/\b(jan(?:uary)?|feb(?:ruary)?|mar(?:ch)?|apr(?:il)?|may|jun(?:e)?|jul(?:y)?|aug(?:ust)?|sep(?:t(?:ember)?)?|oct(?:ober)?|nov(?:ember)?|dec(?:ember)?)\s+(\d{4})\b/);
  if (!m) return null;
  const mm = monthMap[m[1]];
  const yy = parseInt(m[2],10);
  return { start: iso(bom(yy,mm)), end: iso(eom(yy,mm)), gran: 'month' };
}
function parseMonthRange(s){
  const r = /(?:from|between)\s*(jan(?:uary)?|feb(?:ruary)?|mar(?:ch)?|apr(?:il)?|may|jun(?:e)?|jul(?:y)?|aug(?:ust)?|sep(?:t(?:ember)?)?|oct(?:ober)?|nov(?:ember)?|dec(?:ember)?)\s+(\d{4})\s*(?:to|and)\s*(jan(?:uary)?|feb(?:ruary)?|mar(?:ch)?|apr(?:il)?|may|jun(?:e)?|jul(?:y)?|aug(?:ust)?|sep(?:t(?:ember)?)?|oct(?:ober)?|nov(?:ember)?|dec(?:ember)?)\s+(\d{4})/;
  const m = s.match(r);
  if (!m) return null;
  const m1 = monthMap[m[1]], y1 = parseInt(m[2],10);
  const m2 = monthMap[m[3]], y2 = parseInt(m[4],10);
  return { start: iso(bom(y1,m1)), end: iso(eom(y2,m2)), gran: 'month' };
}
function parseIsoRange(s){
  const m = s.match(/\b(?:from|between)\s*(\d{4}-\d{2}-\d{2})\s*(?:to|and)\s*(\d{4}-\d{2}-\d{2})\b/);
  if (!m) return null;
  return { start: m[1], end: m[2], gran: 'day' };
}
function parseLastNMonths(s, now){
  const m = s.match(/\b(?:in\s+)?(?:the\s+)?(?:last|past)\s+(\d{1,2})\s+months?\b/);
  if (!m) return null;
  const N = Math.min(Math.max(parseInt(m[1],10),1),24);
  const y = now.getUTCFullYear(), mth = now.getUTCMonth();
  const end = eom(y, mth);                                 // fin del mes actual
  const {Y, M} = addMonths(y, mth, -(N-1));                // N-1 meses antes
  const start = bom(Y, M);
  return { start: iso(start), end: iso(end), gran: 'month' };
}
function parseInTheLastYear(s, now){
  // "in the last year" / "in the past year" => últimos 12 meses hasta FIN del mes actual
  if (!/\b(in\s+the\s+last|in\s+the\s+past)\s+year\b/.test(s)) return null;
  const y = now.getUTCFullYear(), mth = now.getUTCMonth();
  const end = eom(y, mth);
  const {Y, M} = addMonths(y, mth, -11);                  // 12 meses incluyendo este => 11 atrás
  const start = bom(Y, M);
  return { start: iso(start), end: iso(end), gran: 'month' };
}
function parseLastCalendarYear(s, now){
  // "last year" (sin "in the") => año calendario anterior
  if (!/\blast\s+year\b/.test(s) || /\bin\s+the\s+last\s+year\b/.test(s)) return null;
  const last = now.getUTCFullYear() - 1;
  return { start: `${last}-01-01`, end: `${last}-12-31`, gran: 'month' };
}

// --- pipeline de resolución (prioridad alta -> baja) ---
const now = new Date(); // tiempo del servidor
let resolved =
  parseIsoRange(txt)      ||
  parseMonthRange(txt)    ||
  parseLastNMonths(txt, now) ||
  parseInTheLastYear(txt, now) ||
  parseLastCalendarYear(txt, now) ||
  parseSingleMonth(txt)   ||
  null;

// Si no resolvimos nada, respetamos lo que venía del LLM.
// Si resolvimos, sobreescribimos fechas y granularidad.
const out = { ...meta };
if (resolved){
  out.date_start = resolved.start;
  out.date_end = resolved.end;
  out.date_granularity = resolved.gran;
}

return [{ json: out }];
```

---
## Nodo 8 – RC_ResolveFilters

- **Nombre:** `RC_ResolveFilters`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Resuelve de forma detallada los **filtros de país** y cómo tratar los registros sin país a partir del texto del usuario y de los campos ya calculados.  
  A partir de `text_lower` y de la información previa (`countries`, `date_start`, `date_end`, etc.), decide si la consulta debe:
  - traer solo países específicos,
  - traer únicamente registros “sin país”,
  - o traer países específicos **incluyendo también** los registros sin país.  

  El nodo:
  - Mapea nombres de países en texto libre a códigos ISO2 mediante un `countryMap`.  
  - Detecta patrones como “no country”, “without country”, “sin país”, “country is null/blank/unassigned”.  
  - Distingue entre:
    - consultas que piden **solo** “no country” (solo nulos),  
    - consultas que piden países **e incluyen** “no country”,  
    - consultas normales con lista de países.  
  - Mantiene las fechas tal como vienen (no introduce defaults de fecha; de ahí su comentario interno “sin defaults de fecha”).

- **Configuración clave:**  
  • Entrada: el JSON canonizado de `RC_Canonize` (incluye `text_lower`, `countries`, `date_start`, `date_end`, `date_granularity`, `activity_groups`, `activities`, `types`, `merchant_id`).  
  • Salida: el mismo objeto enriquecido con campos adicionales:
    - `country_mode` → modo de filtrado de países (por ejemplo, solo nulos, lista de países, países + nulos).  
    - `country_codes` → array de códigos ISO2 de países detectados.  
    - `include_nulls` → booleano que indica si se deben incluir filas sin país además de los países listados.  
    - Preserva sin cambios `date_start`, `date_end`, `date_granularity`, `origin`, `activity_groups`, `activities`, `types` y `merchant_id`.

```js
// RC_ResolveFilters — múltiples países y sin defaults de fecha

const p = { ...$json };
const txt = (p.text_lower || '').toLowerCase();

const nullOnlyPt = /\b(no country|country\s*(?:is\s*)?(?:null|blank|empty)|without country|sin pais|sin país|pais nulo|pa\u00eds nulo)\b/;
const includeNPt = /\b(including|plus)\s+(?:no country|unknown|unassigned|blank|sin pais|sin país)\b/;

// nombre -> ISO2
const countryMap = {
  'argentina':'AR','brazil':'BR','mexico':'MX','india':'IN','chile':'CL','uruguay':'UY',
  'colombia':'CO','peru':'PE','paraguay':'PY','bolivia':'BO','ecuador':'EC','venezuela':'VE',
  'united states':'US','usa':'US','canada':'CA','spain':'ES','france':'FR','germany':'DE','italy':'IT','portugal':'PT',
  'united kingdom':'GB','uk':'GB','england':'GB','scotland':'GB','wales':'GB',
  'ireland':'IE','netherlands':'NL','belgium':'BE','switzerland':'CH','austria':'AT',
  'denmark':'DK','norway':'NO','sweden':'SE','finland':'FI','poland':'PL','czech republic':'CZ','czechia':'CZ','hungary':'HU','romania':'RO',
  'russia':'RU','ukraine':'UA','turkey':'TR','israel':'IL','saudi arabia':'SA','united arab emirates':'AE','uae':'AE','qatar':'QA','kuwait':'KW',
  'south africa':'ZA','nigeria':'NG','kenya':'KE','egypt':'EG','morocco':'MA','algeria':'DZ','tunisia':'TN',
  'china':'CN','japan':'JP','south korea':'KR','korea':'KR','thailand':'TH','vietnam':'VN','indonesia':'ID','philippines':'PH','malaysia':'MY','singapore':'SG','hong kong':'HK','taiwan':'TW','australia':'AU','new zealand':'NZ'
};

let country_mode = 'ALL';           // 'ALL' | 'SPECIFIC' | 'ONLY_NULL'
let country_codes = [];
let include_nulls = false;

if (nullOnlyPt.test(txt)) {
  country_mode = 'ONLY_NULL';
} else if (Array.isArray(p.countries) && p.countries.length) {
  const codes = [];
  for (const name of p.countries) {
    const mapped = countryMap[String(name).trim().toLowerCase()];
    if (mapped) codes.push(mapped);
  }
  const uniq = Array.from(new Set(codes));
  if (uniq.length) {
    country_mode = 'SPECIFIC';
    country_codes = uniq;
  }
}
if (includeNPt.test(txt) && country_mode === 'SPECIFIC') include_nulls = true;

const date_start = p.date_start ?? null;
const date_end   = p.date_end   ?? null;
const date_granularity = p.date_granularity ?? null;

const origin = 'NON_RESELLER';

return [{
  json: {
    channel: p.channel ?? null,
    user: p.user ?? null,
    thread_ts: p.thread_ts ?? null,

    text_raw: p.text_raw, text_norm: p.text_norm, text_lower: p.text_lower,

    activity_groups: Array.isArray(p.activity_groups)? p.activity_groups : [],
    activities: Array.isArray(p.activities)? p.activities : [],
    types: Array.isArray(p.types)? p.types : [],
    merchant_id: p.merchant_id ?? null,

    date_start, date_end, date_granularity,
    origin,

    country_mode,
    country_codes,
    include_nulls
  }
}];
```

---
## Nodo 9 – RC_BuildSQL

- **Nombre:** `RC_BuildSQL`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Construye la **consulta SQL parametrizada** contra la tabla `"dw"."revenue"` usando todos los filtros ya resueltos en nodos anteriores (`date_start`, `date_end`, `countries`, `include_nulls`, `activity_groups`, `activities`, `types`, `origin`).  
  Genera:
  - el texto SQL final (`sql`)  
  - el array de parámetros (`params`) en orden de uso (`$1`, `$2`, …)  
  - un flag `expects_series` si corresponde devolver una serie mensual  
  - metadatos sobre el filtro aplicado (`meta.filter_level`, `meta.country_mode`, etc.)  

  Siempre filtra por `origin = 'NON_RESELLER'`, aplica `BETWEEN` cuando hay rango de fechas, y usa `=` o `IN (...)` según haya uno o varios valores en país o taxonomía.  
  Si el rango (o el texto) sugiere consulta mensual, agrega un `GROUP BY date_trunc('month', revenue_date)`.

- **Configuración clave:**  
  • Tabla base: `"dw"."revenue" AS t`  
  • Filtros soportados:
    - Fecha: `t.revenue_date` con `BETWEEN`, `>=` o `<=` según `date_start`/`date_end`.  
    - Países: `UPPER(TRIM(t.country_iso2))` con `=` o `IN`, y lógica especial para incluir / excluir nulos.  
    - Taxonomía: `activity_group`, `activity`, `type` en mayúsculas y trim.  
  • Serie mensual:
    - Detecta consultas tipo “last N months”, “by month”, etc., o rangos de más de un mes.  
    - Si aplica, agrega `date_trunc('month', t.revenue_date) AS period`, `GROUP BY period` y `ORDER BY period ASC`.  
  • Salida:
    - `sql`: texto SQL completo.  
    - `params`: lista de valores en el mismo orden que los placeholders `$1, $2, ...`.  
    - `expects_series`: `true` si se arma serie mensual, `false` si solo se devuelve un total.  
    - `meta.filter_level`: `"type"`, `"activity"`, `"activity_group"` o `null` según el nivel más específico usado.  
    - Reenvía `channel`, `user`, `thread_ts` para el envío posterior a Slack.

- **Código representativo del nodo (estructura principal):**

```js
// ===== RC_BuildSQL (COMPLETO) =====
// - Usa BETWEEN si vienen date_start y date_end
// - Normaliza mayúsculas/espacios en country / activity_group / activity / type
// - Usa "=" cuando hay 1 valor, "IN (...)" cuando hay varios
// - Genera serie mensual si aplica y si el rango abarca >1 mes o el texto pide "last N months"

const p = { ...$json };                      // <- ¡No borrar!
const TABLE = '"dw"."revenue" AS t';

const params = [];
const conds = [];

// helpers
function addParam(v){ params.push(v); return `$${params.length}`; }
function addList(list){ return list.map(v => addParam(v)).join(', '); }
function toUpperArr(a){ return (Array.isArray(a) ? a : []).map(x => String(x).toUpperCase()); }

// 1) ORIGIN fijo
conds.push(`t.origin = 'NON_RESELLER'`);

// 2) FECHAS
if (p.date_start && p.date_end) {
  const ph1 = addParam(p.date_start);
  const ph2 = addParam(p.date_end);
  // BETWEEN inclusivo para DATE
  conds.push(`t.revenue_date BETWEEN ${ph1}::date AND ${ph2}::date`);
} else if (p.date_start) {
  conds.push(`t.revenue_date >= ${addParam(p.date_start)}::date`);
} else if (p.date_end) {
  conds.push(`t.revenue_date <= ${addParam(p.date_end)}::date`);
}

// 3) COUNTRY (ALL | SPECIFIC | ONLY_NULL)
if (p.country_mode === 'ONLY_NULL') {
  conds.push(`COALESCE(NULLIF(TRIM(t.country), ''), '') = ''`);
} else if (p.country_mode === 'SPECIFIC' && Array.isArray(p.country_codes) && p.country_codes.length) {
  const codes = toUpperArr(p.country_codes);
  const lhs = `UPPER(NULLIF(TRIM(t.country), ''))`;
  if (codes.length === 1 && !p.include_nulls) {
    conds.push(`${lhs} = ${addParam(codes[0])}`);
  } else {
    const inList = addList(codes);
    const expr = `${lhs} IN (${inList})`;
    conds.push(p.include_nulls ? `(${expr} OR COALESCE(NULLIF(TRIM(t.country), ''), '') = '')` : expr);
  }
}

// 4) MERCHANT
if (p.merchant_id != null && p.merchant_id !== '') {
  conds.push(`t.merchant_id = ${addParam(Number(p.merchant_id))}`);
}

// 5) TAXONOMÍA (nivel más específico disponible)
const groups = Array.isArray(p.activity_groups) ? p.activity_groups : [];
const acts   = Array.isArray(p.activities) ? p.activities : [];
const types  = Array.isArray(p.types) ? p.types : [];

if (types.length) {
  const vals = toUpperArr(types);
  const lhs = `UPPER(TRIM(t.type))`;
  if (vals.length === 1) conds.push(`${lhs} = ${addParam(vals[0])}`);
  else conds.push(`${lhs} IN (${addList(vals)})`);
} else if (acts.length) {
  const vals = toUpperArr(acts);
  const lhs = `UPPER(TRIM(t.activity))`;
  if (vals.length === 1) conds.push(`${lhs} = ${addParam(vals[0])}`);
  else conds.push(`${lhs} IN (${addList(vals)})`);
} else if (groups.length) {
  const vals = toUpperArr(groups);
  const lhs = `UPPER(TRIM(t.activity_group))`;
  if (vals.length === 1) conds.push(`${lhs} = ${addParam(vals[0])}`);
  else conds.push(`${lhs} IN (${addList(vals)})`);
}

// 6) Serie mensual o total
function ym(d){ const [Y,M]=d? d.split('-').map(Number):[null,null]; return (Y!=null&&M!=null)? Y*12+(M-1) : null; }
const multiMonthRange = (p.date_start && p.date_end) ? ((ym(p.date_end)-ym(p.date_start)) >= 1) : false;
const monthlyKw = /\b(last|past)\s+\d+\s+months?\b|\bin the last year\b|\bpast year\b|\blast year\b|\bby month\b|\bmonthly\b|\bper month\b/;
const wantsMonthly = monthlyKw.test((p.text_lower||'')) || multiMonthRange;

let selectCols = [], groupBy = '', orderBy = '';
if (wantsMonthly) {
  selectCols.push(`date_trunc('month', t.revenue_date) AS period`);
  groupBy = ` GROUP BY period`;
  orderBy = ` ORDER BY period ASC`;
}

const whereSql = conds.length ? `WHERE ${conds.join(' AND ')}` : '';
const selectList = (selectCols.length ? selectCols.join(', ')+', ' : '') + `SUM(t.usd_gain) AS usd_gain_sum`;

const sql = `
  SELECT ${selectList}
  FROM ${TABLE}
  ${whereSql}
  ${groupBy}
  ${orderBy};
`.trim();

return [{
  json: {
    sql,
    params,
    expects_series: wantsMonthly,
    meta: {
      has_date_filter: !!(p.date_start || p.date_end),
      country_mode: p.country_mode,
      country_codes: p.country_codes || [],
      include_nulls: !!p.include_nulls,
      filter_level: types.length ? 'type' : acts.length ? 'activity' : groups.length ? 'activity_group' : null
    },
    channel: p.channel ?? null,
    user: p.user ?? null,
    thread_ts: p.thread_ts ?? null
  }
}];
```

---
## Nodo 10 – RC_DBQuery

- **Nombre:** `RC_DBQuery`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Ejecuta la consulta SQL generada por `RC_BuildSQL` contra el **data warehouse (Redshift/Postgres)**.  
  Usa el texto SQL (`sql`) y el array de parámetros (`params`) ya preparados, y devuelve:
  - una tabla con resultados agregados (total o serie mensual),  
  - o un array vacío si no hay datos.  

  Este nodo es donde ocurre la **extracción real de revenue** desde la base corporativa.

- **Configuración clave:**  
  • Credenciales: conexión del DW usada por Milton  
  • Operación: **Execute Query**  
  • `query`: tomado directamente del output de `RC_BuildSQL`  
  • `values`: usa `params` en el orden exacto generado por el nodo anterior  
  • Devuelve un array de filas, cada fila con sus columnas (`period`, `usd_gain_sum`, etc.)

- **Fragmento representativo del nodo (JSON):**

```json
{
  "type": "n8n-nodes-base.postgres",
  "parameters": {
    "operation": "executeQuery",
    "query": "={{ $json.sql }}",
    "additionalFields": {
      "values": "={{ $json.params }}"
    }
  },
  "credentials": {
    "postgres": {
      "name": "DW (Milton)"
    }
  }
}
```

---
## Nodo 11 – RC_Meta

- **Nombre:** `RC_Meta`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Construye un **resumen estructurado de metadatos** sobre la consulta que se va a responder.  
  Toma como entrada:
  - la query ejecutada,  
  - los parámetros aplicados (país, null-country, activity/activity group/type, merchant, fechas),  
  - indicadores de modo (serie mensual o total).  

  Su función es dejar un objeto compacto que describe *qué filtros se aplicaron realmente* para que los nodos posteriores puedan formatear correctamente la respuesta para Slack.

  En otras palabras, este nodo no modifica los datos del DW sino que **rotula y documenta** los filtros usados para que `RC_SlackFormatter` construya un mensaje claro y consistente.

- **Configuración clave:**  
  • Recibe:
    - `sql`  
    - `params`  
    - `expects_series`  
    - Filtros canonizados (country_mode, country_codes, include_nulls, activity_groups, activities, types, date_start, date_end)  
  • Genera:
    - `meta.summary` (resumen legible)  
    - `meta.filters` (estructura formal para formateo)  
    - mantiene vivos `channel`, `user`, `thread_ts`  

- **Código representativo del nodo:**

```js
// Lee SIEMPRE del nodo RC_BuildSQL (el que aún conserva channel/thread_ts del orquestador)
const src = $items("RC_BuildSQL")[0]?.json ?? {};

return [{
  json: {
    channel:   src.channel ?? null,
    thread_ts: src.thread_ts ?? src.ts ?? null,
    user:      src.user ?? null,
    ts:        src.ts ?? null,      // opcional
    text_in:   src.text ?? null     // opcional, por trazabilidad
  }
}];
```

---
## Nodo 12 – RC_SlackPrep

- **Nombre:** `RC_SlackPrep`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Prepara la **estructura base** del mensaje que se enviará a Slack.  
  No genera aún el texto final, sino que organiza:
  - los resultados del DW (`rows`),  
  - los metadatos (`meta.summary`),  
  - y el modo de presentación (`expects_series`).  

  Su objetivo es dejar un objeto limpio y consistente que el formateador final (`RC_SlackFormatter`) pueda transformar en un mensaje legible para el usuario.

  En esta etapa:
  - Convierte los resultados del query en un formato homogéneo.  
  - Asegura que el campo `rows` exista (aunque la consulta haya devuelto vacío).  
  - Propaga `channel`, `user` y `thread_ts` sin modificarlos.

- **Configuración clave:**  
  • Recibe:
    - El resultado del DW (salida de `RC_DBQuery`)  
    - Los metadatos construidos en `RC_Meta`  
  • Produce:
    - `rows`: array de resultados ya normalizados  
    - `meta`: sin cambios  
    - `expects_series`: boolean  
    - No genera texto final; sólo prepara la estructura.  

- **Código representativo del nodo:**

```js
// RC_SlackPrep: suma los resultados y arma un texto crudo para formatear en Slack

// ---- helpers ----
function toUSD(n){
  const num = Number(n) || 0;
  const fixed = (Math.round((num + Number.EPSILON) * 100) / 100).toFixed(2);
  return '$ ' + fixed.replace(/\B(?=(\d{3})+(?!\d))/g, ','); // $ 12,345.67
}
function joinOrAll(arr, fallback){
  return (Array.isArray(arr) && arr.length) ? arr.join(', ') : fallback;
}
function first(a, d=null){ return Array.isArray(a) && a.length ? a[0] : d; }

// ---- sumar resultados de RC_DBQuery ----
let total = 0;
for (const it of $items()) {
  const v = it.json?.usd_gain_sum ?? it.json?.sum ?? 0;
  total += Number(v) || 0;
}

// ---- leer metadata útil de nodos anteriores ----
const b = $items('RC_BuildSQL')[0]?.json ?? {};
const r = $items('RC_ResolveFilters')[0]?.json ?? {};

const dateStart = (b.params && b.params[0]) || r.date_start || null;
const dateEnd   = (b.params && b.params[1]) || r.date_end   || null;
const gran      = r.date_granularity || null;

const countryCodes = (b.meta && b.meta.country_codes) || [];
const countryMode  = (b.meta && b.meta.country_mode) || 'ALL';
const includeNulls = !!((b.meta && b.meta.include_nulls) || r.include_nulls);

// Si querés mostrar nombres en vez de ISO (AR→Argentina), descomenta este map:
const codeToName = {
  AR:'Argentina', BR:'Brazil', US:'United States', MX:'Mexico', CL:'Chile', UY:'Uruguay',
  CO:'Colombia', PE:'Peru', PY:'Paraguay', BO:'Bolivia', EC:'Ecuador', VE:'Venezuela',
  CA:'Canada', ES:'Spain', FR:'France', DE:'Germany', IT:'Italy', PT:'Portugal',
  GB:'United Kingdom', IE:'Ireland', NL:'Netherlands', BE:'Belgium', CH:'Switzerland',
  AT:'Austria', DK:'Denmark', NO:'Norway', SE:'Sweden', FI:'Finland', PL:'Poland',
  CZ:'Czech Republic', HU:'Hungary', RO:'Romania', RU:'Russia', UA:'Ukraine', TR:'Turkey',
  IL:'Israel', SA:'Saudi Arabia', AE:'United Arab Emirates', QA:'Qatar', KW:'Kuwait',
  ZA:'South Africa', NG:'Nigeria', KE:'Kenya', EG:'Egypt', MA:'Morocco', DZ:'Algeria', TN:'Tunisia',
  CN:'China', JP:'Japan', KR:'South Korea', TH:'Thailand', VN:'Vietnam', ID:'Indonesia',
  PH:'Philippines', MY:'Malaysia', SG:'Singapore', HK:'Hong Kong', TW:'Taiwan',
  AU:'Australia', NZ:'New Zealand'
};
const countryLabel =
  countryMode === 'ONLY_NULL'
    ? 'Only rows with country = null'
    : (countryCodes.length
        ? countryCodes.map(c => codeToName[c] || c).join(', ')
        : 'All countries');

// filtros de taxonomía: mostrar el nivel más específico presente
const types      = Array.isArray(r.types) ? r.types : [];
const activities = Array.isArray(r.activities) ? r.activities : [];
const groups     = Array.isArray(r.activity_groups) ? r.activity_groups : [];

let filterLabel = 'None';
if (types.length) {
  filterLabel = `type = ${types.join(', ')}`;
} else if (activities.length) {
  filterLabel = `activity = ${activities.join(', ')}`;
} else if (groups.length) {
  filterLabel = `activity_group = ${groups.join(', ')}`;
}

const merchant = r.merchant_id ?? null;

// construir rangos/etiquetas
const dateRange =
  (dateStart && dateEnd) ? `${dateStart} → ${dateEnd}` :
  dateStart ? `from ${dateStart}` :
  dateEnd   ? `until ${dateEnd}` :
  'All dates';

const granLabel = gran ? (gran === 'month' ? 'monthly' : 'daily') : 'auto';

// total formateado
const totalFmt = toUSD(total);

// texto crudo para el agente de formato (NO uses markdown avanzado aquí)
const raw =
`Revenue result
Total (USD): ${totalFmt}

Parameters:
- Date range: ${dateRange}  (${granLabel})
- Countries: ${countryLabel}${includeNulls ? '  (including country = null)' : ''}
- Merchant: ${merchant ?? 'All'}
- Filters: ${filterLabel}
- Origin: NON_RESELLER`;

return [{
  json: {
    output: {
      response_to_user: raw   // <- así lo toma tu "Slack formatter" ({{$json.output.response_to_user}})
    },
    total_usd: total,
    total_usd_formatted: totalFmt
  }
}];
```

---
## Nodo 13 – RC_SlackFormatter

- **Nombre:** `RC_SlackFormatter`  
- **Tipo:** AI Agent (`@n8n/n8n-nodes-langchain.agent`)  
- **Qué hace:**  
  Usa un modelo de lenguaje como **formateador para Slack**.  
  Toma el texto ya armado para el usuario (`output.response_to_user`) y lo transforma en un mensaje con buen formato para Slack (negritas, listas, bloques de código, links clickeables), sin cambiar el contenido ni los números, solo la presentación.

- **Configuración clave:**  
  • `promptType`: `define` (prompt definido a mano).  
  • `text`: expresión que pasa el mensaje bruto del flujo:  
    ```js
    ={{ $json.output.response_to_user }}
    ```  
  • `options.systemMessage`: prompt del agente que le indica al modelo:  
    - que es un “message formatter for Slack”,  
    - que debe aplicar formato Slack (listas con `•` o `-`, `*bold*`, backticks para código, etc.),  
    - que **no debe alterar las URLs** y que, si hay links, los devuelva en formato `<URL|View PDF>`,  
    - que no cambie el contenido, solo el formato.  

  El resultado de este nodo es un texto ya listo para ser enviado a Slack por el nodo final del flujo.

```text
System Prompt:

You are a message formatter for Slack.

Your task is to take a plain response and convert it into a clean, readable Slack message using appropriate formatting.

⚠️ VERY IMPORTANT:
- Do NOT alter or re-encode URLs. 
- URLs must be returned exactly as received, character-for-character, with no additions, removals, or replacements.
- If there is a URL, always wrap it in Slack's link syntax: `<URL|View PDF>` so it is clickable.


Formatting rules:
- Use `•` or `-` for lists, each item on a new line.
- Use `*bold*` for emphasis (e.g., section titles).
- Use backticks (`) for inline code.
- Use triple backticks (```) for code blocks.
- Use `>` for blockquotes (e.g., highlighting instructions or notes).
- Do not use Markdown-specific syntax like headings (##), or HTML tags.
- Keep it concise and clean — Slack is for fast reading.

Return only the formatted text.

Examples:

Example with a URL:
Input:
Here is the document you requested:
https://s3.amazonaws.com/bucket/file.pdf?param=abc&token=123

Output:
*Here is the document you requested:*  
<https://s3.amazonaws.com/bucket/file.pdf?param=abc&token=123|View PDF>

Input:
"""
Here are the required steps:
1. Login
2. Verify your identity
3. Make a deposit
"""

Output:
*Here are the required steps:*
• Login  
• Verify your identity  
• Make a deposit

---

Input:
"""
Use the following code:
print("Hello, world!")
"""

Output:
*Use the following code:*
print("Hello, world!")

---

Now format this message for Slack:

{{RAW_MESSAGE_HERE}}
```

---
## Nodo 14 – RC_Merge

- **Nombre:** `RC_Merge`  
- **Tipo:** Merge (`n8n-nodes-base.merge`)  
- **Qué hace:**  
  Fusiona dos flujos paralelos del proceso de formateo antes del envío final a Slack.  
  Concretamente combina:
  - el mensaje ya formateado por el nodo **RC_SlackFormatter**, y  
  - los metadatos esenciales provenientes del flujo principal (canal, usuario, thread_ts).  

  El objetivo es volver a unir texto + metadatos en un solo item, necesarios para que el siguiente nodo (`RC_SendMessage`) pueda enviar correctamente la respuesta.

- **Configuración clave:**  
  • Modo de merge: **Pass-through** o **Append** (según el JSON), pero siempre unifica ambos items en un único output.  
  • Garantiza que el campo `text` (ya formateado para Slack) quede acompañado de:  
    - `channel`  
    - `user`  
    - `thread_ts`  
  • No altera el contenido del mensaje; solo consolida.

---
## Nodo 15 – RC_SlackText

- **Nombre:** `RC_SlackText`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Prepara el **texto final** que va a ser formateado por el agente en `RC_SlackFormatter`.  
  Este nodo toma la estructura generada por `RC_SlackPrep` (que contiene los datos numéricos y metadatos) y construye un campo estandarizado llamado `output.response_to_user`, que es el mensaje en lenguaje natural **antes del formateo**.

  En otras palabras:  
  **RC_SlackPrep → RC_SlackText (mensaje sin formato) → RC_SlackFormatter (mensaje formateado)**

- **Configuración clave:**  
  • Usa los resultados (`rows`) y el modo (`expects_series`) para armar un texto base.  
  • No aplica estilo de Slack todavía, solo genera el contenido.  
  • Devuelve un objeto con:
    - `output.response_to_user` (mensaje bruto)  
    - los metadatos necesarios para el merge posterior  
    - `channel`, `thread_ts`, `user`

- **Código representativo del nodo:**

```js
let text = "";
if (typeof $json.output === "string") {
  text = $json.output;
} else if ($json.output != null) {
  text = JSON.stringify($json.output);
}

// Pasa los metadatos TAL CUAL, sin re-buscarlos.
return [{
  json: {
    ...$json,      // conserva channel, thread_ts, etc.
    text          // el mensaje final para Slack
  }
}];
```

---
## Nodo 16 – RC_SendMessage

- **Nombre:** `RC_SendMessage`  
- **Tipo:** Slack → Send Message (`n8n-nodes-base.slack`)  
- **Qué hace:**  
  Envía el mensaje final de revenue—ya formateado por `RC_SlackFormatter` y consolidado por `RC_Merge`—al **mismo canal y mismo hilo de Slack** donde el usuario hizo la consulta.  
  Este nodo es el paso final del flujo y es el que efectivamente publica la respuesta visible para el usuario.

- **Configuración clave:**  
  • Operación: **Message → Send → Simple Text Message**  
  • `text`: se toma del mensaje formateado (`={{ $json.text }}`)  
  • `channelId`: `={{ $json.channel }}` (para responder en el mismo canal)  
  • `thread_ts`: `={{ $json.thread_ts }}` (para continuar en el mismo hilo)  
  • “Use Markdown”: activado para que Slack renderice correctamente el formato (negritas, listas, etc.)  
  • No incluye adjuntos ni bloques interactivos, manteniendo compatibilidad amplia con cualquier cliente de Slack.

- **Fragmento representativo (JSON):**

```json
{
  "type": "n8n-nodes-base.slack",
  "parameters": {
    "operation": "send",
    "text": "={{ $json.text }}",
    "channelId": "={{ $json.channel }}",
    "thread_ts": {
      "replyValues": {
        "thread_ts": "={{ $json.thread_ts }}"
      }
    },
    "useMarkdown": true
  },
  "credentials": {
    "slackApi": {
      "name": "Milton Account"
    }
  }
}
```
---

