# 设计文档清单日志

> 记录日期：2026-08-24
> 记录范围：设计阶段全部产出文档的存在性检查、通读与统一清单整理
> 说明：本日志为设计文档统一清单，涵盖需求设计、功能分解、资源规划、整合说明、模块设计、审查、工程配置、任务规划共 15 份文档。

## 0. 范围说明

| 纳入情况 | 文件 | 原因 |
|---|---|---|
| ✅ 纳入（15 份） | docs/analysis/design_inputs.md 等 10 份 + docs/design/cubemx_config_checklist.md 等 4 份 + docs/context/project_context.md | 设计阶段产出文档 |
| ❌ 暂不纳入 | docs/tasks/programming_plan_T04.md～T10.md（7 份分任务计划） | 按要求暂不纳入 |
| ❌ 未纳入 | docs/analysis/material_list.md、docs/analysis/requirements_analysis.md | 资料清单与需求分析阶段产出，非设计文档 |
| ❌ 未纳入 | docs/context/context_template.md | 空白模板 |

## 1. 文件清单（15 份）

| 序号 | 文件 | 目录 | 类别 | 编制日期 | 大小 |
|---|---|---|---|---|---|
| 1 | design_inputs.md | docs/analysis/ | 需求设计类 | 2026-08-04 | 7.1 KB |
| 2 | io_objects.md | docs/design/ | 功能分解类 | 2026-08-04 | 4.7 KB |
| 3 | requirements_list.md | docs/design/ | 功能分解类 | 2026-08-04 | 4.1 KB |
| 4 | mcu_resource_plan.md | docs/design/ | 资源规划类 | 2026-08-04 | 6.2 KB |
| 5 | pin_config_spec.md | docs/design/ | 资源规划类 | 2026-08-04 | 4.2 KB |
| 6 | software_function_spec.md | docs/design/ | 整合说明类 | 2026-08-04 | 20.3 KB |
| 7 | module_design.md | docs/design/ | 模块设计类 | 2026-08-05 | 9.7 KB |
| 8 | module_relationship_analysis.md | docs/design/ | 模块设计类 | 2026-08-05 | 18.7 KB |
| 9 | runtime_state_analysis.md | docs/design/ | 运行分析类 | 2026-08-05 | 21.7 KB |
| 10 | design_review.md | docs/design/ | 审查类 | 2026-08-05 | 9.4 KB |
| 11 | cubemx_config_checklist.md | docs/design/ | 配置类 | 2026-08-08 | 20.3 KB |
| 12 | initial_project_analysis.md | docs/analysis/ | 分析类 | 2026-08-08 | 15.0 KB |
| 13 | software_detailed_design.md | docs/design/ | 设计类 | 2026-08-08 | 34.3 KB |
| 14 | programming_task_list.md | docs/tasks/ | 任务类 | 2026-08-08 | 14.8 KB |
| 15 | project_context.md | docs/context/ | 上下文类 | 2026-08-06 | 11.9 KB |

合计 15 份，约 190 KB。

## 2. 文档链路关系

```
design_inputs.md（08-04，功能要求 FR-01~09）
   │
   ├─→ io_objects.md ──┐
   │                   ├─→ mcu_resource_plan.md ──┐
   └─→ requirements_list.md（SF-01~11）           │
                          ├─→ pin_config_spec.md ─┤
                          │                       ▼
                          └───────────→ software_function_spec.md（08-04，整合 5 份文档）
                                             │
                        ┌────────────────────┼─────────────────────┐
                        ▼                    ▼                     ▼
                 module_design.md    module_relationship_   runtime_state_analysis.md
                 （08-05，5 模块）     analysis.md（08-05）   （08-05，流程与状态）
                        │                    │                     │
                        └────────────┬───────┴──────────┬──────────┘
                                     ▼                  ▼
                              design_review.md   project_context.md
                              （08-05，审查 9 问题）  （08-06，项目全貌浓缩）
                                     │
                     ┌───────────────┼────────────────┐
                     ▼               ▼                ▼
          cubemx_config_checklist  initial_project_   software_detailed_design.md
          （08-08，CubeMX 参数）    analysis.md       （08-08，模块→文件、接口、时序）
                                     │                │
                                     └───────┬────────┘
                                             ▼
                                  programming_task_list.md
                                  （08-08，T01~T10）
```

## 3. 各文件内容摘要

### 3.1 需求设计类

**design_inputs.md —— 《项目设计输入》（2026-08-04）**
- 章节：项目名称 / 项目目标 / 项目功能要求（FR-01～FR-09）
- 内容：将候选功能主题整理为 9 条正式功能要求，仅收录已确认内容。关键参数：三组 10 kHz 带死区互补 PWM（第一组动态更新、二三组预设）、三通道注入式采样（1 路电位器动态电压 + 2 路高低参考电平）、DAC 固定频率参考方波、温度 500 Hz 全状态监测、双独立阈值（异常/恢复）避免临界抖动、三状态（停止/运行/异常）、JustFloat/115200 bps/500 Hz。注意：FR-09 仍写"异常状态发送温度"（已被后续修订为"异常不发送"，见 I-09）。

### 3.2 功能分解类

**io_objects.md —— 《软件输入输出对象》（2026-08-04）**
- 章节：输入对象 / 输出对象
- 内容：4 个输入事件（PwmPeriodEvent 10 kHz、AdcSequenceCompleteEvent、UserButtonEvent、Timer500HzEvent）、4 个输入数据（三路 12 位 ADC 值、ChipTemperature）、2 个输入参数（温度双阈值）、7 个输出控制（PwmDutyCh1 动态、Ch2/Ch3 预设、PWM/DAC 使能、DAC 值、LED、采样使能）、2 个输出数据（SerialTxData、SystemState）、2 个输出事件（TemperatureFaultEnter/Exit）。注意：SerialTxData 条目仍写"异常发温度"（见 I-09）。

**requirements_list.md —— 《软件功能需求列表》（2026-08-04）**
- 章节：单张功能列表大表（SF-01～SF-11）
- 内容：11 条软件功能，每条含输入/输出/触发条件/运行效果。核心约束：采样-处理-PWM/DAC 更新保持 10 kHz 稳定周期关系；SF-08 双独立阈值避免临界抖动。注意：SF-11 仍写"停止和异常状态发送 ChipTemperature"（见 I-09）。

### 3.3 资源规划类

**mcu_resource_plan.md —— 《MCU 资源规划》（2026-08-04）**
- 章节：资源映射 / 冲突检查（引脚复用、ADC1 共享、DMA、中断）/ 待确认事项
- 内容：TIM1 中心对齐 PWM（CH1-3 + 互补 6 脚，10 kHz）内部触发 ADC1 注入组；ADC1 注入组 Rank1-3 = PA1/PB1/PB0，规则组 CH16 内部温度传感器由 TIM6_TRGO 触发（500 Hz）；DAC1 CH1（PA4）在 0 与最新值间翻转形成 250 Hz 方波；LPUART1_TX + DMA1；TIM6（2 ms）承担温度触发、DAC 翻转、LED 分频、串口节拍四项职责。冲突检查全部通过，5 项待确认（HW-TBD-01～05）。

**pin_config_spec.md —— 《MCU 引脚配置规范》（2026-08-04）**
- 章节：引脚配置 / 引脚冲突检查（复用、板载电路、调试接口）/ 待确认事项
- 内容：16 个外部引脚完整定义，含 PA1 电位器接法、PB1/PB0 杜邦线参考电平、PC13 按键上升沿、PA5 LED 高电平点亮、PA13/PA14 SWD 禁用风险警示。仅 1 项待确认（PIN-TBD-01：VREF+ 需确认 JP8 置于 [2-3]）。

### 3.4 整合说明类

**software_function_spec.md —— 《软件功能说明书》（2026-08-04）**
- 章节：文档说明 / 项目目标与功能概述 / 软件 IO 对象 / 软件功能需求 / MCU 资源规划 / MCU 引脚配置 / 设计约束与待确认事项
- 内容：前五份文档的整合版。FR-04 明确为 250 Hz 方波，FR-09 明确"异常状态不发送"（与前三份早期文档不同，系 R-02 修订落点）。§4.2 给出三状态 × SF-01～11 行为总表；§7.1 关键约束含"SF-05 在 JEOS 中断中执行须远小于 100 µs"；§7.2 汇总 10 项待确认。注意：TBD-07~10 标注来源于 design_inputs 的 DD-01/02/05/06，但 design_inputs.md 正文无这些 DD 编号条目（见 I-08）。

### 3.5 模块设计类

**module_design.md —— 《软件模块设计》（2026-08-05，同日修订）**
- 章节：模块总览 / 模块详细说明（MD-01～MD-05）
- 内容：5 模块方案（适度方案）——MD-01 板级支持（纯外设原语、统一启停实时链路）、MD-02 采样管理（JEOS/EOC 分发）、MD-03 实时处理（纯计算）、MD-04 系统控制（独占 SystemState、状态机、500 Hz 调度）、MD-05 串口监测（由 MD-04 触发、不读 SystemState）。

**module_relationship_analysis.md —— 《模块关系与数据流分析》（2026-08-05，同日修订）**
- 章节：关系类型 / 关系总览（控制调用、事件通知、数据依赖）/ 主要事件分析 / 关键数据流分析（5 条链路）/ 潜在问题与待确认事项（5.1～5.7）
- 内容：调用关系全部单向无反向依赖（MD-02→MD-03→MD-01 等三条链）；关键时序结论——规则组转换完成与 TIM6 中断调度无确定先后关系，温度判断始终在规则组完成后执行；DAC 寄存器仅在 TIM6 中断中写入；7 项潜在问题（ISR 入口归属、JEOS/EOC 区分方式、DMA 完成标志、初始化顺序、MD-04 职责集中度、停止态温度有效性、module_design 同步更新）。

### 3.6 运行分析类

**runtime_state_analysis.md —— 《运行流程与状态切换分析》（2026-08-05）**
- 章节：运行流程总览 / 10 kHz 实时路径 / 500 Hz 周期处理 / 异步事件处理 / 系统状态切换 / 流程与状态关系图
- 内容：四条并发流程（10 kHz 实时、500 Hz 调度、500 Hz 温度、异步事件）；状态机含全部切换边与模块动作序列（启动：先使能注入组→启 TIM1/PWM→启 DAC；停止：先停 TIM1→停注入组→关 PWM→DAC 置零关闭）；停止态串口发送以"温度值有效"为前置条件。

### 3.7 审查类

**design_review.md —— 《设计审查报告》（2026-08-05）**
- 章节：需求覆盖 / 模块职责与边界 / 数据流与事件流 / 状态切换与运行流程 / 待确认事项汇总与优先级 / 问题汇总
- 内容：审查 5 份文档，发现 9 项问题——需求类 R-01（SF-03 未归模块）、R-02（异常态串口是否发送的前后文档矛盾）；职责类 M-01（NVIC 优先级配置归属）、M-02（DAC 启停遗漏）；数据类 D-01（JustFloat 协议未引用）、D-02（DAC 关闭未置零）、D-03（跨中断共享变量需 volatile）；状态类 S-01（首次温度无效）、S-02（异常状态重复进入）。修复结论：M-02/D-02/S-01/S-02 已修正，其余要求同步文档。待确认分两级：编程前必须确认 8 项、编程阶段细化 2 项。

### 3.8 配置类

**cubemx_config_checklist.md —— 《STM32CubeMX 工程配置清单》（2026-08-08）**
- 章节：MCU 与工程基本设置 / 系统与时钟配置 / GPIO / TIM1 与 PWM / ADC1 / TIM6 / DAC1 / LPUART1 与 DMA / NVIC / 工程生成设置 / 引脚配置汇总 / 后续程序编写阶段确定 / 配置后核对事项
- 内容：全部 CubeMX 配置参数已确定为具体数值——SYSCLK 160 MHz（HSE 24 MHz→PLL，M3×N40÷R2）、TIM1 中心对齐 PWM 10 kHz（PSC=0、ARR=8000）、CH4 产生内部 OC4REF 经 TRGO2 触发 ADC1 注入组（CCR4=4000）、死区 200 ns（DeadTime=32）、TIM6 500 Hz（PSC=159、ARR=1999）触发规则组温度采样、ADC 时钟 HCLK/4（40 MHz）、注入组三通道采样时间 24.5 cycles、温度通道 247.5 cycles、DAC1 软件写入、LPUART1 115200 bps + DMA1 Normal 模式、NVIC 优先级 ADC1_2=0 > TIM6_DAC=1 > 其余=2。唯一未定项：工程路径。§12 明确 8 项参数留给编码阶段（CH2/CH3 预设值、换算公式、温度阈值、消抖等）。

### 3.9 分析类

**initial_project_analysis.md —— 《初始工程结构分析》（2026-08-08）**
- 章节：工程目录与构建结构 / main.c 初始化流程和主循环结构 / 主要外设初始化函数和句柄 / 已生成的中断入口和相关 HAL 回调 / 后续应用代码的主要接入位置 / 其他与后续编程直接相关的情况
- 内容：分析已编译成功的纯 CubeMX 工程。要点：CMake+GCC、C11、`USE_HAL_DRIVER`/`STM32G431xx`；时钟 160 MHz（VOLTAGE_SCALE1_BOOST、FLASH_LATENCY_4）；外设初始化顺序 GPIO→DMA→TIM1→ADC1→LPUART1→DAC1→TIM6；实际 NVIC 优先级为 ADC1_2=0、TIM6_DAC=1、DMA1_Ch2=2、LPUART1=3、EXTI15_10=4（文中声明"已确认为最终配置"）；DMA1_Channel2 负责 LPUART1_TX；5 个 HAL 回调（注入组 JEOS、规则组 EOC、TIM6 周期、EXTI、UART TxCplt）是应用代码主要接入点，全部待实现；外设初始化后均未启动（TIM1/TIM6/ADC 启动由应用代码完成）。

### 3.10 设计类

**software_detailed_design.md —— 《软件详细设计》（2026-08-08，同日修订）**
- 章节：模块与文件组织设计 / 接口与数据传递设计 / 状态与时序设计
- 内容：5 模块落地为 `App/Inc`+`App/Src` 共 10 个文件（app_bsp/sampling/process/control/monitor）；main.c 仅加 2 处 USER CODE；MD-01 公开接口 13 个（实时链路启停、CCR1-3 写入、DAC 写/启/停、LED 翻转/熄灭、DMA 启动、TIM6 启动、ADC 规则组启动）；MD-04 独占 SystemState（STOP/RUN/FAULT 枚举定义于 app_control.c）；跨 ISR 数据原则——`volatile` + 单写单读、不引入临界区；状态转换表含 5 条边及完整进出动作；§3.5 给出实时链路启停的 HAL 调用序列（启动：注入组 IT → PWM_Start；停止：PWM_Stop → 注入组 Stop → DAC 置零关闭 → LED 熄灭）；初始化后首周期温度无效时串口跳过发送。

### 3.11 任务类

**programming_task_list.md —— 《编程任务清单》（2026-08-08，同日重组修订）**
- 章节：T01～T10 十个任务
- 内容：T01 建 App 骨架+CMake → T02 接口声明+空实现+main.c 接入 → T03 实现 MD-01 → T04 板级验证 10 kHz 注入链路（临时代码直启实时链路、LED 翻转验证 5 kHz、三通道数据核对）→ T05 实现 MD-03 并示波器验证 PWM 随电位器变化 → T06 打通 500 Hz+温度+按键（替换 T04 辅助代码）→ T07 完整状态机+DAC 250 Hz 方波+LED 三态 → T08 MD-05 JustFloat+DMA → T09 温度异常板级验证 → T10 综合验证。任务链 T01→T08 串行，T09/T10 并行。前置条件：T05 需 TBD-08/09、T06 需 CH2/CH3 预设值、T07 需 TBD-07。

### 3.12 上下文类

**project_context.md —— 《项目上下文说明》（v1.0 初版，2026-08-06）**
- 章节：项目名称与目标 / 硬件平台与开发环境 / 关键资源与硬件连接 / 软件结构与运行机制 / 设计文档索引 / 验证依据 / 已确认的技术约束
- 内容：基于 5 份已确认设计文档提取的项目全貌浓缩文档。§1 将 FR-01～FR-09 以表格完整呈现（FR-04 明确 250 Hz 方波、FR-09 明确"异常不发送"）；§2 完整外部连接清单与开发环境（VS Code + STM32CubeIDE for VS Code + CMake + GCC，板载 STLINK-V3E 烧录）；§3 六个外设参数表、14 个引脚表、硬件触发关系链（TIM1→ADC1 注入组、TIM6_TRGO→规则组）、上位机 VOFA+；§4 五模块职责、三条事件数据流、三状态行为矩阵、状态切换规则、运行周期表；§7 四类已确认约束（硬件、运行时序、数据通信、模块设计）。§6 验证依据标注 3 项"待确认"（编译验证、板级验证、PWM 波形验证），后续已被编程任务清单 T04～T10 细化。§5 设计文档索引已于 2026-08-24 补入 08-08 之后的 4 份文档。

## 4. 通读发现的问题记录

| 编号 | 问题 | 涉及文件 | 状态 |
|---|---|---|---|
| I-01 | MD-01 接口数量不一致：任务清单 T03 写"11 个"，详细设计 §2.1.1 现为 13 个 | programming_task_list.md vs software_detailed_design.md | 未处理 |
| I-02 | T06 缺口描述过时："缺少 ADC 规则组启动接口需新增 BSP_StartRegularAdc()"，但详细设计 §2.1.1 已包含该接口 | programming_task_list.md | 未处理 |
| I-03 | NVIC 优先级两处描述不同：配置清单 §9.1 给 LPUART1/EXTI/DMA 均标优先级 2；初始工程分析 §4.2 记录实际生成值为 DMA=2、LPUART1=3、EXTI=4，且均声明"已确认" | cubemx_config_checklist.md vs initial_project_analysis.md | 未处理 |
| I-04 | 详细设计 §3.5.1 启动序列中 CH2 的 `HAL_TIM_PWM_Start` 调用时机标注"待确认" | software_detailed_design.md | 未处理（属编码阶段确定） |
| I-05 | T06 定义 `SystemState_t` 仅 STOP/RUN，FAULT 留待 T07 | programming_task_list.md | 非缺陷（增量实现策略） |
| I-06 | project_context.md §5 索引滞后，缺 08-08 之后 4 份文档 | project_context.md | 已处理：2026-08-24 补入 4 项 |
| I-07 | project_context.md §4"调用关系（全部单向）："标题与"关键数据持有"正文挤在同一行 | project_context.md | 未处理（按要求不作修改） |
| I-08 | software_function_spec.md §7.2 将 TBD-07~10 标注来源为 design_inputs 的 DD-01/02/05/06，但 design_inputs.md 正文无这些 DD 编号条目，引用断裂 | software_function_spec.md | 未处理 |
| I-09 | R-02 修订后仍有 3 份早期文档残留旧口径"停止和异常状态发送芯片温度"（FR-09/SF-11/SerialTxData），未纳入 R-02 同步更新范围 | design_inputs.md、io_objects.md、requirements_list.md | 未处理 |
| I-10 | design_review.md 开头声明"docs/analysis/open_questions.md 不存在，TBD 从各文档提取"——该文件至今未创建，审查所提修复的落实跟踪无载体 | design_review.md | 未处理 |

## 5. 变更记录

| 日期 | 变更内容 |
|---|---|
| 2026-08-24 | 检查并通读第一批 10 份设计文档（design_inputs 等），输出清单 |
| 2026-08-24 | 检查并通读第二批 4 份文档（cubemx_config_checklist 等）及 project_context.md，纳入清单 |
| 2026-08-24 | project_context.md §5 设计文档索引补入后续 4 份文档，其他内容未作修改 |
| 2026-08-24 | 将全部 15 份设计文档统一整理，更新本设计文档清单日志（本次） |
