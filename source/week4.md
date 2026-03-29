---
id: week4
title: Week 4
prev: week3
next: week5
---

## 2026.3.1-2026.3.8

### 工作内容

1. 阅读陈林峰同学的工作日志与相关代码，了解模块可加载（LKM）的设计和架构，并总结提炼出阅读文档。在陈林峰同学工作基础上，进行按需加载的设计。
2. 重构一下实习日志文档。
3. 重新构建了按需加载的框架，正在尝试将procfs解除静态挂载并能独立出成为.ko文件，来适配LKM的架构。

**架构图**

```text
┌─────────────────────────────────────────────────────────────┐
│                    User Space (应用程序)                      │
│  open("/proc/meminfo")  stat("/proc")  readlink  access ... │
└────────┬───────────────────┬──────────────────┬──────────────┘
         │ sys_openat        │ sys_stat         │ sys_chdir ...
         ▼                   ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│              api/src/syscall/ + file/fs.rs                    │
│                                                              │
│  with_ondemand(path, || { VFS 操作 })                        │
│    ├─ 先执行 VFS 操作 (resolve / open / ...)                 │
│    ├─ 成功 → 直接返回                                        │
│    └─ NotFound → try_ondemand_load_path(path)                │
│                  ├─ 加载成功 → 重试 VFS 操作                  │
│                  └─ 无匹配 → 返回原始 NotFound                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│            api/src/kmod/ondemand.rs  (集成层)                 │
│                                                              │
│  REGISTRY: Once<ModuleRegistry<KmodOnDemandLoader>>          │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  KmodOnDemandLoader (impl ModuleLoader)            │      │
│  │    load()  → read_ko_file() → init_module()       │      │
│  │    unload()→ MODULES.find()  → delete_module()    │      │
│  └────────────────────────────────────────────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           ondemand-kmod/ (独立 no_std 库)                     │
│                                                              │
│  registry.rs   ← ModuleRegistry<L>  核心注册表               │
│  lifecycle.rs  ← State 状态机 + ModuleGuard 引用计数          │
│  trigger.rs    ← Trigger trait + PathPrefix/Syscall/Device   │
│  monitor.rs    ← IdleMonitor 空闲扫描及自动卸载               │
│  loader.rs     ← ModuleLoader / UsageChecker trait           │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         kmod-loader / ksym / kapi  (现有 LKM 基础设施)        │
│  ELF 解析 → 重定位 → 符号绑定 → init/exit 调用               │
└─────────────────────────────────────────────────────────────┘
```

**状态机**

每个受管理模块的生命周期状态：

```text
Registered ──on_access──► Loading ──success──► Active
    ▲                       │                  │    ▲
    │                     fail                 │    │
    │                       ▼         tick()   │  on_access
    │                    Unloaded ◄── Unloading │    │
    │                       ▲           ▲      ▼    │
    │                       │           └── Idle ───┘
    │                   on_access
    │                       │
    └───────────────────────┘ (重新加载)
```

| 状态 | 含义 | 转换条件 |
|------|------|---------|
| `Registered` | 已注册，尚未加载 | 初始状态 |
| `Loading` | 正在加载（某线程持有） | `on_access` 命中触发器 |
| `Active` | 已加载，可能有活跃引用 | `load()` 成功 |
| `Idle` | 已加载，引用计数为 0 | `tick()` 检测到无引用 |
| `Unloading` | 正在卸载 | 空闲超时且 `prepare_unload()` 成功 |
| `Unloaded` | 已卸载，等待重新加载 | `unload()` 完成 |


我将按需加载功能独立出一个外部库 **ondemand-kmod**

#### ondemand-kmod

##### `ondemand-kmod/src/loader.rs` — 加载器 trait

定义两个核心 trait，由内核集成层实现：

| Trait | 方法 | 说明 |
|-------|------|------|
| `ModuleLoader` | `load(name, ko_path) → Result<u64, LoadError>` | 加载 `.ko` 文件并执行 init |
| `ModuleLoader` | `unload(handle) → Result<(), UnloadError>` | 卸载模块并执行 exit |
| `UsageChecker` | `is_in_use() → bool` | 检查模块是否有活跃用户 |
| `UsageChecker` | `prepare_unload() → Result<(), ()>` | 预卸载清理（必须非阻塞） |


---

##### `ondemand-kmod/src/trigger.rs` — 触发器

`AccessEvent<'a>` 枚举表示三种访问事件：

```rust
pub enum AccessEvent<'a> {
    Path(&'a str),      // 文件路径访问
    Syscall(usize),     // 系统调用号
    Device(&'a str),    // 设备节点访问
}
```

`Trigger` trait 及三个内建实现：

| 实现 | 匹配规则 | 示例 |
|------|---------|------|
| `PathPrefixTrigger` | 路径前缀匹配（含边界检查） | `"/proc"` → 匹配 `/proc/meminfo`，不匹配 `/process` |
| `SyscallTrigger` | 精确系统调用号匹配 | `SyscallTrigger::new(321)` 匹配 SYS_bpf |
| `DeviceTrigger` | 设备路径前缀匹配 | `"/dev/null_blk"` → 匹配 `/dev/null_blk0` |

---

##### `ondemand-kmod/src/lifecycle.rs` — 生命周期

核心类型：

- **`State`** — 6 状态枚举（见上文状态机）
- **`ModuleDesc`** — 模块描述符
  - `name: &'static str` — 唯一模块名
  - `ko_path: &'static str` — `.ko` 文件路径
  - `idle_timeout_ticks: u64` — 空闲超时（0 = 禁用自动卸载）
  - `trigger: Box<dyn Trigger>` — 触发器
  - `usage: Option<Box<dyn UsageChecker>>` — 可选使用检查器
- **`ModuleGuard`** — RAII 引用守卫
  - 持有 `Arc<AtomicUsize>` 引用计数
  - Drop 时自动减计数
  - 存在期间阻止自动卸载
- **`AccessResult`** — `on_access` 返回值（`NoMatch`, `Loaded`, `Loading`, `LoadFailed`, `Unavailable`）
- **`ModuleInfo`** — 模块状态快照（只读）
- **`ManagedModule`** (pub(crate)) — 内部运行时记录

---

##### `ondemand-kmod/src/monitor.rs` — 空闲监视器

`IdleMonitor::tick()` 实现三阶段扫描算法：

```text
Phase 1 (持锁):
  ① Active → Idle: ref_count == 0 时转为空闲
  ② 检查 Idle 模块是否超时
  ③ 调用 prepare_unload() (必须非阻塞)
  ④ 标记为 Unloading, 收集待卸载列表

Phase 2 (无锁):
  ⑤ 调用 loader.unload(handle) (可能涉及 I/O)

Phase 3 (持锁):
  ⑥ 最终状态转为 Unloaded
```

将 I/O 操作移出锁范围，避免自旋锁持有期间的阻塞。

---

##### `ondemand-kmod/src/registry.rs` — 模块注册表

`ModuleRegistry<L: ModuleLoader>` 是核心数据结构：

| 方法 | 签名 | 说明 |
|------|------|------|
| `new` | `const fn new(loader: L) -> Self` | 创建空注册表 |
| `register` | `fn register(&self, desc: ModuleDesc) -> bool` | 注册模块 |
| `on_access` | `fn on_access(&self, event: &AccessEvent, now: u64) -> AccessResult` | 触发加载 |
| `acquire` | `fn acquire(&self, name: &str, now: u64) -> Option<ModuleGuard>` | 获取引用 |
| `tick` | `fn tick(&self, now: u64)` | 空闲扫描 |
| `force_unload` | `fn force_unload(&self, name: &str) -> Result<(), UnloadError>` | 强制卸载 |
| `state_of` | `fn state_of(&self, name: &str) -> Option<State>` | 查询状态 |
| `list_modules` | `fn list_modules(&self) -> Vec<ModuleInfo>` | 列出所有模块 |

`on_access` 采用**无锁加载模式**：先持锁确认需要加载并设为 `Loading`，释放锁后执行 `loader.load()`，再持锁更新状态。

#### StarryOS 集成层

##### `api/src/kmod/ondemand.rs` — 集成桥梁

**`KmodOnDemandLoader`** — 实现 `ModuleLoader` trait：

```rust
impl ModuleLoader for KmodOnDemandLoader {
    fn load(&self, name: &str, ko_path: &str) -> Result<u64, LoadError> {
        let elf_data = read_ko_file(ko_path)?;  // 从文件系统读取 .ko
        super::init_module(&elf_data, None)?;     // 调用现有 LKM 加载
        Ok(simple_hash(name))                     // 返回名称哈希作为 handle
    }

    fn unload(&self, handle: u64) -> Result<(), UnloadError> {
        // 通过 handle (名称哈希) 在 MODULES 全局表中查找
        let name = modules.keys().find(|k| simple_hash(k) == handle);
        super::delete_module(&name)?;             // 调用现有 LKM 卸载
    }
}
```

---

### 下周工作安排

1. 将procfs 拆出并创建对应.ko 内核模块，在init 函数中挂载 procfs，完成测试，发布仓库。
2. 代码规范化。
3. 尝试进行其他文件系统模块化处理。

---

### 老师建议

1. 外接文件系统（u盘）按需加载管理
2. 同一文件系统用户态和内核态对应按需加载（手动/自动）
https://github.com/0voice/kernel_awsome_feature/blob/main/%E8%AF%A6%E8%A7%A3%20FUSE%20%E7%94%A8%E6%88%B7%E6%80%81%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F.md

3. 学习一下linux 文件系统
4. 复现陈林峰同学的测试