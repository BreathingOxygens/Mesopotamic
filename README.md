# IoT Dashboard — Node-RED + MQTT + REST API

Dashboard monitoring suhu & kelembaban (simulasi ESP32) dengan backend Node-RED dan broker Mosquitto.

## Fitur

**Dasar**
- Simulator ESP32 (Python) publish data ke MQTT `iot/sensor` tiap 3 detik
- Node-RED subscribe MQTT, simpan data, dan sediakan REST API
- Dashboard web: kartu sensor, tabel history, kontrol lampu (publish ke `iot/kontrol`)

**Pengembangan tambahan (nilai tambah)**
1. **Endpoint `PUT /api/threshold`** untuk set batas suhu, dengan notifikasi saat suhu melewati batas
   - Notifikasi dikirim lewat MQTT topic `iot/alert`, banner merah + log di dashboard, dan notifikasi browser
   - Notifikasi hanya dikirim saat suhu *baru* melewati batas (tidak spam tiap data)
2. **Grafik real-time Chart.js** di dashboard (suhu, kelembaban, dan garis batas suhu)
3. **Autentikasi API key** lewat header `X-API-Key` pada semua endpoint `/api/*`

## Struktur file

| File | Fungsi |
|---|---|
| `flow_rest_api_iot.json` | Flow Node-RED (import ke Node-RED) |
| `dashboard.html` | Dashboard web |
| `simulasi_esp32.py` | Simulator sensor ESP32 |

## Cara menjalankan

1. Pastikan Mosquitto berjalan di `localhost:1883`.
2. Jalankan Node-RED. API key bisa diatur lewat environment variable (opsional, default `iot-secret-123`):
   ```bash
   API_KEY=rahasia-saya node-red
   ```
   Di Windows (PowerShell): `$env:API_KEY="rahasia-saya"; node-red`
3. Buka `http://localhost:1880`, menu **Import** → pilih `flow_rest_api_iot.json` → **Deploy**.
4. Jalankan simulator:
   ```bash
   pip install paho-mqtt
   python simulasi_esp32.py
   ```
5. Buka `dashboard.html` di browser. Jika API key diubah, samakan nilai `API_KEY` di bagian atas `<script>`.

## Daftar endpoint

Semua endpoint wajib menyertakan header `X-API-Key: <key>`. Tanpa key atau key salah, server membalas `401`.

| Method | Path | Fungsi |
|---|---|---|
| GET | `/api/suhu` | Data sensor terakhir + batas suhu + status melewati batas |
| GET | `/api/history` | Maksimal 50 data sensor terakhir |
| POST | `/api/kontrol` | Kirim perintah, body `{"device":"lamp","action":"on"}` |
| PUT | `/api/threshold` | Set batas suhu, body `{"max": 30}` |
| GET | `/api/threshold` | Lihat batas suhu saat ini |
| GET | `/api/alerts` | Daftar notifikasi terakhir (maks 20) |

## Contoh pengujian (curl)

```bash
# Tanpa API key -> 401
curl -i http://localhost:1880/api/suhu

# Dengan API key -> 200
curl -H "X-API-Key: iot-secret-123" http://localhost:1880/api/suhu

# Set batas suhu jadi 28 °C
curl -X PUT http://localhost:1880/api/threshold \
  -H "Content-Type: application/json" \
  -H "X-API-Key: iot-secret-123" \
  -d '{"max": 28}'

# Lihat notifikasi
curl -H "X-API-Key: iot-secret-123" http://localhost:1880/api/alerts

# Pantau notifikasi lewat MQTT
mosquitto_sub -t iot/alert
```

## Catatan

- Data history dan batas suhu disimpan di *flow context* Node-RED (hilang jika Node-RED restart).
- API key di file HTML bisa dilihat siapa saja yang membuka dashboard, jadi ini autentikasi sederhana untuk tugas, bukan untuk produksi.
- Jika browser menolak request karena CORS, pastikan flow versi v2 yang ter-deploy (header `Access-Control-Allow-Headers` sudah memuat `X-API-Key`).
