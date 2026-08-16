# Domain: IoT / Edge

Applies to: sensor-to-cloud pipelines, device fleets, industrial IoT,
smart building/agriculture/logistics systems, edge gateways, and any
architecture spanning constrained devices, local edge processing, and
cloud back-ends.

---

## Domain Characteristics

- **Scale**: from a handful of devices to millions; fleet management is architectural, not operational
- **Connectivity**: unreliable; devices must tolerate disconnection gracefully
- **Data flow**: predominantly device → cloud (telemetry) with bidirectional control commands
- **Heterogeneity**: different device classes, OS versions, firmware versions coexist in the fleet
- **Security surface**: every device is a potential attack vector; physical access cannot be assumed secure
- **Lifecycle**: devices are deployed for years; OTA updates are mandatory

---

## Recommended Architecture

### Three-Tier Model

```
┌────────────────────────────────────────────────┐
│  Cloud Tier                                    │
│  Device Registry · Rules Engine · Data Store  │
│  API Gateway · Analytics · OTA Service        │
├────────────────────────────────────────────────┤
│  Edge / Gateway Tier (optional)               │
│  Local broker · Store-and-forward             │
│  Edge inference · Protocol translation        │
├────────────────────────────────────────────────┤
│  Device Tier                                  │
│  Sensors · Actuators · Firmware               │
└────────────────────────────────────────────────┘
```

**Use the Edge tier when:**
- Latency requirements < 50 ms (cloud round-trip too slow)
- Bandwidth is expensive or metered (pre-aggregate before upload)
- Devices cannot speak IP (Modbus, Zigbee, BLE → IP translation at edge)
- Regulations require data sovereignty (data must not leave premises)

### Protocol Selection

| Layer | Protocol | When |
|-------|----------|------|
| Device ↔ Cloud | **MQTT** | default for telemetry; low overhead; QoS levels |
| Device ↔ Cloud | **HTTPS/REST** | one-shot commands; simpler devices |
| Device ↔ Cloud | **CoAP** | very constrained devices (< 10 KB RAM) |
| Device ↔ Edge | **MQTT local broker** | LAN-local; store-and-forward on disconnect |
| Edge ↔ Cloud | **AMQP / Kafka** | high-throughput aggregation |
| Config / Commands | **MQTT shadow / device twin** | desired ↔ reported state pattern |

### Telemetry Data Shape

Always timestamp at the device (not the server) and include the device ID:

```json
{
  "device_id": "sensor-042",
  "ts": 1720000000,
  "payload": { "temp_c": 23.4, "humidity_pct": 61 },
  "fw_version": "1.2.3"
}
```

---

## OTA Update Architecture

1. **Dual-partition bootloader** on device (A/B slots)
2. **OTA service** in cloud: manifest + binary store + targeting rules (by device group, fw version)
3. **Device polls or receives push** for new manifest
4. Device downloads, verifies signature, writes to inactive partition
5. Reboots into new partition; reports success; old partition marked fallback
6. Cloud confirms rollout; deletes old version from fleet record

Never deploy OTA without rollback — a bad update bricking a fleet is catastrophic.

---

## Key Decision Checkpoints

1. **Edge tier needed?** Latency, bandwidth cost, offline autonomy, protocol translation?
2. **Managed IoT platform or DIY?** AWS IoT Core / Azure IoT Hub / Google Cloud IoT vs self-hosted MQTT broker.
3. **Device identity / PKI?** Each device needs a unique certificate or pre-shared key — provision at manufacture.
4. **Telemetry frequency vs cost?** High-frequency raw data vs on-device aggregation before upload.
5. **Offline autonomy?** What does the device do with no cloud connection? (store-and-forward, safe-state, degrade gracefully)
6. **Fleet size growth?** Connection limits, message limits, and cost structure of chosen broker.

---

## Common Pitfalls

- **No device identity** — shared credentials mean a compromised device compromises the fleet.
- **Server-side timestamps only** — clock skew and network latency corrupt time-series data.
- **No store-and-forward** — data is lost on every disconnect; buffer locally and flush on reconnect.
- **Polling instead of push for commands** — polling adds latency and wastes battery; use MQTT retained/QoS1.
- **Unversioned firmware in the field** — always track `fw_version` in telemetry; you need to know what's running.
- **No shadow/twin pattern** — without desired/reported state, device configuration races are inevitable.
