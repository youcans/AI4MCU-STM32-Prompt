# 二次开发任务清单：新增 OLED 本地显示

---

## 0. 文档说明

本清单为在 ADC_Sync_PWM_v2 工程基础上新增 OLED 本地显示功能（[FR-10 / SF-12，见 software_spec_SE_v2.md](software_spec_SE_v2.md)）的代码开发与验证任务拆分，任务自 TS01 起编号。

- 依据：[software_spec_SE_v2.md](software_spec_SE_v2.md)（含已确定的 OLED 硬件配置）、[second_develop_analysis.md](second_develop_analysis.md)（可行性分析中的实现条件与注意事项）。
- **不列入本清单的已完成项**：需求分析、工程分析、CubeMX 配置与初始工程确认。
- 本清单只规定任务目标，不规定具体修改方案与尚未确定的代码实现细节。
- 遵守 [CLAUDE.md](../CLAUDE.md) 关于应用代码入用户代码区、保持原命名/缩进/组织风格、每项修改经编译/烧录/运行验证等规则。

---

## 1. 任务清单

### TS01: OLED 显示基础建立

建立 OLED 显示模块，完成 SSD1306 基础驱动和显示所需数据接口，并通过固定测试内容验证 OLED 能够正常初始化和显示。

- **目标**：建立 OLED 显示模块，完成 SSD1306 基础驱动和显示所需数据接口；通过固定测试内容验证 OLED 能够正常初始化和显示。
- **硬件依据**：I2C1 / PB8-I2C1_SCL / PB9-I2C1_SDA / 默认地址 0x3C（[software_spec_SE_v2.md §5.1.1、§5.2.1](software_spec_SE_v2.md)）。
- 依据：[second_develop_analysis.md §2、§3、§5、§6](second_develop_analysis.md)。

### TS02: OLED 显示功能实现

在显示基础上，实现系统状态、芯片温度和主要运行数据的显示与刷新，接入现有运行流程，并验证新增显示功能正常运行且不影响原有功能。

- **目标**：在显示基础上，实现系统状态、芯片温度和主要运行数据的显示与刷新，接入现有运行流程；验证新增 OLED 显示功能正常，并确认原有主要功能运行正常。
- 依据：[software_spec_SE_v2.md SF-12、§4.2](software_spec_SE_v2.md)、[second_develop_analysis.md §3、§4](second_develop_analysis.md)。

---

## 2. 依赖关系

```
TS01 ─▶ TS02
```

TS02 在 TS01 显示基础建立后进行。
