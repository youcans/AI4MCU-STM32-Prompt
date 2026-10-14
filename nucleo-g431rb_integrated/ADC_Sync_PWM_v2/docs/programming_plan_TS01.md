# TS01 代码修改方案：OLED 显示基础建立

---

## 0. 文档说明

本方案为任务 TS01（[见 task_list_v2.md](task_list_v2.md)）的代码修改方案，依据 [software_spec_SE_v2.md](software_spec_SE_v2.md)、[second_develop_analysis.md](second_develop_analysis.md) 与当前实际源码制定。

- 本轮**只制定 TS01 方案，不修改任何程序文件**；不实施 TS02 的显示内容组织、刷新与系统功能接入；不修改已完成的 CubeMX 配置；不修改 HAL/CMSIS 底层文件。
- 所引源码位置均经实际核查（本方案末尾附"源码事实核查"）。
- TS01 目标（据 [task_list_v2.md](task_list_v2.md)）：建立 OLED 显示模块，完成 SSD1306 基础驱动和显示所需数据接口，并通过固定测试内容验证 OLED 能够正常初始化和显示。

---

## 1. 任务目标

建立 OLED 显示模块并完成两件事：

1. **SSD1306 基础驱动**：经已配置的 I2C1 完成 SSD1306 初始化，并提供命令/数据发送与最基础的显存写入能力；
2. **显示所需数据接口**：暴露后续显示要用的只读数据访问接口（重点是当前唯一缺少现成接口的系统状态），供 TS02 显示功能使用。

并通过**固定测试内容**（非运行数据）验证 OLED 能正常初始化并在屏幕上显示出预定内容，证明驱动与 I2C1 通路可用。本任务不接入系统运行数据的实时显示与刷新（属 TS02）。

---

## 2. 现有条件（已完成、可复用，不需重新建立）

### 2.1 I2C1 硬件初始化已由 CubeMX 生成并接入

- 句柄：[i2c.c:27](../Core/Src/i2c.c) `I2C_HandleTypeDef hi2c1;`，[i2c.h:35](../Core/Inc/i2c.h) 已 `extern` 声明 `extern I2C_HandleTypeDef hi2c1;`。
- 初始化函数：[i2c.c:30](../Core/Src/i2c.c) `MX_I2C1_Init()`，配置为 7 位寻址（`I2C_ADDRESSINGMODE_7BIT`）、Timing=`0x10F32A82`（快速模式，与 `.ioc` `I2C1.I2C_Speed_Mode=I2C_Fast` 一致）。
- MspInit 已配置 PB8/PB9 复用开漏等 GPIO（[i2c.c:73 HAL_I2C_MspInit](../Core/Src/i2c.c)）。
- **已在 main() 调用**：[main.c:101](../Core/Src/main.c) `MX_I2C1_Init();`，位于 `USER CODE BEGIN 2`（[main.c:102](../Core/Src/main.c)）之前。即系统启动时 I2C1 已完成初始化，显示模块可直接使用 `hi2c1`。

### 2.2 HAL I2C 阻塞发送 API 可用

- [stm32g4xx_hal_i2c.h:624](../Drivers/STM32G4xx_HAL_Driver/Inc/stm32g4xx_hal_i2c.h) 已声明 `HAL_I2C_Master_Transmit(...)`（阻塞模式）。
- [stm32g4xx_hal_conf.h:50](../Core/Inc/stm32g4xx_hal_conf.h) 已启用 `#define HAL_I2C_MODULE_ENABLED`，I2C HAL 模块参与编译。
- TS01 基础驱动**采用阻塞模式**发送命令与显存数据（[second_develop_analysis.md §4.3](second_develop_analysis.md) 推荐阻塞模式以对现有 NVIC 优先级体系零影响，且显示慢速、主循环空闲——TS01 验证阶段在初始化/固定测试中同步发送，无实时性要求）。

### 2.3 显示数据访问接口现状（据 [second_develop_analysis.md §3](second_develop_analysis.md)）

| 显示数据 | 现有接口 | 现状 |
|---|---|---|
| 芯片温度 | `SAMP_GetTemperature()` ([app_sampling.h:19](../App/Inc/app_sampling.h)) | 已有，TS01 不改 |
| 温度有效性 | `SAMP_IsTemperatureValid()` ([app_sampling.h:20](../App/Inc/app_sampling.h)) | 已有，TS01 不改 |
| DAC 幅值/动态电压 | `PROC_GetDacAmplitude()` ([app_process.h:20](../App/Inc/app_process.h)) | 已有，TS01 不改 |
| **系统状态** | **无** | `SystemState_t` 枚举与 `static volatile system_state` 均定义在 [app_control.c:14-19](../App/Src/app_control.c) 内部，[app_control.h](../App/Inc/app_control.h) 仅声明 3 个函数、无 getter。**TS01 需新增只读状态接口** |

> 注：温度/DAC 幅值等接口虽 TS01 可声明由显示模块引用，但 TS01 的"固定测试内容"验证不依赖运行数据（用硬编码测试内容），故 TS01 阶段这些已有接口仅需保证显示模块头文件有正确的包含路径，不实际调用展示；其实际调用展示属 TS02。本任务"显示所需数据接口"的核心实质新增项是**系统状态接口**。

### 2.4 工程接入点与构建

- main.c 用户代码区齐备：`USER CODE BEGIN Includes`（[main.c:30](../Core/Src/main.c)）、`USER CODE BEGIN 2`（[main.c:102](../Core/Src/main.c)）、`USER CODE BEGIN WHILE`/`BEGIN 3`（[main.c:107](../Core/Src/main.c)）。
- 构建注册：[CMakeLists.txt:46-52](../CMakeLists.txt) 的 `target_sources` 列出 5 个 `App/Src/*.c`；[CMakeLists.txt:56-58](../CMakeLists.txt) 的 `target_include_directories` 已注册 `App/Inc`。新增显示模块源文件需加入前者，头文件放入 `App/Inc` 即可被包含（include 目录已注册，无需改动）。

### 2.5 模块组织风格（须遵循，据 [CLAUDE.md](../CLAUDE.md)）

- 文件命名 `app_<模块>.c/h`、公开接口前缀为模块缩写（参 [app_control.h](../App/Inc/app_control.h)）。
- 头文件含 `@file`/`@brief` 文件头注释、`#ifndef` 守卫、`extern "C"`、`/* Public function declarations */` 风格分节。
- 配置常量用大写 `#define`（参 [app_control.c:22-27](../App/Src/app_control.c)），私有变量 `static`，跨 ISR 共享用 `volatile`。

---

## 3. 需要新增或修改的文件、接口及其职责

### 3.1 新增：`App/Inc/app_display.h`

显示模块头文件。声明模块对外接口（初始化与基础显示/测试入口，具体签名在实施时按风格确定）。职责：对外提供 OLED 初始化与驱动使用入口；暴露 TS02 后续将调用的显示基础接口。

### 3.2 新增：`App/Src/app_display.c`

显示模块实现。职责：

- 经 `hi2c1`（[i2c.h:35](../Core/Inc/i2c.h) extern 句柄）以 `HAL_I2C_Master_Transmit`（阻塞）向 SSD1306（默认 7 位地址 0x3C，[software_spec_SE_v2.md §5.2.1](software_spec_SE_v2.md)）发送命令与显存数据；
- 实现 SSD1306 初始化序列（具体命令清单、显存写入方式在实施时确定）；
- 提供基础绘图原语与固定测试内容显示（用于 TS01 验证，硬编码内容，不取运行数据）。

### 3.3 修改：`CMakeLists.txt`（[CMakeLists.txt:46-52](../CMakeLists.txt)）

在 `target_sources` 列表中加入 `App/Src/app_display.c` 一行。include 目录无需改动。

### 3.4 修改：`App/Inc/app_display.h` 与 `App/Src/app_display.c` 之外的现有文件——暴露系统状态接口

> 此项虽服务于 TS01"显示所需数据接口"，但改的是 `app_control` 模块。为最小改动、不触碰状态机逻辑：

- **`App/Inc/app_control.h`**：将现有 `SystemState_t` 枚举类型移到头文件中公开（保持单一状态类型，不另建等价类型）；新增只读状态访问函数声明（如 `SystemState_t CTRL_GetSystemState(void);`，签名实施时确定）。
- **`App/Src/app_control.c`**：实现该只读访问函数（返回 `system_state`）；删除 `SystemState_t` 的原内部定义（已移至头文件）。
- **不改**：状态机切换逻辑、500 Hz 调度、温度判断、按键处理等任何现有业务（[second_develop_analysis.md §5.3](second_develop_analysis.md) ——纯只读暴露）。

### 3.5 修改：`Core/Src/main.c`（仅用户代码区）

- `USER CODE BEGIN Includes`（[main.c:30](../Core/Src/main.c)）：增加显示模块头文件包含。
- `USER CODE BEGIN 2`（[main.c:102](../Core/Src/main.c)，`MX_I2C1_Init()` 与 `APP_Control_Init()` 之后）：调用 OLED 初始化 + 固定测试内容显示，用于 TS01 验证。
- 不入非用户代码区；不接入主循环刷新（属 TS02）。

### 3.6 不修改的范围

- `.ioc`（CubeMX 配置已完成）；HAL/CMSIS 底层文件；`i2c.c`/`i2c.h`（CubeMX 生成产物）；`adc.c`/`tim.c`/`dac.c`/`usart.c`/`gpio.c`/`dma.c` 等现有外设初始化；`app_bsp.c`/`app_sampling.c`/`app_process.c`/`app_monitor.c`（TS01 不改其业务实现）；`app_sampling.h`/`app_process.h`/`app_monitor.h`（其访问器已满足，TS01 不改）。

---

## 4. 实施步骤及关系

```
步骤1 ─▶ 步骤2 ─▶ 步骤3 ─▶ 步骤4 ──▶ 步骤5 ──▶ 步骤6
(模块骨架)   (SSD1306  (系统状态   (构建接   (main.c    (编译)
            基础驱动)   只读接口)   入与测试) 接入)
```

**步骤1：新增显示模块骨架并纳入构建**
- 新建 `App/Inc/app_display.h`、`App/Src/app_display.c`（含文件头注释、守卫、`extern "C"`、接口声明，实现为可编译的桩）。
- 在 [CMakeLists.txt:46-52](../CMakeLists.txt) `target_sources` 加入 `App/Src/app_display.c`。
- 目的：建立模块与构建集成，保证后续步骤在可编译骨架上推进。

**步骤2：实现 SSD1306 基础驱动**
- 在 `app_display.c` 中实现：基于 `hi2c1` 的命令/数据发送原语、SSD1306 初始化序列、显存写入与基础绘图能力。
- 内容来源：SSD1306 标准命令集（初始化序列、页地址/列地址设置、显存连续写入），具体命令与参数在实施时确定。
- 目的：使 OLED 能被初始化并显示像素/字符。是步骤5固定测试内容的依赖。

**步骤3：暴露系统状态只读接口**（可与步骤2 并行，无相互依赖）
- 改 [app_control.h](../App/Inc/app_control.h)：暴露 `SystemState_t` 类型 + 新增只读状态访问函数声明。
- 改 [app_control.c](../App/Src/app_control.c)：实现该只读函数（返回 `system_state`），并相应调整枚举定义位置。
- 目的：补齐"显示所需数据接口"中唯一缺失项（[second_develop_analysis.md §3](second_develop_analysis.md)）。TS01 验证不强制调用它（固定测试内容硬编码），但接口须就绪供 TS02。

**步骤4：实现固定测试内容显示**
- 在 `app_display.c` 中实现一个固定测试入口：初始化 OLED 后用硬编码内容（如固定字符串/图案）写入显存并输出到屏幕。
- 目的：不依赖运行数据即可验证驱动与 I2C1 通路。依赖步骤2 的驱动能力。

**步骤5：接入 main.c 用户代码区**
- `USER CODE BEGIN Includes`：包含显示模块头文件。
- `USER CODE BEGIN 2`（`MX_I2C1_Init()` [main.c:101](../Core/Src/main.c) 与 `APP_Control_Init()` [main.c:103 附近] 之后）：调用 OLED 初始化 + 步骤4 的固定测试显示。
- 时序：须在 `MX_I2C1_Init()` 之后（I2C1 已就绪）调用显示初始化；与 `APP_Control_Init()` 的先后关系在实施/验证时确认不影响现有启动行为（TS01 验证范围不含原有功能回归，但接入点位置须避免破坏现有调用顺序）。
- 目的：让步骤2/4 的能力在系统启动时执行，形成可观测验证。依赖步骤1–4。

**步骤6：编译验证**
- 全工程编译，确认无编译/链接错误、新模块正确纳入构建、与现有结构整合通过。
- 目的：进入运行验证前的构建级把关。依赖步骤1–5。

> 实施时每完成一步可按 [CLAUDE.md](../CLAUDE.md) 要求审阅差异后再写入；步骤3 与步骤2 互不依赖，可在步骤1 后并行推进。

---

## 5. 验证方法

### 5.1 工程构建验证

- **编译**：执行工程构建（CMake + 交叉编译链），确认：
  1. `app_display.c/h` 正确加入构建（[CMakeLists.txt](../CMakeLists.txt) 已注册源文件）；
  2. 无编译错误（头文件包含、`extern hi2c1` 引用、HAL I2C API 调用、`app_control` 状态接口改动均通过）；
  3. 无链接错误（新模块符号正确解析，未引入未定义引用）；
  4. 现有 5 个 App 模块与 I2C1 初始化代码仍正常编译（未因接入而被破坏）。
- **产物**：生成可烧录镜像（.elf/.bin/.hex）。

### 5.2 OLED 基本显示验证（需烧录运行）

将镜像烧录至 NUCLEO-G431RB，外接 OLED 模块（按 [software_spec_SE_v2.md §5.2.1](software_spec_SE_v2.md) 连接：VCC-CN6-4/3.3V、GND-CN6-6、SCL-CN5-10/PB8、SDA-CN5-9/PB9），上电后观察：

1. **初始化成功**：OLED 完成初始化（屏幕清空/进入正常显示态，无雪花/全亮异常），表明 SSD1306 初始化序列经 I2C1 正常送达。
2. **固定测试内容显示**：屏幕显示步骤4 预定的固定内容（如固定字符串或图案），位置与形态符合预期，表明命令/数据发送与显存写入通路正确。
3. **I2C 基本通信正常**：固定测试内容能够正确显示且无乱码，表明 I2C1 基本通信与 OLED 驱动/显存写入通路可用（固定内容仅在启动时发送一次，本步骤验证基本通信与显示正常，不涉及持续刷新稳定性）。

### 5.3 TS01 验证边界（不涉及 TS02）

- TS01 验证**只确认 OLED 能被初始化并显示固定内容**；不验证运行数据（温度/DAC 幅值/系统状态）的实时显示与刷新，不验证主循环刷新接入、不验证原有功能回归（均属 TS02）。
- 系统状态只读接口（步骤3）在 TS01 中**可不参与显示验证**（固定测试内容硬编码），仅需保证其编译通过、不破坏现有状态机；其功能正确性可在 TS02 显示系统状态时一并验证。

---

## 6. 附：源码事实核查（本方案所据，均已实际查验）

| 事实 | 位置 |
|---|---|
| `hi2c1` 句柄定义 | [i2c.c:27](../Core/Src/i2c.c) |
| `extern I2C_HandleTypeDef hi2c1;` | [i2c.h:35](../Core/Inc/i2c.h) |
| `MX_I2C1_Init()` 定义（7 位寻址、快速模式 Timing） | [i2c.c:30-49](../Core/Src/i2c.c) |
| `HAL_I2C_MspInit`（PB8/PB9 GPIO 配置） | [i2c.c:73](../Core/Src/i2c.c) |
| `MX_I2C1_Init()` 已在 main() 调用 | [main.c:101](../Core/Src/main.c) |
| `USER CODE BEGIN 2` 位置 | [main.c:102](../Core/Src/main.c) |
| `USER CODE BEGIN Includes` 位置 | [main.c:30](../Core/Src/main.c) |
| `USER CODE BEGIN WHILE`/`BEGIN 3` 位置 | [main.c:107](../Core/Src/main.c) |
| `HAL_I2C_Master_Transmit` 声明 | [stm32g4xx_hal_i2c.h:624](../Drivers/STM32G4xx_HAL_Driver/Inc/stm32g4xx_hal_i2c.h) |
| `HAL_I2C_MODULE_ENABLED` 已启用 | [stm32g4xx_hal_conf.h:50](../Core/Inc/stm32g4xx_hal_conf.h) |
| `SystemState_t` 枚举与 `system_state` 定义（模块内部，无对外接口） | [app_control.c:14-19](../App/Src/app_control.c) |
| `app_control.h` 仅 3 个函数、无状态 getter | [app_control.h:19-21](../App/Inc/app_control.h) |
| 温度/温度有效/DAC 幅值访问器 | [app_sampling.h:19-20](../App/Inc/app_sampling.h)、[app_process.h:20](../App/Inc/app_process.h) |
| App 源文件构建注册 | [CMakeLists.txt:46-52](../CMakeLists.txt) |
| App 头目录已注册 | [CMakeLists.txt:56-58](../CMakeLists.txt) |
