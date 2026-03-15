---
id: week5
title: Week 5
prev: week4
next: week6
---

## 2026.3.9-2026.3.15

**工作内容**

修理一些按需加载模块的bug以及架构上不合理的地方，秉持尽可能减少对内核代码的改动原则。将procfs解除静态挂载并能独立出成为.ko文件，完成该模块按需加载测试（调用时加载，5s后自动卸载）



1. 新增独立的 `procfs.ko` 模块 crate，用于承载 procfs 的挂载/卸载逻辑：

- `procfs_init()`：直接使用 `axfs_ng::FS_CONTEXT` 创建 `/proc` 挂载点并挂载 `starry_api::vfs::new_procfs()`
- `procfs_exit()`：直接解析 `/proc` 并执行 `unmount()`
- 模块侧自行依赖 `axfs-ng` 与 `axfs-ng-vfs`，避免在原有 `api/src/vfs/mod.rs` 中增加专用挂载封装
- 通过 `module!` 宏导出模块元数据（`name = "procfs"`）

2. 新增  `api/src/kmod/ondemand_builtin.rs` 将按需加载框架桥接与模块策略注册解耦：

- `ondemand.rs`：仅负责 `ModuleLoader` 适配、全局 `REGISTRY`、`with_ondemand()` 与 `tick_ondemand()`，保持泛型
- `ondemand_builtin.rs`：承载 StarryOS 内建策略（如 procfs 的路径触发注册）

在 `ondemand_builtin::register_builtin_modules()` 中注册 procfs：

```rust
let _ = super::ondemand::register_module(ModuleDesc {
    name: "procfs",
    ko_path: "/root/modules/procfs.ko",
    idle_timeout_ticks: 5_000,
    trigger: Box::new(PathPrefixTrigger::new("/proc")),
    usage: None,
});
```
3. `ondemand-kmod/src/monitor.rs` 修复自动卸载状态机的一致性问题

- 旧行为：`loader.unload(handle)` 失败时，仍会把模块状态置为 `Unloaded`
- 新行为：仅当 unload 成功才置 `Unloaded`
- unload 失败时：恢复 `State::Active`、恢复 `loaded_handle` 并 `touch(now)`，后续 tick 可安全重试

4.  `api/src/kmod/ondemand.rs` 为 `procfs` 增加两阶段卸载保护（loader 层）：

- 第一次触发 unload：先尝试 `unmount /proc`，并返回 `UnloadError::InUse`（延后真正模块释放）
- 后续 tick 再次调用 unload：若 `/proc` 已不是挂载根，再执行 `delete_module("procfs")`

第一次卸载尝试：先 unmount /proc，然后返回 UnloadError::InUse（故意延后一拍，不立刻 free 模块内存）

下一次 tick：检测到 proc 已不是挂载根后，再执行 delete_module


5. `modules/procfs/src/lib.rs`

`/proc` 卸载已在上层两阶段卸载路径中完成，模块 `exit_fn` 只负责退出收尾，避免重复卸载副作用。

已在 lib.rs 将 procfs_exit 改为只记录退出日志，避免重复卸载同一挂载点。

6. 测试：

```bash
# hello.ko 可加载模块测试
# 命令行输出



starry:~# which insmod
/sbin/insmod
starry:~# which rmmod
/sbin/rmmod
starry:~# insmod /root/modules/hello.ko
[251.636017 0:12 starry_api::syscall::kmod:52] [sys_finit_module]: module_len=27512, params=Some("")
[251.642475 0:12 kmod_loader::loader:275] Module(Some("hello")) info: ModuleInfo { name: hello, version: 0.1.0, license: GPL, description: A simple hello world kernel module }
[251.644720 0:12 kmod_loader::loader:528] Skipping section '.modinfo' in skip list
[251.648479 0:12 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819b1000, vaddr=VA:0xffffffc0819b1000, size=4096
[251.651821 0:12 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819b2000, vaddr=VA:0xffffffc0819b2000, size=4096
[251.654396 0:12 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819b3000, vaddr=VA:0xffffffc0819b3000, size=4096
[251.655569 0:12 kmod_loader::loader:576] Allocated section '                     .text' at 0xffffffc0819b1000 [RX] (0x1000)
[251.657817 0:12 kmod_loader::loader:576] Allocated section '                   .rodata' at 0xffffffc0819b2000 [R] (0x1000)
[251.659330 0:12 kmod_loader::loader:576] Allocated section ' .gnu.linkonce.this_module' at 0xffffffc0819b3000 [R] (0x1000)
[251.667945 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::num::imp::<impl core::fmt::Display for i32>::fmt => Some(ffffffc08055cb9e)
[251.669648 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::imp::<impl core::fmt::Display for i32>::fmt' (GLOBAL) to address 0xffffffc08055cb9e
[251.673975 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::num::<impl core::fmt::LowerHex for i32>::fmt => Some(ffffffc08055ba44)
[251.675213 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::LowerHex for i32>::fmt' (GLOBAL) to address 0xffffffc08055ba44
[251.676633 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::num::<impl core::fmt::UpperHex for i32>::fmt => Some(ffffffc08055baae)
[251.677448 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::UpperHex for i32>::fmt' (GLOBAL) to address 0xffffffc08055baae
[251.682926 0:12 starry_api::kmod:113] Resolving symbol: write_char => Some(ffffffc080561794)
[251.684058 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'write_char' (GLOBAL) to address 0xffffffc080561794
[251.689699 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::write => Some(ffffffc0805504de)
[251.690320 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::write' (GLOBAL) to address 0xffffffc0805504de
[251.692134 0:12 starry_api::kmod:113] Resolving symbol: __rustc::__rust_dealloc => Some(ffffffc08051a27a)
[251.693151 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_dealloc' (GLOBAL) to address 0xffffffc08051a27a
[251.694550 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::write_str => Some(ffffffc080550e32)
[251.696320 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::write_str' (GLOBAL) to address 0xffffffc080550e32
[251.697922 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::debug_list => Some(ffffffc080552206)
[251.698686 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_list' (GLOBAL) to address 0xffffffc080552206
[251.700136 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugList::entry => Some(ffffffc08054d240)
[251.701667 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugList::entry' (GLOBAL) to address 0xffffffc08054d240
[251.703312 0:12 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugList::finish => Some(ffffffc08054d500)
[251.704121 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugList::finish' (GLOBAL) to address 0xffffffc08054d500
[251.705665 0:12 starry_api::kmod:113] Resolving symbol: core::panicking::panic_cannot_unwind => Some(ffffffc08054c7ae)
[251.707218 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_cannot_unwind' (GLOBAL) to address 0xffffffc08054c7ae
[251.708668 0:12 starry_api::kmod:113] Resolving symbol: __rust_no_alloc_shim_is_unstable => Some(ffffffc08076c001)
[251.709441 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol '__rust_no_alloc_shim_is_unstable' (GLOBAL) to address 0xffffffc08076c001
[251.714203 0:12 starry_api::kmod:113] Resolving symbol: __rustc::__rust_alloc => Some(ffffffc08051a23a)
[251.715425 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_alloc' (GLOBAL) to address 0xffffffc08051a23a
[251.718636 0:12 starry_api::kmod:113] Resolving symbol: core::result::unwrap_failed => Some(ffffffc08054ca22)
[251.720422 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::result::unwrap_failed' (GLOBAL) to address 0xffffffc08054ca22
[251.723099 0:12 starry_api::kmod:113] Resolving symbol: alloc::alloc::handle_alloc_error => Some(ffffffc08053f0ea)
[251.725478 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::alloc::handle_alloc_error' (GLOBAL) to address 0xffffffc08053f0ea
[251.728225 0:12 starry_api::kmod:113] Resolving symbol: _Unwind_Resume => Some(ffffffc080529520)
[251.728842 0:12 kmod_loader::loader:626]   -> Resolved undefined symbol '_Unwind_Resume' (GLOBAL) to address 0xffffffc080529520
[251.730122 0:12 kmod_loader::loader:730] Applying relocations for section '.rela.text' to '.text', 65 entries
[251.731951 0:12 kmod_loader::loader:730] Applying relocations for section '.rela.rodata' to '.rodata', 12 entries
[251.733564 0:12 kmod_loader::loader:730] Applying relocations for section '.rela.gnu.linkonce.this_module' to '.gnu.linkonce.this_module', 2 entries
[251.735204 0:12 kmod_loader::loader:459] Module init_fn: Some(0xffffffc0819b11d8), exit_fn: Some(0xffffffc0819b134a)
[251.736837 0:12 kmod_loader::loader:386] Section '__param' not found
[251.739371 0:12 kmod_loader::param:165] [hello]: parsing args '""'
[251.740466 0:12 kmod_loader::loader:354] Module(Some("hello")) loaded successfully!
Hello, Kernel Module!
Vector contents: [1, 2, 3, 4, 5]
[251.744709 0:12 starry_api::kmod:151] Module(hello) init returned: 0
starry:~# rmmod hello
[591.718243 0:15 starry_api::syscall::kmod:65] [sys_delete_module]: name=hello
[591.719536 0:15 kmod_loader::loader:122] Calling module exit function...
Goodbye, Kernel Module!
[591.720990 0:15 starry_api::kmod:166] Module(hello) exited
[591.721914 0:15 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b1000, num_pages=1
[591.723063 0:15 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b2000, num_pages=1
[591.724130 0:15 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b3000, num_pages=1


```


```bash
# proc.ko 可加载模块测试
# 命令行输出

starry:~# cat /proc/meminfo
[ 15.924639 0:10 starry_api::kmod::ondemand:23] [ondemand] loading module 'procfs' from '/root/modules/procfs.ko'
[ 16.008734 0:10 kmod_loader::loader:275] Module(Some("procfs")) info: ModuleInfo { name: procfs, version: 0.1.0, license: GPL, description: Procfs loadable kernel module for on-demand mounting }
[ 16.011802 0:10 kmod_loader::loader:528] Skipping section '.modinfo' in skip list
[ 16.013953 0:10 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819ae000, vaddr=VA:0xffffffc0819ae000, size=16384
[ 16.016892 0:10 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819b2000, vaddr=VA:0xffffffc0819b2000, size=12288
[ 16.018067 0:10 starry_api::kmod:92] KmodHelper::vmalloc: Allocated paddr=PA:0x819b5000, vaddr=VA:0xffffffc0819b5000, size=4096
[ 16.019078 0:10 kmod_loader::loader:576] Allocated section '                     .text' at 0xffffffc0819ae000 [RX] (0x4000)
[ 16.021288 0:10 kmod_loader::loader:576] Allocated section '                   .rodata' at 0xffffffc0819b2000 [R] (0x3000)
[ 16.022550 0:10 kmod_loader::loader:576] Allocated section ' .gnu.linkonce.this_module' at 0xffffffc0819b5000 [R] (0x1000)
[ 16.032148 0:10 starry_api::kmod:113] Resolving symbol: core::option::expect_failed => Some(ffffffc08054e3f0)
[ 16.033731 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::option::expect_failed' (GLOBAL) to address 0xffffffc08054e3f0
[ 16.035482 0:10 starry_api::kmod:113] Resolving symbol: __rust_no_alloc_shim_is_unstable => Some(ffffffc08076e001)
[ 16.036806 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '__rust_no_alloc_shim_is_unstable' (GLOBAL) to address 0xffffffc08076e001
[ 16.039334 0:10 starry_api::kmod:113] Resolving symbol: __rustc::__rust_alloc => Some(ffffffc08051c23a)
[ 16.040077 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_alloc' (GLOBAL) to address 0xffffffc08051c23a
[ 16.043095 0:10 starry_api::kmod:113] Resolving symbol: memcpy => Some(ffffffc08056682c)
[ 16.045453 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'memcpy' (GLOBAL) to address 0xffffffc08056682c
[ 16.046880 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.048107 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.050456 0:10 starry_api::kmod:113] Resolving symbol: alloc::alloc::handle_alloc_error => Some(ffffffc0805410ea)
[ 16.052143 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::alloc::handle_alloc_error' (GLOBAL) to address 0xffffffc0805410ea
[ 16.054262 0:10 starry_api::kmod:113] Resolving symbol: <i32 as event_listener::notify::IntoNotification>::into_notification => Some(ffffffc080482660)
[ 16.055778 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<i32 as event_listener::notify::IntoNotification>::into_notification' (GLOBAL) to address 0xffffffc080482660
[ 16.057021 0:10 starry_api::kmod:113] Resolving symbol: <event_listener::notify::Notify as event_listener::notify::NotificationPrivate>::fence => Some(ffffffc0804825ae)
[ 16.058340 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<event_listener::notify::Notify as event_listener::notify::NotificationPrivate>::fence' (GLOBAL) to address 0xffffffc0804825ae
[ 16.060724 0:10 starry_api::kmod:113] Resolving symbol: event_listener::RegisterResult<T>::notified::panic_cold_display => Some(ffffffc080481ff8)
[ 16.061656 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::RegisterResult<T>::notified::panic_cold_display' (GLOBAL) to address 0xffffffc080481ff8
[ 16.063526 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.064605 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.065965 0:10 starry_api::kmod:113] Resolving symbol: core::result::unwrap_failed => Some(ffffffc08054ea22)
[ 16.067215 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::result::unwrap_failed' (GLOBAL) to address 0xffffffc08054ea22
[ 16.068415 0:10 starry_api::kmod:113] Resolving symbol: _Unwind_Resume => Some(ffffffc08052b520)
[ 16.069503 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '_Unwind_Resume' (GLOBAL) to address 0xffffffc08052b520
[ 16.070403 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::panic_in_cleanup => Some(ffffffc08054e7ca)
[ 16.071266 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_in_cleanup' (GLOBAL) to address 0xffffffc08054e7ca
[ 16.073095 0:10 starry_api::kmod:113] Resolving symbol: event_listener::sys::node::TaskWaiting::cancel => Some(ffffffc080482a50)
[ 16.074128 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::cancel' (GLOBAL) to address 0xffffffc080482a50
[ 16.075180 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::panic => Some(ffffffc08054e5ac)
[ 16.076284 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic' (GLOBAL) to address 0xffffffc08054e5ac
[ 16.077845 0:10 starry_api::kmod:113] Resolving symbol: event_listener::sys::node::TaskWaiting::status => Some(ffffffc08048299e)
[ 16.079287 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::status' (GLOBAL) to address 0xffffffc08048299e
[ 16.080941 0:10 starry_api::kmod:113] Resolving symbol: event_listener::TaskRef::into_task => Some(ffffffc0804825ba)
[ 16.081712 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::TaskRef::into_task' (GLOBAL) to address 0xffffffc0804825ba
[ 16.082938 0:10 starry_api::kmod:113] Resolving symbol: event_listener::sys::node::TaskWaiting::register => Some(ffffffc0804829a6)
[ 16.084171 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::sys::node::TaskWaiting::register' (GLOBAL) to address 0xffffffc0804829a6
[ 16.085473 0:10 starry_api::kmod:113] Resolving symbol: __rustc::__rust_dealloc => Some(ffffffc08051c27a)
[ 16.086725 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_dealloc' (GLOBAL) to address 0xffffffc08051c27a
[ 16.087943 0:10 starry_api::kmod:113] Resolving symbol: memset => Some(ffffffc0805667e2)
[ 16.088756 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'memset' (GLOBAL) to address 0xffffffc0805667e2
[ 16.089783 0:10 starry_api::kmod:113] Resolving symbol: core::option::unwrap_failed => Some(ffffffc08054e3d2)
[ 16.090828 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::option::unwrap_failed' (GLOBAL) to address 0xffffffc08054e3d2
[ 16.091907 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::panic_fmt => Some(ffffffc08054e53a)
[ 16.092603 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_fmt' (GLOBAL) to address 0xffffffc08054e53a
[ 16.094400 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.095530 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.096813 0:10 starry_api::kmod:113] Resolving symbol: <axtask::future::AxWaker as alloc::task::Wake>::wake_by_ref => Some(ffffffc0804fb7bc)
[ 16.097993 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<axtask::future::AxWaker as alloc::task::Wake>::wake_by_ref' (GLOBAL) to address 0xffffffc0804fb7bc
[ 16.099194 0:10 starry_api::kmod:113] Resolving symbol: <axtask::future::AxWaker as alloc::task::Wake>::wake => Some(ffffffc0804fb760)
[ 16.100542 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<axtask::future::AxWaker as alloc::task::Wake>::wake' (GLOBAL) to address 0xffffffc0804fb760
[ 16.101604 0:10 starry_api::kmod:113] Resolving symbol: axtask::api::current => Some(ffffffc0804faa18)
[ 16.102323 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::api::current' (GLOBAL) to address 0xffffffc0804faa18
[ 16.104587 0:10 starry_api::kmod:113] Resolving symbol: axtask::task::CurrentTask::clone => Some(ffffffc0804f4bfe)
[ 16.105542 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::CurrentTask::clone' (GLOBAL) to address 0xffffffc0804f4bfe
[ 16.110503 0:10 starry_api::kmod:113] Resolving symbol: axtask::api::yield_now => Some(ffffffc0804fad08)
[ 16.111154 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::api::yield_now' (GLOBAL) to address 0xffffffc0804fad08
[ 16.112533 0:10 starry_api::kmod:113] Resolving symbol: kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::acquire => Some(ffffffc080545eb4)
[ 16.121877 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::acquire' (GLOBAL) to address 0xffffffc080545eb4
[ 16.126215 0:10 starry_api::kmod:113] Resolving symbol: axtask::run_queue::__PERCPU_RUN_QUEUE => Some(38)
[ 16.127655 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::run_queue::__PERCPU_RUN_QUEUE' (GLOBAL) to address 0x0000000000000038
[ 16.130805 0:10 starry_api::kmod:113] Resolving symbol: kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::release => Some(ffffffc080545e98)
[ 16.135173 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'kernel_guard::imp::<impl kernel_guard::BaseGuard for kernel_guard::NoPreemptIrqSave>::release' (GLOBAL) to address 0xffffffc080545e98
[ 16.139390 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.140251 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.141812 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::downgrade::panic_cold_display => Some(ffffffc080372e7e)
[ 16.143080 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::downgrade::panic_cold_display' (GLOBAL) to address 0xffffffc080372e7e
[ 16.144683 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::debug_tuple => Some(ffffffc08055348c)
[ 16.145650 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_tuple' (GLOBAL) to address 0xffffffc08055348c
[ 16.146908 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugTuple::field => Some(ffffffc08054f00a)
[ 16.148306 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugTuple::field' (GLOBAL) to address 0xffffffc08054f00a
[ 16.149673 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugTuple::finish => Some(ffffffc08054f1be)
[ 16.150606 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugTuple::finish' (GLOBAL) to address 0xffffffc08054f1be
[ 16.151877 0:10 starry_api::kmod:113] Resolving symbol: alloc::raw_vec::RawVec<T,A>::grow_one => Some(ffffffc0802b724e)
[ 16.152746 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::raw_vec::RawVec<T,A>::grow_one' (GLOBAL) to address 0xffffffc0802b724e
[ 16.154272 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::panic_bounds_check => Some(ffffffc08054e6ba)
[ 16.155109 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_bounds_check' (GLOBAL) to address 0xffffffc08054e6ba
[ 16.156155 0:10 starry_api::kmod:113] Resolving symbol: event_listener::Task::wake => Some(ffffffc0804825b4)
[ 16.156976 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'event_listener::Task::wake' (GLOBAL) to address 0xffffffc0804825b4
[ 16.160553 0:10 starry_api::kmod:113] Resolving symbol: __rustc::__rust_realloc => Some(ffffffc08051c292)
[ 16.161588 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '__rustc::__rust_realloc' (GLOBAL) to address 0xffffffc08051c292
[ 16.162783 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::num::imp::<impl core::fmt::Display for usize>::fmt => Some(ffffffc08055ed20)
[ 16.166309 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::imp::<impl core::fmt::Display for usize>::fmt' (GLOBAL) to address 0xffffffc08055ed20
[ 16.167634 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::num::<impl core::fmt::LowerHex for usize>::fmt => Some(ffffffc08055dbb8)
[ 16.173268 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::LowerHex for usize>::fmt' (GLOBAL) to address 0xffffffc08055dbb8
[ 16.175452 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::num::<impl core::fmt::UpperHex for usize>::fmt => Some(ffffffc08055dc1e)
[ 16.177304 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::num::<impl core::fmt::UpperHex for usize>::fmt' (GLOBAL) to address 0xffffffc08055dc1e
[ 16.178934 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.180252 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.181299 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.182099 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.183910 0:10 starry_api::kmod:113] Resolving symbol: alloc::raw_vec::handle_error => Some(ffffffc0805410ce)
[ 16.185580 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::raw_vec::handle_error' (GLOBAL) to address 0xffffffc0805410ce
[ 16.187001 0:10 starry_api::kmod:113] Resolving symbol: <str as core::fmt::Display>::fmt => Some(ffffffc080554666)
[ 16.188077 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<str as core::fmt::Display>::fmt' (GLOBAL) to address 0xffffffc080554666
[ 16.189454 0:10 starry_api::kmod:113] Resolving symbol: axtask::task::TaskInner::id_name => Some(ffffffc0804f455a)
[ 16.190350 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::TaskInner::id_name' (GLOBAL) to address 0xffffffc0804f455a
[ 16.191848 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::assert_failed => Some(ffffffc0802afc02)
[ 16.192586 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::assert_failed' (GLOBAL) to address 0xffffffc0802afc02
[ 16.193594 0:10 starry_api::kmod:113] Resolving symbol: core::panicking::panic_cannot_unwind => Some(ffffffc08054e7ae)
[ 16.196362 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::panicking::panic_cannot_unwind' (GLOBAL) to address 0xffffffc08054e7ae
[ 16.197633 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng::highlevel::fs::FS_CONTEXT => Some(ffffffc0805d5f80)
[ 16.199319 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng::highlevel::fs::FS_CONTEXT' (GLOBAL) to address 0xffffffc0805d5f80
[ 16.200876 0:10 starry_api::kmod:113] Resolving symbol: scope_local::scope::ActiveScope::get => Some(ffffffc08045ee24)
[ 16.203430 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'scope_local::scope::ActiveScope::get' (GLOBAL) to address 0xffffffc08045ee24
[ 16.206115 0:10 starry_api::kmod:113] Resolving symbol: scope_local::boxed::layout => Some(ffffffc08045eae0)
[ 16.209081 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'scope_local::boxed::layout' (GLOBAL) to address 0xffffffc08045eae0
[ 16.210675 0:10 starry_api::kmod:113] Resolving symbol: log::MAX_LOG_LEVEL_FILTER => Some(ffffffc080795328)
[ 16.211782 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'log::MAX_LOG_LEVEL_FILTER' (GLOBAL) to address 0xffffffc080795328
[ 16.215965 0:10 starry_api::kmod:113] Resolving symbol: log::__private_api::loc => Some(ffffffc08054659c)
[ 16.217276 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'log::__private_api::loc' (GLOBAL) to address 0xffffffc08054659c
[ 16.221334 0:10 starry_api::kmod:113] Resolving symbol: <log::__private_api::GlobalLogger as log::Log>::log => Some(ffffffc080546530)
[ 16.223394 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<log::__private_api::GlobalLogger as log::Log>::log' (GLOBAL) to address 0xffffffc080546530
[ 16.227046 0:10 starry_api::kmod:113] Resolving symbol: starry_api::vfs::proc::new_procfs => Some(ffffffc080248a48)
[ 16.231608 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'starry_api::vfs::proc::new_procfs' (GLOBAL) to address 0xffffffc080248a48
[ 16.235376 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng_vfs::mount::Location::mount => Some(ffffffc08047eff0)
[ 16.239539 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng_vfs::mount::Location::mount' (GLOBAL) to address 0xffffffc08047eff0
[ 16.242624 0:10 starry_api::kmod:113] Resolving symbol: alloc::sync::Arc<T,A>::drop_slow => Some(ffffffc0802b0356)
[ 16.246601 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'alloc::sync::Arc<T,A>::drop_slow' (GLOBAL) to address 0xffffffc0802b0356
[ 16.249070 0:10 starry_api::kmod:113] Resolving symbol: <bool as core::fmt::Display>::fmt => Some(ffffffc080554304)
[ 16.251027 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<bool as core::fmt::Display>::fmt' (GLOBAL) to address 0xffffffc080554304
[ 16.255407 0:10 starry_api::kmod:113] Resolving symbol: <str as core::fmt::Debug>::fmt => Some(ffffffc080554344)
[ 16.256190 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol '<str as core::fmt::Debug>::fmt' (GLOBAL) to address 0xffffffc080554344
[ 16.257613 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::write_str => Some(ffffffc080552e32)
[ 16.261182 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::write_str' (GLOBAL) to address 0xffffffc080552e32
[ 16.263168 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::debug_tuple_field1_finish => Some(ffffffc0805534d0)
[ 16.268742 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_tuple_field1_finish' (GLOBAL) to address 0xffffffc0805534d0
[ 16.273893 0:10 starry_api::kmod:113] Resolving symbol: axtask::task::TaskInner::current_check_preempt_pending => Some(ffffffc0804f493a)
[ 16.276105 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::task::TaskInner::current_check_preempt_pending' (GLOBAL) to address 0xffffffc0804f493a
[ 16.277791 0:10 starry_api::kmod:113] Resolving symbol: axtask::run_queue::AxRunQueue::resched => Some(ffffffc0804f8e6e)
[ 16.279065 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axtask::run_queue::AxRunQueue::resched' (GLOBAL) to address 0xffffffc0804f8e6e
[ 16.280800 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::Formatter::debug_struct => Some(ffffffc080552e60)
[ 16.282976 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::Formatter::debug_struct' (GLOBAL) to address 0xffffffc080552e60
[ 16.285785 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugStruct::field => Some(ffffffc08054ed80)
[ 16.288874 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugStruct::field' (GLOBAL) to address 0xffffffc08054ed80
[ 16.289912 0:10 starry_api::kmod:113] Resolving symbol: core::fmt::builders::DebugStruct::finish => Some(ffffffc08054efa8)
[ 16.291117 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'core::fmt::builders::DebugStruct::finish' (GLOBAL) to address 0xffffffc08054efa8
[ 16.295022 0:10 starry_api::kmod:113] Resolving symbol: memmove => Some(ffffffc080566844)
[ 16.297779 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'memmove' (GLOBAL) to address 0xffffffc080566844
[ 16.299266 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng::highlevel::fs::FsContext::resolve_nonexistent => Some(ffffffc080459fc2)
[ 16.300197 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng::highlevel::fs::FsContext::resolve_nonexistent' (GLOBAL) to address 0xffffffc080459fc2
[ 16.302809 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng_vfs::mount::Location::create => Some(ffffffc08047ed14)
[ 16.306226 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng_vfs::mount::Location::create' (GLOBAL) to address 0xffffffc08047ed14
[ 16.308719 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng::highlevel::fs::FsContext::resolve_inner => Some(ffffffc080459d16)
[ 16.311706 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng::highlevel::fs::FsContext::resolve_inner' (GLOBAL) to address 0xffffffc080459d16
[ 16.313461 0:10 starry_api::kmod:113] Resolving symbol: axfs_ng::highlevel::fs::FsContext::lookup => Some(ffffffc0804598d6)
[ 16.314296 0:10 kmod_loader::loader:626]   -> Resolved undefined symbol 'axfs_ng::highlevel::fs::FsContext::lookup' (GLOBAL) to address 0xffffffc0804598d6
[ 16.316816 0:10 kmod_loader::loader:730] Applying relocations for section '.rela.text' to '.text', 757 entries
[ 16.319669 0:10 kmod_loader::loader:730] Applying relocations for section '.rela.rodata' to '.rodata', 236 entries
[ 16.322586 0:10 kmod_loader::loader:730] Applying relocations for section '.rela.gnu.linkonce.this_module' to '.gnu.linkonce.this_module', 2 entries
[ 16.327557 0:10 kmod_loader::loader:459] Module init_fn: Some(0xffffffc0819b06b6), exit_fn: Some(0xffffffc0819b0f3a)
[ 16.329107 0:10 kmod_loader::loader:386] Section '__param' not found
[ 16.331792 0:10 kmod_loader::param:165] [procfs]: parsing args '""'
[ 16.333370 0:10 kmod_loader::loader:354] Module(Some("procfs")) loaded successfully!
[ 16.340163 0:10 procfs:21] procfs module loaded and mounted at /proc
[ 16.342321 0:10 starry_api::kmod:151] Module(procfs) init returned: 0
[ 16.344214 0:10 starry_api::kmod::ondemand:43] [ondemand] module 'procfs' loaded, handle=0x653156e4572
MemTotal:       32536204 kB
MemFree:         5506524 kB
MemAvailable:   18768344 kB
Buffers:            3264 kB
Cached:         14454588 kB
SwapCached:            0 kB
Active:         18229700 kB
Inactive:        6540624 kB
Active(anon):   11380224 kB
Inactive(anon):        0 kB
Active(file):    6849476 kB
Inactive(file):  6540624 kB
Unevictable:      930088 kB
Mlocked:            1136 kB
SwapTotal:       4194300 kB
SwapFree:        4194300 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:             47952 kB
Writeback:             0 kB
AnonPages:      10992512 kB
Mapped:          1361184 kB
Shmem:           1068056 kB
KReclaimable:     341440 kB
Slab:             628996 kB
SReclaimable:     341440 kB
SUnreclaim:       287556 kB
KernelStack:       28704 kB
PageTables:        85308 kB
SecPageTables:      2084 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    20462400 kB
Committed_AS:   45105316 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      205924 kB
VmallocChunk:          0 kB
Percpu:            23840 kB
HardwareCorrupted:     0 kB
AnonHugePages:   1417216 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:    477184 kB
FilePmdMapped:    288768 kB
CmaTotal:              0 kB
CmaFree:               0 kB
Unaccepted:            0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:     1739900 kB
DirectMap2M:    31492096 kB
DirectMap1G:     1048576 kB
starry:~# [ 21.354313 0:1 starry_api::kmod::ondemand:55] [ondemand] unload handle=0x653156e4572.  
[ 21.355679 0:1 starry_api::kmod::ondemand:73] [ondemand] procfs unmounted, defer module free to next tick
[ 26.364028 0:1 starry_api::kmod::ondemand:55] [ondemand] unload handle=0x653156e4572
[ 26.365801 0:1 kmod_loader::loader:122] Calling module exit function...
[ 26.366609 0:1 procfs:33] procfs module exit called
[ 26.367307 0:1 starry_api::kmod:166] Module(procfs) exited
[ 26.368144 0:1 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819ae000, num_pages=4
[ 26.370395 0:1 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b2000, num_pages=3
[ 26.372887 0:1 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b5000, num_pages=1
ls
line 38, file /home/codespace/.cargo/git/checkouts/axcpu-4807162bb1a28245/fc3154b/src/riscv/trap.rs: Unhandled Supervisor Page Fault @ 0xffffffc08044fa82, fault_vaddr=VA:0x0 (READ):
TrapFrame {
    regs: GeneralRegisters {
        zero: 0x4e0,
        ra: 0xffffffc080450b4c,
        sp: 0xffffffc0810bea80,
        gp: 0xffffffc08072d000,
        tp: 0x80047000,
        t0: 0x218a392cd3d5dbf,
        t1: 0xffffffc080d3e800,
        t2: 0xffffffc080d3e7f0,
        s0: 0x610,
        s1: 0x3a00,
        a0: 0xffffffc080c7e618,
        a1: 0x61,
        a2: 0x0,
        a3: 0x80000000,
        a4: 0x74493a3a726f7461,
        a5: 0x5e,
        a6: 0xffffffc0805d330c,
        a7: 0x0,
        s2: 0x7f,
        s3: 0x61,
        s4: 0xffffffc0814b9d38,
        s5: 0xffffffc08076e4e8,
        s6: 0xffffffc080c7e600,
        s7: 0xffffffc0810bec58,
        s8: 0xffffffc080c7e618,
        s9: 0xfff,
        s10: 0xffffffc0805d6736,
        s11: 0x2ae,
        t3: 0x49,
        t4: 0xfefefefefefefeff,
        t5: 0x8080808080808080,
        t6: 0x7272727272727272,
    },
    sepc: 0xffffffc08044fa82,
    sstatus: Sstatus {
        bits: 0x200042120,
    },
}
<backtrace disabled>

Rust Panic Backtrace:
#1  0xffffffc08052b6e0 - _Unwind_Backtrace + 0x22
#2  0xffffffc08039e52c - __rustc::rust_begin_unwind + 0xd0
#3  0xffffffc08054e55c - core::panicking::panic_nounwind_fmt + 0x0
#4  0xffffffc080526020 - riscv_trap_handler + 0x0
#5  0xffffffc080526170 - riscv_trap_handler + 0x150
#6  0xffffffc080524f58 - enter_user + 0x2a
line 59, file arceos/modules/axruntime/src/lang_items.rs: panic unreachable: 5
make[1]: Leaving directory '/workspaces/StarryOS/arceos'
@DINGBROK423 ➜ /workspaces/StarryOS (lkmod) $ 

```
**问题报告**

在 QEMU 中执行以下序列时会出现崩溃：

1. `cat /proc/meminfo` 触发 `procfs.ko` 按需加载并读取成功
2. 空闲超时后触发自动卸载（日志显示 unload + exit + 内存释放）
3. 随后执行普通命令（如 `ls`）触发内核异常（Page Fault / IllegalInstruction）

### 已确认成功部分

- 按需加载链路成功：`NotFound` → `with_ondemand()` → `load(procfs.ko)` → `procfs_init()` → `/proc` 可读
- 自动卸载链路可触发：定时器 tick 会进入 unload 路径并调用模块退出

### 根因判断（当前结论）

当前问题不在“是否触发按需加载”，而在“文件系统卸载的生命周期安全性”：

1. `axfs-ng-vfs` 的 `Location::unmount()` 主要完成命名空间摘除（断开挂载点），但未提供“活跃引用归零”保障
2. `axfs-ng` 文件/缓存对象会长期持有 `Location`，卸载后仍可能存在残留引用
3. 当模块内存被释放后，后续普通 VFS 路径访问可能命中失效对象，导致 trap
4. 自动卸载目前由 timer 回调驱动，执行上下文为 irq/preemption-disabled，进一步放大了卸载路径风险

### 目前补丁状态

已实现的缓解项：

- `ondemand-kmod`：修复 unload 失败时状态机误推进（失败不再强制置 `Unloaded`）
- `starry-api`：为 procfs 增加“两阶段卸载”尝试（先 unmount，后续 tick 再 free）

上述缓解项可降低时序风险，但不能从根本上保证卸载安全，崩溃仍可能出现。



**下周工作安排**

1. 查看模块卸载后内核崩溃退出原因（应该是文件系统VFS层的问题）
2. 代码规范化。
3. 尝试进行其他文件系统模块化处理。