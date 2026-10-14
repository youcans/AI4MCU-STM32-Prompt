# ADC1 通道2 中断采集 — 实现任务清单

> 基于工程分析报告 [docs/analysis/overview.md](../analysis/overview.md)。

---

## 任务1：实现 ADC 中断采集

### 任务目标
完成 ADC 校准、周期性软件触发和转换完成中断处理，每 100 ms 采集一次 PA1 引脚的模拟电压，获得 12 位 ADC 原始值。

### 涉及文件
- `Core/Src/main.c`

### 主要修改位置

| 位置 | 标记 | 内容 |
|------|------|------|
| line 47–48 | `USER CODE BEGIN PV` | 新增 `volatile` 全局变量 |
| line 94–96 | `USER CODE BEGIN 2` | ADC 校准 |
| line 103 | `USER CODE BEGIN WHILE` | 每 100ms 调用 `HAL_ADC_Start_IT()` |
| line 265–267 | `USER CODE BEGIN 4` | 实现 `HAL_ADC_ConvCpltCallback()` |

### 实施要点

1. **新增变量**（`USER CODE BEGIN PV`）：
   - `volatile uint32_t adc_raw_value` — ADC 原始值。
   - `volatile uint8_t adc_data_ready` — 数据就绪标志。
   - `float adc_voltage` — 换算后的输入电压。

2. **ADC 校准**（`USER CODE BEGIN 2`）：
   - 在 `MX_ADC1_Init()` 后调用 `HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED)`。
   - 返回值非 `HAL_OK` 时进入 `Error_Handler()`。

3. **中断回调**（`USER CODE BEGIN 4`）：
   - 实现 `void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)`，覆盖 HAL 库 `__weak` 默认实现。
   - 判断 `hadc->Instance == ADC1`。
   - 调用 `adc_raw_value = HAL_ADC_GetValue(&hadc1)` 读取 12 位原始值。
   - 置位 `adc_data_ready = 1`。
   - 不在回调中再次调用 `HAL_ADC_Start_IT()`。

4. **主循环周期触发**（`USER CODE BEGIN WHILE`）：
   - 用 `HAL_GetTick()` 计时，每 100ms 调用一次 `HAL_ADC_Start_IT(&hadc1)`。
   - 不在初始化阶段单独启动转换，统一由主循环管理节奏。

### 验证方式
- 编译通过，下载到 NUCLEO-G431RB。
- 在 `HAL_ADC_ConvCpltCallback()` 内设置断点，确认约每 100ms 进入一次回调。
- 调试器中观察 `adc_raw_value`：
  - PA1 接 GND → 接近 0。
  - PA1 接 3.3V → 接近 4095。

---

## 任务2：实现输入电压换算

### 任务目标
根据 ADC 原始值计算 PA1 引脚的输入电压，并通过调试器观察换算结果。

### 涉及文件
- `Core/Src/main.c`

### 主要修改位置

| 位置 | 标记 | 内容 |
|------|------|------|
| line 47–48 | `USER CODE BEGIN PV` | 新增全局变量 `float adc_voltage` |
| line 103 | `USER CODE BEGIN WHILE` | 在数据就绪时执行电压换算 |

### 实施要点
1. 在主循环中检查 `adc_data_ready == 1` 时：
   - 取出 `adc_raw_value` 的本地副本后立即清零 `adc_data_ready = 0`，防止竞态。
   - 按公式 `voltage = raw_local * 3.3f / 4096.0f` 换算。
2. 可选用板载 LED（LD2, PA5）在每次数据就绪时翻转，直观确认周期性采集正在运行。

### 验证方式
- 编译通过，下载到开发板。
- 调试器中观察 `voltage` 变量约每 100ms 更新一次：
  - PA1 接 GND → 约 0.000V。
  - PA1 接 3.3V → 约 3.300V。
  - PA1 接中间电平（如电阻分压）→ 与实际值接近。

---

## 任务依赖

```
任务1 (ADC中断采集) → 任务2 (电压换算)
```

任务1 提供了完整可验证的原始值采集链路，任务2 在此基础上叠加换算逻辑。

## 不修改的文件

| 文件 | 原因 |
|------|------|
| `ADC_IT.ioc` | CubeMX 配置 |
| `Core/Src/stm32g4xx_it.c` | `ADC1_2_IRQHandler` 已完整 |
| `Core/Src/stm32g4xx_hal_msp.c` | ADC 时钟/GPIO/NVIC 已完整 |
| `Core/Inc/stm32g4xx_it.h` | 中断声明已完整 |
| `startup_stm32g431xx.s` | 向量表已含 `ADC1_2_IRQHandler` |
| `Drivers/` 下所有文件 | HAL/CMSIS 库，不修改 |
