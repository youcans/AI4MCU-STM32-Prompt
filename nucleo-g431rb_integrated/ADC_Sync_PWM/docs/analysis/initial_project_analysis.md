# 初始工程结构分析

> 编制日期：2026-08-08
> 工程名称：ADC_Sync_PWM
> 分析范围：已编译成功的 STM32 初始工程（仅 CubeMX 生成代码，无应用层代码）

---

## 1. 工程目录与构建结构

### 1.1 顶层目录

```
ADC_Sync_PWM/
├── ADC_Sync_PWM.ioc          # CubeMX 工程文件
├── CMakeLists.txt            # 顶层 CMake（用户添加源文件和 include 路径的入口）
├── CMakePresets.json         # CMake 预设配置
├── STM32G431XX_FLASH.ld      # 链接脚本
├── startup_stm32g431xx.s     # 启动文件（中断向量表）
├── cmake/
│   ├── gcc-arm-none-eabi.cmake    # GCC 工具链配置
│   ├── starm-clang.cmake          # Clang 工具链配置（备用）
│   └── stm32cubemx/
│       └── CMakeLists.txt         # CubeMX 生成的核心构建文件
├── Core/
│   ├── Inc/                      # 外设头文件（每个外设一个 .h）
│   │   ├── main.h
│   │   ├── adc.h / dac.h / dma.h / gpio.h / tim.h / usart.h
│   │   ├── stm32g4xx_hal_conf.h
│   │   └── stm32g4xx_it.h
│   └── Src/                      # 外设源文件
│       ├── main.c
│       ├── adc.c / dac.c / dma.c / gpio.c / tim.c / usart.c
│       ├── stm32g4xx_it.c        # 中断入口
│       ├── stm32g4xx_hal_msp.c   # HAL MSP 初始化（空，实际 MSP 在各外设文件中）
│       ├── system_stm32g4xx.c    # 系统初始化
│       └── syscalls.c / sysmem.c
├── Drivers/
│   ├── CMSIS/                    # CMSIS 核心和器件头文件
│   └── STM32G4xx_HAL_Driver/     # HAL 库（仅包含使用的外设模块）
└── build/                        # CMake 构建输出目录
```

### 1.2 构建结构

- **构建系统**：CMake + GCC（arm-none-eabi）
- **编译标准**：C11
- **宏定义**：`USE_HAL_DRIVER`、`STM32G431xx`
- **库组织**：
  - `stm32cubemx`（INTERFACE 库）：提供 include 路径和宏定义
  - `STM32_Drivers`（OBJECT 库）：HAL 驱动源文件，链接到 `stm32cubemx`
  - `${CMAKE_PROJECT_NAME}`（可执行文件）：链接 CubeMX 应用源文件 + `STM32_Drivers`
- **用户扩展点**：顶层 `CMakeLists.txt` 中的 `target_sources`、`target_include_directories`、`target_compile_definitions` 区域，用于添加自定义源文件和路径

### 1.3 HAL 外设模块

已启用的 HAL 模块（`stm32g4xx_hal_conf.h` 中定义）：ADC、DAC、DMA、EXTI、GPIO、PWR、RCC、TIM、UART、CORTEX、FLASH。

未启用：CAN、COMP、CRC、CRYPT、FDCAN、FMAC、HRTIM、I2C、I2S、IRDA、IWDG、LPTIM、NAND、NOR、OPAMP、PCD、QSPI、RNG、RTC、SAI、SMARTCARD、SMBUS、SPI、SRAM、WWDG。

---

## 2. main.c 初始化流程和主循环结构

### 2.1 初始化顺序

```c
main()
  ├── HAL_Init()                        // HAL 库初始化、SysTick 配置
  ├── SystemClock_Config()              // 系统时钟 160 MHz（HSE 24 MHz → PLL）
  ├── MX_GPIO_Init()                    // GPIO（LED PA5、按键 PC13 EXTI）
  ├── MX_DMA_Init()                     // DMA1 时钟使能 + DMA1_Channel2 中断配置
  ├── MX_TIM1_Init()                    // TIM1 10 kHz 中心对齐 PWM + CH4 TRGO2
  ├── MX_ADC1_Init()                    // ADC1 注入组 + 规则组
  ├── MX_LPUART1_UART_Init()            // LPUART1 115200 bps + DMA TX
  ├── MX_DAC1_Init()                    // DAC1 OUT1
  ├── MX_TIM6_Init()                    // TIM6 500 Hz
  └── while(1) { }                      // 主循环（当前为空）
```

### 2.2 SystemClock_Config

| 参数 | 值 | 说明 |
|---|---|---|
| 调压器 | VOLTAGE_SCALE1_BOOST | 170 MHz 模式所需 |
| HSE | RCC_HSE_ON（24 MHz 晶振） | — |
| PLL Source | RCC_PLLSOURCE_HSE | — |
| PLLM | DIV3（HSE/3 = 8 MHz） | — |
| PLLN | ×40（VCO = 320 MHz） | — |
| PLLR | DIV2（PLLR = 160 MHz） | SYSCLK |
| SYSCLK Source | PLLCLK | 160 MHz |
| AHB Divider | DIV1 | HCLK = 160 MHz |
| APB1 Divider | DIV1 | PCLK1 = 160 MHz |
| APB2 Divider | DIV1 | PCLK2 = 160 MHz |
| Flash Latency | FLASH_LATENCY_4 | 160 MHz 需要 4 等待周期 |

### 2.3 主循环

当前 `while(1)` 为空，所有 USER CODE 区域可用于后续应用代码接入。

### 2.4 用户代码可接入位置

| 位置 | 区域 | 用途 |
|---|---|---|
| `/* USER CODE BEGIN Includes */` | 文件顶部 | 添加应用模块头文件 |
| `/* USER CODE BEGIN PV */` | 私有变量 | 全局状态变量、句柄 |
| `/* USER CODE BEGIN 0 */` | main() 之前 | 辅助函数定义 |
| `/* USER CODE BEGIN 1 */` | main() 开头 | 初始化前准备 |
| `/* USER CODE BEGIN Init */` | HAL_Init 之后 | 额外系统初始化 |
| `/* USER CODE BEGIN 2 */` | 外设初始化之后 | 模块初始化、启动配置 |
| `/* USER CODE BEGIN WHILE */` / `BEGIN 3` | while(1) 内 | 后台任务循环 |
| `/* USER CODE BEGIN 4 */` | main() 之后 | 用户函数实现 |

---

## 3. 主要外设初始化函数和句柄

### 3.1 句柄汇总

| 句柄 | 类型 | 定义位置 | 外部引用 |
|---|---|---|---|
| `hadc1` | ADC_HandleTypeDef | adc.c | stm32g4xx_it.c |
| `hdac1` | DAC_HandleTypeDef | dac.c | stm32g4xx_it.c |
| `htim1` | TIM_HandleTypeDef | tim.c | — |
| `htim6` | TIM_HandleTypeDef | tim.c | stm32g4xx_it.c |
| `hlpuart1` | UART_HandleTypeDef | usart.c | stm32g4xx_it.c |
| `hdma_lpuart1_tx` | DMA_HandleTypeDef | usart.c | stm32g4xx_it.c |

### 3.2 TIM1（10 kHz 中心对齐 PWM）

| 参数 | 值 | HAL 常量 |
|---|---|---|
| Prescaler | 0 | — |
| CounterMode | Center Aligned 1 | TIM_COUNTERMODE_CENTERALIGNED1 |
| Period (ARR) | 8000 | — |
| ClockDivision | DIV1 | TIM_CLOCKDIVISION_DIV1 |
| RepetitionCounter | 0 | — |
| AutoReloadPreload | Enable | TIM_AUTORELOAD_PRELOAD_ENABLE |
| CH1-CH3 Mode | PWM mode 1 | TIM_OCMODE_PWM1 |
| CH1-CH3 Pulse | 0 | — |
| CH4 Pulse | 4000 | — |
| CH1-CH4 Polarity | High | TIM_OCPOLARITY_HIGH |
| CH1N-CH3N Polarity | High | TIM_OCNPOLARITY_HIGH |
| MasterOutputTrigger2 | OC4REF | TIM_TRGO2_OC4REF |
| DeadTime | 32（200 ns） | — |
| BreakState | Disable | TIM_BREAK_DISABLE |
| Break2State | Disable | TIM_BREAK2_DISABLE |
| AutomaticOutput | Disable | TIM_AUTOMATICOUTPUT_DISABLE |
| OffStateRunMode | Disable | TIM_OSSR_DISABLE |
| OffStateIDLEMode | Disable | TIM_OSSI_DISABLE |

**注意**：TIM1 的 `MX_TIM1_Init` 调用 `HAL_TIM_Base_Init` + `HAL_TIM_PWM_Init`，但 **不包含** 启动计数器或使能 PWM 输出的调用，这部分由应用代码完成。

### 3.3 TIM6（500 Hz 时基）

| 参数 | 值 | HAL 常量 |
|---|---|---|
| Prescaler | 159 | — |
| CounterMode | Up | TIM_COUNTERMODE_UP |
| Period (ARR) | 1999 | — |
| AutoReloadPreload | Disable | TIM_AUTORELOAD_PRELOAD_DISABLE |
| MasterOutputTrigger | Update | TIM_TRGO_UPDATE |

**注意**：TIM6 的计数器启动由应用代码完成。

### 3.4 ADC1

**公共设置：**

| 参数 | 值 |
|---|---|
| ClockPrescaler | ADC_CLOCK_SYNC_PCLK_DIV4（40 MHz） |
| Resolution | ADC_RESOLUTION_12B |
| DataAlign | ADC_DATAALIGN_RIGHT |
| ScanConvMode | ADC_SCAN_ENABLE |
| EOCSelection | ADC_EOC_SEQ_CONV |
| ContinuousConvMode | DISABLE |
| DMAContinuousRequests | DISABLE |
| Multi Mode | ADC_MODE_INDEPENDENT |
| ADC 时钟源 | RCC_ADC12CLKSOURCE_SYSCLK |

**规则组（温度采样）：**

| 参数 | 值 |
|---|---|
| Channel | ADC_CHANNEL_TEMPSENSOR_ADC1（IN16） |
| SamplingTime | ADC_SAMPLETIME_247CYCLES_5 |
| ExternalTrigConv | ADC_EXTERNALTRIG_T6_TRGO |
| ExternalTrigConvEdge | ADC_EXTERNALTRIGCONVEDGE_RISING |

**注入组（三通道同步采样）：**

| 参数 | 值 |
|---|---|
| Rank 1 | ADC_CHANNEL_2（PA1），24.5 cycles |
| Rank 2 | ADC_CHANNEL_12（PB1），24.5 cycles |
| Rank 3 | ADC_CHANNEL_15（PB0），24.5 cycles |
| InjectedNbrOfConversion | 3 |
| ExternalTrigInjecConv | ADC_EXTERNALTRIGINJEC_T1_TRGO2 |
| ExternalTrigInjecConvEdge | ADC_EXTERNALTRIGINJECCONV_EDGE_RISING |
| InjectedDiscontinuousConvMode | DISABLE |
| AutoInjectedConv | DISABLE |

**注意**：ADC1 的 `MX_ADC1_Init` 不启动 ADC 或注入组转换，由应用代码通过 `HAL_ADCEx_InjectedStart_IT()` 等调用启动。

### 3.5 DAC1

| 参数 | 值 |
|---|---|
| Trigger | DAC_TRIGGER_NONE |
| OutputBuffer | DAC_OUTPUTBUFFER_ENABLE |
| ConnectOnChipPeripheral | DAC_CHIPCONNECT_EXTERNAL |
| SampleAndHold | DAC_SAMPLEANDHOLD_DISABLE |
| SignedFormat | DISABLE |

**注意**：DAC1 初始化后不立即输出，250 Hz 方波由 TIM6 中断中软件翻转实现。

### 3.6 LPUART1 + DMA

**LPUART1：**

| 参数 | 值 |
|---|---|
| BaudRate | 115200 |
| WordLength | UART_WORDLENGTH_8B |
| StopBits | UART_STOPBITS_1 |
| Parity | UART_PARITY_NONE |
| Mode | UART_MODE_TX_RX |
| HwFlowCtl | UART_HWCONTROL_NONE |
| LPUART1 时钟源 | RCC_LPUART1CLKSOURCE_PCLK1（160 MHz） |

**DMA TX：**

| 参数 | 值 |
|---|---|
| Instance | DMA1_Channel2 |
| Request | DMA_REQUEST_LPUART1_TX |
| Direction | DMA_MEMORY_TO_PERIPH |
| PeriphInc | DMA_PINC_DISABLE |
| MemInc | DMA_MINC_ENABLE |
| DataAlignment | BYTE（内存和外设） |
| Mode | DMA_NORMAL |
| Priority | DMA_PRIORITY_LOW |

**注意**：LPUART1 和 DMA 初始化后不立即启动发送，由应用代码在 500 Hz 周期中调用 `HAL_UART_Transmit_DMA()` 启动。

### 3.7 GPIO

| 引脚 | 功能 | 模式 |
|---|---|---|
| PA5 | LD2（LED） | GPIO_MODE_OUTPUT_PP，初始 Low |
| PC13 | B1（按键） | GPIO_MODE_IT_RISING，NOPULL（外部下拉） |
| PA1/PB0/PB1 | ADC 输入 | GPIO_MODE_ANALOG（在 ADC MSP 中配置） |
| PA2/PA3 | LPUART1 TX/RX | GPIO_MODE_AF_PP，AF12（在 LPUART1 MSP 中配置） |
| PA4 | DAC OUT1 | GPIO_MODE_ANALOG（在 DAC MSP 中配置） |
| PA8/PA9/PA10 | TIM1 CH1-CH3 | GPIO_MODE_AF_PP，AF6（在 TIM1 MspPostInit 中配置） |
| PB13/PB14 | TIM1 CH1N-CH2N | GPIO_MODE_AF_PP，AF6 |
| PB15 | TIM1 CH3N | GPIO_MODE_AF_PP，AF4 |

---

## 4. 已生成的中断入口和相关 HAL 回调

### 4.1 中断向量表

启动文件 `startup_stm32g431xx.s` 中定义了全部中断向量，工程中已实现的中断入口：

| 中断入口 | 中断源 | HAL 回调链路 |
|---|---|---|
| `SysTick_Handler` | SysTick | → `HAL_IncTick()` |
| `DMA1_Channel2_IRQHandler` | DMA1 Ch2（LPUART1 TX） | → `HAL_DMA_IRQHandler(&hdma_lpuart1_tx)` |
| `ADC1_2_IRQHandler` | ADC1（注入组 JEOS + 规则组 EOC） | → `HAL_ADC_IRQHandler(&hadc1)` |
| `EXTI15_10_IRQHandler` | EXTI Line 13（按键） | → `HAL_GPIO_EXTI_IRQHandler(B1_Pin)` |
| `TIM6_DAC_IRQHandler` | TIM6 更新 + DAC 欠载 | → `HAL_TIM_IRQHandler(&htim6)` + `HAL_DAC_IRQHandler(&hdac1)` |
| `LPUART1_IRQHandler` | LPUART1 全局 | → `HAL_UART_IRQHandler(&hlpuart1)` |

### 4.2 NVIC 优先级（当前工程实际值）

| 中断入口 | 抢占优先级 | 子优先级 | 设置位置 |
|---|---|---|---|
| ADC1_2_IRQn | 0 | 0 | adc.c `HAL_ADC_MspInit` |
| TIM6_DAC_IRQn | 1 | 0 | tim.c `HAL_TIM_Base_MspInit`（TIM6 分支）+ dac.c `HAL_DAC_MspInit` |
| DMA1_Channel2_IRQn | 2 | 0 | dma.c `MX_DMA_Init` |
| LPUART1_IRQn | 3 | 0 | usart.c `HAL_UART_MspInit` |
| EXTI15_10_IRQn | 4 | 0 | gpio.c `MX_GPIO_Init` |

以上优先级为 CubeMX 生成的实际值，已确认为最终配置。

### 4.3 待实现的 HAL 回调

CubeMX 生成的代码调用了 HAL 中断处理函数，但对应的**回调函数尚未实现**，这些是应用代码的主要接入点：

| HAL 回调 | 触发条件 | 中断入口 | 预期用途（依据设计文档） |
|---|---|---|---|
| `HAL_ADCEx_InjectedConvCpltCallback` | 注入组序列完成（JEOS） | ADC1_2 | MD-02 读取注入组三路采样值 → MD-03 计算 → MD-04 LED 翻转 |
| `HAL_ADC_ConvCpltCallback` | 规则组转换完成（EOC） | ADC1_2 | MD-02 读取温度值 → MD-04 温度判断 |
| `HAL_TIM_PeriodElapsedCallback` | TIM6 更新事件 | TIM6_DAC | MD-04 500 Hz 调度（DAC 翻转、LED 分频、串口触发） |
| `HAL_GPIO_EXTI_Callback` | EXTI13 上升沿 | EXTI15_10 | MD-04 按键事件 → 状态切换 |
| `HAL_UART_TxCpltCallback` | LPUART1 DMA 发送完成 | DMA1_Channel2 | MD-05 清除 DMA 发送忙状态 |

---

## 5. 后续应用代码的主要接入位置

### 5.1 初始化阶段

| 接入位置 | 操作 |
|---|---|
| `/* USER CODE BEGIN Includes */`（main.c） | `#include` 各应用模块头文件（MD-01 ~ MD-05） |
| `/* USER CODE BEGIN PV */`（main.c） | 定义全局状态变量、模块句柄 |
| `/* USER CODE BEGIN 2 */`（main.c `MX_TIM6_Init` 之后） | MD-04 初始化调度：设定初始状态为停止、MD-01 初始化系统、启动 TIM6 |
| `/* USER CODE BEGIN 0 */`（各外设 .c 文件） | 编写 STM32CubeMX 外设的 HAL 用户代码 |

### 5.2 中断回调

在对应的 `stm32g4xx_it.c` 或独立的模块文件中实现上述 HAL 回调函数。

### 5.3 主循环

在 `/* USER CODE BEGIN WHILE */` 内实现后台任务（如目前设计中没有需要主循环处理的任务，可保留为空）。

### 5.4 自定义源文件

在顶层 `CMakeLists.txt` 的 `target_sources` 和 `target_include_directories` 中添加模块源文件路径。

---

## 6. 其他与后续编程直接相关的情况

### 6.1 DMA 句柄和 LPUART 句柄的关联

`hdma_lpuart1_tx` 在 `usart.c` 中定义，通过 `__HAL_LINKDMA(&hlpuart1, hdmatx, hdma_lpuart1_tx)` 与 LPUART1 句柄关联。MD-05 启动 DMA 发送时需使用 `hlpuart1.hdmatx`。

### 6.2 ADC1 注入组和规则组的中断区分

两者共用 `ADC1_2_IRQHandler` → `HAL_ADC_IRQHandler(&hadc1)`，HAL 内部通过 ADC 状态寄存器中的 JEOS 和 EOC 标志分别触发 `HAL_ADCEx_InjectedConvCpltCallback` 和 `HAL_ADC_ConvCpltCallback`。应用代码只需分别实现这两个回调即可，无需手动检查标志位。

### 6.3 TIM6_DAC 中断的共享特性

`TIM6_DAC_IRQHandler` 同时服务 TIM6 和 DAC1/3 的欠载中断。中断入口调用 `HAL_TIM_IRQHandler(&htim6)` 和 `HAL_DAC_IRQHandler(&hdac1)`。本项目不使用 DAC 中断（仅软件写入），DAC 中断不会触发。

### 6.4 TIM1 未启动 PWM 输出

`MX_TIM1_Init` 完成后，TIM1 的计数器和 PWM 输出均处于停止状态。根据 `runtime_state_analysis.md`，TIM1 由 MD-04 在进入运行状态时启动，在进入停止/异常状态时停止。

### 6.5 LPUART1 的 FIFO 模式

`MX_LPUART1_UART_Init` 中调用了 `HAL_UARTEx_DisableFifoMode`，LPUART1 工作在非 FIFO 模式。JustFloat 帧 8 字节/帧，单字节 DMA 传输，与 FIFO 禁用兼容。

### 6.6 未使用引脚

CubeMX 生成的代码中对未使用的 GPIO 引脚未做特殊处理（保持复位默认状态）。PA13/PA14 已被 SWD 占用。

---

> **后续使用建议**：本文档用于指导应用代码（MD-01 ~ MD-05）的编写和集成。具体模块职责、数据流和状态切换规则请参考 `docs/design/module_design.md`、`docs/design/module_relationship_analysis.md` 和 `docs/design/runtime_state_analysis.md`。
