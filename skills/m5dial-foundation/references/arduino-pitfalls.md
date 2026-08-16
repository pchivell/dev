# M5Dial Arduino Compiler Pitfalls & Memory Tips

## Contents
- Raw string delimiter rule
- No HTML entities with # inside raw strings
- No JS backtick template literals inside raw strings
- broadcastTXT lvalue error
- Deferred WiFi connect pattern
- Injecting runtime values into HTML strings
- Memory tips

---

## Raw string delimiter rule

The Arduino IDE preprocessor incorrectly scans inside raw string literals for the
default `)"` terminator. Any HTML that contains `onclick="fn()">` or similar ends
the string early, causing the JS and HTML to spill into C++ and produce hundreds of
cryptic errors.

ALWAYS use the custom `HTML` delimiter:

```cpp
// WRONG — )" inside onclick="..." terminates string early
String html = R"(<button onclick="reset()">Click</button>)";

// CORRECT — only )HTML" terminates; that sequence never appears in HTML
String html = R"HTML(<button onclick="reset()">Click</button>)HTML";
```

This applies to every raw string that contains HTML, CSS, or JavaScript.

---

## No HTML entities with # inside raw strings

The Arduino preprocessor interprets `#` as a preprocessor directive even inside
raw string literals. HTML entities like `&#x1F527;` contain `#` and trigger
a "stray '#' in program" error.

```cpp
// WRONG — preprocessor chokes on &#x...
R"HTML(<button>&#x1F527; Settings</button>)HTML"

// CORRECT — use plain text or copy the actual UTF-8 character
R"HTML(<button>Settings</button>)HTML"
```

---

## No JS backtick template literals inside raw strings

Backtick (`) characters are not valid in C++ source outside of nothing — the
compiler treats them as stray characters. JavaScript template literals inside
a raw string will cause "stray '`' in program" errors.

```cpp
// WRONG
R"HTML(<script>
  return `M ${a[0]} A ${R} ${R} 0 1 ${b[0]} ${b[1]}`;
</script>)HTML"

// CORRECT — use regular string concatenation in JS
R"HTML(<script>
  return 'M ' + a[0] + ' A ' + R + ' ' + R + ' 0 1 ' + b[0] + ' ' + b[1];
</script>)HTML"
```

---

## broadcastTXT lvalue error

`WebSocketsServer::broadcastTXT` takes `String&` (non-const reference).
Passing a temporary value causes: *cannot bind non-const lvalue reference of
type 'String&' to an rvalue of type 'String'*.

```cpp
// WRONG
wsServer.broadcastTXT(String(val));
wsServer.broadcastTXT("V:" + String(val));

// CORRECT — always use a named variable
String msg = "V:" + String(val);
wsServer.broadcastTXT(msg);
```

---

## Deferred WiFi connect pattern

Calling `WiFi.mode()`, `WiFi.begin()`, or `WiFi.disconnect()` from inside an
active HTTP handler callback corrupts the ESP32 TCP/WiFi stack. The symptom is
that `WiFi.status()` never becomes `WL_CONNECTED` after the form is submitted,
and the device falls back to AP mode.

Pattern — set a flag in the handler, act on it in the next loop iteration:

```cpp
bool pendingConnect = false;

// Inside HTTP POST handler:
void handleSaveCredentials() {
  // ... validate, save to NVS, send HTTP response ...
  pendingConnect = true;   // do NOT call startConnecting() here
}

// At the TOP of loop(), before any state processing:
void loop() {
  httpServer.handleClient();   // runs the handler if request pending

  if (pendingConnect) {
    pendingConnect = false;
    delay(200);              // let TCP flush the response to the browser
    startConnecting(storedSSID, storedPass);
    return;
  }
  // ... rest of loop ...
}
```

---

## Injecting runtime values into HTML strings

You cannot put a runtime C++ variable (like the device IP) inside a compile-time
raw string literal. Split the string and concatenate:

```cpp
String buildGaugePage() {
  String ip = WiFi.localIP().toString();

  String html = R"HTML(
    <script>
    // ... static JS here ...
    )HTML";

  // Inject runtime value between raw string segments:
  html += "var WS_URL = 'ws://" + ip + ":81/';\n";

  html += R"HTML(
    // ... rest of JS ...
    </script>
  )HTML";

  return html;
}
```

---

## Memory tips

The ESP32-S3 on the M5Dial has 8 MB PSRAM and 8 MB flash, but the default heap
available to Arduino sketches is limited. Follow these rules to avoid OOM crashes:

1. **Build HTML on demand** — never store the gauge page HTML in a global `String`
   variable. Build it inside the HTTP handler function so it is freed after the
   response is sent. The string can be 4–8 KB; keeping it global wastes heap
   permanently.

2. **Keep WebSocket messages short** — prefix + a few digits (e.g. `V:73`, `T:120,95`).
   Under 20 bytes each. The library allocates a heap buffer per message per client.

3. **Keep particle / animation logic in the browser** — all canvas/particle state
   lives in browser JS heap. The ESP32 only sends small event strings, not frame data.

4. **Avoid `String` fragmentation in tight loops** — if you must build strings
   repeatedly, use a fixed `char` buffer with `snprintf` instead.

5. **`delay(10)` at the end of loop()** — gives the ESP32 WiFi/TCP stack its
   background task time. Removing it can cause watchdog resets or dropped packets.

6. **Do not use `Serial.println` in production** — it adds latency and can
   interfere with timing-sensitive WiFi operations. Remove debug prints before
   final flash.
