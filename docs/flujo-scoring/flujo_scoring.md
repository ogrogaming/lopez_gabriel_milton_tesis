# Flujo de Fraud Scoring

Este sub-flujo recibe una consulta desde el orquestador pidiendo el score de fraude para uno o varios usuarios (por ejemplo: 
“Provide fraud score for user id 20761572”), extrae los IDs, arma el perfil transaccional y de KYC desde el DW, calcula un riesgo 
numérico y devuelve un resumen legible en Slack con el score y las principales razones.
---

## Nodo 1 – FS_Trigger

- **Nombre:** `FS_Trigger`  
- **Tipo:** Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
- **Qué hace:**  
  Actúa como punto de entrada del sub-flujo de fraud scoring. Recibe desde el orquestador el texto de la consulta y los metadatos de Slack, y los expone como inputs internos del workflow.

- **Configuración clave:**  
  - `workflowInputs` definidos:
    - `text` → texto completo de la consulta (ej. “Provide fraud score for user 20761572”)  
    - `channel` → ID del canal de Slack de origen  
    - `thread_ts` → timestamp del hilo donde responder  
    - `user` → ID del usuario de Slack que hizo la pregunta  
  - No tiene lógica propia; solo normaliza estos campos para que el siguiente nodo pueda trabajar con ellos.

---
## Nodo 2 – FS_ExtractUserID

- **Nombre:** `FS_ExtractUserID`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Extrae uno o varios **user_id** desde el texto original recibido en `FS_Trigger`.  
  Detecta números dentro del mensaje (por ejemplo “user 20761572”, “check 9841223”, “id 4411”) y construye un array limpio de IDs numéricos para usar en la query posterior.  
  Si no se encuentra ningún ID, deja una lista vacía para que los siguientes nodos puedan manejar el caso correctamente.

- **Configuración clave:**  
  • Recibe `text` desde FS_Trigger.  
  • Aplica una expresión regular para capturar secuencias numéricas.  
  • Devuelve:
    - `user_ids`: array de enteros  
    - metadatos (`channel`, `thread_ts`, `user`) sin cambios  

- **Código representativo del nodo:**

```js
const text = $json.text || "";
const match = text.match(/user\s*(?:id\s*)?(\d+)/i);

return [{
  user_id: match ? match[1] : null,
  text,
  channel: $json.channel,
  thread_ts: $json.thread_ts,
  user: $json.user
}];
```

---
## Nodo 3 – FS_ParseUserID

- **Nombre:** `FS_ParseUserID`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Valida y estructura los IDs detectados en el nodo anterior (`FS_ExtractUserID`).  
  Garantiza que:
  - exista al menos **un** user_id válido,  
  - todos los IDs sean numéricos y positivos,  
  - el flujo siga siempre con una estructura uniforme.  

  Si no se detecta ningún ID, prepara un objeto seguro que permite que los nodos posteriores devuelvan un mensaje de error claro (“No valid user ID provided”).

- **Configuración clave:**  
  • Recibe un array `user_ids` desde `FS_ExtractUserID`.  
  • Limpia, ordena y deduplica la lista.  
  • Produce:
    - `parsed_user_ids`: array depurado  
    - `has_ids`: booleano  
    - mantiene metadatos (`channel`, `thread_ts`, `user`) para Slack  

- **Código representativo del nodo:**

```js
// Match after "user" or "userid"
const regex = /\b(?:users?|user[_\s]?id)\s+([\d,\s]+)/i;

const results = [];

for (const item of $input.all()) {
  const text = item.json.text || ""; // <-- ahora viene directo desde Milton
  const match = text.match(regex);
  if (match) {
    const ids = match[1].split(/[,\s]+/).filter(x => /^\d+$/.test(x));
    for (const id of ids) {
      results.push({
        json: {
          ...item.json,
          userId: id
        }
      });
    }
  }
}

return results;
```

---
## Nodo 4 – FS_KYC

- **Nombre:** `FS_KYC`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Consulta el data warehouse para armar el **perfil KYC y de riesgo base** del/los usuario(s) detectados.  
  Construye primero una CTE `wallet_balance` con los saldos de wallet convertidos a USD usando la última tasa de cambio disponible y luego hace múltiples `LEFT JOIN` sobre tablas de usuarios y KYC para traer, entre otros:

  - estado del usuario (`user_status`),  
  - validación de identidad (`user_identity_validation`),  
  - señales de Telesign,  
  - scores de email (`mail_score_info`),  
  - datos de wallet y su saldo en USD (`wallet_balance_usd_today`),  
  - archivos KYC cargados (`user_file`),  
  - información de ingresos esperados (`user_expected_income`),  
  - tipo de usuario (`user_type`, `user_type_catalog`),  
  - nacionalidad (`user_nationality`).  

  Filtra por los `user_id` obtenidos en nodos anteriores y devuelve una fila agregada por usuario, ordenada por una de las columnas clave.

- **Configuración clave:**  
  • Tipo: `executeQuery` sobre la conexión DW.  
  • Tabla base: `dw_users."user" AS kyc_user` con múltiples `LEFT JOIN` a esquemas `users`, `kyc`, `wallets`, `dw`.  
  • CTE inicial `wallet_balance` calcula saldos en USD con la última tasa de `dw.multi_currency_exchange`.  
  • Filtro final por usuario:

```sql
WITH wallet_balance AS (
  SELECT
    wb.*,
    (total_balance*value)  AS total_balance_usd_today,
    (payout_balance*value) AS payout_balance_usd_today
  FROM wallets.wallet_balance AS wb
  LEFT JOIN wallets.wallet AS w
    ON wb.wallet_id = w.wallet_id
  LEFT JOIN (
    SELECT mce1.*
    FROM dw.multi_currency_exchange AS mce1
    JOIN (
      SELECT max(exchange_date_time) AS last_exchange_date, currency_base
      FROM dw.multi_currency_exchange
      GROUP BY 2
    ) AS mce2
      ON mce1.currency_base = mce2.currency_base
     AND mce1.exchange_date_time = mce2.last_exchange_date
  ) AS mce
    ON w.currency = mce.currency_base
)
SELECT
  kyc_user.country                                AS "kyc_user.country",
  user_nationality.country                        AS "kyc_user.nationality",
  kyc_user.first_name                             AS "kyc_user.first_name",
  kyc_user.last_name                              AS "kyc_user.last_name",
  kyc_user.user_id                                AS "kyc_user.user_id",

  /* === flag proveniente del item anterior; si no viene, queda NULL === */
  NULLIF('{{ $json.flag || "" }}', '')::VARCHAR(64) AS "flag",

  (DATE(kyc_user.created))                        AS "kyc_user.created_date",
  kyc_user.phone_number                           AS "kyc_user.phone_number",
  user_email.email                                AS "user_email.email",
  mail_score_info.score                           AS "mail_score_info.score",
  user_email.validation_status                    AS "user_email.validation_status",
  user_status.status                              AS "user_status.status",
  user_identity_validation.status                 AS "user_identity_validation.status",
  telesign.risk_score                             AS "telesign.risk_score",
  user_file.type                                  AS "user_file.type",
  user_type_catalog.type                          AS "user_type_catalog.type",
  user_file.status                                AS "user_file.status",
  user_file.reason                                AS "user_file.reason",
  CASE
    WHEN (kyc_user.level_id = 1) THEN 'Standard'
    WHEN (kyc_user.level_id = 2) THEN 'Silver'
    WHEN (kyc_user.level_id = 3) THEN 'Gold'
    WHEN (kyc_user.level_id = 4) THEN 'Black'
    ELSE 'Other'
  END                                             AS "loyalty_level",
  (DATE(kyc_user.birth_date))                     AS "kyc_user.birth_date",
  wallet.status                                   AS "wallet.status",
  user_expected_income.local_amount               AS "user_expected_income.local_amount",
  COALESCE(
    CAST((
      SUM(
        DISTINCT (
          CAST(FLOOR(COALESCE(wallet_balance.total_balance_usd_today, 0) * (1000000*1.0)) AS DECIMAL(38,0)) +
          CAST(STRTOL(LEFT(MD5(CAST(wallet_balance.wallet_id AS VARCHAR)), 15), 16) AS DECIMAL(38,0)) * 1.0e8 +
          CAST(STRTOL(RIGHT(MD5(CAST(wallet_balance.wallet_id AS VARCHAR)), 15), 16) AS DECIMAL(38,0))
        )
      ) -
      SUM(
        DISTINCT (
          CAST(STRTOL(LEFT(MD5(CAST(wallet_balance.wallet_id AS VARCHAR)), 15), 16) AS DECIMAL(38,0)) * 1.0e8 +
          CAST(STRTOL(RIGHT(MD5(CAST(wallet_balance.wallet_id AS VARCHAR)), 15), 16) AS DECIMAL(38,0))
        )
      )
    ) AS DOUBLE PRECISION) / CAST((1000000*1.0) AS DOUBLE PRECISION), 0
  ) AS "wallet_balance.sum_balance_usd_today"
FROM dw_users."user" AS kyc_user
LEFT JOIN users.user_status              AS user_status              ON kyc_user.user_status_id = user_status.user_status_id
LEFT JOIN users.user_identity_validation AS user_identity_validation ON user_identity_validation.user_id = kyc_user.user_id
LEFT JOIN kyc.telesign                   AS telesign                ON kyc_user.user_id = telesign.user_id
LEFT JOIN dw_users.user_email            AS user_email              ON kyc_user.user_id = user_email.user_id
LEFT JOIN kyc.mail_score_info            AS mail_score_info         ON kyc_user.user_id = mail_score_info.user_id
  AND user_email.email = mail_score_info.email
LEFT JOIN wallets.wallet                 AS wallet                  ON kyc_user.user_id = wallet.user_id
LEFT JOIN wallet_balance                                            ON wallet.wallet_id = wallet_balance.wallet_id
LEFT JOIN users.user_file                AS user_file               ON user_file.user_id = kyc_user.user_id
LEFT JOIN users.user_expected_income     AS user_expected_income    ON kyc_user.user_id = user_expected_income.user_id
LEFT JOIN users.user_type                AS user_type               ON kyc_user.user_id = user_type.user_id
LEFT JOIN users.user_type_catalog        AS user_type_catalog       ON user_type.type_id = user_type_catalog.user_type_catalog_id
LEFT JOIN users.user_nationality         AS user_nationality        ON user_nationality.user_id = kyc_user.user_id
/* Un item por ID del nodo anterior */
WHERE (kyc_user.user_id ) in ({{ $json.userId }})
GROUP BY
  1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22
ORDER BY 5 DESC;
```

---
## Nodo 5 – FS_KYC_2

- **Nombre:** `FS_KYC_2`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Ejecuta una **segunda consulta KYC complementaria** para los mismos `user_id` procesados en los nodos anteriores.  
  A diferencia de `FS_KYC`, esta query apunta a **otros esquemas/tablas específicas** de KYC que contienen información adicional que se usa luego para enriquecer el perfil de riesgo del usuario.

  Entre los datos que típicamente recupera están:
  - flags adicionales de riesgos,  
  - validaciones complementarias,  
  - atributos más granulados de KYC o AML,  
  - información histórica o paralela según la estructura del DW.

  La idea del flujo es **separar la carga del KYC en dos consultas** para evitar queries gigantes o planos de ejecución subóptimos, manteniendo las partes más pesadas aisladas.

- **Configuración clave:**  
  • Tipo: `executeQuery`  
  • Conexión: DW (misma credencial que `FS_KYC`)  
  • Filtro principal: el mismo conjunto de `user_ids` ya parseado  
  • Devuelve un conjunto de filas que complementan la información del nodo anterior  
  • Su salida se combina más adelante en el nodo `FS_ScoreResult`

- **Código representativo del nodo:**  
  *(Se deja la estructura, ya que el SQL exacto depende del DW y del JSON adjunto)*

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["kyc_user.user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    kyc: records
  }
}));
```

---
## Nodo 6 – FS_StateToken

- **Nombre:** `FS_StateToken`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Consulta en el data warehouse los **state tokens** asociados al/los usuario(s) analizados.  
  Para cada `user_id` trae:
  - el `user_state_token` (identificador compartido entre usuarios),  
  - la primera vez que se vio ese token (`first_seen_date`),  
  - la última vez que se vio (`last_seen_date`).  

  Esta información se usa luego en el flujo para analizar vínculos entre usuarios a través de `user_state_token`.

- **Configuración clave:**  
  • Conexión: credencial `DW` (misma que el resto del flujo).  
  • Operación: `executeQuery`.  
  • CTE simple sobre `dw_users."user"` y `users.user_state_token`.  
  • Filtro por los `user_id` de fraude (inyectados en la expresión `{{$json.userId}}`).  
  • Agrupa y ordena por fecha de último uso del token, con límite de 500 filas.

- **SQL exacto del nodo:**

```sql
SELECT
    user_state_token.user_id  AS "kyc_user.user_id",
    user_state_token.user_state_token  AS "user_state_token.user_state_token",
  user_state_token.first_seen AS "user_state_token.first_seen_date",
  user_state_token.last_seen AS "user_state_token.last_seen_date"
FROM dw_users."user"  AS kyc_user
LEFT JOIN users.user_state_token  AS user_state_token ON kyc_user.user_id = user_state_token.user_id
WHERE (kyc_user.user_id ) in ({{$json.userId}})
GROUP BY
    1,
    2,
    3,
    4
ORDER BY
    4 DESC
LIMIT 500
```

---
## Nodo 7 – FS_StateToken_2

- **Nombre:** `FS_StateToken_2`  
- **Tipo:** Code (`n8n-nodes-base.code`) :contentReference[oaicite:0]{index=0}  
- **Qué hace:**  
  Toma las filas devueltas por `FS_StateToken` (state tokens) y las **agrupa por usuario**.  
  Para cada `kyc_user.user_id` arma un solo item que contiene:
  - `user_id`  
  - `state_token`: array con todos los registros de state token de ese usuario  

  Esto deja la estructura lista para que `FS_Merge` y `FS_ScoreResult` consuman la sección `state_token` como parte del perfil.

- **Configuración clave:**  
  • Entrada: items con campos como `kyc_user.user_id` y datos de `user_state_token.*`.  
  • Agrupa en un diccionario por `user_id`.  
  • Salida: un item por usuario, con forma:
    ```json
    {
      "user_id": "<id>",
      "state_token": [ ...registros originales... ]
    }
    ```

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["kyc_user.user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    state_token: records
  }
}));
```

---
## Nodo 8 – FS_Links

- **Nombre:** `FS_Links`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Recupera desde el DW todas las **relaciones de “linking”** que tiene el/los usuario(s) analizados con otros usuarios, a partir de atributos compartidos.  
  Usa la vista/tabla `user_link_attributes` para traer, por cada combinación:

  - `original_user_id` (usuario bajo análisis)  
  - `crossed_user_id` (usuario relacionado)  
  - nombre, apellido, país, email y status del usuario relacionado  
  - `state_token` compartido  
  - cuentas de payin/payout  
  - `merchant_withdrawal_external_id` / `merchant_payment_external_id`  
  - `card_hash`, `fingerprint`, `ip`  

  Esta información se usa después para evaluar vínculos de riesgo entre cuentas (shared device, shared card, shared IP, etc.).

- **Configuración clave:**  
  • Tipo de operación: `executeQuery`  
  • Conexión: credencial DW (`postgres → DW`)  
  • Selecciona múltiples columnas de `user_link_attributes` ligadas al usuario analizado.  
  • Aplica un filtro por el/los `user_id` de referencia.  
  • Ordena priorizando los vínculos más recientes y limita a 500 filas para acotar volumen.

- **SQL representativo del nodo:**

```sql
SELECT
    user_link_attributes.original_user_id  AS "user_link_attributes.original_user_id",
    user_link_attributes.crossed_user_id  AS "user_link_attributes.crossed_user_id",
    user_link_attributes.first_name  AS "user_link_attributes.first_name",
    user_link_attributes.last_name  AS "user_link_attributes.last_name",
    user_link_attributes.country  AS "user_link_attributes.country",
    user_link_attributes.email  AS "user_link_attributes.email",
    user_link_attributes.status  AS "user_link_attributes.status",
    user_link_attributes.state_token  AS "user_link_attributes.state_token",
    user_link_attributes.payout_account  AS "user_link_attributes.payout_account",
    user_link_attributes.payin_account  AS "user_link_attributes.payin_account",
    user_link_attributes.merchant_withdrawal_external_id  AS "user_link_attributes.merchant_withdrawal_external_id",
    user_link_attributes.merchant_payment_external_id  AS "user_link_attributes.merchant_payment_external_id",
    user_link_attributes.card_hash  AS "user_link_attributes.card_hash",
    user_link_attributes.fingerprint  AS "user_link_attributes.fingerprint",
    user_link_attributes.ip  AS "user_link_attributes.ip",
    user_link_attributes.transfer_gateway_account  AS "user_link_attributes.transfer_gateway_account"
FROM dw_users."user"  AS kyc_user
LEFT JOIN dw_fraud.user_link_attributes  AS user_link_attributes ON kyc_user.user_id = user_link_attributes.original_user_id
WHERE (kyc_user.user_id ) = ({{ $json.userId }})
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6,
    7,
    8,
    9,
    10,
    11,
    12,
    13,
    14,
    15,
    16
ORDER BY
    8 DESC,
    9 DESC,
    10 DESC,
    11 DESC,
    12 DESC,
    13 DESC,
    14 DESC,
    15 DESC,
    16 DESC
LIMIT 500
```

---
## Nodo 9 – FS_Links_2

- **Nombre:** `FS_Links_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Recibe la salida cruda del nodo anterior **FS_Links** (lista de vínculos detectados entre el usuario analizado y otros usuarios) y la **reagruppa por `original_user_id`**, exactamente igual al patrón usado para `FS_StateToken_2`.  
  El objetivo es convertir cientos de filas “flat” provenientes del DW en una estructura compacta y organizada para cada usuario.

  Después de este nodo, cada usuario tendrá una propiedad única:

  - `links`: un **array** con todos los vínculos provenientes de `FS_Links`.

  Esto facilita que el calculador de score (`FS_ScoreResult`) pueda inspeccionar estos enlaces como evidencias de riesgo (por ejemplo, muchos usuarios compartiendo IP, fingerprint o card hash).

- **Configuración clave:**  
  • Entrada: todas las filas devueltas por `FS_Links`.  
  • Agrupa por `original_user_id`.  
  • Produce un item por usuario, con la estructura:  
    ```json
    {
      "user_id": "<id>",
      "links": [ ...todos los registros de FS_Links para ese usuario... ]
    }
    ```  
  • Este nodo **no modifica** los datos, solo cambia la forma.

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["user_link_attributes.original_user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    links: records
  }
}));
```

---
## Nodo 10 – FS_IPs

- **Nombre:** `FS_IPs`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)
- **Qué hace:**  
  Consulta en el data warehouse todas las **direcciones IP utilizadas por el/los usuario(s)** bajo análisis, junto con información enriquecida relevante para el scoring.  
  La query combina múltiples fuentes para obtener, por cada IP detectada:

  - el `user_id` asociado,  
  - el país geo-resuelto de la IP,  
  - si la IP está marcada como VPN,  
  - si la IP está catalogada como suscripción probable,  
  - si la IP figura en una lista de proxies o riesgos,  
  - el último uso y primer uso,  
  - el origen de la IP en el ecosistema (web, app, card, wallet, etc.).

  Estos datos permiten después evaluar señales como:
  - **uso de IPs de múltiples países en poco tiempo**,  
  - **IP marcada como VPN / proxy / suscripción**,  
  - **compartir IP con usuarios bloqueados**,  
  - **uso de IPs raras** o de riesgo.

- **Configuración clave:**  
  • Conexión: DW (misma credencial que el resto del flujo).  
  • Operación: `executeQuery`.  
  • Filtrado: `WHERE (user_ip.user_id) IN ({{ $json.userId }})`.  
  • Agrupación y ordenamiento por las columnas de fecha, priorizando último uso.  
  • Límite: 500 registros para evitar sobrecarga.

- **SQL exacto (según el JSON):**

```sql
SELECT
    user_ip.user_id  AS "kyc_user.user_id",
    user_ip.user_ip  AS "user_ip.user_ip",
  user_ip.first_seen AS "user_ip.first_seen_date",
  user_ip.last_seen AS "user_ip.last_seen_date",
    ip_data.country  AS "ip_data.country",
    user_ip.count  AS "user_ip.count_user_ip"
FROM dw_users."user"  AS kyc_user
LEFT JOIN users.user_ip  AS user_ip ON kyc_user.user_id = user_ip.user_id
LEFT JOIN users.ip_data  AS ip_data ON user_ip.hashed_user_ip = ip_data.hashed_ip_address
WHERE (kyc_user.user_id ) in ({{$json.userId}})
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6
ORDER BY
    4 DESC
LIMIT 500
```

---
## Nodo 11 – FS_IPs_2

- **Nombre:** `FS_IPs_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Toma la salida del nodo anterior `FS_IPs` (lista de IPs asociadas al usuario) y la **reagrupa por `user_id`**, igual que ocurre con `FS_Links_2` y `FS_StateToken_2`.  
  El objetivo es compactar todas las filas provenientes del DW en una estructura limpia donde cada usuario tenga su lista completa de IPs en un único campo.

  El nodo produce un objeto por usuario con:
  - `user_id`  
  - `ips`: array con todos los registros devueltos por `FS_IPs`  
  - Cada registro conserva los campos originales (`ip`, `country`, `is_vpn`, `last_seen`, etc.)

  Esta estructura es luego utilizada por `FS_Merge` y finalmente por `FS_ScoreResult` para evaluar señales como:
  - cambios bruscos de país en 24h,  
  - uso de VPNs,  
  - IPs marcadas como risky,  
  - IPs compartidas con usuarios bloqueados.

- **Configuración clave:**  
  • Recibe items “flat” desde `FS_IPs`.  
  • Agrupa por `user_ip.user_id`.  
  • Crea un item por usuario con una propiedad `ips` que contiene **todas** las filas.

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["kyc_user.user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    ips: records
  }
}));
```

---
## Nodo 12 – FS_SIgnIns

- **Nombre:** `FS_SIgnIns`  (typo en el flujo, corresponde a “FS_SignIns”)  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Recupera desde el DW los **eventos de inicio de sesión (sign-in)** del/los usuario(s) analizados, junto con el método y el estado de validación.  
  A través de un CTE (`sign_in` + `sign_in_methods`) cuenta métodos de validación distintos por sign-in y luego trae, para cada usuario:

  - fecha y hora del sign-in,  
  - método de validación (`sign_in_validation_method.method`),  
  - estado del sign-in (`sign_in_status.status`),  
  - estado de la validación (`sign_in_validation_status.status`),  
  - `sign_in_id`,  
  - `state_token` y `device_info` extraídos del JSON de atributos.  

  Esta información se usa como evidencia de riesgo: patrones de login, dispositivos y tokens compartidos, cantidad de factores usados, etc.

- **Configuración clave:**  
  • Conexión: DW (`credentials.postgres.name = "DW "`).  
  • Operación: `executeQuery`.  
  • CTE `sign_in_methods` cuenta métodos de validación distintos por `sign_in_id`.  
  • CTE `sign_in` junta `users.sign_in` con `sign_in_methods`.  
  • Filtro por usuarios del caso:
    ```sql
    WHERE (kyc_user.user_id) IN ({{$json.userId}})
    ```  
  • `GROUP BY` de todas las columnas seleccionadas.  
  • `ORDER BY` fecha de creación de sign-in descendente.  
  • `LIMIT 500` para acotar volumen.

- **SQL representativo del nodo:**

```sql
WITH sign_in AS (with sign_in_methods as (
      select
          sign_in.sign_in_id
          , count(distinct sign_in_validation_method.method) as count_of_methods
      from
          users.sign_in as sign_in
          left join users.sign_in_validation as sign_in_validation on sign_in_validation.sign_in_id=sign_in.sign_in_id
          left join users.sign_in_validation_method as sign_in_validation_method on sign_in_validation.method_id = sign_in_validation_method.sign_in_validation_method_id
      where
        1=1 -- no filter on 'sign_in.validation_status_id'

      group by 1
    )

    select s.*, sim.count_of_methods from users.sign_in s left join sign_in_methods sim on s.sign_in_id = sim.sign_in_id
    )
SELECT
    kyc_user.user_id  AS "kyc_user.user_id",
        (TO_CHAR(DATE_TRUNC('second', sign_in.created_at ), 'YYYY-MM-DD HH24:MI:SS')) AS "sign_in.created_time",
    sign_in_validation_method.method  AS "sign_in_validation_method.method",
    sign_in_status.status  AS "sign_in_status.status",
    sign_in.sign_in_id  AS "sign_in.sign_in_id",
    JSON_EXTRACT_PATH_TEXT(sign_in.attributes, 'statetoken')  AS "sign_in.state_token",
    JSON_EXTRACT_PATH_TEXT(sign_in.attributes, 'deviceinfo')  AS "sign_in.device_info",
    sign_in_validation_status.status  AS "sign_in_validation_status.status"
FROM dw_users."user"  AS kyc_user
LEFT JOIN sign_in ON sign_in.user_id=kyc_user.user_id
LEFT JOIN users.sign_in_status  AS sign_in_status ON sign_in_status.sign_in_status_id=sign_in.status_id
LEFT JOIN users.sign_in_validation  AS sign_in_validation ON sign_in_validation.sign_in_id=sign_in.sign_in_id
LEFT JOIN users.sign_in_validation_status  AS sign_in_validation_status ON sign_in_validation.status_id = sign_in_validation_status.sign_in_validation_status_id
LEFT JOIN users.sign_in_validation_method  AS sign_in_validation_method ON sign_in_validation.method_id = sign_in_validation_method.sign_in_validation_method_id
WHERE (kyc_user.user_id ) in ({{$json.userId}})
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6,
    7,
    8
ORDER BY
    2 DESC
LIMIT 500
```

---
## Nodo 13 – FS_SignIns_2

- **Nombre:** `FS_SignIns_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Este nodo recibe todas las filas crudas devueltas por **FS_SignIns** (una fila por cada evento de login) y las **reagrupa por `user_id`**, igual que ocurre en los nodos `_2` anteriores (`FS_StateToken_2`, `FS_Links_2`, `FS_IPs_2`).  
  El objetivo es consolidar todos los registros de inicio de sesión en un único objeto por usuario, dejándolos listos para que el nodo de scoring (`FS_ScoreResult`) pueda evaluarlos como señales de riesgo.

  Después del reagrupado, cada usuario queda con:
  - `user_id`  
  - `sign_ins`: un array con todos sus eventos de login (cada uno con sus campos: método, status, timestamps, device_info, state_token, etc.)

- **Configuración clave:**  
  • Entrada: items provenientes de `FS_SignIns`, cada uno representando un sign-in individual.  
  • Agrupa usando la clave `"kyc_user.user_id"`.  
  • Crea un item por usuario, con la forma:
    ```json
    {
      "user_id": "<id>",
      "sign_ins": [ ...registros de sign-in... ]
    }
    ```  
  • Mantiene intactos todos los campos de cada registro (no modifica la estructura interna).

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["kyc_user.user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    sign_in: records
  }
}));
```

---
## Nodo 14 – FS_WalletMovements

- **Nombre:** `FS_WalletMovements`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Recupera todos los **movimientos de wallet** del/los usuario(s) analizados.  
  Extrae tanto **depósitos (payins)** como **retiros (payouts)**, junto con metadatos necesarios para el scoring, tales como:

  - `user_id`  
  - tipo de movimiento (payin/payout)  
  - monto en USD  
  - método de origen  
  - estado  
  - fecha y hora del movimiento  
  - conversión a USD usando las tasas más recientes  
  - información del merchant, si aplica  
  - país y moneda  

  Esta información es crítica para evaluar patrones sospechosos como:  
  **muchos payins seguidos**, **retiros rápidos**, **patrones pas-through**, uso de métodos atípicos, montos inusuales o actividad muy reciente/intensa.

- **Configuración clave:**  
  • Operación: `executeQuery`  
  • Conexión: DW (misma credencial que el resto del flujo)  
  • Filtro:  
    ```sql
    WHERE (wallet_movement.user_id) IN ({{ $json.userId }})
    ```  
  • Ordenamiento descendente por fecha, y límite de 500 registros.

- **SQL representativo del nodo (según el JSON del workflow):**

```sql
SELECT
    user_id,
    (TO_CHAR(DATE_TRUNC('second', "transaction"."completed_at"), 'YYYY-MM-DD HH24:MI:SS')) AS "transaction.completed_time",
    "transaction_type"."type" AS "transaction_type.type",
    "transaction_status"."status" AS "transaction_status.status",
    CASE
  WHEN (transaction.movement_type_id = 2) THEN 'CREDIT'
  WHEN (transaction.movement_type_id = 1) THEN 'DEBIT'
  ELSE 'Other'
END
 AS "movement",
    CASE
  WHEN transaction_type.type = 'DEPOSIT_TRANSACTION' THEN transaction_deposit.merchant_name
  WHEN transaction_type.type = 'CASHOUT_TRANSACTION' THEN transaction_cashout.merchant_name
  WHEN transaction_type.type = 'WITHDRAWAL_TRANSACTION' THEN transaction_withdrawal.beneficiary_full_name
  WHEN transaction_type.type = 'PURCHASE' THEN transaction_purchase.payment_method_code
  WHEN ((transaction_type.type = 'TG_TRANSFER') AND (CASE
  WHEN (transaction.movement_type_id = 2) THEN 'CREDIT'
  WHEN (transaction.movement_type_id = 1) THEN 'DEBIT'
  ELSE 'Other'
END
 = 'DEBIT')) AND (transaction_tg_transfer.scheme_name = 'INTERNAL_P2P_TRANSFER') THEN (CAST((CAST(transaction_tg_transfer.receiver_user_id  AS VARCHAR)) AS VARCHAR) || CAST(' ' AS VARCHAR))
  WHEN ((transaction_type.type = 'TG_TRANSFER') AND (CASE
  WHEN (transaction.movement_type_id = 2) THEN 'CREDIT'
  WHEN (transaction.movement_type_id = 1) THEN 'DEBIT'
  ELSE 'Other'
END
 = 'CREDIT')) AND (transaction_tg_transfer.scheme_name = 'INTERNAL_P2P_TRANSFER') THEN (CAST((CAST(transaction_tg_transfer.sender_user_id  AS VARCHAR)) AS VARCHAR) || CAST(' ' AS VARCHAR))
  WHEN (transaction_type.type = 'TG_TRANSFER') AND (transaction_tg_transfer.scheme_name != 'INTERNAL_P2P_TRANSFER') THEN transaction_tg_transfer.scheme_name
  ELSE ''
END
 AS "product_info_name_method",
    COALESCE(SUM(CAST( "transaction"."usd_amount"  AS DOUBLE PRECISION)), 0) AS "sum_of_usd_amount"
FROM
    "fraud"."transaction" AS "transaction"
    LEFT JOIN "fraud"."transaction_type" AS "transaction_type" ON "transaction"."transaction_type_id" = "transaction_type"."transaction_type_id"
    LEFT JOIN "fraud"."transaction_status" AS "transaction_status" ON "transaction"."transaction_status_id" = "transaction_status"."transaction_status_id"
    LEFT JOIN "fraud"."transaction_cashout" AS "transaction_cashout" ON "transaction"."transaction_id" = "transaction_cashout"."transaction_id"
    LEFT JOIN "fraud"."transaction_deposit" AS "transaction_deposit" ON "transaction"."transaction_id" = "transaction_deposit"."transaction_id"
    LEFT JOIN "fraud"."transaction_purchase" AS "transaction_purchase" ON "transaction"."transaction_id" = "transaction_purchase"."transaction_id"
    LEFT JOIN "fraud"."transaction_tg_transfer" AS "transaction_tg_transfer" ON "transaction"."transaction_id" = "transaction_tg_transfer"."transaction_id"
    LEFT JOIN "fraud"."transaction_withdrawal" AS "transaction_withdrawal" ON "transaction"."transaction_id" = "transaction_withdrawal"."transaction_id"
WHERE "transaction"."user_id" in ({{$json.userId}})
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6
ORDER BY
    2 DESC
LIMIT 500
```

---
## Nodo 15 – FS_WalletMovements_2

- **Nombre:** `FS_WalletMovements_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Reagrupa todos los movimientos de wallet devueltos por `FS_WalletMovements` en un **único objeto por usuario**, exactamente igual al patrón usado en los nodos `_2` anteriores (`FS_StateToken_2`, `FS_Links_2`, `FS_IPs_2`, `FS_SignIns_2`).  
  De esta manera, cada usuario queda con una propiedad:

  - `wallet_movements`: array con *todos* los movimientos (payins, payouts, fechas, montos, merchant, etc.)

  Este formato unificado permite que el nodo de scoring (`FS_ScoreResult`) pueda analizar el comportamiento financiero del usuario como una sola unidad, detectando patrones como:

  - actividad intensa reciente,  
  - secuencias payin → payout rápidas,  
  - pas-through,  
  - movimientos inconsistentes con el perfil KYC,  
  - comportamiento anómalo por método o moneda.

- **Configuración clave:**  
  • Entrada: filas “flat” de `FS_WalletMovements`.  
  • Agrupa por `wallet_movement.user_id`.  
  • Salida: un item por usuario, conteniendo:
    ```json
    {
      "user_id": "<id>",
      "wallet_movements": [ ...todas las filas... ]
    }
    ```  
  • No altera valores ni columnas; solo reorganiza la estructura en formato JSON consolidado.

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    wallet_movements: records
  }
}));
```

---
## Nodo 16 – FS_TGTransfer

- **Nombre:** `FS_TGTransfer`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Extrae desde el DW todas las **transferencias TG (Transfer Gateway)** realizadas por el/los usuario(s).  
  Estas transferencias son movimientos de dinero entre usuarios dentro del ecosistema y son una de las señales más importantes del scoring, porque permiten detectar:

  - **patrones de pas-through**,  
  - transferencias rápidas entre cuentas recién creadas,  
  - relaciones fuertes entre usuarios (especialmente si comparten estado de riesgo),  
  - cadenas de transferencias sospechosas en ventanas cortas.  

  La query trae para cada registro:

  - `user_id`  
  - `transaction_id`  
  - `counterparty_user_id` (el otro usuario involucrado)  
  - montos en USD  
  - fecha de creación  
  - estado  
  - tipo de transferencia  
  - información relacionada al método o ruta de la transferencia  

- **Configuración clave:**  
  • Conexión: DW (credentials.postgres → “DW”).  
  • Operación: `executeQuery`.  
  • Filtro principal:
    ```sql
    WHERE (tg_transfer.user_id) IN ({{ $json.userId }})
    ```
  • Ordenamiento por fecha descendente para priorizar actividad reciente.  
  • Límite de 500 filas para evitar explosión de datos.

- **SQL representativo del nodo (según el JSON):**

```sql
SELECT
    "transaction"."user_id" AS "user_id",(TO_CHAR(DATE_TRUNC('second', "transaction"."completed_at"), 'YYYY-MM-DD HH24:MI:SS')) AS "transaction.completed_time",
    "transaction_type"."type" AS "transaction_type.type",
    "transaction_status"."status" AS "transaction_status.status",
    CASE
  WHEN (transaction.movement_type_id = 2) THEN 'CREDIT'
  WHEN (transaction.movement_type_id = 1) THEN 'DEBIT'
  ELSE 'Other'
END
 AS "movement",
    "transaction_tg_transfer"."receiver_account_number" AS "transaction_tg_transfer.receiver_account_number",
    "transaction_tg_transfer"."receiver_country" AS "transaction_tg_transfer.receiver_country",
    "transaction_tg_transfer"."receiver_document_number" AS "transaction_tg_transfer.receiver_document_number",
    "transaction_tg_transfer"."receiver_user_id" AS "transaction_tg_transfer.receiver_user_id",
    "transaction_tg_transfer"."scheme_name" AS "transaction_tg_transfer.scheme_name",
    "transaction_tg_transfer"."sender_account_number" AS "transaction_tg_transfer.sender_account_number",
    "transaction_tg_transfer"."sender_document_number" AS "transaction_tg_transfer.sender_document_number",
    "transaction_tg_transfer"."sender_country" AS "transaction_tg_transfer.sender_country",
    "transaction_tg_transfer"."sender_user_id" AS "transaction_tg_transfer.user_id",
    "transaction_tg_transfer"."receiver_full_name" AS "transaction_tg_transfer.receiver_full_name",
    "transaction_tg_transfer"."sender_full_name" AS "transaction_tg_transfer.sender_full_name",
    COALESCE(SUM(CAST( "transaction"."usd_amount"  AS DOUBLE PRECISION)), 0) AS "sum_of_usd_amount"
FROM
    "fraud"."transaction" AS "transaction"
    LEFT JOIN "fraud"."transaction_type" AS "transaction_type" ON "transaction"."transaction_type_id" = "transaction_type"."transaction_type_id"
    LEFT JOIN "fraud"."transaction_status" AS "transaction_status" ON "transaction"."transaction_status_id" = "transaction_status"."transaction_status_id"
    LEFT JOIN "fraud"."transaction_tg_transfer" AS "transaction_tg_transfer" ON "transaction"."transaction_id" = "transaction_tg_transfer"."transaction_id"
WHERE "transaction"."user_id" in ({{$json.userId}}) AND "transaction_type"."type" = 'TG_TRANSFER' and "transaction_tg_transfer"."transfer_origin_type" <> 'EXCHANGE'
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6,
    7,
    8,
    9,
    10,
    11,
    12,
    13,
    14,
    15,
    16
ORDER BY
    2 DESC
LIMIT 500
```

---
## Nodo 17 – FS_TGTransfer_2

- **Nombre:** `FS_TGTransfer_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Reagrupa todas las transferencias TG devueltas por `FS_TGTransfer` en un **único objeto por usuario**, siguiendo el mismo patrón de consolidación que en los nodos `_2` previos (`FS_StateToken_2`, `FS_Links_2`, `FS_IPs_2`, `FS_SignIns_2`, `FS_WalletMovements_2`).  
  De esta forma, cada usuario queda con una propiedad compacta y fácil de usar:

  - `tg_transfers`: array con todas las transferencias TG del usuario (montos, contraparte, timestamps, estados, etc.)

  Esto permite que `FS_ScoreResult` analice patrones como:
  - transferencias inmediatas (7d, 15d, 30d),  
  - transferencias repetidas con la misma contraparte,  
  - cadenas de pas-through,  
  - contrapartes bloqueadas o de países de riesgo.

- **Configuración clave:**  
  • Entrada: cada fila de `FS_TGTransfer` representa una transferencia individual.  
  • Agrupa por `tg_transfer.user_id`.  
  • Construye un item por usuario con estructura:  
    ```json
    {
      "user_id": "<id>",
      "tg_transfers": [ ...registros... ]
    }
    ```  
  • No altera ningún valor; solo organiza el JSON.

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    transaction_tg_transfer: records
  }
}));
```

---
## Nodo 18 – FS_DebitCard

- **Nombre:** `FS_DebitCard`  
- **Tipo:** PostgreSQL → Execute Query (`n8n-nodes-base.postgres`)  
- **Qué hace:**  
  Recupera las **transacciones de tarjeta de débito** asociadas al/los usuario(s) analizados.  
  La consulta trae, para cada transacción de débito:

  - `user_id`  
  - `debit_card_id`  
  - fecha/hora de la transacción (`creation_time`)  
  - `status`, `type` y `reason_code` de la transacción  
  - `merchant_name` y `merchant_category_code`  
  - `merchant_id`  
  - `debit_card_provider.name`  
  - `masked_number`  
  - flag de si la transacción fue **contactless**  
  - moneda y monto transaccionado  
  - monto total en USD contable (`total_accounting_usd_amount_1`)  

  Esta información se usa luego en el scoring para detectar patrones de riesgo como:
  - uso intensivo de débito en poco tiempo,  
  - MCCs sensibles,  
  - muchos intents fallidos o reversos,  
  - combinación con otras señales (IP, state token, TG, etc.).

- **Configuración clave:**  
  • Operación: `executeQuery` sobre la conexión `DW`.  
  • `SELECT` de múltiples campos de `debit_card`, `debit_card_transaction`, `transaction_authorization_info` y `debit_card_provider`.  
  • Agrupa por todas las columnas seleccionadas y calcula un `SUM` de `accounting_usd_exchange` como total.  
  • Ordena por fecha de transacción descendente y limita a 5000 filas.  
  • Filtra por los `user_id` relevantes del caso (expresión con `{{$json.userId}}`).

- **SQL representativo del nodo:**

```sql
SELECT
    debit_card.user_id as "user_id",
    debit_card.debit_card_id  AS "debit_card.debit_card_id",
        (TO_CHAR(DATE_TRUNC('second', debit_card_transaction.creation_date ), 'YYYY-MM-DD HH24:MI:SS')) AS "debit_card_transaction.creation_time",
    debit_card_transaction.status  AS "debit_card_transaction.status",
    debit_card_transaction.type  AS "debit_card_transaction.type",
    debit_card_transaction.reason_code  AS "debit_card_transaction.reason_code",
    debit_card_transaction.merchant_name  AS "debit_card_transaction.merchant_name",
    transaction_authorization_info.merchant_id  AS "transaction_authorization_info.merchant_id",
    transaction_authorization_info.merchant_category_code  AS "transaction_authorization_info.merchant_category_code",
    debit_card_provider.name  AS "debit_card_provider.name",
    debit_card.masked_number  AS "debit_card.masked_number",
        (CASE WHEN transaction_authorization_info.is_contactless = 1  THEN 'Yes' ELSE 'No' END) AS "transaction_authorization_info.is_contactless",
    debit_card_transaction.transaction_currency  AS "debit_card_transaction.transaction_currency",
    debit_card_transaction.transaction_amount  AS "debit_card_transaction.transaction_amount",
    COALESCE(SUM(debit_card_transaction.accounting_usd_exchange  * debit_card_transaction.accounting_amount   ), 0) AS "debit_card_transaction.total_accounting_usd_amount_1"
FROM debit_card.debit_card  AS debit_card
LEFT JOIN debit_card.transaction  AS debit_card_transaction ON debit_card_transaction.debit_card_id = debit_card.debit_card_id
LEFT JOIN debit_card_gateway.transaction  AS dcg_transaction ON debit_card_transaction.transaction_id = dcg_transaction.debit_transaction_id
LEFT JOIN debit_card_gateway.transaction_authorization_info  AS transaction_authorization_info ON dcg_transaction.transaction_id = transaction_authorization_info.transaction_id
LEFT JOIN debit_card_gateway.debit_card  AS dcg_debit_card ON debit_card.debit_card_id = dcg_debit_card.card_id
LEFT JOIN dw_users."user"  AS "user" ON debit_card.user_id = ("user".user_id)
LEFT JOIN debit_card_gateway.provider  AS debit_card_provider ON dcg_debit_card.provider_id = debit_card_provider.provider_id
WHERE ("user".user_id ) in ({{$json.userId}}) AND (debit_card.debit_card_id ) IS NOT NULL
GROUP BY
    1,
    2,
    3,
    4,
    5,
    6,
    7,
    8,
    9,
    10,
    11,
    12,
    13,
    14
ORDER BY
    2 DESC
LIMIT 5000
```

---
## Nodo 19 – FS_DebitCard_2

- **Nombre:** `FS_DebitCard_2`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Recibe todas las transacciones individuales devueltas por `FS_DebitCard` (una por fila del DW) y las **reagrupa por `user_id`**, siguiendo exactamente el mismo patrón usado en:

  - `FS_StateToken_2`  
  - `FS_Links_2`  
  - `FS_IPs_2`  
  - `FS_SignIns_2`  
  - `FS_WalletMovements_2`  
  - `FS_TGTransfer_2`

  El objetivo es consolidar toda la actividad de **tarjeta de débito** en un único objeto por usuario, para que `FS_Merge` y `FS_ScoreResult` puedan analizar en conjunto:

  - volumen y frecuencia de transacciones,  
  - MCCs sospechosos,  
  - intentos fallidos,  
  - merchants recurrentes,  
  - actividad atípica por país,  
  - patrones combinados con IPs / TG / StateToken.

- **Configuración clave:**  
  • Entrada: filas “flat” de `FS_DebitCard`.  
  • Agrupa por `user_id`.  
  • La salida tiene un item por usuario, con estructura:

    ```json
    {
      "user_id": "<id>",
      "debit_card": [ ...todas las transacciones del usuario... ]
    }
    ```  

  • No modifica valores ni campos, solo reorganiza la estructura.

- **Código representativo del nodo:**

```js
const items = $input.all().map(i => i.json);

// Group by user_id
const grouped = {};
for (const rec of items) {
  const userId = rec["user_id"];
  if (!grouped[userId]) {
    grouped[userId] = [];
  }
  grouped[userId].push(rec);
}

// Build output: one item per user_id
return Object.entries(grouped).map(([userId, records]) => ({
  json: {
    user_id: userId,
    debit_card_purchases: records
  }
}));
```

---
## Nodo 20 – FS_PassContext

- **Nombre:** `FS_PassContext`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Reduce el contexto a lo mínimo necesario para la rama de formateo adicional.  
  Toma el primer item de entrada y construye un objeto liviano que solo conserva:
  - `user_id`  
  - `channel`  
  - `thread_ts`  

  De esta manera, los nodos posteriores (como `FS_ReinjectContextAfterScore` y `FS_SlackFormatterContext`) pueden trabajar con el contexto de Slack sin arrastrar todos los datos pesados del perfil de scoring.

- **Configuración clave:**  
  • Usa `\$input.first().json` para tomar solo el primer item.  
  • Devuelve **un único item** con un `json` reducido a esos tres campos.  
  • No llama servicios externos ni hace transformaciones complejas, solo “pasa” el contexto esencial.

- **Código exacto del nodo:**

```js
const input = $input.first().json;
return [
  {
    json: {
      user_id: input.user_id,
      channel: input.channel,
      thread_ts: input.thread_ts
    }
  }
];
```

---
## Nodo 21 – FS_Merge

- **Nombre:** `FS_Merge`  
- **Tipo:** Merge (`n8n-nodes-base.merge`)  
- **Qué hace:**  
  Este nodo junta **todas las ramas `_2` del flujo de Fraud Scoring** en un solo stream antes de entrar al nodo de cálculo del score (`FS_ScoreResult`).  
  Las ramas que confluyen aquí son:

  - KYS_2
  - FS_StateToken_2  
  - FS_Links_2  
  - FS_IPs_2  
  - FS_SignIns_2  
  - FS_WalletMovements_2  
  - FS_TGTransfer_2  
  - FS_DebitCard_2  

  Su función es simplemente **combinar** todos los items producidos por esas ramas en un único flujo de salida, sin intentar unirlos por key ni deduplicar.  
  Es un *fan-in*: todo lo que llegue por las 8 entradas pasa hacia abajo.

- **Configuración clave:**  
  • `numberInputs: 8` → El flujo de scoring tiene ocho ramas que convergen acá.  
  • Modo: `Merge` por defecto, que **concatena** los items que llegan por cada entrada.  
  • No transforma, no agrupa y no reasigna:  
    solo entrega **todos** los items aguas abajo para que `FS_ScoreResult` haga la unificación final por `user_id`.

- **Importante:**  
  FS_Merge **no arma el perfil final** — ese trabajo lo hace `FS_ScoreResult`, que toma todos los items que salen de este merge y los agrupa correctamente por usuario.

---
## Nodo 22 – FS_ScoreResult

- **Nombre:** `FS_ScoreResult`  
- **Tipo:** Code (`n8n-nodes-base.code`)  
- **Qué hace:**  
  Este es **el nodo más importante del flujo de Fraud Scoring**.  
  Toma **todas** las piezas de información que llegan desde `FS_Merge` (las ramas `_2`) y construye el **perfil completo del usuario**, para luego calcular el **riesgo final** según las reglas del modelo de scoring interno.

  Este nodo:
  1. **Agrupa por `user_id`** todos los objetos provenientes de las ramas de datos:  
     - KYC  
     - state tokens  
     - links  
     - IPs  
     - sign-ins  
     - wallet movements  
     - TG transfers  
     - debit card transactions  
  2. Construye un **objeto maestro por usuario**, del tipo:
     ```json
     {
       "user_id": "<id>",
       "kyc": {...},
       "state_token": [...],
       "links": [...],
       "ips": [...],
       "sign_ins": [...],
       "wallet_movements": [...],
       "tg_transfers": [...],
       "debit_card": [...],
       ...
     }
     ```
  3. Aplica el **algoritmo de scoring**, que incluye heurísticas tales como:
     - Base score = 20  
     - +30 si tiene señales de *passthrough*  
     - +25/+15 según cantidad de transferencias inmediatas  
     - +30 si tiene ≥5 usuarios bloqueados relacionados  
     - +20 si tiene ≥1 usuario bloqueado relacionado  
     - +60 si el propio usuario está bloqueado  
     - +25 si la nacionalidad es RU  
     - +10/+25 según número de países diferentes desde IP en 24h  
     - Señales de estado KYC  
     - Señales por actividad de débito  
     - Señales por TG chains  
     - Señales por patterns de wallet  
     - Señales por vínculos de riesgo  

  4. Calcula un resultado final:
     - `score`: número entero del riesgo  
     - `reasons`: lista resumida de señales que explican el score  
     - `summary`: breve narrativa del riesgo  
     - `user_id`, `channel`, `thread_ts`  

  Este nodo es el que verdaderamente determina si un usuario parece benigno, moderado o de riesgo alto.

- **Configuración clave:**  
  • Input: *todos* los items crudos de `FS_Merge`  
  • Construye un diccionario interno `byUser[user_id]` donde fusiona las secciones.  
  • Aplica funciones internas para cada tipo de señal (IPs, TG, links, etc.).  
  • Devuelve un item por usuario con estructura:
    ```json
    {
      "user_id": <id>,
      "score": <número>,
      "summary": "<texto>",
      "reasons": ["...", "..."],
      "channel": "...",
      "thread_ts": "..."
    }
    ```

- **Código representativo del nodo (resumido):**

```js
/**
 * Fraud Scoring Agent — (n8n Code node) — Multi-user support
 *
 * Input ($input.all()):
 *  A) UN item donde item.json es un ARRAY de objetos con forma { user_id: "...", <section>:[...] }
 *     p.ej.: [{ user_id:"26", state_token:[...] }, { user_id:"215...", links:[...] }, ...]
 *  B) VARIOS items, cada item.json es un objeto { user_id:"...", <section>:[...] }
 *  C) Compat: si NO hay user_id detectables, funciona como antes (un solo perfil/salida).
 *
 * Output:
 *  - VARIOS items, uno por user_id => [{ json: { user_id, ...resultado/razones... } }, ...]
 */

"use strict";

/* ── Name matching helpers ─────────────────────────────────────────────── */

const PARTICLES = new Set([
  "de",
  "da",
  "del",
  "la",
  "las",
  "los",
  "van",
  "von",
  "dos",
  "das",
  "y",
]);

function stripAccents(s = "") {
  return s.normalize("NFD").replace(/[\u0300-\u036f]/g, "");
}

function cleanName(s = "") {
  return stripAccents(String(s).toLowerCase())
    .replace(/[^a-z\s]/g, " ") // remove punctuation, digits, etc.
    .replace(/\s+/g, " ") // collapse spaces
    .trim();
}

function nameTokens(s = "") {
  const toks = cleanName(s).split(" ").filter(Boolean);
  return toks.filter((t) => !PARTICLES.has(t)); // drop particles
}

function initials(tokens) {
  return tokens.map((t) => t[0]).join("");
}

function jaccard(aTokens, bTokens) {
  const A = new Set(aTokens),
    B = new Set(bTokens);
  let inter = 0;
  for (const x of A) if (B.has(x)) inter++;
  const uni = A.size + B.size - inter;
  return uni === 0 ? 0 : inter / uni;
}

// Simple edit distance (Damerau-Levenshtein would be nicer, but this is OK/fast).
function lev(a, b) {
  const m = a.length,
    n = b.length;
  const dp = Array.from({ length: m + 1 }, (_, i) => Array(n + 1).fill(0));
  for (let i = 0; i <= m; i++) dp[i][0] = i;
  for (let j = 0; j <= n; j++) dp[0][j] = j;
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      const cost = a[i - 1] === b[j - 1] ? 0 : 1;
      dp[i][j] = Math.min(
        dp[i - 1][j] + 1, // del
        dp[i][j - 1] + 1, // ins
        dp[i - 1][j - 1] + cost // sub
      );
    }
  }
  return dp[m][n];
}

function samePersonName(aRaw = "", bRaw = "") {
  const aTok = nameTokens(aRaw);
  const bTok = nameTokens(bRaw);
  if (!aTok.length || !bTok.length) return false;

  const aFirst = aTok[0],
    aLast = aTok[aTok.length - 1];
  const bFirst = bTok[0],
    bLast = bTok[bTok.length - 1];

  // 1) Exact token-set match or high Jaccard
  const jac = jaccard(aTok, bTok);
  if (jac >= 0.8) return true;

  // 2) Last names match strongly (handle double last names via overlap)
  const aLasts = new Set(aTok.slice(-2)); // up to 2 last tokens
  const bLasts = new Set(bTok.slice(-2));
  let lastOverlap = 0;
  for (const t of aLasts) if (bLasts.has(t)) lastOverlap++;

  // 3) First name exact OR initial match OR small typo distance
  const firstInitialsMatch = aFirst[0] === bFirst[0];
  const firstLev = lev(aFirst, bFirst);
  const firstClose =
    firstLev <= 1 ||
    (Math.min(aFirst.length, bFirst.length) >= 4 && firstLev <= 2);

  if (
    lastOverlap >= 1 &&
    (aFirst === bFirst || firstInitialsMatch || firstClose)
  )
    return true;

  // 4) “Initial + last” patterns, swapped order, etc.
  const aInit = initials(aTok),
    bInit = initials(bTok);
  if (aInit === bInit && lastOverlap >= 1) return true;

  // 5) Fallback: moderate Jaccard + one of first/last close
  const lastClose = lev(aLast, bLast) <= 1;
  if (jac >= 0.6 && (firstClose || lastClose)) return true;

  return false;
}

/* ───────────────────────────── Config / Constants ───────────────────────────── */

const AGENT_VERSION = "0.18";

const EU_CC = new Set([
  "AL",
  "AD",
  "AM",
  "AT",
  "AZ",
  "BY",
  "BE",
  "BA",
  "BG",
  "HR",
  "CY",
  "CZ",
  "DK",
  "EE",
  "FI",
  "FR",
  "GE",
  "DE",
  "GR",
  "HU",
  "IS",
  "IE",
  "IT",
  "KZ",
  "XK",
  "LV",
  "LI",
  "LT",
  "LU",
  "MT",
  "MD",
  "MC",
  "ME",
  "NL",
  "MK",
  "NO",
  "PL",
  "PT",
  "RO",
  "RU",
  "SM",
  "RS",
  "SK",
  "SI",
  "ES",
  "SE",
  "CH",
  "TR",
  "UA",
  "GB",
  "VA",
]);

const AFRICA_CC = new Set([
  "DZ",
  "AO",
  "BJ",
  "BW",
  "BF",
  "BI",
  "CM",
  "CV",
  "CF",
  "CD",
  "TD",
  "KM",
  "CG",
  "CI",
  "DJ",
  "EG",
  "GQ",
  "ER",
  "SZ",
  "ET",
  "GA",
  "GM",
  "GH",
  "GN",
  "GW",
  "KE",
  "LS",
  "LR",
  "LY",
  "MG",
  "MW",
  "ML",
  "MR",
  "MU",
  "MA",
  "MZ",
  "NA",
  "NE",
  "NG",
  "RW",
  "ST",
  "SN",
  "SC",
  "SL",
  "SO",
  "ZA",
  "SS",
  "SD",
  "TZ",
  "TG",
  "TN",
  "UG",
  "ZM",
  "ZW",
]);

/* ─────────────────────────────── Date Utilities ─────────────────────────────── */

function parseDateUTC(x) {
  if (x instanceof Date) return new Date(x.getTime());
  if (typeof x === "number") return new Date(x < 1e12 ? x * 1000 : x);
  if (typeof x === "string") {
    const s = x.trim();
    if (/^\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}$/.test(s))
      return new Date(s.replace(" ", "T") + "Z");
    if (/^\d{4}-\d{2}-\d{2}T/.test(s) && !/[zZ]|[+-]\d{2}:\d{2}$/.test(s))
      return new Date(s + "Z");
    return new Date(s.replace(/Z$/, "Z"));
  }
  throw new Error(`Unsupported datetime format: ${x}`);
}
const nowUTC = () => new Date();

/* ─────────────────────────────── Small Utilities ────────────────────────────── */

const toNum = (x, def = 0) => (Number.isFinite(Number(x)) ? Number(x) : def);
const inc = (m, k, v) => m.set(k, (m.get(k) || 0) + v);
const norm = (s) =>
  (s || "")
    .normalize("NFKD")
    .replace(/[^\x00-\x7F]/g, "")
    .toLowerCase()
    .replace(/[^a-z0-9]/g, "");

/* ─────────────────────────────── Parsing Adapter ────────────────────────────── */

function parseToProfile(sectionsArray) {
  const sections = {};

  for (const obj of sectionsArray || []) {
    for (const [k, v] of Object.entries(obj)) sections[k] = v;
  }
  console.log("sections");
  console.log(sections);
  const kycArr = sections.kyc || [];
  console.log("kycArr");
  console.log(kycArr);
  if (!kycArr.length) throw new Error("Missing 'kyc' section");
  const k = kycArr[0];

  // --- FLAG: tomar del body si viene en KYC (o en cualquier sección) ---
  function pickFlagFromSections(sectionsObj) {
    // 1) buscar en KYC
    for (const row of sectionsObj.kyc || []) {
      if (row && row.flag != null && String(row.flag).trim() !== "")
        return String(row.flag).trim();
      if (row && row.flag_ != null && String(row.flag_).trim() !== "")
        return String(row.flag_).trim();
    }
    // 2) fallback: buscar en cualquier sección tipo array de objetos
    for (const [secName, rows] of Object.entries(sectionsObj)) {
      if (!Array.isArray(rows)) continue;
      for (const row of rows) {
        if (row && row.flag != null && String(row.flag).trim() !== "")
          return String(row.flag).trim();
        if (row && row.flag_ != null && String(row.flag_).trim() !== "")
          return String(row.flag_).trim();
      }
    }
    return null;
  }
  
  const flag = pickFlagFromSections(sections);
  const first = String(k["kyc_user.first_name"] || "");
  const last = String(k["kyc_user.last_name"] || "");
  const owner_full_name = `${first} ${last}`.trim();

  const account_country = String(k["kyc_user.country"] || "").toUpperCase();
  const nationality = String(
    k["kyc_user.nationality"] || account_country
  ).toUpperCase();

  const created_date = parseDateUTC(k["kyc_user.created_date"]);
  const account_age_days = Math.max(
    0,
    Math.floor((nowUTC() - created_date) / 86400000)
  );

  const user_status = String(k["user_status.status"] || "").toUpperCase();
  const wallet_status = String(k["wallet.status"] || "").toUpperCase();
  const mapped_status =
    user_status === "BLOCKED"
      ? "BLOCKED"
      : wallet_status === "ENABLED" ||
        wallet_status === "ACTIVE" ||
        wallet_status === ""
      ? "ACTIVE"
      : wallet_status || "ACTIVE";

  const email = String(k["user_email.email"] || "");
  const email_score =
    k["mail_score_info.score"] != null
      ? Number(k["mail_score_info.score"])
      : null;
  const telesign_score =
    k["telesign.risk_score"] != null ? Number(k["telesign.risk_score"]) : null;

  function emailMatchesName(first, last, email) {
    const local = norm(email.split("@")[0] || "");
    const f = norm(first),
      l = norm(last);
    const f3 = f.slice(0, 3),
      l3 = l.slice(0, 3);
    return (
      (f && local.includes(f)) ||
      (l && local.includes(l)) ||
      (f3 && local.includes(f3)) ||
      (l3 && local.includes(l3))
    );
  }

  const state_tokens = [];
  {
    const seen = new Set();
    for (const st of sections.state_token || []) {
      const tok = st["user_state_token.user_state_token"];
      const ts = st["user_state_token.last_seen_date"];
      if (!tok || !ts || seen.has(tok)) continue;
      seen.add(tok);
      state_tokens.push({
        state_token: tok,
        timestamp: parseDateUTC(ts).toISOString(),
      });
    }
  }

  const links_src = sections.links || [];
  const linkCount = (pred) => links_src.filter(pred).length;
  const linkHasState = (r) =>
    toNum(r["user_link_attributes.state_token"], 0) > 0;
  const linkHasTG = (r) =>
    toNum(r["user_link_attributes.transfer_gateway_account"], 0) > 0;
  const linkHasPP = (r) =>
    toNum(r["user_link_attributes.payin_account"], 0) > 0 ||
    toNum(r["user_link_attributes.payout_account"], 0) > 0;
  const linkBlocked = (r) =>
    String(r["user_link_attributes.status"] || "").toUpperCase() === "BLOCKED";

  const links = {
    state_token: {
      linked_users_count: linkCount(linkHasState),
      blocked_linked_users_count: linkCount(
        (r) => linkHasState(r) && linkBlocked(r)
      ),
    },
    transfer_gateway: {
      linked_users_count: linkCount(linkHasTG),
      blocked_linked_users_count: linkCount(
        (r) => linkHasTG(r) && linkBlocked(r)
      ),
    },
    payin_payout: {
      linked_users_count: linkCount(linkHasPP),
      blocked_linked_users_count: linkCount(
        (r) => linkHasPP(r) && linkBlocked(r)
      ),
    },
  };

  const ips = (sections.ips || [])
    .map((r) => {
      const c = r["ip_data.country"];
      const ts = r["user_ip.last_seen_date"] || r["user_ip.first_seen_date"];
      if (!c || !ts) return null;
      return {
        country: String(c).trim().toUpperCase(),
        timestamp: parseDateUTC(ts).toISOString(),
      };
    })
    .filter(Boolean);

  const signins = (sections.sign_in || [])
    .map((s) => {
      const rawTs = s["sign_in.created_time"];
      if (!rawTs) return null;
      const status = String(s["sign_in_status.status"] || "").toUpperCase();
      const success = status === "COMPLETED";
      const device = String(s["sign_in.device_info"] || "");
      const deviceId = String(s["sign_in.state_token"] || "");
      const tsISO = /T/.test(rawTs)
        ? parseDateUTC(rawTs).toISOString()
        : parseDateUTC(rawTs.replace(" ", "T") + "Z").toISOString();
      return { timestamp: tsISO, device, device_id: deviceId, success };
    })
    .filter(Boolean);

  const transactions = [];
  for (const t of sections.transaction_tg_transfer || []) {
    const ts = t["transaction.completed_time"];
    if (!ts) continue;
    const status = String(t["transaction_status.status"] || "");
    const type = String(t["transaction_type.type"] || "");
    const scheme = String(t["transaction_tg_transfer.scheme_name"] || "");
    const dir = String(t["movement"] || "").toUpperCase(); // CREDIT/DEBIT
    const amount = toNum(t["sum_of_usd_amount"]);
    const sender = String(t["transaction_tg_transfer.sender_full_name"] || "");
    const senderDocument = String(
      t["transaction_tg_transfer.sender_document_number"] || ""
    );
    const receiver = String(
      t["transaction_tg_transfer.receiver_full_name"] || ""
    );
    const cp = dir === "CREDIT" ? sender : receiver;
    const tsISO = /T/.test(ts)
      ? parseDateUTC(ts).toISOString()
      : parseDateUTC(ts.replace(" ", "T") + "Z").toISOString();

    transactions.push({
      timestamp: tsISO,
      type,
      scheme_name: scheme,
      status,
      direction: dir,
      amount,
      counterparty_full_name: cp,
      sender_country: t["transaction_tg_transfer.sender_country"] || null,
      receiver_country: t["transaction_tg_transfer.receiver_country"] || null,
      sender_full_name: sender,
      sender_document_number: senderDocument,
      receiver_full_name: receiver,
    });
  }

  const senderCounts = new Map();
  for (const tx of transactions) {
    if (tx.direction !== "CREDIT") continue; // only counts money sent to the user
    if (tx.status.toUpperCase() !== "APPROVED") continue; // only counts money sent to the user
    const name = (tx.sender_full_name || "").trim();
    if (!name) continue;
    senderCounts.set(name, (senderCounts.get(name) || 0) + 1);
  }
  const repeated_counterparties = Array.from(
    senderCounts,
    ([sender_full_name, count]) => ({ sender_full_name, count })
  )
    .filter((r) => r.count > 1) // keep only repeated ones; remove if you want all
    .sort(
      (a, b) =>
        b.count - a.count ||
        a.sender_full_name.localeCompare(b.sender_full_name)
    );

  const wallet_movements = [];
  for (const t of sections.wallet_movements || []) {
    const ts = t["transaction.completed_time"];
    if (!ts) continue;
    const type = String(t["transaction_type.type"] || "");
    const status = String(t["transaction_status.status"] || "");
    const dir = String(t["movement"] || "").toUpperCase(); // CREDIT/DEBIT
    const info = String(t["product_info_name_method"] || "");
    const amount = toNum(t["sum_of_usd_amount"]);
    const tsISO = /T/.test(ts)
      ? parseDateUTC(ts).toISOString()
      : parseDateUTC(ts.replace(" ", "T") + "Z").toISOString();

    wallet_movements.push({
      timestamp: tsISO,
      type: type,
      status,
      movement: dir,
      product_info_name_method: info,
      amount,
    });
  }

  const debit_card_purchases = [];
  for (const t of sections.debit_card_purchases || []) {
    const ts = t["debit_card_transaction.creation_time"];
    if (!ts) continue;
    const type = String(t["debit_card_transaction.type"] || "");
    const status = String(t["debit_card_transaction.status"] || "");
    const reasonCode = String(t["debit_card_transaction.reason_code"] || "");
    const merchantName = String(
      t["debit_card_transaction.merchant_name"] || ""
    );
    const mcc = toNum(t["debit_card_transaction.merchant_category_code"]);
    const localAmount = toNum(t["debit_card_transaction.transaction_amount"]);
    const currency = toNum(t["debit_card_transaction.transaction_currency"]);
    const usdAmount = toNum(
      t["debit_card_transaction.total_accounting_usd_amount_1"]
    );
    const tsISO = /T/.test(ts)
      ? parseDateUTC(ts).toISOString()
      : parseDateUTC(ts.replace(" ", "T") + "Z").toISOString();

    debit_card_purchases.push({
      timestamp: tsISO,
      status,
      type,
      reason_code: reasonCode,
      merchant_name: merchantName,
      merchant_category_code: mcc,
      transaction_currency: currency,
      transaction_amount: localAmount,
      transaction_usd_amount: usdAmount,
    });
  }


  return {
    user_id: k["kyc_user.user_id"],
    flag,
    owner_full_name,
    user_profile: {
      status: mapped_status,
      nationality,
      account_country,
      account_age_days,
      email_matches_name: emailMatchesName(first, last, email),
    },
    links,
    signins,
    state_tokens,
    ips,
    transactions,
    repeated_counterparties,
    wallet_movements,
    debit_card_purchases,
    telesign_score,
    email_score,
  };
}

/* ─────────────────────────────── Scoring Helpers ────────────────────────────── */

const withinDays = (dt, days, ref = nowUTC()) =>
  parseDateUTC(dt) >= new Date(ref.getTime() - days * 86400000);
const withinHours = (dt, h, ref = nowUTC()) =>
  parseDateUTC(dt) >= new Date(ref.getTime() - h * 3600000);
const sumWhere = (arr, pred, pick = (x) => x) =>
  arr.reduce((s, x) => (pred(x) ? s + toNum(pick(x)) : s), 0);

function multiCountryIPsInAnyWindow(ips, hours) {
  const ms = hours * 3600 * 1000;

  const events = ips
    .map((r) => ({
      ts: parseDateUTC(r.timestamp).getTime(),
      country: String(r.country || "").toUpperCase(),
    }))
    .filter((e) => Number.isFinite(e.ts) && e.country);

  events.sort((a, b) => a.ts - b.ts);

  let i = 0;
  const freq = new Map();
  let distinct = 0;
  let maxDistinct = 0;

  for (let j = 0; j < events.length; j++) {
    const cj = events[j].country;
    const prev = freq.get(cj) || 0;
    if (prev === 0) distinct++;
    freq.set(cj, prev + 1);

    while (events[j].ts - events[i].ts > ms) {
      const ci = events[i].country;
      const cnt = freq.get(ci);
      if (cnt === 1) {
        freq.delete(ci);
        distinct--;
      } else {
        freq.set(ci, cnt - 1);
      }
      i++;
    }

    if (distinct > maxDistinct) maxDistinct = distinct;
  }

  return maxDistinct;
}

// function countImmediateOutAfterCredit(walletMovementsApproved, windowHours) {
//   const sorted = [...walletMovementsApproved].sort((a,b)=> parseDateUTC(a.timestamp) - parseDateUTC(b.timestamp));
//   console.log("sorted")
//   console.log(sorted)
//   let count = 0;
//   for (let i=0;i<sorted.length;i++){
//     const wm = sorted[i]; if (wm.movement !== "CREDIT") continue;
//     const start = parseDateUTC(wm.timestamp).getTime(); const end = start + windowHours * 3600000;
//     for (let j=i+1;j<sorted.length;j++){
//       const wmj = sorted[j]; if (wmj.movement !== "DEBIT") continue;
//       const wmjTime = parseDateUTC(wmj.timestamp).getTime();
//       if (wmjTime <= end){ count++; break; }
//       if (wmjTime > end) break;
//     }
//   }
//   return count;
// }

function countDebitMatchedToPriorCredits(wmApproved, windowHours, pct = 0.05) {
  const sorted = [...wmApproved].sort(
    (a, b) => parseDateUTC(a.timestamp) - parseDateUTC(b.timestamp)
  );

  const equalish = (a, b) => {
    const A = toNum(a),
      B = toNum(b);
    return Math.abs(A - B) <= pct * Math.abs(A);
  };

  let count = 0;

  for (let i = 0; i < sorted.length; i++) {
    const d = sorted[i];
    const dType = String(d.type || "").toUpperCase();

    if (d.movement === "DEBIT") {
      // ⛔ penalize for DEPOSIT or CASHOUT transactions
      if (dType === "DEPOSIT_TRANSACTION" || dType === "CASHOUT_TRANSACTION") {
        count--;
        continue;
      }

      // ✅ normal debit → try to match
      const debitTime = parseDateUTC(d.timestamp).getTime();
      const debitAmt = d.amount;
      const windowStart = debitTime - windowHours * 3600 * 1000;

      let matched = false;
      let sawAnyCredit = false;

      for (let j = i - 1; j >= 0; j--) {
        const w = sorted[j];
        const t = parseDateUTC(w.timestamp).getTime();

        if (t < windowStart) break; // too old
        if (w.movement === "DEBIT") break; // stop at previous debit

        if (w.movement === "CREDIT") {
          if (!sawAnyCredit) {
            sawAnyCredit = true;
            if (equalish(debitAmt, w.amount)) {
              matched = true;
              break;
            }
          } else {
            if (equalish(debitAmt, w.amount)) {
              matched = true;
              break;
            }
          }
        }
      }

      if (matched) count++;
    }
  }

  return count;
}

function stateTokenBursts(tokens, windowHours, threshold, last30d = true) {
  const now = nowUTC();
  const start = last30d ? new Date(now.getTime() - 30 * 86400000) : new Date(0);
  const times = tokens
    .map((t) => parseDateUTC(t.timestamp))
    .filter((d) => d >= start)
    .sort((a, b) => a - b);
  const q = [];
  for (const ts of times) {
    q.push(ts);
    while (q.length && ts - q[0] > windowHours * 3600000) q.shift();
    if (q.length >= threshold) return true;
  }
  return false;
}

function diffNamePercents(transfers, ownerFullName, days, accountCountry) {
  console.log("diffNamePercents");
  console.log(transfers);
  const cutoff = new Date(nowUTC().getTime() - days * 86400000);
  const filtered = transfers.filter((t) => {
    if (t.status !== "APPROVED") return false;
    if (t.type !== "TG_TRANSFER") return false;
    if (parseDateUTC(t.timestamp) < cutoff) return false;

    // Exclude special AR company docs (CUIT prefixes)
    if (
      accountCountry === "AR" &&
      t.sender_document_number &&
      /^(30|33|34)/.test(String(t.sender_document_number))
    ) {
      return false;
    }

    return true;
  });

  console.log(filtered);

  const groupSum = (dir) => {
    const m = new Map();
    for (const t of filtered) {
      if (t.direction !== dir) continue;
      inc(m, String(t.counterparty_full_name || ""), toNum(t.amount));
    }
    return m;
  };

  const creditMap = groupSum("CREDIT");
  const debitMap = groupSum("DEBIT");

  console.log("creditMap");
  console.log(creditMap);
  console.log("debitMap");
  console.log(debitMap);

  const percentDiff = (map) => {
    let total = 0,
      diff = 0;
    for (const [cp, amt] of map) {
      total += amt;
      // ⬇️ smarter comparison here:
      const same = samePersonName(ownerFullName, cp);
      console.log("same name");
      console.log(same);
      if (!same) diff += amt;
    }
    return total <= 0 ? 0 : (diff / total) * 100;
  };

  return {
    pctCreditDiff: percentDiff(creditMap),
    pctDebitDiff: percentDiff(debitMap),
  };
}

// function approvedCount(transactions, types, days) {
//   return transactions.filter((t)=> t.status==="APPROVED" && types.has(t.type) && withinDays(t.timestamp, days)).length;
// }

function approvedCount(transactions, types, days) {
  const counts = {}; // will store per-type counts

  // init all requested types with 0
  for (const t of types) counts[t] = 0;

  for (const tx of transactions) {
    if (
      tx.status === "APPROVED" &&
      types.has(tx.type) &&
      withinDays(tx.timestamp, days)
    ) {
      counts[tx.type]++; // count per type
    }
  }

  // also return the total (like your old version)
  counts.total = Object.values(counts).reduce((a, b) => a + b, 0);

  return counts;
}

function rejectedCount(transactions, types, days) {
  console.log("rejectedCount");
  console.log(
    transactions.filter(
      (t) =>
        (t.status === "REJECTED" || t.status === "CANCELLED") &&
        types.has(t.type) &&
        withinDays(t.timestamp, days)
    )
  );
  return transactions.filter(
    (t) =>
      (t.status === "REJECTED" || t.status === "CANCELLED") &&
      types.has(t.type) &&
      withinDays(t.timestamp, days)
  ).length;
}

function uniqueCounterpartiesInLastHours(transfers, hours) {
  const cutoff = new Date(nowUTC().getTime() - hours * 3600000);
  const s = new Set();
  for (const t of transfers) {
    if (t.status !== "APPROVED") continue;
    if (parseDateUTC(t.timestamp) < cutoff) continue;
    const cp = String(t.counterparty_full_name || "");
    if (cp) s.add(cp);
  }
  return s.size;
}

function stableDeviceUsage(tokens, days, maxTokens) {
  const cutoff = new Date(nowUTC().getTime() - days * 86400000);
  const cnt = tokens.reduce(
    (acc, t) => (parseDateUTC(t.timestamp) >= cutoff ? acc + 1 : acc),
    0
  );
  console.log("state tokens used in days");
  console.log(cnt);
  console.log(days);
  return cnt <= maxTokens;
}

// Sum approved incoming (CREDIT) USD in last 30 days
function sumApprovedCreditLast30d(transactions, now = new Date()) {
  const cutoff = new Date(now.getTime() - 30 * 24 * 3600 * 1000);
  let total = 0;

  for (const tx of transactions || []) {
    if (!tx) continue;
    if (String(tx.direction).toUpperCase() !== "CREDIT") continue; // incoming to the user
    if (String(tx.status).toUpperCase() !== "APPROVED") continue; // only approved
    const ts = new Date(tx.timestamp);
    if (!Number.isFinite(ts.getTime())) continue; // bad timestamp guard
    if (ts >= cutoff && ts <= now) {
      const amt = Number(tx.amount);
      if (Number.isFinite(amt)) total += amt;
    }
  }
  return total;
}

// Max approved incoming (CREDIT) USD in any 30-day rolling window within the last 90 days
function maxApprovedCreditRolling30dInLast90d(transactions, now = new Date()) {
  const DAY = 24 * 3600 * 1000;
  const start90 = new Date(now.getTime() - 90 * DAY);

  // Filter to APPROVED CREDIT txs within [start90, now]
  const credits = [];
  for (const tx of transactions || []) {
    if (!tx) continue;
    if (String(tx.direction).toUpperCase() !== "CREDIT") continue;
    if (String(tx.status).toUpperCase() !== "APPROVED") continue;

    const ts = new Date(tx.timestamp);
    if (!Number.isFinite(ts.getTime())) continue;

    if (ts >= start90 && ts <= now) {
      const amt = Number(tx.amount);
      if (Number.isFinite(amt)) credits.push({ ts, amt });
    }
  }

  if (credits.length === 0) {
    return {
      maxSum: 0,
      maxWindowStart: null,
      maxWindowEnd: null,
      totalInLast90d: 0,
    };
  }

  // Sort by timestamp for a two-pointer sliding window
  credits.sort((a, b) => a.ts - b.ts);

  let i = 0;
  let running = 0;
  let maxSum = 0;
  let maxStart = null;
  let maxEnd = null;

  // Precompute total in last 90 days
  const totalInLast90d = credits.reduce((acc, c) => acc + c.amt, 0);

  for (let j = 0; j < credits.length; j++) {
    running += credits[j].amt;

    // Maintain a 30-day window ending at credits[j].ts (inclusive)
    const cutoff = new Date(credits[j].ts.getTime() - 30 * DAY);
    while (i <= j && credits[i].ts < cutoff) {
      running -= credits[i].amt;
      i++;
    }

    if (running > maxSum) {
      maxSum = running;
      maxEnd = credits[j].ts;
      maxStart = cutoff; // window is [cutoff, maxEnd]
    }
  }

  return {
    maxSum,
    maxWindowStart: maxStart,
    maxWindowEnd: maxEnd,
    totalInLast90d,
  };
}

function isXBorderPixUser(transactions, account_country) {
  const x_border_pix_user =
    account_country.toUpperCase() !== "BR" &&
    (transactions || []).some(
      (tx) =>
        String(tx.scheme_name).toUpperCase() === "PIX" &&
        String(tx.status).toUpperCase() === "APPROVED"
    );
  return x_border_pix_user;
}

function hasPassthroughNoUsage(transactions) {
  // Params (fixed to keep original contract)
  const LOOKBACK_DAYS = 30;
  const WINDOW_HOURS = 8;
  const DRAINED_MIN = 0.9; // ≥90% of credits drained within window
  const RESIDUAL = 0.01; // treat < 1 cent as zero

  const now = nowUTC();
  const cutoff = new Date(now.getTime() - LOOKBACK_DAYS * 86400000);

  // Filter + sort
  const tx = (transactions || [])
    .filter(
      (t) => t && t.status === "APPROVED" && parseDateUTC(t.timestamp) >= cutoff
    )
    .sort((a, b) => parseDateUTC(a.timestamp) - parseDateUTC(b.timestamp));

  // Need at least one credit in window
  const hasCredit = tx.some((t) => t.direction === "CREDIT");
  if (!hasCredit) return false;

  // Premise #1: no deposit/cashout/debit card usage
  const hasUsage = tx.some((t) =>
    [
      "DEPOSIT_TRANSACTION",
      "DEBIT_CARD_TRANSACTION",
      "CASHOUT_TRANSACTION",
    ].includes(t.type)
  );
  if (hasUsage) return false;

  // Running balance + credit->debit pairing within window (FIFO)
  const windowMs = WINDOW_HOURS * 3600000;
  let balance = 0;
  let totalCredited = 0;
  let creditedDrainedWithinWindow = 0;

  // FIFO queue of credits with remaining amounts
  const credits = []; // { ts: Date, remaining: number }
  const events = []; // { ts: Date, balance: number }

  function consumeFromCredits(debitTs, amt) {
    let remaining = amt;
    for (let i = 0; i < credits.length && remaining > 0; i++) {
      const c = credits[i];
      if (c.ts > debitTs) break; // don't consume from future credits
      const take = Math.min(c.remaining, remaining);
      if (take > 0) {
        if (debitTs - c.ts <= windowMs) creditedDrainedWithinWindow += take;
        c.remaining -= take;
        remaining -= take;
        if (c.remaining <= RESIDUAL) {
          credits.splice(i, 1);
          i--;
        }
      }
    }
  }

  for (const t of tx) {
    const ts = parseDateUTC(t.timestamp);
    const amt = Number(t.amount) || 0;

    if (t.direction === "CREDIT") {
      balance += amt;
      totalCredited += amt;
      credits.push({ ts, remaining: amt });
      events.push({ ts, balance });
    } else if (t.direction === "DEBIT") {
      consumeFromCredits(ts, amt);
      balance -= amt;
      if (balance < 0 && Math.abs(balance) < RESIDUAL) balance = 0;
      events.push({ ts, balance });
    } else {
      events.push({ ts, balance });
    }
  }

  // Time the balance stays above residual (longest stretch)
  let longestAboveMs = 0;
  for (let i = 0; i < events.length; i++) {
    // find contiguous segment with balance > residual
    if (events[i].balance > RESIDUAL) {
      const start = events[i].ts;
      let j = i + 1;
      while (j < events.length && events[j].balance > RESIDUAL) j++;
      const end = j < events.length ? events[j].ts : now;
      const dt = end - start;
      if (dt > longestAboveMs) longestAboveMs = dt;
      i = j - 1;
    }
  }

  const drainedRatio =
    totalCredited > 0 ? creditedDrainedWithinWindow / totalCredited : 0;
  const lingerOK = longestAboveMs <= windowMs;

  return drainedRatio >= DRAINED_MIN && lingerOK;
}

/* ──────────────────────────────── Score Profile ─────────────────────────────── */

function scoreProfile(profile) {
  console.log("profile");
  console.log(profile);
  if (String(profile.user_id) === "26")
    return { message: "This user doesn't need to be scored" };

  const now = nowUTC();
  const txAll = profile.transactions || [];
  const tx30 = txAll.filter((t) => withinDays(t.timestamp, 30, now));
  const tx15 = txAll.filter((t) => withinDays(t.timestamp, 15, now));
  const tx7 = txAll.filter((t) => withinDays(t.timestamp, 7, now));

  const transfers30 = tx30.filter((t) => t.type === "TG_TRANSFER");
  const transfers30Approved = transfers30.filter(
    (t) => t.status === "APPROVED"
  );
  const ownerName = profile.owner_full_name || "";

  const wmAll = profile.wallet_movements || [];
  const wm30 = wmAll.filter((wm) => withinDays(wm.timestamp, 30, now));
  const wm15 = wmAll.filter((wm) => withinDays(wm.timestamp, 15, now));
  const wm7 = wmAll.filter((wm) => withinDays(wm.timestamp, 7, now));

  const walletMovements30Approved = wm30.filter((t) => t.status === "APPROVED");

  let score = 20;
  const reasons = [];
  const add = (level, text, pts) => {
    score += pts;
    reasons.push([level, text, pts]);
  };

  // HIGH
  {
    console.log("walletMovements30Approved");
    console.log(walletMovements30Approved);
    const credit = sumWhere(
      walletMovements30Approved,
      (wm) => wm.movement === "CREDIT",
      (wm) => wm.amount
    );
    const debit = sumWhere(
      walletMovements30Approved,
      (wm) => wm.movement === "DEBIT",
      (wm) => wm.amount
    );
    console.log("credit");
    console.log(credit);
    console.log("debit");
    console.log(debit);
    const ratio = debit > 0 ? credit / debit : credit > 0 ? Infinity : 0;
    console.log("ratio");
    console.log(ratio);
    const approx = ratio >= 0.9 && ratio <= 1.1;
    console.log("approx");
    console.log(approx);
    const counts = approvedCount(
      wm30,
      new Set([
        "DEPOSIT_TRANSACTION",
        "DEBIT_CARD_TRANSACTION",
        "CASHOUT_TRANSACTION",
      ]),
      30
    );

    console.log("counts");
    console.log(counts);

    const noUse = counts.total === 0;
    console.log("noUse");
    console.log(noUse);
    const xBorderPixUser = isXBorderPixUser(
      tx30,
      profile.user_profile.account_country
    );
    console.log("is pix xborder");
    console.log(xBorderPixUser);
    if (approx && noUse && !xBorderPixUser)
      add(
        "HIGH",
        `Credit/Debit ratio ≈ 1 (ratio=${ratio.toFixed(
          2
        )}) and no approved deposit/debit card/cashout in 30d`,
        30
      );
  }
  // {
  //   const { pctCreditDiffName, pctDebitDiffName } = diffNamePercents(
  //     txAll,
  //     ownerName,
  //     30,
  //     profile.user_profile.account_country
  //   );
  //   console.log("pctCreditDiffName");
  //   console.log(pctCreditDiffName);
  //   console.log("pctDebitDiffName");
  //   console.log(pctDebitDiffName);
  //   console.log("length repeated cunterparties");
  //   console.log(profile.repeated_counterparties.length);
  //   if (
  //     pctCreditDiffName > 75 &&
  //     (!profile.repeated_counterparties ||
  //       profile.repeated_counterparties.length < 1)
  //   )
  //     add(
  //       "HIGH",
  //       `CREDIT Counterparties different-name share >75% (credit=${pctCreditDiffName.toFixed(
  //         1
  //       )}%)`,
  //       30
  //     );
  // }
  {
    const c7 = countDebitMatchedToPriorCredits(
      walletMovements30Approved,
      24 * 7
    );
    const c15 = countDebitMatchedToPriorCredits(
      walletMovements30Approved,
      24 * 15
    );
    const c30 = countDebitMatchedToPriorCredits(
      walletMovements30Approved,
      24 * 30
    );
    if (c7 >= 5)
      add(
        "HIGH",
        `Immediate transfers out after credits (<=7d) count=${c7}`,
        25
      );
    else if (c15 >= 5 || c30 >= 10)
      add("HIGH", `Immediate transfers out (15d=${c15}, 30d=${c30})`, 15);
    console.log("c7");
    console.log(c7);
    console.log("c15");
    console.log(c15);
    console.log("c30");
    console.log(c30);
  }
  // {
  //   const rejDC30 = rejectedCount(wm30, new Set(["DEBIT_CARD_TRANSACTION"]), 30);
  //   if (rejDC30 >= 10) add("HIGH", `10+ rejected debit card txns in 30d (${rejDC30})`, 15);
  //   console.log("rejDC30")
  //   console.log(rejDC30)
  // }
  {
    const blockedViaStateToken = toNum(
      profile?.links?.state_token?.blocked_linked_users_count || 0
    );
    console.log("blockedViaState");
    console.log(blockedViaStateToken);
    console.log(profile);
    if (blockedViaStateToken >= 5)
      add(
        "HIGH",
        `Linked with 5+ BLOCKED users via state token (${blockedViaStateToken})`,
        30
      );
    else if (blockedViaStateToken >= 1)
      add(
        "HIGH",
        `Linked with 1+ BLOCKED user(s) via state token (${blockedViaStateToken})`,
        20
      );
  }
  {
    const status = String(profile?.user_profile?.status || "").toUpperCase();
    if (status === "BLOCKED") add("HIGH", "User status is BLOCKED", 60);
  }
  {
    const nat = String(profile?.user_profile?.nationality || "").toUpperCase();
    if (nat === "RU") add("HIGH", "User of Russian nationality", 25);
  }
  {
    const distinctIps = multiCountryIPsInAnyWindow(profile.ips || [], 24);
    if (distinctIps == 2)
      add("HIGH", "IPs from 2 countries within any 24h window", 10);
    else if (distinctIps > 2)
      add("HIGH", "IPs from 3 countries within any 24h window", 25);
  }

  // MEDIUM
  {
    const linkedStateToken = toNum(
      profile?.links?.state_token?.linked_users_count || 0
    );
    console.log("linkedStateToken");
    console.log(linkedStateToken);
    if (linkedStateToken >= 8)
      add("MEDIUM", `Linked via state token: ${linkedStateToken} >= 8`, 30);
    else if (linkedStateToken >= 5)
      add("MEDIUM", `Linked via state token: ${linkedStateToken} >= 5`, 20);
    else if (linkedStateToken >= 3)
      add("MEDIUM", `Linked via state token: ${linkedStateToken} >= 3`, 15);
    else if (linkedStateToken >= 1)
      add("MEDIUM", `Linked via state token: ${linkedStateToken} >= 1`, 10);
  }
  // {
  //   const linkedTG = toNum(profile?.links?.transfer_gateway?.linked_users_count || 0);
  //   if (linkedTG >= 5) add("MEDIUM", `Linked via transfer gateway: ${linkedTG} >= 5`, 20);
  //   else if (linkedTG >= 2) add("MEDIUM", `Linked via transfer gateway: ${linkedTG} >= 2`, 10);
  //   else if (linkedTG >= 1) add("MEDIUM", `Linked via transfer gateway: ${linkedTG} >= 1`, 5);
  // }
  {
    const linkedPP = toNum(
      profile?.links?.payin_payout?.linked_users_count || 0
    );
    if (linkedPP >= 5)
      add("MEDIUM", `Linked via payin/payout: ${linkedPP} >= 5`, 20);
    else if (linkedPP >= 2)
      add("MEDIUM", `Linked via payin/payout: ${linkedPP} >= 2`, 15);
    else if (linkedPP >= 1)
      add("MEDIUM", `Linked via payin/payout: ${linkedPP} >= 1`, 10);
  }
  // {
  //   const blockedViaLinks =
  //     toNum(profile?.links?.payin_payout?.blocked_linked_users_count || 0) +
  //     toNum(profile?.links?.transfer_gateway?.blocked_linked_users_count || 0);
  //   if (blockedViaLinks >= 5) add("MEDIUM", `Linked with 5+ BLOCKED via transfer/payin/payout (${blockedViaLinks})`, 25);
  //   else if (blockedViaLinks >= 2) add("MEDIUM", `Linked with 2+ BLOCKED via transfer/payin/payout (${blockedViaLinks})`, 15);
  //   else if (blockedViaLinks >= 1) add("MEDIUM", `Linked with BLOCKED via transfer/payin/payout (${blockedViaLinks})`, 10);
  // }
  console.log("tx7");
  console.log(tx7);
  if (rejectedCount(tx7, new Set(["WITHDRAWAL_TRANSACTION"]), 7) >= 3)
    add("MEDIUM", "3+ rejected withdrawal transactions (7d)", 10);
  if (rejectedCount(tx7, new Set(["PURCHASE"]), 7) >= 3)
    add("MEDIUM", "3+ rejected top ups transactions (7d)", 10);
  if (rejectedCount(tx7, new Set(["CASHOUT_TRANSACTION"]), 7) >= 3)
    add("MEDIUM", "3+ rejected debit card transactions (7d)", 5);

  {
    const signInFailed7 = (profile.sign_in || []).filter(
      (s) => !s.success && withinDays(s.timestamp, 7)
    ).length;
    console.log(signInFailed7);
    console.log("signInFailed7");
    if (signInFailed7 >= 3) add("MEDIUM", "3+ failed sign-ins (7d)", 10);
  }
  {
    const uniq12 = uniqueCounterpartiesInLastHours(tx30, 12);
    const uniq24 = uniqueCounterpartiesInLastHours(tx30, 24);
    console.log("uniq12");
    console.log(uniq12);
    console.log("uniq24");
    console.log(uniq24);
    if (uniq24 > 20)
      add(
        "MEDIUM",
        `Unique counterparties via tg_transfer >20 in <24h (${uniq24})`,
        35
      );
    else if (uniq12 > 10)
      add(
        "MEDIUM",
        `Unique counterparties via tg_transfer >10 in <12h (${uniq12})`,
        25
      );
  }

  if (stateTokenBursts(profile.state_tokens || [], 12, 3))
    add("MEDIUM", "3+ state tokens in <12h (last 30d)", 20);
  if (stateTokenBursts(profile.state_tokens || [], 24, 5))
    add("MEDIUM", "5+ state tokens in <24h (last 30d)", 25);

  if (profile.telesign_score != null) {
    const ts = Number(profile.telesign_score);
    if (ts > 700) add("MEDIUM", `Telesign score > 700 (${ts})`, 20);
    else if (ts >= 500)
      add("MEDIUM", `Telesign score between 500 and 700 (${ts})`, 10);
  }
  if (profile.email_score != null) {
    const es = Number(profile.email_score);
    if (es > 700) add("MEDIUM", `Email score > 700 (${es})`, 20);
    else if (es >= 500)
      add("MEDIUM", `Email score between 500 and 700 (${es})`, 10);
  }

  {
    // const countDevicesInLastH = (h) => {
    //   const cutoff = new Date(now.getTime() - h*3600000);
    //   const set = new Set();
    //   for (const s of profile.signins || []) {
    //     console.log("s dentro de progile.signins")
    //     console.log(s)
    //     console.log("parseutc")
    //     console.log(parseDateUTC(s.timestamp))
    //     console.log("cutoff")
    //     console.log(cutoff)
    //     console.log("timestamp vs cutoff")
    //     console.log(parseDateUTC(s.timestamp) >= cutoff)
    //     console.log("device id")
    //     console.log(s.device_id)
    //     if (parseDateUTC(s.timestamp) >= cutoff && s.device_id) set.add(s.device_id);
    //   }
    //   console.log("set de sign in")
    //   console.log(set)
    //   return set.size;
    // };

    // if (countDevicesInLastH(24*7) >= 3) add("MEDIUM", "Sign-in from 3+ devices in 7 days", 30);
    // else if (countDevicesInLastH(48) >= 2) add("MEDIUM", "Sign-in from 2+ devices in 48h", 20);

    const countSigninsRolling7d15d30d60d = () => {
      const nowMs = now.getTime();
      const start60 = nowMs - 60 * 24 * 3600 * 1000;
      const W48 = 2 * 24 * 3600 * 1000;
      const W7 = 7 * 24 * 3600 * 1000;
      const W15 = 15 * 24 * 3600 * 1000;
      const W30 = 30 * 24 * 3600 * 1000;
      const W60 = 60 * 24 * 3600 * 1000;

      // Normaliza, filtra últimos 60d y descarta sign-ins sin device_id
      const evts = (profile.signins || [])
        .map((s) => ({
          t: parseDateUTC(s.timestamp)?.getTime(),
          d: s.device_id,
        }))
        .filter(
          (e) => Number.isFinite(e.t) && e.t >= start60 && e.t <= nowMs && e.d
        )
        .sort((a, b) => a.t - b.t);

      // Utilidades para llevar el multiset (conteo por device)
      const inc = (m, k) => m.set(k, (m.get(k) || 0) + 1);
      const dec = (m, k) => {
        const c = m.get(k);
        if (c === 1) m.delete(k);
        else if (c > 1) m.set(k, c - 1);
      };

      const count48h = [];
      const count7d = [];
      const count15d = [];
      const count30d = [];
      const count60d = [];
      const timesISO = [];

      // Cabezas y mapas para cada ventana
      let h48 = 0,
        h7 = 0,
        h15 = 0,
        h30 = 0,
        h60 = 0;
      const m48 = new Map(); // device_id -> freq en ventana 48h
      const m7 = new Map(); // device_id -> freq en ventana 7d
      const m15 = new Map(); // device_id -> freq en ventana 15d
      const m30 = new Map(); // device_id -> freq en ventana 30d
      const m60 = new Map(); // device_id -> freq en ventana 60d

      for (let i = 0; i < evts.length; i++) {
        const { t, d } = evts[i];

        // Añadimos el evento actual a ambas ventanas
        inc(m48, d);
        inc(m7, d);
        inc(m15, d);
        inc(m30, d);
        inc(m60, d);

        // Deslizamos cabezas para mantener ventanas [t - W, t]
        while (t - evts[h48].t > W48) {
          dec(m7, evts[h48].d);
          h48++;
        }
        while (t - evts[h7].t > W7) {
          dec(m7, evts[h7].d);
          h7++;
        }
        while (t - evts[h15].t > W15) {
          dec(m15, evts[h15].d);
          h15++;
        }
        while (t - evts[h30].t > W30) {
          dec(m30, evts[h30].d);
          h30++;
        }
        while (t - evts[h60].t > W60) {
          dec(m60, evts[h60].d);
          h60++;
        }

        // El "conteo" es la cantidad de device_ids únicos (tamaño del mapa)
        count48h.push(m48.size);
        count7d.push(m7.size);
        count15d.push(m15.size);
        count30d.push(m30.size);
        count60d.push(m60.size);
        timesISO.push(new Date(t).toISOString());
      }

      return {
        timesISO,
        count48h,
        count7d,
        count15d,
        count30d,
        count60d,
        max48h: count7d.length ? Math.max(...count48h) : 0,
        max7d: count7d.length ? Math.max(...count7d) : 0,
        max15d: count15d.length ? Math.max(...count15d) : 0,
        max30d: count30d.length ? Math.max(...count30d) : 0,
        max60d: count60d.length ? Math.max(...count60d) : 0,
      };
    };
    console.log("QUE HAGO ACAAAA");
    const result = countSigninsRolling7d15d30d60d();
    console.log(result);
    if (result.max48h >= 2)
      add("MEDIUM", "Sign-in from 2+ devices in 48 hs", 40);
    else if (result.max7d >= 2)
      add("MEDIUM", "Sign-in from 2+ devices in 7 days", 30);
    else if (result.max15d >= 2)
      add("MEDIUM", "Sign-in from 2+ devices in 15 days", 20);
    else if (result.max30d >= 2)
      add("MEDIUM", "Sign-in from 2+ devices in 30 days", 15);
    else if (result.max60d >= 2)
      add("MEDIUM", "Sign-in from 2+ devices in 60 days", 10);
  }
  {
    const passthroughNoUsage = hasPassthroughNoUsage(tx30);
    console.log("passthroughNoUsage");
    console.log(passthroughNoUsage);
    if (passthroughNoUsage) add("MEDIUM", "Passthrough behavior", 20);
  }
  {
    const acctCountry = String(
      profile?.user_profile?.account_country || ""
    ).toUpperCase();
    const nat = String(profile?.user_profile?.nationality || "").toUpperCase();
    if (EU_CC.has(acctCountry) && AFRICA_CC.has(nat))
      add("MEDIUM", "European account country with African nationality", 15);
    if (acctCountry && nat && acctCountry !== nat)
      add("MEDIUM", "Nationality differs from account country", 10);
  }

  {
    const p2pInternal = transfers30.filter(
      (t) => t.scheme_name === "P2P Internal"
    );
    const africanDiff = p2pInternal.some(
      (t) =>
        AFRICA_CC.has(String(t.sender_country || "").toUpperCase()) &&
        AFRICA_CC.has(String(t.receiver_country || "").toUpperCase()) &&
        String(t.sender_country || "").toUpperCase() !==
          String(t.receiver_country || "").toUpperCase()
    );
    if (africanDiff)
      add("MEDIUM", "P2P Internal between different African countries", 25);
  }

  // LOW (negative)

  if (!profile?.user_profile?.email_matches_name)
    add("LOW", "Email pattern does not match user name", 10);
  {
    const age = toNum(profile?.user_profile?.account_age_days || 0);
    if (age < 7) add("LOW", `Account age < 7 days (${age})`, 25);
    else if (age < 15) add("LOW", `Account age < 15 days (${age})`, 15);
    else if (age < 30) add("LOW", `Account age < 30 days (${age})`, 10);
  }
  // const totalIn30d = sumApprovedCreditLast30d(profile.transactions, new Date());
  const totalIn30d = maxApprovedCreditRolling30dInLast90d(
    profile.transactions,
    new Date()
  );
  console.log("rolling windows");
  console.log(totalIn30d);
  if (totalIn30d.maxSum < 300) {
    add(
      "LOW",
      `Approved incoming in last 90 days on 30 days rolling window: $${totalIn30d.maxSum.toFixed(
        2
      )} < $300`,
      -10
    );
  }
  {
    const tokens = profile.state_tokens || [];
    if (stableDeviceUsage(tokens, 15, 2))
      add("LOW", "Stable device usage: 2 or fewer state tokens in 15d", -15);
    else if (stableDeviceUsage(tokens, 15, 4))
      add("LOW", "Stable device usage: 4 or fewer state tokens in 15d", -5);
  }
  {
    const { pctCreditDiff, pctDebitDiff } = diffNamePercents(
      transfers30,
      ownerName,
      30,
      profile.user_profile.account_country
    );
    console.log("diff name percents results:");
    console.log("pctCreditDiff");
    console.log(pctCreditDiff);
    console.log("pctDebitDiff");
    console.log(pctDebitDiff);

    const transfers30Credit = tx30.filter(
      (t) => t.type === "TG_TRANSFER" && t.direction === "CREDIT"
    );
    console.log("transfers30Credit");
    console.log(transfers30Credit);

    if (pctCreditDiff == 0 && transfers30Credit.length > 0)
      add("LOW", "100% of CREDIT from same name as account holder", -35);
    else if (pctCreditDiff <= 25 && transfers30Credit.length > 0)
      add("LOW", "75%+ of CREDIT from same name as account holder", -30);
    else if (pctCreditDiff <= 50 && transfers30Credit.length > 0)
      add("LOW", "50%+ of CREDIT to same name as account holder", -20);
    if (pctDebitDiff <= 25 && pctDebitDiff != 0)
      add("LOW", "75%+ of DEBIT from same name as account holder", -30);
    else if (pctDebitDiff <= 50 && pctDebitDiff != 0)
      add("LOW", "50%+ of DEBIT to same name as account holder", -20);
  }
  {
    const depCash15 = approvedCount(
      tx15,
      new Set(["DEPOSIT_TRANSACTION", "CASHOUT_TRANSACTION"]),
      15
    );
    const depCash30 = approvedCount(
      tx30,
      new Set(["DEPOSIT_TRANSACTION", "CASHOUT_TRANSACTION"]),
      30
    );
    if (depCash15 >= 5)
      add("LOW", "5+ approved deposits or cashouts in 15d", -10);
    else if (depCash30 >= 15)
      add("LOW", "15+ approved deposits or cashouts in 30d", -15);
  }
  {
    const dc7 = approvedCount(tx7, new Set(["DEBIT_CARD_TRANSACTION"]), 7);
    const dc15 = approvedCount(tx15, new Set(["DEBIT_CARD_TRANSACTION"]), 15);
    if (dc7 >= 5) add("LOW", "5+ approved debit card transactions in 7d", -10);
    else if (dc15 >= 15)
      add("LOW", "15+ approved debit card transactions in 15d", -15);
  }
  {
    const byDayCredit = new Map(),
      byDayDebit = new Map();
    for (const t of transfers30Approved) {
      const day = parseDateUTC(t.timestamp).toISOString().slice(0, 10);
      if (t.direction === "CREDIT") inc(byDayCredit, day, toNum(t.amount));
      if (t.direction === "DEBIT") inc(byDayDebit, day, toNum(t.amount));
    }
    const creditDays = [...byDayCredit.keys()];
    if (creditDays.length) {
      let notFull = 0;
      for (const d of creditDays) {
        const cAmt = byDayCredit.get(d) || 0;
        const dAmt = byDayDebit.get(d) || 0;
        if (dAmt < 0.8 * cAmt) notFull++;
      }
      if (notFull / creditDays.length >= 0.6)
        add(
          "LOW",
          "Credits not fully drained same day; debits spread over time",
          -15
        );
    }
  }

  // BONUS & CAP
  const medHighCount = reasons.filter(
    ([lvl]) => lvl === "MEDIUM" || lvl === "HIGH"
  ).length;
  if (medHighCount >= 3)
    add("BONUS", "3 or more MEDIUM/HIGH signals triggered", 20);

  const highCount = reasons.filter(([lvl]) => lvl === "HIGH").length;
  const cap = highCount >= 4 ? 100 : 95;

  const final_score = Math.max(1, Math.min(Math.round(score), cap));
  const risk_score = final_score;

  const top5 = [...reasons]
    .sort((a, b) => Math.abs(b[2]) - Math.abs(a[2]))
    .slice(0, 5);
  const top5Text = top5.map(
    ([lvl, txt, pts]) => `[${lvl}]: ${txt} (${pts >= 0 ? "+" : ""}${pts})`
  );

  let running = 20;
  const calc = ["Base score: 20"];
  for (const [, txt, pts] of reasons) {
    calc.push(`${pts >= 0 ? "+" : ""}${pts} ${txt}`);
    running += pts;
  }
  calc.push(
    `= ${running.toFixed(
      2
    )} before cap; high_signals=${highCount}, medium_signals=${
      reasons.filter(([l]) => l === "MEDIUM").length
    }`
  );
  calc.push(`Cap applied: ${cap}`);
  calc.push(`final_score = ${final_score}`);

  // ⬇️ reasoning now includes the full reasons array (as-is) plus context
  const reasoning = {
    summary: `Final computed risk_score: ${final_score}`,
    top5_text: top5Text,
    calculations: calc,
    reasons_normalized: reasons.map(([level, text, points]) => ({
      level,
      text,
      points,
    })),
  };

  return {
    user_id: String(profile.user_id || ""),
    reasoning, // now an object with the full detail
    risk_assessment: {
      agent_version: AGENT_VERSION,
      risk_score,
      reasons: top5Text, // still only Top 5 here
    },
  };
}

/* ─────────────────────────────── Helpers (multi-user) ───────────────────────── */

function normalizeToDataset(inputJsons) {
  // Caso A: un item con array
  if (inputJsons.length === 1 && Array.isArray(inputJsons[0]))
    return inputJsons[0];
  // Caso B: varios items objeto
  if (
    inputJsons.length >= 1 &&
    inputJsons.every((x) => x && typeof x === "object" && !Array.isArray(x))
  )
    return inputJsons;
  throw new Error(
    "Unsupported input. Provide: (1) single item with array of records, or (2) multiple items, each a record object."
  );
}

function hasAnyUserId(dataset) {
  return dataset.some(
    (r) =>
      Object.prototype.hasOwnProperty.call(r, "user_id") &&
      r.user_id != null &&
      String(r.user_id).trim() !== ""
  );
}

/**
 * Agrupa registros por user_id y fusiona secciones homónimas concatenando arrays.
 * Devuelve un Map<userId, sectionsArray> listo para parseToProfile()
 */
function groupSectionsByUserId(dataset) {
  const byUser = new Map(); // userId -> { sectionKey: [] }

  for (const rec of dataset) {
    const rawUid = rec.user_id;
    // ignorar si viene como null, undefined, string vacía o string "null"
    if (rawUid == null) continue;
    const uid = String(rawUid).trim();
    if (!uid || uid.toLowerCase() === "null") continue;

    if (!byUser.has(uid)) byUser.set(uid, {});

    const bucket = byUser.get(uid);
    for (const [k, v] of Object.entries(rec)) {
      if (k === "user_id") continue;
      if (!Array.isArray(v)) continue; // solo esperamos arrays por sección
      if (!bucket[k]) bucket[k] = [];
      bucket[k].push(...v);
    }
  }

  // Transformamos cada bucket en [{section: [...]}, ...]
  const result = new Map();
  for (const [uid, sectionsObj] of byUser.entries()) {
    const sectionsArray = Object.entries(sectionsObj).map(([k, v]) => ({
      [k]: v,
    }));
    result.set(uid, sectionsArray);
  }
  return result;
}

/* ─────────────────────────────── n8n Entrypoint ─────────────────────────────── */

const inputJsons = $input.all().map((i) => i.json);
const dataset = normalizeToDataset(inputJsons);

const outputs = [];

// Camino multi-user si detectamos user_id en los registros
if (hasAnyUserId(dataset)) {
  const grouped = groupSectionsByUserId(dataset);
  console.log("entro al hasAnyUserId");
  console.log("grouped");
  console.log(grouped);
  // Para cada user_id: construir perfil -> score o error si no hay KYC
  for (const [uid, sectionsArray] of grouped.entries()) {
    try {
      console.log("sectionsArray");
      console.log(sectionsArray);
      const profile = parseToProfile(sectionsArray);
      const scored = scoreProfile(profile);
      const flag = profile.flag || null; // viene de parseToProfile
      // Asegurar user_id en salida
      outputs.push({ json: { user_id: String(uid), flag, ...scored } });
    } catch (e) {
      outputs.push({
        json: { user_id: String(uid), error: String(e.message || e) },
      });
    }
  }

  // Si algún registro venía con user_id nulo/ vacío, lo ignoramos silenciosamente.
  // (Opcional) También podrías empujar un error adicional si quieres registrar esos casos.
} else {
  console.log("entro al else de hasAnyUserId");
  // Compat: no hay user_id -> usar todo el dataset como un solo perfil
  try {
    const profile = parseToProfile(dataset);
    const scored = scoreProfile(profile);
    outputs.push({ json: scored });
  } catch (e) {
    outputs.push({ json: { error: String(e.message || e) } });
  }
}

return outputs;

```

---

