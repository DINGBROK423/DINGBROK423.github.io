---
id: week9
title: Week 9
prev: week8
next: week10
---

## 2026.4.6-2026.4.12

### 工作内容


#### 1. 字符设备的异步 I/O (Poller / Waker) 支持完成

在早期的设计中，为了绕开 `read("/dev/fuse")` 导致的线程挂起死锁，我们曾在 `FuseDev` 里通过 `NodeFlags::BLOCKING` 做了直接的同步阻塞处理。目前，该部分已经被重构，彻底改为了通过 `axpoll` 与内核调度层直接对接的事件驱动模式。

**修改内容**：

1. **引入 PollSet 作为调度中心**：
   在 `Starryfuse/src/dev.rs` 中，我们为底层的 `FuseConnection` 增加了 `poll_set: PollSet` 以作为内核独立的等待队列。过去使用的强制阻塞标志 `NodeFlags::BLOCKING` 已被彻底清理，恢复为非阻塞标准位 `NodeFlags::empty()`，并通过覆盖 `as_pollable` 接口允许内核的多路复用器接管该设备。

2. **完整接入 Pollable 事件处理**：
   为 `FuseDev` 正式实现了 `axpoll::Pollable` 接口：当检查请求队列时，如果有待处理的数据则上报 `IoEvents::IN` 读状态；否则，就把当前调用读取的用户态任务通过 `register` 方法安全地寄存入休眠队列排队。

3. **内核下达请求时的安全唤醒**：
   在 VFS 层下发请求命令的关键路径（`Starryfuse/src/vfs.rs`）中，当新的任务指令投入队列时会立刻触发 `conn.poll_set.wake()`。这会自动拉起休眠中的处理程序立即开始工作。

#### 2. 阻塞 IO 与线程唤醒机制

由于标准 `fuse` 库的主循环依赖阻塞式的文件读写，需要修改 `dev.rs` 中的 `read_at` 方法。之前在没有请求时直接返回了 `VfsError::WouldBlock`（对应 EAGAIN），这会导致库报错跳出或死循环。我们需要引入 `WaitQueue`（等待队列），在 `conn.pending.is_empty()` 时将当前线程挂起，并在 FUSE 请求到达时唤醒它。

基于目前的按需加载架构，FUSE 模块能且只能在所有 FUSE 文件系统都被卸载、且所有的 fuse 进程文件描述符被关闭经过一段空闲超时后，才会触发卸载。 

由于用户态 `fuse` 库常态下就是阻塞在 fuse 的 `read()` 调用上的，这意味着 **只要 `fuse` daemon 没有退出，它的 fd 是一直挂着存在的，`FuseUsageChecker` 的 `is_in_use()` 将始终返回 true，从而绝对保证按需加载器不会在工作途中强行把内核模块卸载掉。**

因此，我们需要实现的方案，就是让库安稳地通过 Sleep 原语挂起在 `read()` 上，同时提供一种能够在 `umount` 或连接终止时向它发送 EOF 从而触发卸载的退出手段。 

**修改内容**：

1. **引入睡眠等待队列 (WaitQueue)**
* **改动文件：** dev.rs 
* **改动内容：** 在 `FuseConnection` 结构体中新增一个从 `starry_core` 借用的等待队列，以及一个标志位用于判断是否要主动断开。
    ```rust
    use starry_core::futex::WaitQueue;
    
    pub struct FuseConnection {
        pub pending: Vec<...>,     
        pub processing: BTreeMap<...>,
        pub poll_set: PollSet,
        pub wait_queue: WaitQueue, // <== 新增：内核阻塞挂起队列
        pub aborted: bool,         // <== 新增：断连标志，方便通知退出
        // ...
    }
    ```
* **目的：** 利用现成的协程阻塞机制，避免因 busy-waiting (死循环轮询) 造成 CPU 100% 占用，同时完全兼容同步阻塞调用的 `fuse` crate。

2. **将 `read_at` 由“轮询拦截（EAGAIN）”改为“阻塞排队（Block）”**
* **改动文件：** dev.rs -> `DeviceOps for FuseDev` -> `read_at` 
* **改动内容：** 移除原先 `if conn.pending.is_empty() { return Err(VfsError::WouldBlock); }` 的硬报错，替换为一个阻塞等待循坏：
    ```rust
    loop {
        let mut conn = self.conn.lock();
        if conn.aborted {
            return Ok(0); // 发生中止（如收到退出信号），返回 0（EOF结束符），daemon读到会主动退出并关闭 fd
        }
        if !conn.pending.is_empty() {
            // ... (执行原本的取包出队拷贝逻辑并 return) ...
        }
        // 当为空时，解锁并安全睡眠，直到下一次唤醒
        drop(conn);
        self.conn.lock().wait_queue.wait_if(1, None, || {
            let conn = self.conn.lock();
            conn.pending.is_empty() && !conn.aborted
        }).unwrap_or(false); 
    }
    ```
* **目的：** 适配 FUSE 库的阻塞行为。用户态读取直接休眠，不吃 CPU。当且读取到 0 字节时，标准 FUSE 守护进程知道连接已终止，就可以走清理逻辑，继而引发内核彻底释放对应数据，倒逼 `UsageChecker` 认为它真的 Idle 从而彻底卸除 FUSE 模块。

3. **唤醒 WaitQueue** 
* **改动文件：** vfs.rs（或产生请求的代码处，例如 `send_request_and_wait`）以及卸载准备处
* **改动内容：** 
  - 当 VFS 侧推送一个新的请求给到 Daemon `conn.pending.push(...)` 时，调用 `wait_queue.wake(1, 1)` 以叫醒正阻塞在 `read_at` 的 Daemon 进程。
  - 在类似卸载 / 断开挂载连接处，可以将 `conn.aborted` 置 true，然后调用 `wait_queue.wake(usize::MAX, 1)`。
* **目的：** 当有新请求到来或主内核销毁 FUSE 连接时，挂起的 `read()` 操作能立即恢复执行以接手操作或是安全退出。

**总结：** 利用内核的 WaitQueue 将 `fuse` Daemon 阻塞挂起以响应标准规范，并通过 `aborted` 等信号给它返回 EOF (0) 间接促使守护进程从用户空间自行关闭进程。

#### 3. 修改 build.mk 脚本

starryfuse 库的代码完全没有被静态链接进去，出现了未定义符号 (UND Symbol)。


```makefile
  kmod_extra_rust_libs = $(wildcard $(TARGET_DIR)/$(TARGET)/$(MODE)/deps/libstarryfuse-*.rlib)
```

#### 4. 内核侧 Starryfuse 还需要继续实现部分 VFS 接口
虽然目前的测试能跑通 `ls(Readdir + Lookup)` 和 `cat(Getattr + Read)`，但目前的内核层驱动 `vfs.rs` 仅仅提供了一小撮最基本的操作。

**修改内容**：

修改 vfs.rs（在原有的基础上增加缺失的特性转发）。

我们为 `FuseNode` 补全 `axfs_ng_vfs::DirNodeOps` 和 `axfs_ng_vfs::FileNodeOps` 中缺失的标准操作，这样就可以把内核层的改动和文件创建、移动、权限操作打通。

1.  **文件修改与属性 (FileNodeOps & NodeOps)**
    *   **`write_at`**: 将 VFS 传下来的缓冲区连同 offset 拼装成 `FuseWriteIn` 并附带 payload 发给 daemon 的 `Write` 请求。
    *   **`truncate`**: 把大小变化转成 `FuseSetattrIn` 并置标志发送 `Setattr` 请求。(目前可以映射入 `update_metadata` 方法)

2.  **目录操作 (DirNodeOps)**
    *   **`mkdir`**: 接收名称和权限，组装 `FuseMkdirIn` 及名称并触发 `Mkdir`。
    *   **`rename`**: 将旧文件名和新文件名打包（这属于比较特殊的请求拼包，需要把两个 C-style 字符串连起来），然后发送 `Rename`。
    *   **`remove` / `rmdir`**: 发送对应的 `Unlink` 或 `Rmdir` 指令。

#### 5.复用外部用户态侧fuse驱动 

**链接**
https://github.com/cberner/fuser

**目录结构：**

```text
Starryfuse/
├── Cargo.toml                 # Kernel 驱动的配置文件，no_std
├── src/                       # Kernel 驱动源码：vfs.rs, dev.rs 等
├── tests/                     # 测试脚本和简单测试
│
└── starry_fuser/              # 专门给用户态用的 FUSE 适配器 (std 环境)
    ├── Cargo.toml             # 
    └── src/
        └──lib.rs             # 库入口，包装 fuser API 和 StarryOS 组合
```


### 下周工作安排

1. 完善benchmark，进行内核资源利用提升测试。
2. 工程收尾。
3. 整理文档和代码仓库，准备汇报。