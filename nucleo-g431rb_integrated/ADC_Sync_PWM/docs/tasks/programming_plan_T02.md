# 编程计划 T02 — 建立模块接口与应用程序框架

> 编制日期：2026-08-24
> 任务依据：
> - `docs/tasks/programming_task_list.md` T02（任务目标、涉及文件、实施要点、验证方式）
> - `docs/design/software_detailed_design.md` §1.4（文件依赖关系）、§1.5.1（main.c 接入）、§2.1（模块公开接口）、§2.3（T06 才定义的枚举与常量）
> - `docs/analysis/initial_project_analysis.md` §2.4（main.c USER CODE 区域）
> 状态：待开发者审阅和批准。本阶段只生成修改计划，不修改任何工程文件。

---

## 1. 任务范围与约束遵循

| 硬性约束 | 本计划的遵循方式 |
|---|---|
| 以任务清单、详细设计、实际工程为依据 | 22 个公开接口均取自详细设计 §2.1 各接口表；main.c 插入位置经实际读取核实（Includes 区第 29 行、USER CODE 2 区第 100 行，`MX_TIM6_Init()` 在第 99 行） |
| 严格限定在 T02 任务范围内 | 仅做三件事：①各 .h 声明公开接口并添加必要包含；②各 .c 建立空实现骨架（安全默认返回值）；③main.c 接入 `APP_Control_Init()`。不实现任何业务逻辑（属 T03 以后），不实现 HAL 回调（T04/T06/T08），不定义 `SystemState_t` 枚举和应用配置常量（T06） |
| 不自行改变已确定的模块职责和公开接口 | 接口清单、参数类型、调用方完全按详细设计 §2.1；6 个接口名按 2026-08-24 审核意见统一（见 §4.1 名称对照），不新增、不减并任何接口 |
| 不修改任何工程文件或设计文件 | 本阶段仅产出本计划文档；正文所列代码均为待批准的修改内容，批准前不落盘 |

## 2. 当前工程现状（实测核实）

- T01 已完成：`App/Inc/`、`App/Src/` 共 10 个骨架文件存在，头文件无包含关系、无声明；CMake 已接入 App 源文件与 include 路径，编译链接验证通过。
- `Core/Src/main.c` 关键区域行号（实测）：`/* USER CODE BEGIN Includes */` 第 29～31 行；`/* USER CODE BEGIN 2 */` 第 100～102 行；`MX_TIM6_Init();` 第 99 行（USER CODE 2 区紧跟其后）。
- **bool 类型可用性**：工程未在任何地方 `#include <stdbool.h>`，G4 HAL 头文件（`stm32g4xx_hal_def.h`）也不定义 bool。详细设计 §2.1 中多处接口使用 `bool` 返回值，因此 T02 在需要 bool 的 App 头文件中添加 `#include <stdbool.h>`（标准头文件，属"按接口需要添加"）。
- 详细设计 §2.1 接口表与任务清单中已出现的函数名：`BSP_StartRealtimeChain`、`BSP_StopRealtimeChain`、`BSP_ToggleLed`、`BSP_StartRegularAdc`、`SAMP_GetTemperature`、`SAMP_IsTemperatureValid`、`PROC_GetDacAmplitude`、`CTRL_NotifyInjectedComplete`、`CTRL_ProcessTemperature`、`APP_Control_Init`、`MON_SendData`。
- 其余接口名称经 2026-08-24 审核统一（6 个改名，见 §4.1 名称对照表），T03 起沿用不再变动。

## 3. 修改文件汇总

| 操作 | 文件 | 说明 |
|---|---|---|
| 修改 | `App/Inc/app_bsp.h` | 添加包含关系；声明 MD-01 全部 13 个公开接口 |
| 修改 | `App/Inc/app_sampling.h` | 添加包含关系；声明 MD-02 全部 2 个公开接口 |
| 修改 | `App/Inc/app_process.h` | 添加包含关系；声明 MD-03 全部 2 个公开接口 |
| 修改 | `App/Inc/app_control.h` | 添加包含关系；声明 MD-04 全部 3 个公开接口 |
| 修改 | `App/Inc/app_monitor.h` | 添加包含关系；声明 MD-05 全部 2 个公开接口 |
| 修改 | `App/Src/app_bsp.c` | 13 个公开函数的空实现（无业务逻辑） |
| 修改 | `App/Src/app_sampling.c` | 2 个公开函数的空实现 |
| 修改 | `App/Src/app_process.c` | 2 个公开函数的空实现 |
| 修改 | `App/Src/app_control.c` | 3 个公开函数的空实现 |
| 修改 | `App/Src/app_monitor.c` | 2 个公开函数的空实现 |
| 修改 | `Core/Src/main.c` | USER CODE Includes 加 `#include "app_control.h"`；USER CODE 2 加 `APP_Control_Init();` |

合计：22 个公开函数声明 + 22 个空实现 + main.c 两处接入。本任务**不实现 HAL 回调**、**不定义** `SystemState_t` 与配置常量。

## 4. 接口清单与命名方案

### 4.1 接口总表（22 个）

| 编号 | 函数原型 | 模块 | 调用方 | 设计依据（详细设计） |
|---|---|---|---|---|
| 1 | `void BSP_StartRealtimeChain(void)` | MD-01 | MD-04 | §2.1.1 实时链路启动（任务清单 T04/T06 出现此名） |
| 2 | `void BSP_StopRealtimeChain(void)` | MD-01 | MD-04 | §2.1.1 实时链路停止（任务清单 T06 出现此名） |
| 3 | `void BSP_WritePwmDutyCh1(uint16_t duty)` | MD-01 | MD-03 | §2.1.1 写 CH1 占空比（审核改名） |
| 4 | `void BSP_WritePwmDutyCh2(uint16_t duty)` | MD-01 | MD-04 | §2.1.1 写 CH2 占空比（审核改名） |
| 5 | `void BSP_WritePwmDutyCh3(uint16_t duty)` | MD-01 | MD-04 | §2.1.1 写 CH3 占空比（审核改名） |
| 6 | `void BSP_WriteDac(uint16_t value)` | MD-01 | MD-04 | §2.1.1 写 DAC 输出 |
| 7 | `void BSP_StartDac(void)` | MD-01 | MD-04 | §2.1.1 启动 DAC 输出 |
| 8 | `void BSP_StopDac(void)` | MD-01 | MD-04 | §2.1.1 停止 DAC 输出（置零并关闭） |
| 9 | `void BSP_ToggleLed(void)` | MD-01 | MD-04 | §2.1.1 翻转 LED（任务清单 T04 出现此名） |
| 10 | `bool BSP_StartDmaTx(uint8_t *data, uint16_t len)` | MD-01 | MD-05 | §2.1.1 启动 DMA 发送（返回启动成功与否）（审核改名） |
| 11 | `void BSP_StartTim6(void)` | MD-01 | MD-04 | §2.1.1 启动 TIM6 |
| 12 | `void BSP_StartRegularAdc(void)` | MD-01 | MD-04 | §2.1.1 启动 ADC 规则组（任务清单 T06 出现此名） |
| 13 | `void BSP_SetLedLow(void)` | MD-01 | MD-04 | §2.1.1 熄灭 LED（审核改名） |
| 14 | `float SAMP_GetTemperature(void)` | MD-02 | MD-04 | §2.1.2 获取芯片温度值 |
| 15 | `bool SAMP_IsTemperatureValid(void)` | MD-02 | MD-04 | §2.1.2 温度有效性查询 |
| 16 | `void PROC_HandleSamples(uint16_t ch1, uint16_t ch2, uint16_t ch3)` | MD-03 | MD-02 | §2.1.3 处理注入组采样值（审核改名） |
| 17 | `uint16_t PROC_GetDacAmplitude(void)` | MD-03 | MD-04 | §2.1.3 获取 DAC 方波幅值（任务清单 T06 出现此名） |
| 18 | `void APP_Control_Init(void)` | MD-04 | main.c | §2.1.4 应用初始化 |
| 19 | `void CTRL_NotifyInjectedComplete(void)` | MD-04 | MD-02 | §2.1.4 注入组完成通知（任务清单 T04 出现此名） |
| 20 | `void CTRL_ProcessTemperature(float temperature)` | MD-04 | MD-02 | §2.1.4 温度处理 |
| 21 | `bool MON_SendData(float value)` | MD-05 | MD-04 | §2.1.5 发送数据（任务清单 T08 出现此名） |
| 22 | `bool MON_IsBusy(void)` | MD-05 | MD-04 | §2.1.5 查询忙状态 |

**接口名称对照表**（2026-08-24 审核统一，T03 起沿用）：

| 计划初稿名称 | 审核后名称 |
|---|---|
| `BSP_WriteCh1Duty` | `BSP_WritePwmDutyCh1` |
| `BSP_WriteCh2Duty` | `BSP_WritePwmDutyCh2` |
| `BSP_WriteCh3Duty` | `BSP_WritePwmDutyCh3` |
| `BSP_StartDmaSend` | `BSP_StartDmaTx` |
| `BSP_SetLedOff` | `BSP_SetLedLow` |
| `PROC_ProcessInjectedSamples` | `PROC_HandleSamples` |

**命名规则说明**：函数名前缀 = 模块职责缩写（BSP/SAMP/PROC/CTRL/MON），与任务清单及详细设计既有拼写一致，并按审核意见与后续任务实际采用的名称统一。T02 固定后不再变动。

### 4.2 包含关系（T02 按当前接口实际需要添加）

依据审核意见（2026-08-24）：T02 只按**当前已声明的接口**建立包含关系，不提前加入后续任务才需要的外设头文件和跨模块头文件。各头文件当前接口所用类型仅 `uint8_t`、`uint16_t`、`float`、`bool`，其中 uint 类型经 `main.h`（`stm32g4xx_hal.h` → `stdint.h`）获得，bool 需 `stdbool.h`。后续任务所需的包含关系（外设句柄头、HAL 回调、跨模块调用头）在对应任务实施时添加。

| 头文件 | T02 新增包含 | 说明 |
|---|---|---|
| app_bsp.h | `main.h`、`stdbool.h` | 13 个接口原型仅用 uint/bool 类型；adc.h/tim.h/dac.h/gpio.h/usart.h 等外设句柄头留待 T03 实现时按需添加 |
| app_sampling.h | `main.h`、`stdbool.h` | 接口原型仅用 float/bool；adc.h 留待 T04/T06 实现 HAL 回调时由 .c 添加 |
| app_process.h | `main.h`、`stdbool.h` | 接口原型仅用 uint/bool 类型；app_bsp.h 留待 T05 真正调用 `BSP_WritePwmDutyCh1()` 时由 app_process.c 添加（2026-08-24 审核修订） |
| app_control.h | `main.h`、`stdbool.h` | 接口原型仅用 float/bool；tim.h 留待 T06 实现回调时由 .c 添加 |
| app_monitor.h | `main.h`、`stdbool.h` | 接口原型仅用 float/bool；usart.h 留待 T08 实现时由 .c 添加 |

**源文件 .c 的包含关系（T02）**：每个 .c 仅包含本模块头文件。跨模块头文件（如 app_process.c 含 app_bsp.h、app_sampling.c 含 app_process.h/app_control.h、app_control.c 含其余 App 头、app_monitor.c 含 app_bsp.h）留待对应任务（T03～T08）实现调用时添加。

### 4.3 空实现安全默认返回值

| 函数 | 空实现返回值 |
|---|---|
| 返回 `bool` 的 5 个函数（BSP_StartDmaTx、SAMP_IsTemperatureValid、MON_SendData、MON_IsBusy） | `false` |
| `SAMP_GetTemperature`（float） | `0.0f` |
| `PROC_GetDacAmplitude`（uint16_t） | `0U` |
| 其余返回 `void` 的 15 个函数 | 空函数体 |

## 5. 具体修改方式

### 5.1 App/Inc/app_bsp.h

修改前（T01 骨架）：无包含关系、无声明。
修改后（新增包含与声明区，其余保持）：

```c
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdbool.h>

/* Public function declarations ----------------------------------------------*/
void BSP_StartRealtimeChain(void);
void BSP_StopRealtimeChain(void);
void BSP_WritePwmDutyCh1(uint16_t duty);
void BSP_WritePwmDutyCh2(uint16_t duty);
void BSP_WritePwmDutyCh3(uint16_t duty);
void BSP_WriteDac(uint16_t value);
void BSP_StartDac(void);
void BSP_StopDac(void);
void BSP_ToggleLed(void);
bool BSP_StartDmaTx(uint8_t *data, uint16_t len);
void BSP_StartTim6(void);
void BSP_StartRegularAdc(void);
void BSP_SetLedLow(void);
```

### 5.2 App/Inc/app_sampling.h

```c
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdbool.h>

/* Public function declarations ----------------------------------------------*/
float SAMP_GetTemperature(void);
bool SAMP_IsTemperatureValid(void);
```

### 5.3 App/Inc/app_process.h

```c
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdbool.h>

/* Public function declarations ----------------------------------------------*/
void PROC_HandleSamples(uint16_t ch1, uint16_t ch2, uint16_t ch3);
uint16_t PROC_GetDacAmplitude(void);
```

### 5.4 App/Inc/app_control.h

```c
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdbool.h>

/* Public function declarations ----------------------------------------------*/
void APP_Control_Init(void);
void CTRL_NotifyInjectedComplete(void);
void CTRL_ProcessTemperature(float temperature);
```

### 5.5 App/Inc/app_monitor.h

```c
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <stdbool.h>

/* Public function declarations ----------------------------------------------*/
bool MON_SendData(float value);
bool MON_IsBusy(void);
```

### 5.6 源文件空实现（5 个 .c）

各 .c 的修改要点（T02 每文件仅包含本模块头文件）：

- **app_bsp.c**：包含 `app_bsp.h`。13 个函数空实现；`BSP_StartDmaTx` 内 `(void)data; (void)len;` 消除未使用参数告警，`return false;`。外设头文件（adc.h/tim.h/dac.h/gpio.h/usart.h）留待 T03 实现时添加。
- **app_sampling.c**：包含 `app_sampling.h`。`SAMP_GetTemperature` 返回 `0.0f`；`SAMP_IsTemperatureValid` 返回 `false`。跨模块头（app_process.h/app_control.h）留待 T04/T06 实现回调调用时添加。
- **app_process.c**：包含 `app_process.h`。`PROC_HandleSamples` 内 `(void)ch1; (void)ch2; (void)ch3;`；`PROC_GetDacAmplitude` 返回 `0U`。app_bsp.h 留待 T05 真正调用 `BSP_WritePwmDutyCh1()` 时由本文件添加。
- **app_control.c**：包含 `app_control.h`。3 个函数空实现；`CTRL_ProcessTemperature` 内 `(void)temperature;`。其余 App 头文件留待 T04 起实现调用时添加。
- **app_monitor.c**：包含 `app_monitor.h`。`MON_SendData` 内 `(void)value;` 返回 `false`；`MON_IsBusy` 返回 `false`。app_bsp.h 留待 T08 实现 DMA 启动时添加。

> 空实现完整代码待批准后按上述内容逐文件写入（与 T01 相同，正文给出修改后全文）。

### 5.7 Core/Src/main.c（两处 USER CODE 区）

**修改 1 — USER CODE BEGIN Includes（第 29～31 行）**：

```c
/* USER CODE BEGIN Includes */
#include "app_control.h"
/* USER CODE END Includes */
```

**修改 2 — USER CODE BEGIN 2（第 100～102 行，位于 `MX_TIM6_Init();` 之后）**：

```c
  /* USER CODE BEGIN 2 */
  APP_Control_Init();
  /* USER CODE END 2 */
```

## 6. 接口与调用关系说明

| 项目 | T02 结论 |
|---|---|
| 新增接口 | 22 个公开函数声明（全部来自详细设计 §2.1，无新增设计外接口） |
| 跨模块调用关系（T02 建立的编译级关系） | 仅 main.c→MD-04（APP_Control_Init）一条。其余跨模块调用（MD-02→MD-03/MD-04、MD-03→MD-01、MD-04→各模块、MD-05→MD-01）随 T03～T08 实现时建立 |
| 运行期行为 | 空实现下 `APP_Control_Init()` 无任何动作，main 循环进入空 while(1)，运行行为与 T01 完全一致 |
| 不实现的 HAL 回调 | `HAL_ADCEx_InjectedConvCpltCallback`、`HAL_ADC_ConvCpltCallback`（T04/T06）、`HAL_TIM_PeriodElapsedCallback`、`HAL_GPIO_EXTI_Callback`（T06）、`HAL_UART_TxCpltCallback`（T08）——本任务不声明、不实现 |
| 不定义的数据 | `SystemState_t` 枚举、配置常量（T06）；DAC 幅值/温度值等私有变量（T05/T06） |

## 7. 验证方法

### 7.1 编译与链接验证

1. 使用 Debug 配置重新生成并构建工程（与 T01 相同的 CMake 构建流程）。
2. **验收标准**：
   - 编译 0 错误 0 警告（未使用参数已用 `(void)` 消除；空函数无告警）；
   - 链接通过，生成 `ADC_Sync_PWM.elf`；
   - 22 个接口在 5 个 .h 中声明、5 个 .c 中定义，链接期如缺定义会报 undefined reference，因此链接通过即证明声明与实现齐备。
3. **附加核对**：确认 main.c 两处修改生效（编译包含 app_control.h 且调用 APP_Control_Init 无未定义报错）。

### 7.2 验证范围说明

- T02 只进行编译和链接验证，**不安排烧录和板级验证**。此时全系统可编译链接，应用功能均为空壳，运行行为与初始工程一致。板级验证从 T04 起按任务清单安排执行。

## 8. 实施顺序建议

1. 按 §5.1～5.5 修改 5 个头文件（包含关系 + 声明）；
2. 按 §5.6 修改 5 个源文件（空实现）；
3. 按 §5.7 修改 main.c 两处 USER CODE 区；
4. 执行 §7.1 编译与链接验证；
5. 记录验证结果，待批准后进入 T03。

## 9. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 函数命名 | 本计划 §4.1 已按审核意见统一 22 个接口名（含 6 个改名，见名称对照表），T03 起不再变动 |
| bool 类型 | 工程不含 stdbool.h，T02 由需用 bool 的 5 个头文件自行包含；若开发者希望统一改用 `uint8_t` 返回值，需在批准前说明 |
| 空实现与后续任务衔接 | 空实现不含任何逻辑，T03～T08 逐个填充；T04 临时代码（直启实时链路、LED 翻转）落在 app_control.c 的既有接口内，无需改动接口 |
| 与设计文档的已知差异 | 任务清单 T03 写"11 个公开接口"而详细设计 §2.1.1 为 13 个（问题 I-01）；本计划按 13 个执行，与详细设计一致，待 T03 前与开发者确认口径 |
