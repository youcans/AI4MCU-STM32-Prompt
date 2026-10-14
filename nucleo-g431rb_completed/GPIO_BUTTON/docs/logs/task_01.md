# 任务日志：按键轮询去抖与 LED 翻转逻辑

> 完成日期：2026-07-22

---

## 任务目标

在主循环中轮询读取 NUCLEO-G431RB 开发板上的用户按键（USER_BTN，PC13），使用 `HAL_GetTick()` 实现非阻塞软件去抖，识别有效的按下事件并切换板载 LED2（PA5）状态；持续按住按键时不重复切换，松开后可以响应下一次按键操作。不使用 `HAL_Delay()` 阻塞主循环。

---

## 修改文件

| 文件 | 修改位置 | 行号 |
|------|----------|------|
| `Core/Src/main.c` | `/* USER CODE BEGIN PD */` | L24 |
| `Core/Src/main.c` | `/* USER CODE BEGIN PV */` | L37-39 |
| `Core/Src/main.c` | `/* USER CODE BEGIN 3 */` | L99-118 |

---

## 主要修改

### 1. 宏定义 — `/* USER CODE BEGIN PD */`（L24）

```c
#define DEBOUNCE_DELAY_MS 30U
```

软件去抖延时 30 ms，覆盖机械按键典型抖动（5-20 ms）。

### 2. 状态变量 — `/* USER CODE BEGIN PV */`（L37-39）

```c
static uint8_t  last_reading    = 0;
static uint8_t  stable_state    = 0;
static uint32_t last_debounce_t = 0;
```

| 变量 | 用途 |
|------|------|
| `last_reading` | 上一次物理电平读数，用于检测电平边沿变化 |
| `stable_state` | 去抖后的稳定按键状态，用于边沿检测的参考 |
| `last_debounce_t` | 上次检测到电平变化时的 `HAL_GetTick()` 时间戳 |

### 3. 轮询逻辑 — `/* USER CODE BEGIN 3 */`（L99-118）

```c
uint8_t reading = (uint8_t)HAL_GPIO_ReadPin(USER_BTN_GPIO_Port, USER_BTN_Pin);

if (reading != last_reading)
{
    last_debounce_t = HAL_GetTick();
}

if ((HAL_GetTick() - last_debounce_t) >= DEBOUNCE_DELAY_MS)
{
    if (reading != stable_state)
    {
        stable_state = reading;
        if (stable_state == GPIO_PIN_SET)
        {
            HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
        }
    }
}

last_reading = reading;
```

---

## 实现逻辑

```
while(1) 每次循环：
  1. HAL_GPIO_ReadPin() 读取 PC13 当前物理电平
  2. 若电平与 last_reading 不同 → 记录当前时间戳（重置去抖窗口）
  3. 若距离上次变化 ≥ 30ms（去抖窗口已过）：
     a. 若稳定电平与 reading 不同 → 更新 stable_state
     b. 若新状态为"按下"(GPIO_PIN_SET) → HAL_GPIO_TogglePin 翻转 LED
     c. 否则不做任何操作
  4. 更新 last_reading = reading
```

- **非阻塞**：全程使用 `HAL_GetTick()` 时间比较，不调用 `HAL_Delay()`
- **边沿检测**：仅在 `stable_state` 从"释放"变为"按下"时触发一次 LED 翻转
- **去抖**：电平变化后等待 30ms 窗口期，窗口内不响应

---

## 验证结果

| 测试场景 | 预期行为 | 结果 |
|----------|----------|------|
| 短按按键 | LED2 翻转一次 | 通过 |
| 再次短按 | LED2 再次翻转 | 通过 |
| 持续按住 | LED2 不重复翻转 | 通过 |
| 松开后再短按 | LED2 正常响应 | 通过 |
| 快速连按多次 | 每次有效按下均翻转 | 通过 |

---

## 相关文档

- 工程分析：[docs/analysis/overview.md](../analysis/overview.md)
- 任务清单：[docs/planning/task_list.md](../planning/task_list.md)
