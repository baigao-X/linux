# Linux 内核 QEMU 调试完整指南

本指南介绍如何编译 Linux 内核并使用 QEMU 和 GDB 进行调试。

## 1. 配置内核(启用调试符号)

首先需要配置内核以启用调试支持:

```bash
# 使用默认配置
make defconfig

# 启用调试选项
scripts/config --enable DEBUG_INFO
scripts/config --enable DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT
scripts/config --enable GDB_SCRIPTS
scripts/config --disable DEBUG_INFO_REDUCED
scripts/config --enable DEBUG_KERNEL
scripts/config --enable KGDB

# 应用配置
make olddefconfig
```

### 调试选项说明

- `DEBUG_INFO`: 启用调试信息
- `DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT`: 使用 DWARF 调试格式
- `GDB_SCRIPTS`: 启用 GDB Python 脚本支持
- `DEBUG_INFO_REDUCED`: 禁用精简调试信息(需要完整符号)
- `DEBUG_KERNEL`: 启用内核调试特性
- `KGDB`: 启用内核内置 GDB 调试器

## 2. 编译内核

```bash
# 编译(使用所有 CPU 核心)
make -j$(nproc)

# 编译完成后,关键文件位置:
# - 内核镜像: arch/x86/boot/bzImage
# - 带符号的内核: vmlinux
```

编译时间取决于 CPU 性能,通常需要 10-30 分钟。

## 3. 创建根文件系统(可选)

### 方法 1: 使用 BusyBox 最小 initramfs(推荐)

```bash
# 创建工作目录
mkdir -p initramfs
cd initramfs

# 下载并编译静态链接的 BusyBox
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1

# 配置为静态编译
make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc)

# 安装 BusyBox
cd ..
mkdir -p rootfs/{bin,sbin,etc,proc,sys,dev,usr/{bin,sbin}}
cd rootfs
cp ../busybox-1.36.1/busybox bin/
ln -s busybox bin/sh
ln -s busybox bin/mount
ln -s busybox bin/ls
ln -s busybox bin/cat
ln -s busybox bin/echo

# 创建 init 脚本
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
echo "========================================="
echo "欢迎进入内核调试环境 (BusyBox)"
echo "========================================="
exec /bin/sh
EOF

chmod +x init

# 打包 initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../../initramfs.cpio.gz
cd ../..
```

### 方法 2: 使用系统工具创建 initramfs(快速方法)

```bash
# 创建基本目录结构
mkdir -p initramfs/rootfs/{bin,sbin,etc,proc,sys,dev}
cd initramfs/rootfs

# 复制必要的二进制文件(静态编译或带依赖库)
# 如果系统有 busybox
cp /usr/bin/busybox bin/ 2>/dev/null || cp /bin/busybox bin/

# 如果没有 busybox,复制基本工具和依赖库
if [ ! -f bin/busybox ]; then
    mkdir -p lib lib64
    # 复制 shell 和基本工具
    cp /bin/sh bin/
    cp /bin/mount bin/
    cp /bin/ls bin/

    # 复制动态库依赖
    for bin in bin/*; do
        ldd "$bin" 2>/dev/null | grep -o '/lib[^ ]*' | while read lib; do
            mkdir -p "$(dirname .$lib)"
            cp "$lib" ".$lib" 2>/dev/null
        done
    done
fi

# 创建 BusyBox 符号链接(如果有)
if [ -f bin/busybox ]; then
    ln -sf busybox bin/sh
    ln -sf busybox bin/mount
    ln -sf busybox bin/ls
fi

# 创建 init 脚本
cat > init << 'EOF'
#!/bin/sh
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev
echo "欢迎进入内核调试环境"
exec /bin/sh
EOF

chmod +x init

# 打包 initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
cd ../..
```

### 方法 3: 不使用根文件系统(最简单)

如果只调试内核启动过程和核心功能,可以完全不使用 initramfs:

```bash
# 直接使用内核启动,会在无法挂载根文件系统后进入 panic
# 但这足够调试内核早期启动、内存管理、调度器等核心功能
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -nographic \
    -append "console=ttyS0 nokaslr" \
    -s -S
```

**优点**: 无需额外准备,直接调试内核核心代码
**缺点**: 无法进入用户空间,无法测试系统调用和用户交互

## 4. 启动 QEMU 进行调试

### 基本调试模式

```bash
# 启动 QEMU 并等待 GDB 连接
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -nographic \
    -append "console=ttyS0 nokaslr" \
    -s -S
```

### 使用 initramfs 启动

```bash
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd initramfs.cpio.gz \
    -nographic \
    -append "console=ttyS0 nokaslr" \
    -s -S
```

### QEMU 参数说明

- `-kernel`: 指定内核镜像文件
- `-initrd`: 指定初始 RAM 磁盘
- `-nographic`: 无图形界面,使用串口控制台
- `-append`: 内核启动参数
  - `console=ttyS0`: 使用串口输出(显示在当前终端)
  - `nokaslr`: 禁用内核地址空间随机化(方便调试)
- `-s`: 等同于 `-gdb tcp::1234`,在 1234 端口开启 gdb server
- `-S`: 启动时暂停 CPU,等待 GDB 连接

### 其他有用的 QEMU 参数

```bash
# 增加内存
-m 2G

# 启用 KVM 加速(需要 CPU 虚拟化支持)
-enable-kvm

# 网络调试
-netdev user,id=net0 -device e1000,netdev=net0

# 挂载宿主机目录(需要 9p 文件系统支持)
-virtfs local,path=/host/path,mount_tag=host,security_model=none
```

## 5. 使用 GDB 连接调试

在另一个终端启动 GDB:

```bash
# 启动 GDB(加载带符号的内核)
gdb vmlinux
```

### GDB 基本连接和断点设置

```gdb
# 连接到 QEMU
(gdb) target remote :1234

# 设置常用断点
(gdb) break start_kernel          # 内核启动入口
(gdb) break do_sys_open           # 文件打开系统调用
(gdb) break tcp_v4_connect        # TCP 连接函数
(gdb) break __netif_receive_skb   # 网络包接收

# 继续执行
(gdb) continue

# 查看当前位置
(gdb) list

# 查看调用栈
(gdb) backtrace
```

## 6. 常用 GDB 调试命令

### 断点管理

```gdb
# 设置断点
break function_name              # 函数断点
break file.c:123                # 文件行号断点
break *0xffffffff81000000       # 内存地址断点

# 条件断点
break tcp_v4_connect if port == 80

# 查看和管理断点
info breakpoints                # 列出所有断点
delete 1                        # 删除断点 1
disable 2                       # 禁用断点 2
enable 2                        # 启用断点 2
clear function_name             # 清除函数所有断点
```

### 执行控制

```gdb
continue (c)                    # 继续执行直到断点
step (s)                        # 单步执行(进入函数)
next (n)                        # 单步执行(跳过函数)
finish                          # 执行到当前函数返回
until                           # 执行到下一行
until line_number               # 执行到指定行
stepi (si)                      # 单步执行一条汇编指令
nexti (ni)                      # 跳过执行一条汇编指令
```

### 查看信息

```gdb
# 调用栈
backtrace (bt)                  # 显示调用栈
backtrace full                  # 显示调用栈及局部变量
frame 2                         # 切换到栈帧 2
up                              # 上移一个栈帧
down                            # 下移一个栈帧

# 变量和内存
print variable                  # 打印变量值
print *pointer                  # 打印指针指向的值
print array[0]@10              # 打印数组前 10 个元素
print/x variable                # 十六进制打印
print/t variable                # 二进制打印
display variable                # 每次停止时自动显示

# 查看内存
x/10x address                   # 以十六进制显示 10 个字
x/10i address                   # 反汇编 10 条指令
x/s string_ptr                  # 显示字符串

# 局部信息
info locals                     # 显示局部变量
info args                       # 显示函数参数
info registers                  # 显示寄存器
info threads                    # 显示线程信息
```

### 监视点(Watchpoint)

```gdb
watch variable                  # 变量改变时中断
watch *(int*)0xaddress         # 监视内存地址
rwatch variable                 # 变量被读取时中断
awatch variable                 # 变量被访问时中断
info watchpoints               # 显示所有监视点
```

### 源码查看

```gdb
list                            # 显示当前位置源码
list function_name              # 显示函数源码
list file.c:100                # 显示指定文件行
list -                          # 显示前面的代码
set listsize 20                # 设置显示行数
```

## 7. Linux 内核专用 GDB 功能

### 加载内核 GDB 脚本

内核提供了专门的 GDB Python 脚本,提供便捷的调试命令:

```gdb
# 加载内核脚本(通常自动加载)
source scripts/gdb/vmlinux-gdb.py

# 或者
(gdb) source vmlinux-gdb.py
```

### 内核专用命令

```gdb
# 符号管理
lx-symbols                      # 加载所有模块符号
lx-symbols ../modules          # 从指定目录加载模块

# 内核日志
lx-dmesg                        # 显示内核日志(类似 dmesg)

# 进程和任务
lx-ps                           # 显示进程列表
lx-task                         # 显示当前任务

# 内存信息
lx-mounts                       # 显示挂载信息
lx-cmdline                      # 显示内核命令行

# 设备树
lx-device-list-bus              # 列出总线设备
lx-device-list-class            # 列出设备类

# CPU 和中断
lx-cpus                         # 显示 CPU 信息
lx-interrupt                    # 显示中断统计
```

### 调试内核数据结构

```gdb
# 链表遍历(使用 list_head)
print ((struct task_struct *)0xaddress)->tasks

# 使用内核宏
print container_of(ptr, struct task_struct, tasks)

# 查看内核符号
info address symbol_name
```

## 8. 使用 KGDB(内核内置调试器)

KGDB 允许通过串口直接调试运行中的内核。

### 启用 KGDB

```bash
# 配置内核时已启用 CONFIG_KGDB

# QEMU 启动时添加 KGDB 参数
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -nographic \
    -append "console=ttyS0 nokaslr kgdboc=ttyS0,115200 kgdbwait" \
    -s -S
```

### KGDB 参数说明

- `kgdboc=ttyS0,115200`: 使用 ttyS0 串口,波特率 115200
- `kgdbwait`: 启动时等待调试器连接
- `kgdbcon`: 使用 KGDB 控制台

## 9. 实战调试示例

### 示例 1: 调试内核启动

```bash
# 1. 启动 QEMU(终端 1)
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic \
    -append "console=ttyS0 nokaslr" -s -S

# 2. 启动 GDB(终端 2)
gdb vmlinux
(gdb) target remote :1234
(gdb) break start_kernel
(gdb) continue

# 3. 单步调试启动过程
(gdb) next
(gdb) print init_task
(gdb) backtrace
```

### 示例 2: 调试系统调用

```bash
# 在 GDB 中
(gdb) break sys_openat
(gdb) continue

# 当断点触发时
(gdb) info args                 # 查看参数
(gdb) backtrace                # 查看调用链
(gdb) next                     # 单步执行
```

### 示例 3: 调试网络协议栈

```bash
# 设置网络相关断点
(gdb) break tcp_v4_connect
(gdb) break tcp_rcv_established
(gdb) break ip_output
(gdb) continue

# 分析 sk_buff 结构
(gdb) print *(struct sk_buff *)skb
(gdb) x/100x skb->data         # 查看数据包内容
```

### 示例 4: 调试设备驱动

```bash
# 设置驱动相关断点
(gdb) break driver_probe_device
(gdb) break pci_device_probe
(gdb) continue

# 查看设备结构
(gdb) print *(struct device *)dev
```

## 10. 常见问题和技巧

### 问题: 找不到源码

```gdb
# 设置源码路径
(gdb) directory /path/to/kernel/source
(gdb) set substitute-path /old/path /new/path
```

### 问题: KASLR 导致地址随机

```bash
# 启动时禁用 KASLR
-append "nokaslr"
```

### 技巧: 保存断点

```gdb
# 保存断点到文件
(gdb) save breakpoints breakpoints.txt

# 加载断点
(gdb) source breakpoints.txt
```

### 技巧: 使用 .gdbinit

创建 `.gdbinit` 文件自动执行命令:

```bash
# .gdbinit
target remote :1234
break start_kernel
continue
```

### 技巧: 条件断点优化性能

```gdb
# 仅在特定条件下中断
break tcp_v4_connect if sk->sk_dport == 0x5000  # 端口 80
break kmalloc if size > 4096                     # 大内存分配
```

## 11. 退出调试

```bash
# 在 GDB 中
(gdb) quit

# 在 QEMU 串口中
# 按 Ctrl-A 然后按 X 退出 QEMU
# 或者在另一个终端: killall qemu-system-x86_64
```

## 12. 参考资料

- 内核文档: `Documentation/dev-tools/kgdb.rst`
- 内核文档: `Documentation/dev-tools/gdb-kernel-debugging.rst`
- GDB 官方手册: https://sourceware.org/gdb/documentation/
- QEMU 文档: https://www.qemu.org/docs/master/

## 附录: 快速参考

### 最小调试命令集

```bash
# 编译
make defconfig && scripts/config --enable DEBUG_INFO && make -j$(nproc)

# QEMU 启动
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic -append "console=ttyS0 nokaslr" -s -S

# GDB 连接
gdb vmlinux -ex "target remote :1234" -ex "break start_kernel" -ex "continue"
```

### 常用 GDB 命令速查

| 命令 | 缩写 | 说明 |
|------|------|------|
| continue | c | 继续执行 |
| step | s | 单步进入 |
| next | n | 单步跳过 |
| finish | fin | 执行到返回 |
| backtrace | bt | 调用栈 |
| print | p | 打印变量 |
| break | b | 设置断点 |
| watch | wa | 设置监视点 |
| info breakpoints | i b | 查看断点 |
| list | l | 显示源码 |

祝调试顺利!
