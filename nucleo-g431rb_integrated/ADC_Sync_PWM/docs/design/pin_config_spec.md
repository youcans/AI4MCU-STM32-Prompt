# MCU 引脚配置规范

> 编制日期：2026-08-04
> 依据文件：
> - `docs/design/mcu_resource_plan.md`（MCU 资源规划）
> - `docs/raw/hardware/NUCLEO-G431RB_Schematic_C05.pdf`（开发板原理图 C05）
> - `docs/raw/hardware/NUCLEO-G431RB_User_Manual_UM2505.pdf`（开发板用户手册）
> - `docs/raw/references/STM32G431_Datasheet_DS12589.pdf`（芯片数据手册）
> 说明：本规范列出需要占用或保留的外部引脚及其配置。片内温度传感器、定时器内部触发事件、中断和 DMA 等不占用外部引脚的资源不列入本表。

---

## 1. 引脚配置

| 引脚 | 功能 | 模式 | 关联资源 | 备注 |
|---|---|---|---|---|
| PA8 | TIM1_CH1 | AF 推挽 | TIM1 | PWM 第一组主通道，10 kHz 中心对齐 |
| PB13 | TIM1_CH1N | AF 推挽 | TIM1 | PWM 第一组互补通道，与 PA8 带死区互补输出 |
| PA9 | TIM1_CH2 | AF 推挽 | TIM1 | PWM 第二组主通道，预设占空比 |
| PB14 | TIM1_CH2N | AF 推挽 | TIM1 | PWM 第二组互补通道，与 PA9 带死区互补输出 |
| PA10 | TIM1_CH3 | AF 推挽 | TIM1 | PWM 第三组主通道，预设占空比 |
| PB15 | TIM1_CH3N | AF 推挽 | TIM1 | PWM 第三组互补通道，与 PA10 带死区互补输出 |
| PA1 | ADC1_IN2 | 模拟输入 | ADC1 | 注入组 Rank1，连接 10 kΩ 电位器滑动端（电位器高端接 3.3 V，低端接 GND），驱动 PWM 占空比和 DAC 输出 |
| PB1 | ADC1_IN12 | 模拟输入 | ADC1 | 注入组 Rank2，杜邦线接高电平参考电压，当前不参与动态计算 |
| PB0 | ADC1_IN15 | 模拟输入 | ADC1 | 注入组 Rank3，杜邦线接 GND，当前不参与动态计算 |
| PA4 | DAC1_OUT1 | 模拟输出 | DAC1 | 运行状态输出 250 Hz 方波，仅接高阻抗测量设备 |
| PA2 | LPUART1_TX | AF 推挽 | LPUART1 | 串口数据发送，通过板载 SB17 连接 STLINK-V3E 虚拟串口 |
| PA3 | LPUART1_RX | AF 输入 | LPUART1 | 串口数据接收，通过板载 SB23 连接 STLINK-V3E 虚拟串口，本项目不使用 |
| PC13 | GPIO_EXTI13 | 外部中断 | EXTI | 板载用户按键 B1（外部下拉，按下接通高电平），上升沿触发 |
| PA5 | GPIO_Output | 推挽输出 | GPIO | 板载用户 LED LD2（高电平点亮），初始低电平（熄灭） |
| PA13 | SYS_SWDIO | SWD | SWD | 调试接口数据线，板载 STLINK-V3E，**不得配置为应用 GPIO** |
| PA14 | SYS_SWCLK | SWD | SWD | 调试接口时钟线，板载 STLINK-V3E，**不得配置为应用 GPIO** |

---

## 2. 引脚冲突检查

### 2.1 引脚复用冲突

以上 16 个外部引脚的功能分配不重叠，每个引脚仅被分配一个外设功能，无引脚复用冲突。

### 2.2 板载电路冲突

| 引脚 | 板载连接 | 冲突评估 |
|---|---|---|
| PA2 / PA3 | STLINK-V3E VCP，通过 SB17（PA2）、SB23（PA3）使能 | 无冲突。PA2 和 PA3 的 LPUART1 功能通过板载 solder bridge 连接至 VCP |
| PC13 | 用户按键 B1（外部下拉） | 无冲突。外部中断输入上升沿触发，与板载按键电路匹配 |
| PA5 | 用户 LED LD2 | 无冲突。GPIO 推挽输出直接驱动 LED |
| PA13 / PA14 | STLINK-V3E SWD | 已保留为调试接口，不分配给应用功能 |
| PB0 / PB1 | 无板载有源电路占用 | 无冲突。可用作 ADC 输入。外部杜邦线连接至参考电平时需确保与板上其他功能不短路 |

### 2.3 调试接口

PA13（SWDIO）和 PA14（SWCLK）已保留为 SWD 调试接口，不得分配给其他应用功能。复位后这两个引脚默认处于 SWD 模式，如果在 STM32CubeMX 中禁用调试功能会导致芯片无法通过 SWD 连接，本项目不执行此操作。

### 2.4 其他

- ADC1_IN16（内部温度传感器）为芯片内部通道，不占用外部引脚，与外部 ADC 通道无冲突。
- TIM6 为内部基本定时器，TRGO 输出至 ADC1 规则组为芯片内部触发路由，不占用外部引脚。
- DMA1 和 DMAMux 为芯片内部资源，不涉及引脚分配。

---

## 3. 待确认事项

| 编号 | 待确认内容 | 说明 |
|---|---|---|
| PIN-TBD-01 | VREF+ 供电配置 | 本项目模拟输入最高为 3.3 V，需要确认 JP8 置于 [2-3]，由 VDD 为 VREF+ 供电；如果继续使用板载 VREF 供电，则需要相应调整模拟输入范围 |
