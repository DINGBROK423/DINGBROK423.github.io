---
id: week7
title: Week 7
prev: week6
next: week8
---

## 2026.3.23-2026.3.29

### 工作内容

1. 手写用户态文件系统内核部分逻辑（Starryfuse）。

2. 完成用户态文件系统内核态按需加载集成，并通过按需加载测试。

3. 正在调试接入Fuse用户态库测试。

#### 本周核心代码变更总结

##### 1. 工作空间依赖与构建梳理
- **Submodule 修正**：
  - 清理了意外引入的 `alloystack` 子模块，避免构建污染 (commit: `2ca5cc9`)。
  - 为修复 `fuse.ko` 链接问题，升级并同步了 `arceos` 子模块指针 (commit: `d4c5ff4`)。
- **构建系统集成**：在工作空间及构建脚本中增加规则，确保 `make modules` 能通过 Rust BPF/Kmod 工具链正确编译 `modules/fuse` 并最终输出 `/root/modules/fuse.ko`。

##### 2. Starryfuse 独立协议核心库实现
将 FUSE 核心机制独立剥离为单独的 Crate (`Starryfuse/Cargo.toml`) 全新引入。主要覆盖以下代码：
- **`abi.rs`**：规范对齐了 Linux 标准中 FUSE 协议层的全部上下行（请求/响应）数据排布。
- **`dev.rs`**：专门实现了基于 `FuseDev` 抽象的通信设备节点出入队操作管理（`poll`, `read`, `write`）。
- **`vfs.rs`**：FUSE 层面定制的内挂虚文件系统结构和索引分配屏蔽接口。
- **`lib.rs`**：提供供内核初始化的统一入口和全局状态管理。

##### 3. 构建独立的 FUSE 模块 (`modules/fuse/`)
- 提供了标准的 `kmod` 生命周期函数包裹：
```rust
#[init_fn]
pub fn fuse_init() -> i32 {
    let ret = starryfuse::init_fuse();
    // ...
}

#[exit_fn]
fn fuse_exit() {
    starryfuse::exit_fuse();
}
```
- 将具体的 `devfs` 节点与协议管道注册完全交给下层 `starryfuse` 库实例。

##### 4. StarryOS VFS 层按需加载触发入口注册 (`api/src/kmod/ondemand_builtin.rs`)
通过 `register_builtin_modules` 正式将 FUSE 接轨至 StarryOS On-Demand 框架：
```rust
super::ondemand::register_module(ModuleDesc {
    name: "fuse",
    ko_path: "/root/modules/fuse.ko",
    idle_timeout_ticks: 5_000,
    trigger: Box::new(PathPrefixTrigger::new("/dev/fuse")),
    usage: Some(Box::new(FuseUsageChecker)),
});
```

##### 5. 修复 Ext4 缓存崩溃的 VFS 路径安全检查 (Usage Checker 核心改动)
这是近期最重要的代码修复。解决了老版本在全量轮询 FD 时粗暴调用 `.path()` 引发的 `ext4_bcache_free` 核级崩溃。修复后的检测器加入了底层挂载点分类校验，解决了因为时钟中断导致的主存盘缓存读写冲突。
```rust
// api/src/kmod/ondemand_builtin.rs 最终版修复片段
if let Some(file) = any.downcast_ref::<crate::file::File>() {
    let loc = file.inner().location();
    let fs_name = loc.filesystem().name();
    
    if fs_name == "fuse" { return true; } 
    // 【核心修复点】: 仅在内存文件系统 devfs 才安全解包路径，避开磁盘 I/O 雷区
    else if fs_name == "devfs" {
        let fpath = crate::file::FileLike::path(file);
        if fpath == "/dev/fuse" || fpath == "/fuse" || fpath == "fuse" {
            return true;
        }
    }
}
```

##### 6. VFS 动态文件系统注册机制与底层硬编码解耦 (当前状态核心改动)
主系统内核通过解耦改造，进一步去除了对特定文件系统的硬绑定，提供真正完善的动态化支持：
- **动态注册表 (`api/src/vfs/mod.rs`)**：新增了包含无锁映射表的 `FS_REGISTRY` 与对外接口 `register_filesystem` / `get_filesystem_creator`。这使得 `.ko` 动态外挂模块可以自行向 VFS 注册文件系统的创建闭包，而不是在内核写死。
- **`sys_mount` 改造 (`api/src/syscall/fs/mount.rs`)**：系统调用 `sys_mount` 在处理挂载时不再使用大量的 if-else 硬编码探测文件系统，而是通过动态匹配 `get_filesystem_creator(&fs_type)` 获取目标挂载逻辑。实现了对外部模块的零感知。
- **卸载特化逻辑清理 (`api/src/kmod/ondemand.rs`)**：彻底移除了按需加载器内部针对 `procfs` 硬编码的 "先 unmount 再立即 delete 模块" (Two-phase unload) 特化脏代码。让 `ondemand` 管理器回归纯理性的通用生命周期管控，杜绝模块特权耦合。


##### 7. 用户态 FUSE 守护进程测试 (`fuse_test`) 
额外引入并集成运行了用户态 FUSE 后台测试应用，用于印证完整的交互闭环与稳定性：
- 测试驱动从用户态发起初始 `open("/dev/fuse")` 引发由 VFS Not Found 到内核执行 On-Demand LKM 模块加装的全过程。
- 确立了后台主态轮询事件循环读取 `fuse_in_header` 及相关特约请求 (`lookup`, `getattr` 等)。
- 将模拟好的结构回写确认，标志着 `StarryOS VFS -> Fuse 设备文件 -> Kernel Mod -> User Daemon` 从底至上逻辑贯通。

##### 8. VFS 动态文件系统注册机制与底层硬编码解耦 (当前状态核心改动)
主系统内核通过解耦改造，进一步去除了对特定文件系统的硬绑定，提供真正完善的动态化支持：
- **动态注册表 (`api/src/vfs/mod.rs`)**：新增了包含无锁映射表的 `FS_REGISTRY` 与对外接口 `register_filesystem` / `get_filesystem_creator`。这使得 `.ko` 动态外挂模块可以自行向 VFS 注册文件系统的创建闭包，而不是在内核写死。
- **`sys_mount` 改造 (`api/src/syscall/fs/mount.rs`)**：系统调用 `sys_mount` 在处理挂载时不再使用大量的 if-else 硬编码探测文件系统，而是通过动态匹配 `get_filesystem_creator(&fs_type)` 获取目标挂载逻辑。实现了对外部模块的零感知。
- **卸载特化逻辑清理 (`api/src/kmod/ondemand.rs`)**：彻底移除了按需加载器内部针对 `procfs` 硬编码的 "先 unmount 再立即 delete 模块" (Two-phase unload) 特化脏代码。让 `ondemand` 管理器回归纯理性的通用生命周期管控，杜绝模块特权耦合。

---
### 用户态文件系统（FUSE）按需加载设计

**核心理念**

系统启动时，内核中不包含任何与 FUSE 相关的常驻驱动代码。只有当用户态程序（如 `fuse_test`）首次试图访问 `/dev/fuse` 字符设备时，VFS 捕获到 NotFound，系统才会动态将 `fuse.ko` 装载至内核，创建 FUSE 字符设备节点并开启 IPC（进程间通信）。当所有 FUSE 文件系统挂载点被卸载，且 `/dev/fuse` 句柄被彻底关闭超时后，内核将自动彻底卸载 `fuse.ko` 释放内存。

**架构图**

```text
┌─────────────────────────────────────────────────────────────┐
│                    User Space (应用程序)                      │
│   fuse_test: open("/dev/fuse")   /   mount("/mnt/fuse")     │
└────────┬────────────────────────────────────────────────────┘
         │ 触发
         ▼
┌─────────────────────────────────────────────────────────────┐
│               StarryOS VFS (api/src/syscall/)                 │
│                                                             │
│   with_ondemand("/dev/fuse", || { VFS resolve })            │
│    └─ 解析失败 → 触发 try_ondemand_load_path()               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                ondemand-kmod 框架 & 模块加载                  │
│                                                             │
│  加载 /root/modules/fuse.ko ────────► 映射入内核内存          │
└────────────────────────┬────────────────────────────────────┘
                         │ 
                         ▼
┌─────────────────────────────────────────────────────────────┐
│               模块初始化 (modules/fuse/)                      │
│                                                             │
│   fuse_init() ───►  starryfuse::init_fuse()                 │
│                            │                                │
│                   (注：此处不碰复杂的 FS_CONTEXT)              │
│                            │                                │
│                            ▼                                │
│       API 调用: register_devfs_device("fuse", 10, 229)      │
│        └► 在内存文件系统 devfs 挂载点创建 /dev/fuse 节点         │
└─────────────────────────────────────────────────────────────┘
```

**集成与执行流程**

1. 用户进程（如 `fuse_test`）发起 `open("/dev/fuse", O_RDWR)`。
2. StarryOS VFS `sys_openat` 层向下透传，发现找不到路径。触发 `with_ondemand` hook。
3. 加载器挂载 `fuse.ko`。
4. 模块初始化仅调用 `register_devfs_device`，在 `devfs` 的内存映射树种插入 FUSE Node。
5. 原本的 VFS 请求重新执行，此时成功抓到该节点，并将其作为文件描述符 (fd) 发给该用户程序。
6. Timer 中断每隔 tick() 唤醒 `ondemand` 后台监视器。监视器调用 `FuseUsageChecker` 轮询 VFS 与 FD，确认是否有活跃 FUSE 引用，以此管理空闲回收倒计时。

#### Starryfuse独立库

**位置**：`/workspaces/StarryOS/Starryfuse/*` （之后会发布成外部库）

本阶段完成了用户态文件系统（FUSE）在内核态的支持层库（Starryfuse）的开发工作。整体架构与标准 FUSE 协议保持兼容，主要分为入口注册、字符设备通信、VFS对接近层以及ABI协议定义四部分。具体代码架构与实现情况如下：

##### 1. 核心协议定义：`src/abi.rs`

*   **功能定位**：FUSE 协议的核心数据结构、枚举与相关常量的定义层。
*   **实现细节**：
    *   定义了 FUSE 协议操作码（`FuseOpcode`），支持如 `Lookup`, `Getattr`, `Read`, `Write`, `Readdir`, `Create`, `Unlink` 等常用操作。
    *   按照 Linux ABI 标准，实现了 C 内存布局（`#[repr(C)]`）的请求与响应头结构体（`FuseInHeader`, `FuseOutHeader`）。
    *   定义了各类具体操作对应的底层结构数据模型，如初始化结构 (`FuseInitIn/Out`)、属性（`FuseAttr`）、目录项（`FuseDirent`）等。

##### 2. 通信设备抽象：`src/dev.rs`

*   **功能定位**：用户态 FUSE 守护进程与内核交互的通道实现。
*   **实现细节**：
    *   抽象了 `FuseConnection`，用于管理 FUSE 协议过程中的请求流转状态，包含 `pending`（等待用户态读取）和 `processing`（等待用户态返回）两个请求队列。
    *   实现了 fuse 字符设备的底层操作模型（`DeviceOps` 接口）。
    *   **读操作 (`read_at`)**：用户态守护进程读取 fuse 时，从 `pending` 队列取出 `FuseRequest` 请求，交由用户态处理并转移到 `processing`。
    *   **写操作 (`write_at`)**：用户态守护进程处理完成后向 fuse 回写，内核在此解析 `FuseOutHeader`，通过 `unique` ID 匹配 `processing` 中的请求并将其标记为已完成。

##### 3. 虚拟文件系统对接：`src/vfs.rs`

*   **功能定位**：文件系统逻辑的内核侧抽象，负责将内核 VFS 的标准调用转换为 FUSE 请求。
*   **实现细节**：
    *   针对 `axfs_ng_vfs` 框架实现了文件系统接口的核心特征，包括 `FilesystemOps`, `NodeOps`, `FileNodeOps` 以及 `DirNodeOps`。
    *   **请求发送机制**：实现了核心方法 `send_request`，用于将 VFS 请求封装为 `abi` 定义的格式，推入 `dev` 的等待队列，并通过 `yield_now()` 主动让出 CPU 等待用户态守护进程响应，实现全异步/非阻塞等待。
    *   **基本操作支持**：已成功代理包括文件读写 (`read_at`, `write_at`)、目录读取 (`read_dir` - 解析 `FuseDirent`)、节点查找 (`lookup`)、元数据获取 (`metadata`)、文件创建 (`create`)及删除 (`unlink`) 等文件操作，并将响应的二进制流安全地反序列化回 VFS。

##### 4. 对外入口与注册：`src/lib.rs`

*   **功能定位**：提供供内核初始化的统一入口和全局状态管理。
*   **实现细节**：
    *   声明了全局单例的 `FUSE_CONNECTION` (借助 `spin::Once` 安全初始化)。
    *   **`init_fuse` 接口**：向操作系统 VFS 层注册 fuse（挂载点为字符设备节点 10:229）。同时向内核注册名为 `"fuse"` 的文件系统类型驱动，并在挂载时将其根节点代理给用户态侧指定的实现。
    *   **`exit_fuse` 接口**：支持安全注销释放设备资源。


#### fuse 按需加载模块

**位置**：`/workspaces/StarryOS/modules/fuse/*`

将 `Starryfuse` 的静态能力包裹为 `.ko`。
- `Cargo.toml`：依赖 `starryfuse`、`axfeat`、`kmod` 等。
- `src/lib.rs`：暴露出 `#[init_fn]` 的 `fuse_init` 和 `#[exit_fn]`的 `fuse_exit`。在模块加载和卸载时分别调用底层的相关处理。


#### 运行测试

```bash
starry:~# stat  stat /dev/fuse 
stat: can't stat 'stat': No such file or directory
[  5.166558 0:11 starry_api::kmod::ondemand:23] [ondemand] loading module 'fuse' from '/root/modules/fuse.ko'
[  5.493126 0:11 kmod_loader::loader:275] Module(Some("fuse")) info: ModuleInfo { name: fuse, version: 0.1.0, license: GPL, description: FUSE driver for StarryOS }
[  5.495657 0:11 kmod_loader::loader:528] Skipping section '.modinfo' in skip list
[  5.497620 0:11 starry_api::kmod:99] KmodHelper::vmalloc: Allocated paddr=PA:0x819c8000, vaddr=VA:0xffffffc0819c8000, size=32768
[  5.549736 0:11 starry_api::kmod:99] KmodHelper::vmalloc: Allocated paddr=PA:0x819d0000, vaddr=VA:0xffffffc0819d0000, size=16384
[  5.550700 0:11 starry_api::kmod:99] KmodHelper::vmalloc: Allocated paddr=PA:0x819d4000, vaddr=VA:0xffffffc0819d4000, size=4096
[  5.551710 0:11 starry_api::kmod:99] KmodHelper::vmalloc: Allocated paddr=PA:0x819d5000, vaddr=VA:0xffffffc0819d5000, size=4096
[  5.553231 0:11 kmod_loader::loader:576] Allocated section '                     .text' at 0xffffffc0819c8000 [RX] (0x8000)
[  5.555501 0:11 kmod_loader::loader:576] Allocated section '                   .rodata' at 0xffffffc0819d0000 [R] (0x4000)
[  5.556793 0:11 kmod_loader::loader:576] Allocated section ' .gnu.linkonce.this_module' at 0xffffffc0819d4000 [R] (0x1000)
[  5.558594 0:11 kmod_loader::loader:576] Allocated section '                      .bss' at 0xffffffc0819d5000 [RW] (0x1000)
[  5.576967 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::panic_cannot_unwind => Some(ffffffc0805577ae)
[  5.578366 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_cannot_unwind' (GLOBAL) to address 0xffffffc0805577ae
[  5.579406 0:11 starry_api::kmod:126] Resolving symbol: log::MAX_LOG_LEVEL_FILTER => Some(ffffffc0807a1378)
[  5.582327 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'log::MAX_LOG_LEVEL_FILTER' (GLOBAL) to address 0xffffffc0807a1378
[  5.584673 0:11 starry_api::kmod:126] Resolving symbol: log::__private_api::loc => Some(ffffffc08054f59c)
[  5.585454 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'log::__private_api::loc' (GLOBAL) to address 0xffffffc08054f59c
[  5.587306 0:11 starry_api::kmod:126] Resolving symbol: <log::__private_api::GlobalLogger as log::Log>::log => Some(ffffffc08054f530)
[  5.589498 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<log::__private_api::GlobalLogger as log::Log>::log' (GLOBAL) to address 0xffffffc08054f530
[  5.590929 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::panic_fmt => Some(ffffffc08055753a)
[  5.592895 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_fmt' (GLOBAL) to address 0xffffffc08055753a
[  5.594781 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::panic => Some(ffffffc0805575ac)
[  5.595493 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic' (GLOBAL) to address 0xffffffc0805575ac
[  5.597511 0:11 starry_api::kmod:126] Resolving symbol: __rust_no_alloc_shim_is_unstable => Some(ffffffc08077a001)
[  5.649372 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '__rust_no_alloc_shim_is_unstable' (GLOBAL) to address 0xffffffc08077a001
[  5.653050 0:11 starry_api::kmod:126] Resolving symbol: __rustc::__rust_alloc => Some(ffffffc08052523a)
[  5.654789 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_alloc' (GLOBAL) to address 0xffffffc08052523a
[  5.656769 0:11 starry_api::kmod:126] Resolving symbol: alloc::alloc::handle_alloc_error => Some(ffffffc08054a0ea)
[  5.658096 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::alloc::handle_alloc_error' (GLOBAL) to address 0xffffffc08054a0ea
[  5.659261 0:11 starry_api::kmod:126] Resolving symbol: __rustc::__rust_dealloc => Some(ffffffc08052527a)
[  5.659931 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_dealloc' (GLOBAL) to address 0xffffffc08052527a
[  5.660793 0:11 starry_api::kmod:126] Resolving symbol: memcpy => Some(ffffffc08056f82c)
[  5.661377 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'memcpy' (GLOBAL) to address 0xffffffc08056f82c
[  5.662160 0:11 starry_api::kmod:126] Resolving symbol: core::slice::index::slice_end_index_len_fail => Some(ffffffc08055dfee)
[  5.662959 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::slice::index::slice_end_index_len_fail' (GLOBAL) to address 0xffffffc08055dfee
[  5.664218 0:11 starry_api::kmod:126] Resolving symbol: core::option::unwrap_failed => Some(ffffffc0805573d2)
[  5.664930 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::option::unwrap_failed' (GLOBAL) to address 0xffffffc0805573d2
[  5.667913 0:11 starry_api::kmod:126] Resolving symbol: memmove => Some(ffffffc08056f844)
[  5.668904 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'memmove' (GLOBAL) to address 0xffffffc08056f844
[  5.670650 0:11 starry_api::kmod:126] Resolving symbol: alloc::raw_vec::RawVec<T,A>::grow_one => Some(ffffffc08029d446)
[  5.672180 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::raw_vec::RawVec<T,A>::grow_one' (GLOBAL) to address 0xffffffc08029d446
[  5.674885 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::panic_bounds_check => Some(ffffffc0805576ba)
[  5.675877 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_bounds_check' (GLOBAL) to address 0xffffffc0805576ba
[  5.678114 0:11 starry_api::kmod:126] Resolving symbol: _Unwind_Resume => Some(ffffffc080534520)
[  5.679460 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '_Unwind_Resume' (GLOBAL) to address 0xffffffc080534520
[  5.680274 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::panic_in_cleanup => Some(ffffffc0805577ca)
[  5.682293 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_in_cleanup' (GLOBAL) to address 0xffffffc0805577ca
[  5.683532 0:11 starry_api::kmod:126] Resolving symbol: event_listener::Task::wake => Some(ffffffc08048b5b4)
[  5.684230 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::Task::wake' (GLOBAL) to address 0xffffffc08048b5b4
[  5.686231 0:11 starry_api::kmod:126] Resolving symbol: core::option::expect_failed => Some(ffffffc0805573f0)
[  5.688729 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::option::expect_failed' (GLOBAL) to address 0xffffffc0805573f0
[  5.692425 0:11 starry_api::kmod:126] Resolving symbol: event_listener::TaskRef::into_task => Some(ffffffc08048b5ba)
[  5.694355 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::TaskRef::into_task' (GLOBAL) to address 0xffffffc08048b5ba
[  5.696552 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc08020a99c)
[  5.749382 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc08020a99c
[  5.750885 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc08020a99c)
[  5.752448 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc08020a99c
[  5.754545 0:11 starry_api::kmod:126] Resolving symbol: axfs_ng_vfs::node::dir::DirNode::new => Some(ffffffc08048923e)
[  5.757407 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng_vfs::node::dir::DirNode::new' (GLOBAL) to address 0xffffffc08048923e
[  5.758400 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::downgrade::panic_cold_display => Some(ffffffc08037be7e)
[  5.759906 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::downgrade::panic_cold_display' (GLOBAL) to address 0xffffffc08037be7e
[  5.763647 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc08020a99c)
[  5.764329 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc08020a99c
[  5.765518 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc08020a99c)
[  5.766726 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc08020a99c
[  5.767996 0:11 starry_api::kmod:126] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc08020a99c)
[  5.768782 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc08020a99c
[  5.772960 0:11 starry_api::kmod:126] Resolving symbol: axtask::api::current => Some(ffffffc080503a18)
[  5.774062 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::api::current' (GLOBAL) to address 0xffffffc080503a18
[  5.776698 0:11 starry_api::kmod:126] Resolving symbol: axtask::api::yield_now => Some(ffffffc080503d08)
[  5.777813 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::api::yield_now' (GLOBAL) to address 0xffffffc080503d08
[  5.779672 0:11 starry_api::kmod:126] Resolving symbol: axtask::task::TaskInner::id_name => Some(ffffffc0804fd55a)
[  5.781766 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::TaskInner::id_name' (GLOBAL) to address 0xffffffc0804fd55a
[  5.784539 0:11 starry_api::kmod:126] Resolving symbol: core::panicking::assert_failed => Some(ffffffc08023f3b6)
[  5.786329 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::assert_failed' (GLOBAL) to address 0xffffffc08023f3b6
[  5.789072 0:11 starry_api::kmod:126] Resolving symbol: alloc::raw_vec::handle_error => Some(ffffffc08054a0ce)
[  5.792436 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::raw_vec::handle_error' (GLOBAL) to address 0xffffffc08054a0ce
[  5.793935 0:11 starry_api::kmod:126] Resolving symbol: kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::acquire => Some(ffffffc08054eeb4)
[  5.795870 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::acquire' (GLOBAL) to address 0xffffffc08054eeb4
[  5.849430 0:11 starry_api::kmod:126] Resolving symbol: kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::release => Some(ffffffc08054ee98)
[  5.851069 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::release' (GLOBAL) to address 0xffffffc08054ee98
[  5.852616 0:11 starry_api::kmod:126] Resolving symbol: memset => Some(ffffffc08056f7e2)
[  5.853195 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'memset' (GLOBAL) to address 0xffffffc08056f7e2
[  5.854211 0:11 starry_api::kmod:126] Resolving symbol: core::str::converts::from_utf8 => Some(ffffffc08055e1aa)
[  5.854925 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::str::converts::from_utf8' (GLOBAL) to address 0xffffffc08055e1aa
[  5.855797 0:11 starry_api::kmod:126] Resolving symbol: core::slice::index::slice_start_index_len_fail => Some(ffffffc08055dfde)
[  5.857176 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::slice::index::slice_start_index_len_fail' (GLOBAL) to address 0xffffffc08055dfde
[  5.859103 0:11 starry_api::kmod:126] Resolving symbol: alloc::raw_vec::RawVec<T,A>::grow_one => Some(ffffffc08029d446)
[  5.862792 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::raw_vec::RawVec<T,A>::grow_one' (GLOBAL) to address 0xffffffc08029d446
[  5.864533 0:11 starry_api::kmod:126] Resolving symbol: axfs_ng_vfs::node::DirEntry::new_file => Some(ffffffc08047e6a4)
[  5.865917 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng_vfs::node::DirEntry::new_file' (GLOBAL) to address 0xffffffc08047e6a4
[  5.867890 0:11 starry_api::kmod:126] Resolving symbol: <i32 as event_listener::notify::IntoNotification>::into_notification => Some(ffffffc08048b660)
[  5.871923 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<i32 as event_listener::notify::IntoNotification>::into_notification' (GLOBAL) to address 0xffffffc08048b660
[  5.876763 0:11 starry_api::kmod:126] Resolving symbol: <event_listener::notify::Notify as event_listener::notify::NotificationPrivate>::fence => Some(ffffffc08048b5ae)
[  5.882119 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<event_listener::notify::Notify as event_listener::notify::NotificationPrivate>::fence' (GLOBAL) to address 0xffffffc08048b5ae
[  5.883490 0:11 starry_api::kmod:126] Resolving symbol: event_listener::RegisterResult<T>::notified::panic_cold_display => Some(ffffffc08048aff8)
[  5.890161 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::RegisterResult<T>::notified::panic_cold_display' (GLOBAL) to address 0xffffffc08048aff8
[  5.895004 0:11 starry_api::kmod:126] Resolving symbol: core::result::unwrap_failed => Some(ffffffc080557a22)
[  5.900196 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::result::unwrap_failed' (GLOBAL) to address 0xffffffc080557a22
[  5.903767 0:11 starry_api::kmod:126] Resolving symbol: event_listener::sys::node::TaskWaiting::cancel => Some(ffffffc08048ba50)
[  5.906574 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::cancel' (GLOBAL) to address 0xffffffc08048ba50
[  5.907517 0:11 starry_api::kmod:126] Resolving symbol: event_listener::sys::node::TaskWaiting::status => Some(ffffffc08048b99e)
[  5.962203 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::status' (GLOBAL) to address 0xffffffc08048b99e
[  5.964542 0:11 starry_api::kmod:126] Resolving symbol: event_listener::sys::node::TaskWaiting::register => Some(ffffffc08048b9a6)
[  5.966281 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::register' (GLOBAL) to address 0xffffffc08048b9a6
[  5.968091 0:11 starry_api::kmod:126] Resolving symbol: <str as core::fmt::Display>::fmt => Some(ffffffc08055d666)
[  5.970237 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<str as core::fmt::Display>::fmt' (GLOBAL) to address 0xffffffc08055d666
[  5.971245 0:11 starry_api::kmod:126] Resolving symbol: axtask::run_queue::__PERCPU_RUN_QUEUE => Some(38)
[  5.972292 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::run_queue::__PERCPU_RUN_QUEUE' (GLOBAL) to address 0x0000000000000038
[  5.974905 0:11 starry_api::kmod:126] Resolving symbol: axtask::task::CurrentTask::clone => Some(ffffffc0804fdbfe)
[  5.976696 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::CurrentTask::clone' (GLOBAL) to address 0xffffffc0804fdbfe
[  5.977897 0:11 starry_api::kmod:126] Resolving symbol: axtask::task::TaskInner::current_check_preempt_pending => Some(ffffffc0804fd93a)
[  5.978684 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::TaskInner::current_check_preempt_pending' (GLOBAL) to address 0xffffffc0804fd93a
[  5.982808 0:11 starry_api::kmod:126] Resolving symbol: axtask::run_queue::AxRunQueue::resched => Some(ffffffc080501e6e)
[  5.986348 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::run_queue::AxRunQueue::resched' (GLOBAL) to address 0xffffffc080501e6e
[  5.991416 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::num::imp::<impl core::fmt::Display for u64>::fmt => Some(ffffffc080567d20)
[  5.995674 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::imp::<impl core::fmt::Display for u64>::fmt' (GLOBAL) to address 0xffffffc080567d20
[  6.000339 0:11 starry_api::kmod:126] Resolving symbol: <bool as core::fmt::Display>::fmt => Some(ffffffc08055d304)
[  6.002212 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<bool as core::fmt::Display>::fmt' (GLOBAL) to address 0xffffffc08055d304
[  6.008775 0:11 starry_api::kmod:126] Resolving symbol: <str as core::fmt::Debug>::fmt => Some(ffffffc08055d344)
[  6.060308 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<str as core::fmt::Debug>::fmt' (GLOBAL) to address 0xffffffc08055d344
[  6.070518 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::Formatter::write_str => Some(ffffffc08055be32)
[  6.086898 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::write_str' (GLOBAL) to address 0xffffffc08055be32
[  6.088286 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::Formatter::debug_tuple_field1_finish => Some(ffffffc08055c4d0)
[  6.091625 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_tuple_field1_finish' (GLOBAL) to address 0xffffffc08055c4d0
[  6.096238 0:11 starry_api::kmod:126] Resolving symbol: starry_api::vfs::dev::register_devfs_device => Some(ffffffc0802887c8)
[  6.106054 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'starry_api::vfs::dev::register_devfs_device' (GLOBAL) to address 0xffffffc0802887c8
[  6.108378 0:11 starry_api::kmod:126] Resolving symbol: starry_api::vfs::register_filesystem => Some(ffffffc0802e465e)
[  6.109715 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'starry_api::vfs::register_filesystem' (GLOBAL) to address 0xffffffc0802e465e
[  6.111040 0:11 starry_api::kmod:126] Resolving symbol: starry_api::vfs::dev::unregister_devfs_device => Some(ffffffc080288b46)
[  6.111868 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'starry_api::vfs::dev::unregister_devfs_device' (GLOBAL) to address 0xffffffc080288b46
[  6.113275 0:11 starry_api::kmod:126] Resolving symbol: __rustc::__rust_realloc => Some(ffffffc080525292)
[  6.114414 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_realloc' (GLOBAL) to address 0xffffffc080525292
[  6.116458 0:11 starry_api::kmod:126] Resolving symbol: <axtask::future::AxWaker as alloc::task::Wake>::wake_by_ref => Some(ffffffc0805047bc)
[  6.117701 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<axtask::future::AxWaker as alloc::task::Wake>::wake_by_ref' (GLOBAL) to address 0xffffffc0805047bc
[  6.119313 0:11 starry_api::kmod:126] Resolving symbol: <axtask::future::AxWaker as alloc::task::Wake>::wake => Some(ffffffc080504760)
[  6.121062 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol '<axtask::future::AxWaker as alloc::task::Wake>::wake' (GLOBAL) to address 0xffffffc080504760
[  6.122465 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::num::imp::<impl core::fmt::Display for usize>::fmt => Some(ffffffc080567d20)
[  6.123736 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::imp::<impl core::fmt::Display for usize>::fmt' (GLOBAL) to address 0xffffffc080567d20
[  6.125378 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::num::<impl core::fmt::LowerHex for usize>::fmt => Some(ffffffc080566bb8)
[  6.126329 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::LowerHex for usize>::fmt' (GLOBAL) to address 0xffffffc080566bb8
[  6.177254 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::num::<impl core::fmt::UpperHex for usize>::fmt => Some(ffffffc080566c1e)
[  6.178293 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::UpperHex for usize>::fmt' (GLOBAL) to address 0xffffffc080566c1e
[  6.179383 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::Formatter::debug_tuple => Some(ffffffc08055c48c)
[  6.181264 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_tuple' (GLOBAL) to address 0xffffffc08055c48c
[  6.182260 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::builders::DebugTuple::field => Some(ffffffc08055800a)
[  6.185048 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugTuple::field' (GLOBAL) to address 0xffffffc08055800a
[  6.187991 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::builders::DebugTuple::finish => Some(ffffffc0805581be)
[  6.189417 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugTuple::finish' (GLOBAL) to address 0xffffffc0805581be
[  6.190630 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::Formatter::debug_struct => Some(ffffffc08055be60)
[  6.191823 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_struct' (GLOBAL) to address 0xffffffc08055be60
[  6.193430 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::builders::DebugStruct::field => Some(ffffffc080557d80)
[  6.194220 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugStruct::field' (GLOBAL) to address 0xffffffc080557d80
[  6.195498 0:11 starry_api::kmod:126] Resolving symbol: core::fmt::builders::DebugStruct::finish => Some(ffffffc080557fa8)
[  6.196667 0:11 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugStruct::finish' (GLOBAL) to address 0xffffffc080557fa8
[  6.198828 0:11 kmod_loader::loader:730] Applying relocations for section '.rela.text' to '.text', 1416 entries
[  6.202960 0:11 kmod_loader::loader:730] Applying relocations for section '.rela.rodata' to '.rodata', 415 entries
[  6.203794 0:11 kmod_loader::loader:730] Applying relocations for section '.rela.gnu.linkonce.this_module' to '.gnu.linkonce.this_module', 2 entries
[  6.211003 0:11 kmod_loader::loader:459] Module init_fn: Some(0xffffffc0819c8000), exit_fn: Some(0xffffffc0819c80fc)
[  6.212639 0:11 kmod_loader::loader:386] Section '__param' not found
[  6.215001 0:11 kmod_loader::param:165] [fuse]: parsing args '""'
[  6.216564 0:11 kmod_loader::loader:354] Module(Some("fuse")) loaded successfully!
[  6.221125 0:11 fuse:9] Fuse module loaded via on-demand mechanism.
[  6.221805 0:11 starry_api::kmod:164] Module(fuse) init returned: 0
[  6.222925 0:11 starry_api::kmod::ondemand:43] [ondemand] module 'fuse' loaded, handle=0x17c96ff18
  File: /dev/fuse
  Size: 0               Blocks: 0          IO Block: 0      character special file
Device: 2h/2d   Inode: 30          Links: 1     Device type: a,e5
Access: (0666/crw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 1970-01-01 00:00:00.000000000 +0000
Modify: 1970-01-01 00:00:00.000000000 +0000
Change: 1970-01-01 00:00:00.000000000 +0000
starry:~# ls
kallsyms  ll        modules
starry:~# [ 11.576201 0:6 starry_api::kmod::ondemand:55] [ondemand] unload handle=0x17c96ff18
[ 11.578583 0:6 kmod_loader::loader:122] Calling module exit function...
[ 11.582613 0:6 fuse:19] Fuse module exit called.
[ 11.583442 0:6 starry_api::kmod:179] Module(fuse) exited
[ 11.585196 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819c8000, num_pages=8
[ 11.588582 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819d0000, num_pages=4
[ 11.589830 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819d4000, num_pages=1
[ 11.592366 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819d5000, num_pages=1
ls
kallsyms  ll        modules
```

---

### 问题报告

1. `ext4_bcache_free` 内存重读崩溃（已修复）
早期版本的 `FuseUsageChecker` 在检查一个模块是否被占用时，对遍历到的每一个文件描述符使用了直接转路径的判断 `file.path().contains("fuse")`。

**根因**：调用 `.path()` 方法在面对底层驱动为 `ext4` 的文件时，会触发 Ext4 文件系统反向向磁盘块发起寻找与读流缓存的操作。由于 UsageChecker 是在背景时钟（Timer Callback）以及无预警作用域中频繁执行的，这打乱了正在运作的 Ext4 IO Cache Block 操作（如 `ext4_bcache_drop_buf`）锁机制，引起 Read Page Fault 宕机。

**修复办法**：代码被精修改为“两级拦截匹配”机制。即在调用 `.path()` 方法提取路径名之前，通过 `loc.filesystem().name()` 方法进行文件系统归属分类。只有在目标对象为不涉及磁盘 IO 的 `devfs`（纯内存树）时，才调用其 path 方法，其余一律只进行标识比对，彻底解决了系统颠簸。

2. `fuse_test` 中途引发 `procfs` 重入崩溃锁死（待解决）

单独测试 `procfs` 和 `fuse` 模块按需加载均可正常工作。

当标准库执行 `sys_mount` 或者 `create_dir_all` 探测运行时环境时（用户执行 `/musl/fuse_test` 的幕后行为），标准 libc 自动尝试访问 `/proc` 相关挂载。
由于 `procfs` 被设计为在按需加载后会强制跨边界读取主内核 `FS_CONTEXT.lock()` 宏。在主 syscall 回路已经锁定了一层资源的状态下，该跨模块的宏指针解析重定位错误地访问到了未知地址 `0x1a0bdce93a9ee37f`，引起崩溃。

这里有一个**锁重入导致的数据异常竞争（Deadlock/Memory Corrupt）**，代码调用链如下：

1. `fuse_test` 发起了一次 open 系统调用，进入内核的 `sys_openat`（定义在 `fd_ops.rs`）。

2. 在 `sys_openat` 里，内核执行了 `with_ondemand` 函数：

3. 这个 `with_fs` 函数内部立刻获取了 `FS_CONTEXT.lock()` 并在这个闭包内部一直持有！

4. `open` 发现 `proc` 找不到，于是向外返回一个 `NotFound` 错误。此时闭包执行完毕，刚才那把 `FS_CONTEXT.lock()` 已经被释放。

5. 紧接着，外层的 `with_ondemand` 捕获到了 `NotFound`，进入 `try_ondemand_load_path` 去硬盘里加载 `procfs.ko`（并执行它的入口函数 `procfs_init`）。

6. 灾难发生在此刻：
`procfs_init` 第一句话是去调用 `let fs = FS_CONTEXT.lock();`
但在 RISC-V QEMU 以及这种从 `.ko` 动态模块去强行获取基于 `scope_local` 宏定义的主内核变量时，模块边界的上下文被破坏了或者重定位出错了，使得它读到了内存地址 `0x1a0bdce93a9ee37f` 导致 Supervisor Page Fault 。


---

### 下周工作安排

1. 完成Fuse用户态库接入测试。

2. 完善模块使用期间监测逻辑。

---

### 老师建议
1. 实习报告： 文档、视频、ppt
2. 结合哈工大同学工作，看看是否可以有复用代码
3. 结合 vDSO工作，做可加载模块内核态与用户态的切换