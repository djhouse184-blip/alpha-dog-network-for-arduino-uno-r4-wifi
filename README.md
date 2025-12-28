
# Alpha Dog Minimal Host (Arduino R4 WiFi)  https://sites.google.com/view/arduinounor4wifinetwork/  

A minimal HTTP + UDP host for an Arduino R4 WiFi that runs as a Wi‑Fi Access Point, serves a small offline web dashboard, and accepts short UDP messages from networked devices (for example: ESP32 BLE scanners). Use this README as documentation you can paste into your repository.

## What this code is
This sketch turns the R4 into a small local host that:

- Creates a Wi‑Fi Access Point with SSID `AlphaDogNet` (password `alphadog123`).
- Serves a compact single‑page web dashboard (HTML/CSS/JS embedded in PROGMEM) on port 80.
- Exposes several JSON API endpoints (under `/api/`) to get host status, list registered devices, run a simple UDP broadcast, and more.
- Listens on UDP port `4210` for short messages from devices and registers devices that send `id:<name>:...` messages.
- Keeps an in‑RAM table (max 10 entries by default) of recently seen devices and their IP + last seen timestamp.

It is intended for use in an isolated/local network (offline web UI, lightweight messaging) and pairs well with battery-powered or Wi‑Fi/BLE devices such as ESP32s that send UDP or HTTP reports.

## Key behaviors / components

- WiFi AP:
  - SSID: `AlphaDogNet`
  - Password: `alphadog123`
  - The R4 runs as Access Point using `WiFi.beginAP(...)`.
  - `apIP` is stored after startup (printed on Serial).

- Web server:
  - Listens on port 80 using `WiFiServer server(80)`.
  - Serves embedded dashboard HTML for non-API paths.
  - API endpoints (JSON responses) are under `/api/`.

- API endpoints:
  - GET `/api/ping` → responds with `{"result":"pong","uptime":<ms>}`
  - GET `/api/status` → responds with `{"status":"online","deviceCount":<n>,"ip":"<apIP>"}`
  - GET `/api/devices` → responds with `{"devices":[{"id":"...","ip":"...","lastSeenSec":...}, ...]}`
  - POST `/api/broadcast` → accepts JSON `{"message":"..."};` broadcasts that string via UDP to 255.255.255.255 on port 4210 and returns `{"ok":true,"message":"..."}`

  Example curl:
  - Ping: `curl http://192.168.4.1/api/ping`
  - List devices: `curl http://192.168.4.1/api/devices`
  - Broadcast:
    ```
    curl -X POST -H "Content-Type: application/json" \
      -d '{"message":"ping"}' \
      http://192.168.4.1/api/broadcast
    ```

- UDP listener:
  - Listens on port `4210`.
  - When a UDP payload starts with `id:`, it expects a second colon and treats the content between `id:` and the next `:` as the device id/name:
    - Example messages:
      - `id:myDevice:hello` → registers device id `myDevice` with sender IP
      - `id:AA:BB:CC:DD:EE:FF:seen` → registers `AA:BB:CC:DD:EE:FF` (MAC)
  - When a message begins with `wifi:` it is printed to Serial (hook for future parsing).

- Device storage:
  - `DeviceInfo devices[MAX_DEVICES]` — array of up to 10 devices by default.
  - Each entry stores `id` (String), `ip` (IPAddress) and `lastSeen` (millis()).
  - `registerDevice(id, ip)` will add or refresh a device entry, updating `lastSeen`.

- Frontend dashboard:
  - Embeds a compact HTML/CSS/JS app in `INDEX_HTML` (stored in PROGMEM) that:
    - Shows host status, device list, and an event log.
    - Can call the `/api/*` endpoints to ping, list devices, and broadcast UDP messages.
    - Includes a simple notepad and commands panel for convenience.

## How to use / test

1. Flash the sketch to your Arduino R4 WiFi.
2. Open Serial Monitor at 115200 — you should see:
   - "Alpha Dog Minimal Host starting..."
   - AP start status and AP IP (commonly `192.168.4.1`).
3. Connect a laptop/phone/ESP32 to Wi‑Fi SSID `AlphaDogNet` using password `alphadog123`.
4. Open `http://<apIP>/` in a browser (e.g., `http://192.168.4.1/`) to view the dashboard.
5. Send UDP messages from devices to port `4210` (broadcast or direct to the R4 IP).
   - Example from Linux to broadcast:
     ```
     echo -n "id:esp32-01:hello" | nc -u -w1 192.168.4.255 4210
     ```
   - Or using an ESP32, send UDP to port 4210 and the R4 will register the device id.

6. Use dashboard buttons to fetch `/api/devices` or broadcast messages.

## Connecting an ESP32 BLE scanner
Two commonly used reporting methods:

- UDP broadcast (recommended with current sketch):
  - ESP32 sends `id:<id>:seen` via UDP to broadcast address on port `4210`.
  - R4 registers that id and shows it on the dashboard.

- HTTP POST (optional):
  - ESP32 posts JSON to `http://<apIP>/api/report` (not implemented by default in the sketch).
  - If you prefer HTTP POST reporting, add the small `handleApiReport` handler (example below) and a branch in `handleHttpClient`.

Optional `/api/report` handler to add to the sketch:
```cpp
// Add this to the API handlers in the R4 sketch:
void handleApiReport(WiFiClient &client, const String &body) {
  String mac = extractJsonField(body, "mac");
  if (mac.length() == 0) {
    sendJsonHeader(client, 400, "Bad Request");
    client.println("{\"error\":\"no mac\"}");
    return;
  }
  IPAddress remoteIP = client.remoteIP(); // may be available
  if (remoteIP == (uint32_t)0) remoteIP = apIP;
  registerDevice(mac, remoteIP);
  sendJsonHeader(client, 200, "OK");
  client.println("{\"ok\":true}");
}
// And in handleHttpClient add:
} else if (path == "/api/report") {
  handleApiReport(client, body);
}
```

ESP32 sample reporting JSON (HTTP): `{"mac":"AA:BB:CC:DD:EE:FF","rssi":-62}`

## Important notes / caveats

- Broadcast address: the sketch uses `255.255.255.255` to broadcast. On some networks you might prefer the AP subnet broadcast (e.g. `192.168.4.255`) or send directly to the R4 IP.
- Limited RAM/slots: device list is small (10 entries). Increase `MAX_DEVICES` if needed but watch RAM.
- No persistent storage: registered devices are in RAM only and will be lost on reboot.
- MAC randomization: many phones randomize BLE MACs — these appear as transient IDs. Expect variable item churn if tracking phones.
- Rate limiting: BLE scanners can produce many events; add dedupe or rate limits on the scanner side to avoid flooding the R4.
- Security: this is plaintext local network traffic. If you deploy on shared networks, add authentication or tokens to APIs.
- Serial debugging: useful to watch UDP messages and API handling.

## Libraries & build
- This sketch uses these platform libraries (included with Arduino core for R4/WiFiS3):
  - WiFiS3 (WiFi AP, WiFiServer)
  - WiFiUdp (UDP)
- Build for Arduino R4 WiFi / appropriate core in the Arduino IDE or PlatformIO.

## Quick file map
- `*.ino` — this main sketch (contains web UI and server logic embedded)
- PROGMEM `INDEX_HTML` — single-file offline dashboard

## Troubleshooting
- If the AP fails to start: Serial prints `AP start failed` — check WiFiS3 status and ensure no conflicts.
- If web UI doesn't load: verify you are connected to `AlphaDogNet` and use the AP IP printed on Serial.
- If UDP packets are not seen: verify your sender uses the correct port `4210`, correct broadcast address, and that the sender is on the same AP network.

## License
Place your preferred license here (MIT, Apache, etc.) — currently none is included in the sketch.

---

If you want, I can:
- produce a trimmed README variant for a repo root,
- add an example ESP32 UDP reporter sketch,
- or create a ready-to-paste badge/header and LICENSE file.

Which would you like next?
