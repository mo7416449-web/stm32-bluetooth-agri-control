# STM32 蓝牙农业控制

[English](README.md)

这是一个面向 Codex 的技能包，用于设计、实现和审查 STM32 与 Windows 农业监测控制程序之间的安全蓝牙串口通信。

推荐采用“PC 侧智能决策、STM32 侧可靠执行与安全兜底”的架构：STM32 负责传感器采集、执行器控制和离线安全规则；Windows Python 程序负责数据清洗、聚合、存储、可选的本地大模型决策、安全校验和命令状态跟踪。

## 适用场景

- STM32 通过经典蓝牙 SPP 模块进行 UART 通信
- Windows 虚拟 COM 口与 Python `pyserial` 集成
- 设计按行传输的 `DATA`、`CMD`、`ACK`、`ERR`、`HB` 协议
- 控制继电器、水泵、灯光、风扇、开关、GPIO 和 PWM
- 在 Windows 与 STM32 两侧执行双重安全校验
- 处理心跳、断线重连、通信超时和离线兜底
- 审查半包处理、缓冲区边界、异常输入和命令可追踪性

## 推荐架构

```text
传感器 -> STM32 -> UART -> 蓝牙 SPP 模块 -> Windows 虚拟 COM 口
Windows Python -> 清洗 -> 聚合 -> SQLite -> mock/Ollama LLM -> 安全校验器
Windows 虚拟 COM 口 -> 蓝牙 SPP 模块 -> STM32 -> 继电器/PWM/GPIO
```

第一版建议优先使用经典蓝牙 SPP，因为 Windows 会将其映射成 COM 口，Python 可以直接通过 `pyserial` 访问。如果项目明确需要手机连接或 BLE/GATT 特性，再单独设计 BLE 方案。

## 安装

将仓库克隆到 Codex 的 skills 目录：

```powershell
git clone https://github.com/mo7416449-web/stm32-bluetooth-agri-control.git `
  "$env:USERPROFILE\.codex\skills\stm32-bluetooth-agri-control"
```

安装完成后重启 Codex，使其重新发现该技能。

## 使用示例

```text
使用 stm32-bluetooth-agri-control，为 STM32、HC-05、土壤湿度传感器
和水泵设计一套与 Windows Python 程序通信的协议。
```

```text
审查我的 STM32 UART 接收代码，重点检查半包处理、缓冲区上限、
ACK/ERR 响应、心跳超时和水泵安全兜底。
```

```text
给现有农业数据管线加入 pyserial 通信，默认保持 dry-run，
并分别记录每条控制命令的执行状态。
```

为了获得更准确的设计或代码，请尽量提供 STM32 型号、蓝牙模块、传感器与执行器列表、COM 口、波特率和固件工具链。

## 协议速览

每条消息占一行，字段使用分号分隔，并以 `\n` 结尾：

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
HB;SEQ=3001
```

完整的必填字段、解析规则、命令限制和离线行为见 [references/protocol.md](references/protocol.md)。

## 安全原则

- Windows 与 STM32 必须分别校验每条命令。
- 拒绝未知设备、未知动作、非法数值和超长消息。
- 保证 PC 侧与 STM32 侧的执行器限制一致。
- 水位过低或关键传感器故障时强制关闭水泵。
- PC 心跳超时后进入保守的安全模式。
- 在通信链路和安全规则完成实机测试前，默认使用 `dry-run`。

离线兜底的目标是保障安全和短期连续运行，而不是在 STM32 上实现复杂的自主种植策略。

## 仓库结构

```text
.
|-- SKILL.md                 技能的主要工作说明
|-- agents/
|   `-- openai.yaml          展示信息与默认提示词
`-- references/
    `-- protocol.md          蓝牙串口协议参考
```

## 参与贡献

欢迎提交 Issue 或 Pull Request。修改协议或安全行为时，请同步更新 `SKILL.md` 与 `references/protocol.md`，并保证示例和限制保持一致。

