# STM32G431 ADC1 通道2 软件触发 + 中断采集 — 工程分析报告

## 1. 开发板和MCU型号

| 项目 | 型号 |
|------|------|
| **开发板** | NUCLEO-G431RB |
| **MCU** | STM32G431RBT6 (LQFP64 封装) |
| **MCU家族** | STM32G4 |
| **STM32Cube FW版本** | STM32Cube FW_G4 V1.6.3 |
| **CubeMX版本** | 6.16.0 |

## 2. 外设、端口和引脚配置

### 2.1 ADC 外设配置（来自 .ioc 文件和 `MX_ADC1_Init()`）

| 参数 | 配置值 |
|------|--------|
| **ADC实例** | ADC1 |
| **通道** | ADC_CHANNEL_2 (ADC1_IN2) |
| **引脚** | PA1 |
| **引脚模式** | Analog mode (IN2-Single-Ended) |
| **触发方式** | `ADC_SOFTWARE_START`（软件触发） |
| **转换模式** | 单次转换 (`ContinuousConvMode = DISABLE`) |
| **扫描模式** | 禁用 (`ScanConvMode = DISABLE`) |
| **分辨率** | 12位 (`ADC_RESOLUTION_12B`) |
| **数据对齐** | 右对齐 (`ADC_DATAALIGN_RIGHT`) |
| **EOC选择** | `ADC_EOC_SINGLE_CONV`（单次转换结束标志） |
| **采样时间** | 24.5 ADC时钟周期 (`ADC_SAMPLETIME_24CYCLES_5`) |
| **通道Rank** | Rank 1 (`ADC_REGULAR_RANK_1`) |
| **单端/差分** | 单端 (`ADC_SINGLE_ENDED`) |
| **ADC时钟源** | 系统时钟 SYSCLK，异步4分频 (`ADC_CLOCK_ASYNC_DIV4`) |
| **ADC时钟频率** | 160MHz / 4 = **40 MHz** |
| **DMA** | 禁用 |

### 2.2 GPIO 配置

| 引脚 | 功能 | 模式 | 备注 |
|------|------|------|------|
| **PA1** | ADC1_IN2 | Analog mode (`GPIO_MODE_ANALOG`) | 目标采集引脚 |
| PA5 | LD2 (板载LED) | Output PP | `GPIO_Output` |
| PC13 | USER_BTN | Input | `GPIO_Input` |
| PA13 | T_SWDIO | Serial Wire | 调试接口 |
| PA14 | T_SWCLK | Serial Wire | 调试接口 |

### 2.3 中断配置

| 中断向量 | 中断优先级 | 状态 |
|----------|------------|------|
| **ADC1_2_IRQn** | 优先级 0, 子优先级 0 | **已使能** |

- 中断向量表条目: `startup_stm32g431xx.s:167` — `ADC1_2_IRQHandler`
- NVIC优先级配置在 `stm32g4xx_hal_msp.c:122`: `HAL_NVIC_SetPriority(ADC1_2_IRQn, 0, 0)`
- NVIC使能在 `stm32g4xx_hal_msp.c:123`: `HAL_NVIC_EnableIRQ(ADC1_2_IRQn)`

### 2.4 时钟树

| 时钟节点 | 频率 |
|----------|------|
| HSI | 16 MHz |
| PLL N=20, M=1, P/Q/R=÷2 | VCO=320MHz → 160MHz |
| **SYSCLK** | **160 MHz** |
| HCLK (AHB) | 160 MHz |
| APB1 (PCLK1) | 160 MHz |
| APB2 (PCLK2) | 160 MHz |
| **ADC12 时钟 (异步)** | **40 MHz** (160MHz ÷ 4) |

## 3. 已有初始化函数和宏定义

### 3.1 初始化函数（`Core/Src/main.c`）

| 函数 | 所在文件 | 行号 | 说明 |
|------|----------|------|------|
| `HAL_Init()` | HAL库 | — | 系统初始化，SysTick配置 |
| `SystemClock_Config()` | main.c | 118 | 系统时钟配置 (160MHz) |
| `MX_GPIO_Init()` | main.c | 233 | GPIO初始化 (PC13, PA5) |
| `MX_ADC1_Init()` | main.c | 165 | **ADC1完整初始化** |

### 3.2 ADC MSP初始化（`Core/Src/stm32g4xx_hal_msp.c`）

| 函数 | 行号 | 说明 |
|------|------|------|
| `HAL_ADC_MspInit()` | 90 | ADC时钟配置 + GPIO + NVIC |
| `HAL_ADC_MspDeInit()` | 138 | ADC反初始化 |

MSP初始化内容：
- 时钟: `RCC_ADC12CLKSOURCE_SYSCLK` → `PeriphClkInit.Adc12ClockSelection`
- 外设时钟使能: `__HAL_RCC_ADC12_CLK_ENABLE()`
- GPIO: PA1 → `GPIO_MODE_ANALOG`, `GPIO_NOPULL`
- 中断: NVIC优先级设置和使能

### 3.3 中断服务函数（`Core/Src/stm32g4xx_it.c`）

| 函数 | 行号 | 说明 |
|------|------|------|
| `ADC1_2_IRQHandler()` | 204 | ADC中断服务函数，**已调用 `HAL_ADC_IRQHandler(&hadc1)`** |

### 3.4 关键变量和宏定义

| 名称 | 所在文件 | 类型 | 说明 |
|------|----------|------|------|
| `hadc1` | main.c:45 | `ADC_HandleTypeDef` | ADC1句柄（全局） |
| `BspButtonState` | main.c:44 | `__IO uint32_t` | BSP按键状态 |
| `LD2_Pin` | main.h:64 | `#define` | `GPIO_PIN_5` |
| `LD2_GPIO_Port` | main.h:65 | `#define` | `GPIOA` |
| `USER_BTN_Pin` | main.h:62 | `#define` | `GPIO_PIN_13` |
| `USER_BTN_GPIO_Port` | main.h:63 | `#define` | `GPIOC` |

## 4. 需要使用的 HAL 函数

以下函数是 ADC 中断模式采集的完整调用链：

| 函数 | 用途 | 说明 |
|------|------|------|
| `HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)` | **ADC校准** | ⚠️ 当前工程未调用，推荐在 `HAL_ADC_Init()` 之后添加 |
| `HAL_ADC_Start_IT(&hadc1)` | 启动ADC中断转换 | 触发一次软件启动的单次转换 |
| `HAL_ADC_IRQHandler(&hadc1)` | 中断处理 | ✅ 已在 `ADC1_2_IRQHandler` 中调用 |
| `HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)` | 转换完成回调 | ⚠️ 需用户在代码中重写（`__weak` 函数） |
| `HAL_ADC_GetValue(&hadc1)` | 读取ADC转换结果 | 在回调中调用，返回12-bit原始值 |
| `HAL_ADC_Stop_IT(&hadc1)` | 停止ADC中断模式 | 可选，仅需停止时使用 |

### 调用流程

```
main()
  → MX_ADC1_Init()                      // ADC初始化（已完成）
  → HAL_ADCEx_Calibration_Start()       // 校准（需添加）
     ↓ 主循环（每100ms）
  → HAL_ADC_Start_IT()                  // 启动一次软件触发中断转换（需添加）
     ↓ 转换完成触发中断
  → ADC1_2_IRQHandler()                 // 中断入口（已完成）
     → HAL_ADC_IRQHandler()             // HAL中断处理（已完成）
        → HAL_ADC_ConvCpltCallback()    // 用户回调（需添加）
           → 判断 hadc->Instance == ADC1
           → HAL_ADC_GetValue()         // 读取原始值（需添加）
           → 设置更新标志
     ↓ 主循环根据标志位
  → 换算为电压                          // 公式: Vin = raw * Vref / 4096
```

> **注意**: `HAL_ADC_Start_IT()` 在主循环中按 100ms 周期调用，不要在回调中立即再次调用它，以避免产生高频中断。

## 5. 需要修改的文件和代码位置

| 文件 | 修改位置 | 修改内容 |
|------|----------|----------|
| **main.c** | `USER CODE BEGIN PV` (line 47-48) | 新增 volatile 全局变量：ADC原始值、更新标志 |
| **main.c** | `USER CODE BEGIN 0` (line 60-62) | 添加电压换算函数（可选） |
| **main.c** | `USER CODE BEGIN 2` (line 94-96) | 添加ADC校准 |
| **main.c** | `USER CODE BEGIN WHILE` (line 103) | 每100ms调用 `HAL_ADC_Start_IT()`，检查更新标志后换算电压 |
| **main.c** | `USER CODE BEGIN 4` (line 265-267) | 添加 `HAL_ADC_ConvCpltCallback()` 回调函数实现（含实例判断） |

### 无需修改的文件

| 文件 | 原因 |
|------|------|
| `ADC_IT.ioc` | 不修改.ioc文件 |
| `stm32g4xx_it.c` | `ADC1_2_IRQHandler` 已正确实现，无需修改 |
| `stm32g4xx_hal_msp.c` | MSP初始化已完整，无需修改 |
| `stm32g4xx_it.h` | 中断向量声明已完整，无需修改 |
| `startup_stm32g431xx.s` | 向量表已含 `ADC1_2_IRQHandler`，无需修改 |

## 6. 需要新增的内容

### 6.1 新增变量

所有在中断回调中修改、在主循环中读取的变量必须声明为 `volatile`，防止编译器优化导致主循环读到过期的寄存器缓存值。

| 变量 | 类型 | 说明 |
|------|------|------|
| `volatile uint32_t adc_raw_value` | `uint32_t` | 存储 `HAL_ADC_GetValue()` 返回值，在回调中写入，主循环中读取 |
| `volatile uint8_t adc_data_ready` | `uint8_t` | 更新标志：回调中置1表示新数据就绪，主循环处理完毕后清0 |

换算后的电压值 `float adc_voltage` 可声明为局部变量或在主循环中直接计算，无需作为全局变量。

### 6.2 新增函数

- **`void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)`** — 重写 HAL 库的 `__weak` 回调，在转换完成后被 `HAL_ADC_IRQHandler` 自动调用。实现要点：
  1. 首先判断 `hadc->Instance == ADC1`，区分 ADC1/ADC2 共享中断
  2. 调用 `HAL_ADC_GetValue(&hadc1)` 读取12位原始值
  3. 设置 `adc_data_ready = 1` 通知主循环处理

### 6.3 无需新增文件和中断
- 不需要新建任何文件
- 中断服务函数 `ADC1_2_IRQHandler` 不需要修改

## 7. 风险与注意事项

### 7.1 ⚠️ ADC校准缺失
当前工程 `MX_ADC1_Init()` 调用 `HAL_ADC_Init()` 后**未进行ADC校准**。STM32G4系列的ADC在校准前精度可能不足。需在 `HAL_ADC_Init()` 之后调用：
```c
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);
```

### 7.2 ⚠️ 单次转换模式（周期性触发）
当前配置 `ContinuousConvMode = DISABLE`（单次转换模式），每次调用 `HAL_ADC_Start_IT()` 只触发一次转换。采用主循环中每 100ms 调用 `HAL_ADC_Start_IT()` 的方式实现周期性采集。**不要在 `HAL_ADC_ConvCpltCallback()` 回调中再次调用 `HAL_ADC_Start_IT()`**，否则会产生高频连续中断，影响主循环正常运行。

### 7.3 ⚠️ EOCSelection 配置
当前配置 `EOCSelection = ADC_EOC_SINGLE_CONV`，每个通道转换完成后即产生 EOC 标志和中断，适用于本项目的单通道单次转换场景，保持不变即可。

### 7.4 ⚠️ ADC时钟频率
ADC异步时钟为 SYSCLK/4 = 40 MHz。参考手册规定：12位分辨率时 ADC 时钟不应超过数据手册限值（G431RB 为 60 MHz 最大值），40 MHz 在允许范围内。**总转换时间** = 采样时间(24.5) + 逐次逼近时间(12.5) = 37 ADC周期 = 0.925 μs。

### 7.5 ⚠️ ADC1_2_IRQHandler 共享
中断向量 `ADC1_2_IRQHandler` 是 ADC1 和 ADC2 共享的。当前只使用 ADC1，`HAL_ADC_IRQHandler(&hadc1)` 会正确处理ADC1的中断。但需注意 PA1 引脚 ADC12_IN2 同时是 ADC1_IN2 和 ADC2_IN2，确认 ADC2 未被误配置。

### 7.6 ⚠️ 参考电压
NUCLEO-G431RB 默认以 VDDA 为 ADC 参考电压（3.3V）。换算公式为：
```
Vin = (adc_raw_value * 3.3) / 4096.0
```
如果 VDDA 不是精确 3.3V，换算值会有误差。如需高精度，可考虑使用内部参考电压 VREFINT 进行校准。

### 7.7 ⚠️ 回调函数确认
`HAL_ADC_ConvCpltCallback` 是 `__weak` 函数，定义在 `stm32g4xx_hal_adc.c:2662`。用户重写版本会用强符号覆盖弱符号，编译器自动链接到用户版本。**必须确保回调函数签名完全匹配**：
```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc);
```

## 8. 总结

**工程条件评估：基本就绪。**

CubeMX 已经完整生成了 ADC1+PA1+中断的初始化代码，包括：
- ADC 参数配置（软件触发、单次转换、12bit、通道2）
- PA1 模拟引脚配置
- ADC 时钟配置（40MHz 异步时钟）
- NVIC 中断使能和优先级配置
- ADC1_2_IRQHandler 中断服务函数（已调用 HAL_ADC_IRQHandler）

**待实现功能包括：**
1. **ADC 校准** — 在 ADC 初始化之后调用 `HAL_ADCEx_Calibration_Start()`
2. **周期性软件触发** — 在主循环中每 100ms 调用 `HAL_ADC_Start_IT()` 启动单次转换
3. **中断读取原始值** — 在 `HAL_ADC_ConvCpltCallback()` 中判断 `hadc->Instance == ADC1` 后读取 `HAL_ADC_GetValue()`
4. **电压换算** — 主循环中根据 `adc_data_ready` 标志将原始值换算为电压：`Vin = raw × 3.3 / 4096`

所有修改均在 CubeMX 用户代码区（`USER CODE BEGIN/END`）内完成，不修改 .ioc 文件和 HAL 库文件。
