# 软件功能说明书

> 编制日期：2026-08-04
> 依据文件：
> - `docs/analysis/design_inputs.md`（项目设计输入）
> - `docs/design/io_objects.md`（软件输入输出对象）
> - `docs/design/requirements_list.md`（软件功能需求列表）
> - `docs/design/mcu_resource_plan.md`（MCU 资源规划）
> - `docs/design/pin_config_spec.md`（MCU 引脚配置规范）
> 说明：本文档整合上述文件中的已确认内容，不展开软件模块、模块接口、数据流、状态机、函数、变量和代码实现。

---

## 1. 文档说明

本文档为"基于 ADC 同步采样的 PWM 占空比控制"项目的软件功能说明书，面向软件设计和开发人员，综合描述项目的功能目标、软件输入输出对象、软件功能需求、MCU 资源分配、引脚配置以及设计约束与待确认事项。

本文档不包含需求分析过程、技术资料核实记录和审阅历史。如果不同来源文件之间存在冲突，在相关章节中标注说明。

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
| 输出事件 | 温度异常进入触发 | `TemperatureFaultEnter` | SF-09 | 芯片温度达到 `TemperatureFaultThreshold` 时产生，驱动系统进入异常状态 |
| 输出事件 | 温度异常退出触发 | `TemperatureFaultExit` | SF-09 | 芯片温度降至 `TemperatureRecoveryThreshold` 时产生，驱动系统从异常状态返回停止状态 |

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

状态切换规则：
- 用户按下板载用户按键（B1），在停止状态和运行状态之间切换；
- 芯片温度达到异常阈值时，从当前状态自动进入异常状态；
- 芯片温度下降到恢复阈值时，从异常状态自动返回停止状态；
- 异常状态下按键不产生状态切换；
- 返回停止状态后，需用户再次按下按键才能进入运行状态。

---

## 5. MCU 资源规划

### 5.1 资源映射

| 功能 | MCU 资源 | 工作模式或通道 | 关键参数 | 关联关系 | 说明 |
|---|---|---|---|---|---|
| SF-02 PWM 输出控制 | TIM1 | 高级定时器，中心对齐 PWM 模式；CH1（PA8）/ CH1N（PB13）、CH2（PA9）/ CH2N（PB14）、CH3（PA10）/ CH3N（PB15） | 频率 10 kHz；死区插入使能（时间待确认） | TIM1 内部触发事件 → ADC1 注入组触发 | 运行状态启动 TIM1 和 PWM 输出；停止/异常状态停止 TIM1 及 ADC1 注入组采样。TIM1 触发事件的具体来源待确认 |
| SF-03 ADC 采样控制 | ADC1 | 注入组，由 TIM1 触发事件触发；Rank1=IN2（PA1）、Rank2=IN12（PB1）、Rank3=IN15（PB0） | 12 位分辨率；采样时间待确认 | TIM1 触发 → ADC1 注入组按 Rank1→2→3 顺序转换 | 三通道由同一次 TIM1 触发后顺序转换。停止/异常状态下注入组随 TIM1 停止而停止 |
| SF-04 ADC 采样结果获取 | ADC1 | 注入组 JEOS 标志（在 ADC1_2 全局中断中处理） | — | JEOS 有效 → 读取三路注入组转换结果 | 注入组不使用 DMA。根据 JEOS 和 EOC 标志区分两类 ADC 转换完成事件 |
| SF-05 实时数据处理 | （无独立外设） | 在 ADC1_2 全局中断中 JEOS 分支执行 | 执行时间须远小于 100 µs | 输入 SF-04 的采样值；输出 `PwmDutyCh1` 和 `DacOutputValue` | 仅运行状态下由 JEOS 触发 |
| SF-06 DAC 输出控制 | DAC1 | Channel 1（PA4 / DAC1_OUT1），外部缓冲通道，12 位右对齐 | 输出范围 0～VREF+ | `DacOutputValue` 由 SF-05 更新；`Timer500HzEvent` 触发写入 0 或最新值；`DacOutputEnable` 由 `SystemState` 控制 | 运行状态由 500 Hz 翻转形成 250 Hz 方波；停止/异常关闭 |
| SF-07 芯片温度采样 | ADC1 | 规则组，单通道 CH16（内部温度传感器 ADC1_IN16），由 TIM6_TRGO 触发 | 12 位分辨率；500 Hz | TIM6_TRGO → ADC1 规则组触发；ADC1_2 全局中断中 EOC 分支处理 | 温度传感器内部连接 ADC1_IN16（DS12589 §3.18.1） |
| SF-08 温度异常判断 | （无独立外设） | 在 ADC1_2 全局中断中 EOC 分支执行 | 阈值待确认 | 输入 `ChipTemperature` 和两个阈值参数；输出 `TemperatureFaultEnter` 或 `TemperatureFaultExit` | 软件比较逻辑 |
| SF-09 系统状态切换 | EXTI13 | PC13，外部中断线 13，上升沿触发；板载外部下拉电阻 | 消抖时间待确认 | `UserButtonEvent` → 停止↔运行；`TemperatureFaultEnter`/`TemperatureFaultExit` → 异常进入/退出 | — |
| SF-10 LED 状态指示 | GPIO 输出 | PA5，推挽输出 | — | 运行状态由 JEOS 触发翻转；异常状态由 TIM6 500 Hz 分频至 10 Hz | PA5 连接板载用户 LED LD2 |
| SF-11 串口数据发送 | LPUART1 + DMA1 | LPUART1_TX（PA2），异步模式；DMA1 通道由 DMAMux 分配 | 115200 bit/s；每帧 8 字节 | `Timer500HzEvent` 触发组织 JustFloat 帧并启动 DMA 发送 | PA2 通过板载 SB17 连接 STLINK-V3E 虚拟串口。LPUART1_RX（PA3，通过 SB23 连接）已配置但不使用 |
| 500 Hz 时基 | TIM6 | 基本定时器，向上计数模式；TRGO 输出更新事件 | 500 Hz（周期 2 ms） | TIM6_TRGO → ADC1 规则组触发（SF-07）；TIM6 更新事件 → `Timer500HzEvent`，同时用于 SF-06（DAC 翻转）、SF-10（异常闪烁分频）、SF-11（串口发送节拍） | TIM6 承担四项职责：温度采样触发、DAC 方波翻转、LED 闪烁分频、串口发送节拍 |

### 5.2 中断

ADC1 注入组和规则组共用 **ADC1_2 全局中断**，通过事件标志区分 JEOS 和 EOC 事件来源。其余使用的中断入口（TIM6、EXTI13、DMA1）之间未发现冲突。DMA 具体通道及其中断入口在配置时确认。

### 5.3 冲突检查

- **引脚复用**：16 个外部引脚功能分配不重叠，无复用冲突。
- **ADC1 共享**：注入组（10 kHz）和规则组（500 Hz）共享 ADC1，注入组可中断规则组转换，受 STM32G4 硬件支持。
- **DMA**：仅 LPUART1_TX 使用 DMA，无冲突。

---

## 6. MCU 引脚配置

### 6.1 引脚配置表

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

### 6.2 板载电路连接

| 引脚 | 板载连接 | 说明 |
|---|---|---|
| PA2 / PA3 | STLINK-V3E VCP，通过 SB17（PA2）、SB23（PA3）使能 | 开发板默认配置，无需额外操作 |
| PC13 | 用户按键 B1（外部下拉） | 外部中断输入，与板载按键电路匹配 |
| PA5 | 用户 LED LD2 | GPIO 推挽输出直接驱动 |
| PA13 / PA14 | STLINK-V3E SWD | 保留为调试接口 |
| PB0 / PB1 | 无板载有源电路占用 | 可用作 ADC 输入 |

---

## 7. 设计约束与待确认事项

### 7.1 设计约束

以下约束来自 MCU 资源规划和引脚配置，在软件设计中不得违反：

- PA13（SWDIO）和 PA14（SWCLK）为 SWD 调试引脚，不得配置为应用 GPIO；
- ADC1 注入组（10 kHz）和规则组（500 Hz）共用 ADC1_2 全局中断，通过事件标志区分处理，不可分别设置 NVIC 优先级；
- SF-05 实时数据处理在 JEOS 中断上下文中执行，执行时间须远小于 100 µs（10 kHz 周期）；
- PWM 停止/异常状态下 TIM1 停止，ADC1 注入组随之停止；TIM6 在所有状态下持续运行；
- 串口 JustFloat 帧为 8 字节/帧，500 Hz 发送速率下波特率 115200 bit/s 满足带宽要求。

### 7.2 待确认事项

| 编号 | 待确认内容 | 来源 |
|---|---|---|
| TBD-01 | TIM1 死区时间具体数值 | mcu_resource_plan HW-TBD-02 |
| TBD-02 | ADC1 各通道的采样时间 | mcu_resource_plan HW-TBD-03 |
| TBD-03 | PC13 按键消抖时间 | mcu_resource_plan HW-TBD-04 |
| TBD-04 | TIM1 触发 ADC1 注入组的具体触发源（TRGO/TRGO2） | mcu_resource_plan HW-TBD-05 |
| TBD-05 | LPUART1_TX 使用的 DMA1 通道号及中断入口 | mcu_resource_plan HW-TBD-01 |
| TBD-06 | VREF+ 供电配置（JP8 [2-3] 由 VDD 供电，或继续使用板载 VREF） | pin_config_spec PIN-TBD-01 |
| TBD-07 | 温度异常阈值和恢复阈值的具体数值 | design_inputs DD-05 |
| TBD-08 | PWM 占空比换算公式、限幅范围和数值处理方法 | design_inputs DD-01 |
| TBD-09 | DAC 换算公式、幅值范围和数值处理方法 | design_inputs DD-02 |
| TBD-10 | 第二组和第三组 PWM 的预设占空比具体数值 | design_inputs DD-06 |
