# 工程结构与模块关系解析

## 1. 程序概述

- **项目名称**：ADC_Sync_PWM（基于 ADC 同步采样的 PWM 占空比控制）
- **硬件平台**：NUCLEO-G431RB（STM32G431RBTx，Cortex-M4，128 KB Flash，32 KB RAM）
- **外部晶振**：24 MHz (HSE)，经 PLL 倍频至系统时钟 160 MHz (PLLN=40, PLLM=/3, PLLP=/2)
- **构建系统**：CMake + GCC工具链，STM32CubeMX 生成的 HAL 驱动框架

---

## 2. 目录结构与分工

| 目录 | 说明 | 属性 |
|---|---|---|
| `Core/Src/` `Core/Inc/` | 外设初始化代码 | **CubeMX 生成**（内嵌用户代码段） |
| `App/Src/` `App/Inc/` | 应用层代码（5 个模块） | **用户手写** |
| `Drivers/CMSIS/` | Cortex-M4 内核头文件 | ST 官方提供 |
| `Drivers/STM32G4xx_HAL_Driver/` | HAL 库源码 | ST 官方提供 |
| `startup_stm32g431xx.s` | 启动汇编（中断向量表） | CubeMX 生成 |
| `STM32G431XX_FLASH.ld` | 链接脚本 | CubeMX 生成 |
| `ADC_Sync_PWM.ioc` | CubeMX 工程文件 | CubeMX 项目配置 |
| `CMakeLists.txt` | 顶层 CMake 构建 | CubeMX 生成（用户可在指定区域添加源码/头文件路径） |
| `cmake/stm32cubemx/CMakeLists.txt` | 下层 CMake（构建 `stm32cubemx` 静态库） | CubeMX 生成 |

---

## 3. 程序入口与初始化流程

启动文件 `startup_stm32g431xx.s` → `Reset_Handler` → `SystemInit()`（`Core/Src/system_stm32g4xx.c`）→ `main()`。

**`main()` 位于 [Core/Src/main.c:69](Core/Src/main.c#L69)**，执行流程如下：

1. `HAL_Init()` — 复位所有外设，初始化 Flash 接口和 Systick
2. `SystemClock_Config()` — 配置 PLL，HSE 24 MHz → SYSCLK 160 MHz, HCLK/APB1/APB2 均为 160 MHz，Flash 延迟 4 等待周期
3. `MX_GPIO_Init()` — 初始化 User Button (PC13, EXTI) 和 LD2 LED (PA5, 推挽输出)
4. `MX_DMA_Init()` — 使能 DMA1 和 DMAMUX1 时钟，配置 DMA1_Channel2 中断（优先级2）
5. `MX_TIM1_Init()` — 初始化高级定时器 TIM1 为 PWM 模式（中心对齐）
6. `MX_ADC1_Init()` — 初始化 ADC1（规则组 + 注入组通道）
7. `MX_LPUART1_UART_Init()` — 初始化 LPUART1 (115200-8-N-1) + DMA TX
8. `MX_DAC1_Init()` — 初始化 DAC1 通道1
9. `MX_TIM6_Init()` — 初始化基本定时器 TIM6（500 Hz 更新率）
10. `HAL_ADCEx_Calibration_Start()` — ADC 自校准
11. `APP_Control_Init()` — 应用层初始化
12. `while (1) {}` — 主循环为空，所有逻辑由中断驱动

---

## 4. 外设初始化代码（CubeMX 生成）的详细配置

### TIM1 — 高级PWM定时器 ([Core/Src/tim.c:31](Core/Src/tim.c#L31))

- 模式：中心对齐模式1 (CenterAligned1)
- 预分频器：0，周期：8000 → 产生的 PWM 频率约为 **10 kHz** (160 MHz / 1 / 8000 / 2)
- 4 通道 PWM1 输出，CH1/CH2/CH3 初始占空比 0%，CH4 初始占空比 50% (Pulse=4000)
- 死区时间：32 个 clock tick
- 主/从输出触发：TRGO = Reset, **TRGO2 = OC4REF** — TRGO2 用作 ADC1 注入组的触发源
- GPIO 输出：CH1(PA8)/CH1N(PB13)/CH2(PA9)/CH2N(PB14)/CH3(PA10)/CH3N(PB15)

### ADC1 — 规则组 + 注入组 ([Core/Src/adc.c:30](Core/Src/adc.c#L30))

- 时钟：SYSCLK/4 = 40 MHz，12 位分辨率，右对齐
- 扫描模式使能，非连续转换
- **规则组**：1 个通道 — **CH18 (内部温度传感器)**，采样时间 247.5 ADC周期
  - 触发源：**TIM6 TRGO**（500 Hz 定时触发）
- **注入组**：3 个通道，由外部触发启动：
  - Injected Rank 1：**CH2 (PA1, ADC1_IN2)**
  - Injected Rank 2：**CH12 (PB1, ADC1_IN12)**
  - Injected Rank 3：**CH15 (PB0, ADC1_IN15)**
  - 触发源：**TIM1 TRGO2** (TIM1 CH4 OC4REF，即 PWM 周期中点)，上升沿触发
- 中断：ADC1_2_IRQn（优先级 0，最高）

### TIM6 — 基本定时器 ([Core/Src/tim.c:122](Core/Src/tim.c#L122))

- 预分频器：159，周期：1999 → **500 Hz 更新率** (160 MHz / (160 × 2000))
- TRGO = Update，触发 ADC1 规则组转换（温度传感器采样）
- 中断：TIM6_DAC_IRQn（优先级 1）

### DAC1 — 数模转换 ([Core/Src/dac.c:30](Core/Src/dac.c#L30))

- 通道1 (PA4)，高频自动模式，无触发（软件写入），外部输出，输出缓冲使能
- 中断与 TIM6 共享 TIM6_DAC_IRQn

### LPUART1 — 串口 ([Core/Src/usart.c:32](Core/Src/usart.c#L32))

- 波特率 115200，8N1，TX(PA2)/RX(PA3)
- DMA TX 通道：DMA1_Channel2 (Memory→Peripheral, Byte对齐, Normal模式)
- 中断：LPUART1_IRQn (优先级 3)，DMA1_Channel2_IRQn (优先级 2)

### 中断优先级总结

| 优先级 | 中断源 |
|--------|--------|
| 0 (最高) | ADC1_2 (注入组+规则组转换完成) |
| 1 | TIM6_DAC (TIM6 周期 & DAC 欠载) |
| 2 | DMA1_Channel2 (LPUART TX 完成) |
| 3 | LPUART1 (串口事件) |
| 4 (最低) | EXTI15_10 (User Button) |

---

## 5. 用户应用模块解析 (`App/` 目录)

用户自行编写了 5 个模块，以 `app_control` 为调度中枢，其他模块各司其职。

### 5.1 `app_control` — 系统控制中枢

| 文件 | [App/Inc/app_control.h](App/Inc/app_control.h) | [App/Src/app_control.c](App/Src/app_control.c) |
|---|---|---|

**系统状态机**（三态）：

- `SYSTEM_STATE_STOP` — 停止状态，仅温度监控运行
- `SYSTEM_STATE_RUN` — 运行状态，PWM/ADC/DAC 全速运行
- `SYSTEM_STATE_FAULT` — 异常状态，过温保护触发

**公开接口**：

- `APP_Control_Init()` — 系统初始化：设置 PWM CH2/CH3 预设值、清除 DAC、启动规则组 ADC 和 TIM6
- `CTRL_NotifyInjectedComplete()` — 注入组 ADC 完成通知（运行态下翻转 LED）
- `CTRL_ProcessTemperature(float)` — 温度处理：超过 45°C 进入 FAULT 态，低于 43°C 恢复 STOP 态

**回调注册**（在 `app_control.c` 中覆盖 HAL 弱回调）：

- `HAL_TIM_PeriodElapsedCallback()` — TIM6 周期回调 (500 Hz)：
  - RUN 态：DAC 方波翻转（250 Hz）
  - FAULT 态：LED 分频闪烁（10 Hz）
  - RUN 态：发送 DAC 幅值到串口
  - STOP 态：发送芯片温度到串口
- `HAL_GPIO_EXTI_Callback()` — B1 按键回调（消抖 50 ms）：
  - STOP→RUN：启动实时链路 + DAC
  - RUN→STOP：停止实时链路 + DAC，关闭 LED

### 5.2 `app_sampling` — ADC 采样处理

| 文件 | [App/Inc/app_sampling.h](App/Inc/app_sampling.h) | [App/Src/app_sampling.c](App/Src/app_sampling.c) |
|---|---|---|

**公开接口**：

- `SAMP_GetTemperature()` — 返回芯片温度 (°C)
- `SAMP_IsTemperatureValid()` — 返回温度数据是否有效

**回调注册**：

- `HAL_ADC_ConvCpltCallback()` — 规则组 ADC 完成（温度传感器，500 Hz 触发）：
  - 读取 ADC 原始值，使用 ST 出厂校准值（地址 `0x1FFF75A8` 和 `0x1FFF75CA`）通过两点插值计算温度
  - 调用 `CTRL_ProcessTemperature()` 进行温度判定
- `HAL_ADCEx_InjectedConvCpltCallback()` — 注入组 ADC 完成（3 路外部模拟输入，PWM 周期中点触发）：
  - 读取 CH1/CH2/CH3 三路注入通道值
  - 调用 `PROC_HandleSamples()` 处理采样结果
  - 调用 `CTRL_NotifyInjectedComplete()` 通知控制模块

### 5.3 `app_process` — PWM 占空比与 DAC 计算

| 文件 | [App/Inc/app_process.h](App/Inc/app_process.h) | [App/Src/app_process.c](App/Src/app_process.c) |
|---|---|---|

**公开接口**：

- `PROC_HandleSamples(uint16_t ch1, ch2, ch3)` — 处理三个注入通道的采样值：
  - ch2、ch3 当前预留，仅使用 ch1
  - ch1 线性映射到 PWM 占空比：`duty = ch1 * 8000 / 4095`，写入 TIM1 CH1
  - ch1 原始值直接作为 DAC 方波幅值暂存
- `PROC_GetDacAmplitude()` — 返回暂存的 DAC 幅值（供 `app_control` 在 TIM6 回调中用于方波翻转）

### 5.4 `app_monitor` — 串口数据发送

| 文件 | [App/Inc/app_monitor.h](App/Inc/app_monitor.h) | [App/Src/app_monitor.c](App/Src/app_monitor.c) |
|---|---|---|

**公开接口**：

- `MON_SendData(float value)` — 发送 float 数据，使用 **JustFloat 协议**封装为 8 字节帧（4 字节 float + `0x00 0x00 0x80 0x7F` 帧尾 IEEE 754 正无穷标记），DMA TX 非阻塞发送
- `MON_IsBusy()` — 返回 DMA 是否正在发送

**回调注册**：

- `HAL_UART_TxCpltCallback()` — DMA 传输完成，清除 busy 标志

### 5.5 `app_bsp` — 硬件抽象层

| 文件 | [App/Inc/app_bsp.h](App/Inc/app_bsp.h) | [App/Src/app_bsp.c](App/Src/app_bsp.c) |
|---|---|---|

**公开接口**（15 个函数，封装所有 HAL 调用）：

| 函数 | 说明 |
|---|---|
| `BSP_WritePwmDutyCh1/2/3()` | 设置 TIM1 三路 PWM 占空比 (`__HAL_TIM_SET_COMPARE`) |
| `BSP_WriteDac()` | 设置 DAC 输出电压 |
| `BSP_StartDac()/StopDac()` | DAC 启停 |
| `BSP_ToggleLed()/SetLedLow()` | LED 翻转/拉低 |
| `BSP_StartDmaTx()` | DMA 方式发送 UART 数据 |
| `BSP_StartTim6()` | 启动 TIM6 中断 |
| `BSP_StartRegularAdc()` | 启动 ADC 规则组中断 |
| `BSP_StartRealtimeChain()` — 启动实时链路：先启动ADC注入组中断 → 启动TIM1 CH1/CH2/CH3 三路互补PWM输出 |
| `BSP_StopRealtimeChain()` — 停止实时链路：先通过 `CLEAR_BIT(CR1, CEN)` 立即停止TIM1计数器 → 逐通道停止PWM → 停止ADC注入组 |

---

## 6. 模块间调用依赖关系

```
                    ┌─────────────────────┐
                    │     main.c          │
                    │  (CubeMX 模板)      │
                    └───────┬─────────────┘
                            │ APP_Control_Init()
                            ▼
              ┌─────────────────────────────┐
              │       app_control           │ ◄── 系统控制中枢
              │  (状态机 + HAL回调覆盖)      │
              └──┬───────┬───────┬──────┬───┘
                 │       │       │      │
        ┌────────▼┐ ┌───▼────┐ ┌▼────┐ │  ┌──────────┐
        │ app_    │ │ app_   │ │app_ │ │  │ app_     │
        │ sampling│ │process │ │monitor│ │  │ bsp     │ ◄── HAL 封装层
        └───┬─────┘ └───┬────┘ └──┬───┘ │  └────┬─────┘
            │           │        │      │       │
            │ 回调      │        │      │       │
            ▼           ▼        ▼      ▼       ▼
   ┌─────────────────────────────────────────────────┐
   │  HAL 外设句柄 (hadc1, htim1, htim6, hdac1, hlpuart1)  │
   └─────────────────────────────────────────────────┘
```

**依赖关系细化**：

| 调用者 | 被调用者 | 调用方式 | 说明 |
|---|---|---|---|
| `app_control` | `app_bsp` | 直接函数调用 | 所有外设操作通过 BSP 层 |
| `app_control` | `app_monitor` | `MON_SendData()` | 串口发送（RUN态发DAC幅值，STOP态发温度） |
| `app_control` | `app_process` | `PROC_GetDacAmplitude()` | TIM6 回调中获取 DAC 方波幅值 |
| `app_control` | `app_sampling` | `SAMP_GetTemperature()`, `SAMP_IsTemperatureValid()` | 串口数据源选择时获取温度 |
| `app_sampling` | `app_control` | `CTRL_ProcessTemperature()`, `CTRL_NotifyInjectedComplete()` | 回调链调用 |
| `app_sampling` | `app_process` | `PROC_HandleSamples()` | 注入组数据传递给处理模块 |

`app_bsp` 是唯一的 HAL 封装层，所有其他应用模块不直接调用 HAL 函数（除 `app_monitor` 和 `app_sampling` 中必要的句柄引用外）。

---

## 7. 实时运行机制

系统以两个定时器触发链路驱动：

### 链路一：PWM 同步采样链（高速 ~10 kHz）

```
TIM1 CH4 OC4REF (周期中点) → ADC1 注入组触发 → 3路模拟采样
  → HAL_ADCEx_InjectedConvCpltCallback()
  → PROC_HandleSamples(ch1, ch2, ch3)
  → BSP_WritePwmDutyCh1(duty)  [PWM 占空比闭环调节]
  → CTRL_NotifyInjectedComplete()  [LED 翻转指示]
```

### 链路二：温度监控链（500 Hz）

```
TIM6 更新事件 → ADC1 规则组触发 → 内部温度传感器采样
  → HAL_ADC_ConvCpltCallback()
  → 温度计算 (ST 出厂校准值)
  → CTRL_ProcessTemperature()  [过温保护判定]
```

### 链路三：系统心跳（500 Hz）

```
TIM6 更新中断 → HAL_TIM_PeriodElapsedCallback()
  → RUN 态: DAC 500 Hz 方波 (幅值来自注入通道 ch1)
  → RUN 态: 串口发送 DAC 幅值
  → STOP 态: 串口发送芯片温度
  → FAULT 态: LED 10 Hz 闪烁
```

---

## 8. CubeMX 生成代码与用户应用代码的关系

### CubeMX 生成部分（`.ioc` 文件配置驱动）

| 文件 | 角色 |
|---|---|
| `Core/Src/main.c` | 模板框架，提供 `/* USER CODE BEGIN */` 标记供插入用户代码 |
| `Core/Src/adc.c`, `tim.c`, `dac.c`, `gpio.c`, `dma.c`, `usart.c` | 外设 `MX_xxx_Init()` 初始化 + `MspInit/MspDeInit`，含空 `USER CODE` 段 |
| `Core/Src/stm32g4xx_it.c` | 中断服务函数骨架，调用 HAL 标准 IRQHandler |
| `Core/Src/stm32g4xx_hal_msp.c` | 全局 MSP 初始化 |
| `Core/Inc/stm32g4xx_hal_conf.h` | HAL 模块开关，使能了 ADC/DAC/TIM/UART/GPIO/EXTI/DMA/RCC/FLASH/PWR/CORTEX |
| `Drivers/` | ST 官方 HAL 库与 CMSIS（不变的部分） |

### 用户应用部分（`App/` 目录）

1. **唯一的 CubeMX 插入点**在 [Core/Src/main.c:30](Core/Src/main.c#L30) — 通过 `#include "app_control.h"` 将应用层接入
2. 用户代码通过**覆盖 HAL 弱回调**方式实现逻辑，共覆盖了 6 个回调：
   - `HAL_TIM_PeriodElapsedCallback` — 在 `app_control.c`
   - `HAL_GPIO_EXTI_Callback` — 在 `app_control.c`
   - `HAL_ADC_ConvCpltCallback` — 在 `app_sampling.c`
   - `HAL_ADCEx_InjectedConvCpltCallback` — 在 `app_sampling.c`
   - `HAL_UART_TxCpltCallback` — 在 `app_monitor.c`
   - `HAL_DAC_IRQHandler` — 在 `stm32g4xx_it.c` 中通过 HAL 标准流程调用
3. 用户代码不修改任何 CubeMX 生成的 `USER CODE` 段之外的代码，构建系统 (`CMakeLists.txt`) 通过 `target_sources()` 和 `target_include_directories()` 将 `App/` 目录挂载到工程中。

### 关键设计原则

用户代码通过 `app_bsp` 层隔离 HAL API，应用逻辑（状态机、信号处理、协议封装）无需感知外设句柄细节；仅在必须直接引用 HAL 句柄的回调函数中直接使用 `hadc1` 等变量（如 `HAL_ADC_GetValue` 等），这在覆盖 HAL 回调时是不可避免的。