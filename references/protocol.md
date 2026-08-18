# 蓝牙农业控制串口协议参考

Bluetooth Serial Protocol Reference

Use this reference when designing or reviewing STM32-to-Windows Bluetooth SPP communication for the agricultural control project.

## Transport

- Use classic Bluetooth SPP for the MVP.
- Windows pairs the Bluetooth module and exposes a virtual COM port.
- Python uses `pyserial`.
- STM32 treats the Bluetooth module as a UART peripheral.
- Use `9600` or `115200` baud consistently on both sides.
- Encode messages as UTF-8 or ASCII-compatible bytes.
- Terminate each message with `\n`.

## Frame Format

Each line is a semicolon-delimited key-value message:

```text
TYPE;KEY=VALUE;KEY=VALUE
```

The first token is the message type. All later tokens should be `KEY=VALUE`.

Required parser behavior:

- Trim `\r\n`.
- Reject empty lines.
- Reject lines longer than the configured maximum.
- Reject tokens without `=` except for the first message type token.
- Preserve unknown keys for logging but ignore them during execution.
- Never crash on malformed input.

## STM32 to Windows

Sensor data:

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
DATA;SEQ=1002;SID=S002;AREA=A;METRIC=temperature;VALUE=31.2;UNIT=C
DATA;SEQ=1003;SID=S003;AREA=A;METRIC=light;VALUE=6500;UNIT=LUX
```

Heartbeat:

```text
HB;SEQ=3001
```

Error report:

```text
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
```

## Windows to STM32

Commands:

```text
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
CMD;SEQ=2002;DEV=LIGHT1;ACTION=BRIGHTNESS;VALUE=70
CMD;SEQ=2003;DEV=SWITCH1;ACTION=OFF
```

Heartbeat or ping:

```text
HB;SEQ=3002
```

## STM32 Acknowledgements

Successful command:

```text
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
```

Rejected command:

```text
ERR;SEQ=2001;DEV=PUMP1;REASON=UNKNOWN_DEVICE
ERR;SEQ=2001;DEV=PUMP1;REASON=UNKNOWN_ACTION
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
ERR;SEQ=2002;DEV=LIGHT1;REASON=INVALID_VALUE
```

## Required Fields

`DATA`:

- `SEQ`
- `SID`
- `AREA`
- `METRIC`
- `VALUE`
- `UNIT`

`CMD`:

- `SEQ`
- `DEV`
- `ACTION`

`ACK`:

- `SEQ`
- `STATUS`

`ERR`:

- `SEQ`
- `REASON`

## Safety Limits

Default command limits:

```text
PUMP*.ON duration: 1..30 seconds on STM32 fallback, 1..60 seconds when PC validated
LIGHT*.BRIGHTNESS value: 0..100
SWITCH*.ON/OFF only
FAN*.ON/OFF only if configured
```

Reject or clamp values consistently. For real hardware, prefer rejecting invalid commands and logging `ERR`.

## Offline Behavior

If no valid heartbeat or command is received from the PC within the configured timeout:

```text
enter offline mode
stop non-essential actuators if unsafe
run only conservative fallback rules
keep sending DATA and HB when possible
exit offline mode after valid PC heartbeat or command
```
