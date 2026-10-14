# 任务清单

> 功能目标：按下 NUCLEO-G431RB 用户按键触发外部中断，切换板载 LED2 状态
> 工程分析：参见 [docs/analysis/overview.md](../analysis/overview.md)

---

## 任务 1：实现按键中断回调（含去抖动），切换 LED 状态

| 项目 | 内容 |
|---|---|
| **任务目标** | 重写 `HAL_GPIO_EXTI_Callback`，使得按下 USER_BTN（PC13）时能够可靠地切换 LD2（PA5）的亮灭状态，包含按键去抖动处理。 |
| **涉及文件** | [Core/Src/main.c](../../Core/Src/main.c) |
| **主要修改位置** | 两处：(1) `/* USER CODE BEGIN PV */`（第 36-38 行）声明去抖所需的静态变量；(2) `/* USER CODE BEGIN 4 */`（第 188-190 行）编写 `HAL_GPIO_EXTI_Callback` 函数。 |
| **实施要点** | 1. 在 `USER CODE BEGIN PV` 区声明 `static uint32_t last_tick` 用于记录上次有效触发时刻；<br>2. 在 `USER CODE BEGIN 4` 区重写 `void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`，内部：(a) 判断 `GPIO_Pin == USER_BTN_Pin`；(b) 通过 `HAL_GetTick()` 获取当前时刻，若与 `last_tick` 之差小于阈值（如 50ms）则忽略本次中断；(c) 更新 `last_tick` 并调用 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)`。 |
| **验证方式** | 编译通过后烧录到 NUCLEO-G431RB：(1) 按下用户按键 B1，LD2 在亮/灭之间切换；(2) 快速连续按压，每次按下 LD2 仅切换一次，无抖动误触发。 |

---

> 说明：工程所有外设初始化已在 `MX_GPIO_Init()` 中完成，中断向量已在 `stm32g4xx_it.c` 中连接。无需新增文件、无需修改 `.ioc`、无需修改 BSP/HAL 库文件。全部修改在 CubeMX 用户代码区内完成。
