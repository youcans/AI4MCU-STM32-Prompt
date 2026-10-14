# GPIO IOToggle —— 工程分析报告

> **硬件平台：** NUCLEO-G431RB  
> **目标芯片：** STM32G431RBT6 (LQFP64)  
> **CubeMX 版本：** 6.16.0  
> **固件包：** STM32Cube FW_G4 V1.6.3  
> **工具链：** CMake (GCC)

---

## 1. LED 引脚与 GPIO 配置

| 项目             | 详情                                          |
| :--------------- | :-------------------------------------------- |
| **板载 LED 编号** | LD2（绿色）                                   |
| **GPIO 引脚**     | PA5                                           |
| **信号名**        | `GPIO_Output`                                 |
| **宏定义**        | `LD2_Pin` = `GPIO_PIN_5`                      |
| **端口宏**        | `LD2_GPIO_Port` = `GPIOA`                     |
| **输出模式**      | 推挽输出 (`GPIO_MODE_OUTPUT_PP`)               |
| **上下拉**        | 无 (`GPIO_NOPULL`)                             |
| **输出速度**      | 低速 (`GPIO_SPEED_FREQ_LOW`)                   |
| **初始电平**      | 低电平 (`GPIO_PIN_RESET`)，LED 初始熄灭         |
| **配置来源**      | [Core/Src/main.c](../../Core/Src/main.c#L224-L238) —— `MX_GPIO_Init()` |

> **说明：** NUCLEO-G431RB 的 LD2 为低电平点亮（阴极接 PA5、阳极经限流电阻接 VDD），因此 `GPIO_PIN_RESET` 输出低电平时 LED 亮，`GPIO_PIN_SET` 输出高电平时 LED 灭。`HAL_GPIO_TogglePin()` 会在高低之间自动切换。

---

## 2. 相关宏定义

| 宏名称              | 值           | 定义位置                                        |
| :------------------ | :----------- | :---------------------------------------------- |
| `LD2_Pin`           | `GPIO_PIN_5` | [Core/Inc/main.h:72](../../Core/Inc/main.h#L72) |
| `LD2_GPIO_Port`     | `GPIOA`      | [Core/Inc/main.h:73](../../Core/Inc/main.h#L73) |
| `BTN_Pin`           | `GPIO_PIN_13` | [Core/Inc/main.h:62](../../Core/Inc/main.h#L62) |
| `BTN_GPIO_Port`     | `GPIOC`      | [Core/Inc/main.h:63](../../Core/Inc/main.h#L63) |

---

## 3. 关键函数

| 函数                          | 作用                                  | 定义位置 |
| :---------------------------- | :------------------------------------ | :------- |
| `HAL_Init()`                  | 初始化 HAL 库、配置 SysTick（1ms 时基） | [Core/Src/main.c:78](../../Core/Src/main.c#L78) |
| `SystemClock_Config()`        | 配置系统时钟（PLL, 170 MHz）           | [Core/Src/main.c:118](../../Core/Src/main.c#L118) |
| `MX_GPIO_Init()`              | 初始化 GPIO（含 PA5 推挽输出）          | [Core/Src/main.c:212](../../Core/Src/main.c#L212) |
| `HAL_GPIO_WritePin()`         | 设置 GPIO 输出电平（初始化用）           | HAL 库     |
| `HAL_GPIO_TogglePin()`        | **翻转 GPIO 输出电平（实现闪烁的核心）** | HAL 库     |
| `HAL_Delay()`                 | **毫秒级延时（依赖 SysTick）**          | HAL 库     |
| `HAL_IncTick()`               | SysTick 中断中递增全局 tick 计数器        | [Core/Src/stm32g4xx_it.c:188](../../Core/Src/stm32g4xx_it.c#L188) |
| `SysTick_Handler()`           | SysTick 中断服务函数（每 1ms 触发）       | [Core/Src/stm32g4xx_it.c:183](../../Core/Src/stm32g4xx_it.c#L183) |

---

## 4. 应用代码插入位置

| 位置           | 说明                                                  |
| :------------- | :---------------------------------------------------- |
| **`while(1)` 循环内** | 位于 `USER CODE BEGIN 3` / `USER CODE END 3` 之间 |

代码位于 [Core/Src/main.c:109](../../Core/Src/main.c#L109)（`/* USER CODE BEGIN 3 */` 区域）。在此处只需写入以下两行核心逻辑：

```c
HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
HAL_Delay(500);
```

> **注意：** CubeMX 重新生成代码时，`USER CODE` 区域内的用户代码会被保留。

---

## 5. 是否需要新增文件 / 变量 / 中断 / 外设

| 项目           | 是否需要 | 说明 |
| :------------- | :------- | :--- |
| **新增文件**   | 否       | 仅修改 `main.c` 中 `while(1)` 循环内的用户代码区即可 |
| **新增全局变量** | 否     | 无需额外变量 |
| **新增中断**   | 否       | SysTick 已由 CubeMX 配置并使能，`HAL_Delay()` 直接可用 |
| **新增外设配置** | 否     | PA5 已在 `MX_GPIO_Init()` 中配置为推挽输出 |
| **新增定时器** | 否       | 不需要 TIM 外设；`HAL_Delay(500)` 基于 SysTick 实现 500ms 延时 |
| **GPIO 时钟**  | 已使能   | `__HAL_RCC_GPIOA_CLK_ENABLE()` 已在 `MX_GPIO_Init()` 中调用 |

---

## 6. 闪烁逻辑流程

```
main()
 ├── HAL_Init()              ← SysTick 1ms 时基就绪
 ├── SystemClock_Config()    ← 170 MHz
 ├── MX_GPIO_Init()          ← PA5 初始化为推挽输出、低电平 (LED 初始亮)
 ├── MX_LPUART1_UART_Init()
 └── while (1)
      ├── HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)   ← 翻转 PA5
      └── HAL_Delay(500)                                ← 等待 500 ms
```

> **时序：** 每个循环周期约 500 ms，LED 以 500 ms 间隔亮灭交替（周期 1 s，频率 1 Hz）。

---

## 7. 注意事项

1. **LED 点亮逻辑：** 由于 NUCLEO-G431RB 的 LD2 为低电平点亮（PA5 = LOW → LED ON；PA5 = HIGH → LED OFF），初始 `HAL_GPIO_WritePin(..., GPIO_PIN_RESET)` 实际上使 LED 在程序启动时立即点亮。`HAL_GPIO_TogglePin()` 将在此基础上来回翻转。

2. **延时精度：** `HAL_Delay(500)` 基于 SysTick 1ms 中断，实际延时精度约 ±1ms，足够满足 500ms 闪烁需求。

3. **CubeMX 重新生成：** 用户代码应严格写在 `USER CODE BEGIN` / `USER CODE END` 标记之间，否则重新生成时会被覆盖。
