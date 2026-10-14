# 实现任务清单

> 基于工程分析报告 [docs/analysis/overview.md](../analysis/overview.md)
>
> 目标功能：主循环轮询读取 NUCLEO-G431RB 用户按键（USER_BTN），每检测到一次有效按键操作，切换板载 LED2 状态；持续按住不重复切换，松开后可响应下一次操作。

---

## 任务：实现按键轮询去抖与 LED 翻转逻辑

| 项目 | 内容 |
|------|------|
| **任务目标** | 在主循环中实现按键轮询：每 20-50 ms 检测一次按键状态，软件去抖后识别"按下"边沿事件，触发 LED2 状态翻转。持续按住不重复翻转，松开后恢复响应。 |
| **涉及文件** | [Core/Src/main.c](../../Core/Src/main.c) |
| **主要修改位置** | 1. `/* USER CODE BEGIN PD */`（L23-25）— 定义去抖延时宏 `DEBOUNCE_DELAY_MS`<br>2. `/* USER CODE BEGIN PV */`（L36-38）— 新增状态变量<br>3. `/* USER CODE BEGIN 3 */`（L96-97）— 主循环轮询逻辑 |
| **实施要点** | 1. **宏定义**（PD 区）：`#define DEBOUNCE_DELAY_MS 30U`<br>2. **状态变量**（PV 区）：<br>&nbsp;&nbsp;– `last_reading`：上一次物理电平读数（用于检测电平变化）<br>&nbsp;&nbsp;– `stable_state`：去抖后的稳定按键状态<br>&nbsp;&nbsp;– `last_debounce_time`：上次检测到电平变化时的 SysTick 时间戳<br>3. **轮询逻辑**（while(1) 内）：<br>&nbsp;&nbsp;a. 用 `HAL_GPIO_ReadPin(USER_BTN_GPIO_Port, USER_BTN_Pin)` 读取 PC13 当前物理电平<br>&nbsp;&nbsp;b. 若当前电平与 `last_reading` 不同，更新 `last_debounce_time = HAL_GetTick()`<br>&nbsp;&nbsp;c. 若 `HAL_GetTick() - last_debounce_time >= DEBOUNCE_DELAY_MS`，确认电平已稳定<br>&nbsp;&nbsp;d. 若稳定电平与 `stable_state` 不同且新状态为"按下"，执行 `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin)`<br>&nbsp;&nbsp;e. 更新 `stable_state` 和 `last_reading` |
| **验证方式** | 编译烧录后运行：<br>1. 短按按键 → LED2 翻转一次，再次短按 → LED2 再次翻转<br>2. 持续按住按键 → LED2 不重复翻转<br>3. 松开按键后再短按 → LED2 正常响应翻转<br>4. 快速连按多次 → 每次有效按下均应翻转 LED |
