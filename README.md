# AI4MCU-STM32G4-Prompt

**AI 驱动的 STM32G4 开发流程 · 结构化提示词配套代码**

**中文** | [English](README_en.md)

> **作者**：Huang Xiaofeng <ai4mcu@gmail.com>、Huang Shan <youcans@qq.com>

## 1. 项目简介

本项目是《AI 驱动的 STM32 开发流程》的配套代码仓库，面向 STM32G4 系列 MCU，使用**结构化提示词**实现 AI 协同开发。

仓库提供基础外设、系统架构和综合实验三类**项目工作目录**，每个实验项目均备有开发所需的初始文件，读者可直接进入相应目录开始实践。同时，仓库收录已按完整开发流程完成的**参考项目**，以及 NUCLEO-G431RB 开发板的**硬件资料**，供读者对照学习。

## 2. 仓库目录结构

```text
AI4MCU-STM32G4-Prompt/
├── .vscode/                     VS Code 工作区配置
├── docs/                        开发笔记、教程文档与学习资源
├── nucleo-g431rb_basics/        项目工作目录：基础外设实验（15 个项目）
├── nucleo-g431rb_system/        项目工作目录：系统架构实验（4 个项目）
├── nucleo-g431rb_integrated/    项目工作目录：综合实验
├── nucleo-g431rb_completed/     已完成开发流程的完整参考项目
└── nucleo-g431rb_hardware/      NUCLEO-G431RB 硬件资料
```

## 3. 工作目录说明

### 3.1 项目工作目录

`nucleo-g431rb_basics/`、`nucleo-g431rb_system/`、`nucleo-g431rb_integrated/` 是读者按书籍章节进行实践的**项目工作目录**：

- `nucleo-g431rb_basics/`：基础外设实验，覆盖 GPIO、定时器、PWM、ADC、DAC、USART、I2C、SPI、中断、DMA、RTC、看门狗、Flash 等；
- `nucleo-g431rb_system/`：系统架构实验，覆盖多任务轮询、事件驱动、状态机、FreeRTOS；
- `nucleo-g431rb_integrated/`：综合实验，覆盖 ADC 同步触发 PWM 等硬件协同应用。

每个实验项目目录中，仓库预先准备了**两个基本输入文件**，是项目开发所需的起点：

```text
nucleo-g431rb_basics/
└── GPIO_EXTI/
    ├── GPIO_EXTI.ioc         STM32CubeMX 工程配置文件
    └── requirements.txt      项目需求描述文件
```

- **`.ioc` 文件**：已配置好本实验所需的外设、引脚、时钟等基础参数。读者使用 STM32CubeMX 打开该文件即可生成初始工程，无需从零配置。
- **`requirements.txt`**：以自然语言描述本实验的目标、功能要求和硬件信息。在执行结构化提示词时，可以作为项目需求的输入。

### 3.2 完整参考项目

`nucleo-g431rb_completed/` 收录已按完整开发流程完成的项目，供读者在自主完成实验后对照参考。

### 3.3 硬件资料

`nucleo-g431rb_hardware/` 提供 NUCLEO-G431RB 开发板的数据手册、原理图、用户手册、BOM 等硬件资料，供开发时查阅。

## 4. AI 协同开发流程

本流程以已经完成系统设计、STM32CubeMX 配置并生成工程为前提，包括六个基本环节。

**（1）确认初始工程状态**
确认当前工程可以正常编译，并将这一状态作为后续修改的基线。

**（2）使用 Claude Code 分析工程**
让 Claude Code 读取 `.ioc` 文件、初始化代码和相关源文件，了解实现任务所需的工程条件，为后续任务规划和代码审阅提供依据。

**（3）根据分析结果制定任务清单**
要求 Claude Code 根据功能目标生成代码实现任务清单，任务按完整功能拆分，说明每项任务的目标、涉及文件和主要修改位置。

**（4）按照任务清单逐项修改代码**
每次选择一项任务，要求 Claude Code 生成相应的代码修改，经审阅确认和功能验证后，再进入下一项任务。

**（5）审阅并确认代码修改**
开发者审阅 Claude Code 生成的修改，确认修改文件、代码位置和实现功能是否与当前任务一致，再决定接受或要求重新生成。

**（6）编译、烧录与运行验证**
接受修改后重新编译工程，烧录到开发板，通过实际运行结果验证功能是否满足任务要求；未通过时可将错误信息反馈给 Claude Code 协助分析。

上述环节可归纳为三项基本原则：**修改前要有依据、修改过程要审阅、修改后要验证**。

## 5. 开发环境与工具链

- **MCU**：STM32G4 系列（STM32G431RB 等）
- **开发板**：NUCLEO-G431RB
- **IDE**：STM32 VS Code 扩展
- **固件库**：HAL / LL 库
- **调试工具**：ST-Link
- **AI 协作工具**：Claude Code（或其他支持结构化提示词的 AI 编程助手）

## 6. 快速开始

1. **克隆本仓库**
       git clone https://github.com/youcans/AI4MCU-STM32G4-Prompt.git

2. **进入实验项目目录**
   以基础外设实验 `GPIO_EXTI` 为例：
       cd nucleo-g431rb_basics/GPIO_EXTI

   该目录中已包含本实验所需的两个基本输入文件：
       GPIO_EXTI.ioc
       requirements.txt

3. **使用 STM32CubeMX 生成初始工程**
   双击 `GPIO_EXTI.ioc` 文件，用 STM32CubeMX 打开，检查配置后点击 **GENERATE CODE** 生成初始工程。

4. **确认初始工程状态**
   用 VS Code 打开该项目目录，等待 CMake 自动配置后，点击 **Build** 按钮编译。确认初始工程能够正常编译，并将此状态作为后续修改的基线。

5. **开始 AI 协同开发**
   初始工程确认后，即可按照 [AI 协同开发流程](#4-ai-协同开发流程) 一节所述，依次完成工程分析、任务规划、代码修改与验证。

## 7. 许可证

本项目仅供学习与参考使用。

## 8. 联系作者

- 邮箱：ai4mcu@gmail.com
- GitHub：[@youcans](https://github.com/youcans)