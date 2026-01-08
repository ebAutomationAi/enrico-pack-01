# SETUP — Pack #1 Funnel (Google Form → Sheets → n8n → Gmail)

Este documento asume:
- n8n 2.x (self-hosted, Docker)
- TZ: Europe/Madrid
- Google Form conectado a Google Sheets (tab: “Respuestas de formulario 1”)
- OAuth de Gmail y Google Sheets ya creado en n8n (o lo vas a crear ahora)

---

## 1) Preparar Google Sheets (requisito obligatorio)

### 1.1 Hoja correcta
- Spreadsheet: el que recibe las respuestas del Form
- Tab / Sheet: `Respuestas de formulario 1`

### 1.2 Columnas requeridas (cabeceras exactas)
Debes tener estas columnas (en este orden recomendado):

1) `row_key`
2) `Marca temporal`
3) `Dirección de correo electrónico`
4) `Nombre`
5) `Objetivo`
6) `sent`
7) `sent_at`
8) `status`
9) `err_msg`
10) `gmail_id`
11) `gmail_threadId`

Si ya existen las de Google Form (timestamp/email/nombre/objetivo), añade las restantes.

### 1.3 Crear `row_key` (muy recomendado)
En `docs/SHEET_SCHEMA.md` tienes la fórmula exacta para generar row_key con ARRAYFORMULA.

---

## 2) Importar el workflow en n8n

### 2.1 Importar JSON
1) Abre n8n (Editor)
2) Menu lateral → `Workflows`
3) Botón `Import from File`
4) Selecciona:
   - `workflows/pack-01-funnel-googleform-sheets-gmail.json`

---

## 3) Configurar credenciales (n8n)

### 3.1 Google Sheets Trigger (credencial Trigger)
1) Clic en nodo `Google Sheets Trigger`
2) En `Credentials` selecciona:
   - `Google Sheets Trigger account` (o tu credencial equivalente)
3) En `Document` (Spreadsheet):
   - Selecciona el Spreadsheet desde el desplegable
4) En `Sheet`:
   - Selecciona `Respuestas de formulario 1`
5) `Event`:
   - `On row added`
6) `Poll time`:
   - `Every minute` (o el intervalo que quieras)

### 3.2 Gmail (credencial Gmail OAuth)
1) Clic en nodo `Gmail - Send Pack #1`
2) Selecciona credencial `Gmail account` (o la tuya)
3) Ve a la pestaña `Settings`
4) Activa:
   - `Continue On Fail` = ON

### 3.3 Google Sheets Update (credencial Sheets OAuth)
Hay 2 nodos:
- `Update row in sheet (SENT/ERROR)`
- `Update row in sheet (ERROR Missing email)`

En ambos:
1) Selecciona la credencial `Google Sheets account`
2) Selecciona el mismo Spreadsheet
3) Selecciona la misma Sheet `Respuestas de formulario 1`
4) Confirma que el match es por:
   - `Column to match on` = `row_key`

---

## 4) Configurar el email de entrega

1) Abre nodo `Gmail - Send Pack #1`
2) En `Message`, confirma/actualiza el link de descarga del ZIP:

https://github.com/ebAutomationAi/enrico-pack-01/releases/latest/download/enrico-pack-01.zip

Recomendación: prueba el link en incógnito (sin login).

---

## 5) Activar el workflow

1) Pulsa `Save`
2) Cambia el switch del workflow a `Active`

---

## 6) Prueba end-to-end (obligatoria)

### 6.1 Test real con Google Form
1) Envía una respuesta nueva desde el Form con:
   - email real
   - nombre
   - objetivo
2) Espera al siguiente poll (hasta 1 minuto)
3) Verifica:
   - llega el email
   - en la fila del sheet se actualiza:
     - `sent` = TRUE
     - `sent_at` = ISO con Europe/Madrid
     - `status` = SENT
     - `gmail_id` y `gmail_threadId` poblados

### 6.2 Test de rama ERROR Missing email (solo para comprobar)
En el Sheet, añade una fila nueva AL FINAL dejando el email vacío (sin espacios) y con row_key válido.
Resultado esperado:
- `status` = ERROR
- `err_msg` = Missing email

---

## 7) Producción: checklist mínimo

- Link de Release descargable en incógnito
- IF “Pendiente” evita reenvíos (`sent != true`)
- IF “Email válido” blinda filas raras
- Update por `row_key` (no por email)
- Gmail: Continue On Fail ON
- Backup del volumen de n8n realizado

Después de importarlo en n8n: En Google Sheets Trigger, selecciona el Spreadsheet y la sheet desde el desplegable. 
En ambos Google Sheets: Update, selecciona el mismo Spreadsheet y sheet desde el desplegable. 
En Gmail - Send Pack #1, activa Settings → Continue On Fail = ON (esto no lo fuerza el JSON). 
y estas otras lineas: Notas: Esto rellena row_key automáticamente para filas nuevas. 
LOWER(TRIM(...)) normaliza el email y evita espacios invisibles. 
Si la columna B no está en formato datetime, primero corrige el formato (Formato → Número → Fecha y hora).
