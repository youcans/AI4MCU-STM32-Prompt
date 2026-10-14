# 编程计划 T01 — 创建应用程序目录、文件骨架和构建配置

> 编制日期：2026-08-24
> 任务依据：
> - `docs/tasks/programming_task_list.md` T01（任务目标、涉及文件、实施要点、验证方式）
> - `docs/design/software_detailed_design.md` §1.1～§1.5（目录结构、文件职责、依赖关系、CMake 配置）
> - `docs/analysis/initial_project_analysis.md` §1.2、§5.4（构建结构与用户扩展点）
> 状态：待开发者审阅和批准。本阶段只生成修改计划，不修改任何工程文件。

---

## 1. 任务范围与约束遵循

| 硬性约束 | 本计划的遵循方式 |
|---|---|
| 以任务清单、详细设计、实际工程为依据 | 所有新建文件路径、包含关系、CMake 配置均取自详细设计 §1.1/§1.4/§1.5.4；CMake 插入位置经实际读取 CMakeLists.txt 核实 |
| 严格限定在 T01 任务范围内 | 仅新建 App 目录与 10 个骨架文件、修改 CMakeLists.txt；不声明函数接口、不建立函数骨架（均属 T02），不实现实际业务逻辑（属 T03 以后），不修改 main.c |
| 不自行改变已确定的模块职责和公开接口 | 骨架文件不含任何声明与实现，不引入任何接口；头文件不建立任何包含关系（具体依赖留待 T02 按实际接口需要添加） |
| 不修改任何工程文件或设计文件 | 本阶段仅产出本计划文档；正文所列代码均为待批准的修改内容，批准前不落盘 |

## 2. 当前工程现状（实测核实）

- `App/` 目录**不存在**，需新建 `App/Inc/`、`App/Src/` 两个子目录。
- 顶层 `CMakeLists.txt` 的 USER 扩展区已就位（实测读取）：
  - 第 46～48 行：`target_sources(...)` 块内有注释 `# Add user sources here`；
  - 第 51～53 行：`target_include_directories(...)` 块内有注释 `# Add user defined include paths`；
  - 构建目标名为 `${CMAKE_PROJECT_NAME}`（= `ADC_Sync_PWM`），链接 `stm32cubemx` 库。
- 新文件 include guard 采用普通宏名（`APP_BSP_H` 等），不使用双下划线形式（2026-08-24 审阅修订）。
- `Core/Inc/main.h` 已包含 `stm32g4xx_hal.h` 并定义 `LD2_Pin/LD2_GPIO_Port`、`B1_Pin/B1_GPIO_Port` 等引脚宏；外设头文件（adc.h/tim.h/dac.h/gpio.h/usart.h）均 `#include "main.h"`。
- `build/Debug` 目录已存在（当前工程编译通过），git 工作区干净。

## 3. 修改文件汇总

| 操作 | 文件 | 说明 |
|---|---|---|
| 新建目录 | `App/Inc/`、`App/Src/` | 应用模块头文件与源文件目录 |
| 新建 | `App/Inc/app_bsp.h` | MD-01 板级支持模块头文件 |
| 新建 | `App/Inc/app_sampling.h` | MD-02 采样管理模块头文件 |
| 新建 | `App/Inc/app_process.h` | MD-03 实时处理模块头文件 |
| 新建 | `App/Inc/app_control.h` | MD-04 系统控制模块头文件 |
| 新建 | `App/Inc/app_monitor.h` | MD-05 串口监测模块头文件 |
| 新建 | `App/Src/app_bsp.c` | MD-01 源文件骨架（仅含本模块头文件引用） |
| 新建 | `App/Src/app_sampling.c` | MD-02 源文件骨架（仅含本模块头文件引用） |
| 新建 | `App/Src/app_process.c` | MD-03 源文件骨架（仅含本模块头文件引用） |
| 新建 | `App/Src/app_control.c` | MD-04 源文件骨架（仅含本模块头文件引用） |
| 新建 | `App/Src/app_monitor.c` | MD-05 源文件骨架（仅含本模块头文件引用） |
| 修改 | `CMakeLists.txt` | target_sources / target_include_directories 两处 USER 区 |

## 4. 新建文件具体内容

以下为各文件的完整内容，供开发者审阅。

### 4.1 头文件（5 个）

**包含关系设计依据**（2026-08-24 审阅修订）：T01 阶段各应用头文件**不添加任何包含关系**（不包含 main.h、不包含 adc.h/tim.h/usart.h 等外设头文件、不包含其他 App 头文件），仅建立 include guard 骨架。具体依赖关系在 T02 声明接口时按实际需要添加（详细设计 §1.4 的文件依赖关系届时作为 T02 的包含依据）。本任务头文件中**不声明任何函数**。

#### 4.1.1 App/Inc/app_bsp.h

```c
/**
  ******************************************************************************
  * @file    app_bsp.h
  * @brief   MD-01 板级支持模块头文件（T01 骨架，接口在 T02 声明）
  ******************************************************************************
  */
#ifndef APP_BSP_H
#define APP_BSP_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* （T02 按接口需要添加，本任务不添加） */

/* Public function declarations ----------------------------------------------*/
/* （留待 T02 声明，本任务不添加） */

#ifdef __cplusplus
}
#endif

#endif /* APP_BSP_H */
```

#### 4.1.2 App/Inc/app_sampling.h

```c
/**
  ******************************************************************************
  * @file    app_sampling.h
  * @brief   MD-02 采样管理模块头文件（T01 骨架，接口在 T02 声明）
  ******************************************************************************
  */
#ifndef APP_SAMPLING_H
#define APP_SAMPLING_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* （T02 按接口需要添加，本任务不添加） */

/* Public function declarations ----------------------------------------------*/
/* （留待 T02 声明，本任务不添加） */

#ifdef __cplusplus
}
#endif

#endif /* APP_SAMPLING_H */
```

#### 4.1.3 App/Inc/app_process.h

```c
/**
  ******************************************************************************
  * @file    app_process.h
  * @brief   MD-03 实时处理模块头文件（T01 骨架，接口在 T02 声明）
  ******************************************************************************
  */
#ifndef APP_PROCESS_H
#define APP_PROCESS_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* （T02 按接口需要添加，本任务不添加） */

/* Public function declarations ----------------------------------------------*/
/* （留待 T02 声明，本任务不添加） */

#ifdef __cplusplus
}
#endif

#endif /* APP_PROCESS_H */
```

#### 4.1.4 App/Inc/app_control.h

```c
/**
  ******************************************************************************
  * @file    app_control.h
  * @brief   MD-04 系统控制模块头文件（T01 骨架，接口在 T02 声明）
  ******************************************************************************
  */
#ifndef APP_CONTROL_H
#define APP_CONTROL_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* （T02 按接口需要添加，本任务不添加） */

/* Public function declarations ----------------------------------------------*/
/* （留待 T02 声明，本任务不添加） */

#ifdef __cplusplus
}
#endif

#endif /* APP_CONTROL_H */
```

#### 4.1.5 App/Inc/app_monitor.h

```c
/**
  ******************************************************************************
  * @file    app_monitor.h
  * @brief   MD-05 串口监测模块头文件（T01 骨架，接口在 T02 声明）
  ******************************************************************************
  */
#ifndef APP_MONITOR_H
#define APP_MONITOR_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* （T02 按接口需要添加，本任务不添加） */

/* Public function declarations ----------------------------------------------*/
/* （留待 T02 声明，本任务不添加） */

#ifdef __cplusplus
}
#endif

#endif /* APP_MONITOR_H */
```

### 4.2 源文件骨架（5 个）

每个源文件骨架仅包含本模块头文件引用，无任何函数实现、无私有变量。

#### 4.2.1 App/Src/app_bsp.c

```c
/**
  ******************************************************************************
  * @file    app_bsp.c
  * @brief   MD-01 板级支持模块（T01 骨架，实现留待 T03）
  ******************************************************************************
  */
#include "app_bsp.h"
```

#### 4.2.2 App/Src/app_sampling.c

```c
/**
  ******************************************************************************
  * @file    app_sampling.c
  * @brief   MD-02 采样管理模块（T01 骨架，实现留待 T04/T06）
  ******************************************************************************
  */
#include "app_sampling.h"
```

#### 4.2.3 App/Src/app_process.c

```c
/**
  ******************************************************************************
  * @file    app_process.c
  * @brief   MD-03 实时处理模块（T01 骨架，实现留待 T05）
  ******************************************************************************
  */
#include "app_process.h"
```

#### 4.2.4 App/Src/app_control.c

```c
/**
  ******************************************************************************
  * @file    app_control.c
  * @brief   MD-04 系统控制模块（T01 骨架，实现留待 T04/T06/T07）
  ******************************************************************************
  */
#include "app_control.h"
```

#### 4.2.5 App/Src/app_monitor.c

```c
/**
  ******************************************************************************
  * @file    app_monitor.c
  * @brief   MD-05 串口监测模块（T01 骨架，实现留待 T08）
  ******************************************************************************
  */
#include "app_monitor.h"
```

## 5. CMakeLists.txt 修改方案

在顶层 `CMakeLists.txt` 的两个 USER 区各插入内容，注释行 `# Add user sources here` 与 `# Add user defined include paths` 保留在原位（其为 USER 区定位标记，与工程现状一致）。

### 5.1 修改 1：target_sources（第 46～48 行区域）

修改前：

```cmake
# Add sources to executable
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user sources here
)
```

修改后：

```cmake
# Add sources to executable
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user sources here
    App/Src/app_bsp.c
    App/Src/app_sampling.c
    App/Src/app_process.c
    App/Src/app_control.c
    App/Src/app_monitor.c
)
```

### 5.2 修改 2：target_include_directories（第 51～53 行区域）

修改前：

```cmake
# Add include paths
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined include paths
)
```

修改后：

```cmake
# Add include paths
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined include paths
    App/Inc
)
```

### 5.3 说明

- 源文件逐项显式列出（与详细设计 §1.5.4 完全一致），不采用 `App/Src/*.c` 通配——通配需 `file(GLOB)` 且在新增文件后须重新运行 CMake 才能发现，显式列表在 T02 编译路径上更可控。
- 不修改 `target_compile_definitions` 区域、不修改 `cmake/stm32cubemx/` 下的生成文件、不修改 `CMakePresets.json`。
- 文件路径使用正斜杠相对路径，与 CMake 跨平台惯例一致。

## 6. 接口与调用关系影响分析

| 项目 | T01 结论 |
|---|---|
| 新增接口 | **无**。所有公开函数声明留待 T02，所有函数实现留待 T03 及以后 |
| 对现有代码的调用 | **无**。main.c、各外设 .c、stm32g4xx_it.c 均不做修改，不产生任何调用关系 |
| 模块间依赖 | **无**。T01 头文件不建立任何包含关系，App 模块间无函数级依赖；具体包含关系（main.h、外设头文件、App 头文件）在 T02 声明接口时按详细设计 §1.4 按实际需要添加 |
| 对 HAL 库 | 不包含、不修改 HAL 任何文件 |
| 后续任务衔接 | T02 在各 .h 中按接口实际需要添加包含关系并声明接口、各 .c 中建立空实现并在 main.c 接入 `APP_Control_Init()`；本任务仅提供文件骨架 |

## 7. 验证方法

### 7.1 编译与链接验证

1. 在 VS Code（STM32CubeIDE for VS Code 扩展）中重新配置并构建工程，或使用 CMake 预设命令行构建：
   ```bash
   cmake --preset <现有预设名>
   cmake --build build/Debug
   ```
   （预设名以 `CMakePresets.json` 实际内容为准；工程当前已有可用的构建输出目录 `build/Debug`）
2. **验收标准**：编译 0 错误 0 警告，链接通过，生成目标可执行文件。此时 App 模块代码虽参与编译但未接入运行路径，固件行为与初始工程完全一致（main.c 未改动）。
3. **附加核对**：确认 `App/Inc` include 路径生效——骨架 .c 若 include 失败会在编译期报"找不到头文件"，因此"编译通过"本身即证明目录与 CMake 配置正确。

### 7.2 验证范围说明

- T01 只进行编译和链接验证，**不安排烧录和板级验证**（无功能逻辑变化，任务清单亦未要求）。板级验证从 T04 起按任务清单安排执行。

## 8. 实施顺序建议

1. 创建 `App/Inc/`、`App/Src/` 目录；
2. 按 §4.1 顺序新建 5 个头文件；
3. 按 §4.2 顺序新建 5 个源文件；
4. 按 §5 修改 CMakeLists.txt 两处 USER 区；
5. 执行 §7.1 编译验证；
6. 记录验证结果，待批准后进入 T02。

## 9. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 空 .c 文件 | GCC 编译空翻译单元（仅含本模块头文件引用）无告警问题；若个别工具链版本对"无函数定义"报错，可保留本方案（当前 GCC 环境实测惯例可行） |
| include guard 命名 | 采用 `APP_BSP_H`、`APP_SAMPLING_H` 等普通宏名（2026-08-24 审阅修订，不使用双下划线形式；5 个文件同步） |
| T01 不含包含关系 | 头文件不包含 main.h、外设头文件和其他 App 头文件（2026-08-24 审阅修订）；编译时 App .c 仅解析本模块头文件，不依赖任何工程头文件，编译通过不受包含关系影响 |
| T02 边界 | 头文件"暂不声明接口"为任务清单硬性要求；若审阅时希望提前声明接口，应留待 T02 执行，不在本任务内扩展 |
| 与设计文档的已知差异 | 任务清单 T03 写"11 个公开接口"而详细设计 §2.1.1 为 13 个（问题 I-01），不影响 T01，已在设计日志记录，待 T03 前解决 |
