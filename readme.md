# V2P — 异构 RISC-V 芯片全流程开发验证平台

## 摘要

V2P（Virtual to Physical）是一个异构 RISC-V 芯片全流程开发验证平台。芯片包含以下四大模块：

- **AP（应用处理器四核 SMP）**
- **NPU（神经网络推理加速）**
- **FSI（功能安全岛）**
- **LSIO（低速外设子系统）**

AP 支持 Linux SMP / FreeRTOS SMP / RT-Thread SMP，NPU 与 FSI 各运行 FreeRTOS。项目采用"QEMU 虚拟原型先行 → FPGA 物理验证收官"的两阶段模式。目前处于调研阶段。

> **仓库划分**：各子系统可独立拆分仓库，详见[项目目录](#项目目录)。

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
              │  核间通信 (待定)  │
              └────┬───────┬───┘
                   │       │
              ┌────▼──┐ ┌──▼──────┐
              │ NPU*  │ │ FSI     │
              │ RV32  │ │ RV32    │
              └───────┘ └─────────┘
                       │
              ┌────────┴────────┐
              │  AXI Interconnect │
              ├───────┬────┬────┤
              │ LSIO  │DDR │SRAM│
              │ 外设   │1GB │672K│
              └───────┴────┴────┘
```
> *NPU 为独立仓库管理。DDR3 1GB + SYSTEM SRAM 672KB + BootROM 32KB 通过 AXI 互联连接各子系统。

### 子系统一览

| 子系统 | 处理器 | 关键特性 | 运行环境 | 仓库 |
|--------|--------|----------|----------|------|
| **AP** | **VexRiscv-SMP** RV64 ×4 | MMU、PLIC、CLINT、SBI、缓存一致性、**专属 Timer** | **Linux SMP** / FreeRTOS SMP / RT-Thread SMP | v2p-ap |
| **NPU** | **Coral NPU**（内置 RV32IMF_Zve32x） | 标量核 + 向量单元(Zve32x) + 矩阵 MAC 脉动阵列、ITCM 8KB / DTCM 32KB、**专属 Timer**、调试 UART | FreeRTOS | v2p-npu |
| **FSI** | VexRiscv RV32 | BootROM 启动、AES-256 / SHA-256 / RSA-2048 / TRNG、**专属 Timer**、专属 UART、128KB 私有 SRAM | FreeRTOS | v2p-fsi |
| **LSIO** | 无处理器 | SPI / I2C / DMA / UART(DMA) / GPIO / WDT / **Ethernet** / SD/MMC / **RTC** | 由 AP/FSI 编程控制 | v2p-soc |

## 核心理念

**Virtual First, Physical Final** — QEMU 虚拟原型先行，FPGA 物理验证收官。

- **异构 SoC 全流程**：QEMU → FPGA，覆盖 AP / NPU / FSI / LSIO
- **多架构软件栈**：RV64 SMP + RV32 MCU，Linux / FreeRTOS / RT-Thread
- **AI + 安全**：NPU 推理加速 + FSI 安全启动
- **低成本**：全开源工具链，野火凌云 Kintex-7（二手 ~¥1,600）

## 存储架构

| 层级 | 容量 | 主要用途 |
|------|------|----------|
| **DDR3** | 1GB | AP Linux 主存 + NPU 权重 |
| **BootROM** | 32KB | FSI 启动代码（只读，位流固化） |
| **SYSTEM SRAM（共享）** | 672KB | 核间交换 + 共享缓冲 |
| **NPU ITCM / DTCM** | 8KB / 32KB | Coral 指令 / 数据紧耦合内存 |
| **FSI 私有 SRAM** | 128KB | FSI 固件与加密数据 |

## FPGA 平台

**野火凌云** — XC7K325T（326,080 逻辑单元），二手 ~¥1,600。

| 资源 | 规格 |
|------|------|
| DDR3 | 1GB |
| 存储 | SD 卡槽、32Mb SPI Flash、8KB EEPROM |
| 显示 | LCD 接口 |
| 高速互联 | PCIe ×4、万兆光纤 |
| 网络 | 千兆以太网 |
| 调试 | JTAG、UART |
| 扩展 | SPI / I2C / GPIO |

## 关键设计决策

### Coral NPU：一体化 AI 子系统

Coral NPU（[Apache 2.0](https://github.com/google/coral-npu)）将 **标量核（RV32IMF）+ 向量单元（Zve32x 256-bit）+ 矩阵 MAC 脉动阵列** 三合一，单核运行 FreeRTOS。

- **一体化架构**：免去独立调度核，单一固件直接管理标量/向量/矩阵任务
- **多精度原生支持**：INT8 / INT16 / FP32，峰值 INT8 算力 512 GOPS（@100MHz）
- **IREE (MLIR) 编译栈**：PyTorch / TF / JAX → StableHLO → .vmfb，原生 CNN / Transformer / TinyLLM
- **CHERI 内存安全**：硬件级保护，可与 FSI 安全域联动
- **资源友好**：~30K LUT / ~160 DSP，DSP 可弹性扩容

> ⚠️ Coral FPGA 公开移植案例偏少，TL↔AXI4 桥、QEMU 设备模型需从零搭建。

### BootROM：FSI 信任根启动

BootROM（32KB 独立 BRAM）由 FSI 独占访问、运行时只读。FSI 上电后执行 BootROM 自校验，随后通过 SPI Flash 或 UART XMODEM 加载固件，完成后负责 NPU / AP 镜像的加载与验签。

> 详见 [`boot.md`](boot.md)。

## 三阶段开发流程

| 阶段 | 平台 | 核心任务 | 交付物 |
|------|------|----------|--------|
| **一** | QEMU | 虚拟原型 + 驱动开发 | QEMU 设备模型、镜像 |
| **二** | Verilator | RTL 仿真验证 | RTL 源码、Testbench、波形 |
| **三** | XC7K325T | FPGA 物理验证 | 比特流、BOOT.BIN、联调 |

### 阶段一：QEMU 原型

单仓库构建出唯一 `qemu-system-riscv64`，通过 `-M` 参数切换不同模式。RV64 与 RV32 核同实例共存，与 RTL 各子系统一一对应。

| 模式 | 命令 | 说明 |
|------|------|------|
| **全 SoC** | `-M v2p` | AP+FSI+LSIO 全使能 |
| **AP 专用** | `-M v2p-ap -smp 4` | 仅 AP，独立开发调试 |
| **FSI 专用** | `-M v2p-fsi` | 仅 FSI，固件独立开发 |
| **NPU** | MPACT C-Model | Coral ISS ~500 行封装 |

### 阶段二：独立 RTL 仿真

| 子系统 | 仿真对象 |
|--------|----------|
| **AP** | VexRiscv-SMP RV64 + 缓存一致性 |
| **NPU** | Coral NPU + TileLink→AXI（MPACT 黄金参考） |
| **FSI** | VexRiscv RV32 + AES/SHA/RSA/TRNG |
| **LSIO** | 各外设逐模块独立 testbench |

### 阶段三：FPGA 物理验证

Vivado 综合与布局布线，上板顺序：FSI → AP FreeRTOS → AP Linux SMP → 核间通信 → NPU → LSIO。

## 项目目录

```
V2P/                          # 顶层元仓库（submodule 聚合）
├── qemu/                     # 单仓库·单二进制·多机器类型 (SoC 核心)
│   ├── soc/                  # SoC 顶层: AXI 互联 + 子系统组装
│   ├── ap/                   # AP 子系统: RV64×4 + PLIC + CLINT
│   ├── fsi/                  # FSI 子系统: RV32 + AES/SHA/RSA/TRNG
│   ├── lsio/                 # LSIO: UART/SPI/I2C/DMA/GPIO/ETH/SD/RTC
│   ├── include/              # 全局头文件
│   ├── Kconfig               # 子系统可配置
│   └── meson.build           # QEMU 构建集成
├── rtl/                      # RTL 设计（SoC 核心）
├── software/                 # SoC 公共软件
│   ├── bootrom/              # 一级引导
│   ├── uboot/                # U-Boot + OpenSBI
│   ├── linux/                # Linux SMP 内核 + 驱动
│   └── rtos/                 # RTOS（待定）
├── tools/                    # 编译脚本、仿真环境
└── npu/ → v2p-npu            # NPU 子系统（独立仓库 submodule）
```
