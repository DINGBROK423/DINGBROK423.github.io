---
layout: post
id: week3
title: Week3
prev: week2
next: week4
---

## 2026.2.20-2026.2.28

**工作内容：用户态部分文件系统功能按需加载（procfs）**

1. **procfs** 是理想的一个按需加载模块，其功能集中于用户态中，对于内核捆绑程度较低。对于一个不需要 procfs 的应用场景，节省了 procfs 初始化的 CPU 时间以及procfs 数据结构占用的 **RAM**。

2. 在 StarryOS 中实现了 procfs 文件系统的按需加载机制。改造前，[mount_all()] 在启动阶段立即挂载所有虚拟文件系统（devfs、tmpfs、sysfs、procfs），无论用户程序是否需要访问 [proc]；改造后，procfs 在启动时仅将"挂载点路径 + 构造函数"注册到全局懒挂载注册表中，不分配任何数据结构、不创建虚拟文件节点，实现零初始化开销。当用户程序首次访问 [proc] 下的路径（如 `open("/proc/meminfo")`、`readlink("/proc/self/exe")`、[ls /proc] 等）时，内核在路径解析前主动检查注册表，发现匹配的待挂载条目后调用工厂函数完成 procfs 的一次性初始化和挂载，后续访问不再触发任何额外逻辑。实测启动日志验证：procfs 注册发生在内核初始化阶段（启动后约 1.46 秒），而真正挂载仅在用户首次执行 [ls /proc] 时触发，按需加载机制正确生效。

**procfs 按需加载设计架构**

- **注册表**（[mod.rs] 中的 [lazy_mount] 子模块）：使用 [spin::Mutex>] 在 [no_std] 环境下维护全局线程安全的注册表。每个 `Entry` 包含挂载点路径（如 ["/proc"]）和状态枚举（[Pending] 持有 [Box Filesystem + Send>] 工厂函数 / [Initialized] 表示已挂载）。提供三个接口：[register()] 注册条目并去重；[take_pending_for(path)] 通过路径前缀匹配定位条目，使用 [core::mem::replace] 原子地将状态从 [Pending] 转为 [Initialized] 并取出函数，保证并发场景下构造函数只执行一次；[is_registered()] 查询注册状态。

- **触发机制**（[fs.rs]中的 [with_fs_lazy()]函数）：采用"先检查后解析"的主动策略——在执行文件系统操作前，先调用 [try_lazy_mount(path)] 检查路径是否匹配待挂载条目。之所以不采用"解析失败后重试"的被动策略，是因为 [proc]目录已作为空目录存在于根文件系统（ext4 磁盘镜像）上，路径解析不会返回 [NotFound]，被动策略无法触发挂载。[try_lazy_mount()] 是幂等的，对已挂载条目仅需一次 spin lock + Vec 遍历即返回 false，性能开销可忽略。

- **系统调用覆盖**：改动涉及 6 个源文件。`sys_openat`（文件打开）和 [resolve_at]（通用路径解析，覆盖 [stat]/[statx]/`access`/`fstatat` 等）通过 [with_fs_lazy()]统一接入；[sys_readlinkat]（读取符号链接，覆盖 musl/glibc 常用的 `readlink("/proc/self/exe")`）同样使用 [with_fs_lazy()]；`sys_chdir`（切换工作目录）和 `sys_statfs`（获取文件系统信息）因直接操作 [FS_CONTEXT] 而单独在解析前插入 [try_lazy_mount()] 调用。

- **可扩展性**：注册表设计与具体文件系统解耦，未来可通过一行代码将其他可选文件系统改为按需加载，例如 [lazy_mount::register("/sys",sysfs::new_sysfs())],无需修改触发层代码。

**实例**

procfs 按需加载实现

```rust
// ==================== 1. 启动阶段：注册而非挂载 ====================
// api/src/vfs/mod.rs — mount_all()
pub fn mount_all() -> LinuxResult<()> {
    let fs = FS_CONTEXT.lock();

    // 必需文件系统 — 立即挂载（启动必须可用）
    mount_at(&fs, "/dev", dev::new_devfs())?;       // 设备文件系统
    mount_at(&fs, "/dev/shm", tmp::MemoryFs::new())?; // 共享内存
    mount_at(&fs, "/tmp", tmp::MemoryFs::new())?;    // 临时文件
    mount_at(&fs, "/sys", tmp::MemoryFs::new())?;    // sysfs（占位）
    drop(fs);

    // 可选文件系统 — 仅注册工厂函数到注册表，不调用 new_procfs()
    // 此时没有任何 procfs 数据结构被分配
    lazy_mount::register("/proc", || proc::new_procfs());
    // 日志输出: [lazy-mount] registered procfs at /proc (will mount on first access)
    Ok(())
}

// ==================== 2. 注册表核心数据结构 ====================
// api/src/vfs/mod.rs — lazy_mount 模块
enum State {
    Pending(Box<dyn FnOnce() -> Filesystem + Send>), // 持有构造函数
    Initialized,                                      // 已挂载，构造函数已消费
}

struct Entry {
    mount_point: String,  // 如 "/proc"
    state: State,
}

static REGISTRY: Mutex<Vec<Entry>> = Mutex::new(Vec::new());

// 注册：将条目推入 Vec，去重
pub fn register(mount_point: &str,
    factory: impl FnOnce() -> Filesystem + Send + 'static) {
    let mut reg = REGISTRY.lock();
    if reg.iter().any(|e| e.mount_point == mount_point) { return; }
    reg.push(Entry {
        mount_point: String::from(mount_point),
        state: State::Pending(Box::new(factory)),
    });
}

// 取出：前缀匹配 + 原子状态转换，保证只调用一次
pub fn take_pending_for(path: &str) -> Option<(String, Filesystem)> {
    let mut reg = REGISTRY.lock();
    for entry in reg.iter_mut() {
        // 匹配: path == "/proc" 或 path 以 "/proc/" 开头
        if path == entry.mount_point || path.starts_with(&format!("{}/", entry.mount_point)) {
            if let State::Pending(_) = &entry.state {
                let old = core::mem::replace(&mut entry.state, State::Initialized);
                if let State::Pending(factory) = old {
                    let mp = entry.mount_point.clone();
                    drop(reg);           // 释放锁后再调用工厂函数
                    return Some((mp, factory()));
                }
            }
            return None;  // 已经是 Initialized
        }
    }
    None
}

// ==================== 3. 触发层：路径解析前主动检查 ====================
// api/src/file/fs.rs — with_fs_lazy()
pub fn with_fs_lazy<R>(dirfd: c_int, path: &str,
    f: impl Fn(&mut FsContext) -> AxResult<R>,
) -> AxResult<R> {
    // 幂等检查：若 path 匹配 "/proc" 前缀且还未挂载 → 挂载
    //           若已挂载或不匹配 → 直接跳过（一次 lock + 遍历）
    crate::vfs::try_lazy_mount(path);
    with_fs(dirfd, &f)  // 正常执行文件系统操作
}

// ==================== 4. 实际挂载逻辑 ====================
// api/src/vfs/mod.rs — try_lazy_mount()
pub fn try_lazy_mount(path: &str) -> bool {
    if let Some((mount_point, fs)) = lazy_mount::take_pending_for(path) {
        let fsc = FS_CONTEXT.lock();
        match mount_at(&fsc, &mount_point, fs) {
            Ok(()) => {
                // 日志: [lazy-mount] mounted filesystem at /proc
                //        (triggered by access to /proc/meminfo)
                true
            }
            Err(e) => { warn!("failed to mount: {e:?}"); false }
        }
    } else {
        false  // 无匹配或已挂载
    }
}

// ==================== 5. 系统调用接入点 ====================
// sys_openat: open("/proc/meminfo") 触发懒挂载
with_fs_lazy(dirfd, &path, |fs| options.open(fs, &path))

// sys_readlinkat: readlink("/proc/self/exe") 触发懒挂载
with_fs_lazy(dirfd, &path, |fs| fs.resolve_no_follow(&path))

// sys_chdir / sys_statfs: 直接调用 try_lazy_mount(&path)
crate::vfs::try_lazy_mount(&path);
let entry = FS_CONTEXT.lock().resolve(&path)?;
```

运行结果

```bash
[  1.360327 0 axdisplay:20] Initialize display subsystem...
[  1.360926 0 axdisplay:26]   No display device found!
[  1.361839 0 axinput:21] Initialize input subsystem...
[  1.362788 0 axruntime:246] Initialize interrupt handlers...
[  1.364751 0 axruntime:258] Primary CPU 0 init OK.
[  1.365519 0:2 starry_api:25] Initialize VFS...
[  1.439341 0:2 starry_api::vfs:29] Mounted devfs at /dev
[  1.448172 0:2 starry_api::vfs:29] Mounted tmpfs at /dev/shm
[  1.452313 0:2 starry_api::vfs:29] Mounted tmpfs at /tmp
[  1.453940 0:2 starry_api::vfs:29] Mounted tmpfs at /sys
[  1.461119 0:2 starry_api::vfs:176] [lazy-mount] registered procfs at /proc (will mount on first access)
[  1.467175 0:2 starry_api:28] Initialize /proc/interrupts...
[  1.468158 0:2 starry_api:33] Initialize alarm...
[  1.613711 0:7 starry_api::task:36] Enter user space: ip=0x4056d00, sp=0x3fffffe60
Welcome to Starry OS!
[  1.747682 0:7 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(0x0)
[  1.751416 0:8 starry_api::task:36] Enter user space: ip=0x403d8d2, sp=0x3fffff590
SHLVL=1
HOME=/root
PWD=/
[  1.787119 0:8 starry_api::task:159] Task(8, "busybox") exit with code: 0
[  1.790959 0:8 starry_core::task:485] Send signal SIGCHLD to process 7
[  1.796158 0:7 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(0x0)
[  1.798599 0:7 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(WNOHANG)

Use apk to install packages.

[  1.809065 0:7 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(0x0)
[  1.813959 0:9 starry_api::task:36] Enter user space: ip=0x403d8d2, sp=0x3fffff590

starry:~# ls /proc
[196.301471 0:9 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(WUNTRACED)
[196.302874 0:10 starry_api::task:36] Enter user space: ip=0x403d8d2, sp=0x3fffff6c0
[196.329032 0:10 starry_api::vfs:29] Mounted proc at /proc
[196.329997 0:10 starry_api::vfs:132] [lazy-mount] mounted filesystem at /proc (triggered by access to /proc)
10          9           interrupts  meminfo2    self
7           instret     meminfo     mounts      sys
[196.351831 0:10 starry_api::task:159] Task(10, "busybox") exit with code: 0
[196.353275 0:10 starry_core::task:485] Send signal SIGCHLD to process 9
[196.357802 0:9 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(WUNTRACED)
[196.361613 0:9 starry_api::syscall::task::wait:64] sys_waitpid <= pid: -1, options: WaitOptions(WNOHANG | WUNTRACED)
```

**主要问题**

1. 由于我对于模块按需加载设计是参考Serverless运行时 AlloyStack。但是二者运行环境和设计架构存在很大差异。 AlloyStack 能直接使用 `dlopen`/`dlsym` 这些 Linux 系统调用来实现动态加载。但StarryOS作为系统宏内核，需要自己实现动态链接服务，否则在裸机环境下无法直接生成可加载的`.so`。

|              | AlloyStack                                       | StarryOS                                                     |
| ------------ | ------------------------------------------------ | ------------------------------------------------------------ |
| **本质**     | **Linux 进程**（LibOS / Unikernel 形态）         | **裸机内核**（[#![[no_std\]] + [#![no_main\] )               |
| **运行在**   | Linux 用户态，有完整的 glibc / 动态链接器        | 直接跑在硬件/QEMU 上，没有 OS 支撑                           |
| **编译目标** | `x86_64-unknown-linux-gnu`（standard target）    | `riscv64gc-unknown-none-elf` / `loongarch64-unknown-none-softfloat` |
| **链接方式** | Rust dylib → `.so` 文件，由 Linux 动态链接器加载 | 静态链接为一个裸机 ELF 二进制                                |

2. StarryOS作为宏内核，模块之间耦合依赖较深，不像 AlloyStack 那样通过函数指针解耦。例如目前正在处理文件系统模块按需加载，虽然axfs可以分离作为一个模块，但是却和内核空间功能有紧密联系，无法轻易将整个模块卸载。

3. axfs作为一个模块，粒度较粗，如果对文件系统部分功能进行更细粒度拆卸，需要涉及到外部库的修改。

**下一步的计划/建议**

1. 完善文件系统模块用户态功能按需加载。
2. 和老师进行交流，内核态功能是否需要进行按需加载设计。
3. 模块注册表进一步完善，适用于不同模块。
4. 寻找提高调用未加载模块效率方法（AlloyStack是采用函数指针调用库函数方式，但无法适用于静态链接的StarryOS）