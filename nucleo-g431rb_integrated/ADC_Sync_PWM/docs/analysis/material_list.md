# 项目资料清单

> 编制日期：2026-08-24
> 扫描范围：`docs/raw/` 及其全部子目录
> 编制说明：本清单仅做资料登记与分类核查，不分析项目需求，不提出设计方案。

## 1. 清单编制方法

| 项目 | 说明 |
|---|---|
| 文件发现 | 对 `docs/raw/` 递归遍历，共发现 7 个文件、3 个子目录 |
| 可读性验证 | Markdown 文件直接通读；PDF 使用 `pdftotext` 抽取首页与目录页，确认可正常解析 |
| 版本识别 | 从 PDF 首页页脚提取文档编号、修订号与发布日期 |
| 页数统计 | PDF 按解析出的实际页数统计（原理图另以 `/Type /Page` 交叉校验） |
| 重复性判定 | 对全部 7 个文件计算 MD5 摘要并比对 |

## 2. 资料汇总表

| 序号 | 文件名称 | 所在目录 | 资料类别 | 格式 | 规模 | 版本/日期 | 状态 |
|---|---|---|---|---|---|---|---|
| 1 | requirements_initial.md | docs/raw/requirements/ | 项目需求资料 | Markdown | 335 B / 9 行 | 无版本标识 | 可读，内容偏简 |
| 2 | NUCLEO-G431RB_Schematic_C05.pdf | docs/raw/hardware/ | 硬件设计资料 | PDF | 18.4 MB / 13 页 | Rev C-05，2022-02-07 | 可读 |
| 3 | NUCLEO-G431RB_User_Manual_UM2505.pdf | docs/raw/hardware/ | 硬件设计资料 | PDF | 2.73 MB / 48 页 | UM2505 Rev 7，2026-05 | 可读 |
| 4 | STM32G431_Datasheet_DS12589.pdf | docs/raw/references/ | 技术参考资料 | PDF | 2.54 MB / 198 页 | DS12589 Rev 1 | 可读，版本待核 |
| 5 | STM32G4_Reference_Manual_RM0440.pdf | docs/raw/references/ | 技术参考资料 | PDF | 37.2 MB / 2141 页 | RM0440 Rev 9，2025-03 | 可读 |
| 6 | STM32G4_HAL_LL_Manual_UM2570.pdf | docs/raw/references/ | 技术参考资料 | PDF | 10.4 MB / 2926 页 | UM2570 Rev 2，2020-08 | 可读，版本待核 |
| 7 | VOFA_JustFloat_Protocol.md | docs/raw/references/ | 技术参考资料 | Markdown | 1.18 KB / 53 行 | 无版本标识 | 可读 |

合计：7 份资料，约 71.3 MB，PDF 总页数 5326 页。

## 3. 分类清单与内容说明

### 3.1 项目需求资料（1 份）

#### [1] requirements_initial.md

- **所在目录**：`docs/raw/requirements/`
- **资料类别**：项目需求资料 —— 初始需求描述
- **主要内容**：
  项目标题为"基于 ADC 同步采样的 PWM 占空比控制"。文档分两节：
  - **硬件平台**：指明使用 NUCLEO-G431RB 开发板。
  - **项目初步设想**：以一段文字概述功能轮廓，涉及由 PWM 周期事件触发多通道模拟电压采样、依据采样结果更新 PWM 占空比与 DAC 输出，以及芯片温度监测、系统状态切换、LED 指示、串口波形显示等功能方向。
- **性质说明**：属初步设想性质的纲要文档，非正式需求规格书。

### 3.2 硬件设计资料（2 份）

#### [2] NUCLEO-G431RB_Schematic_C05.pdf

- **所在目录**：`docs/raw/hardware/`
- **资料类别**：硬件设计资料 —— 板级原理图
- **主要内容**：
  ST 官方 NUCLEO64 STM32G4 开发板原理图，参考设计编号 MB1367，变体 G431RB，修订版本 C-05，图纸日期 2022-02-07，图幅 A4。文件共 13 个 PDF 页，其中包含 8 张编号图纸（Sheet 1 of 8 ~ 8 of 8）：

  | 图纸 | 标题 | 内容范围 |
  |---|---|---|
  | Sheet 1 | Project overview | 目录页、图例说明、Open Platform 授权协议条款 |
  | Sheet 2 | mb1367_Top / Top hierarchical view | 顶层层次化框图 |
  | Sheet 3 | STM32 microcontroller I/Os | MCU 各 I/O 引脚连接关系 |
  | Sheet 4 | STM32 microcontroller power part | MCU 供电与去耦网络 |
  | Sheet 5 | Arduino & Morpho extension connectors | ARDUINO Uno V3 与 ST morpho 扩展排针连接 |
  | Sheet 6 | Main power 5V / 3V3 | 主电源与 5V/3V3 稳压电路 |
  | Sheet 7 | STLINK_V3E | 板载调试器/编程器电路 |
  | Sheet 8 | Mechanical | 机械尺寸与结构信息 |

- **使用提示**：该 PDF 为 CAD 矢量导出件，文字层可被提取，但网络连线关系不体现在文本层中，查阅引脚级连接需以图形方式打开。

#### [3] NUCLEO-G431RB_User_Manual_UM2505.pdf

- **所在目录**：`docs/raw/hardware/`
- **资料类别**：硬件设计资料 —— 开发板用户手册
- **主要内容**：
  ST 官方文档 UM2505《STM32G4 Nucleo-64 boards (MB1367)》，Rev 7，2026 年 5 月发布，共 48 页。覆盖 NUCLEO-G431RB / NUCLEO-G474RE / NUCLEO-G491RE 三款同族板卡。主要章节包括：
  - **Features**：板卡特性列表。就 STM32G431RBT6 而言，标注为 LQFP64 封装、Arm Cortex-M4 @ 170 MHz、128 KB Flash、32 KB SRAM；板载 1 个用户 LED、1 个用户按键与 1 个复位按键、32.768 kHz LSE 晶振、24 MHz HSE 振荡器。
  - **Ordering information / Codification**：订购型号与命名编码规则。
  - **Development environment**：系统需求与所支持的开发工具环境。
  - **Conventions / Safety recommendations**：文档约定与安全须知。
  - **Quick start**：上手流程。
  - **Hardware layout and configuration**：板级布局与配置说明，含跳线与选项配置（如 JP3 Target reset、VREF 选择、U12 3V3 稳压器、去耦电容焊接与否的配置项）、LED 定义（LD2 USER、LD3 绿色电源指示、LD4 OC 过流指示）、板载连接器说明。文中标明 LPUART1（PA2/PA3）连接至 STLINK-V3E 的虚拟 COM 端口。
  - **STM32G4 Nucleo-64 I/O assignment**：板级 I/O 分配表。
  - **Board connectors**：USB Micro-AB、MIPI 调试连接器、ARDUINO Uno V3 扩展口、ST morpho 排针的引脚定义。
  - **STM32G4 Nucleo-64 product information**、**Compliance statements and conformity declarations**：产品信息与合规声明。

### 3.3 技术参考资料（4 份）

#### [4] STM32G431_Datasheet_DS12589.pdf

- **所在目录**：`docs/raw/references/`
- **资料类别**：技术参考资料 —— 芯片数据手册
- **主要内容**：
  ST 官方数据手册 DS12589，Rev 1，共 198 页，覆盖 STM32G431x6 / STM32G431x8 / STM32G431xB 全系列（含 STM32G431RB）。属 production data 级别文档。内容涵盖：
  - **特性概览**：Cortex-M4 + FPU 内核（170 MHz / 213 DMIPS，含 ART 加速器、MPU、DSP 指令）；VDD/VDDA 工作范围 1.71–3.6 V；CORDIC 三角函数加速器与 FMAC 滤波数学加速器；128 KB Flash（带 ECC、PCROP）、22 KB SRAM（前 16 KB 带硬件奇偶校验）、10 KB CCM SRAM；复位与电源管理（POR/PDR/BOR、PVD、sleep/stop/standby/shutdown 低功耗模式、VBAT）；时钟管理（4–48 MHz 晶振、32 kHz 振荡器、内部 16 MHz RC + PLL、内部 32 kHz RC）；最多 86 个快速 I/O；互联矩阵；12 通道 DMA 控制器。
  - **通信与外设接口**：1×FDCAN、3×I2C（Fast mode plus）、4×USART/UART、1×LPUART、3×SPI、1×SAI、USB 2.0 全速、UCPD、IRTIM、RNG、CRC、96 位唯一 ID、带闹钟的日历 RTC。
  - **调试支持**：SWD、JTAG、ETM。
  - **器件汇总表**、封装选型（LQFP32/48/64/80/100、UFQFPN32/48、UFBGA64、WLCSP49）。
  - 后续章节为引脚定义、存储器映射、电气特性参数、封装机械图与订购信息。

#### [5] STM32G4_Reference_Manual_RM0440.pdf

- **所在目录**：`docs/raw/references/`
- **资料类别**：技术参考资料 —— 系列参考手册
- **主要内容**：
  ST 官方参考手册 RM0440《STM32G4 series advanced Arm-based 32-bit MCUs》，Rev 9，2025 年 3 月发布，共 2141 页。面向应用开发者，提供 STM32G4 系列存储器与外设的完整寄存器级说明。已确认的主要章节结构（章号 / 起始页）：

  | 章 | 标题 | 页 |
  |---|---|---|
  | 1 | Documentation conventions | 74 |
  | 2 | System and memory overview | 79 |
  | 3–5 | Embedded flash memory (FLASH) | — |
  | 6 | Power control (PWR) | 228 |
  | 7 | Reset and clock control (RCC) | 270 |
  | 8 | Clock recovery system (CRS) | 334 |
  | 9 | General-purpose I/Os (GPIO) | 345 |
  | 10 | System configuration controller (SYSCFG) | 364 |
  | 11 | Peripherals interconnect matrix | 376 |
  | 12 | Direct memory access controller (DMA) | 397 |
  | 13 | DMA request multiplexer (DMAMUX) | 418 |
  | 14 | Nested vectored interrupt controller (NVIC) | 436 |
  | 15 | Extended interrupts and events controller (EXTI) | 441 |
  | 16 | Cyclic redundancy check calculation unit (CRC) | 454 |
  | 17 | CORDIC coprocessor | 461 |
  | 18 | Filter math accelerator (FMAC) | 479 |
  | 19 | Flexible static memory controller (FSMC) | 507 |
  | 20 | Quad-SPI interface (QUADSPI) | 563 |
  | 21 | Analog-to-digital converters (ADC) | 591 |
  | 22 | Digital-to-analog converter (DAC) | 714 |
  | 23 | Voltage reference buffer (VREFBUF) | 764 |
  | 24 | Comparator (COMP) | 768 |
  | 25 | Operational amplifiers (OPAMP) | 776 |
  | 26 | True random number generator (RNG) | 820 |
  | 27 | AES hardware accelerator | 832 |
  | 28 | High-resolution timer (HRTIM) | 881 |
  | 29 | Advanced-control timers (TIM1/TIM8/TIM20) | 1144 |
  | 30 | General-purpose timers (TIM2/TIM3/TIM4/TIM5) | 1287 |
  | 31+ | 其余定时器、通信接口（USART/LPUART/SPI/I2C/FDCAN/USB 等）、RTC、看门狗、调试支持 | 1287 以后 |

  首页"Related documents"列出的关联文档为：Cortex-M4 Technical Reference Manual、STM32G4xx datasheets、STM32G4xx erratasheets、PM0214 编程手册。

#### [6] STM32G4_HAL_LL_Manual_UM2570.pdf

- **所在目录**：`docs/raw/references/`
- **资料类别**：技术参考资料 —— 驱动库 API 手册
- **主要内容**：
  ST 官方文档 UM2570《Description of STM32G4 HAL and low-layer drivers》，Rev 2，2020 年 8 月发布，共 2926 页。属 STM32Cube 生态文档，内容包括：
  - **General information**、**Acronyms and definitions**：文档约定与外设缩写对照表（涵盖 ADC、DAC、DMA、DMAMUX、TIM、HRTIM、LPTIM、USART、UART、LPUART、SPI、I2C、COMP、CORDIC、CRC、NVIC、MSP 等）。
  - **Overview of HAL drivers**：HAL 层设计理念与使用模型。说明 HAL 提供面向功能、可移植性高的通用 API 与扩展 API，内建入参运行时校验；LL 层提供贴近寄存器、更轻量高效但可移植性较低的原子操作接口，仅对部分外设提供。两者互补。代码以 Strict ANSI-C 编写，经 CodeSonar 静态分析，符合 MISRA C:2012。
  - 后续为逐外设的 HAL/LL 驱动 API 参考：数据结构、函数原型、宏定义与使用流程。

#### [7] VOFA_JustFloat_Protocol.md

- **所在目录**：`docs/raw/references/`
- **资料类别**：技术参考资料 —— 上位机通信协议说明
- **主要内容**：
  依据 VOFA+ 官方说明整理的 JustFloat 协议文档，共 53 行。说明波形数据的组织与发送方式：
  - **帧格式**：小端浮点数组字节流，数据帧由若干浮点通道数据加 4 字节固定帧尾组成，并给出 C 结构体表示（`float ch_data[CH_COUNT]` + `unsigned char tail[4]`）。
  - **帧尾**：固定为 `0x00 0x00 0x80 0x7F`。
  - **帧长公式**：`4 × CH_COUNT + 4` 字节。
  - **单通道示例**：`CH_COUNT` 为 1 时每帧 8 字节，前 4 字节为浮点数小端表示，后 4 字节为固定帧尾。
  - **使用说明**：VOFA+ 接收端需在相应通信接口中选择 `JustFloat` 数据引擎。

## 4. 资料检查结果

### 4.1 可读性检查 —— 全部通过

| 文件 | 检查方式 | 结果 |
|---|---|---|
| requirements_initial.md | 全文通读 | 正常，无乱码 |
| VOFA_JustFloat_Protocol.md | 全文通读 | 正常，无乱码 |
| NUCLEO-G431RB_Schematic_C05.pdf | pdftotext 抽取 + 页数校验 | 正常，13 页均可解析 |
| NUCLEO-G431RB_User_Manual_UM2505.pdf | pdftotext 抽取首页与正文 | 正常 |
| STM32G431_Datasheet_DS12589.pdf | pdftotext 抽取首页与目录 | 正常 |
| STM32G4_Reference_Manual_RM0440.pdf | pdftotext 抽取首页与目录 | 正常 |
| STM32G4_HAL_LL_Manual_UM2570.pdf | pdftotext 抽取首页与目录 | 正常 |

**结论**：无无法读取的文件。

需说明的两点解析特征（不影响资料有效性）：
1. 原理图 PDF 的文本层不包含网络连线关系，引脚级连接须以图形方式查阅。
2. UM2505 与 UM2570 的目录页采用表格排版，纯文本抽取时目录条目不完整，需按正文页码定位。

### 4.2 重复性检查 —— 未发现重复

对 7 个文件计算 MD5 摘要，结果两两互不相同，不存在内容完全重复的文件：

```
f0ed16904e074ecd26f7de53108eaf89  requirements/requirements_initial.md
949715f878750bfcf4c526adccdf1cf6  hardware/NUCLEO-G431RB_Schematic_C05.pdf
9c540706d031855fa0a925afc420eef3  hardware/NUCLEO-G431RB_User_Manual_UM2505.pdf
97fd386cd5f31ba35a9dc5a9a7941921  references/STM32G431_Datasheet_DS12589.pdf
4e30e18122ae5636ef4631258733163d  references/STM32G4_Reference_Manual_RM0440.pdf
37bc67911da1d3243b3b0bc891a3cf88  references/STM32G4_HAL_LL_Manual_UM2570.pdf
eca94fc19f1a0e63e9ed46a20f1d92e6  references/VOFA_JustFloat_Protocol.md
```

**内容交叉说明**（属互补关系，非重复，无需处理）：
- [2] 原理图与 [3] 用户手册在引脚分配、跳线配置、连接器定义上存在内容交叉。原理图给出电路实现，用户手册给出表格化说明，查阅时可相互印证。
- [4] 数据手册与 [5] 参考手册在外设特性概述上存在交叉。数据手册侧重电气参数与封装引脚，参考手册侧重寄存器与功能行为，分工明确。

### 4.3 分类合理性检查 —— 分类合理

现有 `requirements/` `hardware/` `references/` 三个子目录与"项目需求资料 / 硬件设计资料 / 技术参考资料"三类一一对应，7 份资料的归属均无误置。三点需留意之处：

1. **[3] UM2505 兼具双重属性**。该手册既含板级硬件描述（I/O 分配、跳线、连接器），也含开发环境与工具链说明，后者偏技术参考性质。现归入 `hardware/` 合理，因其核心内容为板级硬件描述；检索开发环境相关信息时需注意该文档也在此目录下。
2. **[7] VOFA_JustFloat_Protocol.md 的归类**。该文档为外部上位机的协议规范，归入技术参考资料合理。但它同时构成串口数据格式的对接依据，与需求侧关联度高于其余三份参考资料，检索时不应遗漏。
3. **资料分布高度不均**。需求类仅 1 份 335 字节，参考类 4 份约 51 MB。这一分布本身不构成分类问题，但反映出需求侧资料的沉淀明显少于技术侧（详见 4.4）。

### 4.4 缺失资料提示

以下为清单编制阶段发现的资料层面缺口，仅作登记，不涉及需求分析：

| 编号 | 缺口 | 依据 | 影响面 |
|---|---|---|---|
| M1 | 需求资料仅 1 份且篇幅仅 9 行，为"初步设想"性质，未见正式需求规格书或指标清单 | 目录内实际文件情况 | 需求基线 |
| M2 | 缺少 STM32G4 系列勘误手册（Errata Sheet） | RM0440 首页 "Related documents" 明确列出 STM32G4xx erratasheets | 技术参考 |
| M3 | 缺少 PM0214《STM32 Cortex-M4 MCUs and MPUs programming manual》 | RM0440 首页 "Related documents" 明确列出 | 技术参考 |
| M4 | 缺少 Cortex-M4 Technical Reference Manual | RM0440 首页 "Related documents" 明确列出 | 技术参考 |
| M5 | `docs/raw/` 内无 CubeMX 工程配置说明；工程配置文件 `ADC_Sync_PWM.ioc` 位于工程根目录，未纳入资料体系 | 目录对比 | 配置追溯 |
| M6 | 仅有 VOFA+ 协议说明，无 VOFA+ 上位机使用/配置说明 | [7] 内容范围 | 验证环节 |

### 4.5 版本核对提示

以下两份资料的修订号相对偏低，建议确认是否为最新版本后再作为依据使用：

| 文件 | 当前版本 | 说明 |
|---|---|---|
| [4] STM32G431_Datasheet_DS12589.pdf | Rev 1 | 同批参考资料中 RM0440 已至 Rev 9，该数据手册仍为首版 |
| [6] STM32G4_HAL_LL_Manual_UM2570.pdf | Rev 2，2020-08 | 发布日期为 2020 年，是全部资料中最早的一份 |

## 5. 备注

1. 本清单覆盖范围限于 `docs/raw/`。`docs/` 下另有 `analysis/` `context/` `design/` `logs/` 四个目录，其中 `context/context_template.md`（976 B）为各字段均为空的项目上下文空白模板，属待填写的工作文档，不属于原始资料，未纳入本清单。
2. 表中文件大小按 1 MB = 1048576 B 换算，保留一至两位小数。
3. 本清单为资料登记与核查结果，未对项目需求作任何分析，也未提出任何设计方案。
