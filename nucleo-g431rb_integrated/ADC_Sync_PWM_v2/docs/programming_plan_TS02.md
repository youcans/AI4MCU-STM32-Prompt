# TS02 代码修改方案：OLED 显示功能实现

---

## 0. 文档说明

本方案为任务 TS02（[见 task_list_v2.md](task_list_v2.md)）的代码修改方案，依据 [software_spec_SE_v2.md](software_spec_SE_v2.md) SF-12 / §4.2、[second_develop_analysis.md](second_develop_analysis.md) §3 / §4，以及 TS01 完成后的当前实际源码制定。

- TS01 已建立 OLED 显示基础（SSD1306 基础驱动 + 系统状态只读 getter），本方案在其上实现运行数据显示、持续刷新与系统接入。
- 本轮**只制定 TS02 方案，不修改任何程序文件**；不修改已完成的 CubeMX 配置、HAL/CMSIS 底层文件、`.ioc`；不重新分析已确认的工程结构、调用链、CubeMX 配置。
- 所引源码位置均经实际核查（本方案末尾附"源码事实核查"）。
- TS02 目标（据 [task_list_v2.md](task_list_v2.md)）：在显示基础上，实现系统状态、芯片温度和主要运行数据的显示与刷新，接入现有运行流程，并验证新增 OLED 显示功能正常运行且不影响原有主要功能。

### 0.1 已确定的设计取值（原标注“详细设计中确定”，此前已确定或本次确认）

可行性分析与软件说明书中以下项目原标注“详细设计中确定”，在此明确取值以使方案可实施：刷新触发方式与刷新周期（主循环周期轮询、100 ms / 10 Hz）此前已确定；显示字段与版式经本次确认。取值均不违背已确认约束。

| 取值项 | 取值内容 | 依据 |
|---|---|---|
| **刷新触发方式与周期** | **主循环周期轮询刷新**：在 `while(1)`（`USER CODE BEGIN 3`）内用 `HAL_GetTick()` 守时，按 **100 ms（10 Hz）** 周期调用一次组屏 + 刷新；I2C 传输仅发生在主循环，不进任何 ISR | 此前已确定；[second_develop_analysis.md §4.1/§4.2](second_develop_analysis.md)——主循环空闲为天然执行点、周期轮询最简、I2C 传输不得进 ISR |
| **显示字段与版式** | **统一版式：状态 + 温度 + DAC 幅值**，三行固定布局：<br>第 1 行 `STATE: STOP / RUN / FAULT`<br>第 2 行 `TEMP : xx.x C`（温度无效时显示 `--.-`）<br>第 3 行 `DAC  : xxxx`（三态均显示 `PROC_GetDacAmplitude()` 返回的 0~4095 原始值，不按状态切换为 `OFF`）<br>**注**：STOP/FAULT 状态下显示的是 `PROC_GetDacAmplitude()` 保存的 DAC 幅值数据，**不代表 DAC 正在实际输出**——此两态下 DAC 仍处于关闭状态（见 [software_spec_SE_v2.md](software_spec_SE_v2.md) SF-06、§4.2：停止/异常状态关闭 DAC 输出）。仅 RUN 态 DAC 实际输出 | 本次确认；[software_spec_SE_v2.md §4.2](software_spec_SE_v2.md)；字段均取自现有访问器，**无需新增任何数据接口** |

> 说明：刷新周期 100 ms / 10 Hz 此前已确定，实施时以 `#define` 定义该值；温度/DAC 字段的具体排版（整数/小数位宽等）按版式在实施时落实。

---

## 1. 任务目标

在 TS01 显示基础上完成三件事：

1. **运行数据显示**：以统一版式，显示系统状态、芯片温度、DAC 幅值（三态均显示 `PROC_GetDacAmplitude()` 返回值），取自现有只读访问器；
2. **持续刷新**：在主循环以固定周期轮询刷新 OLED，I2C 传输仅在主循环执行，不进 ISR；
3. **系统接入**：将刷新调用接入 `main.c` 用户代码区主循环，并把 TS01 的固定测试内容从启动流程替换为真实组屏刷新（保留初始化）。

并验证：新增 OLED 显示功能正常运行，原有主要功能（ADC 采样、PWM、DAC、串口通信、状态切换、温度异常处理、LED 指示）不受影响。

---

## 2. 现有条件（已完成、可复用，不需重新建立）

### 2.1 SSD1306 基础驱动与显存机制已就绪（TS01 产物）

显示模块 [app_display.c](../App/Src/app_display.c) 已具备：

- 本地显存 `disp_ram[1024]`（128×64/8）、`disp_clear_ram()` 清空、`disp_flush()` 整屏写入 SSD1306（[app_display.c:49](../App/Src/app_display.c)、[:108](../App/Src/app_display.c)、[:101](../App/Src/app_display.c)）；
- I2C 阻塞发送原语 `disp_write_cmd()` / `disp_write_data()`（经 `hi2c1`，`HAL_I2C_Master_Transmit`，地址 `0x3C<<1`，[app_display.c:63](../App/Src/app_display.c)、[:72](../App/Src/app_display.c)）；
- 5×8 ASCII 字模 + 绘图原语 `disp_draw_string(col, page, str)`（[app_display.c:176](../App/Src/app_display.c) 字模、[:298](../App/Src/app_display.c) 绘字符串）；
- `DISP_Init()`（SSD1306 初始化，含清屏 + 开显示，[app_display.c:114](../App/Src/app_display.c)）。

这些原语目前为模块私有（`static`）。TS02 需在其上新增**组帧/刷新对外接口**；驱动层与显存机制可直接复用，无需重建。

### 2.2 显示所需数据访问器全部就位（含 TS01 新增的 getter）

| 显示字段 | 访问器 | 位置 | 说明 |
|---|---|---|---|
| 系统状态 | `CTRL_GetSystemState()` 返回 `SystemState_t`（`SYSTEM_STATE_STOP/RUN/FAULT`） | [app_control.h:30](../App/Inc/app_control.h) / [app_control.c:31](../App/Src/app_control.c) | TS01 新增只读 getter，TS02 直接复用，**不改** |
| 芯片温度 | `SAMP_GetTemperature()` 返回 `float`（℃） | [app_sampling.h:19](../App/Inc/app_sampling.h) | 现有，不改 |
| 温度是否有效 | `SAMP_IsTemperatureValid()` 返回 `bool` | [app_sampling.h:20](../App/Inc/app_sampling.h) | 用于温度无效时显示 `--.-` |
| DAC 幅值 | `PROC_GetDacAmplitude()` 返回 `uint16_t`（0~4095 原始整数） | [app_process.h:20](../App/Inc/app_process.h) | 三态均显示；返回值非换算电压（[second_develop_analysis.md §3](second_develop_analysis.md)） |

**结论**：TS02 的全部显示字段均有现成只读访问器，**无需新增任何跨模块数据接口**。与可行性分析"温度/DAC 幅值/系统状态均有现成访问器（系统状态 getter 由 T01 已补齐）"一致。TS01 的固定测试验证不依赖运行数据，TS02 才实际调用这些访问器组屏。

### 2.3 主循环刷新执行点已就绪

- [main.c:112](../Core/Src/main.c) `while(1)`，[main.c:116](../Core/Src/main.c) `USER CODE BEGIN 3`（循环体内，目前为空）——TS02 在此接入周期轮询刷新。
- `HAL_GetTick()` 可用（SysTick，[main.c:81 HAL_Init](../Core/Src/main.c) 已初始化）——用于主循环守时，无需新硬件资源。
- `DISP_Init()` 已在 `USER CODE BEGIN 2`（[main.c:106](../Core/Src/main.c)，`MX_I2C1_Init()` 与 `APP_Control_Init()` 之后）调用，启动时 OLED 已初始化——TS02 保留初始化，把其后的 `DISP_ShowTestPattern()` 替换为首次组屏刷新。

### 2.4 风险与一致性边界（据 [second_develop_analysis.md §4.4/§6](second_develop_analysis.md)，TS02 须遵守）

- **I2C 传输不得进 ISR**：刷新的 I2C 传输只放主循环。本方案采用阻塞 `HAL_I2C_Master_Transmit`（与 TS01 一致），不引入新 I2C 中断，对现有 NVIC 优先级体系零影响，满足 [FR-10](software_spec_SE_v2.md)。
- **显示数据为只读旁路**：组屏时连续读取 `system_state`/温度/DAC 幅值，各次读取间可能被 ISR 更新，**一帧内容未必是同一时刻完整快照**（如状态已切至 RUN 而 DAC 幅值仍为旧值）。下一次刷新即反映更新后的值，对慢速人眼显示不构成实质风险，亦不改变现有实时链路并发竞态性质。
- **不写入现有共享数据**：TS02 仅读取上述访问器，不写 `system_state`/`dac_amplitude`/`chip_temperature` 等任何现有数据。

---

## 3. 需要新增或修改的文件、接口及其职责

### 3.1 修改：`App/Inc/app_display.h`（[app_display.h](../App/Inc/app_display.h)）

新增对外刷新接口声明（与 TS01 已有 `DISP_Init` 同分节）：

- `void DISP_Refresh(void)` —— 按当前运行数据组屏并整屏刷新一次（读访问器 → 写本地显存 → `disp_flush`）。供主循环周期调用。

接口签名固定为无参 `void`（数据由其内部经各模块 getter 获取）。`DISP_ShowTestPattern` 是否保留在头文件：**保留声明不删**（TS01 验证入口，留作后续排障可调用，不影响 TS02；若实施时为精简而删除，属等价行为，不违背方案）。

需在头文件 `Includes` 区新增对 `app_control.h`（`SystemState_t`）的间接可见性——`app_display.h` 已 `#include "main.h"`，`SystemState_t` 由 `CTRL_GetSystemState()` 返回类型带入；实际组屏逻辑在 `.c` 中 `#include "app_control.h"`/`"app_sampling.h"`/`"app_process.h"` 即可，头文件无需引入这三者（保持头文件轻量）。

### 3.2 修改：`App/Src/app_display.c`（[app_display.c](../App/Src/app_display.c)）

核心实现文件。新增内容：

1. **新增 include**：`app_control.h`、`app_sampling.h`、`app_process.h`、`<stdio.h>`（用于格式化数值字符串，量级可控）。
2. **新增本地格式化辅助**（私有）：将数值格式化为定宽字符串供 `disp_draw_string` 显示。例如温度 `xx.x`（1 位整数+1 位小数，或据实际范围调整位数）、DAC 幅值 `xxxx`（4 位定宽，前置空格补位对齐）。具体实现（`snprintf` 或手工拼接）在实施时确定，遵循"不添加无关复杂结构"。
3. **新增组屏函数**（私有，如 `disp_build_runtime_frame(void)`）：
   - `disp_clear_ram()`；
   - 读取 `CTRL_GetSystemState()` → 第 1 行 `STATE: STOP/RUN/FAULT`；
   - 读取 `SAMP_IsTemperatureValid()`：有效则读 `SAMP_GetTemperature()` 格式化为 `xx.x`，无效则 `--.-` → 第 2 行 `TEMP : xx.x C` / `TEMP : --.- C`；
   - 三态均读 `PROC_GetDacAmplitude()` 格式化 `xxxx`（定宽对齐）→ 第 3 行 `DAC  : xxxx`，不按状态切换为 `OFF`；
   - 字段均为**读取当下值**（只读旁路，不持存、不写共享数据）。
4. **新增对外刷新函数** `void DISP_Refresh(void)`：调用 `disp_build_runtime_frame()` + `disp_flush()`。整屏刷新（每次 1024 字节，I2C 快速模式量级约数十毫秒，主循环可承受）。
5. **TS01 固定测试入口** `DISP_ShowTestPattern()`：保留实现不删（留作排障备用）。

> 不改 TS01 已有的驱动原语、字模、初始化序列（`disp_write_*`/`disp_flush`/`disp_draw_*`/`DISP_Init` 保持不变，组屏在其上调用）。

### 3.3 修改：`Core/Src/main.c`（[main.c](../Core/Src/main.c)，仅用户代码区）

1. **`USER CODE BEGIN 2`**（[main.c:103](../Core/Src/main.c)）：
   - 保留 `DISP_Init();`（OLED 初始化）；
   - **删除** `DISP_ShowTestPattern();`（[main.c:107](../Core/Src/main.c)）调用——TS01 固定测试内容让位给 TS02 真实刷新。改为初始化后立即调用一次 `DISP_Refresh();`，使启动后屏幕即显示真实运行态（系统此时为 STOP，屏幕显示 `STATE: STOP` / 温度 / `DAC: xxxx`），避免出现"先固定测试内容、后切换为真实内容"的中间态。
2. **`USER CODE BEGIN 3`**（[main.c:116](../Core/Src/main.c)，`while(1)` 体内）：新增周期轮询刷新：
   - 用 `HAL_GetTick()` 守时，按 **100 ms（10 Hz）** 周期调用 `DISP_Refresh();`；
   - 周期判定用“时间差比较”（`now - last >= 周期`），不阻塞、不 `HAL_Delay`（保持主循环可被中断正常抢占）；
   - 周期值用 `#define` 常量定义（放 `USER CODE BEGIN PD` 或显示模块内，实施时定）。

不进入非用户代码区；不改 `SystemClock_Config`、外设初始化序列、`Error_Handler` 等。

### 3.4 不修改的范围

- `.ioc`、HAL/CMSIS 底层文件、`i2c.c`/`i2c.h` 等 CubeMX 生成产物；
- 现有 5 个 App 模块的业务实现与头文件（`app_control.c` 状态机/调度/消抖/温度阈值、`app_sampling.c`、`app_process.c`、`app_monitor.c`、`app_bsp.c`）——TS02 仅**读取**其访问器；
- `app_control.h` 已公开的 `SystemState_t` 与 `CTRL_GetSystemState()`（TS01 已完成，TS02 不再改）；
- `app_sampling.h`/`app_process.h`/`app_monitor.h`（访问器已满足，不改）；
- TS01 已建立的 SSD1306 驱动原语与初始化序列（组屏在其上调用，不重建）。

---

## 4. 实施步骤及关系

```
步骤1 ─▶ 步骤2 ─▶ 步骤3 ─▶ 步骤4（编译验证，按要求不执行，留待后续）
(组屏刷新  (main.c    (TS01 测试
 接口)     接入)     退场)
```

**步骤1：实现组屏与刷新接口**（在 [app_display.c](../App/Src/app_display.c)）
- 新增模块 include（`app_control.h`/`app_sampling.h`/`app_process.h`/`<stdio.h>`）；
- 实现数值格式化辅助、`disp_build_runtime_frame()` 组屏（按 §3.2 四步内容组三行）、`DISP_Refresh()`（组屏 + `disp_flush`）；
- 在 [app_display.h](../App/Inc/app_display.h) 声明 `DISP_Refresh(void)`。
- 目的：提供运行数据显示与整屏刷新能力。依赖已有访问器与 TS01 驱动原语。

**步骤2：接入 main.c 用户代码区**（[main.c](../Core/Src/main.c)）
- `USER CODE BEGIN 2`：保留 `DISP_Init()`，把 `DISP_ShowTestPattern()` 换为一次 `DISP_Refresh()`；
- `USER CODE BEGIN 3`：新增 `HAL_GetTick()` 周期轮询（100 ms / 10 Hz）调用 `DISP_Refresh()`；
- 时序：`DISP_Init` 须在 `MX_I2C1_Init()`（[main.c:102](../Core/Src/main.c)）之后（已满足）；`DISP_Refresh` 须在 `APP_Control_Init()`（[main.c:104](../Core/Src/main.c)）之后（系统状态/温度采样已就绪——`APP_Control_Init` 内已 `BSP_StartRegularAdc`/`BSP_StartTim6`，温度将随后产生；启动首次刷新时温度可能暂无效，组屏按 `--.-` 处理，后续刷新即正常显示，不属错误）。
- 目的：让步骤1 能力在启动与运行中持续执行。依赖步骤1。

**步骤3：TS01 固定测试显示退场**
- 步骤2 已将 `DISP_ShowTestPattern()` 调用从 main.c 移除；`app_display.c`/`.h` 中的 `DISP_ShowTestPattern` 实现与声明保留不删（排障备用）。
- 目的：以真实运行数据显示取代 TS01 测试内容，完成验证视角切换。包含于步骤2（移除调用），列此仅为明确该退场动作。

**步骤4：编译验证**（按本次任务要求，编译不属要求范围，此步骤标注为留待后续）
- 全工程编译，确认无编译/链接错误、组屏函数与接入通过、现有模块未受影响。
- 此步骤在具备构建环境时执行；本方案阶段不执行编译、不烧录、不板级运行。

> 步骤1、2 顺序固定（先实现后接入）；步骤3 随步骤2 完成；步骤4 留待后续。每步完成可按 [CLAUDE.md](../CLAUDE.md) 审阅差异后再写入。

---

## 5. 验证方法

### 5.1 工程构建验证（留待后续具备构建环境时执行；本次不属要求范围）

- **编译**：执行工程构建（CMake + 交叉编译链），确认：
  1. `DISP_Refresh` 组屏函数正确编译（访问器调用、`SystemState_t` 判别、数值格式化、`disp_draw_string`/`disp_flush` 调用均通过）；
  2. 无链接错误（新符号正确解析，`snprintf` 等若用到正确链接）；
  3. 现有模块（含 TS01 代码）仍正常编译（未因接入而被破坏）。
- **产物**：生成可烧录镜像（.elf/.bin/.hex）。

### 5.2 OLED 显示功能验证（需烧录运行，留待后续）

将镜像烧录至 NUCLEO-G431RB，外接 OLED（按 [software_spec_SE_v2.md §5.2.1](software_spec_SE_v2.md) 连接），验证：

1. **启动即显示真实内容**：上电后屏幕显示当前运行态（STOP）内容——`STATE: STOP`、温度（冷启动可能短暂 `--.-`，温度有效后显示数值）、`DAC: xxxx`（当前 DAC 幅值访问器返回值），无 TS01 旧固定测试内容残留。
2. **三态显示正确**（统一版式，三态均显示 `状态/温度/DAC 幅值`）：
   - **STOP**：`STATE: STOP` / `TEMP: xx.x C`（无效时 `--.-`）/ `DAC: xxxx`；
   - 按 B1 进入 **RUN**：`STATE: RUN` / 温度 / `DAC: xxxx`（随电位器变化的 0~4095 原始值）；
   - 触发 **FAULT**（升温超阈值）：`STATE: FAULT` / 温度 / `DAC: xxxx`；温度回落至恢复阈值后自动返回 STOP 并对应显示。
3. **持续刷新**：温度与 DAC 幅值随运行按刷新周期持续更新，无冻结、无乱码、无闪烁异常。
4. **显示滞后说明**：组屏连续读取多共享变量、期间可能被 ISR 更新，一帧内容偶发非同一时刻完整快照（如状态切至 RUN 首帧仍为 STOP 的 DAC 值）；下一刷新即修正，属预期、不构成错误（见 [second_develop_analysis.md §4.4](second_develop_analysis.md)）。

### 5.3 原有主要功能不受影响验证（需烧录运行，留待后续）

确认接入刷新调用后原有功能保持正常（对应 [task_list_v2.md](task_list_v2.md) TS02 验证"确认原有主要功能运行正常"）：

- **ADC 采样 / PWM**：RUN 态三组 PWM（10 kHz，CH1 动态、CH2/3 预设）正常输出与更新；
- **DAC**：RUN 态 250 Hz 方波正常；
- **串口通信**：VOFA+ 正常接收 JustFloat 数据（RUN 发 DAC 值、STOP 发温度、FAULT 不发）；
- **状态切换**：按键 STOP↔RUN、温度触发 FAULT 进入与恢复正常；
- **LED 指示**：STOP 熄灭、RUN 翻转~常亮、FAULT 10 Hz 闪烁；
- **刷新影响判定**：因 I2C 阻塞传输仅发生在主循环、不进 ISR，且主循环空闲，上述实时链路的周期与响应不应受影响；若实测发现某实时功能受影响，应回查刷新周期或单次刷新耗时（分页刷新等属后续优化，不在 TS02 必要范围）。

### 5.4 TS02 验证边界

- TS02 完成后 OLED 显示功能应为完整可用（三态显示 + 持续刷新 + 系统接入）；
- 刷新周期为 100 ms / 10 Hz（此前已确定），以 `#define` 定义；若实测刷新对实时功能有影响，可后续做分页刷新等优化，不属本任务必要范围；
- 字段版式与三态内容已确认（§0.1），实施时据此落实，不再变更版式结构。

---

## 6. 附：源码事实核查（本方案所据，均已实际查验）

| 事实 | 位置 |
|---|---|
| 本地显存 `disp_ram[1024]` | [app_display.c:49](../App/Src/app_display.c) |
| `disp_clear_ram()` / `disp_flush()` | [app_display.c:108](../App/Src/app_display.c) / [:101](../App/Src/app_display.c) |
| I2C 发送原语 `disp_write_cmd` / `disp_write_data`（阻塞、`hi2c1`、`0x3C<<1`） | [app_display.c:63](../App/Src/app_display.c) / [:72](../App/Src/app_display.c) |
| 字模 `font5x8` 与 `disp_draw_string(col, page, str)` | [app_display.c:176](../App/Src/app_display.c) / [:298](../App/Src/app_display.c) |
| `DISP_Init()` / `DISP_ShowTestPattern()` | [app_display.c:114](../App/Src/app_display.c) / [:314](../App/Src/app_display.c) |
| `SystemState_t` 枚举（STOP/RUN/FAULT）已公开 | [app_control.h:20-24](../App/Inc/app_control.h) |
| `CTRL_GetSystemState()` 声明 / 实现 | [app_control.h:30](../App/Inc/app_control.h) / [app_control.c:31](../App/Src/app_control.c) |
| `SAMP_GetTemperature()` / `SAMP_IsTemperatureValid()` | [app_sampling.h:19](../App/Inc/app_sampling.h) / [:20](../App/Inc/app_sampling.h) |
| `PROC_GetDacAmplitude()`（uint16_t，0~4095 原始） | [app_process.h:20](../App/Inc/app_process.h) |
| main.c `USER CODE BEGIN Includes` 已含 `app_display.h` | [main.c:32](../Core/Src/main.c) |
| main.c `USER CODE BEGIN 2`（`APP_Control_Init` 后接 `DISP_Init`/`DISP_ShowTestPattern`） | [main.c:103-107](../Core/Src/main.c) |
| main.c `while(1)` 与 `USER CODE BEGIN 3`（空） | [main.c:112](../Core/Src/main.c) / [:116](../Core/Src/main.c) |
| `HAL_Init()`（SysTick/HAL_GetTick 可用） | [main.c:81](../Core/Src/main.c) |
| `MX_I2C1_Init()` 在 `APP_Control_Init` 之前 | [main.c:102](../Core/Src/main.c) |
| 主循环为空（刷新接入点） | [main.c:112-117](../Core/Src/main.c) |
| 刷新执行位置/I2C 不入 ISR/数据只读旁路/非同快照 | [second_develop_analysis.md §4.1/§4.2/§4.4/§6](second_develop_analysis.md) |
