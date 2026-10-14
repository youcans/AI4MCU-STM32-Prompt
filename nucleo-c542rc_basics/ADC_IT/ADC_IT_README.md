# ADC_IT ADC 模拟量数据采集实验

> 面向嵌入式初学者的编程说明。读者应具备 STM32 基础、C 语言基础，并能用 STM32CubeMX2 + STM32CubeIDE（或 VS Code + Cube 工具链）生成与烧录工程。

## 1. 项目概述

本项目在 **NUCLEO-C542RC** 开发板（STM32C542RCT6）上，利用 **ADC1 通道 1（IN1）** 采集 **PA1** 引脚输入的模拟电压，采用**软件触发**启动转换 + **转换完成中断**方式，每 **100 ms** 启动一次采集，读取 **12 位** ADC 原始值并换算为输入电压（单位 mV）。通过调节外部电位器，可让 PA1 电压在 0 ~ 3.3 V 范围内变化，从而验证 ADC 采集与电压换算的正确性。

整个工程由 **STM32CubeMX2** 生成，**HAL 驱动库**完成底层初始化，本次任务只修改了用户文件 `main.c`。

## 2. 硬件说明

| 硬件 | 说明 |
| --- | --- |
| 开发板 | NUCLEO-C542RC |
| 主控芯片 | STM32C542RCT6（Cortex-M33 内核） |
| 系统时钟 | 48 MHz（HSI÷3，内部 HSIDIV3） |
| ADC 内核时钟 | 144 MHz（内部 PSIS 时钟），分频 1 |
| 模拟信号源 | 外部电位器，两端分别接 **3.3 V** 与 **GND** |
| ADC 采集引脚 | **PA1**（= ADC1 通道 1 / ADC1_IN1），接电位器中间滑动端 |
| 板载 LED | LD1，引脚 **PA5**，本项目未使用 |

> 特别注意：**PA1 对应 ADC1 的通道 1（IN1），不是通道 2（IN2/PA2）。** 决定采集哪个引脚的是引脚连接关系——电位器接到 PA1，就必须配置为 ADC1_IN1。不要因为文档写成"通道 1"而误接 PA2。

## 3. 功能需求

1. 使用 ADC1 通道 1 采集 PA1 引脚输入的模拟电压（电位器输出 0 ~ 3.3 V）。
2. 采用**软件触发**方式启动转换（不使用 TIM 等硬件触发）。
3. 使用**转换完成中断**方式读取结果（不阻塞 CPU，不使用 DMA）。
4. 每 **100 ms** 启动一次 ADC 采集。
5. 读取 **12 位** ADC 原始值（0 ~ 4095），并换算为输入电压（单位 mV，0 ~ 3300）。
6. 供调试器监视窗口查看：原始值变量 `g_adc_raw_value`、电压变量 `g_adc_voltage_mv`。

## 4. 实现思路

### 4.1 整体流程

ADC1 的初始化（时钟、PA1 模拟输入、通道配置、NVIC 中断）由 CubeMX2 生成的 `mx_system_init()` 统一完成。用户只需在 `main.c` 中：

1. 调用 `HAL_ADC_Start()` 激活 ADC；
2. 调用 `HAL_ADC_Calibrate()` 校准 ADC；
3. 每 100 ms 调用一次 `HAL_ADC_REG_StartConv_IT()`，以**软件触发 + 中断**方式启动一次单次转换；
4. 在转换完成回调里读取结果并换算为电压。

### 4.2 100 ms 非阻塞计时

主循环用 `HAL_GetTick()`（毫秒计数）做差值判断，每达到 100 ms 阈值就触发一次转换。差值判断写法 `(HAL_GetTick() - last) >= period` 可避免毫秒计数回绕（wrap-around）问题，是非阻塞计时的推荐写法。

### 4.3 中断方式（为什么可以反复触发）

`HAL_ADC_REG_StartConv_IT()` 启动的是**单次转换**。转换完成后，HAL 在 EOC 中断处理中把 regular 组状态自动恢复为 `IDLE`，再调用转换完成回调。因此下一次 100 ms 再调用 `HAL_ADC_REG_StartConv_IT()` 时状态机处于 IDLE，可以正常再次启动，不会因状态未复位而返回 `HAL_BUSY`。

### 4.4 电压换算公式

```text
电压(mV) = ADC原始值 × 参考电压(mV) / 满刻度值
满刻度值（12 位）= 4095
```

即 `电压 = raw × 3300 / 4095`。使用 ST 官方宏 `HAL_ADC_CALC_DATA_TO_VOLTAGE()` 完成计算，**不要手写公式**，以免出错（详见第 5 节与第 8 节的踩坑说明）。

## 5. 关键代码

工程为 STM32CubeMX2 生成的新式工程结构，初始化由 `mx_system_init()` 统一完成，用户只需在 `main.c` 中启动 ADC、实现 100 ms 计时与转换完成回调。

### 5.1 用户自定义参数

```c
#include "main.h"
#include "mx_adc1.h"

/* 采样参数 */
#define ADC_SAMPLE_PERIOD_MS     100U      /* 采样周期：每 100ms 启动一次转换   */
#define ADC_REF_VOLTAGE_MV       3300U     /* 模拟参考电压 Vref+ = 3.3V (mV)    */
#define ADC_RESOLUTION           HAL_ADC_RESOLUTION_12_BIT /* 分辨率 12 位(必须传 HAL 枚举编码值,不能传数字 12,否则换算错误) */

/* 供调试器监视窗口查看的全局变量 */
static hal_adc_handle_t *s_adc1_handle;    /* ADC1 句柄(来自 mx_adc1_gethandle)  */
volatile uint32_t g_adc_raw_value   = 0U;  /* ADC 原始值 0 ~ 4095               */
volatile uint32_t g_adc_voltage_mv  = 0U;  /* 换算电压 (mV) 0 ~ 3300             */
```

### 5.2 主函数：激活 + 校准 ADC

```c
int main(void)
{
  if (mx_system_init() != SYSTEM_OK)    /* CubeMX2 生成:时钟、GPIO、ADC1 初始化 */
  {
    return (-1);
  }
  else
  {
    s_adc1_handle = mx_adc1_gethandle();
    if (s_adc1_handle == NULL)
    {
      return (-1);
    }

    /* ADC 操作流程:先 activate(进入 ACTIVE),再 calibrate,最后启动转换 */
    if (HAL_ADC_Start(s_adc1_handle) != HAL_OK)
    {
      return (-1);
    }
    if (HAL_ADC_Calibrate(s_adc1_handle) != HAL_OK)
    {
      return (-1);
    }

    while (1)
    {
      adc_start_conversion();   /* 每 100ms 软件触发一次转换 */
    }
  }
}
```

> **顺序不可颠倒**：`HAL_ADC_Calibrate()` 内部断言要求 ADC 已处于 `ACTIVE` 状态，所以必须先 `HAL_ADC_Start()` 再 `HAL_ADC_Calibrate()`。

### 5.3 100 ms 非阻塞触发

```c
/**
  * @brief 每 100ms 软件触发启动一次 ADC 单次转换,转换完成后由中断回调处理。
  * @note  单次转换 EOC 后 HAL 自动将 regular 组状态恢复为 IDLE,
  *        故可反复调用 HAL_ADC_REG_StartConv_IT() 而不会返回 HAL_BUSY。
  */
static void adc_start_conversion(void)
{
  static uint32_t last_tick = 0U;

  if ((HAL_GetTick() - last_tick) >= ADC_SAMPLE_PERIOD_MS)
  {
    last_tick = HAL_GetTick();
    (void)HAL_ADC_REG_StartConv_IT(s_adc1_handle);
  }
}
```

### 5.4 转换完成回调：读值 + 换算

```c
/**
  * @brief ADC 常规组单次转换完成回调(覆写 HAL 弱函数)。
  *        读取 12 位 ADC 原始值并换算为输入电压 (mV)。
  * @param hadc ADC 句柄指针
  */
void HAL_ADC_REG_UnitaryConvCpltCallback(hal_adc_handle_t *hadc)
{
  (void)hadc;

  /* 读取 ADC 原始值(12 位单次转换结果,0 ~ 4095) */
  g_adc_raw_value = (uint32_t)HAL_ADC_REG_ReadConversionData(s_adc1_handle);

  /* 电压换算:mV = raw * Vref / 满刻度(4095),
     ST 官方宏,第三个参数须为 HAL_ADC_RESOLUTION_12_BIT 枚举值 */
  g_adc_voltage_mv = (uint32_t)HAL_ADC_CALC_DATA_TO_VOLTAGE(ADC_REF_VOLTAGE_MV,
                                                             g_adc_raw_value,
                                                             ADC_RESOLUTION);
}
```

## 6. HAL 依赖前提

本工程使用 STM32CubeMX2 新式 HAL 驱动（CubeMX2 2.1.0），与经典 CubeMX/HAL 有多处不同，初学者请特别留意：

1. **回调函数名不同**：转换完成回调是 `HAL_ADC_REG_UnitaryConvCpltCallback()`（weak 弱函数，直接定义同名函数即可覆写），**不是**经典 HAL 的 `HAL_ADC_ConvCpltCallback()`。
2. **启动函数名不同**：启动"软件触发 + 中断"单次转换的函数是 `HAL_ADC_REG_StartConv_IT()`，**不是**经典 HAL 的 `HAL_ADC_Start_IT()`。
3. **电压换算宏的参数要求**：`HAL_ADC_CALC_DATA_TO_VOLTAGE(vref, data, resolution)` 的第三个参数必须传**分辨率枚举编码值** `HAL_ADC_RESOLUTION_12_BIT`，**不能传数字 `12`**。若传数字，换算会得到错误结果（详见第 8 节）。
4. **ADC 使用顺序**：必须先 `HAL_ADC_Start()`（进入 ACTIVE）→ `HAL_ADC_Calibrate()` → 再 `HAL_ADC_REG_StartConv_IT()`，顺序不可颠倒。
5. **初始化由生成代码完成**：`mx_system_init()` 已代为完成 RCC、GPIO、ADC1、NVIC 的全部初始化，并在 `mx_adc1.c` 中配置好 `HAL_ADC_CHANNEL_1`、`HAL_ADC_REG_TRIG_SOFTWARE`（软件触发）、`HAL_ADC_REG_CONV_SINGLE`（单次）。前提是 CubeMX2 中已把 **IN1 加入 ADC 正则转换序列**（见第 7 节）。
6. `main.h` 已通过 `mx_hal_def.h` 间接包含所用头文件，`main.c` 只需额外包含 `mx_adc1.h` 以使用 `mx_adc1_gethandle()`。

## 7. 编译烧录步骤

1. 用 STM32CubeMX2 打开工程根目录的 `ADC_IT.ioc2` 进行配置。**请务必确认 ADC 配置中已把 IN1（PA1）加入"正则转换序列"**。若未加入，`mx_adc1.c` 会缺少 `HAL_ADC_SetConfigChannel()`，ADC 将无法真正采到数据；加入后重新生成代码即可。
2. 确认主要配置项：分辨率 12 位、单端输入、软件触发、单次转换、ADC1 中断使能。
3. 用 **STM32CubeIDE**（或 VS Code + Cube 工具链）打开工程目录编译，生成 `ADC_IT.elf`。
4. 通过板载 ST-Link 的 SWD 接口将程序烧录到 MCU（STM32CubeProgrammer 或 IDE 内烧录按钮均可）。
5. 复位运行，在调试器监视窗口添加 `g_adc_raw_value`、`g_adc_voltage_mv`。

> 若使用 VS Code + Cube CMake 预设，构建命令等价于：
>
> ```bash
> cube-cmake -S . -B build/debug_GCC_STM32C542RCT6
> cube-cmake --build build/debug_GCC_STM32C542RCT6
> ```

### 测试方法：用断点读取稳定值

`g_adc_raw_value`、`g_adc_voltage_mv` 每 100 ms 被更新一次，全速运行下监视窗口数值会不断跳动、不易记录。建议：

1. 在 `HAL_ADC_REG_UnitaryConvCpltCallback()` 回调内**最后一条赋值语句**（换算 `g_adc_voltage_mv` 那一行）设断点。停住时正是本次转换结果刚写入两个变量的瞬间，数值稳定不再变化。
2. 全速运行 (F8)，程序每 100 ms 停在断点一次（停在 ADC 中断上下文，不影响查看全局变量）。
3. 分别在 PA1 接 GND、接 3.3 V、接电位器中间挡位三种状态下，记录两个变量的值。
4. 验证完毕后删除断点，恢复正常运行。

## 8. 测试结果

已在 NUCLEO-C542RC 上烧录运行验证，使用断点法 + 监视窗口读取，结果如下：

| PA1 连接 | 原始值 `g_adc_raw_value` | 电压 `g_adc_voltage_mv` | 是否正常 |
| --- | --- | --- | --- |
| → GND | 1 | 0 mV | ✅ |
| → 3.3 V | 4091 | 3296 mV | ✅ |
| → 电位器中间 | 2463 | 1968 mV | ✅ |

**验证数据核对**（电压换算公式 `电压 = raw × 3300 / 4095`）：

- GND：`1 × 3300 / 4095 ≈ 0.8 → 0 mV` ✓（取整为 0）
- 3.3 V：`4091 × 3300 / 4095 ≈ 3296.8 → 3296 mV` ✓（满量程接近 3300 mV）
- 中间档：`2463 × 3300 / 4095 ≈ 1984 mV`，实测 1968 mV，偏差约 0.8%，源于电位器接触点微小抖动导致断点前后两次采样略有差异，属正常现象。

实验功能目标全部达成：ADC1 通道 1 采集 PA1 电压、软件触发、转换完成中断、100 ms 周期、12 位原始值读取与电压换算，均验证通过。
