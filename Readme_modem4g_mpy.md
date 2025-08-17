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
# 🔹 Plataforma MT7628 – Documentación para Clientes

> **Ámbito:** Dispositivos embarcados con MT7628 ejecutando 3 servicios en MicroPython para adquisición y transmisión de datos GPS, API local y lógica de latencia/ahorro de datos.
>
> **Perfil lector:** Operaciones/IT del cliente (no requiere conocimientos de desarrollo), integradores de sistemas y equipo de soporte.

---

## 1) Resumen ejecutivo

* **Qué hace el equipo:**

  * Registra datos de **GPS** localmente con rotación y compresión.
  * Envía datos a un **servidor central** de forma continua o diferida.
  * Expone una **API HTTP local** para consulta de estado y extracción de históricos.
  * Implementa un sistema de **latencia adaptativa** (según velocidad/movimiento) para optimizar consumo de datos.
* **Beneficios clave:** baja latencia en movimiento, ahorro de datos en detención, recuperación de brechas post‑conectividad.

---

## 2) Arquitectura general

```
[GPS] → [Servicio 1: Adquisición & Buffer NDJSON] → [Rotación & Compresión GZIP]
                                         ↘
                                          [Servicio 2: Transmisión/Backhaul → Servidor]
                                                  ↘
                                                   [Servicio 3: API Local HTTP]
```

* **Formato datos:** NDJSON (1 JSON por línea) con tiempo en UTC (ISO‑8601).
* **Compresión:** GZIP por ventanas horarias.
* **Reloj:** sincronizado por GPS (prioritario) o NTP (fallback).

> **Nota:** Reemplazar los marcadores `{{…}}` con valores reales al integrar los scripts.

---

## 3) Instalación y servicios

### 3.1 Servicios del sistema (OpenWrt)

* **gps\_tracker** → adquisición GPS + `latest.json` + NDJSON + consolidación horaria
* **gps\_udp** → publicación UDP de la última muestra a servidor central
* **gps\_api** → API HTTP local (endpoints de listado, `chunk`, `config`, `wifi_fence`)

```sh
# Estado / arranque / reinicio
/etc/init.d/gps_tracker status|start|restart
/etc/init.d/gps_udp     status|start|restart
/etc/init.d/gps_api     status|start|restart

# Logs
logread -e gps
```

### 3.2 Rutas y archivos

* Scripts: `/usr/bin/gps_tracker.py`, `/usr/bin/gps_udp.py`, `/usr/bin/gps_api.py`
* Config: `/etc/gps_fields.json` (latency/fields), `/etc/wifi_fence.json`
* Datos: `/dev/shm/gps_latest.json` (preferido) o `/tmp/gps_latest.json`; `NDJSON` en `/tmp/gps_logs/`; `GZIP` por hora en `/overlay/gps_buffer/`

---

## 4) Almacenamiento local y retención

| Item                            | Valor                                                                                     |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| Carpeta temporal (trozos vivos) | `/tmp/gps_logs/` (`*.ndjson` de **10 min** por archivo)                                   |
| Carpeta de buffer horario       | `/overlay/gps_buffer/` (`*.ndjson.gz` por **hora** cerrada)                               |
| Nomenclatura                    | `SERIAL-YYYYMMDD_HHMM.ndjson` (10 min) · `SERIAL-YYYYMMDD_HH.ndjson.gz` (1 h)             |
| Rotación / cierre               | Cada 10 min se agrega al **slot**; cada \~30 s se intenta **consolidar** la hora anterior |
| Compresión                      | BusyBox `gzip -9` (on-device)                                                             |
| Retención local                 | **48** horas comprimidas por `serial` (limpieza automática)                               |
| Integridad                      | Escritura atómica del `latest.json` y de NDJSON; concat sin cargar todo en RAM            |

> La consolidación corre en **gps\_tracker.py**, que concatena los 6 trozos de 10 min (o los que existan) y comprime a `gz`; luego borra los fuentes del slot. El API sirve tanto los `.gz` como el NDJSON activo.

## 5)  Latencia adaptativa (según velocidad)

* **Objetivo:** optimizar consumo vs. frescura de datos. Configurable vía `/etc/gps_fields.json` (`latency`).
* **Default actual (m/s):**

  * `speed_0` ≤ **0.5 m/s** (\~1.8 km/h) → **30 s** (con *heartbeat* ON)
  * `speed_1` ≤ **5 m/s** (\~18 km/h) → **10 s**
  * `speed_2` ≤ **14 m/s** (\~50 km/h) → **5 s**
  * `speed_3` ≤ **22 m/s** (\~79 km/h) → **1 s**
  * `above`  > **22 m/s** → **1 s**
* **Estado de movimiento:** `track_status = "motion"` si `speed ≥ 1.0 m/s`; caso contrario `"stop"`.
* **Extras temporizados:** `GGA`/`GSA`/`VTG` se inyectan con TTL (5–10 s) para `alt`, `sats`, `hdop/pdop/vdop`, etc.

> **Nota para clientes:** El campo `speed` en el payload se reporta en **m/s**; puede presentarse en km/h con la conversión `km/h = m/s × 3.6`.

---

## 6) Esquema de datos (payload)

### 6.1 Registro NDJSON (ejemplo real)

```json
{"time":"2025-08-16T04:34:01Z","lat":-34.515943,"lon":-58.560051,
 "serial":"WFTXFW321903325","mac":"B0:CE:18:10:12:E7",
 "track_status":"stop","alt":33.7,"speed":0.0,
 "snr":-12.9,"sats":"08"}
```

**Campos**

| Campo                                  | Tipo                | Descripción                                             |
| -------------------------------------- | ------------------- | ------------------------------------------------------- |
| `time`                                 | string ISO‑8601 UTC | Timestamp de la muestra                                 |
| `lat`/`lon`                            | float (6 decimales) | Posición WGS‑84                                         |
| `alt`                                  | float (m)           | Altitud (si hay GGA reciente)                           |
| `speed`                                | float (**m/s**)     | Velocidad (0.0 cuando detenido)                         |
| `track`                                | float (°)           | Rumbo verdadero (se conserva el último en movimiento)   |
| `sats`/`hdop`/`pdop`/`vdop`/`geoidSep` | varios              | Calidad de fix (TTL 5–10 s)                             |
| `snr`                                  | float (proxy)       | Indicador de calidad sintetizado a partir de DOP y sats |
| `serial`/`mac`                         | string              | Identificadores de dispositivo                          |
| `track_status`                         | enum                | `motion` / `stop`                                       |

## 7)  API Local HTTP (Servicio 3)

> Base URL por defecto: `http://<IP_DEL_EQUIPO>:9001/`

### 7.1 Endpoints **GET**

| Endpoint                          | Descripción                                               | Ejemplo                                                           |        |
| --------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------- | ------ |
| `/list_all/{serial}`              | Lista archivos **`gz`** (hora) y **`tmp`** (slots 10 min) | `curl -s http://IP:9001/list_all/WFTXFW321903325`                 |        |
| `/list/{serial}`                  | Solo `*.ndjson.gz` (horas cerradas) + limpieza >48        | `curl -s http://IP:9001/list/WFTXFW321903325`                     |        |
| `/range/{serial}`                 | Horas disponibles `YYYYMMDD_HH` (mezcla gz + tmp)         | `curl -s http://IP:9001/range/WFTXFW321903325`                    |        |
| `/chunk/{serial}/{desde}-{hasta}` | Devuelve **NDJSON** concatenado de horas en rango (incl.) | `curl -s http://IP:9001/chunk/WFTX.../20250816_03-20250816_04`    |        |
| `/config`                         | Config efectiva (solo `latency`, paths y retención)       | \`curl -s [http://IP:9001/config](http://IP:9001/config)          | jq .\` |
| `/wifi_fence`                     | Estado geo‑cerca Wi‑Fi (config + estado en vivo)          | \`curl -s [http://IP:9001/wifi\_fence](http://IP:9001/wifi_fence) | jq .\` |

> **Notas:** `/chunk` sirve `Content-Type: application/x-ndjson` y descomprime `gz` al vuelo. `range` usa **clave de hora** `YYYYMMDD_HH`.

### 7.2 Endpoints **POST/PUT/DELETE**

| Endpoint                 | Body JSON                                                                                                                                                                        | Descripción                                                                                                              |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `/config` (POST/PUT)     | `{ "latency": { ... } }`                                                                                                                                                         | Actualiza **solo** claves `speed_0..speed_3, above`; valida `interval(1..300)`, `max(>=0)`, `heartbeat` (solo `speed_0`) |
| `/wifi_fence` (POST/PUT) | `{ "enabled":true, "radio":"radio0", "lat":-34.5, "lon":-58.56, "enter_m":200, "exit_m":260 }` **o** `{ "zones":[{"name":"X","lat":..,"lon":..,"enter_m":..,"exit_m":..},...] }` | Activa/ajusta geo‑cerca simple o **multi‑zona**                                                                          |
| `/wifi_fence` (DELETE)   | —                                                                                                                                                                                | Desactiva geo‑cerca y **enciende** el Wi‑Fi (disabled=0)                                                                 |

**Comportamiento Wi‑Fi:** cada 5 s evalúa distancia a zonas. **Adentro ⇒ apaga Wi‑Fi** (`wireless.radioX.disabled=1`); **afuera ⇒ prende** (`=0`). Anti‑rebote: cambio mínimo cada 30 s.

## 8) Envío a servidor central (Servicio 2)

| Parámetro  | Valor por defecto                                                      |
| ---------- | ---------------------------------------------------------------------- |
| Protocolo  | **UDP**                                                                |
| Destino    | `192.168.11.8:9000`                                                    |
| Fuente     | Última línea del NDJSON del slot más reciente (`/tmp/gps_logs/`)       |
| Frecuencia | Hasta 2 Hz (chequeo cada 500 ms; solo envía si hay línea nueva válida) |
| Validación | Verifica que la línea sea JSON válido antes de enviar                  |

> Sugerencia: si el backend requiere compresión o cambio de protocolo, este servicio puede adaptarse (HTTP/TCP) manteniendo el mismo buffer local.

**Flujo de recuperación (brechas):**

1. Detecta reconexión. 2) Lista archivos por hora. 3) Compara con servidor. 4) Envía solo lo faltante. 5) Marca confirmación.

---

## 9) Configuración

### 9.1 Archivo `serial_accepted.json`

* Mantiene `serial` válidos y `last_connect`.
* Usado por el monitor para decidir reenvíos.

### 9.2 `wifi_fence.json`

* Zonas: `[{"name":"...","lat":...,"lon":...,"enter_m":...,"exit_m":...}]`
* Estado: `enabled`, radio objetivo `radio0|radio1`.

### 9.3 `tx_config.json`

* `mode`, `server`, `batch_size`, `gzip`, `timeouts`.

---

## 10) Seguridad

* **Red:** acceso API limitado a LAN/VPN.
* **Autenticación:** token simple o lista de IPs permitidas (opcional).
* **Integridad:** timestamps en UTC, validación de JSON.

---

## 11) Salud y monitoreo

* **Endpoints salud:** `/state`.
* **Métricas:** tamaño de cola local, latencia de envío, errores consecutivos.
* **Logs:** `logread -e gps`.

---

## 12) Solución de problemas (FAQ)

* **No responde `/wifi_fence`:** reiniciar servicio API; verificar puerto `9001` en escucha.
* **No rota/comprime:** revisar permisos en carpeta de logs; hora UTC.
* **No envía al servidor:** confirmar DNS, `host:port`, salida WAN y políticas de firewall.

---

## 13) Versionado

* `version.json` con `{ "api":"v{{X.Y}}", "build":"{{fecha}}" }`.
* Mantener **changelog** por release.

---

## 14) Glosario

* **NDJSON:** JSON línea‑a‑línea, facilita streaming y parsing.
* **GZIP:** compresión estándar.
* **UTC:** hora universal coordinada.

---

## 15) Apéndice A – Comandos útiles

```sh
# Ver servicio y puerto
top | head -n 5
netstat -ltnp | grep 9001

# Probar API local
curl -s http://127.0.0.1:9001/state | jq .

# Enumerar archivos locales
ls -lh /tmp/gps_logs/ /overlay/gps_logs/
```

---

## 16) Apéndice B – Ejemplos de integración

* **Descarga horarios**: desde un colector (server) invocar `/range` por serial.
* **Realtime**: si se requiere, combinar con un backend que retransmita por SSE/WS lo recibido por UDP.

---

## 17) Apéndice C – Ejemplos con dispositivo demo

> **Demo:** `SERIAL = WFTXFW321903325` · `HOST = 172.16.20.20` · `PORT = 9001`

### 17.1 Listados de archivos

```sh
# Todos los archivos (horarios .gz y trozos de 10 min .ndjson)
curl -s http://172.16.20.20:9001/list_all/WFTXFW321903325 | jq .

# Solo horas cerradas (gzip)
curl -s http://172.16.20.20:9001/list/WFTXFW321903325 | jq -r '.[]' | tail -n 10

# Horas disponibles (clave YYYYMMDD_HH)
curl -s http://172.16.20.20:9001/range/WFTXFW321903325 | jq .
```

**Ejemplo de salida**

```json
{
  "gz": [
    "WFTXFW321903325-20250816_03.ndjson.gz",
    "WFTXFW321903325-20250816_04.ndjson.gz"
  ],
  "tmp": [
    "WFTXFW321903325-20250816_0330.ndjson",
    "WFTXFW321903325-20250816_0340.ndjson"
  ]
}
```

### 17.2 Descarga de rango horario (NDJSON concatenado)

```sh
# Descarga las horas 03 y 04 (incluidas) del 2025-08-16
curl -s "http://172.16.20.20:9001/chunk/WFTXFW321903325/20250816_03-20250816_04" | head
```

> **Nota:** El endpoint devuelve `Content-Type: application/x-ndjson` y descomprime `.gz` al vuelo.

### 17.3 Configuración de latencia

```sh
# Ver configuración efectiva
curl -s http://172.16.20.20:9001/config | jq .

# Actualizar latencia (ejemplo)
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{
        "latency": {
          "speed_0": {"max":0.6, "interval":20, "heartbeat":true},
          "speed_1": {"interval":8},
          "speed_2": {"interval":5},
          "speed_3": {"interval":1},
          "above":   {"interval":1}
        }
      }' \
  http://172.16.20.20:9001/config | jq .
```

### 17.4 Geo‑cerca Wi‑Fi

```sh
# Ver estado y configuración
curl -s http://172.16.20.20:9001/wifi_fence | jq .

# Configurar UNA zona (legacy)
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"enabled":true,"lat":-34.5159,"lon":-58.5603,"enter_m":200,"exit_m":260,"radio":"radio0"}' \
  http://172.16.20.20:9001/wifi_fence | jq .

# Configurar MULTI‑ZONA
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{
        "enabled": true,
        "radio": "radio0",
        "zones": [
          {"name":"Galpon Norte","lat":-34.5159,"lon":-58.5603,"enter_m":200,"exit_m":260},
          {"name":"Taller Central","lat":-34.60374,"lon":-58.38157,"enter_m":150,"exit_m":220}
        ]
      }' \
  http://172.16.20.20:9001/wifi_fence | jq .

# Desactivar geo‑cerca (enciende el Wi‑Fi del radio indicado)
curl -s -X DELETE http://172.16.20.20:9001/wifi_fence | jq .
```

---

## 18) Apéndice D – Checklist de instalación (OpenWrt)

### 18.1 Requisitos

* **Runtime:** MicroPython y BusyBox `gzip`.
* **Directorios:**

  ```sh
  mkdir -p /tmp/gps_logs /overlay/gps_buffer /dev/shm
  ```
* **Identidad del equipo:** `/etc/mac_serial` (1ª línea MAC, 2ª línea SERIAL)

  ```
  B0:CE:18:10:12:E7
  WFTXFW321903325
  ```
* **Config base:** `/etc/gps_fields.json` (latency por defecto)

  ```json
  {
    "latency": {
      "speed_0": {"max":0.5,  "interval":30, "heartbeat":true},
      "speed_1": {"max":5,    "interval":10},
      "speed_2": {"max":14,   "interval":5},
      "speed_3": {"max":22,   "interval":1},
      "above":   {             "interval":1}
    }
  }
  ```

### 18.2 Instalación de servicios

```sh
# Copiar scripts
install -m 0755 /usr/bin/gps_tracker.py /usr/bin/gps_tracker.py
install -m 0755 /usr/bin/gps_udp.py     /usr/bin/gps_udp.py
install -m 0755 /usr/bin/gps_api.py     /usr/bin/gps_api.py
```

**Init.d (procd) recomendado**

* `/etc/init.d/gps_tracker`

```sh
#!/bin/sh /etc/rc.common
START=70
USE_PROCD=1
start_service() {
  procd_open_instance
  procd_set_param command /usr/bin/micropython /usr/bin/gps_tracker.py
  procd_set_param respawn 1 5 5
  procd_close_instance
}
```

* `/etc/init.d/gps_udp`

```sh
#!/bin/sh /etc/rc.common
START=71
USE_PROCD=1
start_service() {
  procd_open_instance
  procd_set_param command /usr/bin/micropython /usr/bin/gps_udp.py
  procd_set_param respawn 1 5 5
  procd_close_instance
}
```

* `/etc/init.d/gps_api`

```sh
#!/bin/sh /etc/rc.common
START=72
USE_PROCD=1
start_service() {
  procd_open_instance
  procd_set_param command /usr/bin/micropython /usr/bin/gps_api.py
  procd_set_param respawn 1 5 5
  procd_close_instance
}
```

```sh
chmod +x /etc/init.d/gps_*
/etc/init.d/gps_tracker enable && /etc/init.d/gps_tracker start
/etc/init.d/gps_udp     enable && /etc/init.d/gps_udp start
/etc/init.d/gps_api     enable && /etc/init.d/gps_api start
```

### 18.3 Red y firewall

```sh
# Permitir API 9001 desde LAN
uci -q add firewall rule
uci set firewall.@rule[-1].name='Allow-API-9001-LAN'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest_port='9001'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall restart
```

### 18.4 Wi‑Fi fence (UCI)

* Radio por defecto: `radio0` (ajustable por API).
* Ver estado: `uci get wireless.radio0.disabled` (1=apagado, 0=encendido).
* La API usa `uci set wireless.radioX.disabled=<0|1>` + `wifi reload`.

---

Área IT Development © MicroTV - 2025

---
