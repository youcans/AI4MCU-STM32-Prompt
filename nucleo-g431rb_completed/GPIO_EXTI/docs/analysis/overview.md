# 工程分析报告：GPIO_EXTI（按键中断控制 LED）

> 分析日期：2026-07-22
> 功能目标：按下 NUCLEO-G431RB 用户按键触发外部中断，切换板载 LED2 状态

---

## 1. 开发板与 MCU 型号

| 项目 | 型号 |
|---|---|
| **开发板** | NUCLEO-G431RB（NUCLEO-64 封装，板号 MB1367） |
| **MCU** | STM32G431RBT6（STM32G4 系列，Cortex-M4，LQFP64） |
| **CubeMX 版本** | 6.16.0 |
| **固件包** | STM32Cube FW_G4 V1.6.3 |
| **工具链** | CMake / GCC |

来源：[GPIO_EXTI.ioc:2](../../GPIO_EXTI.ioc#L2), [:9](../../GPIO_EXTI.ioc#L9), [:73](../../GPIO_EXTI.ioc#L73), [:75](../../GPIO_EXTI.ioc#L75), [:91](../../GPIO_EXTI.ioc#L91)

---

## 2. 相关外设、端口和引脚配置

### 2.1 用户按键（USER_BTN）

| 属性 | 值 | 来源 |
|---|---|---|
| GPIO 端口 | **GPIOC** | [main.h:63](../../Core/Inc/main.h#L63) |
| 引脚 | **PC13** / `GPIO_PIN_13` | [main.h:62](../../Core/Inc/main.h#L62) |
| 工作模式 | `GPIO_MODE_IT_RISING`（上升沿触发外部中断） | [main.c:168](../../Core/Src/main.c#L168) |
| 上下拉 | `GPIO_NOPULL`（无内部上下拉） | [main.c:169](../../Core/Src/main.c#L169) |
| EXTI 线 | `EXTI_LINE_13`（对应 GPIO13） | [stm32g4xx_nucleo.h:200](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L200) |
| 中断号 | `EXTI15_10_IRQn`（EXTI 线 10~15 共享中断向量） | [main.h:64](../../Core/Inc/main.h#L64) |
| 中断优先级 | 抢占优先级 5，子优先级 0 | [main.c:180](../../Core/Src/main.c#L180) |

.ioc 中对应配置：
```
PC13.GPIO_Label=USER_BTN
PC13.Signal=GPXTI13          → GPIO 外部中断，EXTI 线 13
NVIC.EXTI15_10_IRQn=true\:5\:0\:true\:false\:true\:true\:true\:true
                              → 已使能，优先级 5:0
```

### 2.2 板载 LED（LD2）

| 属性 | 值 | 来源 |
|---|---|---|
| GPIO 端口 | **GPIOA** | [main.h:66](../../Core/Inc/main.h#L66) |
| 引脚 | **PA5** / `GPIO_PIN_5` | [main.h:65](../../Core/Inc/main.h#L65) |
| 工作模式 | `GPIO_MODE_OUTPUT_PP`（推挽输出） | [main.c:174](../../Core/Src/main.c#L174) |
| 上下拉 | `GPIO_NOPULL` | [main.c:175](../../Core/Src/main.c#L175) |
| 速度 | `GPIO_SPEED_FREQ_LOW` | [main.c:176](../../Core/Src/main.c#L176) |
| 初始输出 | `GPIO_PIN_RESET`（低电平，LED 熄灭） | [main.c:164](../../Core/Src/main.c#L164) |

### 2.3 系统时钟

| 属性 | 值 |
|---|---|
| 时钟源 | HSI → PLL（`RCC_SYSCLKSOURCE_PLLCLK`） |
| SYSCLK | 170 MHz |
| HCLK / APB1 / APB2 | 均为 170 MHz |
| PLLM / PLLN | ÷4, ×85 |
| Flash 等待周期 | 4（`FLASH_LATENCY_4`） |

---

## 3. 已有初始化函数、宏定义及所在文件

### 3.1 初始化函数

| 函数 | 文件 | 功能 |
|---|---|---|
| `MX_GPIO_Init()` | [main.c:152-186](../../Core/Src/main.c#L152-L186) | 初始化 GPIO：PC13 配置为 IT_RISING 外部中断模式，使能 GPIOC/GPIOA 时钟，配置 LD2 推挽输出，设置 EXTI15_10_IRQn 优先级并使能 |
| `HAL_Init()` | main.c:66 | 复位外设、初始化 Flash 接口和 SysTick |
| `SystemClock_Config()` | [main.c:105-145](../../Core/Src/main.c#L105-L145) | 配置 170MHz 系统时钟 |
| `EXTI15_10_IRQHandler()` | [stm32g4xx_it.c:204-213](../../Core/Src/stm32g4xx_it.c#L204-L213) | EXTI15_10 中断服务函数，调用 `HAL_GPIO_EXTI_IRQHandler(USER_BTN_Pin)` |

### 3.2 宏定义

**文件：[Core/Inc/main.h](../../Core/Inc/main.h)**

```c
#define USER_BTN_Pin                  GPIO_PIN_13     // L62
#define USER_BTN_GPIO_Port            GPIOC           // L63
#define USER_BTN_EXTI_IRQn            EXTI15_10_IRQn  // L64
#define LD2_Pin                       GPIO_PIN_5      // L65
#define LD2_GPIO_Port                 GPIOA           // L66
```

### 3.3 全局变量

**文件：[Core/Src/main.c:34](../../Core/Src/main.c#L34)**

```c
__IO uint32_t BspButtonState = BUTTON_RELEASED;
```

这是一个由 CubeMX BSP Common Demo 功能生成的变量，用于跟踪按键状态。`BUTTON_RELEASED`（值为 0）定义在 [stm32g4xx_nucleo.h:189](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L189)。

---

## 4. 实现功能需要使用的 HAL 函数

有两种实现路径，所涉及的 HAL 函数不同：

### 4.1 路径 A：使用 HAL 回调（推荐，最简洁）

在 `main.c` 中重写 HAL 弱回调函数即可，不需要在本轮外新增 HAL 调用：

| 函数 | 作用 |
|---|---|
| `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)` | HAL 库提供的 `__weak` 回调，GPIO 外部中断触发后由 `HAL_GPIO_EXTI_IRQHandler()` 内部调用。重写此函数即可响应按键中断。 |
| `HAL_GPIO_TogglePin(GPIOA, LD2_Pin)` | 在回调函数中切换 LED 状态 |

### 4.2 路径 B：使用 BSP API

| 函数 | 文件 | 作用 |
|---|---|---|
| `BSP_PB_Callback(Button_TypeDef Button)` | [stm32g4xx_nucleo.c:354](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L354) | `__weak` 回调，重写后用于响应按键事件 |
| `BSP_LED_Toggle(LED2)` | [stm32g4xx_nucleo.c:241](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L241) | 切换 LED2 |

---

## 5. 需要修改的文件和代码位置

当前工程已有的中断链路为：

```
按键按下（PC13 上升沿）
  → EXTI15_10_IRQHandler()              [stm32g4xx_it.c:204]
    → HAL_GPIO_EXTI_IRQHandler(USER_BTN_Pin)  [stm32g4xx_it.c:209]
      → HAL_GPIO_EXTI_Callback(GPIO_Pin)      [HAL 弱回调，待重写]
```

只需修改 **一个文件**：

| 文件 | 修改位置 | 修改内容 |
|---|---|---|
| [Core/Src/main.c](../../Core/Src/main.c) | `/* USER CODE BEGIN 4 */`（第 188-190 行之间） | 重写 `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`，在函数内判断 `GPIO_Pin == USER_BTN_Pin`，然后调用 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)` |

**不需要**修改的文件：
- `.ioc` — 不需要
- `stm32g4xx_it.c` — EXTI15_10_IRQHandler 已正确调用 `HAL_GPIO_EXTI_IRQHandler(USER_BTN_Pin)`
- `stm32g4xx_nucleo.c` / `.h` — 不需要（走 HAL 回调路径时不依赖 BSP 的按钮 API）
- `stm32g4xx_hal_conf.h` — 不需要

---

## 6. 是否需要新增变量、函数、文件或中断

| 类型 | 是否需要 | 说明 |
|---|---|---|
| **新增文件** | 不需要 | — |
| **新增中断** | 不需要 | `EXTI15_10_IRQn` 已在 CubeMX 中配置并使能 |
| **新增全局变量** | 不需要 | 按 LED 切换功能无需额外状态变量 |
| **新增函数** | 需要 | 重写 `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`（弱回调覆盖） |
| **修改初始化代码** | 不需要 | `MX_GPIO_Init()` 已配置好 PC13 中断模式和 EXTI 使能 |

---

## 7. 风险和注意事项

### 7.1 GPIO_NOPULL 与按键硬件

[main.c:169](../../Core/Src/main.c#L169) 中 PC13 配置为 `GPIO_NOPULL`（无内部上下拉）。在 NUCLEO-G431RB 原理图上，用户按键 B1 连接方式为：**按钮一端接 VDD，另一端接 PC13，PC13 同时通过 100kΩ 电阻（R33）下拉到 GND**。

- 按钮未按下时：PC13 为低电平（外部 100k 下拉）
- 按钮按下时：PC13 被拉高到 VDD（上升沿触发中断）

`GPIO_NOPULL` 在此配置下可正常工作（依赖外部下拉电阻），但上电瞬间或强 EMI 环境下，PC13 浮空状态下可能产生误触发。**建议**将上下拉改为 `GPIO_PULLDOWN`，与 BSP 的 `BSP_PB_Init()` 函数中的配置保持一致（[stm32g4xx_nucleo.c:284](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.c#L284)）。

### 7.2 中断优先级冲突

- 当前工程：`EXTI15_10_IRQn` = 优先级 5（[main.c:180](../../Core/Src/main.c#L180)）
- BSP 配置：`BSP_BUTTON_USER_IT_PRIORITY` = 15（最低优先级，[stm32g4xx_nucleo_conf.h:56](../../Core/Inc/stm32g4xx_nucleo_conf.h#L56)）
- 优先级分组：`NVIC_PRIORITYGROUP_4`（4 位全为抢占优先级，无子优先级，[GPIO_EXTI.ioc:41](../../GPIO_EXTI.ioc#L41)）

两组优先级数字不一致，但当前使用的是 CubeMX 生成的 `MX_GPIO_Init()` 中设置的值（5）。在 `NVIC_PRIORITYGROUP_4` 下，优先级 5 表示中等抢占优先级，对于按键中断来说合理。

### 7.3 按键抖动（Debounce）

硬件按键在按下和释放时存在机械抖动（通常 5~20ms）。`GPIO_MODE_IT_RISING` 配置下，一次按键可能触发多次中断。**建议**在回调中添加软件去抖动逻辑或简单的延时判断。

### 7.4 Bsp_Common_DEMO 与 BspButtonState

[GPIO_EXTI.ioc:31](../../GPIO_EXTI.ioc#L31) 中 `Bsp_Common_DEMO=true` 表示 CubeMX 开启了 BSP Common 演示功能，生成了 `BspButtonState` 变量。但 `main.c` 中所有 USER CODE 区均为空，BSP 初始化代码（`BSP_LED_Init`、`BSP_PB_Init`）未被调用。**当前有效的初始化路径是 `MX_GPIO_Init()`**，而非 BSP API。如果后续代码中混用了 BSP 函数和 HAL 直接调用，需注意两边对 GPIO 和中断的配置可能存在冲突（例如 BSP 使用 `HAL_EXTI_RegisterCallback` 注册了 `BUTTON_USER_EXTI_Callback`，而 HAL 回调路径使用 `HAL_GPIO_EXTI_Callback`）。

### 7.5 有利条件

- `.ioc` 已配置 `PC13 → GPXTI13`，CubeMX 生成的初始化代码已经完整地配置了中断模式、NVIC 优先级和中断使能
- `stm32g4xx_it.c` 中 `EXTI15_10_IRQHandler` 已正确调用 `HAL_GPIO_EXTI_IRQHandler(USER_BTN_Pin)`
- 所有必要的宏定义（`USER_BTN_Pin`、`LD2_Pin` 等）已在 `main.h` 中自动生成
- 只需要在 `main.c` 的 `/* USER CODE BEGIN 4 */` 区域中添加一个简短的 `HAL_GPIO_EXTI_Callback` 重写即可实现完整功能
