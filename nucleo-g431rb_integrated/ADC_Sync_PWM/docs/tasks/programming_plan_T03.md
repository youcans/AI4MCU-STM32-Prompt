# 编程计划 T03 — 实现 MD-01 板级支持模块（app_bsp.c）

> 编制日期：2026-08-24
> 任务依据：
> - `docs/tasks/programming_task_list.md` T03（任务目标、涉及文件、实施要点、验证方式）
> - `docs/design/software_detailed_design.md` §2.1.1（MD-01 接口）、§3.5（实时链路启停顺序）、§1.3.1（MD-01 职责边界）
> - `docs/analysis/initial_project_analysis.md` §3（外设句柄与配置）、§6（DMA 关联、中断共享等）
> - 参考：`docs/tasks/programming_plan_T04.md`（T04 对 BSP_StartRealtimeChain/BSP_ToggleLed 的使用方式）
> 状态：待开发者审阅和批准。本阶段只生成修改计划，不修改任何工程文件。

---

## 1. 任务范围与约束遵循

| 硬性约束 | 本计划的遵循方式 |
|---|---|
| 以任务清单、详细设计、实际工程为依据 | 13 个接口按详细设计 §2.1.1 实现；实时链路启停顺序按 §3.5 与任务清单 T03 实施要点；HAL API 与句柄均经实际工程核实（见 §2） |
| 严格限定在 T03 任务范围内 | 仅修改 `App/Src/app_bsp.c`（外设头文件包含 + 13 个接口实现）；app_bsp.h 保持不变（2026-08-24 审核修订：外设头文件放入 .c）。不触碰 MD-02～MD-05、main.c、HAL 回调（T04 起）、`SystemState` 与业务常量（T06） |
| 不自行改变已确定的模块职责和公开接口 | 13 个接口签名与 T02 批准的 §4.1 完全一致，不新增、不改名；MD-01 保持"纯外设操作原语"：无业务逻辑、不持有业务数据、不访问 SystemState |
| 不修改任何工程文件或设计文件 | 本阶段仅产出本计划文档；正文所列代码均为待批准的修改内容，批准前不落盘 |

## 2. 当前工程现状（实测核实）

- T02 完成：13 个接口已在 `app_bsp.h` 声明并有空实现；`app_bsp.h` 保持仅含 `main.h` + `stdbool.h`，外设头文件在 T03 放入 app_bsp.c（审核修订）。
- 外设句柄 extern 声明已核实：`adc.h`→`hadc1`、`tim.h`→`htim1`/`htim6`、`dac.h`→`hdac1`、`usart.h`→`hlpuart1`；引脚宏 `LD2_Pin/LD2_GPIO_Port` 在 `main.h`。
- HAL API 签名已核实（均为 `HAL_StatusTypeDef` 返回）：`HAL_ADCEx_InjectedStart_IT`/`HAL_ADCEx_InjectedStop_IT`（adc_ex.h）、`HAL_ADC_Start_IT`（adc.h）、`HAL_TIM_PWM_Start`/`HAL_TIM_Base_Start_IT`（tim.h）、`HAL_TIMEx_PWMN_Start`（tim_ex.h）、`HAL_DAC_Start`/`HAL_DAC_Stop`/`HAL_DAC_SetValue`（dac.h）、`HAL_UART_Transmit_DMA`（uart.h）；`__HAL_TIM_SET_COMPARE` 宏（tim.h）。
- 工程配置关键点：TIM6 更新中断与 NVIC（优先级 1）已在 `tim.c HAL_TIM_Base_MspInit` 中使能（`BSP_StartTim6` 用 `HAL_TIM_Base_Start_IT` 即可产生 500 Hz 周期回调）；`hdma_lpuart1_tx` 已通过 `__HAL_LINKDMA` 关联到 `hlpuart1`（`BSP_StartDmaTx` 直接调 `HAL_UART_Transmit_DMA`）；ADC1 注入组 JEOS 中断由 `HAL_ADCEx_InjectedStart_IT` 内部使能。
- 已知差异（I-01）：任务清单 T03 写"11 个公开接口"，详细设计 §2.1.1 为 13 个。T02 已按 13 个声明并经开发者验证通过，本计划按 13 个实现（与详细设计一致）。

## 3. 修改文件汇总

| 操作 | 文件 | 说明 |
|---|---|---|
| 修改 | `App/Src/app_bsp.c` | 添加 5 个外设头文件包含（adc.h/tim.h/dac.h/gpio.h/usart.h）；13 个接口由空实现改为完整实现（纯 HAL 封装，无业务逻辑、无私有变量） |

`App/Inc/app_bsp.h` 保持不变（13 个接口声明与 T02 批准一致，包含关系仍仅 main.h + stdbool.h）。不修改其他任何文件。

## 4. 接口实现设计（13 个）

### 4.1 实时链路启动/停止（核心时序）

**BSP_StartRealtimeChain()** —— 按 §3.5.1，先使能 ADC 注入组中断，再启动 TIM1 与 PWM 输出：

```c
void BSP_StartRealtimeChain(void)
{
    /* ① 先使能 ADC1 注入组中断转换（触发源 TIM1_TRGO2 由硬件路由） */
    HAL_ADCEx_InjectedStart_IT(&hadc1);

    /* ② 启动 TIM1 与 PWM 输出（首次 PWM_Start 同时启动 TIM1 计数器） */
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);

    /* ③ 使能三路互补输出 */
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_2);
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_3);

    /* CH4 无需启动：仅内部 OC4REF 经 TRGO2 路由，不输出引脚 */
}
```

**BSP_StopRealtimeChain()** —— 按 2026-08-24 审核意见调整停止顺序：先直接停止 TIM1 计数器、切断 TRGO2 触发源，再停止各 PWM/互补通道，最后停止 ADC 注入组：

```c
void BSP_StopRealtimeChain(void)
{
    /* ① 直接停止 TIM1 计数器：立即停止计数与 TRGO2 触发输出 */
    htim1.Instance->CR1 &= ~TIM_CR1_CEN;

    /* ② 停止全部 PWM 与互补通道 */
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_2);
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_3);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_1);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_2);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_3);

    /* ③ 触发源已切断后，停止 ADC1 注入组中断转换 */
    HAL_ADCEx_InjectedStop_IT(&hadc1);
}
```

> **实现说明（实测核实）**：HAL 宏 `__HAL_TIM_DISABLE()` 在 CCER 中任意通道使能位（CCxE/CCxNE）置位时**不会**清除 CEN，即无法满足"先直接停止计数器、切断 TRGO2"的审核要求（HAL 的 `HAL_TIM_PWM_Stop` 也只有最后一个活跃通道停止时才清 CEN）。因此 T03 实现以直接寄存器方式清除 CEN 来实现第①步。为不修改 HAL 库文件、保持封装一致性，本计划采用 HAL 宏所操作的同一寄存器位，在第①步使用等效写法（见 §5.2 代码）。此为应用层寄存器访问，与 CLAUDE.md"不擅自修改底层 HAL 和 CMSIS 库文件"不冲突。

> 说明：HAL 返回状态不作运行时分支处理——本模块为纯封装层，不承担错误处理职责（错误处理决策属上层模块职责范围，本阶段不做扩展）。

### 4.2 其余 11 个接口（单 HAL 调用封装）

| 接口 | 实现 |
|---|---|
| `BSP_WritePwmDutyCh1(uint16_t duty)` | `__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, duty);` |
| `BSP_WritePwmDutyCh2(uint16_t duty)` | `__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, duty);` |
| `BSP_WritePwmDutyCh3(uint16_t duty)` | `__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, duty);` |
| `BSP_WriteDac(uint16_t value)` | `HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, value);` |
| `BSP_StartDac(void)` | `HAL_DAC_Start(&hdac1, DAC_CHANNEL_1);` |
| `BSP_StopDac(void)` | 置零后关闭：`HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, 0);` → `HAL_DAC_Stop(&hdac1, DAC_CHANNEL_1);`（对应 §3.5.2 ⑥⑦） |
| `BSP_ToggleLed(void)` | `HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);` |
| `BSP_SetLedLow(void)` | `HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);` |
| `BSP_StartTim6(void)` | `HAL_TIM_Base_Start_IT(&htim6);`（更新中断已在 CubeMX 使能，产生 500 Hz 周期回调） |
| `BSP_StartRegularAdc(void)` | `HAL_ADC_Start_IT(&hadc1);`（规则组中断转换，TIM6_TRGO 触发） |
| `BSP_StartDmaTx(uint8_t *data, uint16_t len)` | `return (HAL_UART_Transmit_DMA(&hlpuart1, data, len) == HAL_OK);` |

## 5. 具体修改方式

### 5.1 App/Inc/app_bsp.h

**不修改**。13 个函数声明与包含关系保持 T02 批准状态（仅 `main.h` + `stdbool.h`）。外设头文件按审核意见放入 app_bsp.c。

### 5.2 App/Src/app_bsp.c（全文重写）

```c
/**
  ******************************************************************************
  * @file    app_bsp.c
  * @brief   MD-01 板级支持模块（T03 实现，纯 HAL 封装，无业务逻辑）
  ******************************************************************************
  */
#include "app_bsp.h"
#include "adc.h"
#include "tim.h"
#include "dac.h"
#include "gpio.h"
#include "usart.h"

void BSP_StartRealtimeChain(void)
{
    /* 先使能 ADC1 注入组中断转换（触发源 TIM1_TRGO2 由硬件路由） */
    HAL_ADCEx_InjectedStart_IT(&hadc1);

    /* 启动 TIM1 与 PWM 输出（首次 PWM_Start 同时启动 TIM1 计数器） */
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);

    /* 使能三路互补输出 */
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_2);
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_3);

    /* CH4 无需启动：仅内部 OC4REF 经 TRGO2 路由，不输出引脚 */
}

void BSP_StopRealtimeChain(void)
{
    /* ① 直接停止 TIM1 计数器：立即停止计数与 TRGO2 触发输出（先切断触发源） */
    htim1.Instance->CR1 &= ~TIM_CR1_CEN;

    /* ② 停止全部 PWM 与互补通道 */
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_2);
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_3);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_1);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_2);
    HAL_TIMEx_PWMN_Stop(&htim1, TIM_CHANNEL_3);

    /* ③ 触发源已切断后，停止 ADC1 注入组中断转换 */
    HAL_ADCEx_InjectedStop_IT(&hadc1);
}

void BSP_WritePwmDutyCh1(uint16_t duty)
{
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, duty);
}

void BSP_WritePwmDutyCh2(uint16_t duty)
{
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, duty);
}

void BSP_WritePwmDutyCh3(uint16_t duty)
{
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, duty);
}

void BSP_WriteDac(uint16_t value)
{
    HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, value);
}

void BSP_StartDac(void)
{
    HAL_DAC_Start(&hdac1, DAC_CHANNEL_1);
}

void BSP_StopDac(void)
{
    /* 先置零再关闭，避免 PA4 保持最后电平 */
    HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, 0);
    HAL_DAC_Stop(&hdac1, DAC_CHANNEL_1);
}

void BSP_ToggleLed(void)
{
    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
}

bool BSP_StartDmaTx(uint8_t *data, uint16_t len)
{
    return (HAL_UART_Transmit_DMA(&hlpuart1, data, len) == HAL_OK);
}

void BSP_StartTim6(void)
{
    HAL_TIM_Base_Start_IT(&htim6);
}

void BSP_StartRegularAdc(void)
{
    HAL_ADC_Start_IT(&hadc1);
}

void BSP_SetLedLow(void)
{
    HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
}
```

> **停止实现说明（实测核实）**：HAL 宏 `__HAL_TIM_DISABLE()` 在 CCER 中任意通道使能位置位时不会清除 CEN（见 `stm32g4xx_hal_tim.h:1300-1309`），无法满足"先直接停止计数器、切断 TRGO2"的要求。故第①步直接写寄存器清除 CEN（与 HAL 宏操作同一寄存器位，应用层代码，不修改 HAL 库文件）。第②步 `HAL_TIM_PWM_Stop` 在 CEN 已清除的情况下仅关闭通道输出使能位，不影响计数器状态。

## 6. 接口与调用关系说明

| 项目 | T03 结论 |
|---|---|
| 接口变化 | 无新增、无改名、无签名变化；13 个接口由空实现变为完整实现 |
| 本模块内部 | 无私有函数、无私有变量（保持 §1.3.1 职责边界） |
| 被调用关系 | T03 阶段暂无调用方触发（MD-04 的调用在 T04/T06 接入）；接口就绪后由 T04 起使用 |
| HAL 依赖 | 经 app_bsp.c 包含的 5 个外设头文件获得句柄与 API；不修改 HAL 库 |
| 与 T04 衔接 | T04 计划使用 `BSP_StartRealtimeChain()`（直启实时链路）与 `BSP_ToggleLed()`（验证 10 kHz），本实现完全满足 |

## 7. 验证方法

### 7.1 编译与链接验证

- Debug 配置重新构建：预期编译 0 错误 0 警告（全部接口均已实现，无未使用函数告警——13 个接口均有外部声明，链接器按需保留），链接通过生成 `ADC_Sync_PWM.elf`。

### 7.2 板级验证（T04 接入后执行）

T03 仅实现底层接口，任务清单 T03 只要求编译通过。**板级现象验证安排在 T04**：T04 在 `APP_Control_Init()` 临时调用 `BSP_StartRealtimeChain()`，并在注入组回调中临时调用 `BSP_ToggleLed()`，预期 PA5 出现约 5 kHz 方波（LED 近似常亮），验证实时链路启动时序正确。若开发者在 T03 阶段希望提前目视验证，可在调试器 Expression 窗口手动调用 `BSP_StartRealtimeChain()` / `BSP_StopRealtimeChain()` 观察 PWM 输出，但不作为 T03 验收项。

## 8. 实施顺序建议

1. 重写 `App/Src/app_bsp.c`（按 §5.2 全文，含 5 个外设头文件包含；app_bsp.h 不动）；
2. 执行 §7.1 编译与链接验证；
3. 记录验证结果，待批准后进入 T04。

## 9. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 接口数量口径（I-01） | 任务清单写 11 个、详细设计为 13 个。本计划按 13 个实现（T02 已按 13 个声明并验证通过），请开发者批准时确认此口径 |
| 启停顺序与 HAL 行为 | 停止顺序按 2026-08-24 审核意见：第①步直接清 CEN 停止计数器并切断 TRGO2（HAL 宏 `__HAL_TIM_DISABLE` 在通道使能时不清 CEN，故用等效寄存器写法），第②步停 PWM/PWMN 通道，第③步停 ADC 注入组 |
| §3.5.1 的"待确认"项 | 详细设计对"CH2 的 PWM_Start 是否需在 CH1 之前单独调用"标注待确认（I-04）。本计划按顺序调用实现；CH2/CH3 为预设占空比输出，启动瞬间一个函数内的先后差异不影响功能，如开发者有异议请在批准时提出 |
| DAC 停止时序 | `BSP_StopDac` 先置零再关闭，对应设计 D-02 修复结论（避免 PA4 保持最后电平） |
| TIM6 中断 | `HAL_TIM_Base_Start_IT` 依赖 CubeMX 已使能的更新中断；本计划已实测核实 tim.c 中 TIM6_DAC_IRQn 优先级 1 已配置 |
| 错误处理 | 纯封装层不做 HAL 状态错误分支（上层模块职责）；`BSP_StartDmaTx` 按接口定义返回启动成功与否（HAL_OK 判定） |
