# V2P — 异构 RISC-V 芯片全流程开发验证平台

## 摘要

V2P（Virtual to Physical）是一个异构 RISC-V 芯片全流程开发验证平台。芯片包含以下四大模块：

- **AP（应用处理器四核 SMP）**
- **NPU（神经网络推理加速）**
- **FSI（功能安全岛）**
- **LSIO（低速外设子系统）**

AP 支持 Linux SMP / FreeRTOS SMP / RT-Thread SMP，NPU 与 FSI 各运行 FreeRTOS。项目采用"QEMU 虚拟原型先行 → Verilator RTL 仿真 → FPGA 物理验证收官"的三阶段模式，硬件地址映射由设备树（Linux）与固定地址表（RTOS/裸机）分别消费。目前处于调研阶段。

> **仓库划分**：各子系统可独立拆分仓库；全局 SPEC 定义由独立仓库 `spec-v2p` 统一管理

[TOC]

## 架构概览

```text
          ┌────────────────────────┐
          │   AP 子系统 (RV64 ×4)   │
          │  VexRiscv-SMP / MMU /   │
          │  PLIC / CLINT / SBI     │
          └────────────┬───────────┘
                       │
              ┌────────┴────────┐
              │  MBOX (8 通道)  │
              └────────┬────────┘
                       │
    ┌──────────┬──────────┬──────────┐
    │          │          │          │
┌───▼──┐  ┌────▼────┐  ┌──▼─────┐  ┌──▼──┐
│ NPU* │  │ FSI    │  │ LSIO   │  │ DDR │
│ RV32 │  │ RV32   │  │ 外设   │  │ 1GB │
└──────┘  └────────┘  └────────┘  └──────┘
```
> *NPU 为独立仓库管理

> DDR3 1GB + SYSTEM SRAM 704KB（共享）通过 AXI 互联连接各子系统。

### 子系统一览

| 子系统 | 处理器 | 关键特性 | 运行环境 | 仓库 |
|--------|--------|----------|----------|------|
| **AP** | **VexRiscv-SMP** RV64 ×4 | MMU、PLIC、CLINT、SBI、缓存一致性、**专属 Timer** | **Linux SMP** / FreeRTOS SMP / RT-Thread SMP | v2p-ap |
| **NPU** | VexRiscv RV32 调度核 + **Gemmini 8×8** | 硬件池化/激活、Gemmini scratchpad、128KB 私有 SRAM、**专属 Timer**、调试 UART | FreeRTOS | v2p-npu |
| **FSI** | VexRiscv RV32 | AES-256 / SHA-256 / RSA-2048 / TRNG、**专属 Timer**、专属 UART、128KB 私有 SRAM | FreeRTOS | v2p-fsi |
| **LSIO** | 无处理器 | SPI / I2C / DMA / UART(DMA) / GPIO / WDT / **Ethernet** / SD/MMC / **RTC** | 由 AP/FSI 编程控制 | v2p-soc |

## 核心理念

**Virtual First, Physical Final** — 先在 QEMU 上做虚拟原型，再迁移到 FPGA 验证。

## 核心价值

- **异构 SoC 全流程开发**：从 QEMU 虚拟设备建模到 FPGA 板级验证，覆盖 AP / NPU / FSI / LSIO 完整设计链。
- **多架构多环境软件栈**：RV64 SMP + RV32 MCU，AP 支持 Linux / FreeRTOS / RT-Thread 三套 SMP 环境。
- **AI + 安全双能力**：NPU 推理加速与 FSI 安全启动，覆盖 SoC 前沿方向。
- **低成本实践**：基于开源工具链，野火凌云 Kintex-7 开发板（二手 ~¥1,600）。

## 存储架构

| 层级 | 主要用途 |
|------|----------|
| **DDR3 (1GB)** | AP Linux 主存 + NPU 权重 |
| **SYSTEM SRAM（共享，704KB）** | 核间交换 + AP 数据 + 共享缓冲 |
| **NPU 私有 SRAM** | NPU 调度核程序与数据 |
| **Gemmini scratchpad** | 权重 / 激活 / 累加 |
| **FSI 私有 SRAM** | BootROM + 完整固件 + 堆栈/数据 |

## 地址空间布局

32-bit 地址空间，DDR（1GB）置于顶部。NPU（RV32）与 FSI（RV32）可直接访问 DDR，无需窗口或桥接机制。

**复位向量**：
- **FSI（RV32）** → FSI 私有 SRAM（BootROM 入口）
- **NPU（RV32）** → NPU 私有 SRAM（由 FSI 预加载固件）
- **AP（RV64）** → DDR（由 FSI 预加载 OpenSBI）

详细地址映射表见 `spec-v2p` 仓库。

## FPGA 平台

> **硬件**：野火凌云 — XC7K325T-2FFG900I（工业级），二手 ~¥1,600。

| 资源 | 规格 |
|------|------|
| DDR3 | 1GB |
| 存储 | SD 卡槽、SPI Flash、EEPROM |
| 显示 | LCD 接口 |
| 高速互联 | PCIe ×4、万兆光纤 |
| 网络 | 千兆以太网 |
| 调试 | JTAG、UART |
| 扩展 | SPI / I2C / GPIO |

## 三阶段开发流程

| 阶段 | 平台 | 核心任务 | 交付物 |
|------|------|----------|--------|
| **一** | QEMU | 虚拟原型 + 驱动开发 | QEMU 设备模型、镜像 |
| **二** | Verilator | RTL 仿真验证 | RTL 源码、Testbench、波形 |
| **三** | XC7K325T | FPGA 物理验证 | 比特流、BOOT.BIN、联调 |

### 阶段一：QEMU 原型

一份 `qemu-system-riscv64` 二进制，通过 `-M` 参数选择不同机器类型：

- `-M v2p` — 全 SoC 模式：AP + FSI + MBOX + LSIO 全部使能
- `-M v2p-ap` — AP 独立模式：仅 AP 子系统
- `-M v2p-fsi` — FSI 独立模式：仅 FSI 子系统
- NPU 通过 Spike 插件模拟 Gemmini 模型

### 阶段二：独立 RTL 仿真

| 子系统 | 仿真对象 |
|--------|----------|
| **AP** | VexRiscv-SMP RV64 + 缓存一致性 |
| **NPU** | Gemmini 8×8 + TileLink→AXI（`v2p-npu` 独立仿真） |
| **FSI** | VexRiscv RV32 + 安全外设 |
| **LSIO** | 各外设模块 |

### 阶段三：FPGA 物理验证

Vivado 综合与布局布线，按 FSI → AP → 核间通信 → NPU → LSIO 顺序上板验证。

## 项目目录

```
V2P/                          # 顶层元仓库（submodule 聚合）
├── dts/                      # 设备树源文件
├── qemu/                     # QEMU 设备模型（SoC 核心）
├── rtl/                      # RTL 设计（SoC 核心）
├── software/                 # SoC 公共软件
│   ├── bootrom/              # BootROM 软件设计
│   ├── uboot/                # U-Boot + OpenSBI
│   ├── linux/                # Linux SMP 内核 + 驱动
│   └── rtos/                 # RTOS 验证套件
├── spec/                     # V2P 全局 SPEC 定义（独立仓库 spec-v2p）
│   ├── qemu/                 # QEMU 设备模型 SPEC
│   ├── software/             # 软件 SDK 和验证框架 SPEC
│   ├── soc/                  # SoC 地址映射 SPEC
│   └── fpga/                 # FPGA 综合 SPEC
├── tools/                    # 编译脚本、仿真环境
└── npu/ → v2p-npu            # NPU 子系统（独立仓库 submodule）
```

> **仓库拆分策略**：`spec/` 为 `spec-v2p` 独立仓库，管理全项目 SPEC 定义；`qemu/` + `rtl/lsio/` 可归入 `v2p-soc`；`rtl/ap-sys/` 可归入 `v2p-ap`；
> `rtl/fsi-sys/` 可归入 `v2p-fsi`；`npu/` 对应独立仓库 `v2p-npu`。
> 各仓库目录结构均遵循上述模式。