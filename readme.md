# V2P — 异构 RISC-V 芯片全流程开发验证平台

## 摘要

V2P（Virtual to Physical）是一个异构 RISC-V 芯片全流程开发验证平台。芯片包含以下四大模块：

- **AP（应用处理器四核 SMP）**
- **NPU（神经网络推理加速）**
- **FSI（功能安全岛）**
- **LSIO（低速外设子系统）**

AP 支持 Linux SMP / FreeRTOS SMP / RT-Thread SMP，NPU 与 FSI 各运行 FreeRTOS。项目采用"QEMU 虚拟原型先行 → FPGA 物理验证收官"的两阶段模式，硬件地址映射由设备树（Linux）与固定地址表（RTOS/裸机）分别消费。目前处于调研阶段。

> **仓库划分**：各子系统可独立拆分仓库

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

| 层级 | 容量 | 实现 | 主要用途 |
|------|------|------|----------|
| **DDR3** | 1GB | 板载 DDR3 (MT41K256M16TW ×2) | AP Linux 主存 + NPU 权重 |
| **SYSTEM SRAM（共享）** | **704KB** | BRAM ×176 | 核间交换 + AP 数据 + 共享缓冲 |
| **NPU 私有 SRAM** | 128KB | BRAM ×32 | NPU 调度核程序与数据 |
| **Gemmini scratchpad** | 40 × 36Kb BRAM | 内部 scratchpad | 权重 / 激活 / 累加 |
| **FSI 私有 SRAM** | 128KB | BRAM ×32 | 含 ~4KB Bootstrap（bitstream 预置）+ 完整固件加载区 |

## 地址空间布局

32-bit 地址空间全貌。**DDR（1GB）置于顶部**，**SYSTEM SRAM 无 Boot ROM 分区**，
FSI 主导所有子系统镜像的加载与验证：

```text
0xFFFF_FFFF ┌──────────────────────────┐ ──┐
            │                          │   │
            │  DDR3 (1GB)              │   │ 1GB
            │  0xC000_0000 ~           │   │ FSI 预加载 OpenSBI/U-Boot/Linux
0xC000_0000 ├──────────────────────────┤ ──┘
            │                          │
            │  (空闲 / 保留)            │
            │                          │
0x4000_0000 ├──────────────────────────┤
            │  CLINT / PLIC / MBOX     │ 核间控制
0x4001_0000 ├──────────────────────────┤
            │                          │
            │  (空闲 / 保留)            │
            │                          │
0x3000_0000 ├──────────────────────────┤
            │  LSIO 外设区 (≤ 4MB)     │ SPI / I2C / UART / DMA / ETH / SD ...
0x2000_0000 ├──────────────────────────┤
            │  FSI 外设区 (≤ 1MB)      │ AES / SHA / RSA / UART
0x2001_0000 ├──────────────────────────┤
            │                          │
            │  (空闲 / 保留)            │
            │                          │
0x0010_0000 ├──────────────────────────┤
            │  NPU 私有 SRAM (128KB)   │ FSI 预加载固件，复位执行
0x000E_0000 ├──────────────────────────┤
            │                          │
            │  FSI 私有 SRAM (128KB)   │
            │  ├─ Bootstrap (~4KB)     │ 复位入口，bitstream 预置
            │  ├─ 完整固件 (~80KB)     │ SPI Flash 加载，SHA 校验
            │  └─ 堆栈/数据 (~44KB)    │
0x000C_0000 ├──────────────────────────┤
            │                          │
            │  SYSTEM SRAM (704KB)     │ 核间共享 / MBOX 数据区
            │  (无 Boot ROM 分区)       │ 启动后 100% 可用
0x0001_0000 ├──────────────────────────┤
            │                          │
            │  (空闲)                  │
            │                          │
0x0000_2000 ├──────────────────────────┤
            │  EEPROM (8KB)           │ 分区表 / 公钥 / 启动计数
            │  (I2C 设备，非执行区)     │ FSI 启动参数来源
0x0000_0000 └──────────────────────────┘
```

> 所有地址在 32-bit 范围内，NPU（RV32）与 FSI（RV32）可直接访问 DDR
> （`0xC000_0000 ~ 0xFFFF_FFFF`），无需窗口或桥接机制。
>
> **复位向量**：
> - **FSI（RV32）** → `0x000C_0000`（私有 SRAM，Bootstrap 入口）
> - **NPU（RV32）** → `0x000E_0000`（私有 SRAM，由 FSI 预加载固件）
> - **AP（RV64）** → `0xC000_0000`（DDR，由 FSI 预加载 OpenSBI）

## FPGA 资源概览

> **硬件平台**：野火凌云 — XC7K325T-2FFG900I（工业级，326,080 逻辑单元），二手 ~¥1,600。

| 资源 | 总量 | 本 SoC 占用 | 占用率 |
|------|------|------------|--------|
| LUT | ~203,800 | ~145K | **71%** |
| DSP48E1 | 840 | **124** | **15%** |
| BRAM (36Kb) | 445 | 312 | **70%** |

Gemmini 8×8 仅需 ~120 DSP、~40 BRAM，LUT 约 85K。整体资源低于 75% 布线阈值，时序收敛可控。

### 板载资源

| 资源 | 规格 |
|------|------|
| DDR3 | 1GB（MT41K256M16TW-107 ×2） |
| 存储 | SD 卡槽、32Mb SPI Flash、8KB EEPROM |
| 显示 | LCD 接口（支持 4.3/5/7 寸 RGB 屏） |
| 高速互联 | PCIe ×4、万兆（10G）光纤 |
| 网络 | 千兆以太网（RGMII） |
| 调试 | JTAG、UART 串口 ×1 |
| 扩展 | SPI / I2C / GPIO（未用 IO 可重映射为多路 UART） |

## NPU 设计亮点

> NPU 子系统由 [`v2p-npu`](#项目目录) 仓库管理，此处仅概述架构决策。

- 集成 **UC Berkeley Gemmini 8×8**，已多次流片验证。
- 通过 TileLink → AXI 桥挂载到系统总线。
- 调度核通过 MMIO 提交推理任务，边缘推理可达 **323× 加速**。
- 相比 16×16 配置性能更优，且硬件资源更节省。

## IP 选型总览

| 组件 | 开源 IP | 来源 | FPGA 成熟度 | QEMU 支持 | 仓库 |
|------|---------|------|------------|-----------|------|
| AP 四核 RV64 SMP | **VexRiscv-SMP** | SpinalHDL | ⭐⭐⭐⭐⭐ | ✅ 原生 | v2p-ap |
| NPU 调度核 RV32 | VexRiscv | SpinalHDL | ⭐⭐⭐⭐⭐ | ✅ -smp 5 | v2p-npu |
| FSI 安全核 RV32 | VexRiscv | SpinalHDL | ⭐⭐⭐⭐⭐ | ⚠️ 独立实例 | v2p-fsi |
| PLIC | OpenPULP PLIC | PULP Platform | ⭐⭐⭐⭐⭐ | ✅ 原生 | v2p-ap |
| CLINT | OpenPULP CLINT | PULP Platform | ⭐⭐⭐⭐⭐ | ✅ 原生 | v2p-ap |
| NPU 脉动阵列 | **Gemmini 8×8** | UC Berkeley | ⭐⭐⭐⭐ | ❌ ~400 行 | v2p-npu |
| MBOX | 自研 | — | — | ❌ ~150 行 | v2p-ap |
| AES-256 | aes_core | OpenCores | ⭐⭐⭐⭐ | ❌ ~100 行 | v2p-fsi |
| SHA-256 | sha256 | OpenCores | ⭐⭐⭐⭐ | ❌ ~80 行 | v2p-fsi |
| RSA-2048 | basicrsa | OpenCores | ⭐⭐⭐ | ❌ ~100 行 | v2p-fsi |
| TRNG | entropy_src | OpenTitan | ⭐⭐⭐⭐ | ❌ ~50 行 | v2p-fsi |
| UART ×3 | uart16550 | OpenCores | ⭐⭐⭐⭐⭐ | ✅ 原生 | v2p-soc |
| SPI | simple_spi | OpenCores | ⭐⭐⭐⭐ | ❌ ~200 行 | v2p-soc |
| I2C | i2c_master_top | OpenCores | ⭐⭐⭐⭐⭐ | ❌ ~150 行 | v2p-soc |
| DMA | wb_dma | OpenCores | ⭐⭐⭐⭐ | ❌ ~300 行 | v2p-soc |
| GPIO | gpio | OpenCores | ⭐⭐⭐⭐ | ❌ ~100 行 | v2p-soc |
| Timer ×3 | timer | OpenCores | ⭐⭐⭐⭐ | ❌ ~100 行 | v2p-soc |
| WDT | watch_dog | OpenCores | ⭐⭐⭐⭐ | ❌ ~80 行 | v2p-soc |
| Ethernet MAC | ethmac | OpenCores | ⭐⭐⭐⭐ | ❌ ~300 行 | v2p-soc |
| SD/MMC | sd_controller | OpenCores | ⭐⭐⭐ | ❌ ~150 行 | v2p-soc |
| RTC | rtc | OpenCores | ⭐⭐⭐ | ❌ ~50 行 | v2p-soc |
| AXI 互联 | OpenPULP axi | PULP Platform | ⭐⭐⭐⭐⭐ | — | v2p-soc |
| DDR3 控制器 | MIG | Xilinx | ⭐⭐⭐⭐⭐ | — | v2p-soc |

> QEMU 原生支持 11 个组件。NPU 部分（Gemmini / 调度核）由 `v2p-npu` 仓库管理。ETH / SD / RTC 等外设自研约 2,100 行。RTL 全栈仅 MBOX 需自研。

## 关键设计决策

### NPU 调度核：独立控制器的价值

> NPU 调度核实现位于 [`v2p-npu`](#项目目录) 仓库，此处仅说明架构原理。

Gemmini 是被动加速器，缺乏程序执行能力。独立的 **VexRiscv RV32 调度核**负责：

- 解耦 AP 与加速器交互。
- 提供 μs 级实时响应。
- 管理 NPU 私有 SRAM、权重搬运、层级调度和后处理。

```text
AP (Linux) → MBOX → 调度核 (裸机) → MMIO → Gemmini 8×8
                            ├─ 128KB 私有 SRAM
                            └─ Gemmini scratchpad
```

该架构契合业界标准，类似 NVDLA / TPU / 其他 NPU 独立控制器设计。

### MBOX：为什么自研

V2P 的 MBOX 目标是极简且可控：

- 双端口 BRAM
- 8 通道门铃中断
- 控制信号与共享 SRAM 数据区分离

| 方案 | 工作量 | 灵活性 | 风险 |
|------|--------|--------|------|
| 自研 MBOX | ~150 行 Verilog | ✅ 完全适配 | 低 |
| 迁移 PULP mailbox | ~300 行集成 | ⚠️ 受限 | 中 |

### FSI 安全启动架构

FSI 作为系统信任根（Root of Trust），主导异构系统的分级安全启动。

**启动源**：FSI Bootstrap 按优先级依次尝试 SPI Flash → SD 卡 → UART XMODEM，
同一介质中按 EEPROM 分区表加载各子系统镜像。EEPROM（8KB）保存公钥哈希、分区表偏移与启动计数等非易失参数。

**启动流程概要**：

| 阶段 | 执行体 | 位置 | 操作 |
|------|--------|------|------|
| 0 | FPGA 位流 | — | 预置 FSI Bootstrap 至 FSI SRAM `0x000C_0000`，DDR 初始化，其余核保持复位 |
| 1 | FSI Bootstrap | FSI SRAM `0x000C_0000` | 从 SPI Flash / SD / UART 加载 FSI 完整固件至 `0x000C_1000`，SHA 自验后跳转 |
| 2 | FSI 完整固件 | FSI SRAM `0x000C_1000` | 加载 NPU 固件 → NPU SRAM，加载 AP OpenSBI/U-Boot/Linux → DDR，RSA 验签后释放各核复位 |
| 3 | 各子系统 | — | AP → OpenSBI → U-Boot → Linux；NPU → FreeRTOS；FSI 降为安全监控 |

> 详细启动协议、分区表格式、通信握手时序见各子系统设计文档。

## 硬件地址映射

以下地址为 SoC 物理地址空间的固定映射，**所有软件环境共享**：

| 模块 | 参数 | 值 |
|------|------|-----|
| **LSIO UART** | 基址 / 中断号 | `0x3003_0000` / `42` |
| LSIO SPI | 基址 / 中断号 | `0x3000_0000` / `43` |
| LSIO I2C | 基址 / 中断号 | `0x3001_0000` / `44` |
| LSIO DMA | 基址 / 中断号 | `0x3002_0000` / `45` |
| LSIO GPIO | 基址 | `0x3004_0000` |
| LSIO WDT | 基址 | `0x3005_0000` |
| **LSIO Ethernet** | 基址 / 中断号 | `0x3006_0000` / `47` |
| **LSIO SD/MMC** | 基址 / 中断号 | `0x3007_0000` / `48` |
| **LSIO RTC** | 基址 | `0x3008_0000` |
| **NPU Debug UART** | 基址 / 中断号 | `0x3009_0000` / `46` |
| **FSI 专属 UART** | 基址 / 中断号 | `0x2005_0000` / `10` |

**消费方式分两类：**

- **Linux** — 由设备树（`.dts` / `.dtb`）描述。QEMU 启动时动态生成 DTB 传递给内核，FPGA 实机由 U-Boot 加载 DTB。设备树源文件位于 [`dts/`]
- **FreeRTOS / RT-Thread / 裸机** — 直接使用上方绝对地址访问，无设备树依赖。

## UART 配置

| UART | 归属 | 特性 |
|------|------|------|
| LSIO UART | AP 控制台 | DMA-capable / AXI-Stream |
| NPU Debug UART | NPU 调试 | 独立串口，推理日志输出 |
| FSI UART | FSI 控制台 | 安全域隔离输出 |

## 三阶段开发流程

| 阶段 | 平台 | 核心任务 | 交付物 |
|------|------|----------|--------|
| **一** | QEMU | 虚拟原型 + 驱动开发 | QEMU 设备模型、镜像 |
| **二** | Verilator | RTL 仿真验证 | RTL 源码、Testbench、波形 |
| **三** | XC7K325T | FPGA 物理验证 | 比特流、BOOT.BIN、联调 |

### 阶段一：QEMU 原型

| 子系统 | 方法 | 说明 |
|--------|------|------|
| **全 SoC** | `qemu-system-riscv64 -M v2p` | 一份二进制，AP+FSI+MBOX+LSIO 单实例缝合，AXI 互联内部路由 |
| **AP 专用** | `qemu-system-riscv64 -M v2p-ap -smp 4` | 仅 AP 子系统（无 FSI/LSIO），AP 软件独立开发调试 |
| **FSI 专用** | `qemu-system-riscv64 -M v2p-fsi` | 仅 FSI 子系统（RV32），FSI 固件独立开发调试 |
| **NPU** | 第 5 核模拟调度核 + Spike 插件 | Gemmini Spike 模型 ~400 行 C（`v2p-npu` 独立仓库） |

> **QEMU 设计原理**：`qemu/` 是单份 QEMU 源码仓库，构建出唯一的 `qemu-system-riscv64` 二进制。
> 内部通过 `-M` 参数选择不同机器类型，各子系统作为 device model 在 AXI 互联上实例化：
>
> ```
> qemu-system-riscv64
>  └── -M v2p              # 全 SoC 模式：所有子系统使能
>  │   ├── AP RV64×4        ─┐
>  │   ├── FSI RV32          ├── AXI Interconnect ── DDR/SRAM
>  │   ├── MBOX              │
>  │   └── LSIO              ─┘
>  ├── -M v2p-ap           # AP 独立模式：仅 AP + MBOX stub
>  └── -M v2p-fsi          # FSI 独立模式：仅 FSI + LSIO stub
> ```
>
> 这种做法的好处：
> - **单源码维护** — 共享 AXI 互联、中断路由、地址映射，改动一处全局生效
> - **DTB 自动生成** — QEMU 根据当前 `-M` 模式动态生成 DTB，Linux 内核通过 DTB 感知硬件拓扑，无需手动维护多份设备树
> - **按需裁剪** — 开发 AP Linux 时用 `-M v2p-ap` 加快启动，联调时用 `-M v2p`
> - **异构原生支持** — RV64 与 RV32 核在同一个 `qemu-system-riscv64` 实例中共存，无需外部 IPC
> - **与 RTL 一一对应** — `qemu/{ap,fsi,mbox,lsio}/` 各模块对应 `rtl/*-sys/` 各子系统，方便交叉验证

### 阶段二：独立 RTL 仿真

| 子系统 | 仿真对象 | 说明 |
|--------|----------|------|
| **AP** | VexRiscv-SMP RV64 + 缓存一致性 | 四核缓存一致性模型 |
| **NPU** | Gemmini 8×8 + TileLink→AXI | `v2p-npu` 独立仿真 |
| **FSI** | VexRiscv RV32 + 安全外设 | AES/SHA/RSA/TRNG 功能验证 |
| **LSIO** | 各外设模块 | 逐模块独立 testbench |

### 阶段三：FPGA 物理验证

- Vivado 综合与布局布线
- 上板顺序：FSI FreeRTOS → AP FreeRTOS/RT-Thread → AP Linux SMP → 核间通信 → NPU 推理 → LSIO 读写

## 项目目录

```
V2P/                          # 顶层元仓库（submodule 聚合）
├── dts/                      # 设备树源文件（硬件权威描述）
│   ├── v2p.dts               # 全 SoC 模式
│   ├── v2p-ap.dts            # AP 独立模式
│   └── v2p-fsi.dts           # FSI 独立模式
├── qemu/                     # 单仓库·单二进制·多机器类型 (SoC 核心)
│   ├── soc/                  # SoC 顶层: AXI 互联 + 子系统组装
│   ├── ap/                   # AP 子系统: RV64×4 + PLIC + CLINT
│   ├── fsi/                  # FSI 子系统: RV32 + AES/SHA/RSA/TRNG
│   ├── mbox/                 # MBOX: 8 通道门铃中断
│   ├── lsio/                 # LSIO: UART/SPI/I2C/DMA/GPIO/ETH/SD/RTC
│   ├── include/              # 全局头文件
│   ├── Kconfig               # 子系统可配置
│   └── meson.build           # QEMU 构建集成
├── rtl/                      # RTL 设计（SoC 核心）
│   ├── ap-sys/               # AP 子系统
│   ├── fsi-sys/              # FSI 子系统
│   └── lsio/                 # LSIO 外设
├── software/                 # SoC 公共软件
│   ├── fsi-boot/             # FSI 启动固件 (bootstrap + 完整固件)
│   ├── uboot/                # U-Boot + OpenSBI
│   ├── linux/                # Linux SMP 内核 + 驱动
│   └── rtos/                 # RTOS（待定）
├── tools/                    # 编译脚本、仿真环境
└── npu/ → v2p-npu            # NPU 子系统（独立仓库 submodule）
    ├── qemu/                 # Gemmini Spike 模型 ~400 行
    ├── rtl/                  # Gemmini 8×8 + 调度核
    └── software/             # 调度核固件 + FreeRTOS
```

> **仓库拆分策略**：`qemu/` + `rtl/lsio/` 可归入 `v2p-soc`；`rtl/ap-sys/` 可归入 `v2p-ap`；
> `rtl/fsi-sys/` 可归入 `v2p-fsi`；`npu/` 对应独立仓库 `v2p-npu`。
> 各仓库目录结构均遵循上述模式。