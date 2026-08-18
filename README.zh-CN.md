# STM32 蓝牙农业监测与控制

**面向农业监测与控制系统的可靠蓝牙串口通信 Codex 技能包。**

[English](README.md)

这个仓库提供一套可复用的设计指南和按行传输的串口协议参考，面向以下组合：

- STM32 负责传感器采集、执行器控制和本地兜底；
- 经典蓝牙 SPP 模块作为无线 UART 桥接；
- Windows Python 程序通过虚拟 COM 口和 `pyserial` 读取数据、生成控制命令。

它是一套可复用的工程技能和通信协议规范，不是现成的 STM32 固件、手机 App，也不是可以直接运行的完整温室控制平台。

## 它能帮助完成什么

这个技能覆盖农业控制系统中从通信到控制边界的主要设计：

- STM32 向 Windows 程序发送传感器数据；
- Windows 程序向继电器、水泵、灯光、风扇、开关、GPIO 和 PWM 输出发送控制命令；
- 设计按行传输的 `DATA`、`CMD`、`ACK`、`ERR` 和 `HB` 消息；
- 使用序列号和逐条状态记录跟踪控制命令；
- 处理 UART 半包、超长消息和有限缓冲区；
- 处理心跳、断线重连、通信超时和离线兜底；
- 在 Windows 和 STM32 两端分别验证控制命令。

## 推荐架构

第一版建议让计算密集型的数据处理放在电脑端，让 STM32 负责实时执行和本地兜底：

```text
传感器 -> STM32 -> UART -> 蓝牙 SPP -> Windows 虚拟 COM 口
Windows Python -> 清洗 -> 聚合 -> SQLite -> 决策 -> 校验器
Windows 虚拟 COM 口 -> 蓝牙 SPP -> STM32 -> 继电器/PWM/GPIO
```

第一版优先使用经典蓝牙 SPP，是因为 Windows 会把已配对模块映射为 COM 口，Python 可以直接通过 `pyserial` 访问。如果产品需要手机连接或 BLE/GATT 特性，应单独设计 BLE 方案。

## 安装

将技能克隆到 Codex 的 skills 目录：

```powershell
git clone https://github.com/mo7416449-web/stm32-bluetooth-agri-control.git `
  "$env:USERPROFILE\.codex\skills\stm32-bluetooth-agri-control"
```

安装完成后重启 Codex，使其发现该技能。

## 使用方式

它可以用于设计新的农业控制系统，也可以用于改造已有的 Python 数据管线。例如：

```text
使用 stm32-bluetooth-agri-control，为 STM32、HC-05、土壤湿度传感器
和水泵设计一套与 Windows Python 程序通信的协议。
```

```text
检查我的 STM32 UART 接收程序，重点处理半包、缓冲区上限、
ACK/ERR 响应、心跳超时和水泵离线兜底。
```

```text
给现有农业数据管线加入 pyserial 通信，默认使用 dry-run，
并分别记录每条控制命令的执行状态。
```

如果需要针对具体项目设计，最好同时提供 STM32 型号、蓝牙模块、传感器、执行器、COM 口、波特率和固件工具链。

## 协议速览

每条消息占一行，字段使用分号分隔，并以 `\n` 结尾：

```text
DATA;SEQ=1001;SID=S001;AREA=A;METRIC=soil_moisture;VALUE=25.4;UNIT=PCT
CMD;SEQ=2001;DEV=PUMP1;ACTION=ON;DURATION=30
ACK;SEQ=2001;DEV=PUMP1;STATUS=OK
ERR;SEQ=2001;DEV=PUMP1;REASON=INVALID_DURATION
HB;SEQ=3001
```

协议使用 `SEQ` 将控制命令与对应的确认或错误关联起来。必填字段、解析规则、命令限制和离线行为见 [`references/protocol.md`](references/protocol.md)。

## 可靠性与安全行为

这套设计把控制路径保持在明确、有限和可回退的范围内：

- STM32 会再次校验命令，而不是完全信任电脑端；
- 拒绝格式错误、未知、超长或超出范围的消息；
- 在设备边界限制水泵时长和灯光亮度；
- 水位过低或关键传感器故障时强制关闭水泵；
- 电脑心跳丢失后进入保守的离线模式；
- 在串口链路接通并完成测试前，默认使用 `dry-run`。

离线规则用于保障安全和短期连续运行，不等同于完整的作物管理策略或自主农业控制器。

## 仓库结构

```text
SKILL.md                  Codex 技能的主要工作说明
agents/openai.yaml        技能展示信息和默认提示词
references/protocol.md    蓝牙串口协议参考
README.md                 English 项目说明
README.zh-CN.md           中文项目说明
```

## 参与贡献

如果修改消息格式、控制限制或离线行为，请同步更新 `SKILL.md` 和 `references/protocol.md`，并保持仓库中的字段名、示例和安全限制一致。

欢迎提交 Issue 或 Pull Request，补充协议规则、传感器和执行器示例，以及 Windows `pyserial` 集成方面的改进。
