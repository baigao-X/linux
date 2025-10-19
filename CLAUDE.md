# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 Linux 内核代码库 (v6.18-rc1)。Linux 内核是一个类 Unix 操作系统内核,支持多种硬件架构,遵循 GPL v2 许可证。

## 构建系统

### 基本构建命令

```bash
# 配置内核
make menuconfig          # 基于菜单的配置界面
make defconfig           # 使用架构默认配置
make localmodconfig      # 基于当前系统模块配置

# 构建内核
make -j$(nproc)          # 编译内核和模块
make vmlinux             # 仅构建内核镜像
make modules             # 仅构建模块

# 清理
make clean               # 清理生成文件但保留配置
make mrproper            # 完全清理包括配置文件
make distclean           # mrproper + 清理编辑器备份文件

# 其他有用命令
make help                # 显示所有可用目标
make V=1                 # 详细构建输出
```

### 架构特定构建

```bash
# 为特定架构构建 (默认是当前系统架构)
make ARCH=arm64 defconfig
make ARCH=x86_64 menuconfig
```

### 测试和检查

```bash
# 代码检查
scripts/checkpatch.pl <patch-file>              # 检查补丁格式
scripts/checkpatch.pl -f <source-file>          # 检查源文件

# 静态分析
make C=1                                        # 使用 sparse 进行检查
make coccicheck                                 # 使用 coccinelle 检查

# 构建测试
make allmodconfig && make -j$(nproc)           # 测试所有模块配置
```

## 代码架构

### 目录结构

- **arch/** - 架构特定代码 (x86, arm, arm64, riscv, powerpc, mips 等)
  - 每个架构有自己的 kernel/, mm/, boot/ 子目录
  - 平台初始化、中断处理、内存管理在此实现

- **kernel/** - 核心内核子系统
  - 进程调度 (sched/)
  - 时间管理 (time/)
  - 中断和软中断处理 (irq/)
  - 同步原语 (locking/)

- **mm/** - 内存管理
  - 页分配器、slab 分配器
  - 虚拟内存管理
  - 内存回收和压缩

- **fs/** - 文件系统
  - VFS (虚拟文件系统) 核心
  - 具体文件系统实现 (ext4, btrfs, xfs 等)

- **drivers/** - 设备驱动
  - 按设备类型组织 (net/, block/, char/, gpu/ 等)
  - 最大的目录,包含数千个驱动程序

- **net/** - 网络协议栈
  - 核心网络代码
  - 各种协议实现 (ipv4/, ipv6/, unix/, netlink/ 等)

- **include/** - 头文件
  - include/linux/ - 内核 API 头文件
  - include/uapi/ - 用户空间 API (UAPI)

- **scripts/** - 构建脚本和工具
  - checkpatch.pl - 代码风格检查
  - 各种构建辅助脚本

- **tools/** - 用户空间工具
  - perf - 性能分析工具
  - bpf - eBPF 工具
  - testing/selftests - 内核自测试

- **Documentation/** - 内核文档
  - 使用 reStructuredText 格式
  - 可通过 `make htmldocs` 生成 HTML

## 编码规范

### 基本规则

- **缩进**: 使用 8 字符宽的 **Tab** (不是空格)
- **行长度**: 建议 80 字符,最多 100 字符
- **命名**:
  - 使用小写加下划线: `function_name`, `variable_name`
  - 避免驼峰命名 (CamelCase)
- **花括号**: K&R 风格
  ```c
  if (condition) {
      do_something();
  }
  ```
- **switch 语句**: case 标签与 switch 对齐

### 代码检查工具

使用前必须通过 checkpatch.pl 检查:
```bash
scripts/checkpatch.pl --strict <patch-file>
```

## 提交补丁

内核开发使用邮件列表进行补丁审查:

```bash
# 生成补丁
git format-patch -1 HEAD

# 检查补丁
scripts/checkpatch.pl 0001-*.patch

# 查找维护者
scripts/get_maintainer.pl 0001-*.patch
```

### 提交信息格式

```
subsystem: brief description (max 70 chars)

Detailed explanation of what this patch does and why.
Wrap at 75 columns.

Signed-off-by: Your Name <your.email@example.com>
```

## 构建要求

最小版本要求 (见 Documentation/process/changes.rst):
- GCC >= 8.1 或 Clang >= 15.0.0
- GNU Make >= 4.0
- binutils >= 2.30
- flex >= 2.5.35
- bison >= 2.0

可选:
- Rust >= 1.78.0 (用于 Rust 支持)
- pahole >= 1.16 (用于 BTF 调试信息)

## 关键文件

- **Kconfig** - 内核配置系统入口
- **Kbuild** - 内核构建系统入口
- **Makefile** - 顶层 Makefile
- **.config** - 当前内核配置 (生成文件,不提交)
- **MAINTAINERS** - 维护者和子系统信息

## 调试

```bash
# 启用调试符号
scripts/config --enable DEBUG_INFO
scripts/config --enable DEBUG_KERNEL

# 使用 QEMU 测试
make ARCH=x86_64 defconfig
make -j$(nproc)
qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage ...
```

## 文档生成

```bash
# 生成 HTML 文档
make htmldocs

# 生成 PDF 文档
make pdfdocs

# 文档位于 Documentation/output/
```

## 注意事项

- 内核代码不使用标准 C 库
- 不能使用浮点运算 (在大多数上下文中)
- 栈空间有限 (通常 8KB-16KB)
- 必须考虑并发和竞态条件
- 中断上下文有特殊限制 (不能睡眠)
- 使用内核特定的数据结构和 API (kzalloc, list_head 等)
