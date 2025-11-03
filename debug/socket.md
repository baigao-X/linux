现在让我完整展开 SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol) 的所有层级宏定义:
完整宏展开及逐行注释
第一层: SYSCALL_DEFINE3
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
根据第227行定义:
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
展开为:
SYSCALL_DEFINEx(3, _socket, int, family, int, type, int, protocol)
第二层: SYSCALL_DEFINEx
根据第234-236行定义:
#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
展开为:
SYSCALL_METADATA(_socket, 3, int, family, int, type, int, protocol)
__SYSCALL_DEFINEx(3, _socket, int, family, int, type, int, protocol)
第三层A: SYSCALL_METADATA 详细展开
根据第178-200行定义,先展开 __MAP 宏: __MAP 宏的展开过程: __MAP(3, __SC_STR_TDECL, int, family, int, type, int, protocol) 展开: 根据第120行和124行:
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
递归展开:
__SC_STR_TDECL(int,family), __SC_STR_TDECL(int,type), __SC_STR_TDECL(int,protocol)
根据第139行 #define __SC_STR_TDECL(t, a) #t,最终得到:
"int", "int", "int"
类似地,__MAP(3, __SC_STR_ADECL, ...) 根据第138行 #define __SC_STR_ADECL(t, a) #a,得到:
"family", "type", "protocol"
完整的 SYSCALL_METADATA 展开:
// 1. 定义参数类型名称数组
static const char *types__socket[] = {
    "int",      // __SC_STR_TDECL(int, family) -> "int"
    "int",      // __SC_STR_TDECL(int, type) -> "int"
    "int"       // __SC_STR_TDECL(int, protocol) -> "int"
};

// 2. 定义参数名称数组
static const char *args__socket[] = {
    "family",   // __SC_STR_ADECL(int, family) -> "family"
    "type",     // __SC_STR_ADECL(int, type) -> "type"
    "protocol"  // __SC_STR_ADECL(int, protocol) -> "protocol"
};

// 3. 创建系统调用进入事件 (展开 SYSCALL_TRACE_ENTER_EVENT)
// 根据第146-160行定义:
static struct syscall_metadata __syscall_meta__socket;  // 前向声明

static struct trace_event_call __used
  event_enter__socket = {
    .class = &event_class_syscall_enter,        // 进入事件类
    {
        .name = "sys_enter" "_socket",          // 事件名: "sys_enter_socket"
    },
    .event.funcs = &enter_syscall_print_funcs,  // 打印函数集
    .data = (void *)&__syscall_meta__socket,    // 指向元数据
    .flags = TRACE_EVENT_FL_CAP_ANY,            // 任何用户都可追踪
};

static struct trace_event_call __used
  __section("_ftrace_events")                   // 放入 ftrace 事件段
 *__event_enter__socket = &event_enter__socket; // 事件指针

// 4. 创建系统调用退出事件 (展开 SYSCALL_TRACE_EXIT_EVENT)
// 根据第162-176行定义:
static struct syscall_metadata __syscall_meta__socket;  // 前向声明

static struct trace_event_call __used
  event_exit__socket = {
    .class = &event_class_syscall_exit,         // 退出事件类
    {
        .name = "sys_exit" "_socket",           // 事件名: "sys_exit_socket"
    },
    .event.funcs = &exit_syscall_print_funcs,   // 打印函数集
    .data = (void *)&__syscall_meta__socket,    // 指向元数据
    .flags = TRACE_EVENT_FL_CAP_ANY,            // 任何用户都可追踪
};

static struct trace_event_call __used
  __section("_ftrace_events")                   // 放入 ftrace 事件段
 *__event_exit__socket = &event_exit__socket;   // 事件指针

// 5. 定义系统调用元数据结构
static struct syscall_metadata __used
  __syscall_meta__socket = {
    .name = "sys" "_socket",                    // 系统调用名: "sys_socket"
    .syscall_nr = -1,                           // 系统调用号(启动时填充)
    .nb_args = 3,                               // 参数个数: 3
    .types = 3 ? types__socket : NULL,          // 指向类型数组(3!=0,所以非NULL)
    .args = 3 ? args__socket : NULL,            // 指向参数名数组(3!=0,所以非NULL)
    .enter_event = &event_enter__socket,        // 指向进入事件
    .exit_event = &event_exit__socket,          // 指向退出事件
    .enter_fields = LIST_HEAD_INIT(__syscall_meta__socket.enter_fields), // 初始化链表
};

// 6. 创建元数据指针并放入特殊段
static struct syscall_metadata __used
  __section("__syscalls_metadata")              // 内核扫描此段注册系统调用
 *__p_syscall_meta__socket = &__syscall_meta__socket;
第三层B: __SYSCALL_DEFINEx 详细展开
根据第246-263行定义:
// 1. 禁用编译器警告
__diag_push();
__diag_ignore(GCC, 8, "-Wattribute-alias",
              "Type aliasing is used to sanitize syscall arguments");

// 2. 定义主系统调用函数 sys_socket,使用 alias 指向 __se_sys_socket
// __MAP(3, __SC_DECL, int, family, int, type, int, protocol) 展开:
// 根据第126行 #define __SC_DECL(t, a) t a
// 得到: int family, int type, int protocol
asmlinkage long sys_socket(int family, int type, int protocol)
    __attribute__((alias(__stringify(__se_sys_socket))));
    // 实际上是 sys_socket 的别名,指向 __se_sys_socket

// 3. 允许错误注入(用于测试)
ALLOW_ERROR_INJECTION(sys_socket, ERRNO);

// 4. 声明内部实现函数 __do_sys_socket (内联函数)
static inline long __do_sys_socket(int family, int type, int protocol);

// 5. 声明符号扩展包装函数 __se_sys_socket
// __MAP(3, __SC_LONG, int, family, int, type, int, protocol) 展开:
// 根据第131行:
// #define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
// 对于 int 类型,不是 long long,所以选择 0L,即 long 类型
// 得到: long family, long type, long protocol
asmlinkage long __se_sys_socket(long family, long type, long protocol);

// 6. 实现符号扩展包装函数
asmlinkage long __se_sys_socket(long family, long type, long protocol)
{
    // 调用实际实现,进行类型转换
    // __MAP(3, __SC_CAST, int, family, int, type, int, protocol) 展开:
    // 根据第132行 #define __SC_CAST(t, a) (__force t) a
    // 得到: (__force int) family, (__force int) type, (__force int) protocol
    long ret = __do_sys_socket((__force int) family, 
                               (__force int) type, 
                               (__force int) protocol);
    
    // 类型检查(编译时)
    // __MAP(3, __SC_TEST, ...) 展开为:
    // 根据第135行,检查类型大小不超过 long
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int) > sizeof(long));
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int) > sizeof(long));
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int) > sizeof(long));
    
    // 保护寄存器中的参数(防止被优化掉)
    // __MAP(3, __SC_ARGS, ...) 展开:
    // 根据第134行 #define __SC_ARGS(t, a) a
    // 得到: family, type, protocol
    __PROTECT(3, ret, family, type, protocol);
    
    return ret;
}

// 7. 恢复编译器诊断设置
__diag_pop();

// 8. 定义实际的内联实现函数(这是用户代码真正实现的地方)
static inline long __do_sys_socket(int family, int type, int protocol)
// 用户在这里写函数体:
{
    return __sys_socket(family, type, protocol);
}
最终完整展开总结
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol) 生成了:
元数据: types__socket[], args__socket[], __syscall_meta__socket
追踪事件: event_enter__socket, event_exit__socket
三个函数:
sys_socket(int, int, int) - 对外的系统调用接口
__se_sys_socket(long, long, long) - 符号扩展包装(处理32位到64位)
__do_sys_socket(int, int, int) - 实际实现(内联)
关键设计目的:
类型安全: 通过 __se_sys_socket 处理参数符号扩展
可追踪性: 通过元数据支持 ftrace/perf/eBPF
架构兼容: 统一处理不同架构的调用约定差异
性能: 内联的 __do_sys_socket 避免额外开销


用户空间程序
    |
    | syscall(SYS_socket, ...)
    ↓
┌─────────────────────────────────────────────────────────┐
│              内核系统调用入口                              │
└─────────────────────────────────────────────────────────┘
    |
    | 系统调用表查找
    ↓
[1] sys_socket(int, int, int)  ← 别名符号
    |                             (在符号表中可见)
    | alias 指向
    ↓
[2] __se_sys_socket(long, long, long)  ← 实际入口
    |                                     (符号扩展包装)
    | 类型转换: long → int
    | 参数检查
    ↓
[3] __do_sys_socket(int, int, int)  ← 系统调用实现
    |                                   (内联函数)
    | return __sys_socket(...)
    ↓
[4] __sys_socket(int, int, int)  ← 核心逻辑
    |                               (可被内核内部调用)
    | 创建 socket
    | 分配 fd
    ↓
    返回文件描述符


完整调用链: socket() → syscall() → sys_socket
┌─────────────────────────────────────────────────────────────┐
│                    用户空间 (User Space)                      │
└─────────────────────────────────────────────────────────────┘
    |
    | [1] 应用程序调用
    |
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    |
    | [2] glibc 包装函数
    ↓
┌─────────────────────────────────────────────────────────────┐
│  glibc: socket() wrapper                                    │
│  位置: glibc/sysdeps/unix/sysv/linux/socket.c               │
│                                                              │
│  int socket(int domain, int type, int protocol) {           │
│      return INLINE_SYSCALL_CALL(socket, domain, type,       │
│                                 protocol);                   │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
    |
    | [3] 展开为 syscall 调用
    ↓
    syscall(__NR_socket, AF_INET, SOCK_STREAM, 0);
    |
    | __NR_socket = 41 (x86-64)
    |
    | [4] syscall() 函数 (也在 glibc 中)
    ↓
┌─────────────────────────────────────────────────────────────┐
│  glibc: syscall() - 汇编实现                                 │
│  位置: glibc/sysdeps/unix/sysv/linux/x86_64/syscall.S       │
│                                                              │
│  syscall:                                                    │
│      movq %rdi, %rax      # 系统调用号 -> rax (41)          │
│      movq %rsi, %rdi      # arg1 -> rdi (AF_INET)           │
│      movq %rdx, %rsi      # arg2 -> rsi (SOCK_STREAM)       │
│      movq %rcx, %rdx      # arg3 -> rdx (0)                 │
│      syscall              # 执行 syscall 指令               │
│      ret                                                     │
└─────────────────────────────────────────────────────────────┘
    |
    | [5] syscall 指令 - CPU 模式切换
    |
    | 硬件动作:
    | - RIP 保存到 RCX
    | - RFLAGS 保存到 R11
    | - 切换到 Ring 0 (内核态)
    | - 加载内核栈
    | - 跳转到 MSR_LSTAR 寄存器指向的地址
    ↓
┌─────────────────────────────────────────────────────────────┐
│                    内核空间 (Kernel Space)                   │
└─────────────────────────────────────────────────────────────┘
    |
    | [6] 内核系统调用入口
    ↓
entry_SYSCALL_64:  (arch/x86/entry/entry_64.S:87)
    |
    | 保存现场:
    | - swapgs (切换 GS 段)
    | - 保存用户栈到 TSS
    | - 切换到内核栈
    | - 构造 pt_regs 结构
    |
    | pushq %rax    # 系统调用号 (41)
    | pushq %rdi    # arg1 (family)
    | pushq %rsi    # arg2 (type)
    | pushq %rdx    # arg3 (protocol)
    | ... (保存其他寄存器)
    |
    | movq %rsp, %rdi      # pt_regs* -> rdi
    | movslq %eax, %rsi    # syscall_nr -> rsi (符号扩展)
    |
    | [7] 调用 C 函数
    ↓
    call do_syscall_64    (arch/x86/entry/syscall_64.c:87)
    |
    | do_syscall_64(struct pt_regs *regs, int nr)
    | {
    |     nr = syscall_enter_from_user_mode(regs, nr);
    |     
    |     [8] 调用分发函数
    |     ↓
    |     do_syscall_x64(regs, nr);
    | }
    ↓
do_syscall_x64(struct pt_regs *regs, int nr)  (syscall_64.c:53)
    |
    | if (likely(nr < NR_syscalls)) {
    |     nr = array_index_nospec(nr, NR_syscalls);  # 防御 Spectre
    |
    |     [9] 通过 switch-case 分发
    |     ↓
    |     regs->ax = x64_sys_call(regs, nr);
    | }
    ↓
x64_sys_call(const struct pt_regs *regs, unsigned int nr)  (syscall_64.c:35)
    |
    | switch (nr) {
    |     case 0: return __x64_sys_read(regs);
    |     case 1: return __x64_sys_write(regs);
    |     ...
    |     [10] 匹配到 case 41
    |     ↓
    |     case 41: return __x64_sys_socket(regs);
    |     ...
    | }
    ↓
__x64_sys_socket(const struct pt_regs *regs)  (自动生成的包装器)
    |
    | # 从 pt_regs 提取参数
    | int family   = regs->di;
    | int type     = regs->si;
    | int protocol = regs->dx;
    |
    | [11] 调用系统调用入口 (别名)
    ↓
    return sys_socket(family, type, protocol);
    |
    | sys_socket 是 __se_sys_socket 的别名
    | (通过 __attribute__((alias(...))) 定义)
    |
    | [12] 实际执行
    ↓
__se_sys_socket(long family, long type, long protocol)
    |
    | # 类型转换和保护
    | long ret = __do_sys_socket(
    |     (__force int)family,
    |     (__force int)type,
    |     (__force int)protocol
    | );
    |
    | __PROTECT(3, ret, family, type, protocol);
    | return ret;
    ↓
__do_sys_socket(int family, int type, int protocol)  (内联)
    |
    | return __sys_socket(family, type, protocol);
    ↓
__sys_socket(int family, int type, int protocol)  (net/socket.c:1742)
    |
    | # 实际的 socket 创建逻辑
    | sock = __sys_socket_create(family, type, protocol);
    | fd = sock_map_fd(sock, flags);
    | return fd;
    |
    | [返回值通过调用链回传]
    ↓
    返回文件描述符 (如 3)
    |
    | [13] 系统调用返回
    ↓
entry_SYSCALL_64 (继续执行)
    |
    | # 恢复用户态寄存器
    | # 设置 rax = 返回值
    | sysretq    # 快速返回用户态
    ↓
┌─────────────────────────────────────────────────────────────┐
│                    用户空间 (返回)                            │
└─────────────────────────────────────────────────────────────┘
    |
    | rax = 3 (文件描述符)
    |
    ↓
    int fd = 3;  // 应用程序得到结果
