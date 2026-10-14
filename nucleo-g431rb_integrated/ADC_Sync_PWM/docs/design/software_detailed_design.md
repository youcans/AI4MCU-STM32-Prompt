# 软件详细设计

> 编制日期：2026-08-08
> 修订日期：2026-08-08（根据审阅意见修订）
> 依据文件：
> - `docs/design/module_design.md`（软件模块设计）
> - `docs/design/module_relationship_analysis.md`（模块关系与数据流分析）
> - `docs/analysis/initial_project_analysis.md`（初始工程结构分析）
> 说明：本文档将已确认的软件模块落实到具体程序文件。本次仅编写"模块与文件组织设计"部分，不展开函数接口、数据结构、状态机和调度策略。

---

## 1. 模块与文件组织设计

### 1.1 应用程序目录结构

在现有 CubeMX 工程基础上，新增 `App/` 目录存放全部应用模块代码：

```
ADC_Sync_PWM/
├── Core/                          # CubeMX 生成（不修改非用户区域）
│   ├── Inc/
│   │   ├── main.h
│   │   ├── adc.h / dac.h / dma.h / gpio.h / tim.h / usart.h
│   │   ├── stm32g4xx_hal_conf.h
│   │   └── stm32g4xx_it.h
│   └── Src/
│       ├── main.c                 # ← 在 USER CODE 区添加模块初始化调用
│       ├── stm32g4xx_it.c         # 中断入口（不修改，HAL 回调在 App 中实现）
│       ├── adc.c / dac.c / dma.c / gpio.c / tim.c / usart.c
│       └── ...
├── App/                           # 【新增】应用程序模块
│   ├── Inc/
│   │   ├── app_bsp.h              # MD-01 板级支持模块
│   │   ├── app_sampling.h         # MD-02 采样管理模块
│   │   ├── app_process.h          # MD-03 实时处理模块
│   │   ├── app_control.h          # MD-04 系统控制模块
│   │   └── app_monitor.h          # MD-05 串口监测模块
│   └── Src/
│       ├── app_bsp.c              # MD-01
│       ├── app_sampling.c         # MD-02（含 ADC HAL 回调）
│       ├── app_process.c          # MD-03
│       ├── app_control.c          # MD-04（含 TIM6/EXTI HAL 回调）
│       └── app_monitor.c          # MD-05（含 DMA 发送完成回调）
├── Drivers/                       # HAL 库（不修改）
├── cmake/                         # 构建配置（不修改）
└── CMakeLists.txt                 # 顶层 CMake（添加 App 源文件和路径）
```

### 1.2 模块与文件对应关系

| 模块编号 | 模块名称 | 头文件 | 源文件 | 关联功能 |
|---|---|---|---|---|
| MD-01 | 板级支持模块 | `App/Inc/app_bsp.h` | `App/Src/app_bsp.c` | 外设操作原语：TIM1/PWM 启停、ADC 注入组启停、DAC 写入、GPIO 翻转、DMA 发送启动 |
| MD-02 | 采样管理模块 | `App/Inc/app_sampling.h` | `App/Src/app_sampling.c` | ADC 转换完成事件处理、采样值读取与分发、芯片温度换算与持有 |
| MD-03 | 实时处理模块 | `App/Inc/app_process.h` | `App/Src/app_process.c` | PWM 占空比计算、DAC 方波幅值计算与暂存 |
| MD-04 | 系统控制模块 | `App/Inc/app_control.h` | `App/Src/app_control.c` | 系统状态管理、温度异常判断、DAC 翻转、LED 指示、500 Hz 周期调度、初始化调度 |
| MD-05 | 串口监测模块 | `App/Inc/app_monitor.h` | `App/Src/app_monitor.c` | JustFloat 组帧、DMA 发送、发送忙状态管理 |

### 1.3 各文件职责

#### 1.3.1 app_bsp.h / app_bsp.c（MD-01 板级支持模块）

**职责**：
- 为上层模块提供外设基础操作接口：定时器启停与比较值写入、ADC 子流程启停、DAC 输出、GPIO 翻转、DMA 发送启动等。
- 统一执行 ADC 注入组、TIM1 和 PWM 的底层启停，封装实时链路启停时序：启动时先使能 ADC 注入组再启动 TIM1 和 PWM 输出；停止时先停止 TIM1 触发源再停止注入组采样并关闭 PWM 输出。
- 外设初始化（`MX_*_Init`）已由 `main.c` 完成，MD-01 负责后续运行阶段的板级操作和外设启停。

**职责边界**：
- 只提供外设操作原语，不包含业务逻辑、状态判断。
- 不持有任何业务数据，不访问 `SystemState`。
- 仅执行上层模块发出的启停命令，不决定操作时机。

**源文件内部主要组成**：

| 组成部分 | 内容 |
|---|---|
| 外部依赖 | CubeMX 外设头文件（`adc.h`、`tim.h`、`dac.h`、`gpio.h`、`usart.h`），以及 `main.h` |
| 公开函数 | 定时器启停与 CCR 写入、ADC 注入组启停、DAC 输出与启停、GPIO 翻转与电平设置、DMA 发送启动 |
| 私有函数 | 无（纯封装层） |
| 私有变量 | 无（不持有业务数据） |

#### 1.3.2 app_sampling.h / app_sampling.c（MD-02 采样管理模块）

**职责**：
- 实现 HAL ADC 回调函数，处理 ADC1 转换完成事件。
- 注入组序列完成后：通过 HAL API 读取三路注入组转换结果，传递给 MD-03，并通知 MD-04。
- 规则组转换完成后：通过 HAL API 读取规则组温度转换结果，换算为芯片温度值，传递给 MD-04 进行温度异常判断。
- 持有芯片温度值（对外提供只读访问），并维护温度值有效性标志。

**职责边界**：
- 负责 ADC 转换完成事件处理和采样值读取，ADC1 底层初始化和运行启停由 MD-01 负责。
- 不执行 PWM 占空比计算和 DAC 值计算（属于 MD-03）。
- 不执行温度异常判断和状态切换（属于 MD-04）。
- 不直接操作 TIM1、DAC1、LPUART1 等外设寄存器。

**源文件内部主要组成**：

| 组成部分 | 内容 |
|---|---|
| 外部依赖 | CubeMX `adc.h`，App 模块 `app_process.h`、`app_control.h`，以及 `main.h` |
| 公开函数 | 获取芯片温度值（供 MD-04 在停止状态串口发送时调用） |
| HAL 回调 | `HAL_ADCEx_InjectedConvCpltCallback` — 注入组 JEOS 处理入口；`HAL_ADC_ConvCpltCallback` — 规则组 EOC 处理入口 |
| 私有函数 | ADC 原始值换算为温度值 |
| 私有变量 | 芯片温度值（持存）、温度值有效标志 |

#### 1.3.3 app_process.h / app_process.c（MD-03 实时处理模块）

**职责**：
- 接收 MD-02 传入的注入组采样值，计算第一组 PWM 占空比和 DAC 方波幅值。
- 计算完成后通过 MD-01 更新 TIM1 比较寄存器（CCR1）。
- DAC 方波幅值由本模块暂存（10 kHz 更新频率），对外提供只读访问，供 MD-04 在 500 Hz 时读取。

**职责边界**：
- 纯计算模块，不接触任何外设寄存器（CCR1 写入通过 MD-01 执行）。
- 不判断当前是否处于运行状态（由调用上下文保证）。
- 不感知 DAC 的实际输出时机（DAC 寄存器写入由 MD-04 在 500 Hz 时完成）。
- 当前仅动态模拟电压（PA1）参与计算，高/低电平参考信号预留不参与动态计算。

**源文件内部主要组成**：

| 组成部分 | 内容 |
|---|---|
| 外部依赖 | `app_bsp.h`、`main.h` |
| 公开函数 | 接收采样值并执行计算处理；获取 DAC 方波幅值（供 MD-04 读取） |
| HAL 回调 | 无 |
| 私有函数 | PWM 占空比换算、DAC 方波幅值换算 |
| 私有变量 | DAC 方波幅值（持存） |

#### 1.3.4 app_control.h / app_control.c（MD-04 系统控制模块）

**职责**：
- 独占 `SystemState` 修改权，实现停止/运行/异常三状态转换。
- 接收 MD-02 传入的芯片温度值，与阈值比较，产生温度异常进入/退出事件。
- 根据状态变化向 MD-01 发出实时链路启停命令和 DAC 启停命令。
- 在 500 Hz 时基事件中执行：DAC 翻转（形成 250 Hz 方波）、LED 分频计数（异常状态 10 Hz 闪烁）、串口数据源选择并触发 MD-05 发送。
- 运行状态每次注入组完成时翻转 LED。
- 上电后执行初始化调度，设定初始状态为停止。

**职责边界**：
- `SystemState` 仅在 MD-04 内部使用，其他模块不读取。
- 不参与 ADC 采样数据读取和换算（属于 MD-02）。
- 不参与 PWM 占空比和 DAC 值的计算（属于 MD-03）。

**源文件内部主要组成**：

| 组成部分 | 内容 |
|---|---|
| 外部依赖 | CubeMX `tim.h`，App 模块 `app_bsp.h`、`app_sampling.h`、`app_process.h`、`app_monitor.h`，以及 `main.h` |
| 公开函数 | 应用初始化入口（在 `main.c` 中调用）；注入组完成通知（MD-02 调用，用于运行状态 LED 翻转） |
| HAL 回调 | `HAL_TIM_PeriodElapsedCallback` — TIM6 500 Hz 调度入口；`HAL_GPIO_EXTI_Callback` — 按键事件入口 |
| 私有函数 | 初始化调度、状态机转换函数、温度异常判断、500 Hz 周期调度子逻辑（DAC 翻转、LED 分频、串口触发） |
| 私有变量 | 系统状态变量、LED 分频计数器、DAC 翻转标志 |

#### 1.3.5 app_monitor.h / app_monitor.c（MD-05 串口监测模块）

**职责**：
- 在 500 Hz 时基事件中由 MD-04 触发，接收 MD-04 已选定的数据（DAC 输出值或芯片温度值）。
- 完成 JustFloat 组帧，通过 MD-01 启动 DMA 发送。
- 维护 DMA 发送忙状态：发送中拒绝重复启动，DMA 传输完成后清除忙标志。

**职责边界**：
- 不读取 `SystemState`，不从 MD-02 或 MD-03 直接读取业务数据（所有数据由 MD-04 选定后传入）。
- 仅负责 JustFloat 组帧和 DMA 发送，不参与 ADC 采样、PWM 控制、DAC 输出等实时控制路径。

**源文件内部主要组成**：

| 组成部分 | 内容 |
|---|---|
| 外部依赖 | CubeMX `usart.h`，App 模块 `app_bsp.h`，以及 `main.h` |
| 公开函数 | 接收数据并组帧启动发送；查询 DMA 发送忙状态（供 MD-04 判断） |
| HAL 回调 | `HAL_UART_TxCpltCallback` — DMA 发送完成处理 |
| 私有函数 | JustFloat 组帧 |
| 私有变量 | DMA 发送忙标志、发送帧缓冲区 |

---

### 1.4 文件依赖关系

```
   App/Inc/                         App/Src/
  ─────────                        ─────────

  app_bsp.h ──────────────────► app_bsp.c
    ▲                        (includes: main.h, adc.h, tim.h, dac.h, gpio.h, usart.h)
    │
    ├── app_process.h ────────► app_process.c
    │     ▲                   (includes: main.h, app_bsp.h)
    │     │
    ├── app_sampling.h ───────► app_sampling.c
    │     ▲                   (includes: main.h, adc.h, app_process.h, app_control.h)
    │     │
    ├── app_monitor.h ────────► app_monitor.c
    │     ▲                   (includes: main.h, usart.h, app_bsp.h)
    │     │
    └── app_control.h ────────► app_control.c
          ▲                   (includes: main.h, tim.h,
          │                     app_bsp.h, app_sampling.h,
          │                     app_process.h, app_monitor.h)
          │
  Core/Inc/
  ─────────
  main.h ◄────────────────── 所有 App 源文件均包含
  adc.h ◄──────────────────── app_bsp.c, app_sampling.c
  tim.h ◄──────────────────── app_bsp.c, app_control.c
  dac.h ◄──────────────────── app_bsp.c
  gpio.h ◄─────────────────── app_bsp.c
  usart.h ◄────────────────── app_bsp.c, app_monitor.c
```

**依赖说明**：
- 所有 App 模块均包含 `main.h`（获得 HAL 句柄和 GPIO 引脚宏定义）。
- MD-01（`app_bsp.c`）包含全部外设头文件，对外设寄存器的写操作提供统一封装。
- MD-02（`app_sampling.c`）直接包含 `adc.h` 以读取 ADC 转换结果，包含 `app_process.h` 和 `app_control.h` 以分发数据。
- MD-03（`app_process.c`）仅包含 `app_bsp.h`，通过 MD-01 写入 CCR1。
- MD-04（`app_control.c`）包含所有 App 模块头文件，协调全系统工作。
- MD-05（`app_monitor.c`）包含 `app_bsp.h`，通过 MD-01 启动 DMA。

### 1.5 与 CubeMX 生成文件的关系

#### 1.5.1 main.c

`main.c` 中只在 `USER CODE` 区添加以下内容，不修改其他区域：

| 区域 | 添加内容 |
|---|---|
| `/* USER CODE BEGIN Includes */` | `#include "app_control.h"` |
| `/* USER CODE BEGIN 2 */` | `APP_Control_Init();`（位于 `MX_TIM6_Init()` 之后） |

其他模块的初始化由 `APP_Control_Init()` 内部调度，不需要在 `main.c` 中逐一调用。

#### 1.5.2 stm32g4xx_it.c

**不修改**。所有 HAL 中断入口保持 CubeMX 生成的原样。应用逻辑通过 HAL 回调函数实现：

| 中断入口 | HAL 回调 | 回调所在文件 |
|---|---|---|
| `ADC1_2_IRQHandler` | `HAL_ADCEx_InjectedConvCpltCallback`、`HAL_ADC_ConvCpltCallback` | `app_sampling.c` |
| `TIM6_DAC_IRQHandler` | `HAL_TIM_PeriodElapsedCallback` | `app_control.c` |
| `EXTI15_10_IRQHandler` | `HAL_GPIO_EXTI_Callback` | `app_control.c` |
| `DMA1_Channel2_IRQHandler` + `LPUART1_IRQHandler`（Normal DMA 完成 → UART TC） | `HAL_UART_TxCpltCallback` | `app_monitor.c` |

#### 1.5.3 CubeMX 外设源文件

**不修改** `adc.c`、`tim.c`、`dac.c`、`gpio.c`、`usart.c`、`dma.c`。以上文件中的 `MX_*_Init` 已由 `main.c` 调用完成外设初始化。MD-01 在运行阶段通过 HAL API 执行外设操作。

#### 1.5.4 CMakeLists.txt

顶层 `CMakeLists.txt` 中在 `target_sources` 和 `target_include_directories` 的 USER 区域添加：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    App/Src/app_bsp.c
    App/Src/app_sampling.c
    App/Src/app_process.c
    App/Src/app_control.c
    App/Src/app_monitor.c
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    App/Inc
)
```

---

## 2. 接口与数据传递设计

### 2.1 模块公开接口

以下仅列出各模块 `.h` 文件中声明的公开函数接口。HAL 回调函数属于模块 `.c` 文件内部的 HAL 接入点，在 §2.2 中单独描述。

#### 2.1.1 MD-01 板级支持模块（app_bsp.h）

| 接口 | 用途 | 调用方 | 输入 | 输出 |
|---|---|---|---|---|
| 实时链路启动 | 按顺序启动 ADC 注入组、TIM1 和 PWM 输出 | MD-04 | 无 | 无 |
| 实时链路停止 | 按顺序停止 TIM1、PWM 输出和 ADC 注入组 | MD-04 | 无 | 无 |
| 写 CH1 占空比 | 更新 TIM1 CCR1（10 kHz 路径中调用） | MD-03 | `uint16_t duty` | 无 |
| 写 CH2 占空比 | 写入 TIM1 CCR2 预设值（初始化时调用） | MD-04 | `uint16_t duty` | 无 |
| 写 CH3 占空比 | 写入 TIM1 CCR3 预设值（初始化时调用） | MD-04 | `uint16_t duty` | 无 |
| 写 DAC 输出 | 写入 DAC DHR 寄存器 | MD-04 | `uint16_t value` | 无 |
| 启动 DAC 输出 | 使能 DAC 输出 | MD-04 | 无 | 无 |
| 停止 DAC 输出 | DAC 输出置零并关闭 | MD-04 | 无 | 无 |
| 翻转 LED | 翻转 PA5 输出电平 | MD-04 | 无 | 无 |
| 启动 DMA 发送 | 启动 LPUART1 DMA 发送 | MD-05 | `uint8_t *data`、`uint16_t len` | `bool`（启动成功返回 true） |
| 启动 TIM6 | 启动 TIM6 计数器 | MD-04 | 无 | 无 |
| 启动 ADC 规则组 | 启动 ADC1 规则组中断转换（温度采样） | MD-04 | 无 | 无 |
| 熄灭 LED | 将 PA5 输出置为低电平 | MD-04 | 无 | 无 |

#### 2.1.2 MD-02 采样管理模块（app_sampling.h）

| 接口 | 用途 | 调用方 | 输入 | 输出 |
|---|---|---|---|---|
| 获取芯片温度值 | 返回当前温度值（只读） | MD-04（停止状态串口发送） | 无 | `float` 温度值 |
| 温度有效性查询 | 返回温度值是否已经过首次有效转换 | MD-04 | 无 | `bool` |

> 注：MD-02 同时实现 `HAL_ADCEx_InjectedConvCpltCallback` 和 `HAL_ADC_ConvCpltCallback` 两个 HAL 回调（见 §2.2），回调内部调用 MD-03 的处理函数（传入三路采样值）和 MD-04 的注入组完成通知函数、温度处理函数。

#### 2.1.3 MD-03 实时处理模块（app_process.h）

| 接口 | 用途 | 调用方 | 输入 | 输出 |
|---|---|---|---|---|
| 处理注入组采样值 | 计算 PWM 占空比 → MD-01 写 CCR1；计算 DAC 幅值 → 暂存 | MD-02（注入组 HAL 回调中） | `uint16_t ch1`、`uint16_t ch2`、`uint16_t ch3` | 无 |
| 获取 DAC 方波幅值 | 返回当前 DAC 幅值（只读） | MD-04（500 Hz 时读取，用于 DAC 翻转和串口数据源） | 无 | `uint16_t` |

#### 2.1.4 MD-04 系统控制模块（app_control.h）

| 接口 | 用途 | 调用方 | 输入 | 输出 |
|---|---|---|---|---|
| 应用初始化 | 应用模块初始化、设定初始状态为停止、启动 TIM6 | `main.c`（USER CODE BEGIN 2） | 无 | 无 |
| 注入组完成通知 | 运行状态下翻转 LED | MD-02（注入组 HAL 回调中） | 无 | 无 |
| 温度处理 | 接收温度值，执行异常判断，驱动状态机 | MD-02（规则组 HAL 回调中） | `float temperature` | 无 |

> 注：MD-04 同时实现 `HAL_TIM_PeriodElapsedCallback` 和 `HAL_GPIO_EXTI_Callback` 两个 HAL 回调（见 §2.2）。

#### 2.1.5 MD-05 串口监测模块（app_monitor.h）

| 接口 | 用途 | 调用方 | 输入 | 输出 |
|---|---|---|---|---|
| 发送数据 | JustFloat 组帧 → MD-01 启动 DMA。若启动成功则置忙，若忙或启动失败则忽略本次调用 | MD-04（500 Hz 调度中） | `float value` | `bool`（发送是否被接受并启动） |
| 查询忙状态 | 返回 DMA 是否正在发送 | MD-04 | 无 | `bool` |

> 注：MD-05 同时实现 `HAL_UART_TxCpltCallback` HAL 回调（见 §2.2）。

---

### 2.2 HAL 回调入口

以下 HAL 回调由各模块 `.c` 文件实现，不属于模块头文件公开接口。它们是 CubeMX 生成的中断入口到应用逻辑的桥接点：

| HAL 回调函数 | 所在文件 | 触发路径 | 处理摘要 |
|---|---|---|---|
| `HAL_ADCEx_InjectedConvCpltCallback` | `app_sampling.c` | `ADC1_2_IRQHandler` → `HAL_ADC_IRQHandler` → JEOS 触发 | 读取三路注入组结果 → MD-03 处理 → MD-04 通知 |
| `HAL_ADC_ConvCpltCallback` | `app_sampling.c` | `ADC1_2_IRQHandler` → `HAL_ADC_IRQHandler` → EOC 触发 | 读取规则组温度结果 → 换算温度 → MD-04 温度处理 |
| `HAL_TIM_PeriodElapsedCallback` | `app_control.c` | `TIM6_DAC_IRQHandler` → `HAL_TIM_IRQHandler` → Update 触发 | 500 Hz 调度：DAC 翻转、LED 分频、串口发送 |
| `HAL_GPIO_EXTI_Callback` | `app_control.c` | `EXTI15_10_IRQHandler` → `HAL_GPIO_EXTI_IRQHandler` 触发 | 检测 B1_Pin → 停止↔运行状态切换 |
| `HAL_UART_TxCpltCallback` | `app_monitor.c` | Normal DMA 模式：`DMA1_Channel2_IRQHandler` → HAL 使能 UART TCIE → `LPUART1_IRQHandler` → `HAL_UART_IRQHandler` → UART TC 触发 | 清除 DMA 发送忙标志 |

---

### 2.3 公共数据类型与配置常量

#### 2.3.1 系统状态枚举（app_control.c 中定义，MD-04 内部使用）

`SystemState_t` 仅在 MD-04 内部使用，不对外公开，定义在 `app_control.c` 中：

```c
typedef enum {
    SYSTEM_STATE_STOP,   // 停止状态
    SYSTEM_STATE_RUN,    // 运行状态
    SYSTEM_STATE_FAULT   // 异常状态
} SystemState_t;
```

#### 2.3.2 应用配置常量（app_control.c 中定义）

| 常量 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| TEMPERATURE_FAULT_THRESHOLD | `float` | 待确认 | 温度异常阈值 |
| TEMPERATURE_RECOVERY_THRESHOLD | `float` | 待确认 | 温度恢复阈值 |
| PWM_DUTY_CH2_PRESET | `uint16_t` | 待确认 | CH2 预设占空比 |
| PWM_DUTY_CH3_PRESET | `uint16_t` | 待确认 | CH3 预设占空比 |
| LED_FAULT_DIVIDER | `uint8_t` | 25 | 异常状态 LED 分频值（500 Hz / 25 = 10 Hz） |

---

### 2.4 主要运行数据的产生与使用关系

| 数据 | 产生位置 | 持有模块 | 更新频率 | 更新上下文 | 读取模块 | 读取上下文 |
|---|---|---|---|---|---|---|
| 注入组采样值（3 路） | MD-02 读取 ADC1 注入组结果寄存器 | 不持有（瞬时传递） | 10 kHz | ADC1_2 ISR（注入组分支） | MD-03 | 同一 ISR 内 |
| PWM CH1 占空比 | MD-03 计算 | MD-03 计算后立即通过 MD-01 写入 CCR1 | 10 kHz | ADC1_2 ISR | — | — |
| DAC 方波幅值 | MD-03 计算 | MD-03（持存） | 10 kHz | ADC1_2 ISR | MD-04 | TIM6 ISR（500 Hz） |
| 芯片温度值 | MD-02 读取 + 换算 | MD-02（持存） | 500 Hz | ADC1_2 ISR（规则组分支） | MD-04 | 规则组回调中（温度判断）；TIM6 ISR（停止状态串口发送） |
| 芯片温度有效标志 | MD-02 首次转换完成后置位 | MD-02（持存） | 首次置位后保持 | ADC1_2 ISR | MD-04 | TIM6 ISR |
| SystemState | MD-04 状态机 | MD-04（独占） | 事件驱动 | EXTI ISR / 规则组回调 | MD-04 内部 | 各回调中 |

### 2.5 模块私有变量与跨模块数据边界

| 模块 | 私有变量 | 跨模块共享数据 | 边界说明 |
|---|---|---|---|
| MD-01 | 无 | 无 | 不持有数据，仅操作外设 |
| MD-02 | 芯片温度值、温度有效标志 | 芯片温度值（对外只读） | 提供 `SAMP_GetTemperature()` 和 `SAMP_IsTemperatureValid()` 返回拷贝值 |
| MD-03 | DAC 方波幅值 | DAC 方波幅值（对外只读） | 提供 `PROC_GetDacAmplitude()` 返回当前值 |
| MD-04 | SystemState、LED 分频计数器、DAC 翻转标志 | SystemState（仅内部使用） | SystemState 不对外暴露，其他模块通过 MD-04 的动作（启停命令、数据选择）间接感知状态变化 |
| MD-05 | DMA 忙标志、帧缓冲区 | 无（仅接收 MD-04 传入数据） | 不持有业务数据，数据由 MD-04 传入 |

### 2.6 跨中断上下文数据共享原则

本项目存在两类跨中断上下文数据共享场景：

| 共享数据 | 写上下文 | 读上下文 | 访问原则 |
|---|---|---|---|
| DAC 方波幅值（MD-03 持有） | ADC1_2 ISR（注入组分支，10 kHz，优先级 0） | TIM6 ISR（500 Hz，优先级 1） | MD-03 以 `volatile` 修饰持存变量；Cortex-M4 上 32 位以内对齐变量的单次读写为原子操作，`uint16_t` 满足条件，可不使用临界区 |
| 芯片温度值（MD-02 持有） | ADC1_2 ISR（规则组分支，优先级 0） | TIM6 ISR（500 Hz，优先级 1） | MD-02 以 `volatile` 修饰持存变量；`float` 类型在 Cortex-M4 上为 32 位对齐单次访问，满足原子条件。如果编译器优化导致多次访问，建议在 TIM6 ISR 读取时使用局部变量快照 |

**通用原则**：
- 所有跨 ISR 共享的变量均以 `volatile` 声明，禁止编译器缓存优化。
- 读方在首次读取后保存局部快照，避免同一变量在函数内被多次读取时出现不一致。
- 不引入临界区保护（仅两个中断，且读写模式为"单写单读"，不需要互斥）。
- 上电后首个有效数据产生之前，读方通过有效性标志或默认值保证安全。

---

## 3. 状态与时序设计

### 3.1 系统状态转换

#### 3.1.1 状态定义

| 状态 | 标识 | 说明 |
|---|---|---|
| 停止 | `SYSTEM_STATE_STOP` | 上电/复位后的初始状态。TIM6 运行、温度采样运行，实时链路关闭 |
| 运行 | `SYSTEM_STATE_RUN` | 实时链路运行，PWM/DAC/ADC 注入组全速工作 |
| 异常 | `SYSTEM_STATE_FAULT` | 温度超限保护状态。实时链路关闭，温度采样继续判断恢复 |

#### 3.1.2 状态转换条件与动作

| 转换 | 触发事件 | 来源 | 条件 | 进入动作 | 退出动作 |
|---|---|---|---|---|---|
| 停止 → 运行 | 按键按下 | `HAL_GPIO_EXTI_Callback`（MD-04） | `GPIO_Pin == B1_Pin` | ① 写 CH2/CH3 预设占空比 → MD-01 ② 启动实时链路 → MD-01 ③ 启动 DAC 输出 → MD-01 ④ 状态 = RUN | 无 |
| 运行 → 停止 | 按键按下 | 同上 | 同上 | ① 停止实时链路 → MD-01 ② DAC 置零并关闭 → MD-01 ③ 熄灭 LED → MD-01 ④ 状态 = STOP | 无 |
| 停止 → 异常 | 温度异常进入 | `HAL_ADC_ConvCpltCallback` → MD-02 → MD-04 温度处理 | `temperature >= THRESHOLD_FAULT` | ① 状态 = FAULT | 无（实时链路本已关闭） |
| 运行 → 异常 | 温度异常进入 | 同上 | 同上 | ① 停止实时链路 → MD-01 ② DAC 置零并关闭 → MD-01 ③ 状态 = FAULT | — |
| 异常 → 停止 | 温度异常退出 | 同上 | `temperature <= THRESHOLD_RECOVERY` | ① LED 熄灭 → MD-01 ② 复位 LED 分频计数器 ③ 状态 = STOP | 无（实时链路已关闭，保持关闭） |

#### 3.1.3 各状态下的行为矩阵

| 行为 | 停止 | 运行 | 异常 |
|---|---|---|---|
| TIM1/PWM 输出 | 关闭 | 10 kHz 三组互补 | 关闭 |
| ADC1 注入组 | 停止 | 10 kHz 三通道 | 停止 |
| MD-03 实时处理 | 不执行 | 10 kHz | 不执行 |
| DAC 输出 | 关闭 | 250 Hz 方波 | 关闭 |
| 温度采样 + 判断 | 500 Hz 执行 | 500 Hz 执行 | 500 Hz（仅判断恢复） |
| LED | 熄灭 | 每 10 kHz 翻转 | 10 Hz 闪烁 |
| 串口发送 | 500 Hz，发温度值 | 500 Hz，发 DAC 值 | 不发送 |
| 按键 | 响应 | 响应 | 不响应 |

---

### 3.2 事件处理链路

#### 3.2.1 10 kHz 实时路径（仅运行状态）

```
TIM1 CH4 比较事件（硬件）
  └─ OC4REF → TRGO2（硬件）
       └─ ADC1 注入组按 Rank1→2→3 转换（硬件）
            └─ JEOS → ADC1_2_IRQHandler
                 └─ HAL_ADC_IRQHandler(&hadc1)
                      └─ HAL_ADCEx_InjectedConvCpltCallback  [app_sampling.c, MD-02]
                           │
                           ├─ ① HAL_ADCEx_InjectedGetValue 读取三通道结果
                           ├─ ② 调用 MD-03 处理采样值 [app_process.c]
                           │      ├─ 计算 PWM 占空比 → MD-01 写 TIM1 CCR1
                           │      └─ 计算 DAC 幅值 → 暂存于 MD-03
                           └─ ③ 调用 MD-04 注入组完成通知 [app_control.c]
                                  └─ 运行状态：MD-01 翻转 LED
```

**时序约束**：从 `HAL_ADCEx_InjectedConvCpltCallback` 进入至 TIM1 CCR1 写入完成须远小于 100 µs。

#### 3.2.2 500 Hz 周期处理（全状态）

500 Hz 周期由同一 TIM6 更新事件驱动两条并发子路径：

**路径 A — 软件调度（TIM6 ISR 上下文）：**

```
TIM6 更新事件
  └─ TIM6_DAC_IRQHandler
       └─ HAL_TIM_IRQHandler(&htim6)
            └─ HAL_TIM_PeriodElapsedCallback  [app_control.c, MD-04]
                 │  （检查 htim->Instance == TIM6）
                 │
                 ├─ ① DAC 翻转：
                 │      读取 MD-03 DAC 幅值 → 交替输出幅值/0 → MD-01 写 DAC
                 │      停止/异常状态：DAC 关闭，跳过
                 │
                 ├─ ② LED 分频：
                 │      异常状态：分频计数器++，每 25 次翻转 LED
                 │
                 └─ ③ 串口发送调度：
                        运行：读取 MD-03 DAC 幅值 → MD-05 发送
                        停止：温度有效时读取 MD-02 温度值 → MD-05 发送
                        异常：不触发
```

**路径 B — 温度采样（硬件触发，异步于路径 A）：**

```
TIM6 更新事件
  └─ TRGO → ADC1 规则组转换 CH16（硬件自动）

  ADC1 规则组转换完成 → EOC → ADC1_2_IRQHandler
       └─ HAL_ADC_IRQHandler(&hadc1)
            └─ HAL_ADC_ConvCpltCallback  [app_sampling.c, MD-02]
                 │
                 ├─ ① HAL_ADC_GetValue 读取温度 ADC 值
                 ├─ ② 换算为芯片温度值 → 本模块持存
                 ├─ ③ 首次转换后置位温度有效标志
                 └─ ④ 调用 MD-04 温度处理 [app_control.c]
                        ├─ 停止/运行状态：温度 ≥ 异常阈值 → 状态 → FAULT + 停止实时链路
                        └─ 异常状态：温度 ≤ 恢复阈值 → 状态 → STOP
```

**路径 A 与路径 B 的时序关系**：同一 TIM6 更新事件同时触发路径 A（ISR 立即进入）和路径 B（TRGO → ADC 转换需要时间）。两者之间无确定的先后关系。温度判断始终在路径 B 的 ADC 转换完成后执行，不受路径 A 的执行时序影响。

#### 3.2.3 异步事件

| 事件 | 中断入口 | HAL 回调 | 处理模块 | 处理逻辑 |
|---|---|---|---|---|
| 用户按键 | `EXTI15_10_IRQHandler` | `HAL_GPIO_EXTI_Callback` | MD-04 | 检查 `GPIO_Pin == B1_Pin`，停止↔运行切换。异常状态下不响应 |
| DMA 传输完成 → UART TC | `DMA1_Channel2_IRQHandler`（HAL 内部使能 UART TCIE）→ `LPUART1_IRQHandler` | `HAL_UART_TxCpltCallback` | MD-05 | Normal DMA 模式：DMA 完成后 HAL 使能 UART TC 中断，最终由 LPUART1 ISR 触发 TxCpltCallback。检查 `huart->Instance == LPUART1`，清除发送忙标志 |

---

### 3.3 中断处理与程序处理的职责分工

| 上下文 | 优先级 | 执行内容 | 约束 |
|---|---|---|---|
| ADC1_2 ISR（注入组） | 0（最高） | MD-02 读取采样值 → MD-03 计算 → MD-01 写 CCR1；MD-04 LED 翻转 | 总执行时间 ≪ 100 µs。此路径不写入 DAC、不启动 DMA |
| TIM6 ISR | 1 | MD-04：DAC 翻转、LED 分频计数、串口发送调度 | 500 Hz 频率低，但应避免在 ISR 中执行耗时操作。DAC 和 LED 操作为寄存器写入，串口发送通过 DMA 异步执行 |
| ADC1_2 ISR（规则组） | 0 | MD-02 读取温度 → 换算 → MD-04 温度判断和状态切换 | 注入转换可以抢占正在进行的规则转换（STM32G4 硬件支持），但两者共用 ADC1_2_IRQn，不会形成 ISR 嵌套 |
| EXTI ISR | 4 | MD-04：状态切换 | 极低频、极短 |
| DMA ISR | 2 | 无应用处理（HAL 内部在 DMA 完成后使能 UART TC 中断） | 极短 |
| LPUART1 ISR | 3 | MD-05：清除 DMA 忙标志（在 `HAL_UART_TxCpltCallback` 中） | Normal DMA 模式：DMA 完成后 HAL 使能 UART TCIE，LPUART1 ISR 触发 TxCpltCallback |
| main() 主循环 | — | 当前为空 | 无后台任务需要轮询 |

---

### 3.4 应用初始化流程

上电后 `main.c` 调用 `APP_Control_Init()`（位于 `MX_TIM6_Init()` 之后），内部执行：

```
APP_Control_Init()  [app_control.c, MD-04]
  │
  ├─ ① 设定 SystemState = SYSTEM_STATE_STOP
  │
  ├─ ② 通过 MD-01 写入 CH2/CH3 预设占空比（CCR2、CCR3 初始值）
  │
  ├─ ③ 复位外设初始状态：
  │       LED 熄灭 → MD-01
  │       DAC 输出关闭 → MD-01
  │       TIM1/PWM 保持停止（main.c 仅初始化未启动）
  │
  ├─ ④ 启动 TIM6 → MD-01（开始 500 Hz 温度采样和周期调度）
  │
  └─ ⑤ 返回（系统处于停止状态，等待按键事件）
```

**初始化时序关键点**：
- `main.c` 的 `MX_*_Init()` 调用已完成外设初始化，`APP_Control_Init()` 不重复调用。
- TIM6 启动后立即开始：TRGO 触发温度采样 + ISR 产生 500 Hz 周期事件。
- 首个 500 Hz ISR 中，路径 B（温度采样）尚未完成首次转换，温度有效标志为 false，停止状态下串口发送被跳过。

---

### 3.5 实时链路启动与停止顺序

#### 3.5.1 停止 → 运行（启动）

由 MD-04 在 `HAL_GPIO_EXTI_Callback`（停止状态分支）中调用 MD-01。

> **HAL 行为说明**：`HAL_TIM_PWM_Start()` 在首次调用时会启动 TIM1 计数器。因此启动序列中，对第一条通道调用 `HAL_TIM_PWM_Start()` 即为实际的计数器启动时刻。

```
MD-01 实时链路启动：
  ① HAL_ADCEx_InjectedStart_IT(&hadc1)     // 先使能 ADC 注入组中断
  ② HAL_TIM_PWM_Start(&htim1, CH1)         // 启动 CH1 PWM（首次调用，同时启动 TIM1 计数器）
  ③ HAL_TIMEx_PWMN_Start(&htim1, CH1N)     // 使能 CH1N 互补输出
  ④ HAL_TIMEx_PWMN_Start(&htim1, CH2N)     // CH2N（CH2 的 PWM_Start 在第②步之前单独调用？待确认）
  ⑤ HAL_TIMEx_PWMN_Start(&htim1, CH3N)     // CH3N
  （CH4 无需调用 Start——仅内部 OC4REF 经 TRGO2 路由，不输出到引脚）
```

**时序要求**：先使能 ADC 注入组中断，后启动 TIM1。CH2/CH3 的 PWM 通道启动需要与 CH1 同批完成，确保所有通道在计数器启动时已配置好输出使能。具体实现方式在编码阶段确定（可选方案：提前用 `TIM_CCxChannelCmd()` 使能各通道输出，最后统一调用一次 `HAL_TIM_Base_Start()` 启动计数器；或按顺序依次 `HAL_TIM_PWM_Start`，但确保在第一条之前 ADC 已就绪）。

#### 3.5.2 运行 → 停止 / 异常（停止）

由 MD-04 调用 MD-01。

> **HAL 行为说明**：`HAL_TIM_PWM_Stop()` 在最后一个通道停止时会自动停止 TIM1 计数器。因此停止序列中应确保计数器停止发生在前，ADC 注入组停止在后。

```
MD-01 实时链路停止：
  ① HAL_TIM_PWM_Stop(&htim1, CH1)          // 停止 CH1 PWM（若为最后活跃通道，同时停止 TIM1 计数器）
  ② HAL_TIMEx_PWMN_Stop(&htim1, CH1N)      // 停止互补输出
  ③ HAL_TIM_PWM_Stop(&htim1, CH2) ~ CH3    // 停止 CH2/CH3
  ④ HAL_TIMEx_PWMN_Stop(&htim1, CH2N) ~ CH3N
  ⑤ HAL_ADCEx_InjectedStop_IT(&hadc1)      // 停止 ADC 注入组（在 TIM1 计数器已停止后）
  ⑥ HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, 0)  // DAC 置零
  ⑦ HAL_DAC_Stop(&hdac1, DAC_CHANNEL_1)    // 关闭 DAC 输出
  ⑧ HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET)    // LED 熄灭
```

**时序要求**：先停止 TIM1 计数器（停止触发源），后停止 ADC 注入组。具体实现方式在编码阶段确定。

---

### 3.6 时序与数据有效性约束

| 约束 | 说明 |
|---|---|
| 10 kHz 路径执行时间 | MD-02 + MD-03 + MD-01 在注入组 ISR 中的总执行时间须远小于 100 µs |
| 温度值有效标志 | 上电后首次规则组转换完成前 `temperature_valid = false`，停止状态串口发送依赖此标志 |
| 跨 ISR 数据 | DAC 幅值和芯片温度值以 `volatile` 声明，读写模式为单写单读 |
| DMA 忙标志 | MD-05 在启动 DMA 前检查，发送成功后置位，DMA 完成回调中清除。忙时不重复启动 |
| 500 Hz 发送 | 若上一帧 DMA 未完成（忙），当前 500 Hz 周期跳过发送，等待下一周期 |
| 异常状态温度判断 | 异常状态下仅检查温度是否降到恢复阈值，不产生异常进入事件 |
| TIM6 全状态运行 | TIM6 在上电后启动，不再停止。停止/运行/异常状态下 500 Hz 调度均持续 |
