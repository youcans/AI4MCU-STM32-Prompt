# T08 实现方案 — 实现 MD-05 串口监测

> 编制日期：2026-08-08
> 修订日期：2026-08-25（依据 T05/T06/T07 实际执行结果和当前工程代码校订，校订版）
> 依据文件：
> - `docs/tasks/programming_task_list.md` §T08
> - `docs/design/software_detailed_design.md`
> - `docs/logs/task_log.md`（T05/T06/T07 执行记录）

---

## 1. 任务实现分析

### 1.1 当前状态

T07 已完成状态机和 500 Hz 调度并板级验证通过（2026-08-25），`MON_SendData()` 已在 `HAL_TIM_PeriodElapsedCallback` 中被调用（当前为空壳返回 false，调用无害）。本任务实现 `app_monitor.c` 中的实际串口发送逻辑。

### 1.2 数据路径

```
MD-04 HAL_TIM_PeriodElapsedCallback [app_control.c]
  ├─ 运行状态: MON_SendData(PROC_GetDacAmplitude())
  └─ 停止状态: MON_SendData(SAMP_GetTemperature())

MON_SendData() [app_monitor.c]
  → 检查 dma_busy → 建 JustFloat 帧 → BSP_StartDmaTx() → 成功则置忙

DMA1_Channel2_IRQHandler
  → DMA 传输完成处理（Normal 模式：HAL 内部使能 UART TCIE）

LPUART1_IRQHandler
  → UART TC 完成处理
    → HAL_UART_TxCpltCallback [app_monitor.c]
      → 清除 dma_busy
```

### 1.3 JustFloat 帧格式

float 数据 4 字节（little-endian）+ 帧尾 4 字节（`0x00 0x00 0x80 0x7F`，即 IEEE 754 正无穷大），共 8 字节。VOFA+ 以 JustFloat 协议解析。

### 1.4 外设配置核实（当前工程代码）

| 配置项 | 实际值 | 来源 |
|---|---|---|
| LPUART1 | 115200 bps、8N1、FIFO 禁用、时钟 PCLK1 | `Core/Src/usart.c` |
| LPUART1_TX DMA | DMA1_Channel2、**Normal 模式**、MEM→PERIPH、字节对齐 | `Core/Src/usart.c` |
| DMA 中断 | DMA1_Channel2_IRQn 已使能，优先级 2 | `Core/Src/dma.c` |
| LPUART1 中断 | LPUART1_IRQn 已使能，优先级 3（TC 完成路径必需） | `Core/Src/usart.c` |
| 发送接口 | `BSP_StartDmaTx()`（T03 实现，`HAL_UART_Transmit_DMA` 返回 `HAL_OK` 判定） | `App/Src/app_bsp.c` |

完成路径与设计 §2.2 一致：Normal DMA 完成后 HAL 使能 UART TCIE，最终由 LPUART1 ISR 的 TC 中断触发 `HAL_UART_TxCpltCallback`。

---

## 2. 前置条件及待确认事项

无。

---

## 3. 需要修改的文件

仅 `App/Src/app_monitor.c`。

- 文件头 `@brief` 更新为「MD-05 串口监测模块（T08 实现）」；
- 其余内容按 §4 实现。

---

## 4. 具体代码修改方案

`App/Src/app_monitor.c` 修改后完整内容：

```c
/**
  ******************************************************************************
  * @file    app_monitor.c
  * @brief   MD-05 串口监测模块（T08 实现）
  ******************************************************************************
  */
#include "app_monitor.h"
#include "main.h"
#include "usart.h"
#include "app_bsp.h"
#include <string.h>

/* JustFloat 帧尾 (IEEE 754 正无穷大，little-endian) */
#define JUSTFLOAT_TAIL_BYTE0  0x00
#define JUSTFLOAT_TAIL_BYTE1  0x00
#define JUSTFLOAT_TAIL_BYTE2  0x80
#define JUSTFLOAT_TAIL_BYTE3  0x7F

/* DMA 发送忙标志（TIM6 ISR 置位、LPUART1 ISR 清除，volatile 跨 ISR 共享） */
static volatile bool  dma_busy  = false;
static uint8_t        tx_frame[8];

static void MON_BuildJustFloatFrame(float value, uint8_t *frame)
{
    /* 写入 float 数据（4 字节，使用 memcpy 保证对齐安全） */
    memcpy(frame, &value, sizeof(value));

    /* 写入帧尾（4 字节） */
    frame[4] = JUSTFLOAT_TAIL_BYTE0;
    frame[5] = JUSTFLOAT_TAIL_BYTE1;
    frame[6] = JUSTFLOAT_TAIL_BYTE2;
    frame[7] = JUSTFLOAT_TAIL_BYTE3;
}

bool MON_SendData(float value)
{
    if (dma_busy) {
        return false;
    }

    MON_BuildJustFloatFrame(value, tx_frame);

    if (BSP_StartDmaTx(tx_frame, sizeof(tx_frame))) {
        dma_busy = true;
        return true;
    }

    return false;
}

bool MON_IsBusy(void)
{
    return dma_busy;
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == LPUART1) {
        dma_busy = false;
    }
}
```

**注意**：
- `dma_busy` 为跨 ISR 共享变量：TIM6 ISR（500 Hz 调用 `MON_SendData`）置位、LPUART1 ISR（TC 完成）清除，`volatile` 修饰；
- 忙时直接返回 false 丢弃当前帧（500 Hz 调用方 T07 已按返回值忽略），不清除数据；
- `tx_frame` 为静态缓冲，发送期间由 `dma_busy` 保护不被覆写；
- `BSP_StartDmaTx` 启动失败（返回 false）时不置忙，下个 500 Hz 周期自然重试。

---

## 5. 编译和验证方案

### 5.1 编译验证

Debug 配置编译通过，仅 `app_monitor.c` 变更。

### 5.2 板级验证

| 验证项 | 方法 | 预期 |
|---|---|---|
| 停止状态串口 | 上电，PC 端运行 VOFA+（JustFloat 协议，115200 bps） | 约 500 Hz 收到 float 温度数据 |
| 运行状态串口 | 按 B1 进入运行 | 数据切换为 DAC 幅值，旋转电位器时数值变化 |
| 异常状态 | 加热超过 45°C 进入 FAULT | 串口数据停止 |

---

## 6. 修改汇总

| 文件 | 修改内容 |
|---|---|
| `App/Src/app_monitor.c` | 更新文件头 `@brief`；新增 `#include "main.h"`、`#include "usart.h"`、`#include "app_bsp.h"`、`<string.h>`；新增 `dma_busy`、`tx_frame[8]` 私有变量和 `MON_BuildJustFloatFrame` 私有函数（`memcpy` 写入 float 保证对齐安全）；实现 `MON_SendData`（组帧 + DMA 启动 + 忙管理）、`MON_IsBusy`、`HAL_UART_TxCpltCallback`（清除忙标志） |

**不修改**：头文件、其他 App 模块（T07 已建立 500 Hz 调用点）、CMakeLists.txt、CubeMX 生成代码、`.ioc`、详细设计文档。

## 7. 实施顺序建议

1. 按 §4 修改 `App/Src/app_monitor.c`，经差异视图提交开发者审阅，确认后写入工程；
2. 执行 §5.1 编译验证；
3. 执行 §5.2 板级验证（开发者执行烧录与观测，VOFA+ 按 JustFloat 协议配置）；
4. 记录验证结果至 `docs/logs/task_log.md`，待批准后进入 T09。

## 8. 风险与注意事项

| 事项 | 说明 |
|---|---|
| 完成路径依赖 | Normal DMA + TC 中断路径需 LPUART1_IRQn 与 DMA1_Channel2_IRQn 均使能（已核实 usart.c/dma.c：优先级 3/2）；若 TxCpltCallback 不触发，`dma_busy` 卡死导致后续帧全部丢弃 |
| 发送吞吐 | 每帧 8 字节 @115200 ≈ 694 µs，远小于 500 Hz 周期 2 ms，稳态下逐帧完成、基本无丢帧；忙时跳过由 T07 调用方自然处理 |
| JustFloat 兼容 | 帧尾为 +inf（LE 字节序 00 00 80 7F）；float 数据经 `memcpy` 按 Cortex-M 小端写入，与 VOFA+ JustFloat 协议一致 |
| 跨 ISR 共享 | `dma_busy` 遵循设计 §2.6：`volatile` + `bool` 原子读写，单写单读，无需临界区 |
| 状态间数据源 | 停止=温度、运行=DAC 幅值、异常=不发送，数据源选择逻辑已在 T07 的 500 Hz 调度中实现，本任务不重复 |
| 修改边界 | 仅 app_monitor.c；不修改 usart.c/dma.c/.ioc；T09/T10 的整体验证不在本任务范围 |
