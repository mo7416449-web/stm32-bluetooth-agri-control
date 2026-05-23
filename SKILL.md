---
name: stm32-bluetooth-agri-control
description: Design, implement, and review STM32-to-Windows Bluetooth serial communication for agricultural monitoring and control systems. Use when Codex needs to build or modify STM32 firmware, Windows Python pyserial integration, DATA/CMD/ACK/ERR text protocols, local LLM-assisted control pipelines, dual safety validation, relay/PWM command handling, reconnect/heartbeat behavior, or offline fallback rules for pumps, lights, fans, switches, and greenhouse sensors.
---

# STM32 Bluetooth Agri Control

## Overview

Use this skill for projects where STM32 handles edge sensing and hardware control while a Windows Python program receives data over Bluetooth serial, runs cleaning/aggregation/storage/LLM decision logic, validates actions, and sends safe commands back to STM32.

Default to a conservative architecture: PC-side intelligence, STM32-side execution and safety fallback. Do not recommend running an LLM on STM32 unless the task explicitly asks for TinyML-style classification.

## Architecture

Prefer this first-version route:

```text
sensors -> STM32 -> UART -> Bluetooth SPP module -> Windows virtual COM port
Windows Python -> cleaning -> aggregation -> SQLite -> mock/Ollama LLM -> validator
Windows virtual COM port -> Bluetooth SPP module -> STM32 -> relays/PWM/GPIO
```

Recommend classic Bluetooth SPP for the MVP because Windows exposes it as a COM port and Python can use `pyserial`. Avoid BLE for the first version unless the user specifically needs BLE/GATT/mobile-app support.

## Implementation Workflow

1. Clarify hardware: STM32 model, Bluetooth module, sensor list, relay/PWM outputs, Windows COM port, baudrate, and toolchain.
2. Start wired if possible: validate the protocol over USB-UART before replacing the cable with Bluetooth SPP.
3. Define line-delimited text protocol with sequence IDs and `\n` terminators.
4. Implement STM32 UART receive buffering with partial-line handling and bounded buffers.
5. Implement Windows Python reader/writer using one opened serial connection where possible.
6. Parse STM32 `DATA` messages into the existing project's sensor model or CSV-equivalent rows.
7. Run PC-side cleaning, aggregation, SQLite logging, mock/Ollama decision, and safety validation.
8. Convert validated actions to `CMD` lines and wait for `ACK` or `ERR`.
9. Store command status per action, not just per batch.
10. Add heartbeat/reconnect handling and STM32 offline fallback rules.

## Protocol Defaults

Use the protocol in `references/protocol.md` unless the repository already has a different established protocol.

Minimum message types:

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
HB;SEQ=3001
```

Rules:

- Use one physical line per message and terminate with `\n`.
- Include `SEQ` on commands and acknowledgements for traceability.
- Treat unknown keys as ignored, not fatal.
- Treat missing required keys as invalid and return `ERR`.
- Bound every receive buffer and reject oversized lines.
- Use stable ASCII tokens for device IDs, metric names, actions, and units.

## Windows Python Guidance

Use `pyserial` against the Windows Bluetooth COM port. Add configuration like:

```yaml
bluetooth:
  port: "COM9"
  baudrate: 9600
  timeout_seconds: 2
  reconnect_seconds: 5
```

When modifying an existing Python agriculture pipeline:

- Add a serial reader that converts `DATA` lines to sensor readings.
- Add a sender that writes `CMD` lines and collects `ACK`/`ERR`.
- Keep dry-run mode for command preview.
- Record raw incoming lines, parsed readings, validated decisions, generated commands, and per-command status.
- Do not let malformed serial input crash the main loop.
- Resolve relative paths from the config file directory, not from the current working directory.

## STM32 Firmware Guidance

Prefer these firmware modules:

```text
sensor.c       periodic sensor reads and validity flags
bt_uart.c      UART RX/TX, line buffering, timeout handling
protocol.c     parse DATA/CMD/ACK/ERR/HB key-value lines
control.c      GPIO, relay, PWM, pump/light/fan/switch drivers
safety.c       command limits and offline fallback rules
main.c         scheduler/state machine
```

STM32 must validate commands again even if the PC already validated them:

- Reject unknown devices and unknown actions.
- Clamp or reject pump duration above the configured maximum.
- Reject brightness outside `0..100`.
- Reject malformed numeric fields.
- Disable pump on low water-level or sensor-fault conditions.
- Enter safe mode if PC heartbeat is missing for the configured timeout.

## Offline Fallback

Use conservative local rules only:

```text
if pc_offline and soil_moisture_valid and soil_moisture < 20:
  run pump for at most 10-30 seconds

if pc_offline and temperature_valid and temperature > 38:
  enable ventilation/fan if configured

if sensor_fault or water_level_low:
  force pump off
```

The fallback goal is safety and continuity, not optimal crop control. Avoid adding complex autonomous behavior to STM32 unless the user explicitly asks for it.

## Review Checklist

Check these before calling a design or patch complete:

- The Bluetooth path is testable with a serial terminal before running the full pipeline.
- Partial serial lines are buffered and not parsed early.
- Every outbound command has a matching status record.
- PC-side and STM32-side safety limits are consistent.
- Reconnect and heartbeat behavior are defined.
- Dry-run remains the default until real hardware tests are requested.
- Unit tests cover malformed input, bad numeric values, unknown devices, timeout/reconnect, and send failures.
