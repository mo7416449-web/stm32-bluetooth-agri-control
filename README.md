# STM32 Bluetooth Agricultural Monitoring and Control

**A Codex skill for building reliable Bluetooth serial links between STM32 devices and agricultural control applications.**

[简体中文](README.zh-CN.md)

This repository provides a practical design guide and a line-based protocol reference for agricultural monitoring and control systems built with:

- STM32 for sensor acquisition, actuator control, and local fallback behavior;
- a classic Bluetooth SPP module as the wireless UART bridge;
- a Windows Python application using a virtual COM port and `pyserial`.

It is a reusable engineering skill and protocol specification. It is not a ready-made STM32 firmware image, mobile application, or complete greenhouse control platform.

## What It Helps Build

The skill covers the communication path and the decisions around it:

- sensor data from STM32 to a Windows application;
- control commands from the application back to relays, pumps, lights, fans, switches, GPIO, and PWM outputs;
- line-delimited `DATA`, `CMD`, `ACK`, `ERR`, and `HB` messages;
- sequence IDs and per-command status tracking;
- bounded UART buffers and partial-line handling;
- heartbeat, reconnect, timeout, and offline fallback behavior;
- validation on both the computer and the microcontroller.

## Architecture

The recommended first version keeps high-level data processing on the computer and keeps time-critical execution and local fallback rules on the STM32:

```text
sensors -> STM32 -> UART -> Bluetooth SPP -> Windows virtual COM port
Windows Python -> cleaning -> aggregation -> SQLite -> decision -> validator
Windows virtual COM port -> Bluetooth SPP -> STM32 -> relays/PWM/GPIO
```

Classic Bluetooth SPP is a practical starting point because Windows exposes the paired module as a COM port and Python can access it with `pyserial`. BLE/GATT should be designed separately when the product requires mobile connectivity or BLE-specific features.

## Install

Clone the skill into the Codex skills directory:

```powershell
git clone https://github.com/mo7416449-web/stm32-bluetooth-agri-control.git `
  "$env:USERPROFILE\.codex\skills\stm32-bluetooth-agri-control"
```

Restart Codex after installation so it can discover the skill.

## Use It

The skill is useful when designing a new system or adapting an existing Python agriculture pipeline. Example requests:

```text
Use stm32-bluetooth-agri-control to design an STM32 and HC-05 protocol
for soil-moisture readings and pump control from a Windows Python app.
```

```text
Review my STM32 UART receiver for partial lines, bounded buffers,
ACK/ERR responses, heartbeat timeout, and safe pump fallback.
```

```text
Add pyserial communication to my agriculture pipeline. Keep dry-run as
the default and record the status of every command separately.
```

For a project-specific design, provide the STM32 model, Bluetooth module, sensors, actuators, COM port, baud rate, and firmware toolchain.

## Protocol At A Glance

Each message occupies one physical line, uses semicolon-separated fields, and ends with `\n`:

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
HB;SEQ=3001
```

The protocol uses `SEQ` to connect commands with their acknowledgements or errors. Required fields, parsing rules, limits, and offline behavior are documented in [`references/protocol.md`](references/protocol.md).

## Reliability And Safety Behavior

The design keeps the control path explicit and bounded:

- the STM32 validates commands again instead of trusting the computer;
- malformed, unknown, oversized, or out-of-range messages are rejected;
- pump duration and light brightness are limited at the device boundary;
- low water level or a critical sensor fault forces the pump off;
- a missing computer heartbeat moves the controller into a conservative offline mode;
- `dry-run` remains the default while the serial path is being connected and tested.

The offline rules are intended to preserve safety and short-term continuity. They are not a replacement for a crop-management strategy or a complete autonomous farming controller.

## Repository Structure

```text
SKILL.md                  Main Codex skill instructions
agents/openai.yaml        Skill display metadata and default prompt
references/protocol.md    Bluetooth serial protocol reference
README.md                 English project overview
README.zh-CN.md           Chinese project overview
```

## Contributing

Changes to the message format, command limits, or fallback behavior should update both `SKILL.md` and `references/protocol.md`. Keep protocol examples, field names, and safety limits consistent across the repository.

Issues and pull requests are welcome for clearer protocol rules, additional sensor or actuator examples, and improvements to the Windows `pyserial` integration guidance.
