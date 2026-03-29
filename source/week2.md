---
layout: post
id: week2
title: Week 2
prev: week1
next: week3
---

## 2026.2.2-2026.2.7

独属于StarryOS的模块

外部模块：

| crate | 功能 | 说明 |
|---|---|---|
| starry-process | 进程管理 | fork/exec/wait/进程树、PID 分配 |
| starry-signal | POSIX 信号 | 信号投递、处理、掩码、sigaction |
| starry-vm | 虚拟地址空间 | 用户态页表、mmap/munmap/mprotect |

本地模块：

| 模块 | 功能 |
|---|---|
| syscall/ | Linux syscall 实现（clone, execve, futex, epoll...） |
| terminal/ | termios、行规程 (ldisc)、作业控制 |
| vfs/dev/ | devfs（tty/pty/pts/ptm, rtc, fb, urandom...） |
| vfs/proc.rs | procfs（/proc/self, meminfo, mounts...） |
| file/ | 特殊 fd（epoll, pipe, signalfd, pidfd） |
| core/futex | 快速用户态互斥锁 |
| core/shm | 共享内存 | 

尝试将POSIX 信号模块卸载然后进行测试，发现该模块和许多核心系统功能耦合很深（例如异常处理、用户地址空间初始化需要信号跳板地址）。有点草率选这个模块下手了，浪费了几天时间。

进一步研究代码，发现StarryOS的三个独立实现完整逻辑的模块都是和宏内核架构紧耦合的，先暂时不做卸载尝试了。

之前我的思路有些问题，不应该通过模块独立性程度来做按需加载尝试，而是应该通过是否可以不依赖其他模块独立运行或和其他模块耦合程度来判断。根据之前参与AlloyStack开发的经验，可以尝试从文件系统入手。而StarryOS的文件系统模块是基于ArceOS的文件系统模块进一步封装完善，整体较为宏大，我打算先选定其中特定一种文件系统做按需加载尝试。

**当前执行计划：**
1. 将fatfs/ext4模块单独卸载。
2. 编写对应测试函数。
3. 先简单实现一个懒加载的功能，调用模块缺失异常函数跳转到对应模块加载，然后重新执行测试函数。

**文件系统模块加载流程:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        启动流程                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  axruntime::rust_main()                                             │
│  ├── init_allocator()                                               │
│  ├── axtask::init_scheduler()                                       │
│  ├── axdriver::init_drivers()  ──► 探测块设备 (virtio-blk)          │
│  │                                                                  │
│  └── axfs::init_filesystems(block_devs)  ◄── 【文件系统初始化入口】   │
│       │                                                             │
│       ├── 1. 获取块设备                                              │
│       │      dev = block_devs.take_one()                            │
│       │                                                             │
│       ├── 2. 创建文件系统实例                                        │
│       │      fs = fs::new_default(dev)                              │
│       │      ├── ext4::Ext4Filesystem::new(dev)  [如果 feature=ext4] │
│       │      └── fat::FatFilesystem::new(dev)    [如果 feature=fat]  │
│       │                                                             │
│       ├── 3. 创建根挂载点                                            │
│       │      mp = Mountpoint::new_root(&fs)                         │
│       │                                                             │
│       └── 4. 初始化全局上下文                                        │
│              ROOT_FS_CONTEXT.call_once(FsContext::new(...))         │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  main() → starry_api::init()                                        │
│  │                                                                  │
│  └── vfs::mount_all()  ◄── 【虚拟文件系统挂载】                       │
│       │                                                             │
│       ├── mount("/dev", devfs)      ─► null, zero, urandom, tty...  │
│       ├── mount("/dev/shm", tmpfs)                                  │
│       ├── mount("/tmp", tmpfs)                                      │
│       ├── mount("/proc", procfs)    ─► meminfo, mounts, self/...    │
│       └── mount("/sys", tmpfs)                                      │
└─────────────────────────────────────────────────────────────────────┘
```

根文件系统实现（ext4/FAT）可以延迟到首次访问磁盘文件时加载。

**下一步的计划/建议**
1. 参考陈林峰同学工作：模块加载
2. 如果模块加载实现较为完善，可以直接在此基础上进行设计按需加载
3. 参考 https://github.com/Starry-OS/StarryOS/tree/dev 模块化更为完善的分支
