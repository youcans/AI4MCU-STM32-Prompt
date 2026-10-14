# 工程结构与设计一致性分析

---

## 0. 文档说明

本文档基于 `ADC_Sync_PWM_v2` 工程的 `.ioc` 配置文件、CubeMX 生成代码（Core 目录）、用户应用代码（App 目录）以及软件说明书 [`docs/software_spec_SE.md`](software_spec_SE.md)，对当前工程进行结构解析、模块关系梳理与设计一致性检查。

- 所有代码解析结论以**当前工程实际源码**为准，不根据缺失信息推测。
- 设计一致性检查同时保留「软件说明书描述」与「工程实际实现」两方面事实，不自行修改任何一方。
- 本轮只进行分析，未修改任何程序文件、`.ioc` 文件或软件说明书。

分析涉及的源文件清单：

| 目录 | 文件 | 性质 |
|---|---|---|
| Core/Src | main.c, gpio.c, adc.c, dac.c, tim.c, usart.c, dma.c, stm32g4xx_it.c, stm32g4xx_hal_msp.c, system_stm32g4xx.c | CubeMX 生成（含用户代码区） |
| Core/Inc | main.h, adc.h, dac.h, tim.h, usart.h, dma.h, gpio.h, stm32g4xx_it.h, stm32g4xx_hal_conf.h | CubeMX 生成 |
| App/Inc | app_bsp.h, app_sampling.h, app_process.h, app_control.h, app_monitor.h | 用户应用 |
| App/Src | app_bsp.c, app_sampling.c, app_process.c, app_control.c, app_monitor.c | 用户应用 |
| 根目录 | ADC_Sync_PWM.ioc, CMakeLists.txt | CubeMX 工程配置 / 构建配置 |

---

## 1. 程序入口与启动流程

程序入口位于 [`Core/Src/main.c`](../Core/Src/main.c) 的 `main()` 函数，启动流程为：

1. `HAL_Init()` — HAL 库初始化、SysTick 配置。
2. `SystemClock_Config()` — 配置系统时钟（详见第 5.3 节）。
3. 按顺序初始化外设（CubeMX 生成代码）：
   - `MX_GPIO_Init()` → `MX_DMA_Init()` → `MX_TIM1_Init()` → `MX_ADC1_Init()` → `MX_LPUART1_UART_Init()` → `MX_DAC1_Init()` → `MX_TIM6_Init()`
4. `APP_Control_Init()`（`USER CODE BEGIN 2` 用户代码区）— 用户应用初始化入口。
5. 进入 `while(1)` 空闲主循环（`USER CODE BEGIN 3` 区为空）。

**关键特征：** 主循环为空，所有应用逻辑由中断驱动。系统一旦完成初始化，控制权完全交给中断服务例程（ADC 注入组完成中断、ADC 规则组完成中断、TIM6 周期中断、EXTI 按键中断、LPUART1 DMA 发送完成中断）。

`APP_Control_Init()`（[app_control.c:35](../App/Src/app_control.c)）完成应用层初始化：设置 STOP 状态、写入 CH2/CH3 预设占空比、LED 熄灭、DAC 关闭、启动规则组温度采样、启动 TIM6 的 500 Hz 周期中断。

---

## 2. 工程目录结构与文件组织

### 2.1 目录树

```
ADC_Sync_PWM_v2/
├── ADC_Sync_PWM.ioc              # CubeMX 工程配置
├── CMakeLists.txt                # 顶层构建脚本
├── CMakePresets.json
├── STM32G431XX_FLASH.ld         # 链接脚本
├── startup_stm32g431xx.s        # 启动文件
├── CLAUDE.md
├── cmake/
│   ├── gcc-arm-none-eabi.cmake
│   ├── starm-clang.cmake
│   └── stm32cubemx/CMakeLists.txt   # CubeMX 生成的源/HAL 构建
├── Core/                         # CubeMX 生成代码
│   ├── Inc/                      # 外设头文件 + main.h + hal_conf
│   └── Src/                      # main.c + 外设初始化 + 中断 + MSP + 启动
├── App/                          # 用户应用代码
│   ├── Inc/                      # 5 个模块头文件
│   └── Src/                      # 5 个模块实现
├── Drivers/                      # HAL + CMSIS 库（不修改）
└── docs/                         # 文档
```

### 2.2 构建集成

[`CMakeLists.txt`](../CMakeLists.txt) 顶层脚本通过 `add_subdirectory(cmake/stm32cubemx)` 引入 CubeMX 生成的源文件与 HAL 库，并在用户代码区显式加入 App 模块：

- `target_sources` 加入 5 个 `App/Src/*.c`
- `target_include_directories` 加入 `App/Inc`

App 模块与 CubeMX 生成代码在构建层面隔离清晰：CubeMX 代码位于 [cmake/stm32cubemx/CMakeLists.txt](../cmake/stm32cubemx/CMakeLists.txt)，由 INTERFACE 库 `stm32cubemx` 提供头文件路径与宏（`USE_HAL_DRIVER`、`STM32G431xx`）；App 源文件在顶层 CMakeLists 中声明，链接 `stm32cubemx` 库。

---

## 3. 应用模块划分与功能

App 目录包含 5 个模块（命名规则 `app_<模块>.c/h`，公开接口前缀为模块缩写 `BSP_` / `SAMP_` / `PROC_` / `CTRL_` / `MON_`）。模块编号 MD-01 ~ MD-05 见各文件注释。

### 3.1 模块总览

| 模块 | 文件 | 编号 | 职责 |
|---|---|---|---|
| 板级支持 | [app_bsp.c](../App/Src/app_bsp.c) / [app_bsp.h](../App/Inc/app_bsp.h) | MD-01 | 对 HAL 的薄封装，提供 PWM/DAC/LED/串口/定时器/ADC 启停与写入接口，无业务逻辑 |
| 采样管理 | [app_sampling.c](../App/Src/app_sampling.c) / [app_sampling.h](../App/Inc/app_sampling.h) | MD-02 | ADC 注入组回调（三通道读取并转发）+ 规则组回调（温度换算并转发）；提供温度查询接口 |
| 实时处理 | [app_process.c](../App/Src/app_process.c) / [app_process.h](../App/Inc/app_process.h) | MD-03 | 根据动态电压采样值计算 PWM CH1 占空比与 DAC 方波幅值 |
| 系统控制 | [app_control.c](../App/Src/app_control.c) / [app_control.h](../App/Inc/app_control.h) | MD-04 | STOP/RUN/FAULT 状态机、500 Hz 调度（DAC 翻转/LED 分频/串口数据源选择）、按键处理、温度异常判断 |
| 串口监测 | [app_monitor.c](../App/Src/app_monitor.c) / [app_monitor.h](../App/Inc/app_monitor.h) | MD-05 | JustFloat 帧组包与 LPUART1 DMA 发送管理 |

### 3.2 各模块公开接口

**MD-01 板级支持（`app_bsp.h`）：**

| 接口 | 作用 |
|---|---|
| `BSP_StartRealtimeChain()` | 开启实时采样链：启动 ADC1 注入组中断转换 + 启动 TIM1 三路 PWM 与三路互补输出 |
| `BSP_StopRealtimeChain()` | 停止实时采样链：先清 TIM1 CR1.CEN 切断触发，再停 PWM/互补通道，最后停注入组中断转换 |
| `BSP_WritePwmDutyCh1/2/3(duty)` | 写入 TIM1 CH1/CH2/CH3 的 CCR 占空比 |
| `BSP_WriteDac(value)` | 设置 DAC1 通道 1 输出值 |
| `BSP_StartDac()` / `BSP_StopDac()` | 启动 / 关闭 DAC1 通道 1（关闭前先置 0） |
| `BSP_ToggleLed()` | 翻转 LD2 |
| `BSP_SetLedLow()` | LD2 置低（熄灭） |
| `BSP_StartDmaTx(data,len)` | LPUART1 DMA 发送，返回是否成功 |
| `BSP_StartTim6()` | 启动 TIM6 周期中断 |
| `BSP_StartRegularAdc()` | ADC1 校准 + 启动规则组中断转换（温度采样） |

**MD-02 采样管理（`app_sampling.h`）：**

| 接口 | 作用 |
|---|---|
| `SAMP_GetTemperature()` | 返回最新芯片温度（°C） |
| `SAMP_IsTemperatureValid()` | 返回温度是否已至少完成一次有效采样 |

该模块同时实现两个 HAL 回调（非头文件声明，由 HAL 弱定义覆盖）：`HAL_ADCEx_InjectedConvCpltCallback`（注入组完成）、`HAL_ADC_ConvCpltCallback`（规则组完成）。

**MD-03 实时处理（`app_process.h`）：**

| 接口 | 作用 |
|---|---|
| `PROC_HandleSamples(ch1,ch2,ch3)` | 接收三路注入采样值，用 ch1 计算 PWM CH1 占空比并写入，暂存 DAC 幅值 |
| `PROC_GetDacAmplitude()` | 返回最新 DAC 方波幅值 |

**MD-04 系统控制（`app_control.h`）：**

| 接口 | 作用 |
|---|---|
| `APP_Control_Init()` | 应用初始化（设置 STOP 状态、写 CH2/CH3 预设、LED 熄灭、DAC 关闭、启动温度采样与 TIM6） |
| `CTRL_NotifyInjectedComplete()` | 由 MD-02 调用，运行状态下翻转 LED |
| `CTRL_ProcessTemperature(temp)` | 由 MD-02 调用，温度异常判断与状态切换 |

该模块同时实现两个 HAL 回调：`HAL_TIM_PeriodElapsedCallback`（TIM6 500 Hz 调度）、`HAL_GPIO_EXTI_Callback`（按键）。

**MD-05 串口监测（`app_monitor.h`）：**

| 接口 | 作用 |
|---|---|
| `MON_SendData(value)` | 组 JustFloat 帧并发起 DMA 发送（忙时丢弃并返回 false） |
| `MON_IsBusy()` | 返回 DMA 是否忙 |

该模块实现 `HAL_UART_TxCpltCallback`（LPUART1 发送完成清忙标志）。

---

## 4. 模块调用与依赖关系

### 4.1 调用/依赖图

```
                         main.c (USER CODE)
                            │
                            ▼ 创建/启动
                  APP_Control_Init()  ── MD-04 系统控制
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
   BSP_StartRegularAdc  BSP_StartTim6      BSP_WritePwmDutyCh2/3
   BSP_StartDac(否)     (启动 500Hz)       BSP_SetLedLow / BSP_StopDac
   (MD-01)             (MD-01)            (MD-01)


中断驱动（运行后由硬件/HAL 路由）：

  TIM1_TRGO2 ──触发──▶ ADC1 注入组转换完成
        │                       │
        │                       ▼ HAL_ADCEx_InjectedConvCpltCallback  [MD-02]
        │                  读取 ch1/ch2/ch3
        │                       │
        │              ┌────────┴────────┐
        │              ▼                 ▼
        │      PROC_HandleSamples   CTRL_NotifyInjectedComplete
        │        (MD-03)               (MD-04)
        │       计算 duty            运行态翻转 LED
        │       BSP_WritePwmDutyCh1     (MD-01)
        │       暂存 dac_amplitude        │
        │              │                 │
        │       返回 dac_amplitude ◀─────┘ (由 MD-04 通过 PROC_GetDacAmplitude 读取)
        │
  TIM6 周期(500Hz) ──▶ HAL_TIM_PeriodElapsedCallback  [MD-04]
        │                ① RUN: dac_toggle 翻转 → BSP_WriteDac(PROC_GetDacAmplitude() 或 0)
        │                ② FAULT: led_div_counter 满 25 → BSP_ToggleLed()
        │                ③ RUN: MON_SendData(DAC幅值)；STOP: MON_SendData(温度)
        │
  TIM6_TRGO ──触发──▶ ADC1 规则组(温度)转换完成
        │                       │
        │                       ▼ HAL_ADC_ConvCpltCallback  [MD-02]
        │                  换算温度 → CTRL_ProcessTemperature(temp)
        │                                    (MD-04)
        │                          FAULT 判断 / 恢复 → 状态切换
        │
  PC13 EXTI ──▶ HAL_GPIO_EXTI_Callback  [MD-04]
                       STOP↔RUN 切换 → BSP_Start/StopRealtimeChain + BSP_Start/StopDac
                                       (MD-01)
        │
  LPUART1 DMA Tx 完成 ──▶ HAL_DMA_IRQHandler (DMA1_Ch2 中断)
                              │ DMA 传输结束
                              ▼ 通知 UART
                       (UART TC 中断) HAL_UART_IRQHandler
                              ▼
                       HAL_UART_TxCpltCallback  [MD-05]
                              清 dma_busy
```

### 4.2 依赖方向

- **MD-04 系统控制** 处于核心调度位置，`#include` 了其余全部 4 个模块（bsp、monitor、process、sampling）。除实现 TIM6/EXTI 两个回调外，还在 500 Hz 调度中调用 `SAMP_IsTemperatureValid`/`SAMP_GetTemperature`（向 MD-02 取温度）。
- **MD-02 采样管理** `#include` 了 `app_process.h` 与 `app_control.h`，实现两个 ADC 回调，并在回调中调用 `PROC_HandleSamples`（→MD-03）和 `CTRL_NotifyInjectedComplete`/`CTRL_ProcessTemperature`（→MD-04）转发采样数据与温度。
- **MD-03 实时处理** 仅 `#include` `app_bsp.h`（写 PWM），无对其他应用模块的反向依赖；提供 `PROC_GetDacAmplitude` 供 MD-04 读取。
- **MD-05 串口监测** 仅 `#include` `app_bsp.h`（DMA 发送），实现 Tx 完成回调。
- **MD-01 板级支持** 是最底层，仅 `#include` HAL 外设头文件，不被任何模块反向包含业务逻辑。

模块依赖关系：MD-01（HAL 封装）位于最底层，供 MD-02/MD-03/MD-05 调用。其中 **MD-04 与 MD-02 之间存在双向依赖**：MD-04 调用 MD-02 的温度查询接口（`SAMP_*`），MD-02 又调用 MD-04 的通知/处理接口（`CTRL_*`）。MD-03 为 MD-02 的下游，MD-05 仅由 MD-04 调用。

### 4.3 跨模块数据流（ISR 间共享变量）

为跨 ISR 共享，以下变量声明为 `volatile`：

- `dac_amplitude`（MD-03）：注入组 10 kHz 写入，TIM6 500 Hz 读取。
- `chip_temperature` / `temperature_valid`（MD-02）：规则组 500 Hz 写入，TIM6 500 Hz 读取。
- `dma_busy`（MD-05）：TIM6 500 Hz 置位，LPUART1 ISR 清除。
- `system_state` / `led_div_counter`（MD-04）：多 ISR 共享。

---

## 5. CubeMX 生成代码与用户应用代码的关系

### 5.1 划分原则

工程遵循 CubeMX 标准的 `USER CODE BEGIN/END` 标记机制：

- **CubeMX 生成代码**：标记区之外的内容，由 CubeMX 根据 `.ioc` 重新生成时覆盖。
- **用户应用代码**：
  1. 全部 App 目录文件（完全用户编写，不在 CubeMX 管理范围内）。
  2. CubeMX 文件中 `USER CODE` 区域内的少量集成代码。

### 5.2 集成点（CubeMX 文件中的用户代码区）

经检查，CubeMX 生成文件中的用户代码区集成点**仅在** [main.c](../Core/Src/main.c)：

- `USER CODE BEGIN Includes`：`#include "app_control.h"`
- `USER CODE BEGIN 2`：调用 `APP_Control_Init()`

这是用户应用启动流程接入 `main.c` 的**唯一 USER CODE 集成点**：CubeMX 完成所有外设初始化后，调用 `APP_Control_Init()` 将控制权交给用户应用层。需要强调，此调用并非用户应用与 CubeMX 生成代码之间的唯一联系——应用代码还通过以下途径与 CubeMX 生成代码关联：①直接使用 CubeMX 生成的全局外设句柄（`htim1`、`htim6`、`hadc1`、`hdac1`、`hlpuart1` 等）；②调用 HAL 接口操作这些外设；③通过**HAL 弱定义回调覆盖**挂接业务（应用在 App 目录中重新定义 `HAL_TIM_PeriodElapsedCallback`、`HAL_GPIO_EXTI_Callback`、`HAL_ADCEx_InjectedConvCpltCallback`、`HAL_ADC_ConvCpltCallback`、`HAL_UART_TxCpltCallback` 等回调，由 HAL 在中断中调用）。

### 5.3 二者职责分工

| 职责 | 承担方 |
|---|---|
| 时钟、外设、GPIO、DMA、NVIC 初始化 | CubeMX 生成代码（Core/Src/*.c，按 .ioc 配置） |
| 中断向量与底层 IRQ Handler | CubeMX 生成（stm32g4xx_it.c，调用 HAL_xxx_IRQHandler） |
| HAL 库 | Drivers/（不修改） |
| 业务状态机、数据处理、调度逻辑、协议组包 | 用户应用（App/） |
| 复用 HAL 回调挂接业务 | 用户应用（App/ 中的 `HAL_*Callback`） |

> 说明：`.ioc` 中 `RCC.HSE_VALUE=24000000`（24 MHz）与 NUCLEO-G431RB 板载 HSE 实际值一致；`ProjectManager.HalAssertFull=false`。时钟配置：HSE 24 MHz → PLLM=/3 → 8 MHz → PLLN=×40 → 320 MHz → PLLR=/2 → **SYSCLK 160 MHz**，APB1/APB2 不分频，TIM 时钟 = 160 MHz。此为后续时序验证的基础。

---

## 6. 设计一致性检查

本章将说明书的功能、输入输出对象、引脚配置、模块实现与工程实际逐一比对。每一项给出「说明书描述」与「工程实现」两方事实及一致性结论。

### 6.1 MCU 引脚配置一致性

| 引脚 | 说明书（5.1） | 工程实现（.ioc / MspInit） | 一致性 |
|---|---|---|---|
| PA8 | TIM1_CH1，PWM 第一组主通道 | `S_TIM1_CH1`，AF6，AF 推挽 | 一致 |
| PB13 | TIM1_CH1N，互补 | `TIM1_CH1N`，AF6，AF 推挽 | 一致 |
| PA9 | TIM1_CH2，第二组主通道 | `S_TIM1_CH2`，AF6 | 一致 |
| PB14 | TIM1_CH2N，互补 | `TIM1_CH2N`，AF6 | 一致 |
| PA10 | TIM1_CH3，第三组主通道 | `S_TIM1_CH3`，AF6 | 一致 |
| PB15 | TIM1_CH3N，互补 | `TIM1_CH3N`，AF4 | 一致 |
| PA1 | ADC1_IN2，注入 Rank1，模拟输入 | `ADC1_IN2`，IN2-Single-Ended，模拟 | 一致 |
| PB1 | ADC1_IN12，注入 Rank2，模拟输入 | `ADC1_IN12`，IN12-Single-Ended，模拟 | 一致 |
| PB0 | ADC1_IN15，注入 Rank3，模拟输入 | `ADC1_IN15`，IN15-Single-Ended，模拟 | 一致 |
| PA4 | DAC1_OUT1，模拟输出 | `DAC1_OUT1`，模拟 | 一致 |
| PA2 | LPUART1_TX，AF 推挽 | `LPUART1_TX`，AF12 | 一致 |
| PA3 | LPUART1_RX，AF 输入（不使用） | `LPUART1_RX`，AF12（Mode 配为 Asynchronous，代码 `UART_MODE_TX_RX`） | 一致（见 6.7 差异 D-1） |
| PC13 | GPIO_EXTI13，上升沿，外部下拉按键 | `GPXTI13`，`GPIO_MODE_IT_RISING`，NoPull | 一致（外部下拉由板载电路提供，MCU 侧 NoPull 兼容） |
| PA5 | GPIO_Output，LD2，推挽，初始低 | `GPIO_Output`，GPIO_Label=LD2，输出 PP，初始 RESET | 一致 |
| PA13/PA14 | SWD（SWDIO/SWCLK） | `SYS_JTMS-SWDIO` / `SYS_JTCK-SWCLK`，Serial_Wire | 一致 |

**引脚总体一致。** MCU 侧 PC13 的 NoPull 与说明书"外部下拉"描述兼容（外部下拉由板载按键电路提供），不构成差异。实质差异仅 PA3 一项，见 6.7。

### 6.2 外设与时序配置一致性

#### 6.2.1 PWM（FR-01 / SF-02）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 三组带死区互补 PWM | 是 | TIM1 CH1/CH1N、CH2/CH2N、CH3/CH3N，`HAL_TIM_PWM_Start` + `HAL_TIMEx_PWMN_Start` | 一致 |
| 频率 10 kHz | 10 kHz | 中心对齐1 + ARR=8000 + PSC=0：160M/(8000×2)=10 kHz | 一致 |
| 死区 | "带死区" | `DeadTime=32`（约 200 ns @160 MHz DT 时钟） | 一致（数值说明书未指明，工程实现含死区） |
| CH2/CH3 预设值 | 预设固定值 | 启动时 `BSP_WritePwmDutyCh2(800)`/`Ch3(1600)`，注释为 10%/20%（ARR=8000） | 一致 |
| 周期事件触发 ADC | 产生 ADC 采样触发事件 | TIM1 TRGO2 = OC4REF，CH4 Pulse=4000（中心对齐周期中点），路由到 ADC 注入组外部触发 | 一致 |

> 工程用 OC4REF（CH4 无输出通道）经 TRGO2 在周期中点触发注入组采样，与说明书"PWM 周期事件触发三通道采样"一致。

#### 6.2.2 ADC（FR-02 / SF-03 / SF-04）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 三通道同步采样 | RUN 状态由 PwmPeriodEvent 触发 | 注入组 3 通道（IN2/IN12/IN15 = Rank1/2/3），外部触发 T1_TRGO2 上升沿 | 一致 |
| 触发源 | TIM1 周期事件 | `ADC_EXTERNALTRIGINJEC_T1_TRGO2` | 一致 |
| 一路动态电压 + 两路参考 | AdcDynamicVoltage(IN2) + HighRef(IN12) + LowRef(IN15) | Rank 映射一致；`PROC_HandleSamples` 仅用 ch1=Rank1=IN2，ch2/ch3 注释预留 | 一致 |
| 启停控制 | RUN 启动、STOP/FAULT 停止 | `BSP_StartRealtimeChain`/`StopRealtimeChain` 调 `HAL_ADCEx_InjectedStart/Stop_IT` | 一致 |

#### 6.2.3 实时数据处理（FR-03 / SF-05）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 触发 | AdcSequenceCompleteEvent（仅 RUN） | 注入组完成回调内执行（注入组仅在 RUN 启动） | 一致 |
| 输入 | AdcDynamicVoltage | ch1 = 注入 Rank1 = IN2 | 一致 |
| 输出 PWM CH1 | PwmDutyCh1 | `duty = ch1*8000/4095`，`BSP_WritePwmDutyCh1` | 一致（算法说明书未限定，工程用线性映射至 0~8000） |
| 输出 DAC 幅值 | DacOutputValue | `dac_amplitude = ch1`（1:1） | 一致 |

#### 6.2.4 DAC 方波（FR-04 / SF-06）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 250 Hz 方波 | 250 Hz | TIM6 500 Hz 在 RUN 态翻转 `dac_toggle` → 250 Hz 方波 | 一致 |
| 幅值随 AdcDynamicVoltage 更新 | 是 | 翻转时读 `PROC_GetDacAmplitude()`，0 与幅值之间翻转 | 一致 |
| 启停 | RUN 开 / STOP/FAULT 关 | 按键/异常处 `BSP_StartDac`/`BSP_StopDac`，TIM6 仅 RUN 写 DAC | 一致 |
| 触发源 | Timer500HzEvent | TIM6 周期中断 | 一致 |

#### 6.2.5 芯片温度（FR-05 / SF-07）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 内部温度传感器 500 Hz | ADC1_IN16，500 Hz，全状态 | 规则组 `ADC_CHANNEL_TEMPSENSOR_ADC1`，外部触发 T6_TRGO，STOP 即启动 | 一致 |
| 触发源 | Timer500HzEvent | `ADC_EXTERNALTRIG_T6_TRGO` | 一致 |
| 全状态运行 | 是 | `APP_Control_Init` 即 `BSP_StartRegularAdc`，不随状态启停 | 一致 |

> 工程在温度换算中做了 VDDA 归一化：工厂校准值在 3.0V 下测得，本工程 VREF≈3.3V，代码以 `raw*3300/3000` 归一化后再用 30°C/130°C 两点线性插值。说明书未提及该换算细节，工程实现属合理补充，不构成冲突。

#### 6.2.6 温度异常处理（FR-06 / SF-08 / SF-09）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 异常阈值/恢复阈值 | 说明书 3.1 标注"详细设计中确定" | 工程已确定：`TEMPERATURE_FAULT_THRESHOLD=45.0`、`RECOVERY=40.0` | 一致（工程补全了 TBD 值，说明书称待确定，二者不矛盾） |
| 独立双阈值防抖 | 两阈值独立 | 45 进入、40 恢复，独立值 | 一致 |
| 异常进入 | 自动进入，停止 PWM/DAC/采样 | `CTRL_ProcessTemperature`：>45 时若 RUN 先停链与 DAC，置 FAULT | 一致 |
| 恢复 | 返回 STOP | <40 时 FAULT→STOP，清 LED 与分频计数 | 一致 |
| 异常态按键不响应 | 是 | `HAL_GPIO_EXTI_Callback` 中 FAULT 直接 return | 一致 |
| 返回 STOP 后需再按键进 RUN | 是 | FAULT 恢复只到 STOP，不自动进 RUN | 一致 |

#### 6.2.7 LED 指示（FR-08 / SF-10）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| 停止熄灭 | 是 | 进入 STOP 调 `BSP_SetLedLow` | 一致 |
| 运行翻转（~常亮） | 每次 AdcSequenceCompleteEvent 翻转 | `CTRL_NotifyInjectedComplete` RUN 态 `BSP_ToggleLed`（10 kHz 翻转） | 一致 |
| 异常 10 Hz 闪烁 | 10 Hz（Timer500Hz 分频） | FAULT 态 TIM6 内 `led_div_counter` 满 25 翻转：500/25=20 次/s=10 Hz | 一致 |

#### 6.2.8 串口发送（FR-09 / SF-11）

| 项 | 说明书 | 工程实现 | 一致性 |
|---|---|---|---|
| JustFloat / 115200 / 500 Hz | 是 | LPUART1 115200，TIM6 500 Hz 调用 `MON_SendData`，4 字节 float + 4 字节帧尾(00 00 80 7F) | 一致 |
| 波特率 | 115200 bit/s | `BaudRate=115200` | 一致 |
| RUN 发 DAC 值 | 是 | `MON_SendData((float)PROC_GetDacAmplitude())` | 一致 |
| STOP 发温度 | 是 | `MON_SendData(SAMP_GetTemperature())`（需 `temperature_valid`） | 一致 |
| 异常不发送 | 是 | FAULT 态 TIM6 不调用 `MON_SendData` | 一致 |

### 6.3 状态机一致性（FR-07 / SF-09）

工程状态枚举 `SYSTEM_STATE_STOP/RUN/FAULT` 与说明书"停止/运行/异常"三状态完全对应。状态切换规则比对：

| 规则 | 说明书 | 工程 | 一致性 |
|---|---|---|---|
| STOP↔RUN 按键切换 | 是 | EXTI 回调实现，含 50 ms 消抖 | 一致（消抖为工程实现细节，说明书未禁止） |
| 进入 RUN 同时启 PWM+采样+DAC | 是 | `BSP_StartRealtimeChain` + `BSP_StartDac` | 一致 |
| 离开 RUN 停并 LED 熄灭 | 是 | `BSP_StopRealtimeChain` + `BSP_StopDac` + `BSP_SetLedLow` | 一致 |
| 温度>阈值→FAULT | 是 | `CTRL_ProcessTemperature` | 一致 |
| 温度<恢复→STOP | 是 | 同上 | 一致 |
| FAULT 不响应按键 | 是 | 回调 return | 一致 |

### 6.4 输入输出对象一致性小结

说明书 3.1/3.2 列出的全部输入输出对象，均能在工程中找到对应实现：

- 输入事件：PwmPeriodEvent→TIM1_TRGO2→注入组触发；AdcSequenceCompleteEvent→注入组完成回调；UserButtonEvent→EXTI13；Timer500HzEvent→TIM6 周期。全部一致。
- 输入数据：四路采样值（IN2/IN12/IN15/IN16）映射一致。
- 输入参数：温度阈值工程已赋值（说明书标 TBD）。
- 输出控制：PWM 占空比/启停、DAC 值/启停、LED、采样启停，全部一致。
- 输出数据：串口 JustFloat、系统状态，一致。
- 输出事件：TemperatureFaultEnter/Exit 由 `CTRL_ProcessTemperature` 的状态切换动作隐含产生（驱动 SF-09），一致。

### 6.5 时序参数一致性

| 信号 | 说明书 | 工程 | 一致性 |
|---|---|---|---|
| PWM | 10 kHz | 160M/(8000×2)=10 kHz | 一致 |
| ADC 注入采样 | 10 kHz | 10 kHz PWM 触发 | 一致 |
| 实时处理 | 10 kHz | 注入完成回调 10 kHz | 一致 |
| TIM6 / 500 Hz 事件 | 500 Hz | 160M/(159+1)/(1999+1)=500 Hz | 一致 |
| DAC 方波 | 250 Hz | 500 Hz 翻转=250 Hz | 一致 |
| 温度采样 | 500 Hz | TIM6 触发规则组 500 Hz | 一致 |
| 串口 | 500 Hz | TIM6 内调用 500 Hz | 一致 |
| LED 异常闪烁 | 10 Hz | 500/25/2=10 Hz | 一致 |
| 串口波特率 | 115200 | 115200 | 一致 |

### 6.6 总体一致性结论

**说明书与工程在功能目标、输入输出对象、MCU 引脚配置、外设配置、时序参数、状态机行为、模块实现等主要方面高度一致。** 说明书标注为"待确定"的少量参数（温度阈值），工程已补全具体数值，二者不构成矛盾。发现的可确认差异列于 6.7。

### 6.7 差异项

> 以下仅列出确认存在的不一致或说明书未涵盖而工程已实现的事项，双方事实并列，不做修改。

#### 差异 D-1：LPUART1_RX（PA3）的使能

- **说明书描述（5.1）**：PA3 LPUART1_RX「串口数据接收……**本项目不使用**」。
- **工程实现**（[usart.c:47](../Core/Src/usart.c)）：`hlpuart1.Init.Mode = UART_MODE_TX_RX`，即 RX 通道与 GPIO（PA3 AF12）均被配置使能。
- **一致性**：**存在差异。** 说明书声明 RX 不使用，而工程实际将 LPUART1 配为收发双模并初始化了 PA3。功能上工程未启用任何接收处理逻辑（无接收回调、主循环不读 RX），故行为上仍只发送；但引脚与外设模式下工程与说明书描述不一致。

#### 差异 D-2：PA3 板载连接表述（SB23）

- **说明书描述（5.1）**：PA3「通过板载 SB23 连接 STLINK-V3E 虚拟串口，本项目不使用」。
- **工程实现**：PA3 已配置为 LPUART1_RX（见 D-1）。
- **一致性**：与 D-1 同源。此为板载 SB23 焊桥状态层面的潜在影响——若按说明书"不使用"，理论上可不配置 PA3；工程选择配置。属同一差异的补充说明。

#### 说明类（非冲突，列出备查）

- **温度换算的 VDDA 归一化**：说明书（FR-05）只说"换算为温度值"，工程额外实现 3.3V→3.0V 归一化与 30/130°C 两点插值。工程为说明书未涵盖的实现细节，不冲突。
- **温度阈值赋值**：说明书 3.1 中两个温度阈值标注"详细设计中确定"；工程已确定为 45°C/40°C。工程补全了说明书的 TBD，二者不矛盾。
- **PWM 占空比算法**：说明书 SF-05 未限定具体公式；工程采用 `ch1×8000/4095`（CH1）与 `dac_amplitude = ch1`（DAC 1:1）。属工程对说明书开放项的具体实现，不冲突。

---

## 7. 附：关键源码事实索引

| 事实 | 代码位置 |
|---|---|
| 主循环为空、中断驱动 | [main.c:106](../Core/Src/main.c) |
| 用户应用唯一入口 | [main.c:101](../Core/Src/main.c) `APP_Control_Init()` |
| 系统状态枚举 | [app_control.c:15](../App/Src/app_control.c) |
| 温度阈值常量 | [app_control.c:25-26](../App/Src/app_control.c) |
| STOP/RUN 按键切换 | [app_control.c:82](../App/Src/app_control.c) `HAL_GPIO_EXTI_Callback` |
| 温度异常判断与恢复 | [app_control.c:123](../App/Src/app_control.c) `CTRL_ProcessTemperature` |
| TIM6 500 Hz 调度 | [app_control.c:52](../App/Src/app_control.c) `HAL_TIM_PeriodElapsedCallback` |
| 注入组读取与转发 | [app_sampling.c:21](../App/Src/app_sampling.c) `HAL_ADCEx_InjectedConvCpltCallback` |
| 温度换算 | [app_sampling.c:44](../App/Src/app_sampling.c) `HAL_ADC_ConvCpltCallback` |
| 占空比与 DAC 幅值计算 | [app_process.c:13](../App/Src/app_process.c) `PROC_HandleSamples` |
| JustFloat 组包 | [app_monitor.c:23](../App/Src/app_monitor.c) |
| 实时链启停 | [app_bsp.c:14](../App/Src/app_bsp.c) / [:32](../App/Src/app_bsp.c) |
| TIM1 配置（10 kHz/死区/TRGO2） | [tim.c:31](../Core/Src/tim.c) `MX_TIM1_Init` |
| TIM6 配置（500 Hz） | [tim.c:122](../Core/Src/tim.c) `MX_TIM6_Init` |
| ADC 注入组配置 | [adc.c:91-125](../Core/Src/adc.c) |
| DAC 配置（PA4） | [dac.c:30](../Core/Src/dac.c) |
| LPUART1（115200/双模/PA2-PA3/DMA） | [usart.c:32](../Core/Src/usart.c) |
| GPIO（PC13 EXTI / PA5 LD2） | [gpio.c:42](../Core/Src/gpio.c) |
| NVIC 优先级 | [dma.c:39](../Core/Src/dma.c) / [adc.c:173](../Core/Src/adc.c) / [tim.c:178](../Core/Src/tim.c) / [usart.c:128](../Core/Src/usart.c) / [gpio.c:70](../Core/Src/gpio.c) |
| 中断向量 | [stm32g4xx_it.c](../Core/Src/stm32g4xx_it.c) |

NVIC 优先级（抢占,子优先级，PriorityGroup=4 即全 4 位抢占）实测：ADC1_2=0、TIM6_DAC=1、DMA1_Ch2=2、LPUART1=3、EXTI15_10=4。即采样中断最高、500Hz 次之、DMA/串口随后、按键最低。
