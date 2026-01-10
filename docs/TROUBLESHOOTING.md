# TROUBLESHOOTING — Pack #1 Funnel (Google Form → Google Sheets → SMTP)

## 1) No hay ejecuciones / el trigger no dispara
**Síntoma:** Google Sheets Trigger no ejecuta, “última ejecución” no cambia.

Checklist:
1. El workflow debe estar **Published** (n8n 2.x).
2. En el Google Sheets Trigger:
   - Trigger On = Row added
   - Spreadsheet y Sheet correctos (misma pestaña real)
   - Polling interval (ej. 1 min)
3. Confirma que el Form está escribiendo en **esa** sheet (Respuestas de formulario 1).
4. En n8n → Executions:
   - filtra por ese workflow y confirma si hay ejecuciones fallidas.

Solución típica:
- Re-selecciona Spreadsheet/Sheet en el Trigger (a veces queda “pegado” tras importar).
- Guarda y vuelve a Publish.

---

## 2) Se ejecuta pero NO actualiza la fila del Sheet
**Síntoma:** el flujo llega a Update Row pero la fila queda igual.

Causas más comunes:
1. `row_key` no coincide (espacios invisibles o saltos de línea).
2. `row_key` no está relleno (ARRAYFORMULA no aplicada o columnas cambiadas).
3. Estás apuntando a otro Spreadsheet/Sheet en el Update Row.

Solución:
1. Asegura `row_key` en el Sheet y que se genera para filas nuevas.
2. En Update Row (match), usa sanitizado:

```js
{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}
Verifica que todos los nodos Google Sheets apuntan al mismo Spreadsheet + tab.

3) “Missing email” aunque el formulario tenía email
Síntoma: rama FALSE del IF Email válido o status_detail=E_MISSING_EMAIL en una fila con email.

Causas:

La columna Dirección de correo electrónico tiene espacios o saltos de línea.

La columna real tiene otro nombre (cabecera distinta / idioma distinto).

Solución:

Confirma que la cabecera en la fila 1 se llama exactamente:

Dirección de correo electrónico

Usa expresión robusta en el IF:

js
Copiar código
{{ ($json["Dirección de correo electrónico"] ?? "").toString().replace(/\r?\n/g, "").trim() !== "" }}
4) El email se envía pero el Sheet marca ERROR
Síntoma: llegó correo pero status=ERROR.

Causa:

El nodo SMTP lanzó error y aun así el correo llegó (raro), o el “Continue On Fail” hace que el flujo siga con un objeto error.

Solución:

Revisa output real del nodo SMTP en ejecución.

Mantén lógica SMTP en Set/Edit Fields (éxito = no hay $json.error).

Evita mezclar lógica Gmail (id/threadId) con SMTP.

5) Se envían 2 correos (duplicado)
Síntoma: llegan dos emails por una misma fila.

Causa típica:

Tienes 2 nodos de envío activos (SMTP + Gmail) o ramas duplicadas.

Solución definitiva:

Deja solo un nodo de envío (SMTP) en el camino TRUE de “Email válido”.

Elimina o deshabilita el nodo Gmail “Send a message”.

Estructura correcta:

IF Email válido (TRUE) → SMTP → Merge → Set → Update principal

6) No puedo poner Reply-To en SMTP (no aparece)
Síntoma: en Options solo ves:

Attachments

CC

BCC

Ignore SSL Issues

Causa:

Tu nodo SMTP en esta versión no expone Reply-To ni Custom headers.

Solución recomendada (sin cambiar arquitectura):

Usa alias Gmail con “+soporte” en el FROM:

Enrico (Soporte) <eb.automation.ai+soporte@gmail.com>

Crea filtro en Gmail:

to:eb.automation.ai+soporte@gmail.com → etiqueta “Soporte”

7) Error SMTP “Username and Password not accepted” (535) / no autentica
Causa:

Estás usando tu contraseña normal, o no tienes App Password.

Solución:

Activa 2-Step Verification en la cuenta emisora.

Crea una App Password para “Mail” / “n8n”.

Pega la App Password en la credencial SMTP.

8) Error TLS/SSL / certificado
Causa:

Puerto/SSL mal configurado, o inspección SSL en red corporativa.

Configuración recomendada Gmail:

Host: smtp.gmail.com

Port: 465

SSL: ON

Client Host Name: localhost

Si sigues con error (solo para pruebas):

Option “Ignore SSL Issues (insecure)” = ON
(Usar solo para diagnosticar; no recomendado permanente.)

9) Los campos se escriben como texto literal (ej. {{ ... }})
Causa:

Pegaste la expresión en modo “texto”, no en modo Expression (fx).

Solución:

En el campo, activa Expression (fx).

Pega la expresión dentro del editor de expresiones.

Guarda y Publish.

10) status_detail no aparece en el Update Row
Causas:

El header no está en fila 1.

Hay cabeceras vacías entre medias.

n8n no ha refrescado el esquema.

Solución:

Asegura que status_detail está en la fila 1 y sin espacios (status_detail).

Evita columnas con header vacío.

En el nodo Update Row:

re-selecciona Spreadsheet/Sheet

guarda y recarga (Ctrl+F5)

Alternativa robusta: usar “Map Automatically” y que el JSON incluya el campo.

11) Anti-reenvío no funciona (reenvía)
Causa:

sent está como "TRUE" / TRUE / "true" y la condición no lo contempla.

Solución:
Usa condición robusta (normaliza a string lower/trim):

js
Copiar código
{{ (($json.sent ?? "").toString().replace(/\r?\n/g, "").trim().toLowerCase() !== "true") }}
12) Checklist de “sanidad” antes de publicar (producción)
Workflow Published

Trigger Row Added (polling 1 min)

Match por row_key sanitizado

Un solo nodo de envío (SMTP)

Merge Input1=SMTP, Input2=row data

Set/Edit Fields con lógica SMTP (usa $json.error)

Update principal escribe sent/status/sent_at/status_detail/err_msg

Rama Missing email escribe ERROR + E_MISSING_EMAIL

Export JSON actualizado y sin secretos

ZIP de release contiene solo docs + workflows + README

makefile
Copiar código
::contentReference[oaicite:0]{index=0}





