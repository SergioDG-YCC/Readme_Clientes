<table width="100%">
  <tr>
    <td align="left" width="33%">
      <img src="https://cdn1.ycc.com.ar/wp-content/uploads/2024/08/logo_ycc.webp" width="150">
    </td>
    <td align="center" width="34%">
      <span style="font-size:18px; font-weight:bold;">IT Development Area</span>
    </td>
    <td align="right" width="33%">
      <img src="https://microtv.com.ar/wp-content/uploads/2025/07/logo_Microtv_Negro.png" width="150">
    </td>
  </tr>
</table>  

---
# Manual de uso – Descarga de históricos GPS (API gzip)

Este documento explica cómo consultar y descargar históricos de posiciones GPS desde la API en los equipos de a bordo. Incluye ejemplos 100% prácticos con `curl` usando IP directa.

---

## 1) Conceptos clave

* **Endpoint base**: `http://<IP_DEL_EQUIPO>:9001`
* **Formato de salida**: **NDJSON** (una línea JSON por registro).
* **Compresión**: la API puede responder **gzip** cuando el cliente lo solicita.
* **Rangos de tiempo**: se piden por **tramos de hora** con formato `YYYYMMDD_HH-YYYYMMDD_HH` (UTC).
* **Serial**: identificador del dispositivo, p. ej. `WFTXFW321903325`.

> Los equipos guardan archivos **.ndjson** (segmentos vivos) y **.ndjson.gz** (horas cerradas). La API concatena ambos y, si el cliente lo pide, envía todo comprimido **en un solo stream gzip**.

---

## 2) Endpoint principal

```
GET /chunk/{serial}/{YYYYMMDD_HH}-{YYYYMMDD_HH}
```

**Ejemplo**:

```
GET /chunk/WFTXFW321903325/20250817_12-20250817_13
```

**Base URL de ejemplo**: `http://172.16.20.20:9001`

---

## 3) Cabeceras HTTP relevantes

* `Accept-Encoding: gzip` → solicita datos comprimidos.
* Respuesta típica al enviar esa cabecera:

  * `Content-Type: application/x-ndjson`
  * `Content-Encoding: gzip`
  * `Vary: Accept-Encoding`

> Si **no** se envía `Accept-Encoding: gzip`, la API responde en **texto plano** NDJSON (más tráfico).

---

## 4) Ejemplos con `curl` (Linux/macOS/Windows 10+)

### 4.1 Ver en consola (rango corto) descomprimido automáticamente

```bash
curl -s --compressed \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13"
```

* `--compressed` hace que `curl` pida gzip y **descomprima automáticamente**.

### 4.2 Guardar comprimido a disco (gzip) y revisar

```bash
curl -s -H "Accept-Encoding: gzip" \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13" \
  -o chunk_20250817_12-13.ndjson.gz

# Validar tipo
file chunk_20250817_12-13.ndjson.gz
# Leer sin descomprimir a disco
zcat chunk_20250817_12-13.ndjson.gz | head
```

### 4.3 Contar registros rápidamente

```bash
# Descomprimido en el cliente (corto):
curl -s --compressed \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13" | wc -l

# Comprimido y expandido localmente (largo):
curl -s -H "Accept-Encoding: gzip" \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_00-20250817_23" | zcat | wc -l
```

### 4.4 Rango grande (muchas horas)

Para rangos muy largos, preferir **guardar comprimido** y descomprimir localmente:

```bash
curl -s -H "Accept-Encoding: gzip" \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250816_03-20250817_22" \
  -o chunk_20250816_03-20250817_22.ndjson.gz

# Visualizar por páginas sin saturar la terminal
ezcat chunk_20250816_03-20250817_22.ndjson.gz | less
```

> Nota: algunos clientes `curl --compressed` pueden no manejar bien **streams gzip con múltiples miembros** en rangos extensos. En ese caso, use la variante **guardar .gz** + `zcat`/`gunzip -c`.

### 4.5 Descargar para usuario final (adjunto)

Si la API expone un modo descarga (opcional):

```
GET /chunk_download/{serial}/{YYYYMMDD_HH}-{YYYYMMDD_HH}
```

El servidor puede agregar:

```
Content-Type: application/gzip
Content-Disposition: attachment; filename="{serial}_{from}-{to}.ndjson.gz"
```

---

## 5) Buenos hábitos y rendimiento

* **Rangos razonables** para trabajo interactivo: 1–4 h. Para más, guardar `.gz` y post-procesar.
* **Ahorro de datos**: NDJSON suele comprimir **70–85%** con gzip.
* **Costo CPU en el equipo**: mínimo. Las horas cerradas `.gz` se envían tal cual; los segmentos vivos se comprimen con nivel bajo sólo si el cliente lo pidió.

---

## 6) Ejemplos de filtrado rápido en consola

* Sólo hora y velocidad:

```bash
curl -s --compressed \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13" \
  | jq -r '{t:.time, v:.speed} | @json'
```

* Filtrar por ventana exacta dentro de la hora (cliente):

```bash
curl -s -H "Accept-Encoding: gzip" \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13" \
  | zcat | awk -F '"' '{print $4}' | sed -n '/2025-08-17T12:15/,/2025-08-17T12:45/p'
```

---

## 7) Errores comunes

* **404 Not Found** con `-I` (HEAD): la API no implementa HEAD; use `GET` y `-D -` para ver headers.
* Respuesta corta con `--compressed` en rangos enormes: use `-H "Accept-Encoding: gzip"` + `zcat`.
* Tiempos locales vs UTC: los parámetros están en **UTC** por diseño.

---

## 8) Apéndice – Ver sólo headers de un GET real

```bash
curl -sS --compressed -D - \
  "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250817_12-20250817_13" \
  -o /dev/null
```

Salida esperada (resumen):

```
HTTP/1.1 200 OK
Content-Type: application/x-ndjson
Content-Encoding: gzip
Vary: Accept-Encoding
```

---

## 9) Verificación y métricas rápidas (en el equipo)

> Útil para soporte o auditoría. Permite evaluar ahorro real y volumen por día.

**9.1 Total ocupado del día (comprimido):**

```sh
du -ch /overlay/gps_buffer/WFTXFW321903325-20250816_*.ndjson.gz
```

**9.2 Contar registros de un rango (multi .gz):**

```sh
zcat /overlay/gps_buffer/WFTXFW321903325-20250816_*.ndjson.gz | wc -l
```

**9.3 Comparar bytes comprimidos vs descomprimidos (ratio):**

```sh
# Comprimido (suma de tamaños .gz)
ls -l /overlay/gps_buffer/WFTXFW321903325-20250816_*.ndjson.gz | awk '{s+=$5} END{print s, "bytes .gz"}'

# Descomprimido (expandido)
zcat /overlay/gps_buffer/WFTXFW321903325-20250816_*.ndjson.gz | wc -c
```

**9.4 Ver contenido de una hora puntual:**

```sh
zcat /overlay/gps_buffer/WFTXFW321903325-20250816_10.ndjson.gz | head
```

> Nota de rendimiento: los archivos `.ndjson.gz` horarios se envían **tal cual** desde la API (costo de CPU mínimo). Los segmentos vivos de 10 minutos se comprimen al vuelo **solo si el cliente lo solicita** (`Accept-Encoding: gzip`), usando nivel bajo para mantener latencia muy baja.

---

### Contacto

Para dudas o integraciones personalizadas, contacte al área de IT con el **serial del equipo** y el **rango de fechas** que desea consultar.

---

Área IT Development © MicroTV - 2025

---

