# STM32CubeMX 工程配置清单

> 编制日期：2026-08-08
> 修订日期：2026-08-08（处理待确认参数，确定时钟方案与外设参数）
> 依据文件：
> - `docs/design/software_function_spec.md`（软件功能说明书）
> - `docs/design/pin_config_spec.md`（MCU 引脚配置规范）
> - `docs/design/runtime_state_analysis.md`（运行流程与状态切换分析）
> 说明：本文档用于指导 STM32CubeMX 实际配置操作。已确认的参数可直接逐项配置，标注"待确认"的参数需开发者在配置前确定。

---

## 1. MCU 与工程基本设置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| MCU 型号 | STM32G431RBT6 | software_function_spec.md：NUCLEO-G431RB 开发板 |
| 工程名称 | ADC_Sync_PWM | 工程文件：ADC_Sync_PWM.ioc |
| 工程路径 | 待确认（开发者指定） | 需在 CubeMX 配置前确定本地工程目录 |

---

## 2. 系统与时钟配置

### 2.1 调试接口（Pinout & Configuration → System Core → SYS）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Debug | Serial Wire | pin_config_spec.md §2.3：PA13/PA14 保留为 SWD 调试接口，不得分配给其他应用功能；不得禁用调试功能 |

### 2.2 时基源

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Timebase Source | SysTick | 常规配置：使用 SysTick 作为 HAL 时基源，无需占用额外定时器 |

### 2.3 系统时钟

#### 2.3.1 RCC 配置（Pinout & Configuration → System Core → RCC）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| HSE | Crystal/Ceramic Resonator | NUCLEO-G431RB 板载 24 MHz HSE 晶振（X3） |
| LSE | Disable | 本项目不使用低频时钟 |

#### 2.3.2 Clock Configuration

在 Clock Configuration 页面配置以下时钟路径：

**HSE = 24 MHz → PLL Source = HSE → SYSCLK Source = PLLCLK → SYSCLK = 160 MHz**

PLL 倍频/分频参数由 CubeMX 根据目标 SYSCLK = 160 MHz 自动计算。配置完成后核对以下时钟树输出值：

| 时钟节点 | 目标值 | 说明 |
|---|---|---|
| HCLK | 160 MHz | AHB 总线时钟 |
| APB1 Timer Clocks | 160 MHz | TIM6 时钟源（APB1 prescaler = 1 时，定时器时钟 = APB1 总线时钟） |
| APB2 Timer Clocks | 160 MHz | TIM1 时钟源（APB2 prescaler = 1 时，定时器时钟 = APB2 总线时钟） |
| LPUART1 Clock Mux | PCLK1 | LPUART1 时钟源选择 APB1 外设时钟（PCLK1 = 160 MHz） |
| LPUART1 频率 | 160 MHz | CubeMX 自动计算波特率分频参数，115200 bps |

---

## 3. GPIO 配置

### 3.1 用户 LED（PA5）

| 配置项 | 配置值 | 对应引脚 | 设计依据 |
|---|---|---|---|
| GPIO Mode | Output Push Pull | PA5 | pin_config_spec.md §1：PA5 GPIO_Output，推挽输出 |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down | PA5 | 常规配置：推挽输出驱动 LED，无需上下拉 |
| Maximum Output Speed | Low | PA5 | 常规配置：LED 指示无高速翻转要求 |
| User Label | LD2 | PA5 | software_function_spec.md §3.2：`LedOutput`，板载用户 LED LD2 |
| Initial Output Level | Low（熄灭） | PA5 | software_function_spec.md §3.2：初始低电平（熄灭） |

### 3.2 用户按键（PC13）

| 配置项 | 配置值 | 对应引脚 | 设计依据 |
|---|---|---|---|
| GPIO Mode | External Interrupt Mode with Rising edge trigger detection | PC13 | software_function_spec.md §5.1：PC13，外部中断线 13，上升沿触发；板载外部下拉电阻 |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down | PC13 | pin_config_spec.md §2.2：板载外部下拉，与板载按键电路匹配（外部已有下拉，不配置内部上拉） |
| User Label | B1 | PC13 | software_function_spec.md §3.1：`UserButtonEvent`，板载用户按键 B1 |

---

## 4. TIM1 与 PWM 配置（Pinout & Configuration → Timers → TIM1）

### 4.1 Mode 设置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Clock Source | Internal Clock | software_function_spec.md §5.1 |
| Channel1 | PWM Generation CH1 CH1N | PA8 / PB13 | software_function_spec.md §6.1：PWM 第一组带死区互补输出 |
| Channel2 | PWM Generation CH2 CH2N | PA9 / PB14 | software_function_spec.md §6.1：PWM 第二组带死区互补输出 |
| Channel3 | PWM Generation CH3 CH3N | PA10 / PB15 | software_function_spec.md §6.1：PWM 第三组带死区互补输出 |
| Channel4 | PWM Generation No Output | — | 产生内部 OC4REF 信号，通过 TRGO2 路由至 ADC1 注入组触发，不占用外部 GPIO |

### 4.2 Parameter Settings

#### 4.2.1 计数器参数

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Prescaler (PSC) | 0 | TIM1 时钟 = 160 MHz，不分频 |
| Counter Mode | Center-aligned mode（中心对齐） | software_function_spec.md §5.1 |
| Counter Period (ARR) | 8000 | 中心对齐模式：fPWM = 160 MHz / (2 × 8000) = 10 kHz |
| Internal Clock Division (CKD) | No Division | 常规配置 |
| Repetition Counter | 0 | 常规配置：每次更新事件均产生更新中断 |
| Auto-reload preload | Enable | 使能 ARR 预装载 |

#### 4.2.2 PWM Generation Channel 1 ~ 3

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| CH1 Mode | PWM mode 1 | 常规配置 |
| CH1 Pulse (CCR1) | 0（初始值，运行时由 MD-03 动态更新） | 初始安全值，确保启动瞬间输出最低占空比 |
| CH2 Mode | PWM mode 1 | 常规配置 |
| CH2 Pulse (CCR2) | 0（CubeMX 初始值，实际占空比由 MD-04 在 TIM1 启动前通过 MD-01 写入） | software_function_spec.md TBD-10 |
| CH3 Mode | PWM mode 1 | 常规配置 |
| CH3 Pulse (CCR3) | 0（CubeMX 初始值，实际占空比由 MD-04 在 TIM1 启动前通过 MD-01 写入） | software_function_spec.md TBD-10 |

#### 4.2.3 Channel 4（ADC 触发）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| CH4 Mode | PWM mode 1 | 产生内部 OC4REF 信号 |
| CH4 Pulse (CCR4) | 4000 | 每个 10 kHz PWM 周期产生一次同步触发 |
| CH4 Output Compare Preload | Enable | 使能 CCR4 预装载 |
| TRGO2 | OC4REF signal | CH4 内部 OC4REF 信号通过 TRGO2 输出 |
| ADC1 注入组 External Trigger | Timer 1 Trigger Out event 2（TIM1_TRGO2） | TRGO2 上升沿触发 ADC1 注入组，每个 PWM 周期一次同步采样 |
| ADC1 注入组 Trigger Edge | Rising | TRGO2 上升沿触发 |

### 4.3 死区配置（Break And Dead Time management - Output Configuration）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Automatic Output State | Disable | 常规配置 |
| DeadTime Preload | Disable | 常规配置 |
| Dead Time | 32 | TIM1 时钟 160 MHz、CKD = No Division → tDTS = 6.25 ns，200 ns / 6.25 ns = 32 |

### 4.4 PWM 输出极性

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| CH1 输出极性 | High | 常规配置：主输出高电平有效 |
| CH1N 输出极性 | High | 互补输出高电平有效，由死区插入保证互补关系 |
| CH2 输出极性 | High | 同上 |
| CH2N 输出极性 | High | 同上 |
| CH3 输出极性 | High | 同上 |
| CH3N 输出极性 | High | 同上 |

### 4.5 刹车配置（Break And Dead Time Management）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Break State | Disable | 本项目温度异常保护由软件实现（MD-04 状态机），不使用定时器硬件刹车 |
| Break2 State | Disable（如果 CubeMX 显示该项） | 同上 |

---

## 5. ADC1 配置（Pinout & Configuration → Analog → ADC1）

### 5.1 Mode 设置

在 Mode 页面启用以下通道：

| 配置项 | 配置值 | 用途 |
|---|---|---|
| IN2 | Single-ended | ADC1 注入组 Rank1（PA1），动态模拟电压 |
| IN12 | Single-ended | ADC1 注入组 Rank2（PB1），高电平参考 |
| IN15 | Single-ended | ADC1 注入组 Rank3（PB0），低电平参考 |
| Temperature Sensor Channel | Enable | ADC1 规则组（ADC1_IN16），芯片温度采样 |

### 5.2 Parameter Settings

#### 5.2.1 ADCs_Common_Settings

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Mode | Independent mode | 常规配置 |

#### 5.2.2 ADC_Settings

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Clock Prescaler | HCLK/4（同步时钟） | SYSCLK=160 MHz → ADC 时钟 = 40 MHz |
| Resolution | 12 bits | software_function_spec.md §5.1 |
| Data Alignment | Right alignment | 与 12 位右对齐的 DAC 格式一致 |
| Scan Conversion Mode | Enable | 注入组多通道顺序转换 |
| Continuous Conversion Mode | Disable | 注入组由 Timer 1 Trigger Out event 2（TIM1_TRGO2） 触发，规则组由 TIM6_TRGO 触发 |
| Discontinuous Conversion Mode | Disable | 每次触发完成全部通道转换 |
| DMA Continuous Requests | Disable | 不使用 DMA |
| End of Conversion Selection | End of Sequence of Conversion | 序列结束时产生 EOC 标志 |
| Analog Watchdog | Disable | 温度异常判断由软件实现 |

#### 5.2.3 ADC_Regular_ConversionMode（规则组：温度采样）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Number of Conversions | 1 | 规则组单通道 |
| External Trigger Source | Timer 6 Trigger Out event | TIM6_TRGO 触发 |
| External Trigger Polarity | Rising edge | — |
| Regular Discontinuous Mode | Disable | 仅 1 个通道 |
| Rank 1 Channel | ADC1_IN16 (Temperature Sensor Channel) | 内部温度传感器 |
| Rank 1 Sampling Time | 247.5 ADC cycles | 内部温度传感器通道，参考 DS12589 推荐 |

#### 5.2.4 ADC_Injected_ConversionMode（注入组：三通道同步采样）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| External Injected Conversions | Enabled | 使能注入组外部触发转换 |
| Number Of Conversions | 3 | 注入组三通道 |
| External Trigger Source | Timer 1 Trigger Out event 2（TIM1_TRGO2） | TIM1 CH4 OC4REF → TRGO2 路由 |
| External Trigger Conversion Edge | Trigger detection on the rising edge | TRGO2 上升沿触发 |
| Injected Conversion Mode | None | 每次触发完成全部 3 个通道转换，不分段 |
| Rank 1 Channel | ADC1_IN2（PA1） | 动态模拟电压 |
| Rank 1 Sampling Time | 24.5 ADC cycles | 外部电位器信号 |
| Rank 2 Channel | ADC1_IN12（PB1） | 高电平参考 |
| Rank 2 Sampling Time | 24.5 ADC cycles | 外部杜邦线参考电平 |
| Rank 3 Channel | ADC1_IN15（PB0） | 低电平参考 |
| Rank 3 Sampling Time | 24.5 ADC cycles | 外部杜邦线 GND |

---

## 6. TIM6 配置

### 6.1 基本配置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Clock Source | Internal Clock | software_function_spec.md §5.1：TIM6 需要内部时钟源 |
| TIM6 时钟 | 160 MHz | §2.3：APB1 定时器时钟 = 160 MHz |
| Prescaler (PSC) | 159 | 160 MHz / (159+1) = 1 MHz 计数时钟 |
| Counter Mode | Up | software_function_spec.md §5.1：向上计数模式 |
| Auto-Reload (ARR) | 1999 | 1 MHz / (1999+1) = 500 Hz 更新频率 |
| 触发频率 | 500 Hz（周期 2 ms） | software_function_spec.md §5.1：500 Hz 时基 |

### 6.2 触发输出配置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Trigger Event Selection | Update Event | software_function_spec.md §5.1：更新事件作为 TRGO 输出 |
| TRGO 去向 | ADC1 规则组触发 | software_function_spec.md §5.1：TIM6_TRGO → ADC1 规则组触发 |

### 6.3 NVIC Settings

在 TIM6 配置页面进入 NVIC Settings 选项卡，勾选：

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| TIM6 global interrupt, DAC1 and DAC3 channel underrun interrupts | Enable | runtime_state_analysis.md §3.1：TIM6 更新中断 → MD-04 执行 500 Hz 调度 |

---

## 7. DAC1 配置（Pinout & Configuration → Analog → DAC1）

### 7.1 Mode 设置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| OUT1 mode | Connected to external pin only | PA4 / DAC1_OUT1，外部缓冲通道 |
| OUT2 mode | Disable | 未使用 DAC Channel 2 |
| External Trigger | 不勾选 | DAC 由 MD-04 在 TIM6 更新中断中软件写入 DHR 寄存器 |

### 7.2 Configuration（Parameter Settings）

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Mode Selected | Normal Mode | 常规配置 |
| Output Buffer | Enable | software_function_spec.md §5.1：外部缓冲通道 |
| Trigger | None | 不使用硬件触发，250 Hz 方波由软件翻转实现 |

---

## 8. LPUART1 与 DMA 配置

### 8.1 LPUART1 基本配置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Mode | Asynchronous | software_function_spec.md §5.1：LPUART1，异步模式 |
| Baud Rate | 115200 bit/s | software_function_spec.md §5.1：115200 bit/s |
| Word Length | 8 Bits | JustFloat 二进制浮点帧，8 位数据位 |
| Parity | None | VOFA+ 数据流无需奇偶校验 |
| Stop Bits | 1 | 常规配置：1 位停止位 |

### 8.2 引脚配置

| 配置项 | 配置值 | 对应引脚 | 设计依据 |
|---|---|---|---|
| LPUART1_TX | 使能 | PA2 | software_function_spec.md §6.1：LPUART1_TX，串口数据发送 |
| LPUART1_RX | 使能（已配置，不使用） | PA3 | software_function_spec.md §6.1：LPUART1_RX 已配置，保持既有引脚设计 |

### 8.3 DMA 配置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| DMA 外设 | DMA1 | software_function_spec.md §5.1：LPUART1 + DMA1 |
| DMA 请求 | LPUART1_TX | 内存→外设，由 CubeMX DMAMux 自动分配通道 |
| DMA 通道 | 由 CubeMX DMAMux 自动分配，配置后核对 | 配置完成后记录分配的通道号 |
| DMA Mode | Normal | 每 500 Hz 事件触发一次 8 字节单次传输，不使用循环模式 |
| DMA Data Width (Memory) | Byte | JustFloat 帧 8 字节，逐字节传输 |
| DMA Data Width (Peripheral) | Byte | LPUART 数据寄存器为 8 位 |
| DMA Increment (Memory) | Enable | 内存地址递增，遍历 8 字节帧缓冲区 |
| DMA Increment (Peripheral) | Disable | 外设地址固定（LPUART TDR 寄存器） |
| DMA Priority | Low | 仅一个 DMA 通道，无竞争 |

---

## 9. NVIC 中断配置

进入 Pinout & Configuration → System Core → NVIC，在 NVIC 选项卡中勾选以下中断并设置优先级：

### 9.1 中断使能与优先级

| 中断入口 | 使能 | 抢占优先级 | 子优先级 | 设计依据 |
|---|---|---|---|---|
| ADC1_2 global interrupt | Enable | 0（最高） | 0 | 10 kHz 实时路径（注入组 JEOS），执行时间须远小于 100 µs |
| TIM6 global interrupt, DAC1 and DAC3 channel underrun interrupts | Enable | 1 | 0 | 500 Hz 调度，低于 ADC1_2 以避免打断 10 kHz 实时处理 |
| LPUART1 global interrupt | Enable | 2 | 0 | LPUART1 外设中断，与 DMA 同为最低优先级 |
| EXTI15_10 global interrupt | Enable | 2 | 0 | 按键事件为异步低频事件 |
| DMA1 Channel x（LPUART1_TX） | Enable | 2 | 0 | DMA 发送完成清除忙标志 |

> **优先级设计约束**：
> - ADC1_2 服务于 10 kHz 实时路径，设为最高优先级（抢占优先级 0）
> - ADC1 注入组可中断规则组转换（STM32G4 硬件支持），注入组和规则组的完成中断共用同一 NVIC 入口，靠软件标志区分

### 9.2 中断入口实现约束

| 约束项 | 说明 | 设计依据 |
|---|---|---|
| ADC1_2 中断区分 | 注入组序列完成（JEOS）和规则组转换完成（EOC）通过事件标志区分，不可分别设置 NVIC 优先级 | software_function_spec.md §7.1 |

---

## 10. 工程生成设置

### 10.1 工具链与 IDE

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| Toolchain / IDE | CMake | project_context.md：VS Code、STM32CubeIDE for VS Code、CMake 工具链、GCC 编译器 |
| Compiler / Linker | GCC | 同上 |

### 10.2 HAL 库配置

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| 固件包 | STM32CubeG4（使用已确定的固件版本） | 本项目继续使用已确定的 STM32CubeG4 固件版本 |
| LL 库使用 | 不使用 LL 库（仅使用 HAL） | HAL 库满足本项目需求，不混合使用 LL 库 |
| 外设初始化代码 | 按外设分别生成 .c/.h | module_design.md：MD-01 负责外设底层初始化，分文件便于模块化 |

### 10.3 代码生成选项

| 配置项 | 配置值 | 设计依据 |
|---|---|---|
| 复制 HAL 库文件 | 仅复制必要的库文件 | 常规配置：CubeMX 默认行为 |
| USER CODE BEGIN/END 保留 | Enable（必须） | CLAUDE.md：只能在 USER CODE BEGIN/END 之间添加用户代码 |

---

## 11. 引脚配置汇总

以下为 CubeMX 中需要配置的功能引脚，供逐项核对（未使用引脚保持 CubeMX 默认状态即可，不逐一列出）：

| 引脚 | 功能 | CubeMX Mode | 备注 |
|---|---|---|---|
| PA1 | ADC1_IN2 | ADC1_IN2 (Injected) | 注入组 Rank1，模拟输入（电位器滑动端） |
| PA2 | LPUART1_TX | LPUART1_TX | AF 推挽，串口数据发送 |
| PA3 | LPUART1_RX | LPUART1_RX | AF 输入，已配置不使用 |
| PA4 | DAC1_OUT1 | DAC1_OUT1 | 模拟输出，250 Hz 方波 |
| PA5 | GPIO_Output | GPIO_Output | 用户 LED LD2 |
| PA8 | TIM1_CH1 | TIM1_CH1 | PWM 第一组主通道 |
| PA9 | TIM1_CH2 | TIM1_CH2 | PWM 第二组主通道 |
| PA10 | TIM1_CH3 | TIM1_CH3 | PWM 第三组主通道 |
| PA13 | SYS_SWDIO | SYS_SWDIO | 调试接口（不得配置为 GPIO） |
| PA14 | SYS_SWCLK | SYS_SWCLK | 调试接口（不得配置为 GPIO） |
| PB0 | ADC1_IN15 | ADC1_IN15 (Injected) | 注入组 Rank3，模拟输入（GND 参考） |
| PB1 | ADC1_IN12 | ADC1_IN12 (Injected) | 注入组 Rank2，模拟输入（高电平参考） |
| PB13 | TIM1_CH1N | TIM1_CH1N | PWM 第一组互补通道 |
| PB14 | TIM1_CH2N | TIM1_CH2N | PWM 第二组互补通道 |
| PB15 | TIM1_CH3N | TIM1_CH3N | PWM 第三组互补通道 |
| PC13 | GPIO_EXTI13 | GPIO_EXTI13 | 用户按键 B1，上升沿触发 |

---

## 12. 后续程序编写阶段确定

以下参数不属于 CubeMX 配置范畴，应在后续程序编写时确定：

| 参数 | 说明 | 确定阶段 |
|---|---|---|
| 第二组 PWM 预设占空比 (CCR2) | CubeMX 初始 Pulse = 0，实际占空比由 MD-04 在 TIM1 启动前通过 MD-01 写入 | 软件详细设计 / 编码阶段 |
| 第三组 PWM 预设占空比 (CCR3) | CubeMX 初始 Pulse = 0，实际占空比由 MD-04 在 TIM1 启动前通过 MD-01 写入 | 软件详细设计 / 编码阶段 |
| PWM 占空比换算公式、限幅范围 | 由 MD-03 实时处理模块在 JEOS 中断中计算 | 软件详细设计 / 编码阶段 |
| DAC 换算公式、幅值范围 | 由 MD-03 实时处理模块计算，MD-04 在 TIM6 中断中使用 | 软件详细设计 / 编码阶段 |
| 温度异常阈值 | 用于 MD-04 温度异常判断（SF-08） | 软件详细设计 / 编码阶段 |
| 温度恢复阈值 | 用于 MD-04 温度异常判断（SF-08） | 软件详细设计 / 编码阶段 |
| 按键消抖时间 | EXTI13 中断中由软件实现消抖 | 软件详细设计 / 编码阶段 |
| 温度传感器换算公式 | 使用 ADC1_IN16 采样值和内部温度传感器工厂校准数据换算温度值 | 软件详细设计 / 编码阶段 |
| DMA 发送忙状态管理 | MD-05 维护忙标志，DMA 完成中断中清除 | 软件详细设计 / 编码阶段 |

---

## 13. 配置后核对事项

CubeMX 配置完成后，应对以下由 CubeMX 自动生成的内容进行核对：

| 核对项 | 核对内容 |
|---|---|
| 时钟树 | 确认 HCLK = 160 MHz、APB1 Timer Clocks = 160 MHz、APB2 Timer Clocks = 160 MHz、ADC Clock = 40 MHz |
| DMA 通道分配 | 记录 LPUART1_TX 的 DMAMux 通道号及对应 DMA 中断入口名称 |
| TIM1_CH4 | 确认 CH4 已配置为 PWM Generation No Output 模式，TRGO2 = OC4REF signal |
| ADC1 触发源 | 确认注入组 External Trigger = Timer 1 Trigger Out event 2（TIM1_TRGO2）；规则组 External Trigger = Timer 6 Trigger Out |
| NVIC | 确认 ADC1_2、TIM6_DAC、LPUART1、EXTI15_10、DMA1 Channel x 中断均已使能，优先级为 0/1/2/2/2 |
| GPIO 复用 | 核对各功能引脚的 AF 编号与 STM32G431 数据手册一致 |
| 引脚 | 确认 PA13/PA14 仅配置为 SWD，未分配给应用功能 |

---

> **配置状态汇总**：所有 CubeMX 配置参数均已确定。系统时钟 160 MHz（HSE 24 MHz → PLL），TIM1 10 kHz（Center Aligned，PSC=0, ARR=8000），TIM6 500 Hz（PSC=159, ARR=1999），死区 200 ns，ADC1 时钟 HCLK/4，采样时间 IN2/IN12/IN15 = 24.5 cycles、IN16 = 247.5 cycles，ADC 注入组由 Timer 1 Trigger Out event 2（TIM1_TRGO2） 触发（CH4 PWM Generation No Output，CCR4=4000，OC4REF → TRGO2）。仅工程路径一项需开发者在 CubeMX 配置前指定。
