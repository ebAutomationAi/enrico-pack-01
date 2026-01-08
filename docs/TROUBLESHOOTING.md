# TROUBLESHOOTING — Pack #1 Funnel (Google Form → Sheets → n8n → Gmail)

Este documento lista los problemas más frecuentes en producción y su solución exacta.

---

## 1) “status” aparece como literal `{{ ... }}` en Google Sheets

### Síntoma
En la columna `status` ves el texto literal:
`{{ $json.id ? "SENT" : "ERROR" }}`

### Causa
En el nodo **Google Sheets → Update Row**, el campo `status` no estaba en modo **Expression (fx)** cuando pegaste la expresión (o se escribió como texto).

### Solución
1) Abre el nodo: `Update row in sheet (SENT/ERROR)`
2) En `Values to Update` → campo `status`
3) Activa **Expression (fx)**
4) Pega exactamente:
```txt
{{ $json.id ? "SENT" : "ERROR" }}
Guarda el workflow.

Prueba creando una nueva fila (nuevo envío del Form).

Limpia manualmente las filas antiguas afectadas en la sheet (pon SENT o ERROR según corresponda).

2) “sent” / “sent_at” aparecen como literales {{ ... }} en la sheet
Síntoma
En sent o sent_at aparece texto tipo:

{{ true }}

{{$now.toISO()}}

Causa
Igual que el caso anterior: el campo no estaba en modo Expression (fx).

Solución
Abre el nodo Update row in sheet (SENT/ERROR)

En cada campo (sent, sent_at, etc.) activa Expression (fx)

Usa expresiones válidas (ejemplo recomendado):

sent:

txt
Copiar código
{{ $json.id ? true : "" }}
sent_at:

txt
Copiar código
{{ $json.id ? $now.setZone('Europe/Madrid').toISO() : "" }}
Guarda y prueba con una nueva fila.

3) Update Row falla por row_key con saltos de línea \n
Síntoma
En el output del Trigger/IF aparece:

row_key: "....|\n\n"
y el Update no encuentra la fila o el match se vuelve inconsistente.

Causa
row_key se escribió manualmente con Enter o se pegó con saltos de línea invisibles.

Solución (doble, obligatoria)
A) En Google Sheets:

No escribas row_key a mano.

Genera row_key con ARRAYFORMULA (ver SHEET_SCHEMA.md), o reescribe el valor en una sola línea.

B) En n8n (sanitizado en el match):

En ambos nodos Update (SENT/ERROR y ERROR Missing email), en:
row_key (using to match) usa:

txt
Copiar código
{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}
4) Email “vacío” pero en realidad contiene un espacio " "
Síntoma
En el item ves:

"Dirección de correo electrónico": " "

Gmail falla con:

Invalid email address ... To field isn't valid

Causa
La celda no está vacía: contiene espacios invisibles.

Solución
A) En el IF “Email válido”, usa trim y elimina saltos:

txt
Copiar código
{{ ($json["Dirección de correo electrónico"] ?? "").toString().replace(/\r?\n/g, "").trim() !== "" }}
B) En Google Sheets:

Borra la celda con Supr/Delete (no con backspace parcial).

Evita “rellenos” manuales con espacios.

5) Google Sheets Trigger devuelve muchos items (no solo 1)
Síntoma
El Trigger devuelve 5–10 items en cada ejecución.

Causa
Polling + estado interno del Trigger (comportamiento normal). No asumas “un evento = un item”.

Solución
El control real es tu IF “Pendiente”. Usa una condición robusta que cubra TRUE, true, espacios, etc.:

Ejemplo recomendado para “pendiente”:

txt
Copiar código
{{ (($json.sent ?? "").toString().replace(/\r?\n/g, "").trim().toLowerCase() !== "true") }}
Con esto no hay reenvíos aunque el Trigger entregue varias filas.

6) Error: row_number is null or undefined
Síntoma
El nodo Update da error:
row_number is null or undefined

Causa
Estás intentando matchear por row_number pero el item que llega a Update no lo contiene (o tu Trigger no lo emite en ese flujo).

Solución recomendada
No dependas de row_number. Usa row_key como columna de match.

En el nodo Update:

Column to match on = row_key

En row_key (using to match):

txt
Copiar código
{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}
Confirma que tu Sheet tiene la columna row_key (ver SHEET_SCHEMA.md).

7) Error: “The 'Column to Match On' parameter is required” (aunque esté seleccionado)
Síntoma
El nodo Update falla con:
The 'Column to Match On' parameter is required

aunque en la UI veas Column to match on seleccionado.

Causa
Bug/desincronización de UI/schema: internamente matchingColumns quedó vacío.

Solución más fiable (la que funciona)
Duplica un nodo Update que funcione (o crea uno nuevo).

Vuelve a seleccionar:

Spreadsheet

Sheet

Column to match on (desde el desplegable)

Guarda el workflow.

Ejecuta de nuevo.

8) Gmail falla pero el workflow no se detiene (Continue On Fail)
Síntoma
Con Continue On Fail = ON, el flujo sigue, pero quieres capturar el motivo real en err_msg.

Solución
En el nodo Update principal, usa este err_msg (Expression):

txt
Copiar código
{{ 
  $json.id 
    ? "" 
    : (
        $json.error?.message 
        ?? $json.error?.description 
        ?? $json.error?.cause?.message
        ?? $json.message
        ?? $json?.response?.body?.error?.message
        ?? $json?.response?.body
        ?? "Gmail send failed"
      ) 
}}
Y usa un status condicional:

txt
Copiar código
{{ $json.id ? "SENT" : "ERROR" }}
9) Rama “ERROR Missing email” no actualiza
Síntoma
La rama de error se ejecuta, pero no cambia la fila en la sheet.

Causa (habitual)
El row_key del item no coincide exactamente con el row_key de la celda (por espacios/saltos).

Solución
Sanitiza el match en n8n:

txt
Copiar código
{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}
Verifica en Sheets que row_key no contiene caracteres invisibles:

En una columna auxiliar:

gs
Copiar código
=LEN(A2)
y:

gs
Copiar código
=REGEXMATCH(A2,CHAR(10))
Si REGEXMATCH da TRUE, hay salto de línea.

10) Log de n8n: “Python 3 is missing… task runner…”
Síntoma
En logs:
Failed to start Python task runner... Python 3 is missing...

Causa
Aviso del runner interno. No afecta a este pack (Sheets/Gmail).

Acción
Ignorable para este pack. Solo importa si más adelante usas un runner Python externo.

makefile
Copiar código
::contentReference[oaicite:0]{index=0}
