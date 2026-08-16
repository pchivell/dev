# M5Dial WiFi Manager & WebSocket Reference

## Contents
- WiFi state machine overview
- NVS credential storage
- AP mode and captive portal
- Connecting and reconnecting
- Background probe (auto-reconnect from AP)
- WebSocket server setup
- WebSocket message protocol
- Pushing messages from ESP32
- Ports used

---

## WiFi state machine overview

Four states — always held in a global `WifiState` enum:

```
STATE_AP          — running open AP + captive portal
STATE_CONNECTING  — STA connection attempt in progress
STATE_CONNECTED   — connected, gauge server + WS server running
STATE_RECONNECTING — lost connection, retrying silently
```

Transitions:
```
Boot
 ├─ no stored creds  ──> STATE_AP
 └─ stored creds     ──> STATE_CONNECTING
                              ├─ success  ──> STATE_CONNECTED
                              └─ timeout  ──> STATE_AP

STATE_CONNECTED
 └─ link drops ──> STATE_RECONNECTING
                        ├─ reconnects (10s retries)  ──> STATE_CONNECTED
                        └─ 2 min timeout             ──> STATE_AP

STATE_AP
 └─ background probe every 10s (if creds stored)
       └─ success ──> STATE_CONNECTED  (no user action needed)
```

Timing constants (adjust as needed):
```cpp
static const unsigned long CONNECT_TIMEOUT    = 15000;   // ms for initial connect
static const unsigned long RECONNECT_INTERVAL = 10000;   // ms between retries
static const unsigned long AP_FALLBACK_AFTER  = 120000;  // 2 min before AP fallback
```

---

## NVS credential storage

Use the ESP32 `Preferences` library to persist SSID/password across reboots.

```cpp
#include <Preferences.h>
Preferences prefs;

void saveCredentials(const String& ssid, const String& pass) {
  prefs.begin("wifi-cfg", false);
  prefs.putString("ssid", ssid);
  prefs.putString("pass", pass);
  prefs.end();
}

bool loadCredentials(String& ssid, String& pass) {
  prefs.begin("wifi-cfg", true);   // true = read-only
  ssid = prefs.getString("ssid", "");
  pass = prefs.getString("pass", "");
  prefs.end();
  return ssid.length() > 0;
}

void clearCredentials() {
  prefs.begin("wifi-cfg", false);
  prefs.clear();
  prefs.end();
}
```

In `setup()`:
```cpp
String storedSSID, storedPass;
if (loadCredentials(storedSSID, storedPass))
  startConnecting(storedSSID, storedPass);
else
  startAPMode();
```

---

## AP mode and captive portal

AP name: `M5Dial-Setup` (open, no password).
AP IP: `192.168.4.1`.

```cpp
#include <DNSServer.h>
DNSServer dnsServer;
static const IPAddress AP_IP(192, 168, 4, 1);

void startAPMode() {
  WiFi.mode(WIFI_AP);
  WiFi.softAPConfig(AP_IP, AP_IP, IPAddress(255,255,255,0));
  WiFi.softAP("M5Dial-Setup");

  dnsServer.setErrorReplyCode(DNSReplyCode::NoError);
  dnsServer.start(53, "*", AP_IP);   // wildcard DNS -> captive portal

  // Register captive portal detection endpoints for iOS/Android/Windows:
  auto redir = []() {
    httpServer.sendHeader("Location", "http://192.168.4.1/", true);
    httpServer.send(302, "text/plain", "");
  };
  httpServer.on("/hotspot-detect.html",       HTTP_GET, redir);
  httpServer.on("/library/test/success.html", HTTP_GET, redir);
  httpServer.on("/generate_204",              HTTP_GET, redir);
  httpServer.on("/gen_204",                   HTTP_GET, redir);
  httpServer.on("/connecttest.txt",           HTTP_GET, redir);
  httpServer.on("/redirect",                  HTTP_GET, redir);
  httpServer.on("/ncsi.txt",                  HTTP_GET, redir);
  httpServer.on("/", HTTP_GET, []() { httpServer.send(200,"text/html", buildPortalPage()); });
  httpServer.on("/save", HTTP_POST, handleSaveCredentials);
  httpServer.onNotFound(redir);
  httpServer.begin();
}
```

Always call in `loop()`:
```cpp
dnsServer.processNextRequest();
httpServer.handleClient();
```

---

## Connecting and reconnecting

```cpp
void startConnecting(const String& ssid, const String& pass) {
  wsServer.close();
  httpServer.stop();
  dnsServer.stop();
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), pass.c_str());
  connectStartTime = millis();
  wifiState = STATE_CONNECTING;
}
```

CRITICAL — deferred connect pattern:
Never call `startConnecting()` from inside an HTTP handler callback.
Switching WiFi mode mid-callback corrupts the ESP32 network stack.
Use a flag instead:

```cpp
bool pendingConnect = false;

void handleSaveCredentials() {
  // ... save creds, send HTTP response ...
  pendingConnect = true;   // <-- flag only, no WiFi calls here
}

void loop() {
  httpServer.handleClient();
  if (pendingConnect) {
    pendingConnect = false;
    delay(200);   // let TCP flush the HTTP response
    startConnecting(storedSSID, storedPass);
    return;
  }
  // ...
}
```

---

## Background probe (auto-reconnect from AP)

While in `STATE_AP`, probe the stored network every `RECONNECT_INTERVAL` ms
using `WIFI_AP_STA` so the AP stays live for the user while the probe runs:

```cpp
if (storedSSID.length() > 0 && millis() - lastReconnectTry > RECONNECT_INTERVAL) {
  lastReconnectTry = millis();
  WiFi.mode(WIFI_AP_STA);
  WiFi.begin(storedSSID.c_str(), storedPass.c_str());
  unsigned long t = millis();
  while (millis() - t < 8000 && WiFi.status() != WL_CONNECTED) {
    dnsServer.processNextRequest();
    httpServer.handleClient();
    delay(100);
  }
  if (WiFi.status() == WL_CONNECTED) {
    WiFi.mode(WIFI_STA);
    wifiState = STATE_CONNECTED;
    startGaugeServer();
  } else {
    WiFi.mode(WIFI_AP);
  }
}
```

---

## WebSocket server setup

```cpp
#include <WebSocketsServer.h>   // "WebSockets by Markus Sattler"
WebSocketsServer wsServer(81);  // port 81; HTTP server uses port 80

void startGaugeServer() {
  wsServer.begin();
  wsServer.onEvent([](uint8_t num, WStype_t type, uint8_t* payload, size_t len) {
    // Server-push only — no client messages expected
  });
}

// In loop() — keep WS alive:
wsServer.loop();
```

---

## WebSocket message protocol

All messages are short ASCII strings prefixed with a type character.
Keep every message under 20 bytes to minimise heap allocation.

| Prefix | Example | Meaning |
|---|---|---|
| `V:` | `V:73` | Gauge/encoder value changed (0–100) |
| `T:` | `T:120,95` | Screen touched at dial coords (0–239 each axis) |
| `B` | `B` | Centre button was pressed |

Add new event types by choosing a new single-letter prefix.

---

## Pushing messages from ESP32

`broadcastTXT` requires a named `String&` — passing a temporary crashes the compiler:

```cpp
// WRONG — compiler error: cannot bind non-const lvalue reference to rvalue
wsServer.broadcastTXT(String(val));

// CORRECT
void wsPushValue(int val) {
  String msg = "V:" + String(val);
  wsServer.broadcastTXT(msg);
}

void wsPushTouch(int x, int y) {
  String msg = "T:" + String(x) + "," + String(y);
  wsServer.broadcastTXT(msg);
}

void wsPushButton() {
  String msg = "B";
  wsServer.broadcastTXT(msg);
}
```

Only push when actually connected:
```cpp
if (wifiState == STATE_CONNECTED) {
  wsPushValue(gaugeValue);
}
```

---

## Ports used

| Port | Protocol | Purpose |
|---|---|---|
| 53 | UDP | DNS — captive portal wildcard redirect (AP mode only) |
| 80 | TCP/HTTP | Captive portal config page OR live gauge page |
| 81 | TCP/WS | WebSocket — live value/event push to browser |
