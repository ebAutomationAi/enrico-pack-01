# SETUP — Pack #1 Funnel (Google Form → Google Sheets → SMTP) — n8n self-hosted (Docker/Windows)

Este pack implementa un funnel de captura y entrega con:
- Captación: Google Form → escribe en Google Sheets (tab “Respuestas de formulario 1”)
- Automatización: n8n (self-hosted)
- Envío: SMTP (Gmail) con App Password
- Logging + anti-reenvío en la misma fila del Sheet

## 0) Requisitos previos

### Infra
- n8n self-hosted corriendo (Docker) y accesible:
  - Local: `http://localhost:5678`
  - LAN (opcional): `http://<IP_LAN>:5678`
- Zona horaria: Europe/Madrid (recomendado)

### Cuentas Google
- Una cuenta Google con acceso al Spreadsheet (para Google Sheets Trigger + Updates).
- Una cuenta Gmail emisora para SMTP:
  - **Obligatorio**: tener **Verificación en 2 pasos** activada
  - **Obligatorio**: crear una **App Password** para “Mail” (o “n8n”)

---

## 1) Google Sheet — esquema y preparación

### 1.1 Tab y cabeceras
Tab (pestaña) esperada: **Respuestas de formulario 1**

Cabeceras exactas (fila 1). Recomendado este orden (puedes reordenar columnas, pero los nombres deben coincidir):

- `row_key`
- `Marca temporal`
- `Dirección de correo electrónico`
- `Nombre`
- `Objetivo`
- `sent`
- `sent_at`
- `status`
- `status_detail`
- `err_msg`
- `gmail_id`
- `gmail_threadId`

Notas:
- `status_detail` es opcional pero recomendado.
- En SMTP-only, `gmail_id` y `gmail_threadId` quedan vacíos (se mantienen por compatibilidad).

### 1.2 row_key (clave estable para Update Row)
Usamos `row_key` como clave estable para hacer Update Row (evita problemas de `row_number` nulo).

Formato:
YYYY-MM-DD HH:mm:ss|email

css
Copiar código

#### Opción recomendada: ARRAYFORMULA para rellenar row_key automáticamente
1. En la celda **A1** pon el header `row_key`.
2. En la celda **A2** pega esta fórmula (ajusta las letras de columna si tu hoja no coincide):
   - Asumiendo:
     - `Marca temporal` está en **B**
     - `Dirección de correo electrónico` está en **C**

```gs
=ARRAYFORMULA(
  IF(
    B2:B="",
    "",
    TEXT(B2:B,"yyyy-mm-dd hh:mm:ss") & "|" & LOWER(TRIM(REGEXREPLACE(C2:C, "\r|\n", "")))
  )
)
Con esto:

row_key se rellena solo en filas nuevas.

Normaliza email (lower + trim) y elimina saltos de línea.

1.3 Columnas de logging (formatos)
sent: Boolean (TRUE/FALSE) o vacío

sent_at: texto timestamp (recomendado YYYY-MM-DD HH:mm:ss)

status: OK_SENT o ERROR

status_detail: OK_SENT, E_MISSING_EMAIL, E_SMTP_SEND

err_msg: vacío o texto con código + detalle

gmail_id, gmail_threadId: vacío en SMTP-only

2) SMTP (Gmail) — credenciales (App Password)
2.1 Crear App Password (Gmail emisor)
Entra en la cuenta emisora (ej. eb.automation.ai@gmail.com)

Google Account → Security

Activa 2-Step Verification

Busca App passwords

Crea una App Password (por ejemplo: “n8n SMTP”)

Copia la contraseña (se muestra una vez)

2.2 SMTP settings (Gmail)
Usaremos:

Host: smtp.gmail.com

Port recomendado: 465 (SSL)

Usuario: tu Gmail completo (ej. eb.automation.ai@gmail.com)

Password: App Password (NO tu contraseña normal)

Client Host Name (si te lo pide): localhost

3) Importar el workflow en n8n
3.1 Importación
En n8n → Workflows

Import from File (o Import)

Selecciona el JSON:

workflows/pack-01-funnel-googleform-sheets-gmail.json

3.2 Publicar (activación en n8n 2.x)
En n8n 2.x, el workflow corre en producción cuando está Published.

Abre el workflow

Arriba a la derecha → Publish

(Opcional) Pon “Version name” y “Describe changes”

Confirmar Publish

4) Configuración nodo por nodo (obligatorio)
Importante: en todos los nodos de Google Sheets debes seleccionar el MISMO Spreadsheet y la MISMA Sheet.

4.1 Google Sheets Trigger
Nodo: Google Sheets Trigger

Trigger On: Row added

Spreadsheet: selecciona tu spreadsheet

Sheet: Respuestas de formulario 1

Polling interval: 1 min (recomendado)

Credenciales:

En el nodo → sección Credentials

Selecciona tu credencial Google Sheets (OAuth)

Si no existe:

Create new → conectar con tu cuenta Google → Allow

4.2 IF Pendiente (anti-reenvío)
Nodo: IF Pendiente
Condición (Expression):

js
Copiar código
{{ (($json.sent ?? "").toString().replace(/\r?\n/g, "").trim().toLowerCase() !== "true") }}
4.3 IF Email válido
Nodo: IF Email válido
Condición (Expression):

js
Copiar código
{{ ($json["Dirección de correo electrónico"] ?? "").toString().replace(/\r?\n/g, "").trim() !== "" }}
FALSE branch (email vacío) irá al Update ERROR.

4.4 Send email (SMTP)
Nodo: SMTP → Send (o “Send Email” vía SMTP)

Campos:

From Email:

Recomendado (para canal soporte sin Reply-To):

Enrico (Soporte) <eb.automation.ai+soporte@gmail.com>

To Email (Expression):

js
Copiar código
{{ ($json["Dirección de correo electrónico"] ?? "").toString().replace(/\r?\n/g, "").trim() }}
Subject:

Pack #1 (GRATIS) — Plantillas n8n + enlace de descarga

Body:

usa el body que has definido (TEXT o HTML)

Options:

Append n8n Attribution: OFF (recomendado)

(Si existe en tu nodo) Settings del nodo:

Continue On Fail: ON (para que el flujo pueda loguear errores en Sheets)

Credenciales SMTP:

Create new credential:

Host: smtp.gmail.com

Port: 465

Secure/SSL: ON

User: eb.automation.ai@gmail.com

Password: App Password

Client Host Name: localhost

4.5 Merge (Combine by Position)
Nodo: Merge

Mode: Combine by Position
Conexiones (crítico):

Merge Input 1 ← salida de Send email (SMTP)

Merge Input 2 ← salida TRUE de IF Email válido (row data)

4.6 Edit Fields / Set (pass-through)
Nodo: Edit Fields (Set)

Include other fields / pass-through: ON

Añade/ajusta campos (Expression):

sent

js
Copiar código
{{ $json.error ? "" : true }}
sent_at (formato legible)

js
Copiar código
{{ $json.error ? "" : $now.setZone('Europe/Madrid').toFormat('yyyy-LL-dd HH:mm:ss') }}
status

js
Copiar código
{{ $json.error ? "ERROR" : "OK_SENT" }}
status_detail

js
Copiar código
{{ $json.error ? "E_SMTP_SEND" : "OK_SENT" }}
err_msg

js
Copiar código
{{
  $json.error
    ? ("E_SMTP_SEND | " + (
        $json.error?.message
        ?? $json.message
        ?? "SMTP send failed"
      ))
    : ""
}}
gmail_id

js
Copiar código
{{ "" }}
gmail_threadId

js
Copiar código
{{ "" }}
4.7 Update row in sheet (principal)
Nodo: Google Sheets → Update Row

Spreadsheet: el mismo

Sheet: Respuestas de formulario 1

Match:

“Using to match”: row_key

Valor (Expression sanitizada):

js
Copiar código
{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}
Mapeo:

Recomendado: Map Automatically

Si usas manual: actualiza estos campos:

sent, sent_at, status, status_detail, err_msg, gmail_id, gmail_threadId

4.8 Update row — ERROR Missing email (rama FALSE)
Nodo: Google Sheets → Update Row (ERROR)
Match por row_key (misma expresión sanitizada).

Valores:

sent = ""

sent_at = ""

status = "ERROR"

status_detail = "E_MISSING_EMAIL"

err_msg = "E_MISSING_EMAIL | Missing email"

gmail_id = ""

gmail_threadId = ""

5) Testing checklist (end-to-end)
5.1 Caso A — Éxito (email válido)
Envía un Google Form con email válido.

Espera hasta 90s (polling).

Confirma que llega 1 correo.

En Google Sheets, en esa fila:

sent = TRUE

sent_at = YYYY-MM-DD HH:mm:ss

status = OK_SENT

status_detail = OK_SENT

err_msg vacío

5.2 Caso B — Error (Missing email)
Inserta una fila manual con email vacío o con un espacio " "

Confirma que NO se envía email.

En el sheet:

status = ERROR

status_detail = E_MISSING_EMAIL

err_msg = E_MISSING_EMAIL | Missing email

5.3 Caso C — Anti-reenvío
Repite el trigger sobre una fila ya enviada (sent TRUE).

Debe bloquearse en IF Pendiente (no reenvía).

6) Publicación para distribución (GitHub Release)
Tu ZIP debe incluir:

README.md

docs/SETUP.md

docs/SHEET_SCHEMA.md

docs/TROUBLESHOOTING.md

workflows/pack-01-funnel-googleform-sheets-gmail.json

Y el enlace de descarga recomendado:

.../releases/latest/download/enrico-pack-01.zip

yaml
Copiar código
