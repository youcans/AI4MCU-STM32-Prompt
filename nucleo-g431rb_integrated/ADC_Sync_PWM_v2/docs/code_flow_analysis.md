# 关键调用链与数据流分析

---

## 0. 文档说明

本文档基于软件说明书 [`docs/software_spec_SE.md`](software_spec_SE.md) 与结构分析 [`docs/code_structure_analysis.md`](code_structure_analysis.md)，进一步解析当前工程的**运行期**行为：主要运行链路、关键数据流、共享数据关系，并据此做设计一致性检查。

- 静态的目录结构、文件职责、模块划分与公开接口已在 `code_structure_analysis.md` 确认，本文不再重复展开。
- 所有运行期结论以**当前实际源码**为准，不根据缺失信息推测。
- 本轮只进行调用链、数据流与设计一致性分析，未修改任何程序文件、`.ioc` 文件或软件说明书。

### 0.1 运行期基础事实（一致性检查前提）

| 事实 | 来源 | 值 |
|---|---|---|
| SYSCLK / HCLK / TIM 时钟 | main.c `SystemClock_Config` + .ioc RCC | 160 MHz |
| NVIC 优先级组 | .ioc `NVIC.PriorityGroup` | PRIORITYGROUP_4（4 位抢占，0 子优先级） |
| ADC1_2_IRQn 抢占优先级 | adc.c:173 / .ioc | 0（最高） |
| TIM6_DAC_IRQn | tim.c:178 / .ioc | 1 |
| DMA1_Channel2_IRQn | dma.c:48 / .ioc | 2 |
| LPUART1_IRQn | usart.c:128 / .ioc | 3 |
| EXTI15_10_IRQn（按键） | gpio.c:70 / .ioc | 4（最低） |
| TIM1 周期 → ADC 注入组触发 | tim.c TRGO2=OC4REF；adc.c 注入外部触发 T1_TRGO2 | PWM 周期中点 10 kHz |
| TIM6 TRGO → ADC 规则组触发 | tim.c TRGO=UPDATE；adc.c 规则外部触发 T6_TRGO | 500 Hz |
| 主循环 | main.c:106 | 空 `while(1)`，纯中断驱动 |

> 由优先级可见：**ADC 中断（注入/规则，优先级 0）可打断 TIM6（1）、DMA（2）、LPUART1（3）、EXTI（4）中的任意一个**；TIM6 可打断 DMA/UART/EXTI；按键优先级最低。这一抢占关系决定了多链路并发时的执行顺序，是数据流分析的关键。

---

## 1. 主要运行链路

系统为中断驱动架构，主循环空转。运行期由 5 条链路构成，外加 1 条上电启动链路。各链路的触发源、函数调用序列与最终行为如下。

### 1.1 链路 L0：上电启动链路（一次性）

```
main()
  → HAL_Init() / SystemClock_Config()
  → MX_GPIO_Init / MX_DMA_Init / MX_TIM1_Init / MX_ADC1_Init / MX_LPUART1_UART_Init / MX_DAC1_Init / MX_TIM6_Init   [CubeMX, 配置外设但不启动计数/转换]
  → APP_Control_Init()  [MD-04]
       ├─ system_state = STOP
       ├─ BSP_WritePwmDutyCh2(800) / BSP_WritePwmDutyCh3(1600)   [写 CCR，但 TIM1 未启动，无输出]
       ├─ BSP_SetLedLow()  → LD2 熄灭                            [LED 初始输出已在 gpio.c 置 RESET]
       ├─ BSP_StopDac()     → DAC 置 0 并关闭
       ├─ BSP_StartRegularAdc()  → HAL_ADCEx_Calibration_Start(单端) + HAL_ADC_Start_IT(hadc1)  [启动规则组温度采样, 触发源 TIM6_TRGO]
       └─ BSP_StartTim6()   → HAL_TIM_Base_Start_IT(htim6)        [启动 500 Hz 周期中断, 同时其 TRGO 触发规则组]
  → while(1) 空转
```

- **触发源**：上电/复位。
- **关键点**：启动后 **TIM1 尚未启动**，PWM 无输出、注入组未启动；只有 **TIM6（500 Hz）和规则组温度采样** 处于运行态。系统停在 STOP。
- **最终行为**：进入 STOP 状态，温度以 500 Hz 采样，串口以 500 Hz 发送温度（待首次温度有效后），LED 熄灭。

### 1.2 链路 L1：动态采样—处理链路（仅 RUN，10 kHz）

```
TIM1 周期事件(中心对齐中点) → TRGO2=OC4REF ─触发─▶ ADC1 注入组(3 通道 IN2/IN12/IN15)
  → 转换完成 → ADC1_2_IRQn(优先级0) → HAL_ADC_IRQHandler → HAL_ADCEx_InjectedConvCpltCallback  [MD-02]
       ├─ 读取 ch1=IN2(Rank1) / ch2=IN12(Rank2) / ch3=IN15(Rank3)
       ├─ PROC_HandleSamples(ch1,ch2,ch3)  [MD-03]
       │     ├─ 计算 duty = ch1 * 8000 / 4095  → BSP_WritePwmDutyCh1(duty)  [CCR1 实时更新]
       │     └─ dac_amplitude = ch1            [暂存 DAC 方波幅值]
       └─ CTRL_NotifyInjectedComplete()  [MD-04]
             └─ 若 RUN → BSP_ToggleLed()  [LED 10 kHz 翻转]
```

- **触发源**：TIM1 周期事件经 TRGO2（约 10 kHz，PWM 周期中点由 CH4 OC4REF 产生）。
- **仅 RUN 有效**：注入组中断转换只在 `BSP_StartRealtimeChain`（进入 RUN）时 `HAL_ADCEx_InjectedStart_IT` 启动；STOP/FAULT 时 `InjectedStop_IT` 关闭，本链路不产生。
- **最终行为**：动态电压（IN2）→ PWM CH1 占空比实时更新 + DAC 方波幅值暂存 + LED 翻转。采样、计算、PWM 更新、DAC 幅值更新维持 10 kHz 同步周期。

### 1.3 链路 L2：500 Hz 周期调度链路（全状态，500 Hz）

```
TIM6 周期溢出 → TIM6_DAC_IRQn(优先级1) → HAL_TIM_IRQHandler → HAL_TIM_PeriodElapsedCallback  [MD-04]
   (htim->Instance==TIM6 才处理)
   ① 若 RUN:  dac_toggle = !dac_toggle
                → BSP_WriteDac( dac_toggle ? PROC_GetDacAmplitude() : 0 )   [DAC 250 Hz 方波]
   ② 若 FAULT: led_div_counter++；满 25(=LED_FAULT_DIVIDER) 则清零并 BSP_ToggleLed()  [LED 10 Hz 闪烁]
   ③ 数据源选择:
        若 RUN:     MON_SendData((float)PROC_GetDacAmplitude())  [发 DAC 幅值]
        若 STOP:    若 SAMP_IsTemperatureValid() → MON_SendData(SAMP_GetTemperature())  [发温度]
        若 FAULT:   (不发送)
```

- **触发源**：TIM6 周期溢出（500 Hz），全状态运行（L0 启动后不停）。
- **关键点**：本链路同时承担三项 500 Hz 任务：DAC 方波（RUN）、LED 异常分频（FAULT）、串口数据源选择（RUN/STOP）。DAC 方波靠 500 Hz 翻转得 250 Hz；LED 闪烁靠 25 分频得 20 次翻转/s = 10 Hz。
- **最终行为**：按状态驱动 DAC 输出、LED 闪烁节奏、串口 500 Hz 发送内容。

### 1.4 链路 L3：温度采样链路（全状态，500 Hz）

```
TIM6 TRGO(UPDATE) ─触发─▶ ADC1 规则组(温度传感器 IN16)
  → 转换完成 → ADC1_2_IRQn(优先级0) → HAL_ADC_IRQHandler → HAL_ADC_ConvCpltCallback  [MD-02]
       ├─ adc_raw = HAL_ADC_GetValue
       ├─ ts_data_3v = raw * 3300 / 3000          [VDDA 3.3V → 3.0V 归一化, 匹配工厂校准基准]
       ├─ temp = (ts_data_3v - TEMP30_CAL)*100 / (TEMP130_CAL - TEMP30_CAL) + 30   [两点线性插值]
       ├─ chip_temperature = temp; temperature_valid = true
       └─ CTRL_ProcessTemperature(temp)  [MD-04]
             ├─ 若 FAULT 且 temp < 40:  → STOP, BSP_SetLedLow(), led_div_counter=0
             └─ 若 非FAULT 且 temp > 45:
                   若 RUN: BSP_StopRealtimeChain() + BSP_StopDac()
                   → FAULT
```

- **触发源**：TIM6 TRGO（UPDATE，500 Hz），全状态运行。
- **并发关系**：L3 与 L2 共用 TIM6——TIM6 周期溢出同时产生两件事：周期中断（L2，经 `TIM6_DAC_IRQn`，优先级1）与 TRGO=UPDATE（触发 L3 规则组，转换完成经 `ADC1_2_IRQn`，优先级0）。两者分属不同中断：L2 在 TIM6 中断中执行，L3 在 ADC 中断中执行。由于 ADC 优先级(0)高于 TIM6(1)，**L3 的 ADC 中断可抢占 L2 的 TIM6 中断**——即一次 TIM6 周期到来后，L2 可能在执行途中被 L3 抢占。
- **最终行为**：温度 500 Hz 采集换算 → 持存 + 喂给状态机做异常判断/恢复，驱动状态切换。

> 注意 L1（注入组完成）与 L3（规则组完成）共用 `ADC1_2_IRQn`，经 `HAL_ADC_IRQHandler` 由 HAL 依据 ADC 状态标志进行回调分发，二者在 NVIC 层面不存在相互抢占；二者的相对执行先后取决于触发相位与中断到达时刻，仅凭当前资料不宜直接认定固定顺序。但任一 ADC 回调（优先级 0）**都可能抢占 TIM6/DMA/UART/EXTI 中断**。

### 1.5 链路 L4：按键状态切换链路（异步，EXTI）

```
PC13 上升沿(按键 B1) → EXTI15_10_IRQn(优先级4, 最低) → HAL_GPIO_EXTI_IRQHandler → HAL_GPIO_EXTI_Callback  [MD-04]
   (GPIO_Pin==B1_Pin 才处理)
   ├─ 若 FAULT: return  [异常态不响应]
   ├─ now = HAL_GetTick(); 若 (now - last_button_tick) < 50ms: return  [软件消抖]
   └─ 切换:
        STOP→RUN:  system_state=RUN; BSP_StartRealtimeChain() [+ TIM1 启动/注入组/PWM]; BSP_StartDac()
        RUN→STOP:  system_state=STOP; BSP_StopRealtimeChain(); BSP_StopDac(); BSP_SetLedLow()
```

- **触发源**：PC13 上升沿（用户按键），异步。
- **关键点**：按键中断优先级最低（4），执行期间可被任意 ADC/TIM6/DMA 中断抢占。这是潜在的并发竞态风险点（见 §5）。`BSP_StartRealtimeChain` 内先 `HAL_ADCEx_InjectedStart_IT` 再 `HAL_TIM_PWM_Start×3` + `PWMN_Start×3`；`BSP_StopRealtimeChain` 先清 `TIM1 CR1.CEN` 切断触发源，再停 PWM/互补，最后 `InjectedStop_IT`。
- **最终行为**：在 STOP 与 RUN 之间切换，连带启停实时链（PWM+注入采样）与 DAC。

### 1.6 链路 L5：串口 DMA 发送完成链路（异步，后处理）

```
L2 中 MON_SendData 发起 → HAL_UART_Transmit_DMA(hlpuart1, 8字节 JustFloat 帧)
   → DMA 搬运完成 → DMA1_Channel2_IRQn(优先级2) → HAL_DMA_IRQHandler(hdma_lpuart1_tx)
        ├─ DMA 传输结束, 通知 UART（HAL 内部 link 机制）
        ▼
   → UART TC 中断 → LPUART1_IRQn(优先级3) → HAL_UART_IRQHandler → HAL_UART_TxCpltCallback  [MD-05]
        └─ dma_busy = false  [释放发送通道, 允许下一帧]
```

- **触发源**：DMA 搬运完成 → UART 传输完成（TC）。两段式：DMA 中断先结束搬运，再由 UART TC 中断进入回调。
- **关键点**：`dma_busy` 在 `MON_SendData`（L2/TIM6 上下文）置位，在 `HAL_UART_TxCpltCallback`（LPUART1 中断上下文）清除，跨 ISR 共享故 `volatile`。若前一帧未发完，`MON_SendData` 直接返回 false 丢弃本次数据。
- **最终行为**：异步完成一帧发送并释放忙标志。

---

## 2. 关键数据流

### 2.1 数据流 DF-1：动态电压 → PWM 占空比 + DAC 幅值

```
[外部] 电位器(PA1/IN2) ─采样─▶ ch1(注入 Rank1, 12bit 原始)
   │  (L1: HAL_ADCEx_InjectedGetValue Rank1)
   ▼
PROC_HandleSamples(ch1, ...)  [MD-03, 注入完成 ISR 上下文]
   ├─ duty = ch1 * 8000 / 4095  ──▶ BSP_WritePwmDutyCh1 ──▶ TIM1 CCR1  [PWM CH1 占空比, 实时输出]
   └─ dac_amplitude = ch1        ──▶ (暂存 volatile uint16_t)
                                      │
                                      │ (L2: PROC_GetDacAmplitude 读取, 500 Hz)
                                      ▼
                                 dac_toggle 翻转 ──▶ BSP_WriteDac(amplitude 或 0) ──▶ DAC1_OUT1 [250 Hz 方波]
                                      │
                                      │ (L2: 同一 TIM6 周期, 串口数据源)
                                      ▼
                                 MON_SendData((float)PROC_GetDacAmplitude()) ──▶ JustFloat ──▶ LPUART1 [500 Hz 串口]
```

- **产生**：ch1 来自 IN2 注入采样（L1）。
- **处理**：MD-03 线性映射（×8000/4095）得 PWM CH1 占空比；1:1 直通得 DAC 幅值。
- **使用**：三处——PWM CH1 输出（10 kHz 实时）、DAC 方波幅值（L2/500 Hz 读取翻转）、串口发送内容（L2/500 Hz，发幅值 float）。
- **未使用项**：ch2（IN12 高参考）、ch3（IN15 低参考）传入 `PROC_HandleSamples` 后被 `(void)ch2;(void)ch3;` 显式丢弃，当前不参与动态计算。

### 2.2 数据流 DF-2：芯片温度 → 状态机 + 串口

```
[内部] 温度传感器(IN16) ─采样─▶ adc_raw(规则组)
   │  (L3: HAL_ADC_GetValue)
   ▼
HAL_ADC_ConvCpltCallback  [MD-02, 规则完成 ISR 上下文]
   ├─ ts_data_3v = raw * 3300/3000           [3.3V→3.0V 归一化]
   ├─ temp = 两点插值(30°C/130°C 工厂校准) + 30
   ├─ chip_temperature = temp; temperature_valid = true  [暂存 volatile]
   ├─ ▶ CTRL_ProcessTemperature(temp)  [状态机输入]
   │      ├─ >45°C: RUN→(停链+停DAC)→FAULT
   │      └─ <40°C(仅FAULT): →STOP
   └─ (供 L2 读取)
                    │ (L2: SAMP_IsTemperatureValid/SAMP_GetTemperature, 500 Hz, 仅 STOP)
                    ▼
               MON_SendData(SAMP_GetTemperature()) ──▶ JustFloat ──▶ LPUART1 [STOP 态发温度]
```

- **产生**：IN16 规则组采样（L3，500 Hz，全状态）。
- **处理**：VDDA 归一化 + 两点插值。
- **使用**：两处——喂状态机做异常判断/恢复（L3 内同步）、串口发送内容（L2，仅 STOP 态发温度 float）。

### 2.3 数据流 DF-3：状态 → 各链路行为分支

```
system_state(volatile, MD-04) ──被以下链路读取──▶
   ├─ L1: CTRL_NotifyInjectedComplete  [仅 RUN 翻转 LED]
   ├─ L2: HAL_TIM_PeriodElapsedCallback [RUN: DAC翻转+发幅值; FAULT: LED分频; STOP: 发温度]
   ├─ L4: HAL_GPIO_EXTI_Callback       [FAULT 不响应; STOP↔RUN 切换]
   └─ L3: CTRL_ProcessTemperature      [据当前态决定进入/恢复动作]
system_state ──被写入──▶ L4(按键切换) / L3(温度切换)
```

- `system_state` 是全部链路的中心分支变量，4 条中断链路读、2 条写。`volatile` 保证跨 ISR 可见性。

### 2.4 数据流 DF-4：串口发送帧

```
L2 选定 value(float) ──▶ MON_SendData(value)  [MD-05]
   ├─ 若 dma_busy: return false(丢弃)
   ├─ MON_BuildJustFloatFrame(value, tx_frame[8]):
   │      [0..3] = float value (memcpy, little-endian)
   │      [4..7] = 00 00 80 7F (IEEE754 +∞, JustFloat 帧尾)
   └─ BSP_StartDmaTx(tx_frame, 8) → HAL_UART_Transmit_DMA(hlpuart1)
            ├─ 成功: dma_busy=true; 交 L5 异步发完
            └─ 失败: return false
```
- JustFloat 帧 = 4 字节 float + 4 字节帧尾，共 8 字节，DMA 发送。

---

## 3. 共享数据关系

工程中跨链路（跨 ISR）共享的数据，及其产生方、使用方、跨链路关系如下。全部声明 `volatile` 修饰。

| 共享数据 | 类型 | 所在模块 | 产生方（写） | 使用方（读） | 跨链路关系 | 关键观察 |
|---|---|---|---|---|---|---|
| `dac_amplitude` | `volatile uint16_t` | MD-03 | L1 注入完成 ISR（PROC_HandleSamples，10 kHz） | L2 TIM6 ISR（PROC_GetDacAmplitude，500 Hz，用于 DAC 翻转 + 串口） | L1 写 → L2 读，跨链路 | 写快读慢（10k 写/500 读），无竞争；但 L2 在单次周期内**连续读取 2 次**（① DAC 翻转用，③ 串口发幅值用），两次间 L1 可能更新使其不一致 |
| `chip_temperature` | `volatile float` | MD-02 | L3 规则完成 ISR（500 Hz） | L2 TIM6 ISR（SAMP_GetTemperature，500 Hz，仅 STOP） | L3 写 → L2 读，同频不同源 | L3 的 ADC 中断(优先级0)可抢占 L2 的 TIM6 中断(优先级1)，故 L2 读取的是最近一次已经完成的温度采样值（未必是本周期刚完成的值） |
| `temperature_valid` | `volatile bool` | MD-02 | L3 规则完成 ISR（首次置 true 后不再清） | L2 TIM6 ISR（SAMP_IsTemperatureValid） | L3 写 → L2 读 | 单向置位，L2 仅在有效后发送温度，避免冷启动发未定义值 |
| `dma_busy` | `volatile bool` | MD-05 | L2 TIM6 ISR 置位（MON_SendData 成功） | L5 LPUART1 ISR 清除（TxCpltCallback）；L2 读之以决定丢弃 | L2 写 → L5 清，跨链路异步 | L2(优先级1)**高于** L5(优先级3)，发送期间 L2 可抢占 L5；若帧发送未完成而下一 500 Hz 到来，L2 见 `dma_busy` 丢弃该帧 |
| `system_state` | `volatile enum` | MD-04 | L4 按键 ISR、L3 温度 ISR | L1/L2/L3/L4 全部读取 | 多读二写，跨链路 | 分支中心变量；写发生在 L4/L3，读发生在各链路 |
| `led_div_counter` | `volatile uint8_t` | MD-04 | L2 TIM6 ISR（FAULT 态递增）、L3 温度 ISR（恢复时清零） | L2（自身判定满 25） | L2 写/读 + L3 清，跨链路 | 恢复退出 FAULT 时由 L3 清零，保证下次进 FAULT 重新分频 |
| `last_button_tick` | `uint32_t`（非 volatile） | MD-04 | L4 按键 ISR | L4 按键 ISR（自读自写） | 仅 L4 内 | 单链路私有，无跨链路共享，故未加 volatile |
| `dac_toggle` | `bool`（非 volatile） | MD-04 | L2 TIM6 ISR | L2 自身 | 仅 L2 内 | 单链路私有（仅 RUN 且 DAC 翻转逻辑在 TIM6 中），未加 volatile |

**共享数据关系要点：**
- 真正跨链路共享的有 5 个（`dac_amplitude`、`chip_temperature`、`temperature_valid`、`dma_busy`、`system_state`），均已 `volatile`。
- `led_div_counter` 跨 L2/L3 共享也已 `volatile`。
- 单链路私有的 `last_button_tick`、`dac_toggle` 未加 `volatile`，符合其使用域。
- **`system_state` 缺少临界区保护**：它被 L4/L3 写、被 L1/L2 读，写操作虽为单条赋值（原子性可接受），但"读-判-写"复合操作（如 L4 先读 state 再写 state、L3 先读 state 再分支写）在多 ISR 并发下理论存在 TOCTOU。当前优先级使 L3(0) 可抢占 L4(4)，详见 §5。

---

## 4. 设计一致性检查

将说明书的主要运行链路、输入输出关系、状态转换、数据流与工程实际比对。

### 4.1 主要运行链路一致性

| 说明书要求 | 工程链路 | 一致性 |
|---|---|---|
| FR-01/SF-02：10 kHz 三组带死区互补 PWM，CH1 动态更新，CH2/3 预设；产生 ADC 采样触发 | L1 触发机制（TIM1 TRGO2=OC4REF 触发注入组）；CH1 由 L1 实时更新 CCR1；CH2/3 在 L0 写预设 800/1600 | 一致 |
| FR-02/SF-03/SF-04：PWM 周期事件触发三通道注入序列采样，仅 RUN；序列完成后读取三路注入结果 | L1：同一 PWM 触发事件启动 3 通道注入序列（IN2/IN12/IN15 在单 ADC 内按序转换，非同时完成），由 `InjectedStart/Stop_IT` 随 RUN 启停；完成回调读取 Rank1/2/3 | 一致 |
| FR-03/SF-05：AdcSequenceCompleteEvent 触发实时处理，更新 CH1 占空比与 DAC 幅值，10 kHz 稳定周期 | L1：注入完成回调→`PROC_HandleSamples`→更新 CCR1 + 暂存 `dac_amplitude`，10 kHz | 一致 |
| FR-04/SF-06：250 Hz 参考方波，幅值随动态电压更新，运行启停 | L2①：500 Hz 翻转 `dac_toggle` 得 250 Hz，幅值取 `PROC_GetDacAmplitude`；L4/L3 启停 DAC | 一致 |
| FR-05/SF-07：内部温度 500 Hz，全状态持续 | L3：规则组 IN16 由 TIM6_TRGO 500 Hz 触发，L0 即启动不随状态停 | 一致 |
| FR-06/SF-08：温度超限进异常、恢复返停止 | L3 内 `CTRL_ProcessTemperature`：>45 进 FAULT、<40 退 FAULT | 一致 |
| FR-07/SF-09：三状态，按键切 STOP↔RUN，温度触发异常进/退，异常不响应按键 | L4 + L3 状态机；L4 FAULT 直接 return | 一致 |
| FR-08/SF-10：停止熄灭、运行翻转~常亮、异常 10 Hz 闪烁 | L1 运行态 LED 翻转(10k)；L2② FAULT 态 25 分频得 10 Hz；L4/L3 进 STOP 调 SetLedLow | 一致 |
| FR-09/SF-11：JustFloat/115200/500 Hz，运行发 DAC 值、停止发温度、异常不发 | L2③ 数据源选择 + DF-4 组包；LPUART1 115200 | 一致 |

**主要运行链路与说明书一致。** 仅链路 L5（DMA+TC 两段式发送完成）属工程实现细节，说明书未描述链路内部，不构成冲突。

### 4.2 输入输出关系一致性

| 说明书对象 | 工程数据流 | 一致性 |
|---|---|---|
| 输入数据 AdcDynamicVoltage(IN2) | DF-1：ch1=Rank1=IN2 | 一致 |
| 输入数据 AdcHighRef(IN12)/AdcLowRef(IN15) | DF-1：ch2/ch3 读入后被丢弃，未参与计算 | 一致（说明书亦称"当前不参与动态计算"） |
| 输入数据 ChipTemperature(IN16) | DF-2：规则组 IN16 | 一致 |
| 输出 PwmDutyCh1 | DF-1：CCR1 经 `PROC_HandleSamples` 更新 | 一致 |
| 输出 PwmDutyCh2/3 | L0 预设 800/1600 | 一致 |
| 输出 DacOutputValue | DF-1：`dac_amplitude`，L2 翻转写入 | 一致 |
| 输出 LedOutput | DF-3 + L1/L2 | 一致 |
| 输出 SerialTxData | DF-4：JustFloat 8 字节 | 一致 |
| 输出 SystemState | DF-3：`system_state` | 一致 |
| 输出事件 TemperatureFaultEnter/Exit | L3 `CTRL_ProcessTemperature` 中状态切换隐含产生 | 一致 |

### 4.3 状态转换一致性

| 说明书状态转换规则 | 工程 | 一致性 |
|---|---|---|
| STOP↔RUN 由按键切换 | L4：EXTI 回调 + 50ms 消抖 | 一致（消抖为额外实现，说明书未禁止） |
| 任状态 → FAULT（温度>阈值） | L3：`CTRL_ProcessTemperature`，若 RUN 先停链+停 DAC | 一致 |
| FAULT → STOP（温度<恢复阈值） | L3：清 LED、清分频计数 | 一致 |
| FAULT 不响应按键 | L4：FAULT 直接 return | 一致 |
| 返回 STOP 后需再按键进 RUN | L3 恢复只置 STOP，不进 RUN | 一致 |
| 阈值 | 说明书 TBD；工程 45/40 | 一致（补全 TBD，不矛盾） |

状态机转换图（工程实际）：

```
            按键(L4)                温度>45°C(L3, 任非FAULT态)
   STOP ──────────▶ RUN ──────────────────────▶ FAULT
     ▲              ▲                            │
     │  按键(L4)    │                            │ 温度<40°C(L3)
     └──────────────┘                            ▼
        (RUN→STOP)                            STOP
                                               (FAULT 仅能→STOP, 按键无效)
```

**与说明书状态切换规则一致。**

### 4.4 数据流一致性

- DF-1 动态电压→PWM/DAC/串口：与说明书 SF-05/SF-06/SF-11 一致（CH1 线性映射、DAC 1:1、串口发幅值）。
- DF-2 温度→状态机/串口：与说明书 SF-07/SF-08/SF-11 一致（500 Hz、全状态、STOP 发温度）。
- DF-3 状态分支：与说明书各状态行为表一致。
- DF-4 串口帧：JustFloat 格式与说明书一致。

### 4.5 本轮新发现的差异/注意点

> 以下为聚焦"调用链/数据流"层面新识别的事项，不与 `code_structure_analysis.md` 已有差异重复。双方事实并列，不修改任何一方。

#### 注意点 N-1：`dac_amplitude` 在单个 TIM6 周期内被读取两次的潜在不一致

- **工程实际**：L2 一次 TIM6 周期内先 `BSP_WriteDac(dac_toggle ? PROC_GetDacAmplitude() : 0)`（①DAC），后 `MON_SendData((float)PROC_GetDacAmplitude())`（③串口）。两次调用 `PROC_GetDacAmplitude()` 之间，L1 注入完成 ISR（优先级 0）可能抢占并更新 `dac_amplitude`，导致 DAC 写入的幅值与串口发送的幅值不是同一采样值。
- **说明书描述**：SF-05 要求"模拟电压采样、数据处理、PWM 更新和 DAC 更新之间保持 10 kHz 稳定周期关系"，SF-06/SF-11 对 DAC 与串口数据来源均为 `DacOutputValue`。说明书未要求单次串口发送的值与该周期 DAC 写入值严格同源。
- **定级**：两次读取之间是否实际发生 ADC 中断抢占，取决于两个中断的相对相位与 TIM6 回调实际执行时间，不能仅据 10 kHz/500 Hz 频率比判定发生概率；属一致性层面值得记录的注意点，不构成功能错误。

#### 注意点 N-2：`system_state` 读-判-写复合操作的并发性

- **工程实际**：L4 按键 ISR 与 L3 温度 ISR 都对 `system_state` 做"先读当前态、再分支写新态"。L3 优先级(0)高于 L4(4)，若 L4 正在执行 STOP→RUN 切换途中（已读为 STOP、尚未写 RUN、尚未 `BSP_StartRealtimeChain`），被 L3 抢占且温度超限，L3 会读为 STOP（不进 `if RUN` 分支）直接置 FAULT；L3 返回后 L4 继续写 RUN 并启动实时链 → 状态与硬件状态不匹配（state=RUN 但实际应 FAULT，PWM/采样已启动）。
- **说明书描述**：SF-09 描述了状态切换的规则，未涉及中断并发下的原子性要求。
- **定级**：理论边界场景，需实际触发条件（按键与温度超限同时发生）才暴露；属潜在一致性风险点，列入需运行验证。

#### 注意点 N-3：DAC 幅值类型从 12bit 整数转 float 发送的精度差异

- **工程实际**：`dac_amplitude` 为 `uint16_t`（取值为 12bit ch1 原始值 0~4095），串口侧 `MON_SendData((float)PROC_GetDacAmplitude())` 转 float 发送；而 DAC 写入侧 `BSP_WriteDac(dac_toggle ? PROC_GetDacAmplitude() : 0)` 用整数。即上位机看到的是 0~4095 的整数 float（非换算电压）。
- **说明书描述**：SF-11 仅说"发送 `DacOutputValue`"，未定义其数值含义/单位。
- **定级**：不构成冲突，但上位机解析时需知该 float 为 0~4095 原始量纲。列为说明类。

---

## 5. 需运行验证的内容

以下仅凭源码与现有资料无法确认，需实际编译/烧录/运行或仪器测量验证：

1. **PWM 输出实测**（FR-01）：PA8/PA9/PA10 与 PB13/PB14/PB15 三组互补输出是否为 10 kHz 中心对齐、死区时间是否符合预期（DeadTime=32 对应 ns 级），CH1 占空比是否随 PA1 电位器实时变化、CH2=10%/CH3=20% 是否稳定。
2. **ADC 注入组采样时序**（FR-02/SF-05）：同一 PWM 触发事件启动的三通道注入序列是否在每个 PWM 周期中点稳定转换完成且不丢点；电位器从 0~3.3V 扫动时 CH1 占空比是否线性（0~8000/满量程）。
3. **DAC 250 Hz 方波**（FR-04/SF-06）：PA4 在 RUN 态是否输出 250 Hz、幅值在 0 与 `dac_amplitude` 之间翻转的方波，幅值是否随 PA1 变化；确认 STOP/FAULT 状态下 DAC 关闭（`BSP_StopDac` 软件上先写 0 再关闭，关闭后 PA4 不再由 DAC 驱动）。
4. **温度采样准确度**（FR-05）：换算公式（3.3V 归一化 + 30/130°C 两点插值）所得温度与实测芯片温度的偏差；500 Hz 是否持续、全状态不停。
5. **温度异常/恢复阈值切换**（FR-06/SF-08/SF-09）：人为加热使芯片温度>45°C 是否进 FAULT（PWM/DAC/采样停、LED 10 Hz 闪烁、串口停发）；冷却至<40°C 是否返 STOP（LED 灭、串口转发温度）；FAULT 期间按键是否确实无效。
6. **状态切换与 LED**（FR-07/FR-08）：按键 STOP↔RUN 切换是否可靠（含 50 ms 消抖是否合适），RUN 时 LED 近似常亮（10 kHz 翻转），STOP 熄灭。
7. **串口 JustFloat 帧**（FR-09/SF-11）：VOFA+ 端是否在 115200/500 Hz 下正确解析为 1 通道 float；RUN 时收到的是 0~4095 的 DAC 幅值 float（见 N-3），STOP 时收到温度 float（°C），FAULT 时无数据。需确认上位机数据量纲预期与工程实际一致。
8. **并发边界场景**（N-2）：在按键切换（L4）执行途中人为制造温度超限中断（L3 抢占），观察是否出现状态与硬件不一致（state=RUN 但应 FAULT）。此场景需刻意构造，源码层面无法确认。
9. **DMA 发送背压行为**（N-1/L5）：当 500 Hz 发送速率下 LPUART1 DMA 未及时完成时，`dma_busy` 丢弃是否导致 VOFA+ 波形丢点；8 字节帧在 115200 bps 下发送耗时（约 0.69 ms）是否始终小于 2 ms 周期，确保不常态背压。
10. **按键进入 RUN 的实时链启动顺序**（L4 + `BSP_StartRealtimeChain`）：先 `InjectedStart_IT` 再 `PWM_Start`，首个 PWM 周期是否产生预期首次注入触发；退出 RUN 时先清 `CR1.CEN` 切断触发再停通道，是否彻底停输出无毛刺。

---

## 6. 附：链路—中断—优先级对照表

| 链路 | 中断(IRQn) | 抢占优先级 | HAL 入口 | 应用入口（回调/函数） | 频率/触发 | 有效状态 |
|---|---|---|---|---|---|---|
| L1 动态采样处理 | ADC1_2_IRQn | 0 | `HAL_ADC_IRQHandler` | `HAL_ADCEx_InjectedConvCpltCallback` [MD-02] | 10 kHz（TRGO2） | 仅 RUN |
| L2 500Hz 调度 | TIM6_DAC_IRQn | 1 | `HAL_TIM_IRQHandler` | `HAL_TIM_PeriodElapsedCallback` [MD-04] | 500 Hz | 全状态 |
| L3 温度采样 | ADC1_2_IRQn | 0 | `HAL_ADC_IRQHandler` | `HAL_ADC_ConvCpltCallback` [MD-02] | 500 Hz（TRGO） | 全状态 |
| L4 按键切换 | EXTI15_10_IRQn | 4 | `HAL_GPIO_EXTI_IRQHandler` | `HAL_GPIO_EXTI_Callback` [MD-04] | 异步 | STOP/RUN（FAULT 不响应） |
| L5 串口发送完成 | DMA1_Ch2 + LPUART1 | 2 / 3 | `HAL_DMA_IRQHandler` → `HAL_UART_IRQHandler` | `HAL_UART_TxCpltCallback` [MD-05] | 异步（L2 转发触发） | STOP/RUN；进入 FAULT 前已启动的发送可能继续完成 |
| L0 启动 | — | — | — | `APP_Control_Init` [MD-04] | 一次性 | 上电 |
