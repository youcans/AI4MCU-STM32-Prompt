# 任务2：实现输入电压换算 — 完成日志

**日期**：2026-07-27  
**状态**：✅ 已完成

---

## 任务目标

在主循环中检测 ADC 数据就绪标志，根据 12 位 ADC 原始值计算 PA1 引脚的输入电压，并通过调试器观察换算结果。

## 修改文件

`Core/Src/main.c`

## 修改摘要

### 1. 新增电压变量 — `USER CODE BEGIN PV` (line 50)

```c
float adc_voltage;
```

- 存储换算后的输入电压值（单位：V）
- 仅在主循环中写入，不在 ISR 中访问，无需 `volatile`

### 2. 主循环数据处理 — `USER CODE BEGIN 3` (line 119–126)

```c
if (adc_data_ready == 1)
{
    adc_data_ready = 0;
    uint32_t raw_local = adc_raw_value;
    adc_voltage = raw_local * 3.3f / 4096.0f;
    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
}
```

- **先清零标志** `adc_data_ready = 0`，尽早将数据所有权从 ISR 转移到主循环，ISR 可立即写入下一轮数据
- **局部副本** `raw_local` 保存原始值，避免后续计算期间被 ISR 修改
- **电压换算** `Vin = raw × 3.3 / 4096`（参考电压 VDDA = 3.3V，12 位分辨率 = 4096 级）
- **LED 翻转** 每次数据就绪翻转 LD2（PA5），直观确认约 100ms 周期采集正常运行

## 实现逻辑

```
主循环（每 100ms）
  → HAL_ADC_Start_IT()              // 任务1：触发单次转换
     ↓ 转换完成
  → HAL_ADC_ConvCpltCallback()      // 任务1：adc_data_ready = 1
     ↓ 主循环下一轮
  → adc_data_ready == 1 ?
     → adc_data_ready = 0           // 【新增】清零标志
     → raw_local = adc_raw_value    // 【新增】取副本
     → adc_voltage = raw × 3.3/4096 // 【新增】电压换算
     → HAL_GPIO_TogglePin(LD2)      // 【新增】LED 翻转
```

## 验证方式

- 编译烧录后通过调试器观察 `adc_voltage`：
  - PA1 接 GND → 约 0.000V
  - PA1 接 3.3V → 约 3.300V
  - PA1 接中间电平（如电阻分压）→ 与实际值接近
- LD2 约每 100ms 翻转一次，确认周期性采集正常运行

## 未修改的文件

| 文件 | 原因 |
|------|------|
| `ADC_IT.ioc` | CubeMX 配置，不修改 |
| `Core/Src/stm32g4xx_it.c` | 中断已由任务1完成，无需修改 |
| `Core/Src/stm32g4xx_hal_msp.c` | 无需修改 |
| `Core/Inc/stm32g4xx_it.h` | 无需修改 |
| `startup_stm32g431xx.s` | 无需修改 |
| `Drivers/` 下所有文件 | HAL/CMSIS 库，不修改 |
