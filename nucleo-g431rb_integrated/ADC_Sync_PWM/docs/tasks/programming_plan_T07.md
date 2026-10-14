# T07 实现方案 — 实现 MD-04 系统状态机与周期控制

> 编制日期：2026-08-08
> 修订日期：2026-08-25（校订版；按开发者审阅意见：温度比较用 > / <，§5.2 阈值字样统一为 45°C/40°C；system_state/led_div_counter 加 volatile）
> 依据文件：
> - `docs/tasks/programming_task_list.md` §T07
> - `docs/design/software_detailed_design.md`
> - `docs/logs/task_log.md`（T05/T06 执行记录）

---

## 1. 任务实现分析

### 1.1 当前状态

T06 已实施并板级验证通过（2026-08-25）：500 Hz 时基、规则组温度采样、按键启停三条链路均正常。当前 `app_control.c` 中仅实现了 STOP/RUN 两状态，500 Hz 回调使用辅助计数变量 `timer_500hz_count`，温度判断为空壳，LED 翻转仍为 T04 辅助代码（无条件翻转）。本任务将其全部替换为正式状态机逻辑。

### 1.2 需要替换的 T04/T06 辅助代码

| 函数 | T06 状态 | T07 目标 |
|---|---|---|
| `HAL_TIM_PeriodElapsedCallback` | `timer_500hz_count++` | 完整 500 Hz 调度（DAC 翻转 + LED 分频 + 串口数据源选择） |
| `CTRL_NotifyInjectedComplete` | `BSP_ToggleLed()`（无条件） | 运行状态下翻转，其他状态跳过 |
| `CTRL_ProcessTemperature` | 空壳 | 温度阈值比较 + 状态机驱动 |
| `HAL_GPIO_EXTI_Callback` | STOP↔RUN | STOP↔RUN，FAULT 状态下不响应 |

---

## 2. 前置条件及待确认事项

| 编号 | 内容 | 确认值 | 状态 |
|---|---|---|---|
| TBD-07 | 温度异常阈值 | 45.0°C | **已确认（2026-08-25）** |
| TBD-07 | 温度恢复阈值 | 40.0°C | **已确认（2026-08-25）** |

> 以上确认结果为 T07 编码的最终依据（2026-08-25），实施时不再改动。

---

## 3. 需要修改的文件

仅 `App/Src/app_control.c`。

- 新增引用：`#include "app_monitor.h"`、`#include "app_process.h"`、`#include "app_sampling.h"`（500 Hz 调度需调用 MD-05 发送接口、MD-03 幅值接口、MD-02 温度接口；原计划仅列 app_monitor.h，属遗漏，按需补齐）
- 文件头 `@brief` 更新为「T07：完整 STOP/RUN/FAULT 状态机与 500 Hz 周期调度」
- 移除 T06 辅助变量 `timer_500hz_count`（含 `[T06 辅助代码]` 注释）

---

## 4. 具体代码修改方案

### 4.1 扩展 SystemState_t

```c
typedef enum {
    SYSTEM_STATE_STOP,
    SYSTEM_STATE_RUN,
    SYSTEM_STATE_FAULT
} SystemState_t;
```

### 4.2 新增配置常量

```c
#define TEMPERATURE_FAULT_THRESHOLD     45.0f   /* 温度异常阈值（TBD-07 已确认） */
#define TEMPERATURE_RECOVERY_THRESHOLD  40.0f   /* 温度恢复阈值（TBD-07 已确认） */
#define LED_FAULT_DIVIDER               25      /* 500 Hz / 25 = 20 次翻转/s，对应 10 Hz LED 闪烁 */
```

> 原计划恢复阈值为 43.0f，按开发者确认的 40.0f 修正。

### 4.3 私有变量调整

```c
static volatile SystemState_t system_state = SYSTEM_STATE_STOP;   /* 跨 ISR 共享（ADC/TIM6/EXTI），volatile */
static volatile uint8_t  led_div_counter = 0;   /* LED 分频计数器（仅异常状态使用） */
static bool     dac_toggle      = false;   /* DAC 翻转标志（仅运行状态使用） */
```

- 删除 T06 辅助变量 `timer_500hz_count` 及其声明注释；
- `system_state` 由 T06 的普通声明改为 `volatile`（ADC/TIM6/EXTI 三个中断上下文共享）；
- `led_div_counter` 在 TIM6 回调递增、温度回调清零，同样以 `volatile` 修饰；
- `dac_toggle` 仅 TIM6 回调访问，无需 `volatile`。
（2026-08-25 审阅修订）

### 4.4 APP_Control_Init()

保持不变（T06 已正确实现，与详细设计 §3.4 一致）。

### 4.5 CTRL_NotifyInjectedComplete()

```c
void CTRL_NotifyInjectedComplete(void)
{
    if (system_state == SYSTEM_STATE_RUN) {
        BSP_ToggleLed();
    }
}
```

### 4.6 CTRL_ProcessTemperature()

```c
void CTRL_ProcessTemperature(float temperature)
{
    if (system_state == SYSTEM_STATE_FAULT) {
        if (temperature < TEMPERATURE_RECOVERY_THRESHOLD) {
            system_state = SYSTEM_STATE_STOP;
            BSP_SetLedLow();
            led_div_counter = 0;
        }
    } else {
        if (temperature > TEMPERATURE_FAULT_THRESHOLD) {
            if (system_state == SYSTEM_STATE_RUN) {
                BSP_StopRealtimeChain();
                BSP_StopDac();
            }
            system_state = SYSTEM_STATE_FAULT;
        }
    }
}
```

> 比较条件按开发者确认的规则：`temperature > 45.0f` 进入 FAULT、`temperature < 40.0f` 恢复到 STOP（不使用 `>=` / `<=`）。运行→异常时先停实时链路和 DAC 再置 FAULT；异常→停止时熄灭 LED、复位分频计数器（设计 §3.1.2）。

### 4.7 HAL_TIM_PeriodElapsedCallback() — 替换为正式调度

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance != TIM6) {
        return;
    }

    /* ① DAC 翻转 — 仅运行状态，形成 250 Hz 方波 */
    if (system_state == SYSTEM_STATE_RUN) {
        dac_toggle = !dac_toggle;
        BSP_WriteDac(dac_toggle ? PROC_GetDacAmplitude() : 0);
    }

    /* ② LED 分频 — 仅异常状态，10 Hz 闪烁 */
    if (system_state == SYSTEM_STATE_FAULT) {
        if (++led_div_counter >= LED_FAULT_DIVIDER) {
            led_div_counter = 0;
            BSP_ToggleLed();
        }
    }

    /* ③ 串口数据源选择 — 运行/停止发送，异常不发送 */
    if (system_state == SYSTEM_STATE_RUN) {
        MON_SendData((float)PROC_GetDacAmplitude());
    } else if (system_state == SYSTEM_STATE_STOP) {
        if (SAMP_IsTemperatureValid()) {
            MON_SendData(SAMP_GetTemperature());
        }
    }
}
```

> 注：`MON_SendData()` 当前为空壳（T08 实现），调用无害。数据源选择逻辑在 T07 中建立，T08 实现实际串口发送。

### 4.8 HAL_GPIO_EXTI_Callback() — 增加 FAULT 保护

在现有实现的 `GPIO_Pin != B1_Pin` 检查之后、软件消抖之前，插入：

```c
    if (system_state == SYSTEM_STATE_FAULT) {
        return;
    }
```

其余逻辑（消抖 + STOP↔RUN 切换）不变。

---

## 5. 编译和验证方案

### 5.1 编译验证

Debug 配置编译通过，仅 `app_control.c` 变更。

### 5.2 板级验证

| 验证项 | 方法 | 预期 |
|---|---|---|
| 状态机 + DAC | 按 B1 进入运行，示波器观察 PA4 | 250 Hz 方波，旋转电位器时幅值变化 |
| LED 运行指示 | 按 B1 进入运行，目视 PA5 | LED 常亮（10 kHz 注入组回调翻转） |
| LED 停止 | 按 B1 返回停止 | LED 熄灭 |
| LED 异常闪烁 + FAULT 保护 | 将 MCU 适度加热至 40°C 以上（超过恢复阈值），再通过调试器将 `system_state` 改为 `SYSTEM_STATE_FAULT` | LED 10 Hz 闪烁，按 B1 无响应 |

> 注意：若常温下直接用调试器修改 `system_state`，下一次温度回调会立即满足 `temperature < 40.0f` 而恢复到 STOP，无法观察 FAULT 状态行为。需先加热使温度超过恢复阈值。正式的温度异常进入与恢复验证留待 T09。

---

## 6. 修改汇总

| 文件 | 修改内容 |
|---|---|
| `App/Src/app_control.c` | 更新文件头；新增 `#include "app_monitor.h"`、`#include "app_process.h"`、`#include "app_sampling.h"`；`SystemState_t` 增加 FAULT；新增温度阈值（45°C/40°C，TBD-07 已确认）和 `LED_FAULT_DIVIDER` 常量；新增 `led_div_counter`/`dac_toggle` 私有变量、移除 `timer_500hz_count`、`system_state`/`led_div_counter` 加 `volatile`；实现正式 500 Hz 调度（DAC 翻转 + LED 分频 + 串口数据源选择）；`CTRL_NotifyInjectedComplete` 改为按状态翻转；`CTRL_ProcessTemperature` 实现阈值比较和状态切换；`HAL_GPIO_EXTI_Callback` 增加 FAULT 保护 |

**不修改**：头文件、其他 App 模块、CMakeLists.txt、CubeMX 生成代码、`.ioc`、详细设计文档。

## 7. 实施顺序建议

1. TBD-07 已确认（2026-08-25：45°C / 40°C），无需再确认；
2. 按 §4 修改 `App/Src/app_control.c`，经差异视图提交开发者审阅，确认后写入工程；
3. 执行 §5.1 编译验证；
4. 执行 §5.2 板级验证（开发者执行烧录与观测）；
5. 记录验证结果至 `docs/logs/task_log.md`，待批准后进入 T08。

## 8. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 阈值已确认 | TBD-07 = 45°C / 40°C（2026-08-25）；比较条件为 `>` / `<`（不使用 `>=` / `<=`） |
| 比较运算符 | 按开发者确认使用 `>` / `<`：`temperature > 45.0f` 进 FAULT、`temperature < 40.0f` 恢复 STOP |
| 跨 ISR 共享 | `system_state`（ADC/TIM6/EXTI 三上下文）与 `led_div_counter`（TIM6 递增、温度回调清零）以 `volatile` 修饰（2026-08-25 审阅修订）；`dac_toggle` 仅 TIM6 回调访问，无需 `volatile` |
| FAULT 调试方法 | 常温下调试器直接改 `system_state` 会被下一次温度回调立即恢复到 STOP（§5.2 注）；需按 §5.2 加热或临时注释恢复判断观察 FAULT 行为 |
| DAC 关闭时序 | 运行→异常经 `BSP_StopDac()` 先置零再关闭（T03 实现），PA4 不保持残留电平；停止状态 500 Hz 回调不写 DAC |
| LED 驱动来源 | 运行=10 kHz 注入组回调翻转（视觉常亮）；异常=500 Hz 分频 10 Hz 闪烁（注入组已停）；停止=熄灭。三个来源按状态互斥，无同 GPIO 并发写入 |
| MON_SendData 空壳 | T08 才实现组帧与 DMA 发送，T07 调用无害；`MON_IsBusy` 由 MD-05 内部管理，MD-04 不直接处理 |
| 修改边界 | 仅 app_control.c；不实现 T08 串口内容、不实现 T09 温度异常完整验证 |
