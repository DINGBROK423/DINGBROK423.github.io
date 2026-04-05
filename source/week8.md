---
id: week8
title: Week 8
prev: week7
next: week9
---

## 2026.3.30-2026.4.5

### 工作内容

1. 完善用户态文件系统（FUSE）的内核态驱动，通过用户态测试（open、read）。
2. 解决FUSE用户态测试中崩溃和卡死问题。
3. 总结归纳文档。

#### 本周核心代码变更总结

### 1. 内核挂载逻辑重构：从硬编码到动态注册
*   **改动点**：重构了 `sys_mount` 系统调用，使用 `get_filesystem_creator` 动态注册机制取代了原来的 `if-else` 开关逻辑。
*   **原因**：
    *   **解耦**：以前挂载新的文件系统（如 FUSE）需要在内核核心代码中手动添加逻辑。现在，FUSE 模块在被加载时（Load）会向 VFS 注册自己。
    *   **按需触发**：当用户执行 `mount -t fuse` 时，内核如果发现 `fuse` 类型尚未注册，可以触发 `ondemand` 机制去寻找并加载 `fuse.ko` 模块。

### 2. FUSE 设备阻塞 I/O 支持 (`Starryfuse/src/dev.rs`)
*   **改动点**：为 `/dev/fuse` 字符设备添加了 `NodeFlags::BLOCKING` 标志。
*   **原因**：
    *   **解决死等问题**：由于 FUSE 守护进程（Daemon）通常通过 `read()` 等待内核发来的请求，如果不设置为阻塞模式，轮询（Poller）路径在未实现 `Pollable` 接口时会导致任务永久睡眠。
    *   **同步机制**：按需加载的 FUSE 模块必须能正确处理来自内核的同步请求，强制阻塞确保了守护进程在到达前能正确挂起。

### 3. VFS 节点元数据更新 (`Starryfuse/src/vfs.rs`)
*   **改动点**：实现了 `FuseNode` 的 `update_metadata` 方法。
*   **原因**：
    *   **资源清理**：在文件 release 或 cleanup 时，内核常会尝试更新时间戳（atime/mtime）。如果不实现此接口，会导致内核抛出告警。
    *   **健壮性**：虽然目前为了简化测试，对 `chmod/chown` 等操作报错（取决于用户态守护进程的支持），但基础的元数据维护是模块正常卸载和资源回收的前提。

### 4. 模块生命周期与清理 (`Starryfuse/src/lib.rs`)
*   **改动点**：在 `exit_fuse`（模块退出函数）中添加了 `unregister_filesystem("fuse")`。
*   **原因**：
    *   **防止悬空指针**：按需加载意味着模块可以被动态卸载（Unload）。如果不取消注册，内核 VFS 可能会保留指向已释放内存块（模块代码段）的指针，导致系统崩溃（Kernel Panic）。
    *   **闭环管理**：确保了模块“加载-注册-使用-注销-卸载”的完整闭环。

### 5. 测试守护进程的修正 (`Starryfuse/tests/fuse_test/src/main.rs`)
*   **改动点**：修正了 `readdir` 的 offset 处理逻辑，并改用带超时的 `poll()` 而非原生 `read()`。
*   **原因**：
    *   **修复活锁**：之前的 `readdir` 忽略了 offset，导致内核在列目录时进入无限循环。
    *   **支持正常卸载**：使用 `poll()` 配合超时，让守护进程能感知到文件系统已被 umount，从而主动退出并关闭 `/dev/fuse` 句柄，允许内核顺利回收 FUSE 模块。

#### Issue #2 解决历程

这一部分详细记录了我们在对齐远程 723fc6b 版本与当前本地最终可工作版本之间的过程。

底层 panic -> 逻辑层挂起阻塞 (卡死) -> 资源层异常反馈 -> 平稳回收释放

以下是基于 `723fc6b` 的完整改动总览。

| 故障类别 | 直接现象 | 根本原因 (Root Cause) | 关键修改文件 | 修复方案 | 结果 |
|---|---|---|---|---|---|
| **P1: 模块代码段的 Use-After-Free** | 卸载模块后系统抛出 `Unhandled Supervisor Page Fault (EXECUTE)`。 | VFS 或子系统中遗留了指向该模块代码段的指针。模块释放后，控制流跳转至该野指针触发指令级缺页。 | `api/src/kmod/ondemand.rs`<br>`modules/fuse/src/lib.rs`<br>`api/src/vfs/mod.rs` | **1. 强化卸载条件：** 仅当引用计数为零且无打开的描述符时允许卸载。<br>**2. 清理野指针：** 在 `fuse_exit()` 中彻底注销相关设备和资源。 | 消除了执行型缺页异常，实现了安全的动态加载/卸载闭环。 |
| **P2: 守护进程 IO 挂起 (Daemon Hang)** | 测试结束后守护进程持续挂起，导致模块无法卸载。 | 守护进程使用了同步阻塞型 `read()`。当内核不再发送请求时，进程进入永久等待，无法释放文件描述符。 | `Starryfuse/tests/fuse_test/src/main.rs` | 引入 IO 多路复用机制，使用带超时的 `libc::poll()`。侦测到超时后主动跳出循环并安全退出。 | 守护进程通过超时机制正常退出，正确释放资源。 |
| **P3: 目录读取活锁 (Readdir Livelock)** | 底层日志被高频循环的 `FUSE_READDIR` 事件抢占。 | 用户态模块未正确处理 `offset` 语义，始终返回首个目录项，导致内核因收不到 EOF 标志而陷入重试读取的死循环。 | `Starryfuse/tests/fuse_test/src/main.rs` | 完善状态机逻辑：收到 `offset != 0` 请求时回复空结构，向内核传递 EOF 信号。 | VFS 正确识别文件末尾，终止检索并跳出死循环。 |
| **P4: 字符设备 IO 阻塞语义缺失** | 设备在被 `poll`/`select` 监管时，唤醒与挂断行为不稳定。 | `/dev/fuse` 驱动未向内核显式注册字符设备的阻塞属性。 | `Starryfuse/src/dev.rs` | 重写 `flags()` 方法，显式返回 `NodeFlags::BLOCKING` 特征。 | 内核进程调度器正确感知节点阻塞特征，多路复用表现恢复正常。 |
| **P5: VFS 元数据更新引发panic** | 节点注销时频繁打印 `Failed to update file times: OperationNotSupported`。 | VFS 在析构 inode 时默认尝试调用 `update_metadata()` 更新时间戳，而虚拟文件系统层缺乏此支持。 | `Starryfuse/src/vfs.rs` | **实现兼容接口：** 手动实现 `update_metadata()`，静默处理时间更新请求并返回 `Ok(())`。 | 适配了 VFS 的通用清理流程，屏蔽了不必要的磁盘同步警告。 |
| **P6: 后台监控引发内核崩溃** | 后台定时检查模块空闲时，随机引发 `ext4_bcache_free` 异常。 | 检查器盲目遍历所有打开文件并获取路径，误触发了底层磁盘缓存操作或死锁。 | `api/src/kmod/ondemand_builtin.rs` | **精细化检查**：优先校验文件系统类型，非内存设备 (`devfs`) 绝对不尝试获取路径。 | 避免了对常规系统文件的误伤，消除了随机崩溃隐患。 |



#### FUSE模块按需加载核心设计片段

```rust
// ==================== 1. 触发点安全过滤与前置校验 ====================
// api/src/kmod/ondemand_builtin.rs 
// 优先比对 filesystem name 和 devfs 内存特征，杜绝误触底层 Ext4 磁盘的 Cache，进而引发 ext4_bcache_free 崩溃。
#[inline]
fn is_fuse_node(loc: &axfs_ng_vfs::Location) -> bool {
    if loc.filesystem().name() == "fuse" {
        return true;
    }
    if let Ok(metadata) = loc.metadata() {
        if metadata.node_type == axfs_ng_vfs::NodeType::CharacterDevice {
            if metadata.rdev == axfs_ng_vfs::DeviceId::new(10, 229) {
                return true;
            }
        }
    }
    false
}

impl UsageChecker for FuseUsageChecker {
    fn is_in_use(&self) -> bool {
        let mut process_data_list = alloc::vec::Vec::new();
        // 收集任务信息，避免在获取 VFS 锁时长时间把持自旋锁
        for task in tasks() {
            process_data_list.push(task.as_thread().proc_data.clone());
        }

        for proc_data in process_data_list {
            let scope_guard = proc_data.scope.read();

            // 前置调用 is_fuse_node() 过滤，取代原版的 .path().contains()
            if is_fuse_node(FS_CONTEXT.scope(&scope_guard).lock().current_dir()) {
                return true;
            }

            // 同样安全地遍历用户进程持有的全量 FD
            let fd_table_scope = FD_TABLE.scope(&scope_guard);
            let fd_table = fd_table_scope.read();
            for fd in fd_table.ids() {
                if let Some(fd_obj) = fd_table.get(fd) {
                    let any = fd_obj.inner.clone().into_any();
                    if let Some(file) = any.downcast_ref::<crate::file::File>() {
                        if is_fuse_node(file.inner().location()) {
                            return true;
                        }
                    }
                }
            }
        }
        false
    }
}

// ==================== 2. 文件系统动态挂载表，抹除硬编码 ====================
// api/src/syscall/fs/mount.rs
// 系统解除针对特定文件系统（如 ext4, procfs）的内部探测 if-else 判断。
// 不受 FS_CONTEXT.lock() 牵制的专属 registry 匹配机制，令 FUSE 模块加载后能够动态挂载自身。
pub fn sys_mount(
    source: *const c_char,
    target: *const c_char,
    fs_type: *const c_char,
    _flags: i32,
    _data: *const c_void,
) -> AxResult<isize> {
    let source = vm_load_string(source)?;
    let target = vm_load_string(target)?;
    let fs_type = vm_load_string(fs_type)?;
    
    let fs = if fs_type == "tmpfs" {
        MemoryFs::new()
    } else if let Some(creator) = crate::vfs::get_filesystem_creator(&fs_type) {
        // 如果系统内部无法处理，下行给全局注册表闭包 (Creator) 支持 On-Demand
        creator()?
    } else {
        return Err(AxError::NoSuchDevice);
    };

    let target = FS_CONTEXT.lock().resolve(target)?;
    target.mount(&fs)?;

    Ok(0)
}


// ==================== 3. 用户态服务多路复用，规避进程僵死挂起 ====================
// Starryfuse/tests/fuse_test/src/main.rs
// 舍弃死板的阻塞 read。利用 libc::poll 获取超时控制，以便在空闲时主导生命周期控制并平滑退出。
loop {
    let mut pfd = libc::pollfd {
        fd: fuse_fd,
        events: libc::POLLIN,
        revents: 0,
    };
    
    // Poll 超时设为 20ms，若队列全空则短暂挂起后苏醒
    let poll_ret = unsafe { libc::poll(&mut pfd as *mut libc::pollfd, 1, 20) };
    if poll_ret < 0 {
        let err = std::io::Error::last_os_error();
        if err.kind() == std::io::ErrorKind::Interrupted {
            continue;
        }
        break;
    }

    // 识别到已无后续指令，主动读取停止标志位退出循环，从根源解决 daemon hang
    if poll_ret == 0 || (pfd.revents & libc::POLLIN) == 0 {
        if child_done.load(std::sync::atomic::Ordering::SeqCst)
            && child_done_since
                .map(|t| t.elapsed() >= Duration::from_millis(500))
                .unwrap_or(false)
        {
            println!("Test complete, daemon exiting.");
            break;
        }
        continue;
    }

    let n = match fuse_dev.read(&mut buf) {
        Ok(n) => n,
        Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => continue,
        Err(e) => break,
    };
    
    // ... [处理具体的 FUSE opcode 逻辑]
}
```

#### 运行测试

```bash
[  5.774269 0:11 starry_api::kmod::ondemand:43] [ondemand] module 'fuse' loaded, handle=0x17c96ff18
Opened /dev/fuse
Mounted /mnt/fuse successfully
About to fork self-test child...
fork returned 13
Spawned self-test child pid=13
Received FUSE request: opcode=26, unique=1, nodeid=0
Sent INIT response
fork returned 0
=== FUSE Self-Test Starting ===
[TEST] ls /mnt/fuse:
Received FUSE request: opcode=28, unique=2, nodeid=1
Sent READDIR response (offset=0, bytes=96)
  test.txt
Received FUSE request: opcode=28, unique=3, nodeid=1
Sent READDIR response (offset=3, bytes=0)
[TEST] ls /mnt/fuse: PASS
Received FUSE request: opcode=1, unique=4, nodeid=1
Sent LOOKUP response for 'test.txt'
Received FUSE request: opcode=3, unique=5, nodeid=100
Sent GETATTR response for nodeid=100
Received FUSE request: opcode=3, unique=6, nodeid=100
Sent GETATTR response for nodeid=100
Received FUSE request: opcode=3, unique=7, nodeid=100
Sent GETATTR response for nodeid=100
Received FUSE request: opcode=15, unique=8, nodeid=100
Sent READ response (nodeid=100, offset=0, req_size=4096, bytes=13)
Received FUSE request: opcode=3, unique=9, nodeid=100
Sent GETATTR response for nodeid=100
[TEST] read test.txt: PASS (contents: "hello, fuse!\n")
=== FUSE Self-Test Complete ===
Self-test child exited, status=0
Test complete, daemon exiting.
starry:~# [ 11.153416 0:6 starry_api::kmod::ondemand:55] [ondemand] unload handle=0x17c96ff18
[ 11.154924 0:6 kmod_loader::loader:122] Calling module exit function...
[ 11.156977 0:6 fuse:53] Fuse module exit called.
[ 11.157721 0:6 starry_api::kmod:179] Module(fuse) exited
[ 11.158666 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819af000, num_pages=9
[ 11.160802 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819b8000, num_pages=5
[ 11.161658 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819bd000, num_pages=1
[ 11.162995 0:6 starry_api::kmod:74] KmodMem::drop: Deallocating paddr=PA:0x819be000, num_pages=1
exit
make[1]: Leaving directory 
```

### 下周工作安排

1. 完善benchmark，进行内核资源利用提升测试。
2. 尝试接入Rust官方库提供的用户态文件系统。
3. 在 `Starryfuse/src/dev.rs` 中，为 `FuseDev` 完整实现操作系统的 `Pollable` 接口。我们需要自主提供一套合规的异步唤醒逻辑。
4. 整理文档和代码仓库，准备汇报。
