---
id: week6
title: Week 6
prev: week5
next: week7
---

## 2026.3.16-2026.3.22

### 工作内容
1. 导致系统崩溃的问题已得到彻底修复，当前 `procfs.ko` 按需加载与自动卸载全流程均稳定通过集成测试。
2. 发布 [ondemand-kmod(按需加载模块)](https://github.com/DINGBROK423/ondemand-kmod)为外部库。提出相关issue追踪问题处理记录。
3. 探索用户态文件系统按需加载工作。

---

### 本周核心代码变更总结--Bug修复

#### 1. 卸载导致内核 Panic 或死锁 (在中断上下文中进行阻塞操作)

**现象**：等待模块空闲卸载时，系统随机发生与 `LruCache` 相关的 Panic（如 `unwrap()` 在 `None` 上失败）或直接死锁导致终端卡死。s

**根因**：
`procfs` 的底层实现使用 `LruCache` 和标准的自旋锁 / 互斥锁。之前系统将自动卸载扫描函数 `tick_ondemand()` 注册在了 `axtask::register_timer_callback` 中，这意味着整个扫描和模块释放操作都执行在**中断上下文**。在禁用中断和抢占的 IRQ 上下文去获取由普通 VFS 流程所竞争的锁，很容易引发上下文错误与内部状态不同步，产生死锁及释放逻辑被意外打断。

**解决方式**：
- **修改文件：** `api/src/lib.rs`, `api/src/kmod/ondemand.rs`
- 在 `api/src/lib.rs` 取消定时器里的轮询。
- 改为通过 `axtask::spawn()` 生成一个专属常驻的**任务 (Task)**，里面包含死循环与异步睡眠 `axtask::future::block_on(axtask::future::sleep(Duration::from_millis(500)))`。垃圾回收过程处于标准的任务上下文中执行，从而可以安全获取 VFS 或驱动的互斥锁。

#### 2. 物理内存复用引发缺页异常 

**现象**：当 `procfs.ko` 被自动卸载后，程序再次执行访问会正常触发重加载，但由于模块或系统尝试访问刚刚释放的物理内存以备他用，内核爆出“未处理缺页异常”并崩溃。


**根因**：
StarryOS 在加载 `.ko` 时，安全机制会对代码段设为“仅执行” (`Execute`)、不可写。管理物理映射的包装体 `KmodMem` 之前在自身 `Drop` 并把申请的多张物理 4K 内存页归还至系统全局分配池前，并没有重置页表状态。导致当再次把这个范围的物理地址分给进程或重加载的模块（作为读写用途）时，仍然因过往残留的安全隔离权限而引发异常拦截。

**解决方式**：
- **修改文件：** `api/src/kmod/mod.rs`
- 修改了 `KmodMem` 的 `Drop` 特质。明确在执行 `dealloc_frames(self.paddr...)` 前，针对相应的虚地址空间 (`kernel_aspace().lock().protect()`) 强制重置保护标志为基本的 `MappingFlags::READ | MappingFlags::WRITE`，消除生命周期外的状态污染。

#### 3. Rust 生命周期误用的编译报错 

**现象**：编译 `starry-api` 时抛出 `[E0716] temporary value dropped while borrowed`。

**根因**：
对 `procfs` 按需加载定制检查条件的函数 `ProcfsUsageChecker::is_in_use()` 中，存在 `let fd_table = FD_TABLE.scope(...).read();`。该语句会在链式调用执行完毕的当下立即析构 `scope()` 抛出的守卫哨兵对象，使得被获取引用的 `fd_table` 变成悬空借用。

**解决方式**：
- **修改文件：** `api/src/kmod/ondemand_builtin.rs`
- 将借用拆分为双步，手动维持外围对象的生命周期不在此大括号前结束：
  ```rust
  let fd_table_scope = FD_TABLE.scope(&scope_guard);
  let fd_table = fd_table_scope.read();
  ```

#### 4. 模块重编译时符号缺失导致的加载器 Panic

**现象**：完全构建基于 RISC-V 架构的新 `procfs.ko` 并打包为 `disk.img` 在 QEMU 中启动后，在最初挂载按需拦截时便报异常：`Symbol alloc::alloc::handle_alloc_error not found` 后 Panic。

**根因**：
Rust Nightly 更新后，对 `no_std` 与自定内存系统的二进制代码做了符号表裁剪优化。动态独立链接的模块文件内隐含调用了 `handle_alloc_error`，却无法在由主体内核提供的底层支持表 `vfs::KALLSYMS` 里找到对应符号地址供地址回填。

**解决方式**：
- **修改文件：** `api/src/kmod/mod.rs`, 以及 `make img` 命令重写磁盘文件系统。
- 在专门服务于模块装载的符号解析函数 `resolve_symbol` 中，补充显式的 fallback 分支。如果符号字典拿不到 `"handle_alloc_error"`，则将其主动定向到已导出的软陷阱 / 挂起函数点 `axhal::power::system_off`。之后通过强制运行 `make img ARCH=riscv64` 使新镜像里包含正确的链接规则与 ELF 模块，彻底解决了动态加载死锁的问题。


---

### 下周工作安排

1. 完成用户态文件系统按需加载工作。
2. 若提前完成1，可以尝试做块设备文件系统按需加载。