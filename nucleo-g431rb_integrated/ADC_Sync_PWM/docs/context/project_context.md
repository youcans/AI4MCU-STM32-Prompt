# 项目上下文说明

> 当前版本：v1.0（初版）
> 更新说明：基于已确认的设计文档（software_function_spec.md、pin_config_spec.md、module_design.md、module_relationship_analysis.md、runtime_state_analysis.md）提取生成，编制日期 2026-08-06。

## 1. 项目名称与目标
- 项目名称：基于 ADC 同步采样的 PWM 占空比控制
- 项目目标：在 NUCLEO-G431RB 开发板上，以 PWM 周期事件触发三通道模拟电压同步采样，在每次采样序列完成后执行一次实时数据处理，根据动态模拟电压的采样结果更新 PWM 占空比和 DAC 方波幅值，同时实现芯片温度监测、系统状态切换、LED 指示和串口数据发送功能。
- 主要功能：

| 编号 | 功能名称 | 概要 |
|---|---|---|
| FR-01 | PWM 输出与周期事件 | 输出三组 10 kHz 带死区互补 PWM，第一组占空比动态更新，第二/三组使用预设值；产生 ADC 采样触发事件 |
| FR-02 | ADC 三通道同步采样 | PWM 周期事件触发三通道模拟电压采样，一路动态电压、两路参考电平 |
| FR-03 | 实时数据处理 | 每次采样序列完成后执行，读取采样值、计算并更新 PWM 占空比和 DAC 方波幅值 |
| FR-04 | DAC 参考方波输出 | 输出 250 Hz 参考方波，幅值随动态模拟电压采样结果更新 |
| FR-05 | 芯片温度监测 | 使用内部温度传感器以 500 Hz 采集芯片温度，全状态持续运行 |
| FR-06 | 温度异常处理 | 温度超限进入异常状态，温度恢复返回停止状态 |
| FR-07 | 系统状态切换 | 停止/运行/异常三状态，按键切换停止↔运行，温度触发异常进入与恢复 |
| FR-08 | LED 状态指示 | 停止熄灭、运行翻转（~常亮）、异常 10 Hz 闪烁 |
| FR-09 | 串口数据发送 | JustFloat / 115200 bps / 500 Hz，运行发送 DAC 值，停止发送温度，异常不发送 |

## 2. 硬件平台与开发环境
- 开发板：NUCLEO-G431RB
- MCU：STM32G431RB
- 外部连接：
  - PA1：连接 10 kΩ 电位器滑动端（电位器高端接 3.3 V，低端接 GND），提供动态模拟电压
  - PB1：杜邦线接高电平参考电压
  - PB0：杜邦线接 GND
  - PA4：DAC 输出，仅接高阻抗测量设备
  - PA2 / PA3：通过板载 SB17（PA2）、SB23（PA3）连接 STLINK-V3E 虚拟串口
  - PC13：板载用户按键 B1（外部下拉，按下接通高电平）
  - PA5：板载用户 LED LD2（高电平点亮）
  - PA13 / PA14：板载 STLINK-V3E SWD 调试接口
- 工程生成工具：STM32CubeMX（来源：pin_config_spec.md §2.3 提及）
- 开发环境与工具链：VS Code、STM32CubeIDE for VS Code、CMake 工具链、GCC 编译器
- 烧录、调试和验证工具：STLINK-V3E（板载），SWD 接口（PA13/SWDIO、PA14/SWCLK）

## 3. 关键资源与硬件连接
- 主要外设：

| 外设 | 用途 | 关键参数 |
|---|---|---|
| TIM1 | 三组带死区互补 PWM 输出；触发 ADC1 注入组 | 10 kHz，中心对齐模式 |
| ADC1 | 注入组三通道模拟电压采样 + 规则组温度采样 | 12 位分辨率；注入组由 TIM1 触发（10 kHz），规则组由 TIM6_TRGO 触发（500 Hz） |
| DAC1 | 250 Hz 参考方波输出 | Channel 1（PA4），12 位右对齐，外部缓冲通道 |
| TIM6 | 500 Hz 时基：温度采样触发、DAC 翻转、LED 闪烁分频、串口发送节拍 | 基本定时器，TRGO 输出更新事件，500 Hz |
| LPUART1 + DMA1 | 串口数据发送 | JustFloat 格式，115200 bit/s，500 Hz，每帧 8 字节 |
| EXTI13 | 用户按键事件 | PC13，上升沿触发 |

- 关键引脚：

| 引脚 | 功能 | 备注 |
|---|---|---|
| PA8 / PB13 | TIM1_CH1 / CH1N | PWM 第一组（动态占空比），带死区互补 |
| PA9 / PB14 | TIM1_CH2 / CH2N | PWM 第二组（预设占空比），带死区互补 |
| PA10 / PB15 | TIM1_CH3 / CH3N | PWM 第三组（预设占空比），带死区互补 |
| PA1 | ADC1_IN2 | 注入组 Rank1，动态模拟电压 |
| PB1 | ADC1_IN12 | 注入组 Rank2，高电平参考 |
| PB0 | ADC1_IN15 | 注入组 Rank3，低电平参考 |
| PA4 | DAC1_OUT1 | 250 Hz 方波输出 |
| PA2 | LPUART1_TX | 串口发送 |
| PA3 | LPUART1_RX | 串口接收（本项目不使用） |
| PC13 | GPIO_EXTI13 | 用户按键 |
| PA5 | GPIO_Output | 用户 LED |
| PA13 / PA14 | SYS_SWDIO / SYS_SWCLK | SWD 调试接口（不得配置为应用 GPIO） |

- 外设之间的触发关系：
  - TIM1 内部触发事件 → ADC1 注入组（硬件自动触发，建立 PWM 与 ADC 采样的相位同步）
  - TIM6_TRGO → ADC1 规则组（硬件自动触发，驱动 500 Hz 温度采样）
  - ADC1 注入组和规则组共用 ADC1_2 全局中断，通过事件标志（JEOS / EOC）区分处理

- 上位机或外部仪器：
  - VOFA+：串口数据接收与显示（JustFloat 格式，115200 bit/s）

## 4. 软件结构与运行机制
- 软件模块：

| 模块编号 | 模块名称 | 主要职责 |
|---|---|---|
| MD-01 | 板级支持模块 | 外设底层初始化；提供外设基础操作接口；统一执行 ADC 注入组、TIM1 和 PWM 的底层启停 |
| MD-02 | 采样管理模块 | ADC 转换完成事件处理；采样结果读取、换算与分发 |
| MD-03 | 实时处理模块 | 根据模拟电压采样值计算 PWM 占空比和 DAC 方波幅值 |
| MD-04 | 系统控制模块 | 系统状态管理；温度异常判断；DAC 翻转；LED 指示；500 Hz 周期调度 |
| MD-05 | 串口监测模块 | 接收 MD-04 选定的数据，完成 JustFloat 组帧和 DMA 发送 |

- 主要事件与数据流：

  - **10 kHz 实时路径**（仅运行状态）：TIM1 周期事件 → ADC1 注入组三通道按 Rank1→2→3 顺序转换 → ADC1_2 中断（注入组完成分支）→ MD-02 读取采样值 → MD-03 计算 PwmDutyCh1 和 DacOutputValue → MD-01 写 TIM1 CCR1；MD-02 同时通知 MD-04 翻转 LED。

  - **500 Hz 周期处理**（全状态）：
    - 软件调度子流程（TIM6 更新中断 → MD-04）：DAC 翻转（运行状态 250 Hz 方波）、LED 分频计数（异常状态 10 Hz 闪烁）、串口发送调度（运行发 DAC 值，停止发温度值，异常不发）。
    - 温度采样与判断子流程（TIM6_TRGO → ADC1 规则组 → ADC1_2 中断规则组完成分支 → MD-02 读取并换算温度 → MD-04 温度异常判断）。

  - **异步事件**：EXTI13 按键事件 → MD-04 状态切换（停止↔运行）；温度异常进入/退出 → MD-04 内部产出 → 状态机转换；DMA 发送完成 → MD-05 清除忙状态。

  - **调用关系（全部单向）**：关键数据持有：DAC 输出值由 MD-03 暂存（10 kHz 更新），芯片温度值由 MD-02 持有，`SystemState` 由 MD-04 独占修改且仅在内部使用。
MD-02 → MD-03 → MD-01
MD-02 → MD-04 → MD-01
MD-04 → MD-05 → MD-01

- 运行状态：

| 状态 | 进入条件 | PWM/DAC/ADC注入组 | 温度采样 | LED | 串口发送 |
|---|---|---|---|---|---|
| 停止 | 上电复位；运行状态下按键；异常状态下温度恢复 | 关闭 | 500 Hz 运行 | 熄灭 | 500 Hz，发温度值 |
| 运行 | 停止状态下按键 | 开启（10 kHz / 250 Hz） | 500 Hz 运行 | 10 kHz 翻转（近似常亮） | 500 Hz，发 DAC 值 |
| 异常 | 停止或运行状态下温度达异常阈值 | 关闭 | 500 Hz 运行（仅判断恢复） | 10 Hz 闪烁 | 不发送 |

- 状态切换规则：
  - 停止 ↔ 运行：用户按键（EXTI13）
  - 停止/运行 → 异常：芯片温度 ≥ TemperatureFaultThreshold
  - 异常 → 停止：芯片温度 ≤ TemperatureRecoveryThreshold
  - 异常状态下按键不响应；返回停止后需再次按键才能进入运行

- 主要运行周期：

| 周期 | 驱动源 | 有效状态 | 处理内容 |
|---|---|---|---|
| 10 kHz | TIM1 周期事件 → ADC1 注入组完成 | 运行 | 三通道模拟电压采样、PWM 占空比计算与更新、DAC 方波幅值更新、LED 翻转 |
| 500 Hz | TIM6 更新事件 | 全状态 | DAC 翻转（运行）、LED 闪烁分频（异常）、串口发送调度（运行/停止）、温度采样触发与判断（全状态） |

## 5. 设计文档索引
- 软件功能说明书：[docs/design/software_function_spec.md](../design/software_function_spec.md)
- MCU 资源与引脚配置：[docs/design/pin_config_spec.md](../design/pin_config_spec.md)
- 软件模块设计：[docs/design/module_design.md](../design/module_design.md)
- 模块关系与数据流：[docs/design/module_relationship_analysis.md](../design/module_relationship_analysis.md)
- 运行流程与状态切换：[docs/design/runtime_state_analysis.md](../design/runtime_state_analysis.md)
- CubeMX 工程配置清单：[docs/design/cubemx_config_checklist.md](../design/cubemx_config_checklist.md)
- 初始工程结构分析：[docs/analysis/initial_project_analysis.md](../analysis/initial_project_analysis.md)
- 软件详细设计：[docs/design/software_detailed_design.md](../design/software_detailed_design.md)
- 编程任务清单：[docs/tasks/programming_task_list.md](../tasks/programming_task_list.md)

## 6. 验证依据
- 编译验证：待确认（设计文档中未描述编译验证方法）
- 板级运行验证：待确认（设计文档中未描述具体的板级运行验证步骤）
- 波形或数据验证：
  - 运行状态：DAC 输出 250 Hz 方波（来源：software_function_spec.md SF-06），可通过高阻抗测量设备观察 PA4 引脚波形
  - LED 指示：停止熄灭、运行近似常亮、异常 10 Hz 闪烁（来源：software_function_spec.md SF-10），可目视验证
  - 串口数据：VOFA+ 接收 JustFloat 格式数据，115200 bit/s，500 Hz（来源：software_function_spec.md SF-11），可通过 VOFA+ 界面验证
  - （有关 PWM 波形验证的具体方法和测量点，设计文档中未展开描述，标记为待确认）

## 7. 已确认的技术约束
- 硬件约束：
  - PA13（SWDIO）和 PA14（SWCLK）为 SWD 调试引脚，不得配置为应用 GPIO
  - 16 个外部引脚功能分配不重叠，无引脚复用冲突
  - ADC1 注入组（10 kHz）和规则组（500 Hz）共用 ADC1_2 全局中断，通过事件标志区分处理，不可分别设置 NVIC 优先级
  - ADC1_IN16（内部温度传感器）为芯片内部通道，不占用外部引脚
  - PB0 / PB1 无板载有源电路占用，可用作 ADC 输入；外部杜邦线连接至参考电平时需确保与板上其他功能不短路

- 运行时序约束：
  - SF-05 实时数据处理在 JEOS 中断上下文中执行，执行时间须远小于 100 µs（10 kHz 周期）
  - PWM 停止/异常状态下 TIM1 停止，ADC1 注入组随之停止；TIM6 在所有状态下持续运行
  - 实时链路启停时序：启动时先使能 ADC 注入组再启动 TIM1 和 PWM 输出；停止时先停止 TIM1 触发源再停止注入组采样并关闭 PWM 输出
  - 上电后首个 TIM6 更新事件同时启动第一次温度采样（TRGO）和第一次 TIM6 更新中断调度；首次规则组转换完成前，停止状态下不应触发温度数据串口发送

- 数据与通信约束：
  - 串口 JustFloat 帧为 8 字节/帧，500 Hz 发送速率下波特率 115200 bit/s 满足带宽要求
  - 仅 LPUART1_TX 使用 DMA，无 DMA 冲突
  - MD-05 需维护 DMA 发送忙状态，避免在前一次发送未完成时重复启动
  - DAC 方波幅值在 ADC1_2 中断（10 kHz）中由 MD-03 更新、在 TIM6 更新中断（500 Hz）中由 MD-04 读取，两个中断上下文之间共享该数据
  - 芯片温度值在 ADC1_2 中断（规则组完成分支）中由 MD-02 更新、在 TIM6 更新中断（500 Hz）中由 MD-04 读取，两个中断上下文之间共享该数据

- 模块设计约束：
  - MD-01 只提供外设操作原语，不包含任何业务逻辑、状态判断；不访问 `SystemState` 等业务状态变量
  - MD-04 独占 `SystemState` 修改权，`SystemState` 仅在 MD-04 内部使用，其他模块不读取
  - MD-05 不读取 `SystemState`，不从 MD-02 或 MD-03 直接读取业务数据——所有数据由 MD-04 选定后传入
  - MD-03 为纯计算模块，不接触任何外设寄存器（所有寄存器写入通过 MD-01 执行）
  - 注入组采样值为瞬时传递（MD-02 → MD-03 直接传参），不暂存于 MD-02；DAC 输出值和芯片温度值为持存数据
