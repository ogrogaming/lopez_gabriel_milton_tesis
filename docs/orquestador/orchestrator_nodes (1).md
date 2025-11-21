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

---
