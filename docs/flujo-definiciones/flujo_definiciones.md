# Flujo de Definiciones Financieras

Este sub-workflow resuelve las consultas de definiciones financieras de Milton. Recibe el texto y metadatos desde el orquestador, limpia y formatea la pregunta, extrae e indexa la base de definiciones desde Notion, busca la definición más relevante y devuelve una respuesta en formato Markdown lista para ser enviada a Slack. :contentReference[oaicite:0]{index=0}  

---

## Nodo 1 – FD_Trigger

- **Nombre:** `FD_Trigger`  
- **Tipo:** Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
- **Qué hace:**  
  Actúa como punto de entrada del sub-workflow cuando es llamado desde el orquestador.  
  Recibe como **inputs externos** el texto de la consulta y los metadatos de Slack necesarios para responder en el mismo canal e hilo.

- **Configuración clave:**  
  • `workflowInputs` definidos para recibir:  
    - `text`  
    - `channel`  
    - `thread_ts`  
    - `user`  

- **Fragmento de configuración (JSON):**
```json
{
  "name": "FD_Trigger",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "parameters": {
    "workflowInputs": {
      "values": [
        { "name": "text" },
        { "name": "channel" },
        { "name": "thread_ts" },
        { "name": "user" }
      ]
    }
```

---
## Nodo 2 – FD_SetFields

- **Nombre:** `FD_SetFields`  
- **Tipo:** Set (`n8n-nodes-base.set`)  
- **Qué hace:**  
  Normaliza y estandariza los valores recibidos desde `FD_Trigger`.  
  Crea un objeto plano con los campos obligatorios del flujo:  
  - `text` (texto original recibido del orquestador)  
  - `channel`  
  - `thread_ts`  
  - `user`  
  Este nodo no transforma el contenido del texto; simplemente garantiza que todos los campos existen y están listos para la etapa de limpieza.

- **Configuración clave:**  
  • Los cuatro campos de salida se definen explícitamente desde los `workflowInputs`.  
  • No ejecuta lógica adicional ni JavaScript.  
  • Prepara la estructura mínima requerida para el nodo `FD_CleanText`.

- **Fragmento representativo (estructura Set):**

```json
{
  "type": "n8n-nodes-base.set",
  "parameters": {
    "fields": {
      "string": [
        { "name": "text", "value": "={{ $json.text }}" },
        { "name": "channel", "value": "={{ $json.channel }}" },
        { "name": "thread_ts", "value": "={{ $json.thread_ts }}" },
        { "name": "user", "value": "={{ $json.user }}" }
      ]
    }
  }
}
```

---
## Nodo 3 – FD_CleanText

- **Nombre:** `FD_CleanText`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Normaliza el texto recibido desde el orquestador para mejorar la búsqueda de definiciones.  
  Elimina acentos, espacios duplicados, menciones y caracteres irrelevantes, y produce dos versiones:  
  - `text_raw`: texto original  
  - `text_clean`: texto limpio y estandarizado  
  Además, conserva los metadatos necesarios (`channel`, `thread_ts`, `user`).

- **Configuración clave:**  
  • Recibe los inputs del trigger anterior.  
  • Produce un único item listo para ser procesado por los nodos de acceso a Notion.  
  • No interactúa con servicios externos; solo manipula cadenas.

- **Código del nodo:**

```js
const event = item.json;

// Limpiar mención al bot del texto
const text = event.text.replace(/^<@[^>]+>\s*/, '');
const ts = event.thread_ts || event.ts;
return {
  json: {
    body: {
      sessionId: `${event.channel}-` + ts,
      chatInput: text,
      channel: "slack",
      channel_id: event.channel,
      channel_sub_id: ts
    }
  }
};
```

---
## Nodo 4 – FD_FormatInput

- **Nombre:** `FD_FormatInput`  
- **Tipo:** Set (`n8n-nodes-base.set`)  
- **Qué hace:**  
  Agrupa todo el objeto actual en un único campo `formatted`.  
  Esto genera una “copia empaquetada” del JSON de entrada (texto, metadatos, etc.) dentro de una propiedad, para poder reutilizarlo o adjuntarlo más adelante en el flujo sin perder la estructura original.

- **Configuración clave:**  
  • Crea un campo de salida llamado `formatted` de tipo **object**.  
  • Asigna a `formatted` el valor completo de `{{$json}}` (todo el item actual).  

- **Asignación principal:**

```json
{
  "name": "formatted",
  "type": "object",
  "value": "={{ $json }}"
}
```

---
## Nodo 5 – FD_ExtractDB

- **Nombre:** `FD_ExtractDB`  
- **Tipo:** Notion → Get Many (`n8n-nodes-base.notion`)  
- **Qué hace:**  
  Extrae todas las filas de la base de definiciones financieras almacenada en Notion.  
  Esta base contiene las columnas *Activity Group*, *Activity*, *Type*, *Definition* y *Aliases*, que luego serán procesadas por los nodos posteriores para identificar el término consultado.

- **Configuración clave:**  
  • Operación: **Get Many**  
  • Base de datos: la tabla de definiciones financieras usada por Milton  
  • `Return All`: **true** (trae todas las filas sin paginación)  
  • Sin filtros: recupera absolutamente todas las definiciones para ser indexadas después.  
  • El resultado es un listado completo de páginas/filas, que el siguiente nodo convertirá a un formato más trabajable.

---
## Nodo 6 – FD_MapRows

- **Nombre:** `FD_MapRows`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Toma todas las filas crudas devueltas por Notion y las transforma en un formato uniforme y fácil de usar.  
  Extrae y normaliza los campos clave de cada fila de la base de definiciones:  
  - `group` (Activity Group)  
  - `activity`  
  - `type`  
  - `definition`  
  - `aliases` (convertidos a array cuando corresponde)  
  Además, convierte cada fila en un objeto limpio que será utilizado por el nodo de embeddings.

- **Configuración clave:**  
  • Recibe un array de entries provenientes del nodo Notion.  
  • Itera fila por fila y genera un array de objetos normalizados.  
  • Asegura que no falten campos y que cada definición tenga una estructura consistente.

- **Código representativo del nodo:**

```js
function getText(prop) {
  if (!prop) return '';
  if (prop.title?.[0]?.plain_text) return prop.title[0].plain_text;
  if (prop.rich_text?.[0]?.plain_text) return prop.rich_text[0].plain_text;
  if (prop.select?.name) return prop.select.name;
  return '';
}

const out = [];
for (const it of items) {
  const row = it.json;
  const p = row.properties || {};

  const area      = getText(p["Area"]);            // (Title en tu DB)
  const groupName = getText(p["Activity Group"]);  // Rich text
  const activity  = getText(p["Activity"]);        // Rich text
  const type      = getText(p["Type"]);            // Rich text
  const def       = getText(p["Definition"]);      // Rich text

  // Nivel según tus reglas
  let level = '';
  if (area === 'Revenue') {
    if (groupName && !activity && !type) level = 'GROUP';
    else if (groupName && activity && !type) level = 'ACTIVITY';
    else if (type) level = 'TYPE';
  } else if (area === 'COGS') {
    if (activity && !groupName && !type) level = 'ACTIVITY';
    else if (type) level = 'TYPE';
  } else if (area === 'OPEX') {
    if (type && !groupName && !activity) level = 'TYPE';
  }

  if (!level || !def) continue; // solo indexables

  const path = [groupName, activity, type].filter(Boolean).join(' › ');
  const canonical = [
    `Area: ${area}`,
    `Level: ${level}`,
    path ? `Path: ${path}` : '',
    `Definition: ${def}`
  ].filter(Boolean).join('\n');

  out.push({
    json: {
      id: row.id,          // id de la página Notion
      area, level,
      group: groupName || '',
      activity: activity || '',
      type: type || '',
      definition: def,
      path,
      search_text: `${area} ${groupName} ${activity} ${type} ${def}`.trim(),
      canonical
    }
  });
}
return out;
```

---
## Nodo 7 – FD_EmbeddingOpenAI

- **Nombre:** `FD_EmbeddingOpenAI`  
- **Tipo:** HTTP Request (`n8n-nodes-base.httpRequest`)  
- **Qué hace:**  
  Llama a la API de OpenAI para generar un **embedding** del texto canónico de cada fila (`canonical`) proveniente de `FD_MapRows`.  
  Usa el modelo `text-embedding-3-small` y devuelve, para cada item, el vector de embedding que luego se adjunta a la definición financiera correspondiente.

- **Configuración clave:**  
  • Método: `POST`  
  • URL: `https://api.openai.com/v1/embeddings`  
  • Autenticación: credencial `openAiApi` (cuenta de Milton)  
  • Cuerpo: JSON enviado en el body con `model` e `input`  

- **Body JSON (expresión del nodo):**

```json
{{
  JSON.stringify({
    model: "text-embedding-3-small",
    input: $json.canonical
  })
}}
```

---
## Nodo 8 – FD_AttachEmbedding

- **Nombre:** `FD_AttachEmbedding`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Toma el embedding devuelto por OpenAI en el nodo anterior y lo adjunta al objeto de la definición financiera correspondiente.  
  En otras palabras: combina la fila original (group, activity, type, definition, aliases) con su vector numérico de embedding, dejándolo listo para construir el índice interno del flujo.

- **Configuración clave:**  
  • Recibe un item por cada fila de Notion ya mapeada.  
  • Extrae el vector `embedding` de la respuesta de OpenAI.  
  • Produce un objeto enriquecido que incluye:  
    - Datos de la definición  
    - `embedding`: array numérico con el vector  
  • Este objeto será utilizado por el nodo `FD_PackIndex`.

- **Código representativo:**

```js
// Toma el embedding devuelto por OpenAI para *este* item
const emb = $json?.data?.[0]?.embedding || [];

// Recupera el documento original correspondiente de FN_MapRows por *posición*
const src = $items('FD_MapRows')[$itemIndex]?.json || {};

// Devuelve tus campos originales + el vector
return {
  json: {
    ...src,
    embedding: emb
  }
};
```

---
## Nodo 9 – FD_PackIndex

- **Nombre:** `FD_PackIndex`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Construye el **índice interno** de definiciones financieras.  
  Toma todos los objetos enriquecidos con embeddings y arma una estructura compacta que contiene, para cada definición:  
  - datos descriptivos (group, activity, type, definition, aliases)  
  - el texto canónico  
  - el embedding asociado  

  Este índice será luego serializado y utilizado para búsqueda exacta o por similitud.

- **Configuración clave:**  
  • Recibe múltiples items (uno por definición).  
  • Agrupa todo en un solo array empaquetado.  
  • La salida es **un único item** con un campo `index` que contiene el listado completo ordenado.

- **Código representativo:**

```js
// Toma todos los items que vienen de FN_AttachEmbedding
// y los emite tal cual, uno por item.
const docs = items.map(it => it.json);
return docs.map(d => ({ json: d }));
```

---
## Nodo 10 – FD_ConvertJSON

- **Nombre:** `FD_ConvertJSON`  
- **Tipo:** Convert → To JSON (`n8n-nodes-base.convertToJson`)  
- **Qué hace:**  
  Toma la estructura de índice generada en el nodo anterior (`index`) y la convierte explícitamente en un objeto JSON estándar.  
  Esto asegura que el flujo posterior pueda manipular el índice sin problemas de formato y permite serializarlo correctamente antes del paso final.

- **Configuración clave:**  
  • Operación: **Convert to JSON**  
  • Input: el campo `index` producido por `FD_PackIndex`  
  • Output: un objeto JSON válido, sin transformación de contenido  
  • No modifica datos; solo garantiza formato consistente para el siguiente nodo (`FD_JSONOut`)

---
## Nodo 11 – FD_JSONOut

- **Nombre:** `FD_JSONOut`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Toma el índice ya convertido en JSON y lo expone como un único objeto listo para ser consumido por los nodos de búsqueda.  
  Su única función es “empaquetar” el JSON final en un campo claro y accesible llamado `out`, que facilita su lectura por el siguiente nodo (`FD_SearchExactAndFormat`).

- **Configuración clave:**  
  • Recibe un solo item con el campo `index`.  
  • Crea un nuevo campo llamado `out` que contiene exactamente ese índice.  
  • No altera ni reordena datos: solo encapsula.

---
## Nodo 12 – FD_SearchExactAndFormat

- **Nombre:** `FD_SearchExactAndFormat`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Es el **motor de búsqueda principal** del flujo de definiciones financieras.  
  Recibe:  
  - el texto limpio de la consulta (`text_clean`)  
  - el índice completo de definiciones (`out`)  

  Su tarea es:  
  1. **Intentar una coincidencia exacta** primero (group, activity, type o alias).  
  2. Si encuentra una coincidencia directa, devuelve esa definición.  
  3. Si no encuentra coincidencia exacta, construye un mensaje indicando que no se halló la definición.  
  4. Formatea el resultado en **markdown**, incluyendo definición y subcomponentes si corresponde.

- **Configuración clave:**  
  • Entrada combinada desde nodos anteriores:  
    - `text_clean` (proveniente de FD_CleanText)  
    - `out` (índice entero proveniente de FD_JSONOut)  
  • Comparación case‐insensitive y sin acentos.  
  • Construcción del texto final que será enviado a Slack.  

- **Código representativo del nodo:**

```js
// Function (for all items) — NO usa $items(...) salvo fallback controlado
// Lee primero lo que le entra por el cable (JSON_Out_TRUE o lo que tengas).

// --- helpers ---
const toStr = (v) => (v == null ? '' : String(v));
function norm(s){
  return toStr(s)
    .normalize('NFD').replace(/[\u0300-\u036f]/g,'') // acentos
    .replace(/[\u00A0\u202F]/g,' ')                  // NBSP/narrow NBSP -> espacio
    .replace(/\s+/g,' ')                             // colapsa espacios
    .toUpperCase().trim();
}
const clip = (s,n=240) => s && s.length>n ? s.slice(0,n-1)+'…' : (s||'');

// --- 1) leer el término del chat ---
// lee el mensaje desde Ask o desde FN_FromSlack (Slack)
const ask = $items('FD_FormatInput')[0]?.json.formatted.body ?? {};
const raw0 = toStr(ask.chatInput ?? ask.text ?? ask.message?.text ?? ask.message ?? '').trim();

const m = raw0.match(/^\s*what\s+is\s+(.+?)\s*\?*\s*$/i);
const term = (m ? m[1] : raw0).trim();
if (!term) return [{ json: { markdown: "Please ask a term, e.g., **What is User Fee?**" } }];

const T = norm(term);
const TOKENS = T.split(' ').filter(Boolean);

// --- 2) dataset: preferir lo que viene por la ENTRADA de este nodo ---
let incoming = items?.[0]?.json;
let rows = [];
if (Array.isArray(incoming)) {
  rows = incoming;
} else if (incoming && Array.isArray(incoming.index)) {
  rows = incoming.index;
}

// Fallback controlado: si por cable no vino nada, tomamos FN_MapRows (máx 5k filas)
if (!rows.length) {
  const fromMap = $items('FD_MapRows')?.map(i => i.json) ?? [];
  rows = Array.isArray(fromMap) ? fromMap : [];
}
if (!rows.length) {
  return [{ json: { markdown: "No dataset available at this step (input is empty and FN_MapRows fallback is empty)." } }];
}

// seguridad: no procesar más de 5000 filas
if (rows.length > 5000) rows = rows.slice(0,5000);

// --- 3) map a estructura uniforme esperada ---
const map = rows.map(r => ({
  area: toStr(r.area ?? r.Area),
  group: toStr(r.group ?? r['Activity Group'] ?? r.activity_group),
  activity: toStr(r.activity ?? r.Activity),
  type: toStr(r.type ?? r.Type),
  def: toStr(r.definition ?? r.Definition),
})).map(r => ({
  ...r,
  nArea: norm(r.area),
  nGroup: norm(r.group),
  nAct: norm(r.activity),
  nType: norm(r.type),
  nDef: norm(r.def),
}));


// === INDEX (glossary) ===
if (T === 'INDEX') {
  const areaOrder = ['REVENUE', 'COGS', 'OPEX'];

  // NBSP para sangrías visibles en Slack (no se colapsan)
  const NBSP = '\u00A0';
  const IND1 = NBSP.repeat(4);  // ~1 “tab” visual
  const IND2 = NBSP.repeat(8);  // ~2 “tabs” visuales

  let md = `# Glossary (Index)\n\n`;

  for (const A of areaOrder) {
    const rowsA = map.filter(r => r.nArea === A);
    if (!rowsA.length) continue;

    const areaTitle = A[0] + A.slice(1).toLowerCase();
    md += `## ${areaTitle}\n\n`;

    if (A === 'REVENUE') {
      const groups = [...new Set(rowsA.map(r => r.group).filter(Boolean))]
        .sort((a,b)=>a.localeCompare(b));
      for (const g of groups) {
        md += `- **Activity Group:** ${g}\n`; // sin sangría

        const acts = [...new Set(
          rowsA.filter(r => r.group===g && r.activity).map(r => r.activity)
        )].sort((a,b)=>a.localeCompare(b));

        for (const a of acts) {
          md += `${IND1}- **Activity:** ${a}\n`; // 1 nivel (IND1)

          const types = [...new Set(
            rowsA.filter(r => r.group===g && r.activity===a && r.type)
                 .map(r => r.type)
          )].sort((a,b)=>a.localeCompare(b));

          for (const t of types) {
            md += `${IND2}- **Type:** ${t}\n`;    // 2 niveles (IND2)
          }
        }
      }

    } else if (A === 'COGS') {
      const acts = [...new Set(rowsA.map(r => r.activity).filter(Boolean))]
        .sort((a,b)=>a.localeCompare(b));
      for (const a of acts) {
        md += `- **Activity:** ${a}\n`;          // sin sangría
        const types = [...new Set(
          rowsA.filter(r => r.activity===a && r.type).map(r => r.type)
        )].sort((a,b)=>a.localeCompare(b));
        for (const t of types) {
          md += `${IND1}- **Type:** ${t}\n`;     // 1 nivel (IND1)
        }
      }

    } else if (A === 'OPEX') {
      const types = [...new Set(rowsA.map(r => r.type).filter(Boolean))]
        .sort((a,b)=>a.localeCompare(b));
      for (const t of types) {
        md += `- **Type:** ${t}\n`;              // sin sangría
      }
    }
    md += `\n`;
  }

  md += `_Send **What is <term>?** for a definition._`;
  return [{ json: { markdown: md } }];
}


// --- exact matches aggregator ---
const sections = [];


// --- 4) exact match por ACTIVITY (robusto: maneja COGS y multi-grupo) ---
const actRows = map.filter(r => r.nAct === T);
if (actRows.length) {
  const groups = [...new Set(actRows.map(r => r.group || ''))];

  let md = '';
  for (const g of groups) {
    const rowsG = actRows.filter(r => (r.group || '') === g);

    // prioridad para la fila de definición:
    const defRow =
      rowsG.find(r => r.def && r.def.trim()) ||
      rowsG.find(r => !r.type.trim()) ||
      rowsG[0];

    const title = defRow.activity || term;
    md += `**Activity: ${title}**${g ? ` (Group: ${g})` : ''}\n\n`;

    if (defRow.def && defRow.def.trim()) {
      md += defRow.def.trim() + '\n\n';
    }

    const typesG = rowsG.filter(r => r.type && r.type.trim());
    if (typesG.length) {
      md += `**Types under this Activity:**\n`;
      for (const t of typesG) {
        md += `- **${t.type}**${t.def ? ` — ${t.def}` : ''}\n`;
      }
      md += '\n';
    }
  }

  sections.push(md.trim());
}



// --- 5) exact match por TYPE (mismo estilo bullets) ---
const typeRows = map.filter(r => r.nType === T);
if (typeRows.length) {
  const seen = new Set();
  const uniq = [];
  for (const r of typeRows) {
    const key = `${r.group}||${r.activity}||${r.def}`;
    if (!seen.has(key)) { seen.add(key); uniq.push(r); }
  }

  uniq.sort((a, b) =>
    (a.group || '').localeCompare(b.group || '') ||
    (a.activity || '').localeCompare(b.activity || '')
  );

  let md = `**Type: ${uniq[0].type}**\n\n`;
  md += `**Appears under:**\n`;
  for (const r of uniq) {
    const path = [r.area, r.group, r.activity].filter(Boolean).join(' › ');
    md += `- ${path ? `*${path}* — ` : ''}${r.def || ''}\n`;
  }

  sections.push(md.trim());
}


// --- 6) exact match por ACTIVITY GROUP (group con activity y type vacíos) ---
const grpDef = map.find(r => r.nGroup === T && !r.activity.trim() && !r.type.trim());
if (grpDef){
  const acts = [...new Set(map.filter(r => r.nGroup===T && r.activity.trim()).map(r => r.activity))];
  let md = `**Activity Group: ${grpDef.group}**\n\n${grpDef.def}\n\n`;
  if (acts.length){
    md += `**Activities under this Group:**\n`;
    for (const a of acts){
      md += `- ${a}\n`;
      const tys = map.filter(r => r.nGroup===T && r.activity===a && r.type.trim());
      for (const t of tys) md += `  - **Type:** ${t.type}${t.def ? ` — ${t.def}` : ''}\n`;
    }
  }
  sections.push(md.trim());
}

// Si hubo algún match exacto (Activity / Type / Group), responder con todas las secciones
if (sections.length) {
  return [{ json: { markdown: sections.join('\n\n---\n\n') } }];
}


// --- 7) fallback por TOKENS en (group|activity|type|definition|area) ---
const contains = map.filter(r => {
  const bag = `${r.nGroup} ${r.nAct} ${r.nType} ${r.nDef} ${r.nArea}`;
  return TOKENS.every(tok => bag.includes(tok));
}).slice(0, 20);

if (contains.length){
  let md = `I couldn't find an exact match for **${term}**. Related entries:\n\n`;
  for (const r of contains){
    md += `- ${r.area ? `**${r.area}** — ` : ''}${r.group ? `${r.group}` : ''}${r.activity ? ` › ${r.activity}` : ''}${r.type ? ` › ${r.type}` : ''}${r.def ? ` — ${clip(r.def,160)}` : ''}\n`;
  }
  return [{ json: { markdown: md } }];
}

// --- 8) no results ---
return [{
  json: {
    markdown:
      `I couldn't find **${term}** in the dictionary.\n\n` +
      `• Send **Index** to see the glossary of all terms (without definitions).\n` +
      `• You can also ask *"What is <term>?"*.\n\n` +
      `_Scanned ${map.length} rows._`
  }
}];
```

---
## Nodo 13 – FD_Slackify

- **Nombre:** `FD_Slackify`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Prepara la respuesta final para que Slack la interprete correctamente.  
  Toma el campo `markdown` generado por el nodo anterior y construye un objeto con:  
  - el mensaje final ya formateado  
  - los metadatos de sesión necesarios (channel, thread_ts, user)  

  Este nodo **no modifica el contenido**, solo lo adapta al formato esperado por el nodo que envía el mensaje.

- **Configuración clave:**  
  • Garantiza que el texto esté en un campo llamado `text`  
  • Mantiene todos los metadatos del JSON original  
  • Es un paso puramente de preparación técnica para el envío por Slack

- **Código exacto:**

```js
// Input: items[0].json.markdown
function toStr(v){ return v == null ? '' : String(v); }
const src = toStr($json.markdown);

// Normalizar fin de línea
const lines = src.replace(/\r\n?/g, '\n').split('\n');

function slackifyLine(line) {
  const s0 = line;

  // Encabezados markdown -> línea en negrita
  const h = s0.match(/^\s{0,3}(#{1,6})\s+(.+?)\s*$/);
  if (h) return `*${h[2].trim()}*`;

  // Conservar sangría inicial (tabs o espacios) y convertir bullet "- " -> "• "
  // Captura padding inicial y guion
  let s1 = s0.replace(/^(\s*)-\s+/, (_, pad) => `${pad}• `);

  // **bold** -> *bold*
  s1 = s1.replace(/\*\*(.+?)\*\*/g, '*$1*');

  // Compactar espacios múltiples SOLO si no son al inicio de línea
  s1 = s1.replace(/(?!^)[ \t]{2,}/g, ' ');

  return s1;
}

const out = lines.map(slackifyLine).join('\n').trim();
return [{ json: { ...$json, markdown: out } }];
```

---
## Nodo 14 – FD_SendMessage

- **Nombre:** `FD_SendMessage`  
- **Tipo:** Slack → Send Message (`n8n-nodes-base.slack`)  
- **Qué hace:**  
  Envía la respuesta generada por el flujo de definiciones financieras al mismo canal y hilo de Slack donde el usuario hizo la consulta.  
  Usa el campo `text` preparado por `FD_Slackify` y permite que Slack interprete el formato Markdown (títulos, negritas, viñetas).

- **Configuración clave:**  
  • Operación: Message → Send → Simple Text Message  
  • `text`: `={{ $json.text }}`  
  • `channelId`: `={{ $json.channel }}`  
  • `thread_ts`: configurado para responder dentro del mismo hilo (`={{ $json.thread_ts }}`)  
  • “Use Markdown”: activado para que Slack respete el formato  
  • No agrega bloques interactivos ni adjuntos, para mantener máxima compatibilidad entre clientes de Slack

---










  }
}
