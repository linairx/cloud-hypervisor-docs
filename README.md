# Cloud Hypervisor 项目分析

## 项目概览

| 项目 | 信息 |
|------|------|
| **名称** | Cloud Hypervisor |
| **类型** | 虚拟机监视器 (VMM) |
| **语言** | Rust (Edition 2024) |
| **许可证** | Apache-2.0 / BSD-3-Clause |
| **仓库** | https://github.com/cloud-hypervisor/cloud-hypervisor |

### 项目定位

Cloud Hypervisor 是一个开源的虚拟机监视器 (VMM)，运行在 KVM 和 Microsoft Hypervisor (MSHV) 之上。

### 设计目标

- **高性能**: 低延迟、高吞吐量
- **低资源占用**: 最小内存占用、小攻击面
- **现代架构**: 仅支持 64 位，无传统设备依赖
- **云原生**: 针对云工作负载优化

### 支持的架构

| 架构 | 状态 |
|------|------|
| x86-64 | ✅ 完全支持 |
| AArch64 | ✅ 完全支持 |
| riscv64 | ⚠️ 实验性支持 |

### 支持的客户机

- 64-bit Linux
- Windows 10 / Windows Server 2019+

---

## 目录结构

```
cloud-hypervisor/
├── src/                    # 主程序入口
├── vmm/                    # VMM 核心逻辑
├── arch/                   # 架构相关代码
│   ├── x86_64/
│   ├── aarch64/
│   └── riscv64/
├── hypervisor/             # Hypervisor 抽象层
│   ├── kvm/               # KVM 后端
│   └── mshv/              # Microsoft Hypervisor 后端
├── virtio-devices/         # VirtIO 设备实现
├── devices/                # 平台设备
├── pci/                    # PCI 总线实现
├── block/                  # 块设备后端
├── net_util/               # 网络工具
├── vm-allocator/           # 资源分配器
├── vm-device/              # 设备抽象
├── vm-migration/           # 迁移支持
├── vm-virtio/              # VirtIO 队列
├── api_client/             # HTTP API 客户端
├── vhost_user_block/       # vhost-user 块设备
├── vhost_user_net/         # vhost-user 网络设备
└── docs/                   # 文档
```

---

## 核心模块

### 1. VMM 核心 (`vmm/`)

| 文件 | 功能 |
|------|------|
| `vm.rs` | VM 生命周期管理 |
| `cpu.rs` | vCPU 管理 |
| `memory_manager.rs` | Guest 内存管理 |
| `device_manager.rs` | 设备管理 |
| `interrupt.rs` | 中断路由 |
| `config.rs` | 配置解析 |
| `api/` | HTTP API 服务器 |

### 2. 架构层 (`arch/`)

| 目录 | 架构 | 特定功能 |
|------|------|----------|
| `x86_64/` | Intel/AMD | MTRR, MSRs, PMU, CPUID |
| `aarch64/` | ARM64 | GIC 中断控制器 |
| `riscv64/` | RISC-V | AIA 中断控制器 |

### 3. Hypervisor 抽象 (`hypervisor/`)

```rust
// 核心抽象
pub trait Hypervisor: Send + Sync { ... }
pub trait Vm: Send + Sync { ... }
pub trait Vcpu: Send + Sync { ... }
pub trait Device: Send + Sync { ... }

// 后端实现
mod kvm;  // Linux KVM
mod mshv; // Microsoft Hypervisor
```

### 4. 设备模型 (`devices/`)

| 设备 | 文件 | 说明 |
|------|------|------|
| 中断控制器 | `interrupt_controller.rs` | 中断控制器抽象 |
| GIC | `gic.rs` | ARM GIC 实现 |
| IOAPIC | `ioapic.rs` | x86 IOAPIC |
| AIA | `aia.rs` | RISC-V AIA |
| ACPI | `acpi.rs` | ACPI 表生成 |
| IVSHMEM | `ivshmem.rs` | 共享内存设备 |
| TPM | `tpm.rs` | TPM 设备 |
| PVPanic | `pvpanic.rs` | Panic 通知 |

---

## VirtIO 设备

### 设备列表 (`virtio-devices/`)

| 设备 | 文件 | 功能 |
|------|------|------|
| Block | `block.rs` | 块存储设备 |
| Net | `net.rs` | 网络设备 |
| Console | `console.rs` | 控制台 |
| Balloon | `balloon.rs` | 内存气球 |
| RNG | `rng.rs` | 随机数生成器 |
| PMEM | `pmem.rs` | 持久内存 |
| MEM | `mem.rs` | 内存设备 |
| IOMMU | `iommu.rs` | IOMMU 设备 |
| Vsock | `vsock/` | VSocket |
| Watchdog | `watchdog.rs` | 看门狗 |
| VDPA | `vdpa.rs` | vDPA 支持 |

### VirtIO Transport (`virtio-devices/transport/`)

- **PCI**: VirtIO over PCI (主要方式)
- **MMIO**: VirtIO over MMIO (嵌入式场景)

### Vhost-user 支持 (`virtio-devices/vhost_user/`)

- Block: vhost-user-blk
- Net: vhost-user-net
- FS: virtio-fs

---

## 块设备后端 (`block/`)

### 支持的格式

| 格式 | 文件 | 说明 |
|------|------|------|
| Raw | `raw_sync.rs`, `raw_async.rs` | 原始镜像 |
| QCOW2 | `qcow/`, `qcow_sync.rs` | QEMU QCOW2 |
| VHD | `fixed_vhd*.rs` | 固定 VHD |
| VHDX | `vhdx/`, `vhdx_sync.rs` | VHDX 格式 |

### IO 模式

- **同步**: 直接阻塞 IO
- **异步**: 基于 io_uring / AIO

---

## PCI 子系统 (`pci/`)

| 文件 | 功能 |
|------|------|
| `bus.rs` | PCI 总线枚举 |
| `configuration.rs` | PCI 配置空间 |
| `device.rs` | PCI 设备抽象 |
| `msi.rs` | MSI 中断 |
| `msix.rs` | MSI-X 中断 |
| `vfio.rs` | VFIO 直通 |
| `vfio_user.rs` | vfio-user 协议 |

---

## 核心依赖

### rust-vmm 生态

| Crate | 版本 | 用途 |
|-------|------|------|
| `kvm-bindings` | 0.12.1 | KVM 结构体绑定 |
| `kvm-ioctls` | 0.22.1 | KVM ioctl 封装 |
| `mshv-bindings` | 0.6.7 | MSHV 绑定 |
| `mshv-ioctls` | 0.6.7 | MSHV ioctl |
| `vm-memory` | 0.16.1 | Guest 内存抽象 |
| `virtio-queue` | 0.16.0 | VirtIO 队列 |
| `virtio-bindings` | 0.2.6 | VirtIO 结构体 |
| `vhost` | 0.14.0 | vhost 协议 |
| `vhost-user-backend` | 0.20.0 | vhost-user 后端 |
| `vfio-bindings` | 0.6.0 | VFIO 绑定 |
| `vfio-ioctls` | 0.5.1 | VFIO ioctl |
| `seccompiler` | 0.5.0 | seccomp 过滤器 |
| `linux-loader` | 0.13.1 | 内核加载器 |
| `acpi_tables` | 0.2.0 | ACPI 表 |

### IGVM 支持

| Crate | 版本 | 用途 |
|-------|------|------|
| `igvm` | 0.4.0 | IGVM 格式支持 |
| `igvm_defs` | 0.4.0 | IGVM 定义 |

---

## API 设计

### HTTP API (`vmm/src/api/`)

| 端点 | 方法 | 功能 |
|------|------|------|
| `/vmm.ping` | GET | 健康检查 |
| `/vmm.shutdown` | PUT | 关闭 VMM |
| `/vm.info` | GET | VM 信息 |
| `/vm.counters` | GET | 性能计数器 |
| `/vm.add-device` | PUT | 热插设备 |
| `/vm.remove-device` | PUT | 热拔设备 |
| `/vm.resize` | PUT | 调整资源 |
| `/vm.create-snapshot` | PUT | 创建快照 |
| `/vm.restore` | PUT | 恢复快照 |
| `/vm.migrate` | PUT | 迁移 |

### CLI 接口

```bash
cloud-hypervisor \
  --kernel <path> \           # 内核镜像
  --firmware <path> \         # 固件
  --disk path=<path> \        # 磁盘镜像
  --cpus boot=<n> \           # CPU 数量
  --memory size=<size> \      # 内存大小
  --net tap=<tap>,mac=<mac> \ # 网络配置
  --serial <mode> \           # 串口模式
  --console <mode>            # 控制台模式
```

---

## 高级特性

### 热插拔支持

- CPU 热插
- 内存热插
- VFIO 设备热插
- virtio-{net,block,pmem,fs,vsock} 热插

### 快照与迁移

- 快照创建/恢复
- 跨主机迁移
- 状态序列化

### 安全特性

- **TDX**: Intel Trust Domain Extensions
- **SEV-SNP**: AMD Secure Encrypted Virtualization
- **Seccomp**: 系统调用过滤
- **Landlock**: Linux 安全模块

### 性能优化

- **io_uring**: 异步 IO
- **vhost**: 内核态加速
- **vDPA**: vhost 数据路径加速

---

## 设计模式

### Vmm 核心结构

```rust
pub struct Vmm {
    config: VmConfig,
    vm: Arc<dyn HypervisorVm>,
    vcpus: Vec<Arc<dyn Vcpu>>,
    memory_manager: Arc<MemoryManager>,
    device_manager: Arc<DeviceManager>,
    // ...
}
```

### 设备抽象

```rust
pub trait Device: Send + Sync {
    fn device_type(&self) -> DeviceType;
    fn reset(&mut self);
}
```

### VirtIO 设备 Trait

```rust
pub trait VirtioDevice: Send {
    fn device_type(&self) -> u32;
    fn queue_max_sizes(&self) -> &[u16];
    fn features(&self) -> u64;
    fn activate(&mut self, ...);
}
```

---

## 构建与运行

### 构建

```bash
cargo build --release
```

### 运行示例

```bash
# 直接内核启动
./cloud-hypervisor \
  --kernel ./vmlinux \
  --disk path=disk.raw \
  --cpus boot=4 \
  --memory size=4G \
  --net tap=tap0

# 固件启动
./cloud-hypervisor \
  --firmware ./hypervisor-fw \
  --disk path=cloud-image.raw \
  --cpus boot=4 \
  --memory size=4G
```

---

## 与其他项目的关系

### vs Firecracker

| 特性 | Cloud Hypervisor | Firecracker |
|------|------------------|-------------|
| 目标场景 | 通用云工作负载 | Serverless/容器 |
| 热插拔 | ✅ 支持 | ❌ 不支持 |
| 迁移 | ✅ 支持 | ❌ 不支持 |
| 架构 | x86, ARM, RISC-V | x86, ARM |

### vs crosvm

| 特性 | Cloud Hypervisor | crosvm |
|------|------------------|--------|
| 目标场景 | 云工作负载 | Chrome OS |
| Hypervisor | KVM, MSHV | KVM |
| 代码复用 | rust-vmm | rust-vmm |

---

## 文档资源

- [API 文档](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/api.md)
- [设备模型](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/device_model.md)
- [内存管理](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/memory.md)
- [热插拔](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/hotplug.md)
- [实时迁移](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/live_migration.md)
- [快照恢复](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/snapshot_restore.md)

---

## 总结

Cloud Hypervisor 是一个现代、高性能、安全的 VMM 实现，具有以下特点：

1. **纯 Rust 实现**: 利用 Rust 的内存安全特性
2. **模块化设计**: 基于 rust-vmm 生态构建
3. **多架构支持**: x86-64, AArch64, RISC-V
4. **多 Hypervisor**: KVM 和 MSHV 后端
5. **丰富的设备模型**: VirtIO 设备 + 平台设备
6. **企业级特性**: 热插拔、迁移、快照
7. **安全增强**: TDX, SEV-SNP, seccomp 支持

适合作为学习现代 VMM 实现的参考项目。
