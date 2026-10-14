# 代码审查日志 — T06 未提交修改

> 本日志记录 T06 任务未提交代码修改的审查过程、核验依据与结论。
> 审查日期：2026-08-25
> 审查依据：`docs/tasks/programming_plan_T06.md`（2026-08-25 校订版）
> 审查范围：工作区相对于最近一次提交的差异，仅以下 3 个代码文件：
> - `App/Src/app_bsp.c`
> - `App/Src/app_control.c`
> - `App/Src/app_sampling.c`
> 其他文档修改不在本次审查范围内；审查过程未修改任何文件。

---

## 1. 差异内容与方案符合性

三个文件的 git diff 与方案 §5.1/§5.2/§5.3 的代码逐字一致，无超出方案的改动：

| 文件 | 方案要求 | 实际差异 | 结论 |
|---|---|---|---|
| `App/Src/app_bsp.c` | `BSP_StartRegularAdc()` 内、`HAL_ADC_Start_IT()` 之前补充 ADC1 校准 | 仅 +2 行校准调用及注释，接口声明未动 | 一致 |
| `App/Src/app_control.c` | 更新文件头；新增 `#include "tim.h"`；新增 `SystemState_t` 枚举、3 个配置常量、3 个私有变量；重写 `APP_Control_Init()` 为正规 STOP 初始化；新增 `HAL_TIM_PeriodElapsedCallback`、`HAL_GPIO_EXTI_Callback`；`CTRL_NotifyInjectedComplete`/`CTRL_ProcessTemperature` 保持并补注释 | 与 §5.1 逐字一致，无额外改动 | 一致 |
| `App/Src/app_sampling.c` | 更新文件头；新增温度校准宏、2 个私有变量；新增 `HAL_ADC_ConvCpltCallback`；实现 `SAMP_GetTemperature`/`SAMP_IsTemperatureValid`；注入组回调仅更新注释 | 与 §5.2 逐字一致；注入组回调逻辑零改动（diff 证实仅 2 行注释变化） | 一致 |

## 2. 专项核验（接口与调用关系）

1. **规则组 500 Hz 重复触发（最关键项）**：本工程规则组为「非连续 + 外部触发 T6_TRGO + EOC=序列结束」（`Core/Src/adc.c:53,55,58`），`HAL_ADC_Start_IT` 因此只使能 `ADC_IT_EOS`（`Drivers/STM32G4xx_HAL_Driver/Src/stm32g4xx_hal_adc.c:1814`）。G4 HAL 在中断处理中「序列结束后禁用 EOC/EOS 中断」的分支受 `LL_ADC_REG_IsTriggerSourceSWStart()` 门控（同文件 2342、2367-2381 行），该函数检查 CFGR.EXTEN（`Drivers/STM32G4xx_HAL_Driver/Inc/stm32g4xx_ll_adc.h:4683-4686`），本工程 EXTEN=上升沿 → 返回 0 → 禁用分支被跳过，EOS 中断保持使能，`HAL_ADC_ConvCpltCallback` 在每次 TIM6 触发后重复进入。500 Hz 温度链路成立。
2. **TS_CAL 地址**：`0x1FFF75A8`/`0x1FFF75CA` 与 HAL 官方 `TEMPSENSOR_CAL1_ADDR`/`TEMPSENSOR_CAL2_ADDR`（`stm32g4xx_ll_adc.h:366,369`）及校准温度 30°C/130°C（公式中 `30.0f`/`100.0f`）一致。
3. **温度换算公式**：`raw × 3300/3000` 归一化方向正确（3.3V 参考下同温度码值偏小，需 ×1.1 折算回 3.0V 校准标度）；线性插值公式与 ST 标准公式一致；两个校准值在相减前均已做 `(float)` 转换。
4. **校准调用**：`HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)` 签名存在（`stm32g4xx_hal_adc_ex.h:1491`），`ADC_SINGLE_ENDED` 已定义（同文件 391 行）；`hadc1` 经 `adc.h` 可见；调用时机满足「首次启动前校准」。
5. **中断链完整性**：`Core/Src/stm32g4xx_it.c` 中 `ADC1_2_IRQHandler→HAL_ADC_IRQHandler(&hadc1)`（222 行）、`TIM6_DAC_IRQHandler→HAL_TIM_IRQHandler(&htim6)`（250 行）、`EXTI15_10_IRQHandler→HAL_GPIO_EXTI_IRQHandler(B1_Pin)`（236 行）均已生成；优先级 ADC1_2=0、TIM6=1、EXTI=4 与方案 §1.3 一致。
6. **回调定义唯一性**：三个 HAL 回调各只有 App 层一处强定义，HAL 层均为 `__weak` 默认实现，无链接冲突。
7. **接口调用关系**：`APP_Control_Init()` 已接入 `main.c:101`（USER CODE 2，位于全部 MX 初始化之后）；Init 内调用的 `BSP_WritePwmDutyCh2/3`、`BSP_SetLedLow`、`BSP_StopDac`、`BSP_StartRegularAdc`、`BSP_StartTim6` 及按键回调调用的 `BSP_StartRealtimeChain`、`BSP_StartDac`、`BSP_StopRealtimeChain` 均在 `app_bsp.h` 声明且签名匹配；`PROC_HandleSamples(uint16_t,uint16_t,uint16_t)` 与实参强转匹配（`app_process.h:19`）；`B1_Pin` 经 `app_control.h→main.h` 可见；`bool` 经各模块头文件包含 `stdbool.h` 可用。
8. **软件消抖**：`(now - last_button_tick) < DEBOUNCE_MS` 为无符号运算，跨 tick 回绕安全；`last_button_tick`、`system_state` 仅 EXTI 回调上下文访问，无共享竞争。
9. **跨 ISR 共享变量**：`chip_temperature`、`temperature_valid` 均 `volatile` 修饰，32 位对齐原子访问，符合方案 §4 约定。
10. **运行/停止切换时序**：STOP→RUN 与 RUN→STOP 两侧调用组合与方案 §4 一致；RUN 状态下注入组（10 kHz，单次序列约 2.8 µs）挂起/延迟规则组转换（约 6.5 µs/2 ms 周期）不构成冲突；OVR=DATA_PRESERVED 下 500 Hz 结果均被及时读取清除，无溢出风险。

## 3. 审查结论

**未发现明确错误、严重遗漏、接口或调用关系错误，以及与 T06 修改方案不一致的问题。**

三个文件的未提交修改与校订版方案完全一致，三条事件链路（TIM6 500 Hz 调度、规则组温度采样、按键启停）的外设配置、HAL 行为与回调归属均经核验成立，可进入编译与板级验证（方案 §6）。
