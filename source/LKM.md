---
layout: post
id: LKM
title: 内核可加载模块
prev: BOOT_ANALYSIS
---




内核可加载模块（LKM, Loadable Kernel Module）是把一部分内核功能做成“可插拔”的二进制模块：系统运行中可以把它**加载进内核地址空间**（扩展内核能力），也可以**卸载**（移除功能/释放资源），从而避免为加一个驱动/文件系统/网络功能就必须重启并更换整套内核。

------

## 1) LKM解决了什么问题

- **按需加载**：只在需要时加载对应功能（比如某个网卡驱动、某个文件系统）。
- **降低维护成本**：驱动/子系统更新不必重新编译整个内核；调试迭代更快。
- **减少内核体积**：不常用功能不必常驻内核镜像。
- **更灵活的发行/部署**：厂商可以单独交付模块（前提是版本/符号匹配）。

------

## 2) LKM能做什么（常见类型）

在 Linux 语境里，LKM 常见用途包括：

- **设备驱动**：字符设备、块设备、网卡、USB、GPU、声卡等。
- **文件系统**：如某些文件系统驱动、加密/压缩相关扩展。
- **网络协议/Netfilter**：协议栈扩展、iptables/nftables 相关模块。
- **安全/监控**：LSM 扩展、审计、探针（部分能力也可能用 eBPF 实现）。

------

## 3) 工作机制（内核视角）

加载一个模块时大致发生：

1. **把 .ko（kernel object）装入内核**：模块代码/数据被映射到内核空间。
2. **重定位与符号解析**：模块引用内核导出的符号（函数/变量）；需要能解析到匹配的内核符号表。
3. **执行初始化入口**：模块的 init 函数被调用（注册驱动、创建设备节点、注册文件系统、挂钩回调等）。
4. 卸载时执行 **exit 入口**：撤销注册、释放资源、确保无引用后移除。

模块通常会有“依赖模块”，因此加载器会尝试先加载依赖（或要求你先加载）。

------

## 4) 模块与内核版本/ABI兼容性

LKM有个很现实的约束：它必须和当前运行的内核**在接口/符号层面兼容**，否则会出现：

- “Unknown symbol …”
- “Invalid module format”
- “Version magic mismatch”
- 或者更糟：能加载但运行崩溃（接口语义变了）

Linux 内核并不承诺稳定的内核内部 ABI，因此常见做法是：**模块与内核同版本构建**，或使用发行版提供的 **kABI/符号白名单**（各发行版策略不同）。

------

## 5) 安全性与风险

LKM属于**最高权限代码**（ring0），一旦有 bug 或恶意行为，影响极大：

- **稳定性风险**：空指针、越界、死锁、引用计数错误 → 直接 kernel panic。
- **安全风险**：恶意模块可隐藏进程/文件/网络连接（rootkit 常用手段之一）。
- **防护措施**（Linux常见思路）：
  - **模块签名/安全启动链（Secure Boot）**：只允许加载签名模块（视配置）。
  - **限制模块加载**：某些系统会禁止运行时加载（或仅允许管理员且强约束）。
  - **最小化内核攻击面**：不需要的模块不装、不自动加载。

------

## 6) 和 eBPF 的区别（简要对比）

- **LKM**：可做的事更“底层/自由”，但风险更高、开发复杂、对内核版本更敏感。
- **eBPF**：通常在验证器/受控沙箱模型下运行，适合观测、过滤、网络/安全策略等；受限于验证器与可用 hook 点，但**更安全、更易部署**（很多场景已替代“写个小模块”).

------

## 7) 一个最小的心智模型

可以把 LKM 理解为：

- “内核的动态链接库（但更危险）”
- 具有明确生命周期：**load → init → 提供服务 → exit → unload**
- 通过“注册/回调/hook”方式把自己接入内核框架（驱动模型、VFS、netfilter 等）
  

## 

# eBPF

eBPF（extended Berkeley Packet Filter）是一种在 Linux 内核里运行的“受控小程序”机制：你把一段 eBPF 程序加载进内核，它会在特定的内核事件点（hook）被触发执行，用于**观测、过滤、统计、跟踪、网络处理**等；内核会通过**验证器（verifier）**和**受限的运行时模型**尽量保证它不会把内核搞崩。

------

## 1) eBPF能用来做什么（常见场景）

- **可观测性/性能分析**：跟踪系统调用、调度、内存分配、块 IO、网络栈路径等（常见工具栈：bpftrace、BCC、perf 的一些能力也可结合）。
- **安全**：运行时策略与审计（例如基于事件的告警/阻断），部分场景可与 LSM/eBPF LSM 结合。
- **网络/云原生**：高性能包处理、负载均衡、流量整形、DDoS 缓解、容器网络（如 XDP/TC 路径；Cilium 等方案大量使用）。
- **动态故障排查**：线上临时挂探针收集数据，不用重启、不用插内核模块（很多场景）。

------

## 2) 它和“写内核模块”有什么本质区别

- **安全模型不同**
  - LKM：完全内核权限，几乎“想干什么都能干”，也最容易把系统搞崩。
  - eBPF：程序要先过 **verifier**（静态检查），并在受限环境里跑；不能随意访问内存，只能通过“helper”或受控指针访问特定数据结构。
- **部署/迭代方式不同**
  - eBPF：更像“加载脚本/字节码”，可动态挂载到 hook 点；更适合线上快速诊断和策略更新。
  - LKM：需要编译成内核模块并处理符号/版本兼容、签名等。
- **能力边界**
  - eBPF 很强，但不是万能：能做的事取决于内核提供的 hook 点与 helper；某些深度改动需内核代码/LKM。

------

## 3) eBPF是怎么跑起来的（简化流程）

1. **编写程序**：常用 C/Clang 编译为 eBPF 字节码（或用 bpftrace 这种高级脚本语言）。
2. **加载到内核**：通过 `bpf()` 系统调用（通常由 libbpf、bcc、bpftrace 等封装）。
3. **验证（verifier）**：检查程序是否安全（例如：不会无界循环、不会越界访问、栈/指针使用是否可证明安全等）。
4. **附加到 hook**：比如 XDP、TC、kprobe、tracepoint、LSM、cgroup 等。
5. **运行与通信**：通过 eBPF map（内核/用户态共享的数据结构）把统计信息、事件数据传回用户态；或通过 ring buffer/perf buffer 推送事件。
6. **JIT（可选）**：多数系统会把 eBPF 字节码 JIT 成机器码提升性能。

------

## 4) 你可能听过的关键词（快速解释）

- **XDP**：在网卡驱动更早的位置处理包，极快，适合丢包/转发/负载均衡等。
- **TC（Traffic Control）**：在 Linux 流量控制路径挂程序，功能强，位置相对靠后。
- **kprobe/tracepoint**：内核动态/静态探针，用于跟踪内核函数或固定事件点。
- **uprobes**：给用户态程序函数打探针。
- **maps**：键值存储/数组/哈希/队列等，用于状态保存与内核↔用户态交换数据。
- **helpers**：内核提供的受控 API，eBPF 通过它们完成取时间、取进程信息、操作 map、输出日志等。



## kmod

## 1）整体架构

```test
┌──────────────────────────────────────────────────────────────────┐
│                    构建时 (Host)                                  │
│                                                                  │
│  modules/hello/src/lib.rs  ──cargo build──→ libhello.rlib        │
│                              ──ld -r──→ hello.ko (可重定位 ELF)   │
│                                                                  │
│  内核 ELF ──nm──→ gen_ksym ──→ kallsyms (二进制符号表文件)         │
│                                                                  │
│  hello.ko + kallsyms ──→ 写入磁盘镜像 /root/modules/             │
├──────────────────────────────────────────────────────────────────┤
│                    运行时 (内核)                                   │
│                                                                  │
│  用户态: insmod /root/modules/hello.ko                           │
│    ↓ sys_init_module / sys_finit_module                          │
│    ↓                                                             │
│  ┌─────────────────────────────────────────────┐                 │
│  │ api/src/kmod/mod.rs — init_module()         │                 │
│  │   ↓                                         │                 │
│  │ kmod-loader (外部 crate):                    │                 │
│  │   1. 解析 ELF sections (.text/.data/.bss)   │                 │
│  │   2. vmalloc 分配内核页面                     │                 │
│  │   3. 复制段内容到页面                         │                 │
│  │   4. 重定位: 查 kallsyms 解析外部符号         │                 │
│  │   5. 设置页面权限 (RX/RW)                    │                 │
│  │   6. 调用 module_init()                      │                 │
│  └─────────────────────────────────────────────┘                 │
│             ↕ 符号解析                                            │
│  ┌─────────────────────────────────────────────┐                 │
│  │ kallsyms 符号表                              │                 │
│  │  (从 /root/kallsyms 文件加载到内存)           │                 │
│  │  lookup_name("mount_at") → 0xffff...1234    │                 │
│  └─────────────────────────────────────────────┘                 │
│             ↕ C ABI 垫片                                         │
│  ┌─────────────────────────────────────────────┐                 │
│  │ api/src/kmod/shim/ — 内核 API 垫片层         │                 │
│  │  kprint.rs: _printk, snprintf, sprintf      │                 │
│  │  block.rs:  __register_blkdev, device_add.. │                 │
│  │  mq.rs:    blk_mq_* (块设备多队列)           │                 │
│  │  xarray.rs: XArray 操作                      │                 │
│  └─────────────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────────────┘
```



## 文件清单 + 功能 + 代码解读

### 1. 构建系统

#### kmod.mk — 模块构建入口

```makefile
$(MODULES):
    KMOD=y APP=$(abspath $(MODULE_PATHS)/$@) make -C arceos build_ko
```

遍历 modules 目录下的每个子目录，对每个模块调用 arceos 的 `build_ko` 目标。

#### arceos `scripts/make/build.mk` 中的 `$(OUT_KO)` 规则

```makefile
$(OUT_KO): oldconfig
    $(call cargo_build,$(APP),$(AX_FEAT) $(LIB_FEAT) $(APP_FEAT))
    $(call run_cmd,$(LD), -r -T $(KMOD_LINKER_SCRIPT) -o $@ \
        --whole-archive $(rust_lib) --strip-debug --build-id=none --gc-sections)
```

**流程**：
1. `cargo build` — 把模块编译为 `libhello.rlib`（Rust 静态库）
2. `ld -r` — **关键**！`-r` 表示"partial link"（部分链接），产出可重定位 ELF（`.ko`），保留所有未解析的外部符号（如 `write_char`、`alloc` 等），这些符号会在运行时由内核符号表解析

#### kmod-linker.ld — 模块链接脚本

```ld
SECTIONS {
    . = 0x00000000;           ← 基地址为 0，运行时由加载器重定位
    .modinfo : { ... }        ← 模块元信息（name, license, version）
    .text    : { ... }        ← 代码段（包含 init/exit 入口）
    .gnu.linkonce.this_module ← Linux 兼容的模块描述符
    .data    : { ... }        ← 已初始化数据
    .bss     : { ... }        ← 未初始化数据
    /DISCARD/ : { ... }       ← 丢弃调试信息
}
```

#### arceos `build.mk` 中的 `$(OUT_KSYM)` 规则 — 内核符号表生成

```makefile
$(OUT_KSYM): _cargo_build
    nm -n -C $(OUT_ELF) | grep ' [TtDBR] ' | grep -v '\.L' | \
        RUSTFLAGS= gen_ksym > $@
```

用 `nm` 从编译好的内核 ELF 中提取所有全局符号（函数、数据），通过 `gen_ksym` 工具（来自 [Starry-OS/ksym](https://github.com/Starry-OS/ksym) crate）编码为紧凑的二进制格式 `kallsyms`。

#### Makefile `img` 目标 — 组装磁盘镜像

```makefile
img: build
    mount disk.img ./disk
    cp kallsyms ./disk/root/kallsyms        ← 符号表写入磁盘
    make kmod                                ← 编译所有模块
    cp ./*.ko ./disk/root/modules/           ← .ko 写入磁盘
    umount ./disk
```

---

### 2. 内核符号表

#### mod.rs `mount_all()` 中加载

```rust
fn read_kallsyms() -> LinuxResult<Vec<u8>> {
    // 从磁盘读取 /root/kallsyms 二进制文件
    OpenOptions::new().read(true).open(&FS_CONTEXT.lock(), "/root/kallsyms")...
}

pub fn mount_all() -> LinuxResult<()> {
    let kallsyms = read_kallsyms()?;
    let kallsyms = kallsyms.leak();  // 泄漏到 'static 生命周期
    let ksym = ksym::KallsymsMapped::from_blob(kallsyms, _stext as u64, _etext as u64);
    // ksym 存入 KALLSYMS（LazyInit<KallsymsMapped>）
    ...
}
```

`KallsymsMapped` 提供：
- `lookup_name("fn_name") → Option<u64>` — 按名字查地址（模块加载时用）
- `lookup_address(addr) → Option<(name, size, offset, type)>` — 按地址查名字（panic 回溯时用）

---

### 3. 核心模块加载器

#### mod.rs — 加载/卸载入口

**关键类型**：
```rust
pub struct KmodHelper;  // 实现 kmod_loader::KernelModuleHelper trait
```

**三个 trait 方法**：

| 方法                      | 功能                        | 实现                                                |
| ------------------------- | --------------------------- | --------------------------------------------------- |
| `vmalloc(size)`           | 为模块代码/数据分配内核页面 | `alloc_frames()` 分配物理帧 → `phys_to_virt()` 映射 |
| `resolve_symbol(name)`    | 解析模块引用的外部符号      | `KALLSYMS.get().lookup_name(name)` 查符号表         |
| `flush_cache(addr, size)` | 刷新指令缓存                | `flush_tlb(None)`                                   |

**内存管理**（`KmodMem`）：
```rust
struct KmodMem { paddr, vaddr, num_pages }
impl SectionMemOps for KmodMem {
    fn as_mut_ptr(&mut self) → 返回虚拟地址指针（供加载器写入代码/数据）
    fn change_perms(&mut self, perms) → 修改页表权限（加载完成后 .text 设 RX，.data 设 RW）
}
impl Drop for KmodMem {
    fn drop() → dealloc_frames() 释放物理帧
}
```

**加载流程** (`init_module`):
```rust
pub fn init_module(elf: &[u8], params: Option<&str>) -> AxResult<()> {
    // 1. 创建加载器，解析 ELF
    let loader = ModuleLoader::<KmodHelper>::new(elf)?;
    // 2. 加载模块（分配内存、复制段、重定位、设置权限）
    let mut owner = loader.load_module(params)?;
    // 3. 调用模块的 init 函数
    let res = owner.call_init()?;
    // 4. 保存到全局模块表
    MODULES.lock().insert(name, owner);
}
```

**卸载流程** (`delete_module`):
```rust
pub fn delete_module(name: &str) -> AxResult<()> {
    let mut owner = MODULES.lock().remove(name)?;
    owner.call_exit();  // 调用模块的 exit 函数
    // owner drop → KmodMem drop → dealloc_frames
}
```

**外部 crate 依赖**：
- `kmod-loader`（来自 [Starry-OS/rkm](https://github.com/Starry-OS/rkm)）：ELF 解析 + 段加载 + 重定位引擎
- `kmod`：提供 `#[init_fn]`、`#[exit_fn]`、`module!` 宏、`#[capi_fn]`、`kbindings::*`（C 类型绑定）
- `kapi`：内核 API 接口定义
- `ksym`：内核符号表格式和查询

---

### 4. 系统调用入口

#### mod.rs — 三个系统调用

| 系统调用                              | 功能                         | 用户态触发方式            |
| ------------------------------------- | ---------------------------- | ------------------------- |
| `sys_init_module(ptr, len, params)`   | 从用户态内存缓冲区加载 `.ko` | `insmod` 读文件到内存再传 |
| `sys_finit_module(fd, params, flags)` | 从文件描述符加载 `.ko`       | `insmod` 直接传 fd        |
| `sys_delete_module(name, flags)`      | 按名称卸载模块               | `rmmod hello`             |

`sys_finit_module` 流程：
```rust
pub fn sys_finit_module(module_fd: i32, ...) -> AxResult<isize> {
    let file = get_file_like(module_fd)?;       // 从 fd 获取文件
    let mut module_data = Vec::with_capacity(fsize);
    file.read(&mut module_data)?;               // 读取整个 .ko 到内存
    crate::kmod::init_module(&module_data, ...)?; // 交给加载器
    Ok(0)
}
```

---

### 5. C ABI 垫片层

#### shim — 让 .ko 能调用"内核函数"

模块编译时引用了很多 Linux 内核 C 函数符号（如 `_printk`、`__kmalloc_noprof`、`__register_blkdev` 等）。这些符号需要在 StarryOS 内核中有实现，否则模块加载时 `resolve_symbol` 会 panic。

垫片层用 `#[capi_fn]` 宏标注函数，使其符号名和 C ABI 兼容，编译进内核后会出现在 `kallsyms` 符号表中：

| 文件           | 垫片内容                                                     | 行数    |
| -------------- | ------------------------------------------------------------ | ------- |
| shim/kprint.rs | `_printk`、`snprintf`、`sprintf`、`write_char` — 打印/格式化 | ~91 行  |
| shim/block.rs  | `__register_blkdev`、`device_add_disk` — 块设备注册          | ~149 行 |
| shim/mq.rs     | `blk_mq_*` — 块设备多队列请求处理                            | ~960 行 |
| shim/xarray.rs | XArray 数据结构操作                                          | ~12 行  |
| shim/mod.rs    | `kmalloc`、`mutex`、`ida_alloc`、大量 `not_impl!()` 桩       | ~242 行 |

大量函数用 `not_impl!()` 宏实现为空桩（打印 error 日志返回 0），说明这是一个**进行中的工作**，只实现了 hello 模块和 null_blk 块设备驱动需要的最小子集。

---

### 6. 示例模块

#### lib.rs — 最简模块

```rust
#![no_std]
extern crate alloc;
extern crate starry_api;

#[init_fn]
pub fn hello_init() -> i32 {         // insmod 时执行
    // 通过垫片层的 write_char 输出
    write_char(b'H'); write_char(b'e'); ...
    let v = vec![1,2,3,4,5];        // 可以用 alloc!
    0                                // 返回 0 表示成功
}

#[exit_fn]
fn hello_exit() { ... }             // rmmod 时执行

module!(name: "hello", license: "GPL", ...);
```

#### lib.rs — eBPF 模块（更复杂的示例）

```rust
#[init_fn]
pub fn kebpf_init() -> i32 {
    // 关键：动态注册 bpf 系统调用处理器
    starry_api::syscall::register_syscall_handler(Sysno::bpf, &Handler);
    0
}
```

这个模块展示了 LKM 的强大能力：**模块可以在运行时给内核添加新的系统调用**。`register_syscall_handler` 把处理函数插入全局 `SYSCALL_HANDLER: RwLock<BTreeMap<Sysno, &dyn SyscallHandler>>`，之后用户态调用 `syscall(SYS_bpf, ...)` 就会路由到模块代码。

---

### 7. Syscall 派发改造

#### mod.rs — 支持动态 handler

```rust
_ => {
    // 所有未硬编码的 syscall → 查动态 handler 表
    if let Some(handler) = SYSCALL_HANDLER.read().get(&sysno) {
        handler.handle(uctx)
    } else {
        warn!("Unimplemented syscall: {sysno}");
        Err(AxError::Unsupported)
    }
}
```

---

## 完整逻辑链

```
编译时:
  内核 ELF ──nm + gen_ksym──→ kallsyms 文件
  modules/hello/ ──cargo build──→ libhello.rlib ──ld -r──→ hello.ko
  kallsyms + hello.ko ──→ 写入磁盘镜像 /root/

运行时:
  内核启动 → mount_all() → read_kallsyms("/root/kallsyms")
                         → KallsymsMapped::from_blob() → 存入 KALLSYMS
                         → kmod::init_kmod() (初始化 lwprintf)

  用户态: insmod /root/modules/hello.ko
    → sys_finit_module(fd, params, flags)
    → 读取 .ko 文件到 Vec<u8>
    → kmod::init_module(elf_bytes, params)
    → ModuleLoader::new(elf_bytes)           // 解析 ELF
    → loader.load_module(params)
        → KmodHelper::vmalloc(text_size)     // alloc_frames 分配物理页
        → 复制 .text/.data 段到页面
        → 遍历重定位条目:
            → KmodHelper::resolve_symbol("write_char")
            → KALLSYMS.lookup_name("write_char") → 0xffff...addr
            → 写入重定位地址
        → KmodMem::change_perms(.text → RX)
        → KmodMem::change_perms(.data → RW)
    → owner.call_init()                      // 跳转执行 hello_init()
        → "Hello, Kernel Module!"
    → MODULES.insert("hello", owner)         // 保存到全局模块表

  用户态: rmmod hello
    → sys_delete_module("hello")
    → MODULES.remove("hello")
    → owner.call_exit()                      // 跳转执行 hello_exit()
    → drop(owner) → drop(KmodMem) → dealloc_frames  // 释放内存
```

