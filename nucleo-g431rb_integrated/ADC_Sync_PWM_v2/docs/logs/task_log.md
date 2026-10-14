# 任务日志

---

## TS01：OLED 显示基础建立

- **执行时间**：2026-08-30
- **任务来源**：[task_list_v2.md](../task_list_v2.md) TS01
- **修改方案**：[programming_plan_TS01.md](../programming_plan_TS01.md)

### 任务目标

建立 OLED 显示模块，完成 SSD1306 基础驱动和显示所需数据接口，并通过固定测试内容验证 OLED 能够正常初始化和显示。本任务不接入系统运行数据的实时显示与刷新（属 TS02）。

### 主要修改

#### 一、新增文件

**1. [App/Inc/app_display.h](../../App/Inc/app_display.h)**

OLED 显示模块头文件。遵循现有 `app_*.h` 风格（`@file`/`@brief` 文件头、`#ifndef` 守卫、`extern "C"`、`/* Public function declarations */` 分节）。对外声明两个接口：

- `void DISP_Init(void)` —— SSD1306 初始化（经 I2C1）；
- `void DISP_ShowTestPattern(void)` —— 固定测试内容显示（TS01 验证用，硬编码内容，不取运行数据）。

**2. [App/Src/app_display.c](../../App/Src/app_display.c)**

显示模块实现，涵盖方案步骤2 与步骤4：

- 命令/数据发送原语：`disp_write_cmd()`、`disp_write_data()`，以 `HAL_I2C_Master_Transmit`（阻塞模式）经 `hi2c1` 发送，地址 `0x3C<<1`（7 位地址左移为 HAL 写地址，对应 [software_spec_SE_v2.md §5.2.1](../software_spec_SE_v2.md)）。
- SSD1306 初始化序列：`DISP_Init()`，按标准命令集配置电荷泵升压、段重映射、COM 扫描方向、COM 引脚、对比度、VCOMH、页寻址模式等。
- 显存写入：本地 GDDRAM 副本 `disp_ram[1024]`（128×64/8），`disp_flush()` 设置列/页范围后整屏写入。
- 基础绘图原语：内置 5×8 ASCII 字模（0x20~0x7E），`disp_draw_char_5x8()`、`disp_draw_string()`。
- 固定测试内容：`DISP_ShowTestPattern()` 清屏后写三行硬编码字符串（`ADC_Sync_PWM` / `OLED OK` / `TS01 Test`）并输出，不读取任何运行数据。

#### 二、修改文件

**3. [CMakeLists.txt](../../CMakeLists.txt)**（方案步骤1）

`target_sources` 列表新增 `App/Src/app_display.c` 一行。include 目录 `App/Inc` 已注册，未改动。

**4. [App/Inc/app_control.h](../../App/Inc/app_control.h)**（方案步骤3）

- 新增 `/* Public types */` 分节，将原 `app_control.c` 内部的 `SystemState_t` 枚举移至头文件公开（保持单一状态类型，未另建等价类型）；
- 新增只读状态访问函数声明 `SystemState_t CTRL_GetSystemState(void);`。

**5. [App/Src/app_control.c](../../App/Src/app_control.c)**（方案步骤3）

- 删除原内部 `SystemState_t` 枚举定义（已移至头文件，留注释说明）；
- 在 `system_state` 私有变量声明之后新增 `CTRL_GetSystemState()` 实现，`return system_state;`。
- 状态机切换逻辑、500 Hz 调度、温度判断、按键处理等现有业务均未改动（纯只读暴露）。

**6. [Core/Src/main.c](../../Core/Src/main.c)**（方案步骤5，仅用户代码区）

- `USER CODE BEGIN Includes`：新增 `#include "app_display.h"`；
- `USER CODE BEGIN 2`：在 `APP_Control_Init()` 之后调用 `DISP_Init();` 与 `DISP_ShowTestPattern();`。接入点位于 `MX_I2C1_Init()`（main.c:101）之后，确保 I2C1 已就绪；未进入非用户代码区，未接入主循环刷新（属 TS02）。

#### 三、未改动范围

- `.ioc` 文件、HAL/CMSIS 底层文件、`i2c.c`/`i2c.h` 等 CubeMX 生成产物均未改；
- `app_sampling`/`app_process`/`app_monitor`/`app_bsp` 模块业务实现与头文件未改（其访问器已满足，TS01 不动）；
- `app_control.c` 状态机逻辑、调度、消抖、温度阈值等均未改，仅新增只读 getter 与类型位置调整；
- main.c `USER CODE BEGIN WHILE` / `BEGIN 3` 主循环未接入刷新（属 TS02）。

### 验证结果

- **编译验证**：未执行（不在本次任务要求范围内）。
- **烧录与板级运行验证**：未执行（按任务约束不进行）。
- **设计符合性**：实施过程未发现实际源码与已确认方案存在冲突，新增/修改内容与 [programming_plan_TS01.md](../programming_plan_TS01.md) 规定的文件、接口、接入点一致。

### 后续验证待办（TS01 完整交接）

以下验证项超出本次任务要求，留待后续在具备构建与硬件环境时执行：

1. 工程构建验证（CMake + 交叉编译链），确认无编译/链接错误、新模块正确纳入构建；
2. 烧录至 NUCLEO-G431RB，外接 OLED（按 [software_spec_SE_v2.md §5.2.1](../software_spec_SE_v2.md) 连接），上电后确认：
   - OLED 完成初始化（无雪花/全亮异常）；
   - 固定测试内容正确显示且无乱码（表明 I2C1 基本通信与 OLED 驱动/显存写入通路可用）。

---

## TS02：OLED 显示功能实现

- **执行时间**：2026-08-30
- **任务来源**：[task_list_v2.md](../task_list_v2.md) TS02
- **修改方案**：[programming_plan_TS02.md](../programming_plan_TS02.md)

### 任务目标

在 TS01 显示基础上，实现系统状态、芯片温度和主要运行数据的显示与刷新，接入现有运行流程，并验证新增 OLED 显示功能正常运行且不影响原有主要功能。

### 设计取值（方案 §0.1 已确认）

- **刷新触发方式与周期**：主循环周期轮询刷新，`HAL_GetTick()` 守时，**100 ms（10 Hz）**；I2C 传输仅在主循环执行，不进任何 ISR。
- **显示字段与版式**：统一版式 + 分状态内容，三行固定布局——`STATE: STOP/RUN/FAULT`、`TEMP : xx.x C`（温度无效时 `--.-`）、`DAC  : xxxx`（仅 RUN，0~4095 原始值）/ `OFF`（STOP/FAULT）。字段均取自现有只读访问器，无需新增数据接口。

### 主要修改

#### 一、修改 app_display.h（[App/Inc/app_display.h](../../App/Inc/app_display.h)）

- @brief 更新为"TS01 基础驱动 + TS02 运行数据显示与刷新"；
- `/* Public function declarations */` 新增 `void DISP_Refresh(void);`（按当前运行数据组屏并整屏刷新一次，供主循环周期调用）；
- `DISP_ShowTestPattern` 声明保留不删（排障备用）。

#### 二、修改 app_display.c（[App/Src/app_display.c](../../App/Src/app_display.c)）

- 文件头 @brief 与依据注释更新，纳入 TS02 说明（刷新周期 100 ms/10 Hz、三行统一版式）；
- 新增 include：`app_control.h`、`app_sampling.h`、`app_process.h`、`<stdio.h>`；
- 新增布局常量（`DISP_ROW_STATE/TEMP/DAC_PAGE`、`DISP_TEMP_BUF_LEN`）；
- 新增私有原型与组屏函数 `disp_build_runtime_frame()`：清屏后按统一版式组三行——`STATE: STOP/RUN/FAULT`、`TEMP : xx.x C`（温度无效显示 `--.-`）、`DAC  : xxxx`（仅 RUN，`%4u` 定宽对齐）/ `OFF`（STOP/FAULT）；字段均取自现有只读访问器，不写任何共享数据；
- 新增对外 `DISP_Refresh()`：组屏 + `disp_flush()` 整屏刷新；
- TS01 驱动原语、字模、初始化序列、`DISP_ShowTestPattern` 实现均保留未改。

#### 三、修改 main.c（[Core/Src/main.c](../../Core/Src/main.c)，仅用户代码区）

- `USER CODE BEGIN PD`：新增 `#define DISP_REFRESH_PERIOD_MS 100U`（100 ms / 10 Hz）；
- `USER CODE BEGIN PV`：新增 `static uint32_t disp_last_tick = 0U;`；
- `USER CODE BEGIN 2`：保留 `DISP_Init()`，将 `DISP_ShowTestPattern()` 替换为 `DISP_Refresh()`（启动首帧真实组屏，系统此时为 STOP）；
- `USER CODE BEGIN 3`：`while(1)` 体内新增 `HAL_GetTick()` 时间差比较的周期轮询，达 100 ms 调用 `DISP_Refresh()`；不阻塞、不 `HAL_Delay`。

#### 四、未改动范围

- `.ioc`、HAL/CMSIS 底层、`i2c.c/h` 等 CubeMX 生成产物未改；
- 现有 5 个 App 模块业务实现与头文件未改（`app_control.c` 状态机/调度/消抖/温度阈值、`app_sampling`/`app_process`/`app_monitor`/`app_bsp`）；TS02 仅读取访问器 `CTRL_GetSystemState()`/`SAMP_GetTemperature()`/`SAMP_IsTemperatureValid()`/`PROC_GetDacAmplitude()`；
- `app_control.h` 的 `SystemState_t` 与 `CTRL_GetSystemState()`（TS01 已完成）未再改；
- TS01 的 SSD1306 驱动原语与初始化序列未改；
- main.c 的 `SystemClock_Config`、外设初始化序列、`Error_Handler` 等非用户代码区未改。

### 验证结果

- **编译验证**：通过（工程正常编译生成镜像）。
- **烧录与板级运行验证**：通过。实测结果：
  - OLED 稳定显示系统状态、芯片温度、DAC 幅值三行信息；
  - STOP、RUN、FAULT 三态切换时，OLED 状态显示均正确；
  - RUN 态旋转 PA1 电位器，OLED 显示的 DAC 幅值可在 0~4095 范围内变化；
  - FAULT 态 OLED 显示符合预期（统一版式：状态/温度/DAC 幅值；显示的是 `PROC_GetDacAmplitude()` 保存值，不代表 DAC 实际输出，该态下 DAC 关闭）；
  - 新增 OLED 显示功能后，原有主要功能运行正常，未发现明显影响。
- **设计符合性**：与 [programming_plan_TS02.md](../programming_plan_TS02.md) 及 [software_spec_SE_v2.md](../software_spec_SE_v2.md)（三态统一显示状态/温度/DAC 幅值）一致。

### 实施后调试修正（板级验证中发现并修正）

下列两项为 TS02 初始实施后，在板级验证中发现问题并据实修正，已纳入最终代码：

1. **SSD1306 寻址模式**：原 `DISP_Init()` 设 `0x02`（页寻址），与 `disp_flush()` 整屏连续写入方式不匹配——症状为 OLED 只显示一行并快速闪烁。改为 `0x00`（水平寻址），列地址到尾自动换页，整屏连续写入正确铺满 8 页，问题解决。
2. **温度浮点格式化失效**：原有效温度分支用 `snprintf("%.1f", ...)`，嵌入式 C 库未启用 `printf` 浮点支持，`temp_buf` 为空，表现为 `TEMP: C`。改为温度乘 10 取整（按符号四舍五入：`temp >= 0` 加 0.5、`temp < 0` 减 0.5）后用 `%lu.%lu` 输出一位小数（如 `34.6`），不依赖浮点 `printf`；无效分支 `--.-` 保留。`DISP_TEMP_BUF_LEN` 由 8U 增至 16U 并删除原注释。

**TS02 通过。**

---

