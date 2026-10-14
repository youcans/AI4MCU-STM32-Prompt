# 编程任务清单

> 编制日期：2026-08-08
> 修订日期：2026-08-08（按新任务编排原则重组 T04 以后任务）
> 依据文件：
> - `docs/design/software_detailed_design.md`（软件详细设计）
> - `docs/analysis/initial_project_analysis.md`（初始工程结构分析）
> 说明：本清单优先验证核心时序和数据链路，再在已验证的基础上实现业务逻辑。T01~T03 已完成并保持不变。

---

## T01 — 创建应用程序目录、文件骨架和构建配置

**任务目标**：建立 `App/Inc/` 和 `App/Src/` 目录，创建全部 5 个模块的 `.h` 和 `.c` 文件骨架，配置 CMake。

**涉及文件**：
- 新建：`App/Inc/app_bsp.h`、`App/Inc/app_process.h`、`App/Inc/app_sampling.h`、`App/Inc/app_control.h`、`App/Inc/app_monitor.h`
- 新建：`App/Src/app_bsp.c`、`App/Src/app_process.c`、`App/Src/app_sampling.c`、`App/Src/app_control.c`、`App/Src/app_monitor.c`
- 修改：`CMakeLists.txt`

**依赖关系**：无。

**实施要点**：
- 创建 `App/Inc/` 和 `App/Src/` 目录。
- 5 个头文件：包含 include guard、必要的 `#include`（`main.h` 及对应的 CubeMX 外设头文件）。头文件中暂不声明公开函数接口（留待 T02）。
- 5 个源文件：包含对应头文件，`.c` 文件为空（无函数实现）。此时除 `main.c` 外所有文件均不引用 App 模块的任何函数，编译应通过。
- `CMakeLists.txt`：在 USER 区域添加 `App/Src/*.c` 源文件和 `App/Inc` include 路径（参考 `software_detailed_design.md` §1.5.4）。
- 本任务**不修改** `main.c`。

**验证方式**：
- 工程编译通过（App 目录和 CMake 配置就绪，模块代码尚未接入）。

---

## T02 — 建立模块接口与应用程序框架

**任务目标**：在各模块头文件中声明完整的公开函数接口，在源文件中建立空实现函数骨架，在 `main.c` 中接入应用初始化入口。

**涉及文件**：
- 修改：`App/Inc/app_bsp.h`、`App/Inc/app_process.h`、`App/Inc/app_sampling.h`、`App/Inc/app_control.h`、`App/Inc/app_monitor.h`
- 修改：`App/Src/app_bsp.c`、`App/Src/app_process.c`、`App/Src/app_sampling.c`、`App/Src/app_control.c`、`App/Src/app_monitor.c`
- 修改：`Core/Src/main.c`

**依赖关系**：依赖 T01。

**实施要点**：
- 5 个头文件：按 `software_detailed_design.md` §2.1 声明各模块的公开函数接口。头文件只声明本模块公开函数，不声明 HAL 回调函数，不包含其他 App 模块头文件（跨模块调用由 `.c` 文件自行包含对方头文件）。
- 5 个源文件：包含对应头文件和所需的 CubeMX/其他 App 模块头文件。公开函数以空函数体和安全默认返回值实现。
- `main.c`：在 `USER CODE BEGIN Includes` 添加 `#include "app_control.h"`；在 `USER CODE BEGIN 2`（`MX_TIM6_Init()` 之后）添加 `APP_Control_Init();`。
- 本任务**不定义** `SystemState_t` 枚举和应用配置常量（属于 MD-04 的具体实现内容，在 T06 完成）。

**验证方式**：
- 工程编译通过。此时全系统可编译链接，应用功能均为空壳。

---

## T03 — 实现 MD-01 板级支持模块（app_bsp.c）

**任务目标**：完整实现 MD-01 模块的全部外设操作接口。

**涉及文件**：
- 修改：`App/Src/app_bsp.c`

**依赖关系**：依赖 T02。

**实施要点**：
- 按 `software_detailed_design.md` §2.1.1 实现全部 11 个公开接口。
- 实时链路启动/停止：封装 ADC 注入组 + TIM1 + PWM 启停时序（§3.5）。启动：先使能 ADC 注入组中断，再启动 TIM1；停止：先清除 TIM1 CEN 立即停止计数和 TRGO2，再依次停止 PWM/PWMN 和 ADC 注入组。
- TIM1 CCR 写入、DAC 输出、GPIO 翻转/电平设置、DMA 发送启动（返回执行结果）、TIM6 启动。
- 模块不包含业务逻辑、不访问 `SystemState`、不持有业务数据。

**验证方式**：
- 编译通过。

---

## T04 — 打通并验证 10 kHz ADC 注入采样链路

**任务目标**：实现 MD-02 注入组 HAL 回调，通过辅助代码临时启动实时链路，验证 10 kHz 核心时序和三通道数据来源。

**涉及文件**：
- 修改：`App/Src/app_sampling.c`（实现 `HAL_ADCEx_InjectedConvCpltCallback`）
- 修改：`App/Src/app_control.c`（在 `APP_Control_Init()` 中临时调用 `BSP_StartRealtimeChain()` 启动实时链路，提供临时 LED 翻转入口）

**依赖关系**：依赖 T03（MD-01 已实现）。

**实施要点**：
- 在 `app_sampling.c` 中实现 `HAL_ADCEx_InjectedConvCpltCallback`：使用 `HAL_ADCEx_InjectedGetValue` 读取三路注入组结果，调用 MD-03 的处理函数（当前为空壳，后续 T05 实现），调用 `CTRL_NotifyInjectedComplete()`。
- 辅助代码：在 `APP_Control_Init()` 中临时调用 `BSP_StartRealtimeChain()` 直接进入运行态（不经过按键状态机，不启动 TIM6）。在 `CTRL_NotifyInjectedComplete()` 中临时调用 `BSP_ToggleLed()` 翻转 PA5，用于示波器验证 10 kHz 事件频率。TIM6 属于 500 Hz 链路，留待 T06。
- 验证完成后，`APP_Control_Init()` 中的辅助启动代码由 T06 的正规初始化流程替换；`CTRL_NotifyInjectedComplete()` 中的临时翻转由 T07 中按状态判断的正式逻辑替换。

**验证方式**：
- 编译通过。
- **板级验证（10 kHz 时序）**：上电后 LED 应当以约 5 kHz 的频率闪烁（10 kHz 注入组回调用临时代码翻转 LED），肉眼观察为近似常亮，确认 10 kHz 事件链路正确运行。
- **三通道数据验证**：在注入组回调中分别读取 Rank1(PA1)/Rank2(PB1)/Rank3(PB0)，通过调试器或辅助方式确认三路数据来源正确。PB1 接高电平、PB0 接 GND 时，应分别读到接近 4095 和 0 的值。

---

## T05 — 实现并验证 MD-03 实时处理

**任务目标**：在已验证的 10 kHz 注入链路上实现 PWM 占空比和 DAC 幅值计算，并通过示波器验证实时控制效果。

**涉及文件**：
- 修改：`App/Src/app_process.c`

**依赖关系**：依赖 T04（10 kHz 链路已验证）。

**前置条件**：以下待确认事项需在编码前由开发者确认：
- TBD-08：PWM 占空比换算公式、限幅范围和数值处理方法
- TBD-09：DAC 换算公式、幅值范围和数值处理方法

**实施要点**：
- 按 `software_detailed_design.md` §2.1.3 实现公开接口。接收三路注入组采样值，当前仅 PA1 参与计算，PB1/PB0 预留。
- 按已确认的换算公式计算 PWM CH1 占空比 → 调用 MD-01 写 CCR1。
- 按已确认的换算公式计算 DAC 方波幅值 → 暂存（`volatile` 修饰），提供只读接口。

**验证方式**：
- 编译通过。
- **板级验证**：上电后用示波器观察 PA8（TIM1_CH1）→ 10 kHz PWM，旋转电位器时占空比应随之变化。观察 PA1 电位器电压变化在示波器上的 PWM 占空比响应。

---

## T06 — 打通并验证 500 Hz 周期、规则采样和按键事件

**任务目标**：在 MD-04 中实现 TIM6 周期回调和 EXTI 按键回调，在 MD-02 中实现规则组温度采样和温度数据接口，验证 500 Hz 软件事件、温度采样链路和按键驱动的启停逻辑，将系统控制依赖的主要事件源和数据源全部确认。

**涉及文件**：
- 修改：`App/Src/app_control.c`（实现 `HAL_TIM_PeriodElapsedCallback`、`HAL_GPIO_EXTI_Callback`）
- 修改：`App/Src/app_sampling.c`（实现 `HAL_ADC_ConvCpltCallback`、`SAMP_GetTemperature`、`SAMP_IsTemperatureValid`）
- 修改：`App/Inc/app_bsp.h`、`App/Src/app_bsp.c`（新增 `BSP_StartRegularAdc()` 接口）
- 修改：`docs/design/software_detailed_design.md`（同步更新 MD-01 接口定义）

**依赖关系**：依赖 T05（10 kHz 链 + MD-03 已就绪）。T04 中的辅助启动代码在此任务中被替换。

**前置条件**：CH2/CH3 预设占空比的具体数值（如尚未确认，列为 T06 编码前置条件）。

**实施要点**：
- 在 `app_control.c` 中定义 `SystemState_t` 枚举（STOP/RUN）和配置常量（CH2/CH3 预设值、温度阈值占位）。
- `APP_Control_Init()`：设定 STOP 状态 → 写 CH2/CH3 预设值 → LED 熄灭/DAC 关闭 → 调用 MD-01 启动 ADC 规则组中断转换 → 启动 TIM6。替换 T04 的辅助启动代码。
- 现有 MD-01 接口中缺少 ADC 规则组启动接口。需在编码前新增并确认，例如 `BSP_StartRegularAdc()`（调用 `HAL_ADC_Start_IT(&hadc1)` 启动规则组中断转换），同时更新详细设计和 MD-01 接口定义。
- `HAL_TIM_PeriodElapsedCallback`：检查 `htim->Instance == TIM6`，使用辅助计数变量验证 500 Hz 事件频率（例如每 500 次累加一次全局标志，通过调试器确认）。不在 PA5 上叠加 500 Hz 翻转，避免与 10 kHz 注入组回调同时操作同一 GPIO。
- `HAL_GPIO_EXTI_Callback`：检查 `GPIO_Pin == B1_Pin`，加入软件消抖处理后执行停止↔运行切换。运行状态调用 `BSP_StartRealtimeChain()` + `BSP_StartDac()`，停止状态调用 `BSP_StopRealtimeChain()` + `BSP_StopDac()` + LED 熄灭。
- 在 `app_sampling.c` 中实现 `HAL_ADC_ConvCpltCallback`：读取温度原始值 → 换算为 °C → 持存并置位有效标志 → 调用 `CTRL_ProcessTemperature(temperature)` 将温度值传递给 MD-04。实现 `SAMP_GetTemperature()` 返回当前温度值，`SAMP_IsTemperatureValid()` 返回有效性标志。
- T06 时 `CTRL_ProcessTemperature()` 仍为空壳（T07 实现异常判断逻辑），但数据传递链路已建立。

**验证方式**：
- 编译通过。
- **板级验证（500 Hz 时基）**：通过调试器观察 500 Hz 回调中的辅助计数变量持续递增，确认 TIM6 500 Hz 调度正确。
- **板级验证（按键启停）**：按 B1 → LED 由熄灭变为常亮（10 kHz 注入组回调翻转），同时 PWM 输出出现。再按 B1 → LED 熄灭，PWM 关闭。确认按键驱动的启停逻辑正确，无抖动误触发。
- **温度数据验证**：通过调试器确认温度值约 25~35°C、温度有效标志在首次转换后置位，确认规则组采样链路正常。

---

## T07 — 实现 MD-04 系统状态机与周期控制

**任务目标**：实现完整的 STOP/RUN/FAULT 三状态转换、实时链路启停决策、DAC 250 Hz 方波调度、LED 状态控制和温度异常/恢复判断。

**涉及文件**：
- 修改：`App/Src/app_control.c`

**依赖关系**：依赖 T06（事件源和数据源已就绪）。

**前置条件**：以下待确认事项需在编码前由开发者确认：
- TBD-07：温度异常阈值和恢复阈值的具体数值

**实施要点**：
- 按 `software_detailed_design.md` §3.1 实现完整状态机（STOP/RUN/FAULT）。
- `HAL_TIM_PeriodElapsedCallback`：移除 T06 中的 500 Hz 辅助计数验证代码，按 §3.2.2 路径 A 实现正式 500 Hz 周期调度：DAC 翻转（读取 MD-03 幅值，交替输出 0/幅值，形成 250 Hz 方波）→ LED 分频（异常状态 10 Hz 闪烁）→ 串口数据源选择（运行=MD-03 幅值，停止=MD-02 温度值（有效时），异常=不触发）。
- `HAL_GPIO_EXTI_Callback`：按 §3.1.2 实现按键状态切换。异常状态不响应按键。
- `CTRL_NotifyInjectedComplete()`：运行状态下调用 MD-01 翻转 LED（由 MD-02 在注入组回调中调用）。
- `CTRL_ProcessTemperature()`：按 §3.1.2 执行温度异常判断，产生 TemperatureFaultEnter/Exit。
- `APP_Control_Init()`：按 §3.4 完整实现初始化。

**验证方式**：
- 编译通过。
- **板级验证（状态机 + DAC）**：按 B1 进入运行，用示波器观察 PA4 → 250 Hz 方波，旋转电位器时幅值变化。按 B1 返回停止 → DAC 关闭。
- **板级验证（LED）**：停止=熄灭，运行=常亮，异常=10 Hz 闪烁。

---

## T08 — 实现 MD-05 串口监测

**任务目标**：在已验证的 500 Hz 调度和状态机基础上，实现 JustFloat 组帧、DMA 发送、发送忙状态管理和完成回调。

**涉及文件**：
- 修改：`App/Src/app_monitor.c`

**依赖关系**：依赖 T07（MD-04 状态机和 500 Hz 调度已实现）。

**实施要点**：
- 按 `software_detailed_design.md` §2.1.5 实现公开接口。
- JustFloat 组帧：float 数据 4 字节 + JustFloat 帧尾 4 字节，共 8 字节帧缓冲区。
- DMA 忙标志：发送前检查，启动成功后置位。`HAL_UART_TxCpltCallback` 中清除（检查 `huart->Instance == LPUART1`）。
- Normal DMA 模式：DMA 完成后 HAL 使能 UART TCIE，最终由 LPUART1 ISR 触发 TxCpltCallback。
- MD-04 的 500 Hz 调度中调用 `MON_SendData()`，忙时跳过当前周期。

**验证方式**：
- 编译通过。
- **板级验证**：VOFA+ 接收串口数据（115200 bps），停止状态收到温度值，运行状态收到 DAC 幅值，旋转电位器时 DAC 幅值数据变化。

---

## T09 — 板级验证：温度异常保护和状态转换

**任务目标**：完整验证温度异常进入和恢复的硬件现象和状态转换。

**验证环境**：NUCLEO-G431RB 开发板，可用外部加热方式对 MCU 局部加热。

**验证步骤**：
1. 按 B1 进入运行状态。
2. 加热 MCU 使芯片温度超过异常阈值。
3. 确认异常进入：LED 由常亮变为 10 Hz 闪烁，示波器确认 PWM 和 DAC 输出关闭。
4. 停止加热，温度降到恢复阈值以下。
5. 确认异常恢复：LED 由闪烁变为熄灭（返回停止）。
6. 按 B1 → LED 变为常亮，确认异常恢复后可再次进入运行。

**验证方式**：
- 开发者通过 LED、示波器和 VOFA+ 确认。若难以创造温度变化条件，此项验证可延后。

---

## T10 — 系统综合验证

**任务目标**：全功能运行验证，覆盖 PWM、DAC、LED、串口及整体运行。

**验证环境**：NUCLEO-G431RB，JP8=[2-3]，PA1 接电位器，示波器 + VOFA+。

**验证步骤**：
1. 上电 → LED 熄灭，VOFA+ 收到温度数据。
2. 按 B1 → LED 常亮，VOFA+ 切换为 DAC 幅值，PA4 输出 250 Hz 方波，PA8/PB13 输出带死区互补 PWM。
3. 旋转电位器 → CH1 占空比、DAC 幅值、串口数据同步变化。CH2/CH3 保持固定预设值。
4. 按 B1 → LED 熄灭，PWM/DAC 全部关闭。
5. 重复启停，确认可靠性。

---

> **任务依赖链**：T01 → T02 → T03 → T04 → T05 → T06 → T07 → T08 → T09/T10（验证任务可并行或按序执行）。
>
> **前置条件**：T05 需 TBD-08/TBD-09（PWM/DAC 换算公式）已确认；T06 需 CH2/CH3 预设占空比数值已确认（如尚未确认）；T07 需 TBD-07（温度阈值）已确认。
>
> **辅助验证代码**：T04 和 T06 中使用临时代码产生可观测现象，验证完成后由后续任务的正规实现自然替换。
