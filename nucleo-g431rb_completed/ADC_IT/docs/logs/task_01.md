# 任务1：实现 ADC 中断采集 — 完成日志

**日期**：2026-07-27  
**状态**：✅ 已验证通过

---

## 任务目标

完成 ADC 校准、周期性软件触发和转换完成中断处理，每 100 ms 采集一次 PA1 引脚的模拟电压，获得 12 位 ADC 原始值。

## 修改文件

`Core/Src/main.c`

## 修改摘要

### 1. 新增全局变量 — `USER CODE BEGIN PV` (line 48–49)

```c
volatile uint32_t adc_raw_value;
volatile uint8_t  adc_data_ready;
```

- `adc_raw_value` — 存储 ADC 12 位原始值，在中断回调中写入，主循环中读取
- `adc_data_ready` — 数据就绪标志，回调中置 1，主循环处理完毕后清 0
- 均声明为 `volatile`，防止编译器优化导致主循环读到过期的寄存器缓存值

### 2. ADC 校准 — `USER CODE BEGIN 2` (line 96–99)

```c
if (HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED) != HAL_OK)
{
    Error_Handler();
}
```

- 在 `MX_ADC1_Init()` 之后执行单端模式校准
- 返回值非 `HAL_OK` 时进入 `Error_Handler()`，确保校准异常时系统不继续运行

### 3. 主循环周期触发 — `USER CODE BEGIN WHILE` (line 110–115)

```c
static uint32_t last_tick = 0;
if (HAL_GetTick() - last_tick >= 100)
{
    last_tick = HAL_GetTick();
    HAL_ADC_Start_IT(&hadc1);
}
```

- 使用 `HAL_GetTick()` 实现非阻塞 100ms 计时
- `last_tick` 声明为 `static`，在 `while` 块作用域内保留上一次触发时刻
- `HAL_GetTick() - last_tick >= 100` 采用无符号运算，自然处理 tick 溢出回绕（~49.7 天）
- 每次调用 `HAL_ADC_Start_IT()` 启动一次软件触发的单次转换（`ContinuousConvMode = DISABLE`）

### 4. 转换完成回调 — `USER CODE BEGIN 4` (line 274–283)

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        adc_raw_value = HAL_ADC_GetValue(&hadc1);
        adc_data_ready = 1;
    }
}
```

- 覆盖 HAL 库 `__weak` 默认实现
- 判断 `hadc->Instance == ADC1` 区分 ADC1/ADC2 共享中断
- 读取 12 位原始值后置位 `adc_data_ready = 1`
- 不在回调中再次调用 `HAL_ADC_Start_IT()`，保持由主循环统一管理转换节奏

## 调用流程

```
main()
  → MX_ADC1_Init()
  → HAL_ADCEx_Calibration_Start()       // 【新增】校准
     ↓ 主循环（每 100ms）
  → HAL_ADC_Start_IT()                  // 【新增】软件触发单次转换
     ↓ 转换完成 → 中断
  → ADC1_2_IRQHandler()                  // 已有
     → HAL_ADC_IRQHandler()              // 已有
        → HAL_ADC_ConvCpltCallback()    // 【新增】读取原始值、置位标志
```

## 验证结果

| 测试条件 | 预期结果 | 实际结果 |
|----------|----------|----------|
| PA1 接 GND | `adc_raw_value` ≈ 0 | ✅ 通过 |
| PA1 接 3.3V | `adc_raw_value` ≈ 4095 | ✅ 通过 |
| 回调触发周期 | 约 100ms | ✅ 通过 |

## 未修改的文件

| 文件 | 原因 |
|------|------|
| `ADC_IT.ioc` | CubeMX 配置，不修改 |
| `Core/Src/stm32g4xx_it.c` | `ADC1_2_IRQHandler` 已正确实现 |
| `Core/Src/stm32g4xx_hal_msp.c` | ADC 时钟/GPIO/NVIC 已完整 |
| `Core/Inc/stm32g4xx_it.h` | 中断声明已完整 |
| `startup_stm32g431xx.s` | 向量表已含 `ADC1_2_IRQHandler` |
| `Drivers/` 下所有文件 | HAL/CMSIS 库，不修改 |
