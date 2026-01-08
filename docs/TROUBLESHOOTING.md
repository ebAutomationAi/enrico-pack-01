```md
# TROUBLESHOOTING — Pack #1 Funnel

Este documento cubre los fallos típicos que aparecen en producción y cómo resolverlos.

---

## 1) “status” aparece como literal `{{ ... }}` en Google Sheets

### Síntoma
En la celda `status` aparece el texto:
`{{ $json.id ? "SENT" : "ERROR" }}`

### Causa
El campo `status` en el nodo Google Sheets Update no estaba en modo **Expression (fx)** (o se pegó la expresión sin activar fx).

### Solución
1) Abre el nodo: `Update row in sheet (SENT/ERROR)`
2) En el campo `status`, activa **Expression (fx)**
3) Pega:
```txt
{{ $json.id ? "SENT" : "ERROR" }}

Guarda y prueba con una fila nueva.

Limpia manualmente las filas antiguas afectadas (pon SENT).

2) Update Row falla por row_key con saltos de línea
Síntoma

En el output del Trigger o de un IF aparece algo como:
row_key: "....|\n\n"

Causa

Se escribió row_key manualmente con Enter o se pegó texto con saltos de línea.

Solución (doble)

A) En el Sheet:

Reescribe el row_key sin saltos de línea, o usa ARRAYFORMULA (recomendado).

B) En n8n (match sanitizado):

En row_key (using to match) usa:

{{ ($json.row_key ?? "").toString().replace(/\r?\n/g, "").trim() }}

3) Email “vacío” pero en realidad tiene un espacio " "
Síntoma

El item muestra:
"Dirección de correo electrónico": " "

Causa

La celda no está vacía: contiene espacios invisibles.

Solución

A) En el IF “Email válido”, usa trim:

{{ ($json["Dirección de correo electrónico"] ?? "").toString().replace(/\r?\n/g, "").trim() !== "" }}


B) En el Sheet:

Borra la celda con Supr/Delete (no con backspace parcial).

4) Google Sheets Trigger trae muchos items (no solo 1)
Síntoma

El Trigger devuelve 5–10 items en vez de 1.

Causa

Polling + estado interno del Trigger (normal). No confíes en “1 item”.

Solución

El control real es el IF “Pendiente”. Ejemplo de condición robusta:

{{ (($json.sent ?? "").toString().replace(/\r?\n/g, "").trim().toLowerCase() !== "true") }}


Si esto está bien, no hay reenvíos aunque el Trigger entregue muchas filas.

5) Error: “The 'Column to Match On' parameter is required”
Síntoma

El nodo Update Row falla aunque visualmente parezca que Column to match on está seleccionado.

Causa

Desincronización UI/schema: internamente matchingColumns quedó vacío (bug/estado de UI).

Solución más fiable

Duplica un nodo Update que funcione y reutilízalo.

Vuelve a seleccionar Column to match on desde el desplegable.

Guarda el workflow.

Vuelve a ejecutar.

6) Gmail falla pero el workflow no se detiene (Continue On Fail)
Síntoma

El workflow continúa pero quieres capturar el error real en err_msg.

Solución

En el Update principal, usa un err_msg con múltiples “rutas”:

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

7) Log n8n: “Python 3 is missing… task runner…”
Síntoma

En logs aparece el aviso de Python.

Causa

Aviso del task runner interno. No afecta al funnel Gmail/Sheets.

Acción

Ignorable para este pack. Solo es relevante si más adelante usas un runner Python externo.

::contentReference[oaicite:0]{index=0}
