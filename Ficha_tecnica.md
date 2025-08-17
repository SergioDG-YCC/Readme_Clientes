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
# Ficha técnica — Plataforma MT7628 (MicroPython) · Cliente

## 1) Alcance
- Adquisición GPS, almacenamiento local con rotación/compresión y envío a servidor.
- API HTTP local para consulta/control.
- Latencia adaptativa según velocidad.

## 2) Servicios
- **gps_tracker**: captura GPS, NDJSON (10 min), GZIP (1 h), `latest.json`.
- **gps_udp**: publica última muestra válida al servidor central.
- **gps_api**: API HTTP local (listados, rangos, chunk, configuración, geo-cerca Wi-Fi).

## 3) Datos y almacenamiento
- **Formato**: NDJSON (UTC) + GZIP por hora.
- **Carpetas**: `/tmp/gps_logs/` (10 min) · `/overlay/gps_buffer/` (1 h).
- **Retención**: limpieza automática por serial (p.ej., 48 h).
- **Integridad**: escrituras atómicas; compresión incremental por ventana.

## 4) Frecuencia / latencia (adaptativa)
- Configurable por **umbrales de velocidad (m/s)** e **intervalos (s)**.
- Detenido → intervalos largos; tránsito urbano → medios; ruta → cortos.
- “Heartbeat” opcional a velocidad ≈ 0.

## 5) API HTTP local (puerto 9001)
**GET**
- `/list_all/{serial}` — Archivos disponibles (GZIP horarios + NDJSON 10 min).
- `/list/{serial}` — Solo horas cerradas (GZIP).
- `/range/{serial}` — Horas disponibles `YYYYMMDD_HH`.
- `/chunk/{serial}/{desde}-{hasta}` — Flujo NDJSON concatenado del rango.

**POST/PUT**
- `/config` — Actualiza latencia (umbrales/intervalos/heartbeat).
- `/wifi_fence` — Geo-cerca: una o múltiples zonas; radio objetivo.

**DELETE**
- `/wifi_fence` — Desactiva geo-cerca (restaura radio).

## 6) Payload (campos)
- `time` (ISO-8601 UTC), `lat`, `lon`, `alt`, `speed` (m/s), `track` (°).
- Calidad: `sats`, `hdop`, `pdop`, `vdop`, `geoidSep`, `snr` (proxy).
- Identidad: `serial`, `mac`. Estado: `track_status` (`motion`/`stop`).

## 7) Envío a servidor
- **Protocolo**: UDP (actual; HTTP/TCP opcional).
- **Frecuencia**: hasta 2 Hz (solo nuevas muestras válidas).
- **Recuperación**: prioriza horas faltantes al reconectar (sin duplicados).

## 8) Geo-cerca Wi-Fi
- Zonas con `lat/lon` y radios `enter_m/exit_m` por `radioX`.
- Lógica: adentro → apaga Wi-Fi; afuera → enciende. Anti-rebote temporizado.

## 9) Seguridad
- API restringida a LAN/VPN (recomendado); token/allow-list opcional.
- Timestamps en UTC; validación de JSON.

## 10) Monitoreo
- Métricas: cola local, latencia de envío, errores consecutivos.
- Observabilidad vía endpoints y registros del sistema.

## 11) Requisitos mínimos
- MicroPython + BusyBox `gzip`; reloj UTC (GPS/NTP).
- Estructura de carpetas y permisos de escritura.

## 12) Versionado
- `version.json` con `api` y `build`; changelog por release.
---

Área IT Development © MicroTV - 2025

---
