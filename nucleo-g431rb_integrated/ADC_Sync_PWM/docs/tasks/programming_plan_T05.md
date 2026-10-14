# T05 实现方案 — 实现并验证 MD-03 实时处理

> 编制日期：2026-08-08
> 修订日期：2026-08-24（校订版；按开发者审阅意见确认 TBD-08/TBD-09 并修正 §6.2 示例）
> 依据文件：
> - `docs/tasks/programming_task_list.md` §T05
> - `docs/design/software_detailed_design.md`
> - `docs/analysis/initial_project_analysis.md`
> - `docs/logs/task_log.md`（T04 执行记录）

---

## 1. 任务实现分析

### 1.1 当前状态

T04 已完成并板级验证通过（2026-08-24）：PA5 示波器观察到约 5 kHz 方波，确认 10 kHz 注入采样链路正确运行；断点实测 ch1 随 PA1 电位器约在 50～4000 范围变化，ch2≈4000（PB1 接 3.3V），ch3≈50（PB0 接 GND）。

`PROC_HandleSamples()` 已在注入组回调中被调用，当前为空壳（`(void)` 丢弃三路参数）；`PROC_GetDacAmplitude()` 返回固定 `0U`。本任务只需实现 `app_process.c` 中的具体计算逻辑，不涉及其他模块改动。

### 1.2 数据路径

```
HAL_ADCEx_InjectedConvCpltCallback [app_sampling.c]
  → PROC_HandleSamples(ch1, ch2, ch3) [app_process.c]
       ├─ ch1 (PA1 电位器, 0~4095) → 计算 PWM CH1 占空比 → BSP_WritePwmDutyCh1()
       └─ ch1 (PA1 电位器, 0~4095) → 计算 DAC 方波幅值 → 暂存于 dac_amplitude
```

### 1.3 依赖关系

- T04 已完成，10 kHz 链路已验证，`PROC_HandleSamples()` 调用链路已就绪。
- `BSP_WritePwmDutyCh1()` 已在 T03 中实现（内部为 `__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, duty)`）。
- TIM1 配置已核实（`Core/Src/tim.c`）：Prescaler=0、Period=8000、中心对齐模式 1 → CCR1 有效范围 0～8000，与 §2.1 建议映射一致。
- 无其他模块依赖。

---

## 2. 前置条件（换算公式已确认）

以下两项在 T05 任务描述中列为编码前置条件，已于 2026-08-24 经开发者审阅确认：

### 2.1 TBD-08：PWM 占空比换算公式（已确认）

ADC 值（12-bit, 0~4095）映射到 TIM1 CCR1（0 ~ ARR=8000）。

已确认公式：
```
duty = (uint16_t)(((uint32_t)ch1 * 8000) / 4095)
```
即将 ADC 满量程 4095 线性映射到 CCR1 满量程 8000。CCR1 上限为 ARR=8000（中心对齐模式下 CCR=ARR 对应约 100% 占空比，无需额外限幅）。

### 2.2 TBD-09：DAC 换算公式（已确认）

ADC 值（12-bit, 0~4095）映射到 DAC 输出值（12-bit, 0~4095）。

已确认公式：
```
dac_amplitude = ch1   // 同为 12-bit 量程，直接赋值
```
ADC 和 DAC 均为 12-bit 分辨率，满量程一致，映射为 1:1 直通。

> 以上两项公式为 T05 编码的最终依据，实施时不再改动。

---

## 3. 需要修改的文件和函数

### 3.1 app_process.c（MD-03，唯一修改文件）

| 修改位置 | 修改内容 |
|---|---|
| 文件头 `@brief` 注释 | 更新为「MD-03 实时处理模块（T05 实现）」 |
| 新增引用 | `#include "app_bsp.h"` |
| 新增私有变量 | `static volatile uint16_t dac_amplitude = 0;` |
| `PROC_HandleSamples()` | 替换空壳为实际计算逻辑（PWM 占空比计算 + DAC 幅值暂存） |
| `PROC_GetDacAmplitude()` | 返回当前 DAC 幅值（替换固定返回 `0U`） |

### 3.2 不修改的文件

- `App/Inc/app_process.h`：`PROC_HandleSamples`/`PROC_GetDacAmplitude` 接口已于 T02 定型，无需改动。
- 其他 App 模块（`app_bsp.*`、`app_sampling.*`、`app_control.*`、`app_monitor.*`）：T04/T06/T07 任务范围，本任务不触碰。
- `CMakeLists.txt`：`App/Src/app_process.c` 已加入构建，无需改动。
- CubeMX 生成代码、HAL/CMSIS 库、`.ioc` 文件。

---

## 4. 接口及调用关系

| 调用关系 | 说明 |
|---|---|
| MD-02 → MD-03：`HAL_ADCEx_InjectedConvCpltCallback`（app_sampling.c）→ `PROC_HandleSamples(ch1, ch2, ch3)` | T04 已接通，本任务不改动调用方 |
| MD-03 → MD-01：`PROC_HandleSamples` → `BSP_WritePwmDutyCh1(duty)` | 本任务新增调用；MD-01 内部写 TIM1 CCR1，10 kHz 即时更新 |
| MD-03 内部：`PROC_HandleSamples` → 静态变量 `dac_amplitude` | ADC1_2 ISR（10 kHz，优先级 0）中写入，`volatile` 修饰 |
| MD-03 → MD-04（后续）：`PROC_GetDacAmplitude()` | 只读接口，T07 由 500 Hz 调度（TIM6 ISR）读取用于 DAC 方波翻转和串口数据源，本任务仅实现返回值 |
| 跨 ISR 共享 | `dac_amplitude` 遵循详细设计 §2.6：`volatile` 修饰，`uint16_t` 原子读写，单写单读，无需临界区 |

公开接口和模块职责均不变，仅补齐 MD-03 内部实现。

---

## 5. 具体代码修改方案

`App/Src/app_process.c` 修改后完整内容：

```c
/**
  ******************************************************************************
  * @file    app_process.c
  * @brief   MD-03 实时处理模块（T05 实现）
  ******************************************************************************
  */
#include "app_process.h"
#include "app_bsp.h"

/* DAC 方波幅值（持存，10 kHz 更新，volatile 供跨 ISR 共享） */
static volatile uint16_t dac_amplitude = 0;

void PROC_HandleSamples(uint16_t ch1, uint16_t ch2, uint16_t ch3)
{
    uint16_t duty;

    /* ch2、ch3 预留，当前不参与动态计算 */
    (void)ch2;
    (void)ch3;

    /* 计算 PWM CH1 占空比并写入 CCR1（TBD-08 已确认：ch1 × 8000 / 4095） */
    duty = (uint16_t)(((uint32_t)ch1 * 8000) / 4095);
    BSP_WritePwmDutyCh1(duty);

    /* 计算并暂存 DAC 方波幅值（TBD-09 已确认：1:1 直通） */
    dac_amplitude = ch1;
}

uint16_t PROC_GetDacAmplitude(void)
{
    return dac_amplitude;
}
```

**注意**：
- 以上换算公式已经开发者确认（2026-08-24，见 §2）。后续如需调整，直接修改对应两行即可。
- 局部变量 `duty` 声明于函数开头，与 T04 校订版风格保持一致。
- 乘法先强转为 `uint32_t` 再除以 4095，避免 16 位溢出（`ch1=4095` 时乘积约 3276 万）。

---

## 6. 编译、调试和板级验证方案

### 6.1 编译验证

与 T04 相同方式执行 Debug 配置编译，确认 `app_process.c` 无编译错误和警告（`(void)ch2; (void)ch3;` 消除未使用参数警告），成功链接生成 `.elf`。

### 6.2 调试验证（调试器）

> 设置断点会暂停 CPU，不能同时用于时序观测。调试与板级示波器验证分开执行。

1. 断点设于 `App/Src/app_sampling.c` 中 `PROC_HandleSamples(ch1, ch2, ch3);` 所在行，观察输入：ch1 随电位器 0～4095 变化，ch2≈4095，ch3≈0。
2. 单步进入 `PROC_HandleSamples()`，观察局部变量 `duty` 计算值符合 TBD-08 换算关系（如 ch1=2048 → duty=4000）。
3. 调试器寄存器视图观察 `htim1.Instance->CCR1`，确认等于刚写入的 `duty`。
4. 观察静态变量 `dac_amplitude`（或调用 `PROC_GetDacAmplitude()` 的返回值），确认等于 ch1，符合 TBD-09 换算关系（当前为 1:1 直通）。

### 6.3 板级验证

**前提**：T04 的辅助启动代码仍然有效（`APP_Control_Init` 调用 `BSP_StartRealtimeChain`，`CTRL_NotifyInjectedComplete` 调用 `BSP_ToggleLed`）；接线沿用 T04：JP8=[2-3]、PA1 接电位器。

1. 烧录固件，上电。
2. 示波器观察 PA8（TIM1_CH1）→ 应显示 10 kHz PWM 波形（实测约 10.6 kHz）。
3. 旋转 PA1 电位器 → PWM 占空比应随之线性变化（从 0% 到接近 100%；按 T04 实测 ch1 范围 50～4000，对应约 1%～98%）。
4. 调试器观察 `dac_amplitude` 或 `PROC_GetDacAmplitude()` 返回值 → 确认其随 PA1 采样值 1:1 变化，并符合 TBD-09 确定的换算关系（当前为 1:1 直通）。
5. 示波器观察 PA5（LED）→ 仍为约 5 kHz 方波（T04 辅助代码），确认 10 kHz 链路持续运行。

---

## 7. 修改汇总

| 文件 | 修改内容 |
|---|---|
| `App/Src/app_process.c` | 更新文件头 `@brief`；新增 `#include "app_bsp.h"`；新增 `static volatile uint16_t dac_amplitude`；实现 `PROC_HandleSamples()` 计算逻辑（PWM 占空比计算 + DAC 幅值暂存）；实现 `PROC_GetDacAmplitude()` 返回幅值 |

**不修改**：头文件、CMakeLists.txt、CubeMX 生成代码、其他 App 模块文件。

## 8. 实施顺序建议

1. TBD-08/TBD-09 换算公式已由开发者确认（2026-08-24，见 §2），无需再确认；
2. 按 §5 修改 `App/Src/app_process.c`，经差异视图提交开发者审阅，确认后写入工程；
3. 执行 §6.1 编译验证；
4. 执行 §6.2 调试验证和 §6.3 板级验证（开发者执行烧录与观测）；
5. 记录验证结果至 `docs/logs/task_log.md`，待批准后进入 T06。

## 9. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 换算公式已确认 | TBD-08/TBD-09 已由开发者确认（2026-08-24）；后续如需调整仅改 §5 代码中对应两行 |
| 占空比上限 | CCR1=8000=ARR 时中心对齐模式 1 下占空比约 100%（峰值处差 1 个计数）；0～4095 映射结果恒 ≤ 8000，无需限幅 |
| 跨 ISR 共享 | `dac_amplitude` 由 ADC1_2 ISR（10 kHz）写、TIM6 ISR（500 Hz，T07 起）读，`volatile` + `uint16_t` 原子读写满足详细设计 §2.6 要求，不引入临界区 |
| PWM 更新即时性 | `__HAL_TIM_SET_COMPARE` 直接写 CCR1，10 kHz 更新频率下无明显延迟 |
| 板级验证前置条件 | 需保持 T04 接线（JP8=[2-3]、PA1 电位器）和 T04 辅助启动代码有效 |
| 辅助代码遗留 | `app_control.c` 中 T04 辅助代码由 T06/T07 替换，与本任务无关，仅提示 |
