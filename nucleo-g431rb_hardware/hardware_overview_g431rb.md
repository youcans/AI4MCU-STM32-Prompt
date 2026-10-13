# NUCLEO-G431RB 硬件概述

## 1. 硬件信息

### 1.1 开发板信息

- 开发板型号：NUCLEO-G431RB
- 板卡编号：MB1367
- 板卡类型：STM32 Nucleo-64
- MCU封装：LQFP64
- 扩展接口：
  - Arduino Uno V3 接口
  - ST morpho 扩展接口
- 调试接口：
  - ST-LINK/V3E
  - SWD 调试接口
  - Virtual COM Port

### 1.2 MCU信息

- MCU型号：STM32G431RBT6
- MCU系列：STM32G4
- CPU核心：Arm Cortex-M4 with FPU and DSP
- 最大主频：170 MHz
- Flash容量：128 KB
- SRAM容量：32 KB
- 工作电压：1.71 V ~ 3.6 V
- 工作温度：-40 °C ~ +85 °C
- 数学加速器：
  - CORDIC
  - FMAC

主要内部资源：

- GPIO：
  - 多功能复用GPIO
  - 外部中断支持
- ADC：
  - ADC1
  - ADC2
  - 12-bit ADC
- DAC：
  - DAC1
- 定时器：
  - TIM1
  - TIM2
  - TIM3
  - TIM4
  - TIM6
  - TIM7
  - TIM8
  - TIM15
  - TIM16
  - TIM17
  - LPTIM
- 通信接口：
  - USART/UART
  - LPUART
  - SPI
  - I2C
  - FDCAN
  - USB FS
- DMA：
  - DMA控制器
- 模拟资源：
  - COMP
  - OPAMP


### 1.3 板载资源

| 资源 | MCU引脚 | STM32功能 | 用途 |
|---|---|---|---|
| LD2 用户LED | PA5 | GPIO | LED输出 |
| B1 用户按键 | PC13 | GPIO/EXTI | 用户输入 |
| B2 Reset按键 | NRST | Reset | MCU复位 |
| ST-LINK Virtual COM TX | PA2 | LPUART1_TX | 调试串口发送 |
| ST-LINK Virtual COM RX | PA3 | LPUART1_RX | 调试串口接收 |
| SWDIO | PA13 | SWDIO | 程序下载与调试 |
| SWCLK | PA14 | SWCLK | 程序下载与调试 |

说明：

- ST-LINK/V3E Virtual COM Port 默认通过 LPUART1 与目标 MCU 通信。
- PA2/PA3 同时支持其他串口复用功能，实际使用由 STM32CubeMX 配置确定。


## 2. MCU引脚与接口分配

说明：

- 本表根据 NUCLEO-G431RB 用户手册、I/O assignment 和 STM32G431RB 数据手册整理。
- MCU主要复用功能用于说明芯片能力。
- Nucleo默认连接用于说明开发板实际硬件连接。
- Arduino接口表示默认Arduino Uno V3映射。
- 可选焊桥修改后的连接关系不作为默认配置。


| MCU引脚 | MCU主要复用功能 | Arduino接口 | ST Morpho接口 | Nucleo默认连接 | 备注 |
|---|---|---|---|---|---|
| PA0 | GPIO / ADC1_IN1 / TIM2_CH1 | A0 | CN7 | | 模拟输入、定时器 |
| PA1 | GPIO / ADC1_IN2 / TIM2_CH2 | A1 | CN7 | | 模拟输入、定时器 |
| PA2 | GPIO / ADC1_IN3 / LPUART1_TX / USART2_TX | - | CN7 | ST-LINK VCP TX | 默认连接LPUART1 |
| PA3 | GPIO / ADC1_IN4 / LPUART1_RX / USART2_RX | - | CN7 | ST-LINK VCP RX | 默认连接LPUART1 |
| PA4 | GPIO / ADC12_IN17 / DAC1_OUT1 / SPI1_NSS | A2 | CN7 | | SPI片选、模拟输出 |
| PA5 | GPIO / ADC12_IN13 / SPI1_SCK / TIM2_CH1 | D13 | CN7 | LD2 LED | SPI时钟与LED共用 |
| PA6 | GPIO / ADC2_IN3 / SPI1_MISO / TIM3_CH1 / TIM16_CH1 / TIM8_BKIN | D12 | CN7 | | SPI输入 |
| PA7 | GPIO / ADC2_IN4 / SPI1_MOSI / TIM3_CH2 / TIM17_CH1 / TIM8_CH1N | D11 | CN7 | | SPI输出 |
| PA8 | GPIO / TIM1_CH1 | D7 | CN7 | | PWM输出 |
| PA9 | GPIO / USART1_TX / TIM1_CH2 | D8 | CN7 | | USART/PWM |
| PA10 | GPIO / USART1_RX / TIM1_CH3 | D2 | CN7 | | USART输入 |
| PA11 | GPIO / USB_DM | D10 | CN10 | | USB数据 |
| PA12 | GPIO / USB_DP | D9 | CN10 | | USB数据 |
| PA13 | GPIO / SWDIO | - | CN4 | ST-LINK | 调试接口 |
| PA14 | GPIO / SWCLK | - | CN4 | ST-LINK | 调试接口 |
| PA15 | GPIO / SPI1_NSS | D10 | CN10 | | JTAG复用 |
| PB0 | GPIO / ADC1_IN15 / TIM3_CH3 | A3 | CN7 | | 模拟输入/PWM |
| PB1 | GPIO / ADC1_IN12 / TIM3_CH4 | - | CN7 | | 模拟输入/PWM |
| PB2 | GPIO / ADC2_IN12 | - | CN10 | | 模拟输入 |
| PB3 | GPIO / SPI1_SCK | D3 | CN10 | | JTAG复用 |
| PB4 | GPIO / SPI1_MISO / JTRST | D5 | CN10 | | 默认具有JTAG复位功能 |
| PB5 | GPIO / SPI1_MOSI | D4 | CN10 | | SPI输出 |
| PB6 | GPIO / I2C1_SCL / USART1_TX | D10 | CN9 | | I2C/USART复用 |
| PB7 | GPIO / I2C1_SDA / USART1_RX | D9 | CN9 | | I2C/USART复用 |
| PB8 | GPIO / I2C1_SCL / TIM4_CH3 | D15 | CN9 | | I2C复用 |
| PB9 | GPIO / I2C1_SDA / TIM4_CH4 | D14 | CN9 | | I2C复用 |
| PB10 | GPIO / USART3_TX | - | CN9 | | USART |
| PB11 | GPIO / USART3_RX | - | CN9 | | USART |
| PB12 | GPIO / SPI2_NSS | - | CN10 | | SPI |
| PB13 | GPIO / SPI2_SCK | - | CN10 | | SPI |
| PB14 | GPIO / SPI2_MISO | - | CN10 | | SPI |
| PB15 | GPIO / SPI2_MOSI | - | CN10 | | SPI |
| PC0 | GPIO / ADC1_IN6 | A5 | CN7 | | 模拟输入 |
| PC1 | GPIO / ADC1_IN7 | A4 | CN7 | | 模拟输入 |
| PC2 | GPIO / ADC1_IN8 | - | CN10 | | 模拟输入 |
| PC3 | GPIO / ADC1_IN9 | - | CN10 | | 模拟输入 |
| PC4 | GPIO / ADC1_IN5 / USART1_TX | - | CN10 | Arduino D1 | 默认串口发送 |
| PC5 | GPIO / ADC1_IN11 / USART1_RX | - | CN10 | Arduino D0 | 默认串口接收 |
| PC6 | GPIO / USART6_TX | - | CN9 | | USART |
| PC7 | GPIO / USART6_RX | - | CN9 | | USART |
| PC8 | GPIO / TIM8_CH3 | - | CN9 | | 定时器 |
| PC9 | GPIO / TIM8_CH4 | - | CN9 | | 定时器 |
| PC10 | GPIO / USART3_TX | - | CN9 | | USART |
| PC11 | GPIO / USART3_RX | - | CN9 | | USART |
| PC12 | GPIO / USART3_TX | - | CN9 | | USART |
| PC13 | GPIO / EXTI | - | CN10 | USER Button | 用户按键 |
| PC14 | GPIO / LSE_IN | - | CN10 | | 低速时钟 |
| PC15 | GPIO / LSE_OUT | - | CN10 | | 低速时钟 |
| PF0 | GPIO / HSE_IN | - | CN9 | | 高速时钟 |
| PF1 | GPIO / HSE_OUT | - | CN9 | | 高速时钟 |

### 引脚使用注意事项

- PA5 同时连接 LD2 LED、Arduino D13 和 SPI1_SCK，使用 SPI1 或高速采样时需要注意板载LED电路影响。
- PB4 默认具有 JTRST 功能，作为普通GPIO或外设复用功能使用前需要释放JTAG功能。
- PA13、PA14 默认用于 SWD 调试，不建议基础实验中作为普通GPIO使用。
- I2C1 支持 PB6/PB7 和 PB8/PB9 两组复用引脚，实际工程只能选择其中一组，由 STM32CubeMX 配置确定。

## 3. MCU外设资源

### 3.1 GPIO资源

STM32G431RBT6 提供多功能GPIO资源，支持：

- 输入模式；
- 输出模式；
- 外部中断模式；
- 外设复用模式。

常用开发板GPIO资源：

| 资源 | MCU引脚 | 用途 |
|---|---|---|
| LD2 用户LED | PA5 | GPIO输出 |
| USER Button | PC13 | GPIO输入/EXTI |
| Arduino接口GPIO | 多个GPIO | 外部扩展实验 |
| ST Morpho接口GPIO | 多个GPIO | 外部扩展实验 |

注意：

- GPIO复用功能由STM32CubeMX配置确定。
- PA13、PA14用于SWD调试时，不建议作为普通GPIO使用。


### 3.2 ADC/DAC/模拟资源

STM32G431RBT6包含丰富的模拟资源。

| 外设 | 资源 | 用途 |
|---|---|---|
| ADC1 | 12-bit ADC，多通道 | 模拟采样实验 |
| ADC2 | 12-bit ADC，多通道 | 模拟采样实验 |
| DAC1 | DAC_OUT1 | 模拟输出实验 |
| COMP | 高速比较器 | 模拟比较实验 |
| OPAMP | 运算放大器 | 模拟信号处理实验 |

常用ADC输入：

| MCU引脚 | ADC通道 | Arduino接口 |
|---|---|---|
| PA0 | ADC1_IN1 | A0 |
| PA1 | ADC1_IN2 | A1 |
| PA4 | ADC12_IN17 | A2 |
| PB0 | ADC1_IN15 | A3 |
| PC1 | ADC1_IN7 | A4 |
| PC0 | ADC1_IN6 | A5 |

说明：

- 同一GPIO可能支持多个ADC复用通道。
- ADC1、ADC2的具体选择由STM32CubeMX配置确定。


### 3.3 定时器资源

STM32G431RBT6包含高级控制定时器、通用定时器和低功耗定时器。

| 定时器 | 类型 | 主要用途 |
|---|---|---|
| TIM1 | 高级控制定时器 | PWM、电机控制、互补PWM |
| TIM8 | 高级控制定时器 | PWM、高级控制应用 |
| TIM2 | 通用定时器 | 定时、输入捕获、PWM |
| TIM3 | 通用定时器 | PWM、输入捕获 |
| TIM4 | 通用定时器 | PWM、输入捕获 |
| TIM6 | 基本定时器 | 基本定时 |
| TIM7 | 基本定时器 | 基本定时 |
| TIM15 | 通用定时器 | 定时、PWM |
| TIM16 | 通用定时器 | 定时、输入捕获 |
| TIM17 | 通用定时器 | 定时、输入捕获 |
| LPTIM | 低功耗定时器 | 低功耗计时 |

常用定时器资源：

| 功能 | 引脚 | 定时器 |
|---|---|---|
| LD2 LED PWM | PA5 | TIM2_CH1 |
| PWM输出 | PA8 | TIM1_CH1 |
| PWM输出 | PA6 | TIM3_CH1 |
| PWM输出 | PA7 | TIM3_CH2 |

说明：

- TIM1、TIM8适用于高级PWM和电机控制实验。
- 定时器通道复用关系由STM32CubeMX配置确定。


### 3.4 通信资源

STM32G431RBT6提供多种通信接口。

#### LPUART/USART

| 外设 | 引脚 | 开发板连接 | 用途 |
|---|---|---|---|
| LPUART1 | PA2/PA3 | ST-LINK Virtual COM | 默认调试串口 |
| USART1 | PC4/PC5 | Arduino D1/D0 | Arduino默认串口 |
| USART1 | PA9/PA10 | Arduino接口 | 外部串口通信 |
| USART3 | PB10/PB11、PC10/PC11 | ST Morpho | 外部通信 |
| USART6 | PC6/PC7 | ST Morpho | 外部通信 |

说明：

- NUCLEO-G431RB 的 ST-LINK Virtual COM 默认连接 LPUART1。
- PA2/PA3 支持 USART2 复用功能，但不是默认Virtual COM接口。


#### SPI

| 外设 | 引脚 | 功能 | 注意 |
|---|---|---|---|
| SPI1 | PA4 | NSS | 片选 |
| SPI1 | PA5 | SCK | 与LD2 LED共用 |
| SPI1 | PA6 | MISO | 数据输入 |
| SPI1 | PA7 | MOSI | 数据输出 |
| SPI2 | PB12 | NSS | SPI接口 |
| SPI2 | PB13 | SCK | SPI接口 |
| SPI2 | PB14 | MISO | SPI接口 |
| SPI2 | PB15 | MOSI | SPI接口 |

说明：

- SPI1默认使用PA4~PA7。
- PA5同时连接LD2 LED和Arduino D13，使用SPI1时需注意硬件影响。


#### I2C

| 外设 | 引脚 | 功能 |
|---|---|---|
| I2C1 | PB6/PB7 | SCL/SDA |
| I2C1 | PB8/PB9 | SCL/SDA |

说明：

- I2C1支持两组复用引脚。
- 同一工程只能选择其中一组SCL/SDA。
- 实际配置由STM32CubeMX确定。


#### FDCAN

| 外设 | 用途 |
|---|---|
| FDCAN1 | CAN通信实验 |

FDCAN具体引脚由STM32CubeMX复用配置确定。


#### USB

| 外设 | 引脚 | 功能 |
|---|---|---|
| USB FS | PA11/PA12 | USB通信 |


### 3.5 系统资源

#### 时钟资源

| 资源 | 信息 |
|---|---|
| HSE | 24 MHz板载高速晶振 |
| LSE | 32.768 kHz板载低速晶振 |
| HSI | MCU内部高速时钟 |

#### 调试资源

| 资源 | 引脚 | 用途 |
|---|---|---|
| SWDIO | PA13 | 程序下载与调试 |
| SWCLK | PA14 | 程序下载与调试 |

#### DMA资源

STM32G431RBT6包含DMA控制器，用于：

- ADC数据传输；
- UART/LPUART数据收发；
- SPI数据传输；
- 定时器事件处理。

DMA通道和请求映射由STM32CubeMX配置确定。


#### EXTI资源

STM32G431支持GPIO外部中断。

典型开发板资源：

| 资源 | 引脚 | 用途 |
|---|---|---|
| USER Button | PC13 | EXTI输入 |


#### RTC资源

支持：

- 实时时钟；
- 低功耗计时。

常用时钟源：

- LSE 32.768 kHz；
- LSI内部低速时钟。


## 4. 硬件资源文件

| 文件 | 用途 |
|---|---|
| NUCLEO-G431RB_User_Manual_UM2505.pdf | 开发板用户手册，包含开发板介绍、连接关系和I/O assignment |
| NUCLEO-G431RB_Schematic_c05.pdf | 开发板原理图，用于确认板载资源和硬件连接 |
| MB1367-G431RB-C05_BOM.xlsx | 开发板元器件清单 |

后续可根据需要增加其他硬件资源文件。
