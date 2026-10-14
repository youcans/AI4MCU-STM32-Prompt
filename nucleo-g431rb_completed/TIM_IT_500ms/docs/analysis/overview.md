# 工程分析报告：TIM6 中断方式实现 LED2 500ms 周期闪烁

## 1. 开发板和 MCU 型号

| 项目 | 型号 |
|---|---|
| **开发板** | NUCLEO-G431RB (`TIM_IT_500ms.ioc:2`) |
| **MCU** | STM32G431RBT6 (`TIM_IT_500ms.ioc:9`)，LQFP64 封装 (`TIM_IT_500ms.ioc:18`) |
| **CubeMX 版本** | 6.16.0 (`TIM_IT_500ms.ioc:31`) |
| **固件包** | STM32Cube FW_G4 V1.6.3 (`TIM_IT_500ms.ioc:76`) |

## 2. 相关外设、端口和引脚配置

| 外设 | 端口/引脚 | 功能 | 配置 |
|---|---|---|---|
| **TIM6** | 内部时钟源 (APB1 Timer Clock) | 基本定时器，用作周期中断 | PSC=1599, ARR=49999 → **500ms 周期** |
| **GPIOA Pin 5** | PA5 (标签 `LD2`) | 推挽输出，驱动板载 LED2 | `GPIO_MODE_OUTPUT_PP`, No Pull, Low Speed, 初始输出 RESET |
| **GPIOC Pin 13** | PC13 (标签 `USER_BTN`) | 输入，板载用户按钮 | `GPIO_MODE_INPUT`, No Pull |
| **NVIC** | TIM6_DAC_IRQn | TIM6 全局中断 | PreemptPriority=5, 已使能 (`TIM_IT_500ms.ioc:45`) |

**时钟系统**（`TIM_IT_500ms.ioc:100-108`）：
- HCLK / SYSCLK / APB1 Timer Clock / APB2 Timer Clock 均 = **160 MHz**
- 时钟源：HSI (16 MHz) → PLL (×20 / 2) → 160 MHz

**TIM6 500ms 周期验证**：

```text
TIM6_CLK = APB1 Timer Clock = 160 MHz
CNT_CLK  = 160 MHz / (1599 + 1) = 100 kHz
周期     = (49999 + 1) / 100 kHz = 0.5 s = 500 ms ✓
```

## 3. 已有初始化函数、宏定义及所在文件

### 3.1 初始化函数

| 函数 | 所在文件 | 说明 |
|---|---|---|
| `SystemClock_Config()` | [main.c:118](../../Core/Src/main.c#L118) | 系统时钟配置 (HSI→PLL→160MHz) |
| `MX_GPIO_Init()` | [main.c:203](../../Core/Src/main.c#L203) | PA5(LD2)、PC13(USER_BTN) 初始化 |
| `MX_TIM6_Init()` | [main.c:165](../../Core/Src/main.c#L165) | TIM6 时基初始化 (PSC=1599, ARR=49999) |
| `HAL_TIM_Base_Init()` | 在 `MX_TIM6_Init()` 中调用 | HAL 层 TIM 时基初始化 |
| `TIM6_DAC_IRQHandler()` | [stm32g4xx_it.c:204](../../Core/Src/stm32g4xx_it.c#L204) | TIM6 中断服务函数，已调用 `HAL_TIM_IRQHandler(&htim6)` |

### 3.2 关键宏定义

| 宏 | 值 | 所在文件 |
|---|---|---|
| `LD2_Pin` | `GPIO_PIN_5` | [main.h:64](../../Core/Inc/main.h#L64) |
| `LD2_GPIO_Port` | `GPIOA` | [main.h:65](../../Core/Inc/main.h#L65) |
| `USER_BTN_Pin` | `GPIO_PIN_13` | [main.h:62](../../Core/Inc/main.h#L62) |
| `USER_BTN_GPIO_Port` | `GPIOC` | [main.h:63](../../Core/Inc/main.h#L63) |
| `LED2` | `0U` | [stm32g4xx_nucleo.h:62](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L62) |

### 3.3 关键全局变量

| 变量 | 类型 | 所在文件 |
|---|---|---|
| `htim6` | `TIM_HandleTypeDef` | [main.c:45](../../Core/Src/main.c#L45)，在 [stm32g4xx_it.c:58](../../Core/Src/stm32g4xx_it.c#L58) 通过 `extern` 引用 |

### 3.4 BSP LED 函数（可供调用）

| 函数 | 所在文件 |
|---|---|
| `BSP_LED_Init(LED2)` | [stm32g4xx_nucleo.c:168](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L168) |
| `BSP_LED_On(LED2)` | [stm32g4xx_nucleo.c:213](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L213) |
| `BSP_LED_Off(LED2)` | [stm32g4xx_nucleo.c:227](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L227) |
| `BSP_LED_Toggle(LED2)` | [stm32g4xx_nucleo.c:241](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L241) |

## 4. 实现功能需要使用的 HAL 函数

| HAL 函数 | 用途 | 调用位置 |
|---|---|---|
| `HAL_TIM_Base_Start_IT(&htim6)` | 启动 TIM6 并使能更新中断 | `main()` 中 `/* USER CODE BEGIN 2 */` 区域 |
| `HAL_TIM_PeriodElapsedCallback(&htim6)` | 定时器周期中断回调（weak 函数，需重写） | 在 `main.c` 中 `/* USER CODE BEGIN 4 */` 区域实现 |
| `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)` | 翻转 LED2 状态 | 在 `HAL_TIM_PeriodElapsedCallback` 中调用 |

> **备选**：也可以用 `BSP_LED_Toggle(LED2)` 替代 `HAL_GPIO_TogglePin()`，效果等价。

**调用链**（已存在，无需修改）：

```text
TIM6 计数溢出
  → TIM6_DAC_IRQHandler()              [stm32g4xx_it.c:204]
    → HAL_TIM_IRQHandler(&htim6)       [stm32g4xx_it.c:209] ← 已有
      → HAL_TIM_PeriodElapsedCallback() ← 需要用户实现
```

## 5. 需要修改的文件和代码位置

| 文件 | 修改位置 | 修改内容 |
|---|---|---|
| **Core/Src/main.c** | `/* USER CODE BEGIN 2 */`（第 94-96 行之间） | 添加 `HAL_TIM_Base_Start_IT(&htim6);` |
| **Core/Src/main.c** | `/* USER CODE BEGIN 4 */`（第 235-237 行之间） | 实现 `HAL_TIM_PeriodElapsedCallback()` 回调函数 |

**不需要修改的文件**：
- `Core/Src/stm32g4xx_it.c` — `TIM6_DAC_IRQHandler` 已正确调用 `HAL_TIM_IRQHandler(&htim6)`
- `Core/Inc/main.h` — LD2 相关宏定义已完备
- `TIM_IT_500ms.ioc` — TIM6 外设配置已完备

## 6. 是否需要新增变量、函数、文件或中断

| 项目 | 需要? | 说明 |
|---|---|---|
| **新增 .c/.h 文件** | 否 | 功能简单，所有代码可写入已有文件的 USER CODE 区域 |
| **新增全局变量** | 否 | `htim6` 已由 CubeMX 生成 |
| **新增中断服务函数** | 否 | `TIM6_DAC_IRQHandler` 已生成且实现正确 |
| **新增用户函数** | 是，需要一个 | 实现弱回调 `void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)` |
| **修改 NVIC 配置** | 否 | TIM6 中断已在 CubeMX 中使能，优先级=5 |

## 7. 风险与注意事项

| # | 风险/注意事项 | 说明 |
|---|---|---|
| 1 | **TIM6 中断优先级为 5** | SysTick 优先级更高（通常为 0），如果 SysTick 或其他高优先级 ISR 占用时间过长，TIM6 中断响应会有抖动。对于 500ms LED 闪烁无影响，但对精确计时场景需留意。 |
| 2 | **TIM6 是基本定时器** | TIM6 没有输出比较/输入捕获通道，仅支持时基和 DAC 触发。`HAL_TIM_PeriodElapsedCallback` 是唯一有效的用户回调入口。 |
| 3 | **回调方式为 Weak 重写** | `.ioc` 中 `RegisterCallBack=` 为空，表示未启用 HAL 寄存器回调机制。因此必须通过重写 `__weak` 函数 `HAL_TIM_PeriodElapsedCallback()` 来实现中断响应。不要在中断服务函数 `TIM6_DAC_IRQHandler` 的 USER CODE 区直接写业务代码——不是不可以，但不是推荐做法。 |
| 4 | **CubeMX 重新生成会覆盖非 USER 区** | 代码必须写在 `/* USER CODE BEGIN ... */` 和 `/* USER CODE END ... */` 之间，否则重新生成时会被覆盖。 |
| 5 | **调用顺序** | `HAL_TIM_Base_Start_IT(&htim6)` 必须在 `MX_TIM6_Init()` 之后调用。当前 `main()` 在 `MX_TIM6_Init()` 之后的 `/* USER CODE BEGIN 2 */` 区写入即可满足。 |
| 6 | **LD2 初始状态** | `MX_GPIO_Init()` 中 LD2 初始输出为 `GPIO_PIN_RESET`（LED 灭）。首次 500ms 中断触发后 LED 翻转点亮，行为正确。 |
| 7 | **时钟源确认** | TIM6 使用内部时钟（`VP_TIM6_VS_ClockSourceINT`），不从外部引脚输入，配置正确。 |
| 8 | **内核时钟 Scaling** | 代码使用了 `PWR_REGULATOR_VOLTAGE_SCALE1_BOOST` 以支持 160MHz，这是 STM32G4 的最高频率模式，功耗较高但功能正常。 |

## 结论

工程条件完全具备。TIM6 已配置为 500ms 周期，NVIC 中断已使能，ISR 框架已生成，LD2 引脚已初始化。仅需在 [main.c](../../Core/Src/main.c) 的 **USER CODE BEGIN 2** 区添加 `HAL_TIM_Base_Start_IT(&htim6)`，在 **USER CODE BEGIN 4** 区重写 `HAL_TIM_PeriodElapsedCallback()` 即可实现功能。
