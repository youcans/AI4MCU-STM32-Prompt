# T06 实现方案 — 打通并验证 500 Hz 周期、规则采样和按键事件

> 编制日期：2026-08-08
> 修订日期：2026-08-25（校订版；按开发者审阅意见确认 §2 待确认事项，恢复原版板级验证方案）
> 依据文件：
> - `docs/tasks/programming_task_list.md` §T06
> - `docs/design/software_detailed_design.md`
> - `docs/analysis/initial_project_analysis.md`
> - `docs/logs/task_log.md`（T04/T05 执行记录）

---

## 1. 任务实现分析

### 1.1 当前状态

- T04（2026-08-24）：10 kHz 注入采样链路已验证，`app_control.c` 中保留 T04 辅助代码（`APP_Control_Init()` 临时调用 `BSP_StartRealtimeChain()`，`CTRL_NotifyInjectedComplete()` 临时翻转 LED）。
- T05（2026-08-24）：MD-03 实时处理已实现并板级验证通过（PA8 10 kHz PWM 占空比随电位器 1%～99% 变化）。
- `BSP_StartRegularAdc()`、`BSP_SetLedLow()` 已在 T02 声明、T03 实现；详细设计 §2.1.1 已含全部 13 项 MD-01 接口（含"启动 ADC 规则组""熄灭 LED"）。**旧计划的 app_bsp 接口新增和设计文档同步两项工作已不存在。**
- 本任务实质为：替换 T04 辅助代码为正规 STOP 初始化流程，新增 TIM6/EXTI/规则组三个 HAL 回调，打通 500 Hz 周期、温度采样和按键启停三条事件链路。

### 1.2 三条硬件事件路径

```
路径 A: 500 Hz 软件调度
  TIM6 Update → TIM6_DAC_IRQHandler → HAL_TIM_IRQHandler(&htim6)
    → HAL_TIM_PeriodElapsedCallback [app_control.c]
      → 辅助计数变量验证 500 Hz 频率（T07 替换为正式调度）

路径 B: 温度采样（硬件触发，异步于路径 A）
  TIM6 Update → TRGO → ADC1 Regular Group (温度传感器)
    → EOC → ADC1_2_IRQHandler → HAL_ADC_IRQHandler
      → HAL_ADC_ConvCpltCallback [app_sampling.c]
        → 读取温度 → 换算 → 持存 → CTRL_ProcessTemperature()（空壳，T07 实现判断）

路径 C: 按键事件
  PC13 rising（板载外部下拉，按下接通高电平）→ EXTI15_10_IRQHandler
    → HAL_GPIO_EXTI_Callback [app_control.c]
      → 软件消抖 → 停止↔运行切换
```

### 1.3 外设配置核实（.ioc / 当前工程代码）

| 配置项 | 实际值 | 来源 |
|---|---|---|
| TIM6 | Prescaler=159、Period=1999、TRGO=UPDATE → 160 MHz/160/2000 = **500 Hz 精确** | `Core/Src/tim.c` |
| SYSCLK | HSE=24 MHz，PLLM/3 ×N40 ÷P2 = **160 MHz**（HSE_VALUE=24000000） | `main.c`、`stm32g4xx_hal_conf.h` |
| ADC1 规则组 | 温度传感器通道、Rank1、采样 247.5 cycles、外部触发 T6_TRGO 上升沿、EOC=序列结束 | `Core/Src/adc.c`、`.ioc` |
| ADC1 中断 | ADC1_2_IRQn 优先级 0（注入组与规则组共用，无嵌套） | `Core/Src/adc.c` |
| TIM6 中断 | TIM6_DAC_IRQn 优先级 1 | `Core/Src/tim.c` |
| B1 按键 | PC13（`B1_Pin`）、`GPIO_MODE_IT_RISING`、NOPULL（板载外部下拉）、EXTI15_10_IRQn 优先级 4 | `Core/Src/gpio.c`、`main.h` |
| 规则组启动 | `HAL_ADC_Start_IT(&hadc1)`（由 `BSP_StartRegularAdc()` 封装，已存在） | `App/Src/app_bsp.c` |

---

## 2. 前置条件及待确认事项

| 编号 | 内容 | 状态 / 建议 |
|---|---|---|
| CH2/CH3 预设占空比 | T06 编码前置条件 | **已确认**：CH2=800（10%）、CH3=1600（20%）（ARR=8000） |
| 按键消抖时间 | 设计 TBD-03 待确认 | **已确认（2026-08-25）**：`DEBOUNCE_MS = 50` |
| 温度换算公式 | 设计文档未固化（cubemx_config_checklist 归入编码阶段） | **已确认（2026-08-25）**：按 §5.2 公式实施——TS_CAL 工厂校准线性插值 + 3.3V 标称 VDDA 归一化（×3300/3000） |
| ADC1 校准 | 旧计划称"已在验证阶段补充"，**当前工程（.ioc/adc.c/main.c）实际无校准调用** | **已确认（2026-08-25）**：在 `BSP_StartRegularAdc()` 内、`HAL_ADC_Start_IT()` 之前执行 `HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)`（ADC 首次启动前校准，接口不变） |

> 以上确认结果（2026-08-25）为 T06 编码的最终依据，实施时不再改动。

---

## 3. 需要修改的文件和函数

### 3.1 app_control.c（MD-04）

| 修改位置 | 修改内容 |
|---|---|
| 文件头 `@brief` 注释 | 更新为「T06：正规 STOP 初始化 + 500 Hz/按键事件链路；状态机完整逻辑留待 T07」 |
| 新增引用 | `#include "tim.h"`（htim6 声明；main.h 已由 app_control.h 包含） |
| 新增类型 | `SystemState_t` 枚举（T06 仅 STOP/RUN，FAULT 由 T07 扩展） |
| 新增配置常量 | `PWM_DUTY_CH2_PRESET`=800、`PWM_DUTY_CH3_PRESET`=1600、`DEBOUNCE_MS`=50 |
| 新增私有变量 | `system_state`、`timer_500hz_count`（辅助验证计数）、`last_button_tick` |
| `APP_Control_Init()` | 替换 T04 辅助代码为正规 STOP 初始化流程 |
| 新增 `HAL_TIM_PeriodElapsedCallback()` | TIM6 检查 + 500 Hz 辅助计数（T07 替换为正式调度） |
| 新增 `HAL_GPIO_EXTI_Callback()` | B1_Pin 检查 + 软件消抖 + 停止↔运行切换 |
| `CTRL_NotifyInjectedComplete()` | 保持 T04 临时 LED 翻转（T07 替换为按状态控制） |
| `CTRL_ProcessTemperature()` | 保持空壳（T07 实现异常判断） |

### 3.2 app_sampling.c（MD-02）

| 修改位置 | 修改内容 |
|---|---|
| 文件头 `@brief` 注释 | 更新为「T06：注入组回调 + 规则组温度采样」 |
| 新增宏 | `TEMP30_CAL_ADDR`/`TEMP130_CAL_ADDR`（工厂校准值地址） |
| 新增私有变量 | `chip_temperature`、`temperature_valid` |
| 新增 `HAL_ADC_ConvCpltCallback()` | 读取温度原始值 → 换算 → 持存 → `CTRL_ProcessTemperature()` |
| `SAMP_GetTemperature()` | 返回当前温度值（替换 `0.0f` 空壳） |
| `SAMP_IsTemperatureValid()` | 返回有效性标志（替换 `false` 空壳） |
| `HAL_ADCEx_InjectedConvCpltCallback()` | 仅更新注释（"空壳"→"T05 已实现计算逻辑"），逻辑不动 |

### 3.3 app_bsp.c（MD-01）

| 修改位置 | 修改内容 |
|---|---|
| `BSP_StartRegularAdc()` | 在 `HAL_ADC_Start_IT()` 之前补充 ADC1 校准调用（已确认，见 §2）；接口声明不变 |

### 3.4 不修改的文件

- `App/Inc/app_bsp.h`：接口已齐（T02），无需改动。
- `docs/design/software_detailed_design.md`：§2.1.1 已含全部 13 项 MD-01 接口，无需同步。
- `main.c`：`APP_Control_Init()` 已在 USER CODE 2 接入（T02），无需改动。
- CMakeLists.txt、CubeMX 生成代码、`.ioc`、其他 App 模块。

---

## 4. 接口及调用关系

| 调用关系 | 说明 |
|---|---|
| main.c → `APP_Control_Init()` | T02 已接入，本任务仅重写其内部逻辑 |
| `APP_Control_Init` → MD-01 | `BSP_WritePwmDutyCh2/3`（预设值）→ `BSP_SetLedLow`、`BSP_StopDac`（初始复位）→ `BSP_StartRegularAdc`（内部补充 ADC 校准）、`BSP_StartTim6`（启动两条 500 Hz 子路径）。全部接口已存在，**无接口变更** |
| TIM6 ISR → `HAL_TIM_PeriodElapsedCallback`（MD-04） | 本任务新增回调；T06 仅辅助计数，T07 替换为 DAC 翻转/LED 分频/串口调度 |
| TIM6 TRGO → ADC1 规则组（硬件）→ `HAL_ADC_ConvCpltCallback`（MD-02） | 本任务新增回调；温度换算后调用 `CTRL_ProcessTemperature()`（空壳，链路建立） |
| `HAL_GPIO_EXTI_Callback`（MD-04） | 运行：`BSP_StartRealtimeChain()` + `BSP_StartDac()`；停止：`BSP_StopRealtimeChain()` + `BSP_StopDac()` + `BSP_SetLedLow()` |
| MD-02 → MD-04 数据 | `chip_temperature`（持存，`SAMP_GetTemperature`/`SAMP_IsTemperatureValid` 只读接口，T07 的 500 Hz 调度使用） |
| 跨 ISR 共享 | `chip_temperature`（float）由 ADC1_2 ISR（优先级 0）写、TIM6 ISR（优先级 1，T07 起）读；`temperature_valid` 同理。均 `volatile` 修饰，32 位对齐原子访问，符合详细设计 §2.6，无临界区 |

公开接口和模块职责均不变；三个 HAL 回调按详细设计 §2.2 归属各模块 `.c` 文件，不进入头文件。

---

## 5. 具体代码修改方案

### 5.1 App/Src/app_control.c（修改后完整内容）

```c
/**
  ******************************************************************************
  * @file    app_control.c
  * @brief   MD-04 系统控制模块（T06：正规 STOP 初始化 + 500 Hz/按键事件链路；状态机完整逻辑留待 T07）
  ******************************************************************************
  */
#include "app_control.h"
#include "app_bsp.h"
#include "tim.h"

/* 系统状态（T06 仅 STOP/RUN，FAULT 由 T07 扩展） */
typedef enum {
    SYSTEM_STATE_STOP,
    SYSTEM_STATE_RUN
} SystemState_t;

/* 配置常量 */
#define PWM_DUTY_CH2_PRESET   800U   /* CH2 预设占空比 10%（已确认） */
#define PWM_DUTY_CH3_PRESET   1600U  /* CH3 预设占空比 20%（已确认） */
#define DEBOUNCE_MS           50U    /* 按键消抖时间（已确认：50 ms） */

/* 私有变量 */
static SystemState_t system_state = SYSTEM_STATE_STOP;
static volatile uint32_t timer_500hz_count = 0;   /* [T06 辅助代码] 500 Hz 验证计数，T07 移除 */
static uint32_t last_button_tick = 0;             /* 消抖用最近一次按键时间戳 */

void APP_Control_Init(void)
{
    system_state = SYSTEM_STATE_STOP;

    /* 写入 CH2/CH3 预设占空比（10% / 20%） */
    BSP_WritePwmDutyCh2(PWM_DUTY_CH2_PRESET);
    BSP_WritePwmDutyCh3(PWM_DUTY_CH3_PRESET);

    /* 复位外设初始状态：LED 熄灭、DAC 关闭（BSP_StopDac 内部先置零） */
    BSP_SetLedLow();
    BSP_StopDac();

    /* 启动温度采样（规则组）与 500 Hz 周期调度 */
    BSP_StartRegularAdc();
    BSP_StartTim6();
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    /* 仅处理 TIM6 */
    if (htim->Instance == TIM6) {
        timer_500hz_count++;   /* [T06 辅助代码] T07 替换为正式 500 Hz 调度 */
    }
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    uint32_t now;

    if (GPIO_Pin != B1_Pin) {
        return;
    }

    /* 软件消抖 */
    now = HAL_GetTick();
    if ((now - last_button_tick) < DEBOUNCE_MS) {
        return;
    }
    last_button_tick = now;

    /* 停止 ↔ 运行切换 */
    if (system_state == SYSTEM_STATE_STOP) {
        system_state = SYSTEM_STATE_RUN;
        BSP_StartRealtimeChain();
        BSP_StartDac();
    } else {
        system_state = SYSTEM_STATE_STOP;
        BSP_StopRealtimeChain();
        BSP_StopDac();
        BSP_SetLedLow();
    }
}

void CTRL_NotifyInjectedComplete(void)
{
    /* [T04 辅助代码] 临时翻转 LED 验证 10 kHz 事件频率，T07 替换为按状态判断的正式逻辑 */
    BSP_ToggleLed();
}

void CTRL_ProcessTemperature(float temperature)
{
    /* 空壳保留：T07 实现异常判断逻辑 */
    (void)temperature;
}
```

### 5.2 App/Src/app_sampling.c（修改后完整内容）

```c
/**
  ******************************************************************************
  * @file    app_sampling.c
  * @brief   MD-02 采样管理模块（T06：注入组回调 + 规则组温度采样）
  ******************************************************************************
  */
#include "app_sampling.h"
#include "app_process.h"
#include "app_control.h"

/* 温度传感器工厂校准值地址（30°C / 130°C，STM32G431） */
#define TEMP30_CAL_ADDR    ((uint16_t *)0x1FFF75A8)
#define TEMP130_CAL_ADDR   ((uint16_t *)0x1FFF75CA)
#define TEMP30_CAL_VALUE   (*TEMP30_CAL_ADDR)
#define TEMP130_CAL_VALUE  (*TEMP130_CAL_ADDR)

/* 芯片温度值与有效标志（500 Hz 更新，volatile 供跨 ISR 共享） */
static volatile float chip_temperature = 0.0f;
static volatile bool  temperature_valid = false;

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

    /* 传递给 MD-03 处理（T05 已实现计算逻辑） */
    PROC_HandleSamples(ch1, ch2, ch3);

    /* 通知 MD-04（T04 中临时翻转 LED，T07 替换为正式状态控制） */
    CTRL_NotifyInjectedComplete();
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    uint32_t adc_raw;
    float ts_data_3v;
    float temp;

    /* 仅处理 ADC1 */
    if (hadc->Instance != ADC1) {
        return;
    }

    /* 读取温度原始值；工厂校准值在 VREF+=3.0V 下测量，本工程 VREF+=VDD≈3.3V（JP8=[2-3]），
       需归一化到 3.0V 等效值 */
    adc_raw = HAL_ADC_GetValue(hadc);
    ts_data_3v = (float)adc_raw * 3300.0f / 3000.0f;

    /* 按工厂校准值线性插值换算为 °C（已确认：3.3V 标称 VDDA 归一化） */
    temp = ((ts_data_3v - (float)TEMP30_CAL_VALUE) * 100.0f)
           / ((float)TEMP130_CAL_VALUE - (float)TEMP30_CAL_VALUE)
           + 30.0f;

    /* 持存并置位有效标志 */
    chip_temperature = temp;
    temperature_valid = true;

    /* 传递给 MD-04 温度处理（T06 建立数据链路，T07 实现异常判断） */
    CTRL_ProcessTemperature(chip_temperature);
}

float SAMP_GetTemperature(void)
{
    return chip_temperature;
}

bool SAMP_IsTemperatureValid(void)
{
    return temperature_valid;
}
```

**注意**：
- §2 事项已全部确认（2026-08-25）：温度公式（3.3V 标称 VDDA 归一化）、消抖 50 ms、ADC 校准（见 §5.3）。
- 局部变量声明于函数开头（沿用 T04/T05 校订风格）。
- 注入组回调仅更新注释，逻辑与 T04/T05 验证通过的版本完全一致。

### 5.3 App/Src/app_bsp.c（仅修改 `BSP_StartRegularAdc()`）

```c
void BSP_StartRegularAdc(void)
{
    /* ADC1 校准（首次启动前执行，已确认：2026-08-25） */
    HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);

    HAL_ADC_Start_IT(&hadc1);
}
```

- 校准调用不检查返回值，与现有 BSP 启动类接口风格一致；
- `hadc1` 由 adc.h 声明（app_bsp.c 已包含），`ADC_SINGLE_ENDED` 来自 HAL ADC 头文件，无需新增引用；
- 其余内容与 T03 实现完全一致，不逐行列出。

---

## 6. 编译和板级验证方案

### 6.1 编译验证

与 T05 相同方式执行 Debug 配置编译，确认 `app_control.c`、`app_sampling.c`、`app_bsp.c` 无编译错误和警告，成功链接生成 `.elf`。三个 HAL 回调均覆盖 HAL 弱定义，签名须与 HAL 声明一致。

### 6.2 板级验证

**接线**：沿用 T04/T05（JP8=[2-3]、PA1 电位器）。

> 设置断点会暂停 CPU，不能同时用于时序观测。断点观察与时序/示波器观测分开执行。

**500 Hz 时基**：调试器观察 `timer_500hz_count` 持续递增（约每秒 500），确认 TIM6 500 Hz 调度正确。

**温度采样**：调试器断点设在 `HAL_ADC_ConvCpltCallback`，观察 `adc_raw` 和换算后的 `temp` 值，确认温度值处于合理范围（室温约 25～35°C）且随芯片温度变化趋势正确。确认 `temperature_valid` 在首次转换完成后置为 true。

**按键启停**：
1. 上电 → LED 熄灭（STOP 状态），PWM/ADC 关闭。
2. 按 B1 → LED 常亮（T04 辅助代码 10 kHz 翻转），示波器确认 PA8/PB13 PWM 输出。
3. 按 B1 → LED 熄灭，PWM 关闭。
4. 重复按键，确认无抖动误触发。

---

## 7. 修改汇总

| 文件 | 修改内容 |
|---|---|
| `App/Src/app_control.c` | 更新文件头；新增 `#include "tim.h"`；定义 `SystemState_t`（STOP/RUN）、配置常量、私有变量；重写 `APP_Control_Init()` 正规 STOP 初始化；新增 `HAL_TIM_PeriodElapsedCallback`（500 Hz 辅助计数）、`HAL_GPIO_EXTI_Callback`（消抖 + 启停切换） |
| `App/Src/app_sampling.c` | 更新文件头；新增温度校准宏、私有变量；新增 `HAL_ADC_ConvCpltCallback`（温度换算 + 持存 + 传递）；实现 `SAMP_GetTemperature`/`SAMP_IsTemperatureValid`；注入组回调仅更新注释 |
| `App/Src/app_bsp.c` | `BSP_StartRegularAdc()` 内在 `HAL_ADC_Start_IT()` 之前补充 ADC1 校准调用 |

**不修改**：app_bsp.h（接口已齐）、app_process.*、app_monitor.*、main.c、CMakeLists.txt、CubeMX 生成代码、`.ioc`、详细设计文档（§2.1.1 已同步）。

## 8. 实施顺序建议

1. §2 待确认事项已全部由开发者确认（2026-08-25：温度公式 3.3V 标称 VDDA 归一化、消抖 50 ms、ADC 校准在 `BSP_StartRegularAdc()` 内补充）；
2. 按 §5 修改 `app_control.c`、`app_sampling.c`、`app_bsp.c`，经差异视图提交开发者审阅，确认后写入工程；
3. 执行 §6.1 编译验证；
4. 执行 §6.2 板级验证（开发者执行烧录与观测）；
5. 记录验证结果至 `docs/logs/task_log.md`，待批准后进入 T07。

## 9. 风险与注意事项

| 事项 | 说明 |
|---|---|
| ADC1 校准 | 已确认（2026-08-25）：`HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)` 置于 `BSP_StartRegularAdc()` 内、`HAL_ADC_Start_IT()` 之前，满足"ADC 首次启动前校准"；校准后 T04 实测的 ch3≈50（GND 输入）零位偏差应改善，可在板级验证中复核 |
| 温度换算公式 | 已确认（2026-08-25）：TS_CAL 工厂校准线性插值 + 3.3V 标称 VDDA 归一化；TS_CAL 地址 0x1FFF75A8/0x1FFF75CA 为 G431 官方地址，归一化方向已核对（3.3V 测量值 ×3.3/3.0 折算到 3.0V 校准标度） |
| 消抖时间 | `DEBOUNCE_MS=50` 已确认（2026-08-25） |
| 500 Hz 精度 | TIM6 = 160 MHz/(159+1)/(1999+1) = 500 Hz 精确（HSE=24 MHz、SYSCLK=160 MHz 已核实） |
| 中断优先级 | ADC1_2=0、TIM6=1、EXTI=4，与设计 §3.3 一致；注入/规则组共用 ADC1_2_IRQn 无嵌套，温度换算在优先级 0 下执行，计算量小 |
| T04 辅助代码遗留 | `CTRL_NotifyInjectedComplete()` 的 LED 翻转保留至 T07；`timer_500hz_count` 为 T06 辅助验证变量，T07 移除 |
| 越界控制 | T07 的 FAULT 状态、温度判断、DAC 250 Hz、LED 分频、串口调度不在本任务实现；CH2/CH3 预设值在 Init 写入一次（CCR 在停链后保留，满足 T06 功能） |
| 修改边界 | 不修改 main.c（Init 已接入）、不修改 .ioc、不修改 HAL 库、不修改 app_bsp.h（接口不变）；app_bsp.c 仅按 §5.3 修改 `BSP_StartRegularAdc()` 内部实现 |
