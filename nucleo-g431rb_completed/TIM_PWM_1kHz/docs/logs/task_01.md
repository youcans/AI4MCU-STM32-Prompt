# 任务日志 — 实现呼吸灯占空比渐变循环

> 日期：2026-07-25
> 对应任务清单：任务 2 — 实现呼吸灯占空比渐变循环

---

## 任务目标

在 TIM2 通道 1 持续输出 PWM 信号的过程中，以非阻塞方式周期性调节比较值，使 PWM 占空比在约 10%～90% 之间往复变化，从而使板载 LED2 亮度在较暗和较亮状态之间逐渐变化并连续循环。

---

## 修改文件

`Core/Src/main.c`（仅此一个文件）

## 修改位置与内容

### 1. `/* USER CODE BEGIN PD */` — 参数常量宏定义

```c
#define PWM_PULSE_MIN     100U   /* 最暗, 约 10% 占空比 */
#define PWM_PULSE_MAX     900U   /* 最亮, 约 90% 占空比 */
#define PWM_PULSE_STEP    5U     /* 每步变化量 */
#define PWM_UPDATE_MS     10U    /* 更新间隔 (ms) */
```

### 2. `/* USER CODE BEGIN PV */` — 私有变量

```c
static uint16_t pwm_pulse  = PWM_PULSE_MIN;  /* 当前比较值 (100..900) */
static uint8_t  direction  = 0;              /* 0=增亮, 1=变暗        */
static uint32_t last_tick  = 0;              /* 上次更新时间戳         */
```

### 3. `/* USER CODE BEGIN 3 */` — 非阻塞呼吸逻辑

```c
uint32_t now = HAL_GetTick();
if ((now - last_tick) >= PWM_UPDATE_MS)
{
    last_tick = now;

    if (direction == 0)
    {
        pwm_pulse += PWM_PULSE_STEP;
        if (pwm_pulse >= PWM_PULSE_MAX)
        {
            pwm_pulse = PWM_PULSE_MAX;
            direction = 1;
        }
    }
    else
    {
        pwm_pulse -= PWM_PULSE_STEP;
        if (pwm_pulse <= PWM_PULSE_MIN)
        {
            pwm_pulse = PWM_PULSE_MIN;
            direction = 0;
        }
    }
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pwm_pulse);
}
```

---

## 实现逻辑

1. 使用 `HAL_GetTick()` 获取系统毫秒时间戳，每次循环判断是否距上次更新已过 10 ms。
2. 未到 10 ms 则跳过，继续下一轮循环——不阻塞 while(1)。
3. 到达 10 ms 时，根据 `direction` 方向将 `pwm_pulse` 递增或递减 `5`，并通过 `__HAL_TIM_SET_COMPARE` 立即写入 CCR1 寄存器更新占空比。
4. 当比较值触及上限（900 ≈ 90%）或下限（100 ≈ 10%）时翻转方向，实现往复渐变。
5. 一个完整呼吸周期（100→900→100）共 320 步，每步 10 ms，约 3.2 秒。

---

## 验证结果

待编译烧录运行验证。
