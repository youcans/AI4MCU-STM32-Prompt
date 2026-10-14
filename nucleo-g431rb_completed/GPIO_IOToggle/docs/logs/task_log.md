# 任务日志

## 2026-07-21 — LED 闪烁间隔调整

| 项目 | 修改前 | 修改后 |
|:-----|:-------|:-------|
| **文件** | `Core/Src/main.c` 第 110 行 | — |
| **延时值** | `HAL_Delay(500)` | `HAL_Delay(200)` |
| **闪烁周期** | 1 s（1 Hz） | 400 ms（2.5 Hz） |

### 修改位置

```c
/* USER CODE BEGIN 3 */
HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
HAL_Delay(200);   // ← 500 → 200
/* USER CODE END 3 */
```

> 仅修改了 `main.c` 中一个数值（110 行），其余文件均未改动。
