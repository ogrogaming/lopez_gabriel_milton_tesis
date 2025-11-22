# Flujo de Ingresos (WF_RevenueConsult_Sub)

Este sub-workflow procesa las consultas de revenue que el orquestador le deriva a Milton. Recibe el texto de la pregunta y los metadatos de Slack, normaliza el mensaje, extrae filtros (fechas, países, taxonomía, merchant), construye y ejecuta una consulta SQL sobre el data warehouse y finalmente devuelve una respuesta resumida y formateada para Slack. :contentReference[oaicite:0]{index=0}  

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
```

---
  }
}
