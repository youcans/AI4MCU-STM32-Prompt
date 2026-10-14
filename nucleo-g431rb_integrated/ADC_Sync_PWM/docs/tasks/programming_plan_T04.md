# 编程计划 T04 — 打通并验证 10 kHz ADC 注入采样链路

> 编制日期：2026-08-24（在 2026-08-09 初版基础上按当前工程状态校订）
> 任务依据：
> - `docs/tasks/programming_task_list.md` T04（任务目标、涉及文件、实施要点、验证方式）
> - `docs/design/software_detailed_design.md` §3.2.1（10 kHz 实时路径）、§2.2（HAL 回调入口）
> - `docs/analysis/initial_project_analysis.md` §4.3（待实现的 HAL 回调）
> 状态：待开发者审阅和批准。本阶段只生成修改计划，不修改任何工程文件。

---

## 0. 校订说明（相对 2026-08-09 初版）

初版内容基本保留，仅按当前工程状态做以下校订：

| 校订项 | 初版 | 校订后 | 原因 |
|---|---|---|---|
| app_sampling.c 引用列表 | 含 `#include "adc.h"` | 移除 adc.h | 实测核实：`HAL_ADCEx_InjectedConvCpltCallback` 与 `HAL_ADCEx_InjectedGetValue` 经 main.h → stm32g4xx_hal.h → stm32g4xx_hal_adc.h → stm32g4xx_hal_adc_ex.h 链路可达，且回调参数 `hadc` 直接使用，不引用 `hadc1`。遵循"按接口/实现实际需要添加包含"原则 |
| 回调内局部变量声明位置 | 语句中间声明 | 移至函数开头 | 与工程编码风格一致 |
| 其余内容 | — | 保持不变 | 函数名（PROC_HandleSamples、CTRL_NotifyInjectedComplete、BSP_StartRealtimeChain、BSP_ToggleLed）与 T02/T03 定型名称完全一致，无需修改 |

## 1. 10 kHz ADC 注入采样链路分析

### 1.1 硬件触发路径

```
TIM1 CH4 (CCR4=4000, PWM mode 1, center-aligned, ARR=8000)
  → 内部 OC4REF 信号
    → TIM1 TRGO2 (MasterOutputTrigger2 = TIM_TRGO2_OC4REF)
      → ADC1 注入组 (ExternalTrigInjecConv = ADC_EXTERNALTRIGINJEC_T1_TRGO2)
        → Rank1=IN2(PA1) → Rank2=IN12(PB1) → Rank3=IN15(PB0)
          → JEOS → ADC1_2_IRQHandler (NVIC 优先级 0)
            → HAL_ADC_IRQHandler(&hadc1)
              → HAL_ADCEx_InjectedConvCpltCallback(&hadc1)  ← 在 app_sampling.c 中实现
```

### 1.2 当前注入组配置（adc.c）

| 参数 | 值 |
|---|---|
| InjectedNbrOfConversion | 3 |
| Rank1 | ADC_CHANNEL_2 (PA1), 24.5 cycles |
| Rank2 | ADC_CHANNEL_12 (PB1), 24.5 cycles |
| Rank3 | ADC_CHANNEL_15 (PB0), 24.5 cycles |
| ExternalTrigInjecConv | ADC_EXTERNALTRIGINJEC_T1_TRGO2 |
| ExternalTrigInjecConvEdge | ADC_EXTERNALTRIGINJECCONV_EDGE_RISING |
| AutoInjectedConv | DISABLE |
| InjectedDiscontinuousConvMode | DISABLE |

### 1.3 HAL 回调签名（实测核实）

```c
void HAL_ADCEx_InjectedConvCpltCallback(ADC_HandleTypeDef *hadc);
```

- HAL 库以 `__weak` 定义于 `stm32g4xx_hal_adc_ex.c:1189`，应用代码在 `app_sampling.c` 中覆盖实现；
- 回调声明经 main.h → stm32g4xx_hal_adc_ex.h 可达，app_sampling.c **无需**再包含 adc.h。

### 1.4 读取注入组结果

```c
uint32_t HAL_ADCEx_InjectedGetValue(const ADC_HandleTypeDef *hadc, uint32_t InjectedRank);
```

Rank 参数为 `ADC_INJECTED_RANK_1` / `ADC_INJECTED_RANK_2` / `ADC_INJECTED_RANK_3`。

## 2. 需要修改的文件和函数

### 2.1 app_sampling.c（MD-02）

**新增引用**：
```c
#include "app_process.h"
#include "app_control.h"
```

（`main.h` 已由 app_sampling.h 提供；adc.h 不需要——见 §0 校订说明。）

**新增函数**：`HAL_ADCEx_InjectedConvCpltCallback()`

```c
void HAL_ADCEx_InjectedConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    uint16_t ch1;
    uint16_t ch2;
    uint16_t ch3;

    /* 仅处理 ADC1 */
    if (hadc->Instance != ADC1) {
        return;
    }

    /* 读取三路注入组结果 */
    ch1 = (uint16_t)HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_1);
    ch2 = (uint16_t)HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_2);
    ch3 = (uint16_t)HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_3);

    /* 传递给 MD-03 处理（当前为空壳，T05 实现计算逻辑） */
    PROC_HandleSamples(ch1, ch2, ch3);

    /* 通知 MD-04（T04 中临时翻转 LED，T07 替换为正式状态控制） */
    CTRL_NotifyInjectedComplete();
}
```

**不修改**：`SAMP_GetTemperature` / `SAMP_IsTemperatureValid`（保持 T02 空实现，T06 实现）。

### 2.2 app_control.c（MD-04）

**新增引用**：
```c
#include "app_bsp.h"
```

**修改 `APP_Control_Init()`**：添加临时启动代码。

```c
void APP_Control_Init(void)
{
    /* [T04 辅助代码] 临时启动 10 kHz 实时链路，T06 替换为正规初始化流程 */
    BSP_StartRealtimeChain();
}
```

**修改 `CTRL_NotifyInjectedComplete()`**：添加临时 LED 翻转。

```c
void CTRL_NotifyInjectedComplete(void)
{
    /* [T04 辅助代码] 临时翻转 LED 验证 10 kHz 事件频率，T07 替换为按状态判断的正式逻辑 */
    BSP_ToggleLed();
}
```

**不修改**：`CTRL_ProcessTemperature`（保持 T02 空实现，T06/T07 实现）。

## 3. 接口与调用关系说明

| 项目 | T04 结论 |
|---|---|
| 新增 HAL 回调 | `HAL_ADCEx_InjectedConvCpltCallback`（app_sampling.c，覆盖 HAL 弱定义） |
| 调用关系 | 回调 → `PROC_HandleSamples`（MD-03，空壳）→ `CTRL_NotifyInjectedComplete`（MD-04）→ `BSP_ToggleLed`（MD-01）；初始化 → `APP_Control_Init`（main.c，T02 已接入）→ `BSP_StartRealtimeChain`（MD-01，T03 已实现） |
| 接口变化 | 无新增/改名/签名变化，全部使用 T02 定型接口 |
| 辅助代码生命周期 | `APP_Control_Init` 中临时启动由 T06 替换；`CTRL_NotifyInjectedComplete` 中临时翻转由 T07 替换 |

## 4. 辅助验证方案

### 4.1 10 kHz 事件频率验证

- **方法**：`CTRL_NotifyInjectedComplete()` 每次调用翻转 PA5（板载 LED LD2）。
- **预期现象**：10 kHz 翻转 = 5 kHz 方波，肉眼观察 LED 为近似常亮。用示波器观察 PA5 应看到约 5 kHz 方波。
- **辅助代码生命周期**：T07 实现正式状态机后，T04 的临时翻转被替换为运行状态时才翻转的正式逻辑。

### 4.2 三通道数据来源验证

- **方法**：在 `App/Src/app_sampling.c` 中的 `PROC_HandleSamples(ch1, ch2, ch3);` 所在行设置断点——此时三路采样值已全部读取完成，可直接观察 `ch1`、`ch2`、`ch3` 局部变量。
- **预期值**：
  - PA1（电位器滑动端）：旋转电位器时 0 ~ 4095 范围变化。
  - PB1（高电平参考，杜邦线接 3.3V）：接近 4095。
  - PB0（低电平参考，杜邦线接 GND）：接近 0。
- **辅助代码生命周期**：此为调试验证代码，通过断点或 watch 窗口观察，不产生持久代码修改。若需持久输出，可在后续任务中通过 MD-05 串口发送。

### 4.3 验证环境

- NUCLEO-G431RB 开发板，JP8=[2-3]
- PA1 接 10 kΩ 电位器（高端 3.3V，低端 GND）
- PB1 杜邦线接 3.3V
- PB0 杜邦线接 GND
- 示波器观察 PA5（LD2）

## 5. 编译和板级验证方案

### 5.1 编译验证

修改完成后，Debug 配置编译，确认 `app_sampling.c` 和 `app_control.c` 无编译错误，成功链接生成 `.elf`。

### 5.2 板级验证

**步骤一：10 kHz 时序验证（不设断点）**

1. 烧录固件，上电。
2. 示波器观察 PA5 → 约 5 kHz 方波（10 kHz 注入组回调中翻转），确认 10 kHz 事件链路正确运行。

**步骤二：三通道数据验证（设置断点）**

> 注意：设置断点会暂停 CPU，不能同时用于判断 10 kHz 时序。步骤一和步骤二应分开独立执行。

1. 调试器断点设在 `App/Src/app_sampling.c` 中的 `PROC_HandleSamples(ch1, ch2, ch3);` 所在行（此时三路采样值已读取完成）。
2. 观察 `ch1`/`ch2`/`ch3`：
   - 旋转 PA1 电位器 → `ch1` 在 0 ~ 4095 范围变化。
   - 确认 `ch2` 接近 4095（PB1 接 3.3V）。
   - 确认 `ch3` 接近 0（PB0 接 GND）。

## 6. 修改汇总

| 文件 | 修改内容 |
|---|---|
| `App/Src/app_sampling.c` | 新增 `#include "app_process.h"`、`#include "app_control.h"`；实现 `HAL_ADCEx_InjectedConvCpltCallback()` |
| `App/Src/app_control.c` | 新增 `#include "app_bsp.h"`；`APP_Control_Init()` 添加临时 `BSP_StartRealtimeChain()` 调用；`CTRL_NotifyInjectedComplete()` 添加临时 `BSP_ToggleLed()` 调用 |

**不修改**：头文件、CMakeLists.txt、CubeMX 生成代码、`app_bsp.c`、`app_process.c`、`app_monitor.c`、`main.c`。

## 7. 实施顺序建议

1. 修改 `App/Src/app_sampling.c`（新增引用 + 回调实现）；
2. 修改 `App/Src/app_control.c`（新增引用 + 两处辅助代码）；
3. 执行 §5.1 编译验证；
4. 执行 §5.2 板级验证（开发者执行烧录与观测）；
5. 记录验证结果，待批准后进入 T05。

## 8. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 回调覆盖机制 | HAL 以 `__weak` 定义回调，app_sampling.c 中的同名强定义自动覆盖；本计划已实测核实弱定义位置（stm32g4xx_hal_adc_ex.c:1189） |
| `hadc->Instance != ADC1` 检查 | 保留初版的防御性检查（本工程仅 ADC1，此检查恒为假但无害，且未来扩展 ADC2 时安全） |
| 辅助代码标识 | 临时代码均以 `[T04 辅助代码]` 注释标明替换任务（T06/T07），防止遗漏 |
| 板级验证前置条件 | 需先完成 JP8=[2-3]、PA1 电位器、PB1/PB0 杜邦线接线（§4.3），由开发者执行 |
| T05 前置条件 | T05 编码前需确认 TBD-08/TBD-09（PWM/DAC 换算公式），与本任务无关，仅提示 |
