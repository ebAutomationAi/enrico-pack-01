# enrico-pack-01 — Pack #1 (Funnel Google Form → Google Sheets → n8n → Gmail)

Pack #1 es un funnel de captación y entrega automática:

1) Capturas leads con **Google Form**  
2) Se registran en **Google Sheets**  
3) **n8n self-hosted** detecta nuevas filas (trigger)  
4) Envía un email (Gmail) con el link del pack  
5) Actualiza la fila con logging: `sent`, `sent_at`, `status`, `err_msg`, `gmail_id`, `gmail_threadId`

---

## Descarga

Recomendado: descargar el ZIP desde GitHub Releases.

- Última versión (descarga directa):
  `https://github.com/<TU_USUARIO>/enrico-pack-01/releases/latest/download/enrico-pack-01.zip`

---

## Requisitos

- n8n **2.x** (recomendado self-hosted en Docker)
- Zona horaria recomendada: `Europe/Madrid`
- Credenciales OAuth en n8n:
  - Google Sheets (lectura/escritura)
  - Gmail (envío)

---

## Estructura esperada en Google Sheets

**Spreadsheet:** el que recibe respuestas del Google Form  
**Sheet/Tab:** `Respuestas de formulario 1`

Columnas (cabeceras) recomendadas:

- `Marca temporal`
- `Dirección de correo electrónico`
- `Nombre`
- `Objetivo`
- `sent`
- `sent_at`
- `status`
- `err_msg`
- `gmail_id`
- `gmail_threadId`
- `row_key` *(recomendado como ID estable para match)*

### `row_key` (por qué y cómo)
Se recomienda `row_key` para matchear la actualización de la fila de forma robusta y evitar problemas con emails repetidos o filas ambiguas.

Formato recomendado:
- `YYYY-MM-DD HH:mm:ss|email`

Ejemplo:
- `2026-01-07 23:06:41|test@gmail.com`

Si `row_key` se genera por fórmula en Google Sheets (ARRAYFORMULA), mejor: así evitas entradas manuales con espacios o saltos de línea.

---

## Instalación (paso a paso)

### 1) Descarga y descomprime
1. Descarga `enrico-pack-01.zip` desde Releases.
2. Descomprime el ZIP.

### 2) Importa el workflow en n8n
1. En n8n: **Workflows**
2. **Import from File**
3. Selecciona el/los archivos `.json` del directorio `workflows/`

### 3) Configura credenciales
En los nodos de Google/Gmail:
- Selecciona tus credenciales OAuth de **Google Sheets**
- Selecciona tus credenciales OAuth de **Gmail**

### 4) Revisa parámetros obligatorios
En el workflow, verifica:

- **Google Sheets Trigger**
  - Spreadsheet ID correcto
  - Sheet = `Respuestas de formulario 1`
  - Evento: `rowAdded`
  - Polling: (por ejemplo, cada minuto)

- **Gmail: Send a message**
  - `To`: usa la columna `Dirección de correo electrónico`
  - Sustituye el placeholder del link del pack por el enlace de tu Release

- **Google Sheets: Update Row**
  - Match por `row_key`
  - Mapea columnas: `sent`, `sent_at`, `status`, `err_msg`, `gmail_id`, `gmail_threadId`

### 5) Activa el workflow
1. Guarda el workflow.
2. Actívalo (**Active**).

---

## Lógica del workflow (resumen técnico)

- **Trigger (rowAdded)**: trae items nuevos (polling).  
- **IF pendiente**: procesa solo filas “no enviadas” (`sent != true`).  
- **IF email válido**: evita items con email vacío/espacios (blindaje).  
- **Gmail Send**: envía el email (recomendado `Continue On Fail` ON).  
- **Merge + Update**: actualiza por `row_key`:
  - `sent = true` solo si Gmail devolvió `id`
  - `sent_at` solo en éxito
  - `status = SENT | ERROR`
  - `err_msg` con el mensaje real si falla
  - `gmail_id` y `gmail_threadId` para trazabilidad

---

## Logging / Errores

- **Éxito**
  - `sent = TRUE`
  - `status = SENT`
  - `err_msg = ""`
  - `gmail_id`, `gmail_threadId` poblados

- **Fallo de Gmail**
  - `status = ERROR`
  - `err_msg` registra el motivo (si está disponible)
  - `sent_at` queda vacío

- **Email vacío/espacios**
  - `status = ERROR`
  - `err_msg = Missing email`

---

## Buenas prácticas (producción)

- No guardes credenciales en archivos del repo.
- Mantén `row_key` como identificador estable para actualizaciones.
- Evita espacios “invisibles” en celdas (usa `trim()` en n8n donde aplique).
- Prueba el enlace de descarga en ventana incógnito.

---

## Privacidad (GDPR-safe)

- Este pack no incluye credenciales ni datos sensibles.
- En demos usa datos ficticios. Sustituye por tus datos reales.
- Si procesas leads reales, asegúrate de tener base legal y política de privacidad.

---

## Versionado

- Releases en GitHub: `vMAJOR.MINOR.PATCH`
- Recomendación: mantener el asset ZIP con nombre fijo:
  - `enrico-pack-01.zip`
  para poder usar el enlace estable `releases/latest/download/...`

---

## Soporte / Contacto

Si necesitas adaptar este pack (otros providers, Notion/CRM, segmentación por “Objetivo”, etc.), crea un Issue en este repo con:
- tu estructura de columnas
- tu versión de n8n
- captura de ejecución (sin datos sensibles)

