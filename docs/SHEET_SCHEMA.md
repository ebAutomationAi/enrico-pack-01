# SHEET_SCHEMA — Columnas y row_key (Pack #1 Funnel)

Hoja: `Respuestas de formulario 1`

---

## 1) Columnas (cabeceras exactas)

| Columna | Tipo recomendado | Ejemplo | Propósito |
|---|---|---|---|
| row_key | string | 2026-01-08 18:14:00|test@gmail.com | ID estable para hacer match en Update Row |
| Marca temporal | datetime | 8/01/2026 18:14:00 | Campo original de Google Form |
| Dirección de correo electrónico | string | test@gmail.com | Destinatario del email |
| Nombre | string | Carlos | Personalización del email |
| Objetivo | string | Empleo / Formación / Pymes | Segmentación futura |
| sent | boolean/string | TRUE | Evitar reenvíos |
| sent_at | string ISO | 2026-01-08T19:21:46.026+01:00 | Timestamp del envío |
| status | string | SENT / ERROR | Estado de envío |
| err_msg | string | Missing email / mensaje real | Diagnóstico en errores |
| gmail_id | string | 19b9... | ID del mensaje enviado |
| gmail_threadId | string | 19b9... | ID del hilo |

---

## 2) Fórmula recomendada para generar row_key (ARRAYFORMULA)

Suposición de columnas:
- A: row_key
- B: Marca temporal
- C: Dirección de correo electrónico

### Opción recomendada (header manual + fórmula en A2)
1) A1 = `row_key` (escrito manualmente)
2) En A2 pega:

```gs
=ARRAYFORMULA(
  IF(
    B2:B="",
    "",
    TEXT(B2:B,"yyyy-mm-dd hh:mm:ss")&"|"&LOWER(TRIM(C2:C))
  )
)


Notas

Esto rellena row_key automáticamente para filas nuevas.

LOWER(TRIM(...)) normaliza el email y evita espacios invisibles.

Si la columna B no está en formato fecha/hora, corrige primero el formato:

Formato → Número → Fecha y hora

Evita escribir row_key manualmente si usas esta fórmula (para no meter saltos de línea o espacios).

3) Valores esperados por el workflow
En éxito (Gmail ok)

sent = TRUE

sent_at = ISO (zona Europe/Madrid)

status = SENT

err_msg = ""

gmail_id y gmail_threadId rellenados

En error (email vacío / item raro)

status = ERROR

err_msg = Missing email (o el error real de Gmail)

sent y sent_at vacíos

gmail_id y gmail_threadId vacíos
