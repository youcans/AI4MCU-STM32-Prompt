# 软件功能说明书

---

## 1. 文档说明

本文档为"基于 ADC 同步采样的 PWM 占空比控制"项目的软件功能说明书，面向软件设计和开发人员，综合描述项目的功能目标、软件输入输出对象、软件功能需求、MCU 资源分配、引脚配置以及设计约束与待确认事项。

本文档不包含需求分析过程、技术资料核实记录和审阅历史。如果不同来源文件之间存在冲突，在相关章节中标注说明。

**版本记录：**

| 修订版本 | 说明 |
|---|---|
| 初始版本 | 基于 ADC 同步采样的 PWM 占空比控制基本功能（PWM 输出、ADC 同步采样、实时数据处理、DAC 方波、芯片温度监测、温度异常处理、系统状态切换、LED 指示、串口数据发送） |
| 本修订版 | 在现有系统基础上新增 OLED 本地显示功能（SSD1306 / 128×64 / I2C 接口 / 3.3V 供电），用于本地显示系统状态、芯片温度与主要运行数据；其余功能需求保持不变。本次仅新增显示需求，不确定具体 I2C 外设、MCU 引脚、刷新周期、软件模块及代码实现方式 |
| 硬件配置补充 | 确定新增 OLED 使用 I2C1，PB8 配置为 I2C1_SCL，PB9 配置为 I2C1_SDA，并补充外接 OLED 模块与 NUCLEO-G431RB 的连接关系 |
| 显示内容确定 | 确定 OLED 在停止、运行、异常三种状态下均显示系统状态、芯片温度和 DAC 幅值（取自 `PROC_GetDacAmplitude()` 保存数据）；停止/异常状态下 DAC 处于关闭状态，显示的是保存的 DAC 幅值数据，不代表 DAC 实际输出 |

---

## 2. 项目目标与功能概述

### 2.1 项目目标

在 NUCLEO-G431RB 开发板上，以 PWM 周期事件触发三通道模拟电压同步采样，在每次采样序列完成后执行一次实时数据处理，根据动态模拟电压的采样结果更新 PWM 占空比和 DAC 方波幅值，同时实现芯片温度监测、系统状态切换、LED 指示和串口数据发送功能。

### 2.2 功能概述

| 编号 | 功能名称 | 概要 |
|---|---|---|
| FR-01 | PWM 输出与周期事件 | 输出三组 10 kHz 带死区互补 PWM，第一组占空比动态更新，第二/三组使用预设值；产生 ADC 采样触发事件 |
| FR-02 | ADC 三通道同步采样 | PWM 周期事件触发三通道模拟电压采样，一路动态电压、两路参考电平 |
| FR-03 | 实时数据处理 | 每次采样序列完成后执行，读取采样值、计算并更新 PWM 占空比和 DAC 方波幅值 |
| FR-04 | DAC 参考方波输出 | 输出 250 Hz 参考方波，幅值随动态模拟电压采样结果更新 |
| FR-05 | 芯片温度监测 | 使用内部温度传感器以 500 Hz 采集芯片温度，全状态持续运行 |
| FR-06 | 温度异常处理 | 温度超限进入异常状态，温度恢复返回停止状态 |
| FR-07 | 系统状态切换 | 停止/运行/异常三状态，按键切换停止↔运行，温度触发异常进入与恢复 |
| FR-08 | LED 状态指示 | 停止熄灭、运行翻转（~常亮）、异常 10 Hz 闪烁 |
| FR-09 | 串口数据发送 | JustFloat / 115200 bps / 500 Hz，运行发送 DAC 值，停止发送温度，异常不发送 |
| FR-10 | OLED 本地显示 | 通过 I2C 接口驱动 0.96 英寸 OLED（SSD1306 / 128×64）本地显示系统状态、芯片温度和主要运行数据；显示内容随系统状态和相关数据变化更新。新增显示功能不得影响原有 ADC 采样、PWM、DAC、串口通信、状态切换和温度异常处理功能 |

---

## 3. 软件输入输出对象

### 3.1 输入对象

| 类型 | 对象名称 | 对象标识 | 来源 | 说明 |
|---|---|---|---|---|
| 输入事件 | PWM 周期事件 | `PwmPeriodEvent` | TIM1 | 10 kHz 周期性事件，在运行状态下触发 ADC 三通道同步采样 |
| 输入事件 | ADC 采样序列完成事件 | `AdcSequenceCompleteEvent` | ADC1 | 三通道采样序列转换结束后产生，10 kHz，仅运行状态下有效。触发实时数据处理（FR-03）执行 |
| 输入事件 | 用户按键事件 | `UserButtonEvent` | PC13 / EXTI13 | 按键按下时产生，用于在停止状态和运行状态之间切换。异常状态下不响应 |
| 输入事件 | 500 Hz 周期事件 | `Timer500HzEvent` | TIM6 | 500 Hz 周期性事件，触发芯片温度采样、DAC 翻转和串口数据发送，在停止、运行和异常三种状态下均持续产生 |
| 输入数据 | 动态模拟电压采样值 | `AdcDynamicVoltage` | ADC1_IN2 | 12 位原始采样值，由电位器提供的可调模拟电压转换而来，用于计算 PWM 占空比和 DAC 输出值 |
| 输入数据 | 高电平参考信号采样值 | `AdcHighRef` | ADC1_IN12 | 12 位原始采样值，由固定的高电平参考信号转换而来，当前不参与动态计算 |
| 输入数据 | 低电平参考信号采样值 | `AdcLowRef` | ADC1_IN15 | 12 位原始采样值，由固定的低电平参考信号转换而来，当前不参与动态计算 |
| 输入数据 | 芯片温度采样值 | `ChipTemperature` | ADC1_IN16（内部温度传感器） | 500 Hz，在停止、运行和异常三种状态下均持续提供。用于温度异常判断和串口发送 |
| 输入参数 | 温度异常阈值 | `TemperatureFaultThreshold` | 开发者配置 | 芯片温度超过此阈值时触发进入异常状态。具体数值在详细设计中确定 |
| 输入参数 | 温度恢复阈值 | `TemperatureRecoveryThreshold` | 开发者配置 | 芯片温度低于此阈值时触发退出异常状态并返回停止状态。具体数值在详细设计中确定 |

### 3.2 输出对象

| 类型 | 对象名称 | 对象标识 | 去向 | 说明 |
|---|---|---|---|---|
| 输出控制 | 第一组 PWM 占空比 | `PwmDutyCh1` | TIM1 | 由实时数据处理根据 `AdcDynamicVoltage` 计算并更新，运行状态下持续更新，停止/异常状态下不更新 |
| 输出控制 | 第二组 PWM 占空比 | `PwmDutyCh2` | TIM1 | 预设固定值，运行状态下输出，停止/异常状态下不输出 |
| 输出控制 | 第三组 PWM 占空比 | `PwmDutyCh3` | TIM1 | 预设固定值，运行状态下输出，停止/异常状态下不输出 |
| 输出控制 | PWM 输出启停 | `PwmOutputEnable` | TIM1 | 运行状态开启三组 PWM 输出，停止和异常状态关闭全部 PWM 输出 |
| 输出控制 | DAC 输出值 | `DacOutputValue` | DAC1 | 由实时数据处理根据 `AdcDynamicVoltage` 计算并更新，运行状态下持续更新，停止/异常状态下不更新 |
| 输出控制 | DAC 输出启停 | `DacOutputEnable` | DAC1 | 运行状态开启 DAC 输出，停止和异常状态关闭 DAC 输出 |
| 输出控制 | LED 输出 | `LedOutput` | PA5（板载用户 LED） | 停止状态输出熄灭，运行状态每采样序列翻转一次，异常状态以 10 Hz 闪烁 |
| 输出控制 | 模拟电压采样启停 | `AdcSamplingEnable` | ADC1 | 运行状态启动三通道模拟电压采样，停止和异常状态停止采样 |
| 输出数据 | 串口发送数据 | `SerialTxData` | LPUART1（至 VOFA+） | JustFloat 格式，115200 bit/s，500 Hz。运行状态发送 `DacOutputValue`，停止状态发送 `ChipTemperature`，异常状态不发送 |
| 输出数据 | 系统运行状态 | `SystemState` | 各功能模块 | 停止、运行或异常，由 `UserButtonEvent`、`TemperatureFaultEnter` 和 `TemperatureFaultExit` 共同决定，控制各项功能的启停和数据流向切换 |
| 输出数据 | OLED 显示输出 | `OledDisplayContent` | OLED（SSD1306 / I2C） | 本地显示当前系统状态、芯片温度和主要运行数据。显示内容随 `SystemState` 及相关运行数据变化更新，具体显示字段与刷新方式在详细设计中确定 |
| 输出事件 | 温度异常进入触发 | `TemperatureFaultEnter` | SF-09 | 芯片温度达到 `TemperatureFaultThreshold` 时产生，驱动系统进入异常状态 |
| 输出事件 | 温度异常退出触发 | `TemperatureFaultExit` | SF-09 | 芯片温度降至 `TemperatureRecoveryThreshold` 时产生，驱动系统从异常状态返回停止状态 |

> `OledDisplayContent` 所用的具体运行数据字段以当前工程实际可用数据为准；其刷新周期、软件模块及代码实现方式在详细设计中确定。OLED 硬件连接（I2C1 / PB8-SCL / PB9-SDA / 默认地址 0x3C）已确定，见 [§5.1.1](#511-oled-i2c引脚配置) 与 [§5.2.1](#521-外接-oled-模块连接)。

---

## 4. 软件功能需求

### 4.1 功能列表

| 功能编号 | 功能名称 | 输入 | 输出 | 触发条件 | 运行效果 |
|---|---|---|---|---|---|
| SF-01 | 系统初始化 | — | `SystemState`（停止）、`PwmOutputEnable`（关闭）、`AdcSamplingEnable`（停止）、`DacOutputEnable`（关闭）、`LedOutput`（熄灭）、`PwmDutyCh2`（预设值）、`PwmDutyCh3`（预设值） | 系统上电或复位 | 系统进入停止状态，各功能模块处于初始状态 |
| SF-02 | PWM 输出控制 | `SystemState`、`PwmDutyCh2`（预设值）、`PwmDutyCh3`（预设值） | `PwmOutputEnable` | `SystemState` 进入或离开运行状态 | 运行状态下开启三组 PWM 输出（10 kHz，死区互补）；停止和异常状态下关闭全部 PWM 输出 |
| SF-03 | ADC 采样控制 | `SystemState` | `AdcSamplingEnable` | `SystemState` 进入或离开运行状态 | 运行状态下启动三通道模拟电压采样（由 `PwmPeriodEvent` 触发，按预定顺序转换）；停止和异常状态下停止采样 |
| SF-04 | ADC 采样结果获取 | `AdcSequenceCompleteEvent` | `AdcDynamicVoltage`、`AdcHighRef`、`AdcLowRef` | `AdcSequenceCompleteEvent`（仅运行状态） | 每次三通道采样序列完成后读取三路注入组转换结果 |
| SF-05 | 实时数据处理 | `AdcDynamicVoltage`（来自 SF-04） | `PwmDutyCh1`、`DacOutputValue` | `AdcSequenceCompleteEvent`（仅运行状态） | 根据 `AdcDynamicVoltage` 计算并更新第一组 PWM 占空比和 DAC 方波幅值。模拟电压采样、数据处理、PWM 更新和 DAC 更新之间保持 10 kHz 稳定周期关系 |
| SF-06 | DAC 输出控制 | `SystemState`、`DacOutputValue`（来自 SF-05）、`Timer500HzEvent` | `DacOutputEnable` | `SystemState` 进入或离开运行状态，以及运行状态下每次 `Timer500HzEvent` | 运行状态下由 500 Hz 周期事件触发 DAC 在 0 与最新 `DacOutputValue` 之间翻转，形成 250 Hz 方波；停止和异常状态下关闭 DAC 输出 |
| SF-07 | 芯片温度采样 | — | `ChipTemperature` | `Timer500HzEvent`（全状态） | 采集 MCU 内部温度传感器数据并换算为温度值。在停止、运行和异常三种状态下均持续执行 |
| SF-08 | 温度异常判断 | `ChipTemperature`、`TemperatureFaultThreshold`、`TemperatureRecoveryThreshold` | `TemperatureFaultEnter`、`TemperatureFaultExit` | 每次获取 `ChipTemperature` 后 | 将当前温度与 `TemperatureFaultThreshold` 和 `TemperatureRecoveryThreshold` 比较：达异常阈值时产生 `TemperatureFaultEnter`，达恢复阈值时产生 `TemperatureFaultExit`。两阈值为独立值以避免临界抖动 |
| SF-09 | 系统状态切换 | `UserButtonEvent`、`TemperatureFaultEnter`、`TemperatureFaultExit` | `SystemState` | `UserButtonEvent`、`TemperatureFaultEnter` 或 `TemperatureFaultExit` | 停止状态与运行状态之间由用户按键切换；芯片温度达异常阈值时从当前状态自动进入异常状态；温度达恢复阈值时自动从异常状态返回停止状态。异常状态下按键不产生切换；返回停止后需再次按键才能进入运行 |
| SF-10 | LED 状态指示 | `SystemState`、`AdcSequenceCompleteEvent` | `LedOutput` | `SystemState` 变化，或运行状态下每次 `AdcSequenceCompleteEvent`，或异常状态下 10 Hz 周期 | 停止状态：LED 熄灭；运行状态：每次采样序列完成后翻转，肉眼观察近似常亮；异常状态：以 10 Hz 频率闪烁（由 `Timer500HzEvent` 分频） |
| SF-11 | 串口数据发送 | `SystemState`、`DacOutputValue`、`ChipTemperature` | `SerialTxData` | `Timer500HzEvent`（全状态） | 以 JustFloat 格式、115200 bit/s、500 Hz 向上位机 VOFA+ 发送数据。运行状态发送 `DacOutputValue`，停止状态发送 `ChipTemperature`，异常状态不发送 |
| SF-12 | OLED 本地显示 | `SystemState`、`ChipTemperature`、主要运行数据（具体数据项待确定） | `OledDisplayContent` | 显示内容随系统状态和相关运行数据更新，具体刷新方式和刷新周期在后续设计中确定 | 通过 I2C 接口驱动 0.96 英寸 OLED（SSD1306 / 128×64）进行本地显示，显示内容随 `SystemState` 与相关运行数据更新。在停止、运行、异常三种状态下均有对应显示内容（见 4.2）。新增显示功能不得影响原有 ADC 采样、PWM、DAC、串口通信、状态切换和温度异常处理功能 |

### 4.2 系统状态与功能行为

系统设置停止、运行和异常三种状态。各状态下每项软件功能的执行情况如下：

| 功能 | 停止 | 运行 | 异常 |
|---|---|---|---|
| SF-01 系统初始化 | —（上电后执行一次） | — | — |
| SF-02 PWM 输出控制 | 关闭 | 开启（10 kHz） | 关闭 |
| SF-03 ADC 采样控制 | 停止 | 启动（10 kHz） | 停止 |
| SF-04 ADC 采样结果获取 | 不执行 | 执行（10 kHz） | 不执行 |
| SF-05 实时数据处理 | 不执行 | 执行（10 kHz） | 不执行 |
| SF-06 DAC 输出控制 | 关闭 | 开启（250 Hz 方波） | 关闭 |
| SF-07 芯片温度采样 | 执行（500 Hz） | 执行（500 Hz） | 执行（500 Hz） |
| SF-08 温度异常判断 | 执行 | 执行 | 执行（判断恢复） |
| SF-09 系统状态切换 | 响应按键 | 响应按键 | 响应温度恢复 |
| SF-10 LED 状态指示 | 熄灭 | 翻转（~常亮） | 10 Hz 闪烁 |
| SF-11 串口数据发送 | 执行（500 Hz，温度） | 执行（500 Hz，DAC 值） | 不执行 |
| SF-12 OLED 本地显示 | 显示（状态、芯片温度、DAC 幅值） | 显示（状态、芯片温度、DAC 幅值） | 显示（状态、芯片温度、DAC 幅值） |

状态切换规则：
- 用户按下板载用户按键（B1），在停止状态和运行状态之间切换；
- 芯片温度达到异常阈值时，从当前状态自动进入异常状态；
- 芯片温度下降到恢复阈值时，从异常状态自动返回停止状态；
- 异常状态下按键不产生状态切换；
- 返回停止状态后，需用户再次按下按键才能进入运行状态。

OLED 显示补充说明：
- OLED 显示在停止、运行、异常三种状态下均持续工作，显示内容随 `SystemState` 与相关运行数据更新；
- 停止、运行、异常三种状态下均显示系统状态、芯片温度和 DAC 幅值（取自 `PROC_GetDacAmplitude()` 保存的 DAC 幅值数据）；
- 注：停止和异常状态下 DAC 处于关闭状态（无实际输出），此时 OLED 显示的是保存的 DAC 幅值数据，不表示 DAC 正在实际输出；
- 显示内容和刷新方式不得影响原有各项功能；具体显示字段、刷新周期、软件模块及代码实现方式在详细设计中确定。OLED 硬件配置（I2C1 / PB8-SCL / PB9-SDA / 默认地址 0x3C）已确定，见 [§5.1.1](#511-oled-i2c引脚配置) 与 [§5.2.1](#521-外接-oled-模块连接)。

---

## 5. MCU 引脚配置

### 5.1 引脚配置表

| 引脚 | 功能 | 模式 | 关联资源 | 备注 |
|---|---|---|---|---|
| PA8 | TIM1_CH1 | AF 推挽 | TIM1 | PWM 第一组主通道，10 kHz 中心对齐 |
| PB13 | TIM1_CH1N | AF 推挽 | TIM1 | PWM 第一组互补通道，与 PA8 带死区互补输出 |
| PA9 | TIM1_CH2 | AF 推挽 | TIM1 | PWM 第二组主通道，预设占空比 |
| PB14 | TIM1_CH2N | AF 推挽 | TIM1 | PWM 第二组互补通道，与 PA9 带死区互补输出 |
| PA10 | TIM1_CH3 | AF 推挽 | TIM1 | PWM 第三组主通道，预设占空比 |
| PB15 | TIM1_CH3N | AF 推挽 | TIM1 | PWM 第三组互补通道，与 PA10 带死区互补输出 |
| PA1 | ADC1_IN2 | 模拟输入 | ADC1 | 注入组 Rank1，连接 10 kΩ 电位器滑动端（电位器高端接 3.3 V，低端接 GND），驱动 PWM 占空比和 DAC 输出 |
| PB1 | ADC1_IN12 | 模拟输入 | ADC1 | 注入组 Rank2，杜邦线接高电平参考电压，当前不参与动态计算 |
| PB0 | ADC1_IN15 | 模拟输入 | ADC1 | 注入组 Rank3，杜邦线接 GND，当前不参与动态计算 |
| PA4 | DAC1_OUT1 | 模拟输出 | DAC1 | 运行状态输出 250 Hz 方波，仅接高阻抗测量设备 |
| PA2 | LPUART1_TX | AF 推挽 | LPUART1 | 串口数据发送，通过板载 SB17 连接 STLINK-V3E 虚拟串口 |
| PA3 | LPUART1_RX | AF 输入 | LPUART1 | 串口数据接收，通过板载 SB23 连接 STLINK-V3E 虚拟串口，本项目不使用 |
| PC13 | GPIO_EXTI13 | 外部中断 | EXTI | 板载用户按键 B1（外部下拉，按下接通高电平），上升沿触发 |
| PA5 | GPIO_Output | 推挽输出 | GPIO | 板载用户 LED LD2（高电平点亮），初始低电平（熄灭） |
| PA13 | SYS_SWDIO | SWD | SWD | 调试接口数据线，板载 STLINK-V3E，**不得配置为应用 GPIO** |
| PA14 | SYS_SWCLK | SWD | SWD | 调试接口时钟线，板载 STLINK-V3E，**不得配置为应用 GPIO** |
| PB8 | I2C1_SCL | AF4 复用开漏 | I2C1 | **新增**：OLED 显示模块 I2C 时钟线，连接开发板 CN5-10 |
| PB9 | I2C1_SDA | AF4 复用开漏 | I2C1 | **新增**：OLED 显示模块 I2C 数据线，连接开发板 CN5-9 |

### 5.1.1 OLED（I2C）引脚配置

OLED 使用 I2C1 通信。PB8 配置为 I2C1_SCL，连接 CN5-10；PB9 配置为 I2C1_SDA，连接 CN5-9。PB8 和 PB9 均采用 AF4 复用开漏模式，未被当前工程其他功能占用，与现有 ADC、PWM、DAC、LPUART、按键、LED、SWD 和晶振引脚不存在冲突。MCU 端不启用内部上拉，使用 OLED 模块已有的 SCL、SDA 上拉电阻。

### 5.2 板载电路连接

| 引脚 | 板载连接 | 说明 |
|---|---|---|
| PA2 / PA3 | STLINK-V3E VCP，通过 SB17（PA2）、SB23（PA3）使能 | 开发板默认配置，无需额外操作 |
| PC13 | 用户按键 B1（外部下拉） | 外部中断输入，与板载按键电路匹配 |
| PA5 | 用户 LED LD2 | GPIO 推挽输出直接驱动 |
| PA13 / PA14 | STLINK-V3E SWD | 保留为调试接口 |
| PB0 / PB1 | 无板载有源电路占用 | 可用作 ADC 输入 |

> OLED 模块为外接显示模块（3.3V 供电、I2C 接口），不属于 NUCLEO-G431RB 板载电路；其供电、I2C 信号路径与器件地址信息见 §5.2.1，本表（板载电路）不新增条目。

### 5.2.1 外接 OLED 模块连接

OLED 显示模块（0.96" SSD1306 / 128×64 / I2C，3.3V 供电）为外接模块，与 NUCLEO-G431RB 开发板的连接关系如下：

- VCC 连接 CN6-4，使用 3.3V 供电；
- GND 连接 CN6-6；
- SCL 连接 CN5-10，对应 MCU 的 PB8，配置为 I2C1_SCL（AF4 复用开漏）；
- SDA 连接 CN5-9，对应 MCU 的 PB9，配置为 I2C1_SDA（AF4 复用开漏）。

OLED 通过 I2C1 与 MCU 通信，默认 7 位 I2C 地址为 0x3C。模块 SCL 和 SDA 已配置上拉电阻，MCU 端不启用内部上拉。引脚与连接信息见 [§5.1.1](#511-oled-i2c引脚配置)。

---
