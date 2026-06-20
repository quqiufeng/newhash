# Newhash — 架构设计文档

## 1. 概述

Newhash 是一套**通用分布式数据路由与弹性扩缩容调度方案**。它不绑定任何具体存储引擎，而是提炼出加机器时"需要把什么迁到新机器上"这个核心问题的通用解法。

### 核心思想

**把数据按 Namespace 组织为目录树，用一致性哈希环做路由，用实际负载做环的切分。加机器时，哈希环自动算出哪些 Namespace 需要迁移，精度到一个目录，复杂度 O(1)。**

---

## 2. 架构总览

```
                    ┌─────────────────────────┐
                    │      Coordinator        │
                    │   (Raft 3~5 节点集群)    │
                    │                         │
                    │  ┌───────────────────┐  │
                    │  │   HashRing        │  │
                    │  │   负载感知切分      │  │
                    │  │   CalcMigration() │  │
                    │  └───────────────────┘  │
                    └───────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                      │
          ▼                     ▼                      ▼
  ┌───────────────┐   ┌───────────────┐     ┌───────────────┐
  │   StorageNode  │   │  StorageNode  │     │  StorageNode  │
  │                │   │               │     │               │
  │ /data/        │   │ /data/        │     │ /data/        │
  │  ├─user/      │   │  ├─order/     │     │  ├─log/       │
  │  ├─shop/      │   │  ├─pay/       │     │  ├─backup/    │
  │  └─.../       │   │  └─.../       │     │  └─.../       │
  └───────┬───────┘   └───────┬───────┘     └───────┬───────┘
          │                   │                      │
          ▼                   ▼                      ▼
   ┌──────────┐        ┌──────────┐          ┌──────────┐
   │  Storage  │        │  Storage  │          │  Storage  │
   │  Engine   │        │  Engine   │          │  Engine   │
   │ (可插拔)   │        │ (可插拔)   │          │ (可插拔)   │
   └──────────┘        └──────────┘          └──────────┘
```

### 两层抽象

| 层 | 职责 | 实现 |
|----|------|------|
| **控制面** (Coordinator) | 节点管理、哈希环、迁移计算、元数据存储 | 通用，不感知底层存储 |
| **数据面** (Storage Engine) | Namespace 的 CRUD、目录级迁移、加锁 | 可插拔，由具体引擎实现 |

---

## 3. Namespace 树模型

### 3.1 定义

系统数据按 Namespace 路径组织为**目录树**，每个 Namespace 节点对应物理上的一个目录：

```
/data/storage/
  ├── user/                     ← Namespace "user"
  │   ├── profile/              ← Namespace "user/profile"
  │   │     ├── u1              ← KV 数据
  │   │     ├── u2
  │   │     └── ...
  │   └── setting/              ← Namespace "user/setting"
  │         └── ...
  ├── order/                    ← Namespace "order"
  └── log/                      ← Namespace "log"
```

### 3.2 路由单位

**路由与迁移的最小单位是 Namespace 节点，不是 Key，也不是物理机。**

- `hash("user")` → 在环上固定位置
- `hash("user/profile")` → 在环上另一个固定位置
- 每个 Namespace 独立路由，互不干扰

### 3.3 父→子 / 子→父 搜索

Namespace 路径天然包含层级关系，搜索引擎可以利用路径前缀实现：
- **父→子**：列出 `user/` 下的所有子 Namespace
- **子→父**：从 `user/profile/u1` 向上回溯到 `user`

---

## 4. 负载感知哈希环

### 4.1 设计目标

- 根据 Namespace 实际数据大小切分环，使各物理节点负载均衡
- 加机器时自动重算，精确得出哪些 Namespace 需要迁移
- 避免频繁颠簸，设阈值控制重平衡触发条件

### 4.2 环结构

```
环上的元素：
  ┌─────────────────────────────────────────────┐
  │  hash("user")    → size=80GB  → Node_1      │
  │  hash("order")   → size=200GB → Node_1      │
  │  hash("profile") → size=50GB  → Node_2      │
  │  hash("log")     → size=400GB → Node_2      │
  │  hash("backup")  → size=270GB → Node_3      │
  │  hash("pay")     → size=90GB  → Node_3      │
  │  hash("shop")    → size=30GB  → Node_1      │
  └─────────────────────────────────────────────┘

环的切分（负载均衡版）：
  总负载 = Σ size = 1120GB
  3 台节点，目标 = 1120/3 ≈ 373GB 每台

  按 Namespace 哈希排序，贪心累积到目标阈值：
  ┌─────────────────────────────────────────────┐
  │  Node_1: user(80) + order(200) + shop(30)   │
  │         = 310GB ✅  (接近 373)               │
  │  Node_2: profile(50) + log(400) = 450GB     │
  │         = 450GB ⚠️ 超过 373，需重平衡        │
  │  Node_3: backup(270) + pay(90) = 360GB ✅   │
  └─────────────────────────────────────────────┘
```

### 4.3 重平衡阈值策略

```
if max(节点负载) / min(节点负载) > 阈值(默认 1.2):
    触发全局重平衡
```

- 阈值可配置（默认 1.2，即偏差 20% 内不触发）
- 重平衡时重新切分环，最小化移动的 Namespace 数
- 优先搬大数据量的 Namespace（效率最高）

### 4.4 与传统一致性哈希的区别

| 对比项 | 传统一致性哈希 | Newhash |
|--------|--------------|---------|
| 环上元素 | 物理节点的虚拟节点 | Namespace 的哈希位置 |
| 切分方式 | 哈希空间均匀切 | 按 Namespace 实际负载切 |
| 迁移计算 | 所有 Key 重新哈希 | 只算 Namespace 的归属变化 |
| 迁移单位 | 零散 Key | 整个 Namespace 目录 |

---

## 5. 核心算法：迁移计算

### 5.1 输入

```
- 当前所有物理节点列表（含负载信息）
- 所有 Namespace 树（含路径、大小）
- 新加入的节点
```

### 5.2 算法步骤

```
1. 将新节点加入环（当前空负载）

2. 对每个 Namespace n:
    新归属 = 负载感知切分后的 Owner
    旧归属 = 当前 Owner
    if 新归属 == newNode && 旧归属 ≠ newNode:
        标记 n 待迁移

3. 去重（父迁则子不迁）:
    遍历所有被标记的 n:
        if n 的父节点也被标记:
            跳过（父节点迁移时会带走整个子树）
        else:
            加入最终迁移列表

4. 输出:
    [{SrcNode, DstNode, Namespace, Size}, ...]
```

### 5.3 复杂度

- **时间复杂度**：O(N)，N = Namespace 数量
- **空间复杂度**：O(M)，M = 待迁移 Namespace 数量
- 万级 Namespace 计算时间在毫秒级

---

## 6. 添加节点的完整流程

### 6.1 无停机方案（加只读锁）

```
阶段1：拓扑计算（控制面，微秒级）
─────────────────────────────────
  Node_New 向 Coordinator 注册
  Coordinator 运行 HashRing.CalcMigration()
  输出迁移列表：[{src: Node_1, ns: "user", size: 80GB}, ...]

阶段2：加锁（数据面，毫秒级）
─────────────────────────────────
  Coordinator 通知 Node_1 对 "user" 加只读锁
  Node_1 执行 StorageEngine.LockNamespace("user", READONLY)
  结果：
    ├── 已有读请求 → 继续服务
    ├── 新的读请求 → 正常处理（缓存）
    └── 写请求 → 排队等待，或返回"暂时只读"

阶段3：物理搬迁（数据面，时间取决于数据量）
─────────────────────────────────
  Coordinator 通知 Node_1 搬迁 "user" 到 Node_New
  Node_1 执行 StorageEngine.MigrateOut("user", Node_New.Address)
  Node_New 执行 StorageEngine.MigrateIn("user", Node_1.Address)
  传输方式（由 StorageEngine 决定）：
    ├── 磁盘 KV → rsync / sendfile 目录
    ├── 内存 KV → RDB dump + scp + load
    └── 文件存储 → cp / rsync

  搬迁期间：
    ├── 源节点 "user" 的读请求 → 正常服务（目录还在，数据完整）
    └── 其他 Namespace → 完全不受影响

阶段4：原子切换（控制面，毫秒级）
─────────────────────────────────
  传输完成后 Coordinator 发起 etcd 事务：
    ├── Owner("user") = Node_New
    ├── Epoch++
    └── 事务提交

阶段5：恢复与清理（数据面，异步）
─────────────────────────────────
  Coordinator 通知 Node_New → 解锁 "user"（读写全开）
  Coordinator 通知 Node_1 → 删除 "/data/storage/user/"
```

### 6.2 时序图

```
Node_New     Coordinator         Node_Src          Node_New
  │              │                  │                 │
  │──Register───►│                  │                 │
  │              │──CalcMigration──│                 │
  │              │──LockNamespace─►│                 │
  │              │   (READONLY)    │                 │
  │              │                  │                 │
  │              │──MigrateOut────►│                 │
  │              │                  │──Directory────►│
  │              │                  │   Transfer     │
  │              │                  │◄──ACK─────────│
  │              │◄──Complete──────│                 │
  │              │                  │                 │
  │              │──etcd Commit───│                 │
  │              │   Owner转交     │                 │
  │              │                  │                 │
  │              │──Unlock────────►│                 │
  │              │──Delete────────►│                 │
  │              │   旧数据        │                 │
```

---

## 7. 接口抽象

### 7.1 StorageEngine 接口

```go
// StorageEngine 存储引擎接口
// Coordinator 通过此接口与具体存储引擎交互。
type StorageEngine interface {
    // ListNamespaces 返回当前节点上所有 Namespace 的信息（路径+大小）
    ListNamespaces() ([]*NamespaceNode, error)

    // NamespaceSize 返回指定 Namespace 的数据量大小（用于负载均衡计算）
    NamespaceSize(nsPath string) (int64, error)

    // LockNamespace 对 Namespace 加锁
    //   mode = READONLY: 只允许读，写操作排队等待
    //   mode = UNLOCK:   恢复读写
    LockNamespace(nsPath string, mode LockMode) error

    // MigrateOut 将本节点的某个 Namespace 目录迁出到目标节点
    // 实现方决定具体传输方式（sendfile、rsync、RPC 流等）
    MigrateOut(nsPath string, dstAddr string) error

    // MigrateIn 从源节点接收某个 Namespace
    // 与 MigrateOut 对应
    MigrateIn(nsPath string, srcAddr string) error

    // DeleteNamespace 删除本节点上指定 Namespace 的目录和数据
    DeleteNamespace(nsPath string) error
}
```

### 7.2 NamespaceNode 定义

```go
type NamespaceNode struct {
    Path     string           // 命名空间路径，如 "user/profile"
    Size     int64            // 数据量（字节）
    Children []*NamespaceNode // 子 Namespace
}
```

### 7.3 迁移任务定义

```go
type MigrationTask struct {
    SrcNode   *Node  // 源节点
    DstNode   *Node  // 目标节点（新加入的节点）
    Namespace string // 要迁移的 Namespace 路径
    Size      int64  // 数据量大小
}
```

---

## 8. 可插拔的存储引擎

### 8.1 分布式 Redis 引擎

```
┌──────────────────────────────────────────────┐
│  RedisEngine                                 │
│                                              │
│  ListNamespaces() → 扫描 /data/redis/ 目录    │
│                                              │
│  LockNamespace("user") → 该 Namespace 停写    │
│     ● Redis 中 "user" 前缀的 key 只读         │
│                                              │
│  MigrateOut("user", dst) →                   │
│     ● SAVE/BGSAVE → dump.rdb                │
│     ● scp dump.rdb dst:/data/redis/user/    │
│     ● 通知 dst 加载                           │
│                                              │
│  MigrateIn("user", src) →                    │
│     ● 接收 dump.rdb                          │
│     ● 加载到内存                              │
│     ● 注册路由                                │
│                                              │
│  DeleteNamespace("user") →                    │
│     ● FLUSHDB 或删除目录                       │
└──────────────────────────────────────────────┘
```

### 8.2 分布式 LSM-Tree / RocksDB 引擎

```
┌──────────────────────────────────────────────┐
│  RocksDBEngine                               │
│                                              │
│  ListNamespaces() → 扫描 /data/rocks/ 目录    │
│                                              │
│  LockNamespace("user") →                    │
│     ● 该 RocksDB 实例暂停写入                  │
│     ● flush WAL                              │
│                                              │
│  MigrateOut("user", dst) →                   │
│     ● 硬链接快照目录                           │
│     ● rsync/ sendfile SST 文件到 dst         │
│                                              │
│  MigrateIn("user", src) →                    │
│     ● 接收 SST 文件                           │
│     ● 加载或移动到 RocksDB 的 db path        │
│     ● Replay WAL 增量                        │
│                                              │
│  DeleteNamespace("user") →                    │
│     ● DestroyDB("user")                      │
└──────────────────────────────────────────────┘
```

### 8.3 分布式文件存储引擎

```
┌──────────────────────────────────────────────┐
│  FileEngine                                  │
│                                              │
│  ListNamespaces() → 扫描目录树                │
│  LockNamespace("user") → 对目录加 flcok      │
│  MigrateOut("user", dst) → rsync 目录        │
│  MigrateIn("user", src) → rsync 接收         │
│  DeleteNamespace("user") → rm -rf            │
└──────────────────────────────────────────────┘
```

### 8.4 不同引擎的特点对比

| 引擎 | 迁移方式 | 热点数据 | 适用场景 |
|------|---------|---------|----------|
| Redis | dump + scp + load | 纯内存，快 | 缓存、会话、计数器 |
| RocksDB | SST rsync / sendfile | 磁盘顺序 IO | 时序、IoT、KV 存储 |
| 文件系统 | rsync / cp | 无额外开销 | 图片、对象、日志 |

---

## 9. 关键设计决策

### 9.1 为什么用 Namespace 做路由单位，而不是用 Key？

- 迁移精度 O(Namespace数) vs O(Key数)，差几个数量级
- 目录级搬迁，物理操作简单可靠
- 父→子、子→父的树形搜索天然支持
- 业务上 Namespace 是有意义的隔离边界

### 9.2 为什么用负载感知切分，而不是 vnode 统计均匀？

- vnode 只能保证哈希均匀，不能保证数据量均匀
- 10 个 Namespace 中有一个 500GB，vnode 再多也扛不住
- 实际负载切分能确保各节点磁盘使用率接近

### 9.3 为什么迁移时只加读锁，不停写？

- 加锁方案实现简单（一个 RWMutex）
- 读完全不受影响（缓存一直可用）
- 写暂停时间 ≈ 搬目录时间，秒级到分钟级
- 相比双写+WAL 方案，无数据不一致风险

### 9.4 为什么父迁子不迁？

- 目录结构天然保证包含关系
- 搬父目录时，子目录在物理上已经跟着走了
- 避免重复计算和重复搬迁

---

## 10. 部署结构

```
              ┌──────────────────────┐
              │   Coordinator        │
              │   (Raft 3/5节点)     │  ← 控制面
              │   etcd 存储元数据     │
              └──────────┬───────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐
    │ Storage   │  │ Storage   │  │ Storage   │
    │ Node 1    │  │ Node 2    │  │ Node 3    │
    │           │  │           │  │           │  ← 数据面
    │ Engine_A  │  │ Engine_A  │  │ Engine_B  │
    └───────────┘  └───────────┘  └───────────┘
```

- Coordinator 负责一致性决策（加机器、迁移、路由变更）
- 数据面节点负责实际数据存储与迁移执行
- 不同节点可用不同引擎（混布）
- 元数据存储在 etcd（Raft 保证一致性）

---

## 11. 总结

Newhash 方案以 **Namespace 目录树** 为核心抽象，在一致性哈希环上做 **负载感知切分**，实现了：

1. **精确到 Namespace 级别的迁移计算** — 加机器时 O(N) 算出要搬什么
2. **目录级物理搬迁** — 不需要逐 Key 扫描和重哈希
3. **可插拔的 StorageEngine 接口** — Redis、RocksDB、文件系统统一接入
4. **迁移期间缓存读可用** — 加读锁，读不受影响
5. **父迁子不迁** — 目录结构带来的天然优化
