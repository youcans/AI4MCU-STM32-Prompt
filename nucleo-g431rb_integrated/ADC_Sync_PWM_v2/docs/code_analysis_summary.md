# 代码解析报告

---

## 0. 文档说明

本文档对前两份解析文档 [`code_structure_analysis.md`](code_structure_analysis.md)（结构/模块）与 [`code_flow_analysis.md`](code_flow_analysis.md)（调用链/数据流）中**已经确认的结果**进行整理、归纳与整合，面向后续二次开发。

- 已有明确结论的内容直接采用，不重复其形成过程，不重新分析源码、不重新推导技术结论。
- 与软件说明书 [`software_spec_SE.md`](software_spec_SE.md) 的设计一致性结论一并整合。
- 若两份文档的结论有冲突，仅指出冲突，不自行重新分析并给出新结论（本轮整理未发现冲突）。
- 本轮仅整合代码解析结果，未修改任何程序文件、`.ioc` 文件或软件说明书。

**关键入口速查（二次开发）：**

| 关注点 | 入口位置 |
|---|---|
| 用户应用唯一启动集成点 | [main.c:101](../Core/Src/main.c) `APP_Control_Init()`（`USER CODE BEGIN 2`） |
| 系统状态机 / 500 Hz 调度 / 按键 / 温度判断 | [app_control.c](../App/Src/app_control.c) [MD-04] |
| 注入组三通道读取 + 温度换算 | [app_sampling.c](../App/Src/app_sampling.c) [MD-02] |
| 占空比 / DAC 幅值计算 | [app_process.c](../App/Src/app_process.c) [MD-03] |
| 串口 JustFloat 组包 / DMA 发送 | [app_monitor.c](../App/Src/app_monitor.c) [MD-05] |
| 外设驱动薄封装（启停/写入） | [app_bsp.c](../App/Src/app_bsp.c) [MD-01] |

---

## 1. 工程总体结构

- **目标平台**：NUCLEO-G431RB（STM32G431RBT6），SYSCLK=160 MHz（HSE 24 MHz 经 PLLM/3 → PLLN×40 → PLLR/2）。HAL 库，CubeMX 生成 + CMake 构建。
- **主要目录**：
  - `Core/` — CubeMX 生成代码：`main.c`、外设初始化（`gpio/adc/dac/tim/usart/dma.c`）、中断向量（`stm32g4xx_it.c`）、`main.h` 等。
  - `App/` — 用户应用代码：5 个模块（`Inc/` 头文件、`Src/` 实现），是二次开发的主要工作区。
  - `Drivers/` — HAL + CMSIS 库，不修改。
  - `ADC_Sync_PWM.ioc` — CubeMX 工程配置；`CMakeLists.txt` / `cmake/` — 构建配置。
- **代码组织**：App 模块在顶层 `CMakeLists.txt` 显式加入 `target_sources` / `target_include_directories`；CubeMX 代码经 `cmake/stm32cubemx/CMakeLists.txt` 由 INTERFACE 库 `stm32cubemx` 提供头路径与宏（`USE_HAL_DRIVER`、`STM32G431xx`）。两层在构建层面隔离清晰。
- **程序运行方式**：**纯中断驱动**。`main()` 完成初始化后进入空 `while(1)`；全部应用逻辑由 5 条中断链路驱动（ADC 注入/规则完成、TIM6 周期、EXTI 按键、LPUART1 发送完成）。

---

## 2. 主要应用模块

App 目录 5 个模块，前缀 `BSP_`/`SAMP_`/`PROC_`/`CTRL_`/`MON_`，编号 MD-01~MD-05。

| 模块（编号） | 职责 | 主要公开接口 |
|---|---|---|
| 板级支持 MD-01 | 对 HAL 的薄封装，启停/写入外设，无业务逻辑 | `BSP_StartRealtimeChain` / `BSP_StopRealtimeChain`（启停实时链）；`BSP_WritePwmDutyCh1/2/3`、`BSP_WriteDac`、`BSP_StartDac`/`BSP_StopDac`、`BSP_ToggleLed`/`BSP_SetLedLow`、`BSP_StartDmaTx`、`BSP_StartTim6`、`BSP_StartRegularAdc` |
| 采样管理 MD-02 | 注入组完成回调（读三通道转发）+ 规则组完成回调（温度换算转发）；实现两 ADC 回调 | `SAMP_GetTemperature` / `SAMP_IsTemperatureValid`（另实现 `HAL_ADCEx_InjectedConvCpltCallback`、`HAL_ADC_ConvCpltCallback`） |
| 实时处理 MD-03 | 由动态电压算 PWM CH1 占空比与 DAC 方波幅值 | `PROC_HandleSamples(ch1,ch2,ch3)`、`PROC_GetDacAmplitude` |
| 系统控制 MD-04 | STOP/RUN/FAULT 状态机、500 Hz 调度、按键、温度异常判断；实现 TIM6/EXTI 两回调 | `APP_Control_Init`、`CTRL_NotifyInjectedComplete`、`CTRL_ProcessTemperature`（另实现 `HAL_TIM_PeriodElapsedCallback`、`HAL_GPIO_EXTI_Callback`） |
| 串口监测 MD-05 | JustFloat 组包与 DMA 发送管理；实现 Tx 完成回调 | `MON_SendData`、`MON_IsBusy`（另实现 `HAL_UART_TxCpltCallback`） |

---

## 3. 启动流程

`main()` → `HAL_Init()` → `SystemClock_Config()`（160 MHz）→ 外设初始化序列（CubeMX，仅配置不启动计数/转换）：

`MX_GPIO_Init → MX_DMA_Init → MX_TIM1_Init → MX_ADC1_Init → MX_LPUART1_UART_Init → MX_DAC1_Init → MX_TIM6_Init`

→ `APP_Control_Init()`（用户代码区，MD-04）：
1. `system_state = STOP`；
2. 写 CH2/CH3 预设占空比（800/1600，对应 10%/20%）；
3. LED 熄灭、DAC 关闭；
4. `BSP_StartRegularAdc`（ADC1 校准 + 启动规则组温度采样，触发源 TIM6_TRGO）；
5. `BSP_StartTim6`（启动 500 Hz 周期中断，其 TRGO 同时驱动温度采样）。

→ 进入空 `while(1)`。

**启动后状态**：TIM1 尚未启动，PWM 无输出、注入组未运行；只有 TIM6（500 Hz）与规则组温度采样运行。系统停在 STOP，温度以 500 Hz 采样，按 500 Hz 周期调度串口发送温度（待 `temperature_valid` 后；实际成功帧率受 `dma_busy` 制约，可能低于 500 Hz），LED 熄灭。

---

## 4. 模块关系

- **MD-01** 位于最底层（仅依赖 HAL 头文件），供 MD-03、MD-04 和 MD-05 调用。
- **MD-04** 处于核心调度位置，`#include` 其余全部 4 个模块，在 500 Hz 调度中调用 `SAMP_*`（取温度）与 `MON_SendData`/`PROC_GetDacAmplitude`。
- **MD-04 与 MD-02 双向依赖**：MD-04 调用 MD-02 的 `SAMP_IsTemperatureValid`/`SAMP_GetTemperature`；MD-02 又在回调中调用 MD-04 的 `CTRL_NotifyInjectedComplete`/`CTRL_ProcessTemperature` 转发数据。
- **MD-03** 为 MD-02 的下游（仅依赖 MD-01），提供 `PROC_GetDacAmplitude` 供 MD-04 读取。
- **MD-05** 仅由 MD-04 调用（仅依赖 MD-01）。

> 应用与 CubeMX 代码的关联不止 `APP_Control_Init` 一处：①直接使用 CubeMX 全局外设句柄（`htim1`/`htim6`/`hadc1`/`hdac1`/`hlpuart1`）；②调用 HAL 接口操作外设；③通过 HAL 弱定义回调覆盖挂接业务（5 个 `HAL_*Callback` 在 App 内重定义）。

---

## 5. 关键运行链路与数据流

### 5.1 主要运行链路（5 条 + 启动）

| 链路 | 触发源 | 中断/优先级 | 应用入口 | 频率 | 有效状态 |
|---|---|---|---|---|---|
| L0 启动 | 上电/复位 | — | `APP_Control_Init` [MD-04] | 一次 | 上电 |
| L1 动态采样处理 | TIM1 周期→TRGO2→注入组 | ADC1_2 / 0 | `HAL_ADCEx_InjectedConvCpltCallback` [MD-02] | 10 kHz | 仅 RUN |
| L2 500 Hz 调度 | TIM6 周期溢出 | TIM6_DAC / 1 | `HAL_TIM_PeriodElapsedCallback` [MD-04] | 500 Hz | 全状态 |
| L3 温度采样 | TIM6 TRGO→规则组 | ADC1_2 / 0 | `HAL_ADC_ConvCpltCallback` [MD-02] | 500 Hz | 全状态 |
| L4 按键切换 | PC13 上升沿 | EXTI15_10 / 4 | `HAL_GPIO_EXTI_Callback` [MD-04] | 异步 | STOP/RUN（FAULT 不响应） |
| L5 串口发送完成 | DMA 完成→UART TC | DMA1_Ch2=2 + LPUART1=3 | `HAL_UART_TxCpltCallback` [MD-05] | 异步 | STOP/RUN；进入 FAULT 前已启动的发送可能继续完成 |

**每条链路作用（已确认结论）：**
- **L1**（RUN）：读注入三通道→`PROC_HandleSamples` 算 CH1 占空比写 CCR1 + 暂存 DAC 幅值→`CTRL_NotifyInjectedComplete` 翻转 LED（10 kHz）。注入组由 `BSP_StartRealtimeChain`/`StopRealtimeChain` 随 RUN 启停。
- **L2**（全状态）：一次 TIM6 周期内三件事——①RUN：`dac_toggle` 翻转写 DAC（250 Hz 方波）；②FAULT：`led_div_counter` 满 25 翻转 LED（10 Hz 闪烁）；③数据源选择并按 500 Hz 周期发起串口发送请求：RUN 发 DAC 幅值、STOP 发温度、FAULT 不发（实际成功帧率受 `dma_busy` 制约，可能低于 500 Hz）。
- **L3**（全状态）：规则组温度换算（3.3V→3.0V 归一化 + 30/130°C 两点插值）→持存并喂 `CTRL_ProcessTemperature` 做异常判断/恢复，驱动状态切换。L3（ADC，优先级0）可抢占 L2（TIM6，优先级1）。
- **L4**（STOP/RUN）：消抖 50ms 后 STOP↔RUN 切换，连带启停实时链与 DAC。优先级最低，存在并发竞态风险点（见 §9）。
- **L5**：两段式——DMA 中断完成搬运→UART TC 中断→`HAL_UART_TxCpltCallback` 清 `dma_busy`。

### 5.2 关键数据流（4 条）

- **DF-1 动态电压→PWM/DAC/串口**：电位器(PA1/IN2)→注入 ch1(Rank1)→`PROC_HandleSamples`：`duty=ch1×8000/4095`→PWM CH1（CCR1，10 kHz 实时）；`dac_amplitude=ch1`（1:1）→L2 翻转写 DAC（250 Hz 方波）+ L2 按 500 Hz 周期发起串口发送幅值 float 的请求。**ch2(IN12)/ch3(IN15) 读入后丢弃，不参与计算**。
- **DF-2 温度→状态机/串口**：内部传感器(IN16)→规则组→换算→持存 `chip_temperature`→①`CTRL_ProcessTemperature` 状态切换；②L2 仅 STOP 态按 500 Hz 周期发起发送温度 float 的请求。
- **DF-3 状态→各链路分支**：`system_state` 为中心分支变量，4 链路读、L4/L3 写。
- **DF-4 串口帧**：`MON_SendData(value)`→8 字节 JustFloat 帧（4 字节 float + 帧尾 `00 00 80 7F`）→`HAL_UART_Transmit_DMA`；`dma_busy` 期间丢弃新帧。

### 5.3 中断优先级（NVIC，PRIORITYGROUP_4 全抢占）

ADC1_2=0 ＞ TIM6_DAC=1 ＞ DMA1_Ch2=2 ＞ LPUART1=3 ＞ EXTI15_10=4。任一 ADC 回调可抢占 TIM6/DMA/UART/EXTI。L1 与 L3 共用 ADC1_2_IRQn，由 HAL 按 ADC 状态标志分发回调，NVIC 层面不互斥抢占、执行先后不固定。

---

## 6. 系统状态

状态枚举 `SYSTEM_STATE_STOP/RUN/FAULT`（MD-04 `app_control.c:15`）。配置常量：CH2=800(10%)、CH3=1600(20%)、消抖 50ms、温度阈值 45°C 进入 / 40°C 恢复、LED 25 分频。

**状态行为：**

| 功能 | STOP | RUN | FAULT |
|---|---|---|---|
| PWM 输出（10 kHz） | 关 | 开（CH1 动态、CH2/3 预设） | 关 |
| 注入采样（10 kHz） | 停 | 启 | 停 |
| 实时处理 | 不执行 | 执行（10 kHz） | 不执行 |
| DAC（250 Hz 方波） | 关 | 开 | 关 |
| 温度采样（500 Hz） | 执行 | 执行 | 执行 |
| 温度异常判断 | 执行 | 执行 | 判断恢复 |
| 状态切换响应 | 响应按键 | 响应按键 | 响应温度恢复 |
| LED | 熄灭 | 翻转~常亮（10 kHz） | 10 Hz 闪烁 |
| 串口（500 Hz 发起发送请求） | 发温度 | 发 DAC 幅值 | 不发送 |

**状态转换（工程实际）：**

```
            按键(L4)               温度>45°C(L3, 任非FAULT态)
   STOP ──────────▶ RUN ──────────────────────▶ FAULT
     ▲                                             │
     │  按键(L4)                                   │ 温度<40°C(L3)
     └───────────────                              ▼
        (RUN→STOP)                               STOP
                                                (FAULT 仅能→STOP, 按键无效)
```

- STOP↔RUN 由按键切换（L4，含 50ms 消抖）；
- 任非FAULT态 + 温度>45°C → FAULT（L3，RUN 时先停链+停 DAC）；
- FAULT + 温度<40°C → STOP（L3，清 LED、清分频计数）；
- FAULT 不响应按键；恢复到 STOP 后需再按键才进 RUN。

---

## 7. 关键共享数据

跨链路（跨 ISR）共享，均已 `volatile`（单链路私有的 `last_button_tick`/`dac_toggle` 未加）。

| 共享数据 | 类型/模块 | 写入方 | 读取方 | 作用与关键关系 |
|---|---|---|---|---|
| `system_state` | volatile enum / MD-04 | L4、L3 | L1/L2/L3/L4 | 中心分支变量；"读-判-写"复合操作在并发下理论存在 TOCTOU（见 §9 N-2） |
| `dac_amplitude` | volatile uint16_t / MD-03 | L1（10 kHz） | L2（500 Hz：DAC 翻转 + 串口幅值） | L2 单周期连续读 2 次，其间 L1 可能更新致两次值不同源（见 §9 N-1） |
| `chip_temperature` | volatile float / MD-02 | L3（500 Hz） | L2（仅 STOP） | 读到的是最近一次已完成的温度值，未必本周期值 |
| `temperature_valid` | volatile bool / MD-02 | L3（首次置 true 不清） | L2 | 单向置位，避免冷启动发未定义温度 |
| `dma_busy` | volatile bool / MD-05 | L2 置位 | L5 清除、L2 读 | L2(优先级1)高于 L5(3)，发送未完时 L2 丢弃新帧 |
| `led_div_counter` | volatile uint8_t / MD-04 | L2（FAULT 递增）、L3（恢复清零） | L2（判满 25） | 影响 10 Hz 闪烁；恢复时由 L3 清零重新分频 |

---

## 8. CubeMX 生成代码与用户代码关系

- **职责划分**：CubeMX 代码负责时钟/外设/GPIO/DMA/NVIC 初始化与底层 IRQ Handler（调 `HAL_xxx_IRQHandler`）；用户代码负责业务状态机、数据处理、调度逻辑、协议组包，并通过复用 HAL 回调挂接业务。HAL 库不修改。
- **划分机制**：CubeMX 用 `USER CODE BEGIN/END` 标记区隔离，标记区外重新生成会覆盖。
- **集成方式**：
  1. **唯一 USER CODE 集成点**：`main.c` 的 `USER CODE BEGIN Includes`（`#include "app_control.h"`）与 `USER CODE BEGIN 2`（调 `APP_Control_Init()`）。
  2. App 全部文件在顶层 CMakeLists 加入构建，不在 CubeMX 管理范围。
  3. 应用还经外设句柄、HAL 接口、HAL 回调覆盖与 CubeMX 代码关联。
- **修改边界**（二次开发）：应用代码写入用户代码区与 App 目录；不修改底层 HAL/CMSIS；修改 `.ioc` 后须以 CubeMX 重新生成代码并妥善保留用户代码区内容；保持原命名/缩进/组织风格。App 目录为二次开发主工作区。

---

## 9. 设计一致性结论

### 9.1 一致内容（已确认）

软件说明书与本工程在主要方面高度一致：
- **引脚**：PA8/PB13、PA9/PB14、PA10/PB15（三组互补 PWM）、PA1/PB1/PB0（注入 IN2/IN12/IN15）、PA4（DAC1_OUT1）、PC13（按键 EXTI13 上升沿）、PA5（LD2）、PA13/PA14（SWD）一致；PA2（LPUART1_TX）一致，PA3（LPUART1_RX）存在差异，见 D-1。
- **外设与时序**：PWM 10 kHz（中心对齐1 + ARR=8000 + PSC=0）、TIM6 500 Hz、DAC 250 Hz 方波、温度 500 Hz、LED 异常 10 Hz 均一致；串口 JustFloat / 115200 bps、按 500 Hz 周期发起发送请求一致（实际成功帧率受 `dma_busy` 制约）TIM1 TRGO2=OC4REF 触发注入组、TIM6 TRGO=UPDATE 触发规则组。
- **功能链路与数据流**：FR-01~FR-09 / SF-01~SF-11 对应 L0~L5 链路与 DF-1~DF-4 数据流一致；CH1 占空比线性映射 `ch1×8000/4095`、DAC 幅值 1:1、ch2/ch3 不参与计算，均与说明书一致。
- **状态机**：三状态、STOP↔RUN 按键切换、温度 45°C/40°C 双阈值独立防抖、FAULT 不响应按键、恢复返 STOP 需再按键，均一致。
- **温度阈值等 TBD 项**：说明书标"详细设计中确定"，工程补全为 45/40，不构成矛盾。

### 9.2 已确认差异

- **D-1：LPUART1_RX（PA3）使能**（来源：`code_structure_analysis.md`）。说明书 5.1 称 PA3"本项目不使用"，但工程 `hlpuart1.Init.Mode = UART_MODE_TX_RX`（[usart.c:47](../Core/Src/usart.c)），RX 通道与 PA3 均配置使能。功能上无接收逻辑、行为仍只发送；但引脚与外设模式与说明书描述不一致。
- **D-2：PA3 板载连接表述（SB23）**（来源：同上）。与 D-1 同源，属 SB23 焊桥层面补充说明。
- **说明类（非冲突，备查）**：①温度换算的 VDDA 归一化（说明书未述细节，工程补充）；②温度阈值由工程补全；③PWM 占空比算法为说明书开放项的具体实现。

### 9.3 需关注事项（调用链/数据流层面）

> 以下来源：`code_flow_analysis.md` §4.5，均为潜在一致性风险或注意点，非已确认功能错误。

- **N-1**：L2 单个 TIM6 周期内连续两次读 `dac_amplitude`（DAC 翻转用、串口发幅值用），其间 L1 可能更新致两者不同源。是否发生取决于中断相对相位与 TIM6 回调执行时间，不能仅据频率比判定；不构成功能错误。
- **N-2**：`system_state` 的"读-判-写"复合操作在 L4(优先级4)/L3(优先级0) 并发下理论可致状态与硬件不一致（如 L4 切换途中被 L3 抢占置 FAULT，L4 返回后仍写 RUN 并启实时链）。理论边界场景，需刻意构造触发，**列入需运行验证**。
- **N-3**：串口发送的 DAC 幅值为 0~4095 原始整数 float（非换算电压）；上位机解析需知此量纲。说明书 SF-11 未定义数值含义，不构成冲突。
- **潜在并发竞态点**：L4 按键 ISR（优先级最低）执行期间可被任意 ADC/TIM6/DMA 中断抢占。

### 9.4 需运行验证的内容（仅保留必要说明）

仅凭源码与现有资料无法确认、需实际运行验证的事项汇总（详见 `code_flow_analysis.md` §5）：PWM 实测波形/死区/占空比、注入组采样时序与线性度、DAC 250 Hz 方波与关闭确认、温度准确度、阈值切换与 LED、串口 JustFloat 解析与量纲、N-2 并发边界场景、DMA 背压丢点、实时链启停顺序与首触发/停输出毛刺。本报告仅整合列举，不展开验证方法。

---

## 10. 关键源码事实索引（二次开发常用）

| 事实 | 代码位置 |
|---|---|
| 用户应用启动集成点 | [main.c:101](../Core/Src/main.c) `APP_Control_Init()` |
| 主循环空转、中断驱动 | [main.c:106](../Core/Src/main.c) |
| 状态枚举 / 阈值常量 / 配置常量 | [app_control.c:15](../App/Src/app_control.c) / [:25-26](../App/Src/app_control.c) / [:22-27](../App/Src/app_control.c) |
| 500 Hz 调度（DAC/LED/串口） | [app_control.c:52](../App/Src/app_control.c) `HAL_TIM_PeriodElapsedCallback` |
| 按键 STOP↔RUN 切换 | [app_control.c:82](../App/Src/app_control.c) `HAL_GPIO_EXTI_Callback` |
| 温度异常判断/恢复 | [app_control.c:123](../App/Src/app_control.c) `CTRL_ProcessTemperature` |
| 注入组读取与转发 | [app_sampling.c:21](../App/Src/app_sampling.c) `HAL_ADCEx_InjectedConvCpltCallback` |
| 温度换算 | [app_sampling.c:44](../App/Src/app_sampling.c) `HAL_ADC_ConvCpltCallback` |
| 占空比 / DAC 幅值计算 | [app_process.c:13](../App/Src/app_process.c) `PROC_HandleSamples` |
| JustFloat 组包 / DMA 发送 | [app_monitor.c:23](../App/Src/app_monitor.c) / [:35](../App/Src/app_monitor.c) |
| 实时链启停（启停 PWM+注入组） | [app_bsp.c:14](../App/Src/app_bsp.c) / [:32](../App/Src/app_bsp.c) |
| TIM1 配置（10 kHz/死区/TRGO2） | [tim.c:31](../Core/Src/tim.c) `MX_TIM1_Init` |
| TIM6 配置（500 Hz） | [tim.c:122](../Core/Src/tim.c) `MX_TIM6_Init` |
| ADC 注入组 / 规则组配置 | [adc.c:91](../Core/Src/adc.c) / [:78](../Core/Src/adc.c) |
| NVIC 优先级设定 | dma.c:48 / adc.c:173 / tim.c:178 / usart.c:128 / gpio.c:70 |
| 中断向量 | [stm32g4xx_it.c](../Core/Src/stm32g4xx_it.c) |
