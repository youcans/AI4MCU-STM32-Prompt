# 软件功能需求列表

> 编制日期：2026-08-04
> 依据文件：
> - `docs/analysis/design_inputs.md`（项目设计输入）
> - `docs/design/io_objects.md`（软件输入输出对象）
> 说明：将项目功能要求转化为软件需要实现的功能条目。不展开 MCU 资源、引脚配置、软件模块和代码实现；不补充未经确认的功能和参数。

---

| 功能编号 | 功能名称 | 输入 | 输出 | 触发条件 | 运行效果 |
|---|---|---|---|---|---|
| SF-01 | 系统初始化 | — | `SystemState`（停止）、`PwmOutputEnable`（关闭）、`AdcSamplingEnable`（停止）、`DacOutputEnable`（关闭）、`LedOutput`（熄灭）、`PwmDutyCh2`（预设值）、`PwmDutyCh3`（预设值） | 系统上电或复位 | 系统进入停止状态，各功能模块处于初始状态 |
| SF-02 | PWM 输出控制 | `SystemState`、`PwmDutyCh2`（预设值）、`PwmDutyCh3`（预设值） | `PwmOutputEnable` | `SystemState` 进入或离开运行状态 | 运行状态下开启三组 PWM 输出（10 kHz，死区互补）；停止和异常状态下关闭全部 PWM 输出 |
| SF-03 | ADC 采样控制 | `SystemState` | `AdcSamplingEnable` | `SystemState` 进入或离开运行状态 | 运行状态下启动三通道模拟电压采样（由 `PwmPeriodEvent` 触发，按预定顺序转换）；停止和异常状态下停止采样 |
| SF-04 | ADC 采样结果获取 | `AdcSequenceCompleteEvent` | `AdcDynamicVoltage`、`AdcHighRef`、`AdcLowRef` | `AdcSequenceCompleteEvent`（仅运行状态） | 每次三通道采样序列完成后读取并保存三路原始采样值 |
| SF-05 | 实时数据处理 | `AdcDynamicVoltage`（来自 SF-04） | `PwmDutyCh1`、`DacOutputValue` | `AdcSequenceCompleteEvent`（仅运行状态） | 将动态模拟电压采样值转换为第一组 PWM 占空比和 DAC 输出值并更新。模拟电压采样、数据处理、PWM 更新和 DAC 更新之间保持 10 kHz 稳定周期关系 |
| SF-06 | DAC 输出控制 | `SystemState`、`DacOutputValue`（来自 SF-05）、`Timer500HzEvent` | `DacOutputEnable` | `SystemState` 进入或离开运行状态，以及运行状态下每次 `Timer500HzEvent` | 运行状态下开启 DAC 输出（固定频率参考方波，幅值随采样结果更新）；停止和异常状态下关闭 DAC 输出 |
| SF-07 | 芯片温度采样 | — | `ChipTemperature` | `Timer500HzEvent`（全状态） | 采集 MCU 内部温度传感器数据并换算为温度值。在停止、运行和异常三种状态下均持续执行 |
| SF-08 | 温度异常判断 | `ChipTemperature`、`TemperatureFaultThreshold`、`TemperatureRecoveryThreshold` | `TemperatureFaultEnter`、`TemperatureFaultExit` | 每次获取 `ChipTemperature` 后 | 将当前温度与 `TemperatureFaultThreshold` 和 `TemperatureRecoveryThreshold` 比较：达异常阈值时产生 `TemperatureFaultEnter`，达恢复阈值时产生 `TemperatureFaultExit`。两阈值为独立值以避免临界抖动 |
| SF-09 | 系统状态切换 | `UserButtonEvent`、`TemperatureFaultEnter`、`TemperatureFaultExit` | `SystemState` | `UserButtonEvent`、`TemperatureFaultEnter` 或 `TemperatureFaultExit` | 停止状态与运行状态之间由用户按键切换；芯片温度达异常阈值时从当前状态自动进入异常状态；温度达恢复阈值时自动从异常状态返回停止状态。异常状态下按键不产生切换；返回停止后需再次按键才能进入运行 |
| SF-10 | LED 状态指示 | `SystemState`、`AdcSequenceCompleteEvent`（运行状态） | `LedOutput` | `SystemState` 变化，或运行状态下每次 `AdcSequenceCompleteEvent`，或异常状态下 10 Hz 周期 | 停止状态：LED 熄灭；运行状态：每次采样序列完成后翻转，肉眼观察近似常亮；异常状态：以 10 Hz 频率闪烁 |
| SF-11 | 串口数据发送 | `SystemState`、`DacOutputValue`、`ChipTemperature` | `SerialTxData` | `Timer500HzEvent`（全状态） | 以 JustFloat 格式、115200 bit/s、500 Hz 向上位机 VOFA+ 发送数据。运行状态发送 `DacOutputValue`，停止和异常状态发送 `ChipTemperature` |
