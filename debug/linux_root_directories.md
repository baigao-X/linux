# Linux 内核根目录结构详解

本文档详细说明 Linux 内核源码树中每个根目录的作用和功能。

## 核心子系统目录

### arch/
**架构特定代码**
- 包含所有支持的硬件架构的特定实现
- 主要架构：x86, arm, arm64, riscv, powerpc, mips, s390, sparc 等
- 每个架构子目录包含：
  - `kernel/` - 进程切换、系统调用、中断处理
  - `mm/` - 内存管理（页表、TLB 管理）
  - `boot/` - 启动加载器代码
  - `include/` - 架构特定头文件
- 负责：CPU 初始化、中断控制器、定时器、电源管理等底层硬件交互

### kernel/
**核心内核子系统**
- 进程调度器（`sched/`）- CFS、实时调度、负载均衡
- 时间管理（`time/`）- 定时器、时钟源、tickless 支持
- 中断处理（`irq/`）- 通用 IRQ 框架、中断线程化
- 同步机制（`locking/`）- 互斥锁、信号量、RCU
- 进程管理 - fork、exec、exit、信号处理
- 模块加载器（`module/`）
- 跟踪和性能（`trace/`, `events/`）
- cgroup 和命名空间
- 电源管理（`power/`）

### mm/
**内存管理子系统**
- 物理内存管理：
  - 页分配器（buddy allocator）
  - slab/slub/slob 分配器（小对象分配）
  - 内存池（mempool）
- 虚拟内存管理：
  - 进程地址空间管理
  - 页表操作
  - 内存映射（mmap）
- 页面回收和交换（`vmscan.c`, `swap.c`）
- 内存压缩和迁移
- 透明大页（THP）
- 内存控制组（memory cgroup）
- NUMA 支持

### fs/
**文件系统层**
- VFS（虚拟文件系统）核心 - 提供统一的文件系统接口
- 具体文件系统实现：
  - `ext4/` - 第四代扩展文件系统
  - `btrfs/` - B-tree 文件系统
  - `xfs/` - 高性能日志文件系统
  - `f2fs/` - Flash-Friendly 文件系统
  - `nfs/` - 网络文件系统
  - `proc/`, `sysfs/`, `debugfs/` - 虚拟文件系统
- 文件缓存和页缓存
- 目录项缓存（dcache）
- inode 缓存
- 文件锁机制

### net/
**网络协议栈**
- 核心网络代码（`core/`）- socket API、sk_buff 管理
- 协议实现：
  - `ipv4/`, `ipv6/` - IP 协议
  - `unix/` - Unix 域套接字
  - `packet/` - 原始包套接字
  - `netlink/` - 内核-用户空间通信
  - `bridge/` - 网桥
  - `wireless/` - 无线网络
- 网络设备接口
- 路由和转发
- 防火墙（netfilter/iptables）
- 流量控制（`sched/`）

### drivers/
**设备驱动程序**（内核最大的目录）
- 按设备类型组织：
  - `net/` - 网络设备驱动
  - `block/` - 块设备驱动
  - `char/` - 字符设备驱动
  - `gpu/` - 图形处理器驱动
  - `usb/` - USB 设备驱动
  - `pci/` - PCI 总线和设备
  - `i2c/`, `spi/` - 串行总线
  - `input/` - 输入设备（键盘、鼠标、触摸屏）
  - `tty/` - 终端设备
  - `scsi/` - SCSI 和 SATA 驱动
  - `md/` - 软件 RAID
  - `media/` - 多媒体设备
  - `platform/` - 平台特定驱动

### block/
**块 I/O 层**
- 通用块层（block layer）
- I/O 调度器（`mq-deadline`, `kyber`, `bfq`）
- 块设备请求队列管理
- 多队列块 I/O（blk-mq）
- 分区支持
- 块设备缓存

## 初始化和库

### init/
**内核初始化代码**
- `main.c` - 内核启动入口点（`start_kernel()`）
- 早期初始化流程
- initramfs 处理
- 第一个用户空间进程（init）启动

### lib/
**内核库函数**
- 通用数据结构：
  - 链表、红黑树、基数树
  - 位图、位操作
  - 哈希表
- 字符串处理函数
- 数学函数（除法、平方根等）
- 压缩/解压缩算法
- CRC 校验
- 排序算法
- 调试辅助函数

## 安全和加密

### security/
**安全框架和模块**
- SELinux - 强制访问控制
- AppArmor - 应用程序安全配置
- Smack - 简化的强制访问控制
- TOMOYO - 路径名基础的 MAC
- LSM（Linux Security Modules）框架
- 密钥管理
- 完整性子系统（IMA/EVM）

### crypto/
**加密 API**
- 对称加密算法（AES, DES, ChaCha20 等）
- 非对称加密（RSA, ECDSA 等）
- 哈希算法（SHA, MD5, BLAKE2 等）
- 随机数生成器
- 加密 API 框架
- 硬件加密加速支持

### certs/
**证书和密钥**
- 系统证书存储
- 模块签名验证密钥
- 内核镜像签名
- 可信密钥环

## 进程间通信和 I/O

### ipc/
**进程间通信（IPC）**
- System V IPC：
  - 消息队列（`msg.c`）
  - 信号量（`sem.c`）
  - 共享内存（`shm.c`）
- POSIX 消息队列
- 命名空间支持

### io_uring/
**高性能异步 I/O 框架**
- 基于共享内存环的 I/O 接口
- 零拷贝 I/O
- 批量 I/O 提交和完成
- 支持文件、网络、定时器等操作
- 现代高性能应用的首选 I/O 机制

## 虚拟化

### virt/
**虚拟化支持**
- KVM（Kernel-based Virtual Machine）核心代码
- 虚拟化相关的通用代码
- 与架构特定的虚拟化代码（在 `arch/*/kvm/`）协同工作

## 音频

### sound/
**音频子系统**
- ALSA（Advanced Linux Sound Architecture）框架
- 音频驱动程序：
  - PCI 声卡
  - USB 音频设备
  - SoC 音频（ARM 平台）
- 音频编解码器
- MIDI 支持
- OSS 模拟层

## 开发工具和示例

### scripts/
**构建脚本和工具**
- `checkpatch.pl` - 代码风格检查工具
- `get_maintainer.pl` - 查找子系统维护者
- `kernel-doc` - 文档生成工具
- Kconfig 处理脚本
- 设备树编译器（dtc）
- 模块签名工具
- 各种构建辅助脚本

### tools/
**用户空间工具**
- `perf/` - 性能分析工具（CPU profiling、trace 等）
- `bpf/` - eBPF 工具和库
- `testing/selftests/` - 内核自测试套件
- `power/` - 电源管理工具（cpupower, turbostat 等）
- `vm/` - 虚拟内存测试工具
- `gpio/` - GPIO 工具
- `objtool/` - 对象文件验证工具

### samples/
**示例代码**
- 内核模块示例
- BPF 程序示例
- 各种子系统的使用示例
- 用于学习内核编程的参考代码

## 文档和许可证

### Documentation/
**内核文档**
- 使用 reStructuredText (RST) 格式
- 分类：
  - `admin-guide/` - 系统管理员指南
  - `driver-api/` - 驱动开发 API
  - `dev-tools/` - 开发工具文档
  - `process/` - 开发流程和规范
  - `core-api/` - 核心 API 文档
  - 设备树文档（`devicetree/`）
  - 架构特定文档（`arch/`）
- 可通过 `make htmldocs` 生成 HTML 文档

### LICENSES/
**许可证文本**
- GPL-2.0 - Linux 内核主许可证
- 其他兼容许可证文本
- SPDX 许可证标识符

## 头文件

### include/
**内核头文件**
- `include/linux/` - 核心内核 API 头文件
- `include/uapi/` - 用户空间 API（UAPI）头文件
  - 这些头文件可以被用户空间程序包含
  - 定义系统调用接口、ioctl 命令等
- `include/asm-generic/` - 通用架构头文件
- `include/net/` - 网络子系统头文件
- `include/crypto/` - 加密 API 头文件

## 现代语言支持

### rust/
**Rust 语言支持**（自 v6.1 引入）
- Rust 绑定和抽象
- Rust 内核模块基础设施
- Rust 驱动程序示例
- 安全的内核编程替代方案
- 目前处于实验阶段，逐步增加支持

## 其他

### usr/
**内置 initramfs**
- 用于构建内置的 initramfs（初始 RAM 文件系统）
- 包含早期用户空间所需的文件
- `gen_init_cpio` 工具

## 目录统计

按代码量排序的主要目录（大致）：
1. **drivers/** - 约占 60% 的内核代码（设备驱动最多）
2. **arch/** - 约占 15-20%（支持众多架构）
3. **fs/** - 约占 5-8%（众多文件系统）
4. **net/** - 约占 5-7%（完整的网络协议栈）
5. **sound/** - 约占 3-5%（音频驱动和框架）
6. 其余目录合计约占 10-15%

## 总结

Linux 内核的目录结构反映了操作系统的层次化设计：

- **硬件抽象层**: `arch/`, `drivers/`
- **核心子系统**: `kernel/`, `mm/`, `fs/`, `net/`
- **安全和加密**: `security/`, `crypto/`, `certs/`
- **用户接口**: `include/uapi/`, `ipc/`, `io_uring/`
- **支持设施**: `lib/`, `init/`, `scripts/`, `tools/`
- **文档和示例**: `Documentation/`, `samples/`

这种组织方式使得内核既模块化又高度集成，便于开发、维护和理解。
