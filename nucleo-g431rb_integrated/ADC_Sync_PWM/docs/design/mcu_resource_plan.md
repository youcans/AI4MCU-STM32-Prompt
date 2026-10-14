# MCU 资源规划

> 编制日期：2026-08-04
> 依据文件：
> - `docs/analysis/design_inputs.md`（项目设计输入）
> - `docs/design/io_objects.md`（软件输入输出对象）
> - `docs/design/requirements_list.md`（软件功能需求列表）
> - `docs/raw/hardware/NUCLEO-G431RB_Schematic_C05.pdf`（原理图 C05）
> - `docs/raw/hardware/NUCLEO-G431RB_User_Manual_UM2505.pdf`（用户手册）
> - `docs/raw/references/STM32G431_Datasheet_DS12589.pdf`（数据手册）
> 说明：将软件功能映射到 MCU 外设和系统资源。不展开寄存器配置和程序代码实现。

---

## 1. 资源映射

| 功能 | MCU 资源 | 工作模式或通道 | 关键参数 | 关联关系 | 说明 |
|---|---|---|---|---|---|
| SF-02 PWM 输出控制 | TIM1 | 高级定时器，中心对齐 PWM 模式；CH1（PA8）/ CH1N（PB13）、CH2（PA9）/ CH2N（PB14）、CH3（PA10）/ CH3N（PB15） | 频率 10 kHz；死区插入使能（时间待确认） | TIM1 内部触发事件 → ADC1 注入组触发 | 运行状态启动 TIM1 和 PWM 输出；停止/异常状态停止 TIM1 及 ADC1 注入组采样。TIM1 触发事件的具体来源（TRGO/TRGO2）待确认 |
| SF-03 ADC 采样控制 | ADC1 | 注入组，由 TIM1 触发事件触发；Rank1=IN2（PA1）、Rank2=IN12（PB1）、Rank3=IN15（PB0） | 12 位分辨率；采样时间待确认 | TIM1 触发 → ADC1 注入组按 Rank1→2→3 顺序转换 | 三通道由同一次 TIM1 触发后顺序转换。停止/异常状态下注入组随 TIM1 停止而停止 |
| SF-04 ADC 采样结果获取 | ADC1 | 注入组 JEOS 标志（在 ADC1_2 全局中断中处理） | — | JEOS 有效 → 读取三路注入组转换结果，获得 `AdcDynamicVoltage`、`AdcHighRef`、`AdcLowRef` | 注入组不使用 DMA |
| SF-05 实时数据处理 | （无独立外设） | 在 ADC1_2 全局中断中 JEOS 分支执行 | 执行时间须远小于 100 µs（10 kHz 周期） | 输入 SF-04 的采样值；输出 `PwmDutyCh1` 和 `DacOutputValue`（均为软件对象，不直接操作外设寄存器） | 仅运行状态下由 JEOS 触发。SF-05 根据 SF-04 获取的采样值更新第一组 PWM 占空比和 DAC 方波幅值 |
| SF-06 DAC 输出控制 | DAC1 | Channel 1（PA4 / DAC1_OUT1），外部缓冲通道，12 位右对齐 | 输出范围 0～VREF+ | `DacOutputValue` 由 SF-05 根据采样结果更新；`Timer500HzEvent` 触发 DAC 在 0 与最新 `DacOutputValue` 之间写入；`DacOutputEnable` 由 `SystemState` 控制 | 运行状态下由 500 Hz 周期事件触发 DAC 电平翻转，形成 250 Hz 方波；停止和异常状态下关闭 DAC 输出 |
| SF-07 芯片温度采样 | ADC1 | 规则组，单通道 CH16（内部温度传感器 ADC1_IN16），由 TIM6_TRGO 触发 | 12 位分辨率；500 Hz | TIM6_TRGO（500 Hz）→ ADC1 规则组触发；在 ADC1_2 全局中断中判断 EOC 标志后读取温度转换结果，获得 `ChipTemperature` | 温度传感器内部连接 ADC1_IN16（DS12589 §3.18.1），无需外部引脚 |
| SF-08 温度异常判断 | （无独立外设） | 在 ADC1_2 全局中断中 EOC 分支执行 | 阈值待确认 | 输入 `ChipTemperature`、`TemperatureFaultThreshold`、`TemperatureRecoveryThreshold`；输出 `TemperatureFaultEnter` 或 `TemperatureFaultExit` | 软件比较逻辑 |
| SF-09 系统状态切换 | EXTI13 | PC13，外部中断线 13，上升沿触发；板载外部下拉电阻 | 消抖时间待确认 | `UserButtonEvent` → 停止↔运行切换；`TemperatureFaultEnter` / `TemperatureFaultExit` → 异常进入/退出 | — |
| SF-10 LED 状态指示 | GPIO 输出 | PA5，推挽输出 | — | 运行状态由 JEOS 触发翻转；异常状态由 TIM6 500 Hz 分频至 10 Hz 闪烁 | PA5 连接板载用户 LED LD2 |
| SF-11 串口数据发送 | LPUART1 + DMA1 | LPUART1_TX（PA2），异步模式；DMA1 通道由 DMAMux 分配 | 115200 bit/s；每帧 8 字节 | `Timer500HzEvent` 触发 SF-11 组织 JustFloat 帧并启动 DMA 发送 | PA2 通过板载 STLINK-V3E 转为 USB 虚拟 COM 端口。LPUART1_RX（PA3）已配置但不使用 |
| 500 Hz 系统时基 | TIM6 | 基本定时器，向上计数模式；TRGO 输出更新事件 | 500 Hz（周期 2 ms） | TIM6_TRGO → ADC1 规则组触发（SF-07）；TIM6 更新事件 → `Timer500HzEvent`，同时用于 SF-06 DAC 翻转、SF-10 异常闪烁分频、SF-11 串口发送节拍 | TIM6 承担四项职责：温度采样触发、DAC 方波翻转、LED 闪烁分频、串口发送节拍 |

---

## 2. 冲突检查

### 2.1 引脚复用

TIM1 PWM（PA8/PB13/PA9/PB14/PA10/PB15）、ADC1（PA1/PB1/PB0）、DAC1（PA4）、LPUART1（PA2/PA3）、GPIO（PA5/PC13）、SWD（PA13/PA14）引脚功能不重叠，无复用冲突。

### 2.2 ADC1 共享

ADC1 注入组（10 kHz，3 通道）和规则组（500 Hz，1 通道）共享同一 ADC 外设。注入组具有优先级，可中断规则组转换。此共享模式受 STM32G4 硬件支持。

注入组 JEOS 和规则组 EOC 共用 **ADC1_2 全局中断**入口，在中断服务中根据 JEOS 和 EOC 标志区分两类 ADC 转换完成事件，分别处理 SF-04/SF-05（JEOS，10 kHz）和 SF-07/SF-08（EOC，500 Hz）。

### 2.3 DMA

仅 LPUART1_TX 使用 DMA 发送，ADC1 注入组和规则组均不使用 DMA，DAC1 不使用 DMA。DMA 资源无冲突。

### 2.4 中断

ADC1 注入组和规则组共用 ADC1_2 全局中断，并通过事件标志区分处理；其余计划使用的中断入口（TIM6、EXTI13、DMA1）之间未发现冲突。DMA 具体通道及其中断入口在配置时确认。

---

## 3. 待确认事项

| 编号 | 待确认内容 | 说明 |
|---|---|---|
| HW-TBD-01 | LPUART1_TX 使用的 DMA1 通道号及中断入口 | 由 DMAMux 灵活分配，需在软件配置中确定 |
| HW-TBD-02 | TIM1 死区时间具体数值 | 对应 DD-07，由开发者在详细设计中确定 |
| HW-TBD-03 | ADC1 各通道的采样时间 | 根据输入阻抗和信号特性确定 |
| HW-TBD-04 | PC13 按键消抖时间 | 消抖策略影响 SF-09 实现 |
| HW-TBD-05 | TIM1 触发 ADC1 注入组的具体触发源 | TIM1 内部触发事件（TRGO/TRGO2）来源在 STM32CubeMX 配置时确认，需确保每个完整 PWM 周期产生一次 10 kHz 采样触发 |
