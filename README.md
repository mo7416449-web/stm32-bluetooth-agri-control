# STM32 蓝牙农业监测与控制

**STM32 Bluetooth Agricultural Monitoring and Control**

[简体中文](README.zh-CN.md)

面向 STM32 与 Windows 农业监测控制程序的安全蓝牙串口通信设计、实现与审查技能包。

This Codex skill designs, implements, and reviews safe Bluetooth serial communication between STM32 devices and Windows-based agricultural monitoring and control applications.

The skill favors a practical architecture: STM32 performs sensing, actuator control, and local safety fallback, while a Windows Python application handles data processing, storage, optional local-LLM decisions, validation, and command tracking.

## What this skill helps with

- STM32 UART communication through classic Bluetooth SPP modules
- Windows virtual COM port integration with Python and `pyserial`
- Line-delimited `DATA`, `CMD`, `ACK`, `ERR`, and `HB` messages
- Relay, pump, light, fan, switch, GPIO, and PWM control
- PC-side and STM32-side safety validation
- Heartbeat, reconnect, timeout, and offline fallback behavior
- Review of protocol parsing, bounded buffers, malformed input, and command traceability

## Recommended architecture

```text
sensors -> STM32 -> UART -> Bluetooth SPP module -> Windows virtual COM port
Windows Python -> cleaning -> aggregation -> SQLite -> mock/Ollama LLM -> validator
Windows virtual COM port -> Bluetooth SPP module -> STM32 -> relays/PWM/GPIO
```

Classic Bluetooth SPP is recommended for the first version because Windows exposes it as a COM port and Python can access it with `pyserial`. BLE/GATT is better treated as a separate design when mobile support or BLE-specific requirements are involved.

## Installation

Clone this repository into your Codex skills directory:

```powershell
git clone https://github.com/mo7416449-web/stm32-bluetooth-agri-control.git `
  "$env:USERPROFILE\.codex\skills\stm32-bluetooth-agri-control"
```

Restart Codex after installation so the skill can be discovered.

## Example requests

```text
Use stm32-bluetooth-agri-control to design an STM32 and HC-05 protocol
for soil-moisture readings and pump control from a Windows Python app.
```

```text
Review my STM32 UART receive code for partial-line handling, bounded
buffers, ACK/ERR responses, heartbeat timeout, and safe pump fallback.
```

```text
Add pyserial communication to my agriculture pipeline. Keep dry-run as
the default and record the status of every command separately.
```

Useful project details to provide include the STM32 model, Bluetooth module, sensors, actuators, COM port, baud rate, and firmware toolchain.

## Protocol at a glance

Messages use one semicolon-delimited line per frame and end with `\n`:

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
HB;SEQ=3001
```

See [references/protocol.md](references/protocol.md) for required fields, parser behavior, command limits, and offline behavior.

## Safety principles

- Validate every command on both Windows and STM32.
- Reject unknown devices, actions, malformed values, and oversized lines.
- Keep actuator limits consistent across both sides.
- Force the pump off when water level is low or a relevant sensor fails.
- Enter a conservative safe mode when the PC heartbeat times out.
- Keep real hardware control in dry-run mode until the communication path and safety rules have been tested.

The fallback controller is intended for safety and short-term continuity, not optimal autonomous crop management.

## Repository structure

```text
.
|-- SKILL.md                 Main skill instructions
|-- agents/
|   `-- openai.yaml          Skill display metadata and default prompt
`-- references/
    `-- protocol.md          Bluetooth serial protocol reference
```

## Contributing

Contributions are welcome. When changing protocol or safety behavior, update both `SKILL.md` and `references/protocol.md`, and keep examples consistent with the documented limits.

