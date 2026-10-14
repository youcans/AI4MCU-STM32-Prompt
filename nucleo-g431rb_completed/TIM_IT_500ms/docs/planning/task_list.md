# 任务清单：TIM6 中断控制 LED2 周期闪烁

## 任务：TIM6 中断控制 LED2 周期闪烁

- **任务目标**：使用 TIM6 基本定时器中断，控制板载 LED2 每 500 ms 翻转一次输出状态，实现亮 500 ms、灭 500 ms 的周期闪烁（1 Hz）。
- **涉及文件**：[Core/Src/main.c](../../Core/Src/main.c)
- **主要修改位置**：
  1. `/* USER CODE BEGIN 2 */`（第 94–96 行之间）—— 启动 TIM6 并使能中断。
  2. `/* USER CODE BEGIN 4 */`（第 235–237 行之间）—— 实现 `HAL_TIM_PeriodElapsedCallback` 回调函数，在其中翻转 LED2 输出。

### 实施要点

1. **启动定时器**：在 `MX_TIM6_Init()` 之后、`while(1)` 之前，调用 `HAL_TIM_Base_Start_IT(&htim6)` 启动 TIM6 并使其能更新中断。
2. **实现回调**：重写 HAL 库弱回调 `HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)`，在函数内判断 `htim->Instance == TIM6`，确认是 TIM6 溢出后调用 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)` 翻转 LED2。回调中仅做翻转，不调用耗时操作。
3. **代码位置**：以上两处修改均在 CubeMX USER CODE 区域内，重新生成时不会被覆盖。

### 验证方式

1. **编译**：修改后编译工程，应 0 错误 0 警告。
2. **烧录运行**：将固件烧录至 NUCLEO-G431RB 开发板，观察板载 LED2（绿色），应持续以亮 500 ms、灭 500 ms 交替闪烁（频率 1 Hz）。
3. **（可选）测量**：有示波器或逻辑分析仪时，测量 PA5 引脚输出电平，翻转周期应为 500 ms。
