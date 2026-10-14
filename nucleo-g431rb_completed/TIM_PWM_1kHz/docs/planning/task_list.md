# 任务清单 — TIM2 PWM LED2 呼吸灯

> 日期：2026-07-24
> 参考分析：docs/analysis/overview.md

---

根据工程分析结果，该功能可拆分为 **2 项实现任务**：

---

## 任务清单

### 任务 1：启动 TIM2_CH1 PWM 输出

**任务目标**

在系统初始化完成后启动 TIM2 通道 1 的 PWM 输出，使 LED2 以固定初始占空比点亮，验证 PWM 硬件通路正常。

**涉及文件**

`Core/Src/main.c`

**主要修改位置**

- `/* USER CODE BEGIN 2 */` — `/* USER CODE END 2 */` 之间（第 94-96 行）

**实施要点**

1. 调用 `HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1)` 启动 PWM 输出。
2. 确认全局句柄 `htim2` 已由 `MX_TIM2_Init()` 完成初始化（PSC=159, ARR=999, CCR1=100），无需重复配置。
3. 注意 `htim2` 为文件级全局变量（`main.c:45`），可直接引用。
4. **禁止**调用 `BSP_LED_Init()` 或任何操作 PA5 为 GPIO 的函数——PA5 已由 `HAL_TIM_MspPostInit()` 配置为 AF1_TIM2 复用输出模式。

**验证方式**

编译通过后烧录运行，观察开发板上 **LED2 以约 10% 亮度（较暗）常亮**。若 LED2 不亮，检查 PA5 是否被后续代码覆盖为 GPIO 模式。

---

### 任务 2：实现呼吸灯占空比渐变循环

**任务目标**

在 while(1) 主循环中以非阻塞方式实现占空比周期性渐变：占空比在约 10% ~ 90% 之间往复变化，连续循环，使 LED2 呈现呼吸灯效果。

**涉及文件**

`Core/Src/main.c`

**主要修改位置**

- `/* USER CODE BEGIN PD */` — `/* USER CODE END PD */` 之间（第 33-35 行）— 添加参数常量宏定义
- `/* USER CODE BEGIN PV */` — `/* USER CODE END PV */` 之间（第 47-48 行）— 添加私有变量
- `/* USER CODE BEGIN 3 */` — `/* USER CODE END 3 */` 之间（第 108-109 行）— 添加呼吸逻辑

**实施要点**

1. 在 `/* USER CODE BEGIN PD */` 中定义参数常量：
   ```c
   #define PWM_PULSE_MIN     100U   /* 最暗, 约 10% 占空比 */
   #define PWM_PULSE_MAX     900U   /* 最亮, 约 90% 占空比 */
   #define PWM_PULSE_STEP    5U     /* 每步变化量 */
   #define PWM_UPDATE_MS     10U    /* 更新间隔 (ms) */
   ```

2. 在 `/* USER CODE BEGIN PV */` 中声明变量（仅在主循环中访问，无需 `volatile`）：
   - `uint16_t pwm_pulse`：当前 CCR 比较值，初始值为 `PWM_PULSE_MIN`（100）。
   - `uint8_t direction`：方向标志，0 = 递增（变亮），1 = 递减（变暗），初始为 0。
   - `uint32_t last_tick`：上次更新时间戳，初始值为 0。

3. 在 while(1) 循环中实现非阻塞更新逻辑：
   - 调用 `HAL_GetTick()` 获取当前系统 tick。
   - 判断当前 tick 与 `last_tick` 之差是否 >= `PWM_UPDATE_MS`：
     - 若未到 10 ms，跳过本次更新，继续下一次循环。
     - 若已到 10 ms，更新 `last_tick`，根据 `direction` 调整 `pwm_pulse`：
       - 递增阶段：`pwm_pulse += PWM_PULSE_STEP`；若 >= `PWM_PULSE_MAX`，翻转为递减。
       - 递减阶段：`pwm_pulse -= PWM_PULSE_STEP`；若 <= `PWM_PULSE_MIN`，翻转为递增。
     - 调用 `__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pwm_pulse)` 更新占空比。
   - 主循环中不调用 `HAL_Delay()`，保持非阻塞。

4. 一次完整呼吸周期（100→900→100）约 3.2 秒，视觉效果平滑自然。

**验证方式**

编译通过后烧录运行，观察开发板上 **LED2 亮度在较暗和较亮之间平滑渐变，持续循环**。

---

### 任务关系

```
任务 1 (PWM 启动)  →  任务 2 (呼吸循环)
     ↑                      ↑
   可独立验证             依赖任务 1 完成
```

任务 1 完成后即可看到 LED 点亮（常亮），验证 PWM 通路。任务 2 在任务 1 的基础上实现完整呼吸灯效果。
