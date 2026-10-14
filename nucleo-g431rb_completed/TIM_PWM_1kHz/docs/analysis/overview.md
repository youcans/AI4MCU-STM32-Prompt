# 工程分析报告 — TIM2 PWM LED2 呼吸灯

> 分析日期：2026-07-24

---

## 1. 开发板和MCU型号

| 项目 | 内容 |
|------|------|
| **开发板** | NUCLEO-G431RB |
| **MCU型号** | STM32G431RBT6 |
| **MCU系列** | STM32G4 |
| **封装** | LQFP64 |
| **内核** | Cortex-M4 |
| **CubeMX版本** | 6.16.0 |
| **HAL库版本** | STM32Cube FW_G4 V1.6.3 |

---

## 2. 相关外设、端口和引脚配置

### 系统时钟

- 时钟源：HSI (16 MHz) → PLL (×20, /2) → **SYSCLK = 160 MHz**
- APB1 定时器时钟 = **160 MHz**（不分频）
- APB2 定时器时钟 = 160 MHz

### TIM2 定时器 (PWM)

| 参数 | 配置值 | 说明 |
|------|--------|------|
| 通道 | TIM2_CH1 | PWM Generation1 CH1 |
| 引脚 | **PA5** | 复用功能 AF1 (TIM2_CH1) |
| GPIO 模式 | `GPIO_MODE_AF_PP` | 复用推挽输出 |
| 上下拉 | `GPIO_NOPULL` | 无上下拉 |
| 速度 | `GPIO_SPEED_FREQ_LOW` | 低速 |
| 时钟源 | Internal | 内部时钟（APB1 Timer Clock = 160 MHz） |
| 预分频器 (PSC) | **159** | PSC = 159 |
| 计数器模式 | `TIM_COUNTERMODE_UP` | 向上计数 |
| 周期 (ARR) | **999** | Period = 999 |
| 初始占空比 (CCR1) | **100** | Pulse = 100 |
| 输出比较模式 | `TIM_OCMODE_PWM1` | PWM 模式 1 |
| 输出极性 | `TIM_OCPOLARITY_HIGH` | 高电平有效 |
| 自动重载预装载 | DISABLE | 关闭 |
| 主从模式 | `TIM_MASTERSLAVEMODE_DISABLE` | 关闭 |

### PWM 频率计算验证

```
TIM2_CLK   = APB1 Timer Clock = 160 MHz
PWM 频率   = TIM2_CLK / [(PSC + 1) × (ARR + 1)]
           = 160,000,000 / [160 × 1000]
           = 160,000,000 / 160,000
           = 1000 Hz = 1 kHz  ✓
```

### LED2 与 PA5 的关系

根据 BSP 定义（`Drivers/BSP/STM32G4xx_Nucleo/stm32g4xx_nucleo.h`）：

- `LED2_PIN` = `GPIO_PIN_5`
- `LED2_GPIO_PORT` = `GPIOA`

NUCLEO-G431RB 的 **板载 LED2 连接在 PA5**，恰好是 TIM2_CH1 的输出引脚。

---

## 3. 已有初始化函数、宏定义及所在文件

### `Core/Inc/main.h` — 主头文件

| 内容 | 说明 |
|------|------|
| `#include "stm32g4xx_hal.h"` | HAL 库主头文件 |
| `#include "stm32g4xx_nucleo.h"` | Nucleo BSP 头文件 |
| `void HAL_TIM_MspPostInit(...)` | MSP 后初始化声明 |
| `void Error_Handler(void)` | 错误处理声明 |
| `#define T_SWDIO_Pin` | SWD 调试引脚宏（PA13） |
| `#define T_SWDIO_GPIO_Port` | GPIOA |
| `#define T_SWCLK_Pin` | SWD 时钟引脚宏（PA14） |
| `#define T_SWCLK_GPIO_Port` | GPIOA |

### `Core/Src/main.c` — 主程序文件

| 内容 | 行号 | 说明 |
|------|------|------|
| `TIM_HandleTypeDef htim2` | 45 | **全局** TIM2 句柄，应用代码可直接使用 |
| `void SystemClock_Config(void)` | 118-158 | 系统时钟配置：HSI→PLL→160 MHz |
| `static void MX_TIM2_Init(void)` | 165-217 | TIM2 PWM 初始化：PSC=159, ARR=999, CCR1=100 |
| `static void MX_GPIO_Init(void)` | 224-236 | GPIO 时钟使能 |
| `int main(void)` | 68-112 | 主函数，包含用户代码区 |

### `Core/Src/stm32g4xx_hal_msp.c` — MSP 初始化

| 内容 | 说明 |
|------|------|
| `HAL_MspInit()` | 全局 MSP 初始化 |
| `HAL_TIM_Base_MspInit()` | TIM2 时钟使能：`__HAL_RCC_TIM2_CLK_ENABLE()` |
| `HAL_TIM_MspPostInit()` | **PA5 GPIO 配置为 AF1_TIM2** |

### `Core/Src/stm32g4xx_it.c` — 中断服务

- `SysTick_Handler()`：调用 `HAL_IncTick()`，已配置
- **无 TIM2 中断处理函数**（未开启 TIM2 中断）

### `Core/Inc/stm32g4xx_hal_conf.h`

| 宏定义 | 状态 |
|--------|------|
| `HAL_TIM_MODULE_ENABLED` | ✅ 已启用 |
| `HAL_GPIO_MODULE_ENABLED` | ✅ 已启用 |
| `HAL_DMA_MODULE_ENABLED` | ✅ 已启用 |
| `HAL_RCC_MODULE_ENABLED` | ✅ 已启用 |
| `HAL_PWR_MODULE_ENABLED` | ✅ 已启用 |
| `HAL_CORTEX_MODULE_ENABLED` | ✅ 已启用 |

### `Core/Inc/stm32g4xx_nucleo_conf.h`

- `#define USE_STM32G4XX_NUCLEO`：使能 NUCLEO 板支持
- `USE_BSP_COM_FEATURE` = 0：不使用串口 COM 功能

---

## 4. 实现功能需要使用的 HAL 函数

| 步骤 | HAL 函数 / 宏 | 作用 |
|------|---------------|------|
| **1. 启动 PWM** | `HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1)` | 启动 TIM2_CH1 的 PWM 输出 |
| **2. 调节占空比** | `__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, value)` | 更新 CCR1 寄存器的比较值（推荐用宏，比函数调用快） |
| **3. 延时** | `HAL_Delay(ms)` | 在占空比变化之间插入延时，控制亮度变化速率 |
| **或 2. 替代方案** | `HAL_TIM_PWM_Stop(&htim2, TIM_CHANNEL_1)` | 停止 PWM（如需要） |

> **关键点**：`MX_TIM2_Init()` 只完成了 TIM2 的 Base 初始化和 PWM 配置初始化，但**没有调用 `HAL_TIM_PWM_Start()` 启动 PWM 输出**。需要在应用代码中手动调用。

> **CCR 范围**：0 ~ 999（与 ARR 对应）。初始 CCR1 = 100，对应约 10% 占空比。

---

## 5. 需要修改的文件和代码位置

| 文件 | 修改位置 | 说明 |
|------|----------|------|
| `Core/Src/main.c` | `/* USER CODE BEGIN 2 */` | 调用 `HAL_TIM_PWM_Start()` 启动 PWM |
| `Core/Src/main.c` | `/* USER CODE BEGIN WHILE */` 和 `/* USER CODE BEGIN 3 */` | 在 while(1) 循环中周期性调节占空比并延时 |

**不需要修改的文件：**

- ❌ `MX_TIM2_Init()` — PWM 参数已正确配置
- ❌ `stm32g4xx_hal_msp.c` — GPIO 复用和时钟已配置
- ❌ `stm32g4xx_it.c` — 不使用中断
- ❌ `.ioc` 文件 — 当前配置已满足需求

---

## 6. 是否需要新增变量、函数、文件或中断

| 类别 | 是否需要 | 说明 |
|------|----------|------|
| **全局变量** | 可选 | 可新增一个 `uint16_t` 变量记录当前 CCR 值，用于递增/递减判断 |
| 新增方向标志 | 可选 | 一个 `uint8_t` 标志记录当前是"增亮"还是"变暗"阶段 |
| **新函数** | 不需要 | 逻辑简单，直接在 main() 的 while 循环中实现 |
| **新文件** | 不需要 | 仅修改 `main.c` 即可 |
| **TIM2 中断** | 不需要 | 使用软件延时 + 轮询方式即可实现，`HAL_Delay` 基于 SysTick |
| **DMA** | 不需要 | 占空比更新直接写寄存器即可 |

---

## 7. 风险与注意事项

| # | 风险/注意事项 | 详细说明 |
|---|--------------|----------|
| **1** | **LED2 = PA5 = TIM2_CH1 资源冲突** | PA5 同时被配置为 TIM2_CH1 的 AF 输出和 BSP 的 LED2 GPIO 输出。当前 `MX_TIM2_Init()` 之后 PA5 被初始化为 AF 模式（复用推挽），而 `BSP_LED_Init()` 会将其初始化为 GPIO 推挽输出。两者互相冲突。**不能同时使用 BSP_LED 函数和 TIM2 PWM**。如果调用了 `BSP_LED_Init(LED2)`，PA5 将脱离 AF 模式，PWM 无法输出。 |
| **2** | **BSP Demo 模式开启** | `.ioc` 中 `NUCLEO-G431RB.Bsp_Common_DEMO=true`，表示当前工程启用了 BSP 演示。需确认 DEMO 代码是否已经在 main() 中调用 `BSP_LED_Init()` 等函数。当前 main.c 中 `/* USER CODE BEGIN BSP */` 区域为空，说明 DEMO 代码尚未插入。 |
| **3** | **SysTick 依赖** | `HAL_Delay()` 依赖 SysTick 中断（1ms 周期），需确保 `HAL_Init()` 已调用且 SysTick 中断正常工作（当前工程已正确配置）。 |
| **4** | **PWM 初始状态** | `MX_TIM2_Init()` 后 PWM **尚未启动**。调用 `HAL_TIM_PWM_Start()` 前 PA5 不会输出 PWM 波形，LED 不亮或处于不确定状态。 |
| **5** | **占空比范围** | CCR 有效值为 0~999。0 对应完全关断（最暗），999 对应接近 100%（最亮）。初始 CCR=100（约 10%）。占空比步进值应合理选择（例如 ±10~50/步），配合适当延时（例如 10~50ms），使亮度变化平滑可见。 |
| **6** | **HSI 时钟精度** | 系统使用内部 HSI 作为时钟源（非外部晶振 HSE），HSI 标称精度约 ±1%，实际 PWM 频率可能有微小偏差，但这对 LED 亮度控制没有实际影响。 |
| **7** | **编译器优化级别较高** | `.ioc` 中 `CompilerOptimize=6`（`-Os` 优化体积）。变量如果不用 `volatile` 修饰，在优化级别下可能被优化掉。建议在 while 循环中使用的 PWM 比较值变量加上 `volatile`。 |

---

## 总结

当前工程**已具备实现 PWM LED 呼吸灯的基本硬件配置**：

- TIM2_CH1 通过 PA5 输出 PWM，频率正好 1 kHz
- LED2 连接在 PA5，两者电气上是同一引脚
- 只需在 `main.c` 的用户代码区（`USER CODE BEGIN 2` 和 `USER CODE BEGIN 3`）添加 PWM 启动和占空比调节逻辑即可
- **核心风险是 PA5 的 AF 模式与 BSP GPIO 模式的冲突**，必须避免调用 `BSP_LED_Init()` 等会覆盖 PA5 模式的函数
