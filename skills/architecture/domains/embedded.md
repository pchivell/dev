# Domain: Embedded / Firmware

Applies to: microcontrollers (ESP32, STM32, RP2040, AVR, ARM Cortex-M),
bare-metal and RTOS systems, custom hardware, robotics controllers,
industrial PLCs, and any system where software runs directly on hardware
with tight resource constraints.

---

## Domain Characteristics

- **Memory**: RAM in kilobytes to low megabytes; flash in kilobytes to a few MB
- **CPU**: Single or dual core, 8–240 MHz; no virtual memory; no OS scheduler (bare metal) or cooperative/preemptive RTOS
- **Power**: Battery-operated or bus-powered; sleep modes are first-class concerns
- **Real-time**: Hard or soft deadlines driven by hardware interrupts and peripheral timing
- **I/O**: GPIO, SPI, I²C, UART, PWM, ADC, DMA — peripheral access is everything
- **Debugging**: Limited to JTAG/SWD, serial logging, logic analyser, oscilloscope

---

## Recommended Architecture

### Layer Model (outermost → innermost)

```
┌─────────────────────────────────────┐
│  Application Logic (state machines) │  ← business rules, sequences
├─────────────────────────────────────┤
│  Driver / HAL Abstraction           │  ← peripheral-agnostic interfaces
├─────────────────────────────────────┤
│  BSP / HAL Implementation           │  ← chip-specific register writes
├─────────────────────────────────────┤
│  Hardware                           │
└─────────────────────────────────────┘
```

- **Application layer** must never touch registers directly — always through the HAL interface.
- **HAL interfaces** are defined as C structs of function pointers (C) or pure-virtual classes (C++) — this enables unit testing without hardware.
- **BSP** (Board Support Package) ties a HAL implementation to one specific board/chip variant.

### Concurrency Model

| Approach | When to use |
|----------|------------|
| **Super-loop** (`while(1)`) | Simple, single-concern firmware; few peripherals; no timing pressure |
| **Interrupt-driven super-loop** | Time-critical ISRs, deferred processing in main loop via flag/queue |
| **Cooperative RTOS tasks** | Multiple concerns; deterministic scheduling not required |
| **Preemptive RTOS** (FreeRTOS, Zephyr) | Multiple hard deadlines; resource isolation; priority inversion must be managed |

### State Machine Pattern

All behaviour with modes or sequences MUST be modelled as an explicit finite state machine (FSM):

```c
typedef enum { STATE_IDLE, STATE_RUNNING, STATE_ERROR } AppState;

void app_tick(AppContext *ctx) {
    switch (ctx->state) {
        case STATE_IDLE:    handle_idle(ctx);    break;
        case STATE_RUNNING: handle_running(ctx); break;
        case STATE_ERROR:   handle_error(ctx);   break;
    }
}
```

Never use nested `if/else` chains to represent states — they become unmaintainable.

---

## Technology Selection Framework

| Decision | Default choice | Alternatives & when |
|----------|---------------|---------------------|
| RTOS | FreeRTOS | Zephyr (security/networking), ThreadX (safety-critical) |
| Language | C | C++ (classes/RAII OK; exceptions and RTTI off); Rust (safety-critical new designs) |
| Build system | CMake + Ninja | PlatformIO (Arduino ecosystem), ESP-IDF, Zephyr west |
| Comms stack | vendor SDK | lwIP (TCP/IP), Mbed TLS (TLS) |
| OTA | custom bootloader | MCUboot, ESP-IDF OTA partition scheme |

---

## Key Decision Checkpoints

Ask these before designing:

1. **Hard real-time required?** If yes → preemptive RTOS or ISR-driven architecture.
2. **Power budget?** If battery-operated → sleep modes from day one; design all I/O with wake sources.
3. **Connectivity?** WiFi/BLE adds a second CPU-heavy task; plan stack size and heap accordingly.
4. **OTA updates needed?** If yes → dual-partition flash layout required from the start (cannot be retrofitted easily).
5. **Unit testable?** If yes → HAL abstraction mandatory; application logic must compile on host.
6. **Safety-critical?** (medical, automotive, aviation) → MISRA C, DO-178, IEC 61508 compliance path from day one.

---

## Common Pitfalls

- **Blocking in ISRs** — ISRs must return in microseconds; use a flag or ring buffer, process in main loop.
- **Stack overflow** — every RTOS task needs a measured stack; use `uxTaskGetStackHighWaterMark` (FreeRTOS).
- **Shared state without critical sections** — any variable touched by both an ISR and main code needs `volatile` + atomic access or a mutex.
- **Magic numbers for peripherals** — register addresses and bit masks must be named constants, never literals.
- **No bootloader** — shipping without a bootloader means no field update path; always include one.
- **Floating point on cores without FPU** — soft-float is 10–100× slower; use fixed-point where possible.
- **Printf in production** — serial logging must be conditionally compiled out or replaced with a ring-buffer trace.
