<div align="center">

# 🤖 RoboMotion-FPGA

**基于 FPGA 的全向移动机械臂控制系统**

![HDL](https://img.shields.io/badge/HDL-Verilog-1572B6?style=flat-square&logo=verilog&logoColor=white) ![FPGA](https://img.shields.io/badge/FPGA-Xilinx-E01F27?style=flat-square&logo=xilinx&logoColor=white) ![IDE](https://img.shields.io/badge/IDE-Vivado-029FCE?style=flat-square) ![License](https://img.shields.io/badge/License-Educational-success?style=flat-square) [![Modules](https://img.shields.io/badge/Verilog_模块-56-1572B6?style=flat-square)](verilog/) [![XDC](https://img.shields.io/badge/约束文件-2-red?style=flat-square)](constrain/)

[English](README.md) &nbsp;·&nbsp; [系统架构](#-系统架构) &nbsp;·&nbsp; [核心模块](#-核心模块) &nbsp;·&nbsp; [UART 协议](#-uart-协议) &nbsp;·&nbsp; [快速开始](#-快速开始)

</div>

---

## 📖 项目概述

> **RoboMotion-FPGA** 是一个面向 Xilinx FPGA 平台的完整 Verilog HDL 机器人控制系统。工程将全向四轮移动底盘、六路舵机机械臂子系统、传感器与显示外设、以及多路 UART 指令接口集成到统一的顶层硬件设计中——专为自主移动抓取应用而构建。

---

## ✨ 项目亮点

<table>
<tr>
<td width="33%" valign="top">

<h3 align="center">🚗<br/>移动<br/>底盘</h3>
<p>4 路直流电机驱动，集成编码器反馈、PWM 输出，支持速度 PID 闭环控制。</p>

</td>
<td width="33%" valign="top">

<h3 align="center">🧮<br/>运动学<br/>引擎</h3>
<p>支持 <strong>麦克纳姆轮</strong>、<strong>全向四轮</strong>、<strong>全向三轮</strong> 三种运动模型，可通过 UART 实时切换。</p>

</td>
<td width="33%" valign="top">

<h3 align="center">🦾<br/>机械臂<br/>控制</h3>
<p>6 路舵机 PWM 输出，基于 <strong>CORDIC 算法的逆运动学解算</strong>，用于机械臂定位与夹爪控制。</p>

</td>
</tr>
<tr>
<td width="33%" valign="top">

<h3 align="center">📡<br/>多协议<br/>UART</h3>
<p>蓝牙控制、机器视觉输入、配置成功回传、FSM 指令解析——全部通过可配置的 UART 通道实现。</p>

</td>
<td width="33%" valign="top">

<h3 align="center">🌡️<br/>传感器<br/>套件</h3>
<p>DHT11 温湿度采集、基于 UART 的测距输入（×3）、按键输入、74HC595 数码管显示——环境感知与信息输出一体化。</p>

</td>
<td width="33%" valign="top">

<h3 align="center">🔧<br/>开发者<br/>友好</h3>
<p>模块化层级结构清晰，顶层集成简洁明了——易于理解、便于扩展。</p>

</td>
</tr>
</table>

---

## 🏗️ 系统架构

```mermaid
flowchart LR
  classDef external fill:#f8fafc,stroke:#94a3b8,color:#0f172a;
  classDef top fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e;
  classDef core fill:#ecfdf5,stroke:#059669,color:#064e3b;
  classDef output fill:#fff7ed,stroke:#ea580c,color:#7c2d12;
  classDef feedback fill:#fefce8,stroke:#ca8a04,color:#713f12;

  subgraph IN["外部接口"]
    command_in["指令输入<br/>蓝牙 UART / MV UART / 急停"]
    sensor_in["传感输入<br/>测距 UART / DHT11 / 按键"]
    enc["电机反馈<br/>编码器 A/B x4"]
  end

  robot_top["robot_top<br/>系统集成顶层"]

  subgraph CHASSIS["motor_chip_top - 底盘控制"]
    chassis_uart["uart_ctrl<br/>速度与参数解析"]
    wheel_ctrl["wheel_ctrl<br/>运动学与电机目标"]
    motor_driver["dc_motor_driver_top x4<br/>PID + 编码器 + PWM"]
  end

  subgraph ARM["arm_top - 六路舵机机械臂控制"]
    arm_uart["uart_arm_mv / uart_arm_ble<br/>目标点与动作解析"]
    arm_motion["motion<br/>CORDIC 逆运动学"]
    servo_pwm["steer_pwm x6<br/>舵机脉宽生成"]
  end

  subgraph DISPLAY["disp_top - 传感与显示"]
    sensing["distance x3 + dht11_ctrl<br/>传感器采样"]
    display["hex_top + hc595_driver<br/>数码管显示"]
    guard["distance_en / direction_en<br/>底盘辅助信号"]
  end

  command_in --> robot_top
  sensor_in --> robot_top
  enc --> robot_top

  robot_top --> chassis_uart
  chassis_uart --> wheel_ctrl
  wheel_ctrl --> motor_driver
  motor_driver --> motors["M1-M4<br/>H 桥 + PWM"]
  enc -. 反馈 .-> motor_driver
  chassis_uart --> uart_tx["uart_tx<br/>配置 ACK"]

  robot_top --> arm_uart
  arm_uart --> arm_motion
  arm_motion --> servo_pwm
  servo_pwm --> servos["steera-steerf<br/>舵机 PWM"]

  robot_top --> sensing
  sensing --> display
  display --> panel["shcp / stcp / ds"]
  sensing --> guard -. 安全反馈 .-> wheel_ctrl

  class command_in,sensor_in external;
  class robot_top top;
  class chassis_uart,wheel_ctrl,motor_driver,arm_uart,arm_motion,servo_pwm,sensing,display,guard core;
  class motors,uart_tx,servos,panel output;
  class enc,guard feedback;
```

---

## 🧱 核心模块

| 子系统 | 顶层模块 | 关键文件 |
|:---|:---|:---|
| 🧩 系统集成 | [`robot_top.v`](verilog/rtl/top/robot_top.v) | 连接底盘、机械臂、显示、传感器与 UART 通道 |
| ⚙️ 底盘控制 | [`motor_chip_top.v`](verilog/rtl/top/motor_chip_top.v) | [`wheel_ctrl.v`](verilog/rtl/motion/wheel_ctrl.v)、[`dc_motor_driver_top.v`](verilog/rtl/top/dc_motor_driver_top.v)、[`uart_ctrl.v`](verilog/rtl/comm/uart_ctrl.v) |
| 🔄 轮系运动学 | [`wheel_ctrl.v`](verilog/rtl/motion/wheel_ctrl.v) | [`Mcknum_wheel_calculate.v`](verilog/rtl/motion/Mcknum_wheel_calculate.v)、[`all_direction_wheel_four_calculate.v`](verilog/rtl/motion/all_direction_wheel_four_calculate.v)、[`all_direction_wheel_three_calculate.v`](verilog/rtl/motion/all_direction_wheel_three_calculate.v) |
| 🔌 电机闭环驱动 | [`dc_motor_driver_top.v`](verilog/rtl/top/dc_motor_driver_top.v) | [`controller.v`](verilog/rtl/motor/controller.v)、[`controller_PID.v`](verilog/rtl/motor/controller_PID.v)、[`encoder.v`](verilog/rtl/motor/encoder.v)、[`pwm_generate.v`](verilog/rtl/motor/pwm_generate.v) |
| 🦾 机械臂控制 | [`arm_top.v`](verilog/rtl/top/arm_top.v) | [`motion.v`](verilog/rtl/motion/motion.v)、[`steer_pwm.v`](verilog/rtl/motor/steer_pwm.v)、[`uart_arm_ble.v`](verilog/rtl/comm/uart_arm_ble.v)、[`uart_arm_mv.v`](verilog/rtl/comm/uart_arm_mv.v) |
| 📐 CORDIC 运算 | [`motion.v`](verilog/rtl/motion/motion.v) | [`cordic_sincos.v`](verilog/rtl/math/cordic_sincos.v)、[`cordic_arctan.v`](verilog/rtl/math/cordic_arctan.v)、[`cordic_arccos.v`](verilog/rtl/math/cordic_arccos.v)、[`cordic_arctanh.v`](verilog/rtl/math/cordic_arctanh.v) |
| 📟 传感与显示 | [`disp_top.v`](verilog/rtl/top/disp_top.v) | [`dht11_ctrl.v`](verilog/rtl/peripheral/dht11_ctrl.v)、[`distance.v`](verilog/rtl/peripheral/distance.v)、[`hc595_driver.v`](verilog/rtl/peripheral/hc595_driver.v)、[`hex_top.v`](verilog/rtl/top/hex_top.v) |

---

## 📡 UART 协议

> 💡 **提示：** 配置指令成功后，UART 将返回 `Set Successful!`。
>
> 当前工程同时使用 ASCII 关键字命令和固定二进制帧两种解析方式。

| 类别 | 指令 | 功能 | 示例 / 范围 |
|:---|:---|:---|:---|
| 🔧 系统 | `baud+<n>` | 设置 UART 波特率预设 | `baud+0` – `baud+4` |
| 🚗 运动 | `wheel+<n>` | 选择轮系运动模型 | `wheel+0` 麦克纳姆，`wheel+1` 全向四轮，`wheel+2` 全向三轮 |
| ⚙️ 参数 | `gr+<value>` | 设置减速比 | `gr+30` |
| ⚙️ 参数 | `Ppr+<value>` | 设置编码器 PPR | `Ppr+13` |
| ⚙️ 参数 | `alen+<mm>` | 设置底盘尺寸 A | `alen+200` |
| ⚙️ 参数 | `blen+<mm>` | 设置底盘尺寸 B | `blen+150` |
| 🎮 控制 | `55 A5 sx x sy y sz z F0` | 底盘速度控制帧，`sx/sy/sz` 为符号位，`x/y/z` 为幅值 | `sx/sy/sz`：`00` 正、`01` 负 |
| 💃 动作 | `55 C5 mode F0` | 预设舞蹈控制帧 | `mode=01` 启动，`mode=00` 清零 / 复位 |
| 🦾 机械臂 BLE | `DD EE ... FF` | 机械臂动作帧，控制观察 / 运动 / 收拢 / 放置 / 夹取 | 由 [`uart_arm_ble.v`](verilog/rtl/comm/uart_arm_ble.v) 解析 |
| 🦾 机械臂 MV | `AA BB ... CC` | 视觉目标帧，携带颜色、带符号 X/Y 与 theta | 由 [`uart_arm_mv.v`](verilog/rtl/comm/uart_arm_mv.v) 解析 |

---

## 📁 工程结构

```text
RoboMotion-FPGA/
├── verilog/                          📦 56 个硬件源文件
│   ├── rtl/                          🔧 RTL 设计源码
│   │   ├── top/                      🧩 系统顶层（robot_top、arm_top、motor_chip_top ……）
│   │   ├── comm/                     📨 UART 通信（uart_*、fsm_baud）
│   │   ├── motion/                   🔄 运动学与运动控制
│   │   ├── motor/                    🔌 电机驱动、PID、编码器、PWM
│   │   ├── math/                     📐 CORDIC、乘法器、除法器
│   │   ├── peripheral/               🌡️ 传感器、显示、定时器、按键滤波
│   │   └── protocol/                 📋 FSM 协议解析（fsm_gr、fsm_ppr ……）
│   ├── filelists/                    📋 历史文件列表（当前为 Windows 绝对路径）
│   ├── tb/                           🧪 测试平台目录（当前为空）
│   └── sim/                          📊 仿真目录（当前为空）
├── constrain/                        📏 管脚约束
│   ├── robot_top_constrain.xdc
│   └── disp_top_constrain.xdc
└── README.md / README_CN.md          📖 说明文档
```

<details>
<summary><b>🔍 展开完整硬件拓扑</b></summary>

```text
robot_top
├── disp_top
│   ├── hex_top / hc595_driver
│   ├── dht11_ctrl
│   ├── distance ×3
│   ├── uart_ble
│   └── switch_key_filter
├── motor_chip_top
│   ├── wheel_ctrl
│   │   ├── Mcknum_wheel_calculate
│   │   ├── all_direction_wheel_four_calculate
│   │   ├── all_direction_wheel_three_calculate
│   │   └── dc_motor_driver_top ×4
│   │       ├── encoder / encoder_AB_detect
│   │       ├── controller / controller_PID / controller_Speed_loop
│   │       ├── pwm_generate
│   │       └── dc_motor_driver
│   └── uart_ctrl
│       ├── fsm_gr / fsm_ppr / fsm_wheel / fsm_a_len / fsm_b_len / fsm_baud
│       ├── uart_cmd / dance_cmd
│       ├── multiplier / divider
│       └── uart_data_tx
└── arm_top
    ├── motion
    │   ├── cordic_sincos
    │   ├── cordic_arctan
    │   └── cordic_arccos
    ├── steer_pwm ×6
    ├── uart_arm_mv
    └── uart_arm_ble
```

</details>

---

## 🛠️ 开发环境

| 工具 | 用途 |
|:---|:---|
| ![Vivado](https://img.shields.io/badge/Xilinx_Vivado-综合与实现-029FCE?style=flat-square) | 综合、实现、时序分析与比特流生成 |
| ![FPGA](https://img.shields.io/badge/Xilinx_FPGA-目标器件-E01F27?style=flat-square) | 目标硬件平台，具体管脚分配见 `.xdc` 约束文件 |

---

## 🚀 快速开始

| 步骤 | 操作 |
|:---:|:---|
| **1** | 在 **Xilinx Vivado** 中创建新工程 |
| **2** | 添加 [`verilog/rtl/`](verilog/rtl/) 目录下的全部 Verilog 源文件 |
| **3** | 添加 [`constrain/`](constrain/) 目录下的 XDC 约束文件 |
| **4** | 如果当前工具流无法直接使用 [`verilog/filelists/`](verilog/filelists/) 中的 Windows 绝对路径文件列表，请先重新生成或替换 |
| **5** | 将 [`robot_top.v`](verilog/rtl/top/robot_top.v) 设置为 **系统顶层模块** |
| **6** | 依次运行 **综合 → 实现 → 生成 Bitstream** |
| **7** | 将 bitstream 下载到 FPGA 开发板并上电运行 |

---

## 📊 项目统计

| 指标 | 数量 |
|:---|:---:|
| Verilog 模块 | **56** |
| XDC 约束文件 | **2** |
| CORDIC 运算器 | **4** |
| 运动学模型 | **3** |
| UART 接口 | **4+** |
| 电机通道 | **4** |
| 舵机通道 | **6** |

---

## 📜 许可

本项目仅供 **教育与研究** 使用。

---

<div align="center">

[⬆ 回到顶部](#-robomotion-fpga)

</div>
