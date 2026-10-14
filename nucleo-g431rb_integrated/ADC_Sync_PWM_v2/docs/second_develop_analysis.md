# 二次开发可行性分析：新增 OLED 本地显示

---

## 0. 文档说明

本文档面向二次开发，评估在现有"基于 ADC 同步采样的 PWM 占空比控制"工程基础上新增 OLED 本地显示功能（[FR-10 / SF-12，见 software_spec_SE_v2.md](software_spec_SE_v2.md)）的可行性、实现条件与影响范围。

- 分析依据为**当前 `.ioc` 配置与实际源码**，不依据经验猜测；涉及但无法由现有资料确认的事项，明确标注"待确认/需确认"，不给出推测结论。
- 输入文档：[software_spec_SE_v2.md](software_spec_SE_v2.md)（含 OLED 新需求的功能说明书）、[code_analysis_summary.md](code_analysis_summary.md)（代码解析汇总报告）。后者中的运行链路 L0–L5、数据流 DF-1–DF-4、共享数据、中断优先级等结论直接采用，不重复推导。
- 本轮仅做可行性分析，未修改任何程序文件、`.ioc` 文件或软件说明书。
- 本章涉及的关键源码事实均给出位置，二次开发常用入口见 [code_analysis_summary.md §10](code_analysis_summary.md)。

---

## 1. 硬件资源可行性

### 1.1 目标硬件

- MCU：STM32G431RBT6（NUCLEO-G431RB 开发板），SYSCLK=160 MHz，APB1/APB2 不分频（TIM 与 PCLK 均为 160 MHz）。详见 [code_analysis_summary.md §1](code_analysis_summary.md)。
- OLED 模块（据 [software_spec_SE_v2.md FR-10](software_spec_SE_v2.md)）：0.96" SSD1306 / 128×64 / I2C 接口 / 3.3V 供电。供电电压与开发板逻辑电平一致，电源层面可行。

### 1.2 I2C 外设当前配置情况（依据 `.ioc`）

查验 `ADC_Sync_PWM.ioc`：

- `Mcu.IP0~Mcu.IP9` 列出当前启用的 IP 为：ADC1、DAC1、DMA、LPUART1、NVIC、RCC、SYS、TIM1、TIM6、NUCLEO-G431RB，**不含任何 I2C 外设**。
- `Mcu.IPNb=10`（共 10 个 IP 项，与上列一致）。
- NVIC 节中已启用外设中断为 `ADC1_2_IRQn`、`TIM6_DAC_IRQn`、`DMA1_Channel2_IRQn`、`LPUART1_IRQn`、`EXTI15_10_IRQn`（优先级依次 0/1/2/3/4）及系统异常，**无 `I2C1_EV_IRQn` / `I2C1_ER_IRQn` 等 I2C 中断**。
- `RCC` 节中存在 `I2C1Freq_Value=160000000`、`I2C2Freq_Value=160000000`、`I2C3Freq_Value=160000000` 条目。该值是 CubeMX 对 I2C 时钟分支的时钟树计算/记录值；**判定 I2C 外设当前是否启用的权威依据是 `Mcu.IP` 列表与 NVIC 列表，二者均不含 I2C，故判定当前工程未启用任何 I2C 外设**。RCC 中存在该条目不表示 I2C 已启用。

**结论**：STM32G431 片上具备 I2C1/I2C2/I2C3 外设资源可供启用，当前工程尚未启用其中任何一个。新增 OLED 显示需在 CubeMX 中启用一个 I2C 外设并生成对应初始化代码，这一步是本次二次开发**必须修改 `.ioc` 的唯一实质内容**（详见第 5 节）。

### 1.3 引脚资源

依据 `.ioc` 的 `Mcu.Pin*` 清单，当前已占用引脚为：

- PA1、PA2、PA3、PA4、PA5、PA8、PA9、PA10、PA13、PA14
- PB0、PB1、PB13、PB14、PB15
- PC13
- PF0-OSC_IN、PF1-OSC_OUT

其中 PA13/PA14 为 SWD 调试口，[software_spec_SE_v2.md §5.1](software_spec_SE_v2.md) 已注明"不得配置为应用 GPIO"；PC13 为按键、PF0/PF1 为 HSE 晶振（24 MHz），均不可挪用。其余未出现在占用清单中的 GPIO 引脚，原则上可作为 I2C SCL/SDA 的候选。

**待确认（不在本文结论范围内）**：哪一对空闲引脚映射到 STM32G431 的 I2C 复用功能（I2C1/2/3 的 SCL/SDA 复用映射需结合 STM32G431 数据手册的复用功能表确认），以及该引脚对在 NUCLEO-G431RB 开发板上是否引出到 Arduino/CN 接头并便于外接 OLED 模块。这些属于硬件接线层面的设计事项，应在详细设计阶段用 CubeMX 引脚视图与开发板原理图确认，本文不据经验指定。

### 1.4 时钟与存储

- I2C 时钟：当前 CubeMX 时钟树记录的 I2C 内核时钟值为 160 MHz（`.ioc` `RCC.I2C1/2/3Freq_Value=160000000`），时钟源充足。具体时钟源与 I2C 时序参数在启用 I2C 后由 CubeMX 确认；SSD1306 常用的 I2C 标准模式 100 kHz / 快速模式 400 kHz 在该时钟量级下通常可由 CubeMX 计算出合规时序，可行性成立（本文不据经验指定具体参数）。
- 存储：STM32G431RBT6 名义为 128 KB Flash / 32 KB SRAM。SSD1306 显示所需资源大致为 GDDRAM 显存缓冲 128×64/8 = 1024 字节、驱动代码与字模（按所用字体而定，量级有限），相对 32 KB SRAM 与 128 KB Flash 可承受。**待确认**：当前工程实际 Flash/RAM 占用量须以编译后 map 文件为准，确认新增模块后留有余量。

### 1.5 硬件小结

硬件资源可行：I2C 外设可启用、空闲引脚具备候选、电源与时钟满足要求、存储量级可承受。需在详细设计阶段确认的具体引脚、板载接线、上拉电阻与最终存储占用，见各节"待确认"项。

---

## 2. 软件实现条件

### 2.1 运行架构对显示功能的适配

本工程为**纯中断驱动**架构：[main.c 的 `while(1)` 主循环为空](../Core/Src/main.c)（`USER CODE BEGIN WHILE`，[main.c:106-111](../Core/Src/main.c)）；全部业务逻辑由 5 条中断链路（L0–L5）驱动（见 [code_analysis_summary.md §5](code_analysis_summary.md)）。

这一架构对 OLED 显示恰好有利：

- **主循环空闲**，可在其中放入 OLED 刷新逻辑而不抢占任何实时中断的执行时间；
- OLED 属慢速外设（人眼可辨的刷新频率远低于 kHz 级实时链路），把其重负载的 I2C 传输放在主循环轮询执行，与 10 kHz ADC 采样/500 Hz 调度等实时链路天然隔离。

### 2.2 新增模块

建议按现有 App 模块组织方式（`app_<模块>.c/h`，公开接口前缀为模块缩写，编号继续向后）新增一个显示模块（例如 `app_display.c/h`，前缀如 `DISP_`），职责为：SSD1306 初始化、显存管理、组帧与经 I2C 输出。模块仅依赖 HAL I2C 接口与其它 App 模块的只读访问器，不承担业务状态机逻辑，符合现有"薄封装+业务分离"的风格（参 [code_analysis_summary.md §2](code_analysis_summary.md)）。

构建层面需将该模块源文件加入 [CMakeLists.txt:46-52](../CMakeLists.txt) 的 `target_sources`；头文件目录 `App/Inc` 已在 [CMakeLists.txt:56-58](../CMakeLists.txt) 注册，新增 `app_display.h` 放入即可被包含，不需改动 include 目录。

### 2.3 I2C HAL 与 CubeMX 生成代码

启用 I2C 后 CubeMX 将生成：

- `MX_I2C1_Init()`（或所选实例）及全局句柄 `hi2c1`；
- 所选 SCL/SDA 引脚的 GPIO 复用配置（在 `gpio.c` / `stm32g4xx_hal_msp.c`）；
- 若采用中断/DMA 方式传输，还需配置 `I2C1_EV_IRQn` / `I2C1_ER_IRQn` 及其 NVIC 优先级（详见第 4、6 节对优先级的要求）。

驱动 SSD1306 可用 `HAL_I2C_Master_Transmit`（阻塞）发送命令与显存数据；是否采用中断/DMA 方式属设计选择，见第 4 节。

### 2.4 软件小结

软件实现条件具备：主循环空闲为显示刷新提供天然执行点；新增一个 App 模块并在 CMake 注册即可融入现有结构；I2C 初始化由 CubeMX 生成。无需改动现有 HAL/CMSIS 库。

---

## 3. 显示数据来源

[software_spec_SE_v2.md SF-12 / §4.2](software_spec_SE_v2.md) 要求 OLED 显示系统状态、芯片温度及主要运行数据。下表对照当前工程对每项数据的可获取性（依据实际源码，非推测）：

| 显示项 | 现有访问器 | 位置 | 可获取性 / 缺口 |
|---|---|---|---|
| 系统状态 `SystemState`（停止/运行/异常） | **无** | [app_control.h:19-21](../App/Inc/app_control.h) 仅声明 `APP_Control_Init`、`CTRL_NotifyInjectedComplete`、`CTRL_ProcessTemperature` 三个函数 | **缺口**：`SystemState_t` 枚举与 `static volatile system_state` 均定义在 [app_control.c:15-19](../App/Src/app_control.c) / [:30](../App/Src/app_control.c) 内部，外部模块无法访问。需新增 getter（如 `CTRL_GetSystemState`）并将枚举类型暴露至头文件，方可供显示模块读取 |
| 芯片温度 `ChipTemperature` | `SAMP_GetTemperature()` | [app_sampling.h:19](../App/Inc/app_sampling.h) | 可获取（`volatile float`，最新一次已完成值） |
| 温度是否有效 | `SAMP_IsTemperatureValid()` | [app_sampling.h:20](../App/Inc/app_sampling.h) | 可获取；可用于避免冷启动显示未定义温度 |
| DAC 方波幅值 `DacOutputValue`（= 动态电压 1:1） | `PROC_GetDacAmplitude()` | [app_process.h:20](../App/Inc/app_process.h) | 可获取；注意返回值为 0~4095 的 12 bit 原始整数 float（非换算电压，量纲说明见 [code_analysis_summary.md §9.3 N-3](code_analysis_summary.md)） |
| 动态电压采样值 `AdcDynamicVoltage`（ch1 原始） | 无独立 getter | `PROC_HandleSamples` 内 `dac_amplitude = ch1`（[app_process.c:13](../App/Src/app_process.c)） | 与 `DacOutputValue` 一一对应（1:1），可经 `PROC_GetDacAmplitude` 间接获取 |
| 第一组 PWM 占空比 `PwmDutyCh1` | 无 getter | `PROC_HandleSamples` 内即时写 CCR1，不持存 | **缺口**：显示模块无法直接读到占空比。设计阶段可选——由 `dac_amplitude` 推算 `duty = ch1*8000/4095`，或新增占空比 getter；属设计选择 |
| 第二/三组占空比、参考通道 ch2/ch3 | 不持存 | `PROC_HandleSamples` 中 ch2/ch3 被丢弃（参 DF-1） | 一般不作为显示项；如需显示须新增持存与 getter |

**缺口情况**：在当前已经明确的显示数据中，系统状态缺少现成访问接口（需新增 getter 并暴露 `SystemState_t` 类型）；其余已经明确的运行数据（温度、DAC 幅值/动态电压、温度有效性）均有现成访问器。其他主要运行数据是否还需新增接口，取决于后续确定的具体显示字段（如是否显示 PWM 占空比、ch2/ch3 等，见上表"设计选择/须新增持存与 getter"各行）。

> 备注：显示模块对上述变量的访问均为**只读**，不写入任何现有共享数据；多在主循环上下文读取（非 ISR 上下文），与现有 ISR 写入方的关系为"ISR 写、主循环读"，`volatile` 已保证可见性，单次读取为整型/float 单拷贝原子（Cortex-M4 上 32 位与 float 读写原子），不存在显示侧导致的撕裂。需注意：显示模块在组屏时连续读取多个共享变量（如 `system_state` 与温度、DAC 幅值），各次读取之间可能被 ISR 更新，因此一帧显示内容**未必是同一时刻的完整快照**（例如状态已切至 RUN 而 DAC 幅值仍为切换前的旧值）。这与各实时链路本身的并发竞态点（[code_analysis_summary.md §9.3 N-1/N-2](code_analysis_summary.md)）相互独立，不改变其性质，且对 OLED 显示（慢速、人眼辨识）而言不构成实质风险。

---

## 4. 运行机制与刷新条件

### 4.1 刷新执行位置

OLED 刷新的**重负载部分（I2C 传输数百字节显存）应放在主循环 `while(1)` 内轮询执行**，即 [main.c `USER CODE BEGIN WHILE`](../Core/Src/main.c)（[main.c:110](../Core/Src/main.c)）。原因：

1. 主循环空闲，I2C 传输不占用任何实时中断时间，确保 ADC 注入链路（L1，10 kHz）等 5 条链路的实时性不受影响——这直接满足 [FR-10"新增显示功能不得影响原有 ADC 采样、PWM、DAC、串口通信、状态切换和温度异常处理功能"](software_spec_SE_v2.md) 的约束。
2. I2C 阻塞传输期间若被任意实时中断抢占，硬件 I2C 外设自身维持总线状态，传输在中断返回后正常继续，仅时长略增，不产生数据错误。

初始化部分应在系统初始化阶段调用一次（可新增 `DISP_Init()`，调用位置紧邻 [main.c:101 `APP_Control_Init()`](../Core/Src/main.c) 的 `USER CODE BEGIN 2`，或并入 `APP_Control_Init` 末尾）。

### 4.2 刷新方式与周期

[software_spec_SE_v2.md SF-12](software_spec_SE_v2.md) 将刷新周期与刷新方式明确列为"详细设计中确定"。本文仅给出工程层面的可行性边界：

- **全屏刷新耗时**：128×64 显存 1024 字节，按 I2C 快速模式 400 kHz 计，单次传输量级约二十余毫秒（含控制字节与 HAL 开销），可支撑数 Hz 级全屏刷新；若需更高刷新率或降低总线占用，可采用分页/分区局部刷新。具体刷新周期与策略由详细设计在满足"不影响原有功能"前提下确定。
- **触发方式**：可采用主循环按周期轮询刷新（最简）；若希望状态变化时立即刷新，可在状态切换的 ISR 内置一个 `volatile` 标志（仅置位，不做 I2C 传输），主循环查询该标志后执行刷新——**任何 I2C 传输都不得放在 ISR 内**，以免占用实时中断时间。

### 4.3 I2C 传输模式与 NVIC 影响

- **阻塞模式（`HAL_I2C_Master_Transmit`）**：不引入新中断，对现有 NVIC 优先级体系零影响。鉴于显示刷新慢速、主循环空闲，推荐优先采用此模式。
- **中断/DMA 模式**：需启用 `I2C1_EV_IRQn`/`I2C1_ER_IRQn`，其抢占优先级数值须**大于等于 4**（即低于或等于现有最低的 EXTI15_10=4），确保不抢占任何现有实时链路。此模式为可选项，非必要。

### 4.4 与共享数据的一致性

显示为被动反映系统快照：主循环每次刷新读取当时的 `system_state`、温度、DAC 幅值等。由于组屏时连续读取多个共享变量、之间可能被 ISR 更新，**一帧显示内容未必是同一时刻的完整快照**（状态切至 RUN 而 DAC 幅值仍是切换前旧值即属此类）。但下一次刷新即会反映更新后的值，偶发的"显示略滞后于实际状态一拍"对人眼无实际影响，也不违背任何功能一致性要求。这与各实时链路的并发竞态点（[code_analysis_summary.md §9.3 N-1/N-2](code_analysis_summary.md)）相互独立，不改变其性质，对 OLED 显示（慢速、人眼辨识）不构成实质风险。

---

## 5. 修改影响范围

### 5.1 必须修改的范围

| 范围 | 修改内容 | 依据 |
|---|---|---|
| `ADC_Sync_PWM.ioc` | 在 CubeMX 中启用一个 I2C 外设，选定 SCL/SDA 引脚并配置复用与时序；若采用中断/DMA 模式则一并配置其 NVIC | 新增外设功能，[software_spec_SE_v2.md SF-12](software_spec_SE_v2.md) 要求 I2C 接口；当前 `.ioc` 无 I2C（第 1.2 节）。这是本次唯一必须改动的 `.ioc` 内容 |
| CubeMX 重新生成代码 | `MX_I2C1_Init`、I2C 的 MSP/GPIO 初始化、（中断模式下）`I2C1_EV/ER` 中断接入 | CubeMX 重新生成会覆盖标记区外内容，`USER CODE BEGIN/END` 区保留（机制见 [code_analysis_summary.md §8](code_analysis_summary.md)）；生成后须以差异视图审阅，确认用户代码区未被破坏 |

> 关于 [CLAUDE.md "修改代码前……不修改.ioc文件和与当前任务无关的文件"] 项目规则与"需启用 I2C"的冲突：启用 I2C 外设是 OLED 功能的前提，无替代途径。但 [CLAUDE.md](../CLAUDE.md) 目前仍规定"不修改.ioc"，本文档不藉由软件说明书的修订来绕开该规则。**在后续实施前，须先明确修订 [CLAUDE.md](../CLAUDE.md) 的该项规则，或就本次 CubeMX 修改 `.ioc` 取得单独授权**，再据此进行 I2C 启用与代码重生成；此前本文档仅将"需启用 I2C（须经 CubeMX 正常流程完成、审阅差异）"列为必要前提，不构成对现有规则的自行放开。

### 5.2 新增文件

| 文件 | 职责 |
|---|---|
| `App/Inc/app_display.h`、`App/Src/app_display.c` | OLED 显示模块：SSD1306 初始化、显存组帧、经 I2C 输出、读取显示数据组屏 |
| 字模/字体资源（可选独立文件或并入模块） | 显示文字所需字模 |

新增文件在 [CMakeLists.txt:46-52](../CMakeLists.txt) 的 `target_sources` 增加 `App/Src/app_display.c` 一行；`App/Inc` 已注册，无需改 include 目录。

### 5.3 现有文件的最小改动

| 文件 | 改动 | 性质 |
|---|---|---|
| [app_control.h](../App/Inc/app_control.h) | 暴露 `SystemState_t` 枚举类型；新增 `CTRL_GetSystemState()` 声明 | 纯新增声明，不改任何逻辑 |
| [app_control.c](../App/Src/app_control.c) | 实现 `CTRL_GetSystemState()` 返回 `system_state`；`SystemState_t` 定义移至头文件或保留并经头文件可见 | 纯新增只读访问器 + 类型位置调整，不触碰状态机逻辑 |
| [main.c](../Core/Src/main.c) | `USER CODE BEGIN 2` 区增加 `DISP_Init()` 调用；`USER CODE BEGIN WHILE` 区增加刷新调用 | 仅写入用户代码区，不入非用户区 |

这些改动均为**纯增量、只读暴露**，不修改状态机、数据处理、调度、串口等任何现有业务逻辑，也不写入现有共享数据。

### 5.4 不修改的范围

- 现有 5 个 App 模块的业务实现（`app_bsp.c`、`app_sampling.c`、`app_process.c`、`app_monitor.c` 以及 `app_control.c` 中除新增 getter/类型暴露外的全部逻辑）；
- `app_sampling.h`/`app_process.h`/`app_monitor.h`（其访问器已满足显示需求，不需改动）；
- Core 下外设初始化源码（除非 CubeMX 重新生成时因新增 I2C 而由 CubeMX 自身写入，此时属生成产物而非手改）；
- `stm32g4xx_it.c`（阻塞模式下不需新中断；中断模式下中断 handler 由 CubeMX 生成）；
- HAL 与 CMSIS 库文件；
- [software_spec_SE.md](software_spec_SE.md)（原版说明书）。

---

## 6. 风险与注意事项

| 项 | 说明与规避 |
|---|---|
| **I2C 传输不得进入 ISR** | 显示刷新的 I2C 传输若放在 ADC/TIM6 等 ISR 内，会直接占用实时中断时间，可能突破 10 kHz 采样周期，违反 FR-10。**必须放在主循环**；如需事件触发，ISR 仅置标志、主循环执行传输（见第 4.2 节） |
| **I2C 中断优先级（若采用 IT/DMA 模式）** | `I2C1_EV/ER_IRQn` 抢占优先级数值须 ≥4，低于全部现有实时链路（ADC=0、TIM6=1、DMA=2、LPUART=3、EXTI=4）。推荐阻塞模式以彻底规避此风险 |
| **`.ioc` 变更与 CubeMX 重生成** | 重新生成会覆盖用户标记区外内容。生成后须以差异视图审阅，确认 [main.c](../Core/Src/main.c) 等处的 `USER CODE` 区内容完好、I2C 引脚与现有已占用引脚无冲突 |
| **引脚冲突** | 所选 SCL/SDA 须避开第 1.3 节列出的已占用引脚；尤其不可使用 PA13/PA14（SWD）、PC13（按键）、PF0/PF1（HSE）。具体引脚与复用须在 CubeMX 引脚视图确认 |
| **I2C 上拉电阻** | SSD1306 模块通常自带 SCL/SDA 上拉；若所用模块无上拉，须外接（典型 4.7 kΩ）。是否需要外接属硬件接线事项，**待确认** |
| **总线占用与板载 I2C 设备** | NUCLEO-G431RB 板上是否有已占用的 I2C 设备，本文不能由 `.ioc` 确认（当前 `.ioc` 无 I2C 故无记录）。**待确认**开发板原理图，避免地址或总线冲突 |
| **显示数据滞后** | 状态切换或数据更新与刷新非同步，显示可能滞后一拍；人眼无感，且不构成功能错误（见第 4.4 节） |
| **Flash/RAM 占用** | 新增驱动与字模的占用须在编译后据 map 文件确认留有余量（见第 1.4 节） |
| **与现有并发竞态点的关系** | [code_analysis_summary.md §9.3 N-1/N-2](code_analysis_summary.md) 所列并发注意点属现有实时链路内部，OLED 显示为只读旁路、不参与其读写，不改变其性质。但显示模块组屏时连续读取多个共享变量、其间可能被 ISR 更新，一帧显示内容未必是同一时刻的完整快照；对 OLED 显示（慢速、人眼辨识）不构成实质风险，仅作说明 |

---

## 7. 可行性结论

**总体可行。** 在当前工程基础上新增 OLED 本地显示功能，硬件资源、软件架构、数据来源三方面均具备条件：

1. **硬件**：片上 I2C 外设可启用（当前未启用）、空闲引脚具备候选、3.3V 电源与时钟（160 MHz）满足 SSD1306 I2C 时序、存储量级可承受。
2. **软件**：纯中断驱动架构下主循环空闲，为慢速 OLED 刷新提供天然执行点；按现有模块风格新增显示模块并在 CMake 注册即可融入；I2C 初始化由 CubeMX 生成。
3. **数据**：在当前已经明确的显示数据中，芯片温度、温度有效性、DAC 幅值/动态电压等均有现成只读访问器；系统状态缺少现成接口，需在 `app_control` 模块新增 getter 并暴露 `SystemState_t` 类型（纯只读暴露）。其他主要运行数据是否还需新增接口，取决于后续确定的具体显示字段。
4. **非干扰性**：只要把 I2C 传输放在主循环（不进 ISR）、且新 I2C 中断（若有）优先级低于现有实时链路，则现有 5 条中断链路与 4 条数据流不受影响，满足 [FR-10](software_spec_SE_v2.md)"不得影响原有功能"的约束。显示为只读旁路，不修改任何现有共享数据；组屏连续读取多个共享变量时一帧内容未必是同一时刻完整快照，但对慢速人眼显示不构成实质风险（见第 6 节）。

**必要前提**：须在 CubeMX 中启用 I2C 并重新生成代码（本次唯一必须改动的 `.ioc` 内容）；改动须经差异视图审阅、编译/烧录/运行验证。

**留待详细设计确认（本文不据经验指定）**：I2C 外设实例与 SCL/SDA 具体引脚及复用映射、板载接线与上拉、是否选用中断/DMA 模式、刷新周期与刷新策略、具体显示字段与版式、最终 Flash/RAM 占用。这些项与 [software_spec_SE_v2.md SF-12](software_spec_SE_v2.md) 标注"详细设计中确定"一致，进入设计阶段后据 CubeMX、开发板原理图与编译产物确认。

---

## 8. 附：关键事实索引（本次分析所据）

| 事实 | 来源 |
|---|---|
| 当前启用 IP（无 I2C） | `ADC_Sync_PWM.ioc` `Mcu.IP0~IP9` / `Mcu.IPNb=10` |
| 已启用中断及优先级（无 I2C 中断） | `ADC_Sync_PWM.ioc` NVIC 节 |
| RCC 中 I2C 频率记录值 | `ADC_Sync_PWM.ioc` `RCC.I2C1/2/3Freq_Value=160000000` |
| 主循环为空、中断驱动 | [main.c:106-111](../Core/Src/main.c)（[code_analysis_summary.md §3](code_analysis_summary.md)） |
| 用户应用启动集成点 | [main.c:101](../Core/Src/main.c) `APP_Control_Init()` |
| `SystemState_t` 枚举与 `system_state` 定义位置 | [app_control.c:15-19](../App/Src/app_control.c) / [:30](../App/Src/app_control.c) |
| `app_control.h` 仅 3 个函数、无状态 getter | [app_control.h:19-21](../App/Inc/app_control.h) |
| 温度 / 温度有效 / DAC 幅值访问器 | [app_sampling.h:19-20](../App/Inc/app_sampling.h) / [app_process.h:20](../App/Inc/app_process.h) |
| App 模块构建注册 | [CMakeLists.txt:46-58](../CMakeLists.txt) |
| 运行链路/数据流/共享数据/并发注意点 | [code_analysis_summary.md §5 / §7 / §9](code_analysis_summary.md) |
| OLED 功能需求与待定项 | [software_spec_SE_v2.md FR-10 / SF-12 / §4.2 / §5](software_spec_SE_v2.md) |
