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
  
}
