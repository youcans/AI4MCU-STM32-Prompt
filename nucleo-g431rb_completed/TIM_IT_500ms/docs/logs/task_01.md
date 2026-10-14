# 任务日志：TIM6 中断控制 LED2 周期闪烁

- **日期**：2026-07-23
- **任务名称**：TIM6 中断控制 LED2 周期闪烁

## 任务目标

使用 TIM6 基本定时器中断，控制板载 LED2 每 500ms 翻转一次输出状态，实现 1 Hz 周期闪烁。

## 修改文件

| 文件 | 修改位置 | 说明 |
|---|---|---|
| `Core/Src/main.c` | `/* USER CODE BEGIN 2 */`（第 95 行） | 新增 `HAL_TIM_Base_Start_IT(&htim6);` |
| `Core/Src/main.c` | `/* USER CODE BEGIN 4 */`（第 237–246 行） | 实现 `HAL_TIM_PeriodElapsedCallback()` 回调函数 |

## 实现逻辑

1. **启动定时器**：在 `MX_TIM6_Init()` 之后调用 `HAL_TIM_Base_Start_IT(&htim6)`，启动 TIM6 并使能更新中断。TIM6 配置为 PSC=1599、ARR=49999，APB1 Timer Clock = 160 MHz，溢出周期 = 500ms。

2. **中断回调**：重写 HAL 库 `__weak` 函数 `HAL_TIM_PeriodElapsedCallback()`，判断 `htim->Instance == TIM6` 后调用 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)` 翻转 PA5（LD2）输出。

3. **中断调用链**（已存在，未修改）：
   ```
   TIM6 溢出 → TIM6_DAC_IRQHandler() → HAL_TIM_IRQHandler(&htim6)
   → HAL_TIM_PeriodElapsedCallback()  ← 本次实现
   ```

## 验证方式

1. 编译工程，应 0 错误 0 警告。
2. 烧录至 NUCLEO-G431RB，观察 LD2（绿色）以亮 500ms、灭 500ms 交替闪烁。
