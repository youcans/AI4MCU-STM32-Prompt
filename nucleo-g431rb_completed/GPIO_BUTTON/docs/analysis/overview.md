# 工程分析报告 — GPIO 按键控制 LED

> 分析日期：2026-07-22
>
> 目标功能：主循环轮询读取 NUCLEO-G431RB 用户按键（USER_BTN），每检测到一次有效按键操作，切换板载 LED2 状态；持续按住不重复切换，松开后可响应下一次操作。

---

## 1. 开发板和 MCU 型号

| 项目 | 型号 |
|------|------|
| **开发板** | NUCLEO-G431RB |
| **MCU** | STM32G431RBT6（Cortex-M4，LQFP64 封装） |

来源：[GPIO_BUTTON.ioc:2](../../GPIO_BUTTON.ioc#L2)、[GPIO_BUTTON.ioc:9](../../GPIO_BUTTON.ioc#L9)、[GPIO_BUTTON.ioc:16-17](../../GPIO_BUTTON.ioc#L16-L17)

---

## 2. 相关外设、端口和引脚配置

| 功能 | 引脚 | 端口 | 模式 | 上下拉 | 速度 | .ioc 行 |
|------|------|------|------|--------|------|---------|
| **USER_BTN**（用户按键） | PC13 | GPIOC | GPIO_Input | 无上下拉 (GPIO_NOPULL) | — | [L58-61](../../GPIO_BUTTON.ioc#L58-L61) |
| **LD2**（板载绿色 LED） | PA5 | GPIOA | GPIO_Output (PP) | 无上下拉 (GPIO_NOPULL) | Low | [L54-57](../../GPIO_BUTTON.ioc#L54-L57) |
| T_SWDIO（调试） | PA13 | GPIOA | Serial_Wire (AF) | — | — | [L44-48](../../GPIO_BUTTON.ioc#L44-L48) |
| T_SWCLK（调试） | PA14 | GPIOA | Serial_Wire (AF) | — | — | [L49-53](../../GPIO_BUTTON.ioc#L49-L53) |

**时钟配置**（来源 [main.c:105-144](../../Core/Src/main.c#L105-L144)）：

- 时钟源：HSI（16 MHz）→ PLL（/4 × 85 = 340 MHz VCO）→ SYSCLK = 170 MHz
- HCLK = APB1 = APB2 = 170 MHz
- Flash Latency = 4
- SysTick 已使能（[.ioc:42](../../GPIO_BUTTON.ioc#L42)）

**BSP 配置**：

- BSP 已使能：`BSP_Common_DEMO=true`（[.ioc:31-32](../../GPIO_BUTTON.ioc#L31-L32)）
- COM 功能已禁用：`USE_BSP_COM_FEATURE=0U`（[stm32g4xx_nucleo_conf.h:53](../../Core/Inc/stm32g4xx_nucleo_conf.h#L53)）
- 按键中断优先级：15（最低）（[stm32g4xx_nucleo_conf.h:56](../../Core/Inc/stm32g4xx_nucleo_conf.h#L56)）

---

## 3. 已有初始化函数、宏定义及所在文件

### 3.1 宏定义 — [main.h:62-69](../../Core/Inc/main.h#L62-L69)

```c
#define USER_BTN_Pin         GPIO_PIN_13
#define USER_BTN_GPIO_Port   GPIOC
#define LD2_Pin              GPIO_PIN_5
#define LD2_GPIO_Port        GPIOA
```

### 3.2 BSP 层宏定义 — [stm32g4xx_nucleo.h](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h)

| 宏 | 值 | 行号 |
|----|-----|------|
| `LED2_PIN` | `GPIO_PIN_5` | [L174](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L174) |
| `LED2_GPIO_PORT` | `GPIOA` | [L175](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L175) |
| `USER_BUTTON_PIN` | `GPIO_PIN_13` | [L195](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L195) |
| `USER_BUTTON_GPIO_PORT` | `GPIOC` | [L196](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L196) |
| `BUTTON_RELEASED` | `0U` | [L189](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L189) |
| `BUTTON_PRESSED` | `1U` | [L190](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L190) |

### 3.3 初始化函数

| 函数 | 文件 | 功能 |
|------|------|------|
| `HAL_Init()` | （HAL 库） | 配置 Flash、NVIC 优先级分组、SysTick |
| `SystemClock_Config()` | [main.c:105](../../Core/Src/main.c#L105) | 配置 HSI+PLL → 170 MHz |
| `MX_GPIO_Init()` | [main.c:152](../../Core/Src/main.c#L152) | 初始化 PC13（输入，NOPULL）、PA5（输出 PP，初始低电平） |
| `HAL_MspInit()` | [stm32g4xx_hal_msp.c:63](../../Core/Src/stm32g4xx_hal_msp.c#L63) | 使能 SYSCFG/PWR 时钟，禁用 UCPD Dead Battery |

### 3.4 BSP 层 API — [stm32g4xx_nucleo.h:261-276](../../Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h#L261-L276)

| 函数 | 说明 |
|------|------|
| `BSP_PB_Init(Button, ButtonMode)` | 按键初始化（GPIO 或 EXTI 模式） |
| `BSP_PB_GetState(Button)` | 读取按键状态（返回 `int32_t`，0=释放，1=按下） |
| `BSP_LED_Init(Led)` | LED 初始化 |
| `BSP_LED_On(Led)` / `BSP_LED_Off(Led)` / `BSP_LED_Toggle(Led)` | LED 操作 |
| `BSP_PB_Callback(Button)` | 按键回调（`__weak`，需用户重写，用于 EXTI 模式） |

### 3.5 已有全局变量 — [main.c:34](../../Core/Src/main.c#L34)

```c
__IO uint32_t BspButtonState = BUTTON_RELEASED;
```

CubeMX BSP Demo 模式自动生成的变量，当前未被使用。

---

## 4. 实现功能需要使用的 HAL 函数

本次需求为**轮询方式**（主循环中读取按键状态），所需 HAL 函数：

| HAL 函数 | 用途 | 所在头文件 |
|----------|------|------------|
| `HAL_GPIO_ReadPin(GPIOx, GPIO_Pin)` | 读取按键引脚电平（返回 GPIO_PIN_SET/RESET） | stm32g4xx_hal_gpio.h |
| `HAL_GPIO_TogglePin(GPIOx, GPIO_Pin)` | 翻转 LED 引脚电平 | stm32g4xx_hal_gpio.h |
| `HAL_GPIO_WritePin(GPIOx, GPIO_Pin, state)` | 设置 LED 引脚电平 | stm32g4xx_hal_gpio.h |
| `HAL_GetTick()` | 获取 SysTick 计数值（用于软件去抖延时） | stm32g4xx_hal.h |

> **备选方案**：也可使用 BSP 层 API（`BSP_PB_GetState()` + `BSP_LED_Toggle()`）实现，无需直接调用 HAL GPIO 函数。两种方案均可，HAL 层更直接，BSP 层更抽象。

---

## 5. 需要修改的文件和代码位置

| 文件 | 修改位置 | 修改内容 |
|------|----------|----------|
| [Core/Src/main.c](../../Core/Src/main.c) | `/* USER CODE BEGIN PV */`（L36-38） | 新增软件去抖所需的状态变量（上次按键状态、去抖时间戳等） |
| [Core/Src/main.c](../../Core/Src/main.c) | `/* USER CODE BEGIN 2 */`（L81-83） | （可选）若使用 BSP API 初始化按键/LED |
| [Core/Src/main.c](../../Core/Src/main.c) | `/* USER CODE BEGIN 3 */`（L96-97） | 主循环中实现轮询读取按键、去抖判断、翻转 LED |

> 所有修改均在 CubeMX 用户代码区（`USER CODE BEGIN/END`）内，符合 CLAUDE.md 修改边界要求。

### 无需修改的文件

- `.ioc` 文件（不修改）
- `main.h`（已有足够宏定义）
- `stm32g4xx_hal_msp.c`（无需新增 MSP 初始化）
- `stm32g4xx_it.c`（轮询方式不需要中断服务函数）
- `stm32g4xx_nucleo.c` / `.h`（BSP 库文件不修改）

---

## 6. 是否需要新增变量、函数、文件或中断

| 类别 | 是否需要 | 说明 |
|------|----------|------|
| **新增变量** | ✅ 需要 | 在 `main.c` 用户变量区新增：上一次按键物理读数、去抖后的稳定状态、去抖时间戳 |
| **新增函数** | ❌ 不需要 | 逻辑简单，直接在 `while(1)` 内实现即可 |
| **新增文件** | ❌ 不需要 | 全部修改集中在 `main.c` |
| **新增中断** | ❌ 不需要 | 轮询方式不需要 EXTI 中断，SysTick 已有 |

### 状态机变量建议

```c
static uint8_t  last_reading    = 0;   // 上一次物理电平读数
static uint8_t  stable_state    = 0;   // 去抖后的稳定状态
static uint32_t last_debounce_t = 0;   // 上次电平变化的时间戳
```

---

## 7. 风险和注意事项

### 7.1 ✅ 按键引脚上下拉配置

- **MX_GPIO_Init()** 将 PC13 配置为 `GPIO_NOPULL`（[main.c:169](../../Core/Src/main.c#L169)）
- NUCLEO-G431RB 原理图：USER_BTN（PC13）在按键未按下时通过**外部下拉电阻**接地，按下时接通 VDD。外部下拉已提供确定的默认电平，因此 `GPIO_NOPULL` 配置正确，无需额外配置内部上下拉。
- BSP_PB_Init() 中的 `GPIO_PULLDOWN` 为 BSP 库的内部实现，与当前工程使用的 MX_GPIO_Init 配置无关。

### 7.2 软件去抖必要性

机械按键在按下/释放瞬间会产生电平抖动（典型 5-20 ms），必须在软件中做去抖处理。建议去抖延时 20-50 ms，用 `HAL_GetTick()` 获取时间基准。

### 7.3 "持续按住不重复切换"逻辑

需要实现**边沿检测**而非电平检测：仅在检测到"释放→按下"跳变时执行一次 LED 翻转。这需要记录上一次的稳定按键状态。

### 7.4 BSP Demo 模式

`.ioc` 中 `Bsp_Common_DEMO=true`（[L31](../../GPIO_BUTTON.ioc#L31)）表示 CubeMX 启用了 BSP 通用示例模式，这导致 `main.c` 中自动生成了 `BspButtonState` 变量。该变量当前未被使用，可保留或忽略。

### 7.5 EXTI 线已定义但未使能

BSP 中已定义 `USER_BUTTON_EXTI_IRQn = EXTI15_10_IRQn` 和 `USER_BUTTON_EXTI_LINE = EXTI_LINE_13`，但当前工程中该中断未使能（NVIC 中无对应配置，`stm32g4xx_it.c` 中无对应 ISR）。如未来改用中断方式，可直接使用 BSP 提供的 `BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI)` 和 `BSP_PB_Callback()` 实现，无需从头配置 EXTI。

### 7.6 无串口日志输出

`USE_BSP_COM_FEATURE=0U` 且 `USE_COM_LOG=0U`，LPUART1 未初始化。调试阶段若需要串口输出日志，需额外配置。

---

## 8. 实现任务清单

### 任务：实现按键轮询去抖与 LED 翻转逻辑

| 项目 | 内容 |
|------|------|
| **任务目标** | 在主循环中实现按键轮询：每 20-50 ms 检测一次按键状态，软件去抖后识别"按下"边沿事件，触发 LED2 状态翻转。持续按住不重复翻转，松开后恢复响应。 |
| **涉及文件** | [Core/Src/main.c](../../Core/Src/main.c) |
| **主要修改位置** | 1. `/* USER CODE BEGIN PD */`（L23-25）— 定义去抖延时宏 `DEBOUNCE_DELAY_MS`<br>2. `/* USER CODE BEGIN PV */`（L36-38）— 新增状态变量<br>3. `/* USER CODE BEGIN 3 */`（L96-97）— 主循环轮询逻辑 |
| **实施要点** | 1. **宏定义**（PD 区）：`#define DEBOUNCE_DELAY_MS 30U`<br>2. **状态变量**（PV 区）：<br>&nbsp;&nbsp;– `last_reading`：上一次物理电平读数（用于检测电平变化）<br>&nbsp;&nbsp;– `stable_state`：去抖后的稳定按键状态<br>&nbsp;&nbsp;– `last_debounce_time`：上次检测到电平变化时的 SysTick 时间戳<br>3. **轮询逻辑**（while(1) 内）：<br>&nbsp;&nbsp;a. 用 `HAL_GPIO_ReadPin(USER_BTN_GPIO_Port, USER_BTN_Pin)` 读取 PC13 当前物理电平<br>&nbsp;&nbsp;b. 若当前电平与 `last_reading` 不同，更新 `last_debounce_time = HAL_GetTick()`<br>&nbsp;&nbsp;c. 若 `HAL_GetTick() - last_debounce_time >= DEBOUNCE_DELAY_MS`，确认电平已稳定<br>&nbsp;&nbsp;d. 若稳定电平与 `stable_state` 不同且新状态为"按下"，执行 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)`<br>&nbsp;&nbsp;e. 更新 `stable_state` 和 `last_reading` |
| **验证方式** | 编译烧录后运行：<br>1. 短按按键 → LED2 翻转一次，再次短按 → LED2 再次翻转<br>2. 持续按住按键 → LED2 不重复翻转<br>3. 松开按键后再短按 → LED2 正常响应翻转<br>4. 快速连按多次 → 每次有效按下均应翻转 LED |
