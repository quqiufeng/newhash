# Newhash — 架构设计文档

## 1. 概述

Newhash 是一套**通用分布式数据路由与弹性扩缩容调度方案**。它不绑定任何具体存储引擎，而是提炼出加机器时"需要把什么迁到新机器上"这个核心问题的通用解法。

### 核心思想

**把数据按 Namespace 组织为目录树，用一致性哈希环做路由，用实际负载做环的切分。加机器时，哈希环自动算出哪些 Namespace 需要迁移，精度到一个目录，复杂度 O(1)。**

### 设计哲学：Namespace 是自然容量边界，不追求机器间平均负载

**现实中，只有 20% 的 Namespace 消耗 80% 的资源（Zipf 定律）。**

热门的资源永远消耗最大的服务器资源——`user_session` 突然 2 亿用户，它就该独占一台大内存机器；`archive_log` 常年无人问津，跟其他小 Namespace 挤一台小机器就够了。

```
现实负载分布：
  ┌─────────────────────────────────────┐
  │ "user_session" (50GB, 100K QPS)     │ → 独占大机器
  │ "feed"          (30GB, 80K QPS)     │ → 独占大机器
  │ "cart"          (10GB, 20K QPS)     │ → 跟小的一起
  │ "archive_log"   (1GB,  10  QPS)     │ → 跟小的一起
  │ "backup_config" (500MB, 1  QPS)     │ → 跟小的一起
  └─────────────────────────────────────┘
```

传统分布式方案追求"每台机器负载完全平均"，这不符合现实规律。为了平均硬把一个 Namespace 拆散到多台机器，引入跨节点通信、分布式事务、一致性哈希——复杂度和收益完全不成正比。

**Newhash 的选择：顺着规律走，不对抗它。**

- Namespace 大 → 给它大机器（换一台更大的机器，搬一个 bin 文件就行）
- Namespace 小 → 小机器上挤一挤
- Namespace 长大了 → 换台大机器，不需要拆散
- 加机器 → 不是"为了平均而搬数据"，而是"这个 Namespace 长大了，该换房子了"

**Namespace 就是自然的容量和资源单位。** 以它为基础做路由和迁移，所有复杂的分布式问题都被挡住外面了。

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

### 关键性质：每台机器独立运行，无跨节点协作

**Namespace 完整落在一台机器上**，这个性质是一切的基础：

```
客户端读 "/coding/cpp/move"
  → hash("/coding/cpp/move") → Node_3
  → 只连 Node_3，不涉及其他机器

客户端写 "/agent/personality"
  → hash("/agent/personality") → Node_1
  → 只连 Node_1，不涉及其他机器
```

这意味着：

- **存储引擎不需要改**：每台机器跑一份独立的单机实例（如 kvCache 的单线程 mmap 模型），不需要分布式事务、分布式锁、跨节点查询
- **加机器不影响现有代码**：新机器也跑同样的单机实例，只多了一个路由注册
- **全局复杂度 = 单机复杂度 + Coordinator**，Coordinator 只负责搬家和路由，不参与数据路径

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

### 3.3 核心性质：Namespace 是哈希分布 + 迁移单位的统一

**首要目的：利用 Namespace 的哈希均匀性做路由分布。**

```
hash("user")    → 0x7A3F   ← 大 Namespace，散开
hash("order")   → 0xB1E2   ← 另一个位置
hash("video")   → 0x3C9D   ← 又一个位置
...
```

不管 Namespace 大小，`hash(path)` 在环上总是均匀分布。大 Namespace 自然散到不同机器，小 Namespace 自动填缝。这是哈希函数本身的特性，也是整个方案的基石。

**附带效应：迁移时整体搬目录。**

```
hash("user") → Node_1
  → user 的所有 Key 完整落在 Node_1 的 /data/storage/user/ 下
  → 没有"半个 user 在 Node_1、半个在 Node_2"的情况
```

所以搬迁时是目录级别的原子操作，不需要逐 Key 扫描和重哈希。

```
正确理解主次关系：
  首要 → Namespace hash 使环上均匀分布（核心价值）
  次要 → 目录级整块迁移（附带的自然结果）

不正确的理解：
  ❌ "为了搬目录方便才用 Namespace"
  ✅ "Namespace 先保证了哈希均匀，迁移时自然整块搬"
```

对比纯 Key 级别的哈希分片：

| 对比 | 纯 Key 哈希 | Namespace 哈希 |
|------|------------|----------------|
| 分布 | Key 均匀散落 | Namespace 均匀散落 |
| 迁移 | 一堆 Key 逐个算、逐个搬 | 一个 Namespace 目录一次搬完 |
| 原子性 | 没有，部分搬走部分留下 | 完整，要么全迁要么全不迁 |

### 3.4 父→子 / 子→父 搜索

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
环上的元素（按 hash 排序后）：
  ┌─────────────────────────────────────────────┐
  │  hash("user")    → size=80GB                │
  │  hash("order")   → size=200GB               │
  │  hash("profile") → size=50GB                │
  │  hash("log")     → size=400GB               │
  │  hash("backup")  → size=270GB               │
  │  hash("pay")     → size=90GB                │
  │  hash("shop")    → size=30GB                │
  └─────────────────────────────────────────────┘

环的切分（负载均衡版）：
  总负载 = 1120GB，3 台节点，目标 = 1120/3 ≈ 373GB 每台

  按 hash 值从小到大排序后，从前到后贪心累加，到目标附近切分：
  ┌─────────────────────────────────────────────┐
  │  hash最小                          hash最大  │
  │    ↓                                 ↓      │
  │  Node_1: user(80) + order(200) + profile(50)│
  │         = 330GB ✅  (接近 373)               │
  │  Node_2: log(400) = 400GB ⚠️ 需重平衡       │
  │  Node_3: backup(270) + pay(90) + shop(30)   │
  │         = 390GB ✅                          │
  └─────────────────────────────────────────────┘

  (注：例子中的 Namespace 名称仅为可读性，实际排序由 hash 值决定，名称顺序不代表 hash 大小)
```

### 4.3 约束条件

#### 约束一：哈希有序连续分配

**Namespace 在环上必须按哈希顺序连续分配，不允许交错。**

```
正确（连续）：
  Node_1: hash(a) < hash(b) < hash(c)   ← 三段连续
  Node_2: hash(d) < hash(e) < hash(f)   ← 三段连续
  
错误（交错）：
  Node_1: a, c, e        ← 跳过了 b、d、f，不连续
  Node_2: b, d, f
```

这个约束保证了算法简单高效（一次贪心遍历），代价是调度自由度受限。**接受此约束**，因为最终目的是精确到每个 Namespace 的迁移，而非全局最优负载均衡。

#### 约束二：只一个 Namespace 就不需要本系统

**本系统在只有一个 Namespace 时没有意义。**

```
只有一个 Namespace "all_data"：
  10 台机器 → "all_data" 只能完整落在一台机器上
  → 那和单机没有区别，不需要分布式路由
```

**原因：** 本系统的核心就是"多个 Namespace 在环上均匀分布 + 负载感知切分"。只有一个 Namespace 时，没有可分布的对象，路由层退化为一个查表项，所有价值归零。

**要求：** 部署时 Namespace 数量至少远大于物理节点数量（推荐 100:1 以上），哈希均匀性和负载均衡效果才明显。

### 4.4 重平衡阈值策略

```
if max(节点负载) / min(节点负载) > 阈值(默认 1.2):
    触发全局重平衡
```

- 阈值可配置（默认 1.2，即偏差 20% 内不触发）
- 重平衡时重新切分环，最小化移动的 Namespace 数
- 优先搬大数据量的 Namespace（效率最高）

### 4.5 已知边界：单一大 Namespace

**问题：** 当一个 Namespace 的数据量超过单机目标负载时，该 Namespace 所在的节点必然超载。

```
例子：
  hash("video") → 800GB
  目标负载: 200GB/台

  无论怎么切，"video" 所在的节点至少有 800GB，比其他节点高 4 倍
```

**影响：** 不影响算法正确性，只影响负载均衡度。

**缓解手段：**
- 设计上接受此边界，不增加复杂度去解决
- 实际部署时，业务层控制单个 Namespace 的数据量在合理范围内
- 如果必须支持超大 Namespace，可在存储引擎层内部二次分片（如按 Key 前缀拆成多个物理子目录）

### 4.6 与传统一致性哈希的区别

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
- 当前所有物理节点列表（含节点数 N）
- 所有 Namespace 列表（含路径、大小、旧归属节点）
- 新加入的节点 newNode
```

**Namespace 大小从何而来？**
每个 StorageNode 定期向 Coordinator 上报本机 Namespace 列表及大小：

```
Node_1 → Coordinator:
  [{path: "/user", size: 80GB, node: "Node_1"},
   {path: "/order", size: 200GB, node: "Node_1"},
   ...]

Node_2 → Coordinator:
  [{path: "/log", size: 400GB, node: "Node_2"},
   ...]
```

上报周期可配置（默认 60 秒），Coordinator 聚合后作为算法输入。这样 Coordinator 始终持有全局的 Namespace 大小视图，不需要自己扫描任何数据。

### 5.2 算法步骤

```
1. 汇总所有 Namespace，按 hash(path) 排序（确定、不改变）
   得到有序序列: [n1, n2, n3, ..., nK]

2. 计算目标负载: avg = Σ(size) / (N + 1)          ← 把新节点算进来

3. 贪心切分:
    当前节点 idx = 0
    遍历有序 Namespace 序列:
      累加 size 到当前节点
      如果 累计 >= avg，切分到下一个节点
    剩余 Namespace 全部分给最后一个节点

    得到新分配方案:
      Node_0: [n1, n2, n3]
      Node_1: [n4, n5]
      ...
      newNode: [...]

    注：贪心算法保证前 (N-1) 个节点负载接近 avg，
    最后一个节点拿到所有剩余 Namespace，可能远小于 avg。
    这是贪心切分的固有特性，不影响迁移计算的正确性。
    如果最后一个节点负载过小，重平衡阈值（4.4）会在后续周期触发调整。

4. 计算迁移:
   对每个 Namespace n:
     新归属 = 步骤 3 得到的分配结果
     旧归属 = 输入中的旧归属节点
     if 新归属 == newNode && 旧归属 ≠ newNode:
         标记 n 待迁移

5. 去重（父迁则子不迁）:
     遍历所有被标记的 n:
         if n 的父节点也被标记:
             跳过（父节点迁移时会带走整个子树）
         else:
             加入最终迁移列表

6. 输出:
     [{SrcNode, DstNode: newNode, Namespace, Size}, ...]
```

### 5.3 算法性质

| 性质 | 说明 |
|------|------|
| **确定性** | 相同输入 → 完全相同的输出。不依赖随机或概率 |
| **单调性** | Namespace 的 hash(path) 固定，在环上的相对位置永远不变 |
| **连续性** | Namespace 按哈希顺序连续分配，不会交错 |
| **最小迁移** | 只迁移换了 Owner 的 Namespace，不产生多余迁移 |

### 5.4 复杂度

- **时间复杂度**：O(K log K)，K = Namespace 数量（排序主导）
- **空间复杂度**：O(K + M)，K = Namespace 总数，M = 待迁移数
- 万级 Namespace 计算时间在毫秒级

---

## 6. 算法正确性验证

### 6.1 正向验证

```
输入：
  Namespaces（按 hash 排序）:
    n1=80GB, n2=200GB, n3=50GB, n4=400GB, n5=270GB, n6=90GB, n7=30GB
  当前节点: Node_1, Node_2, Node_3
  新节点: Node_4
  旧分配: Node_1={n1,n2,n3}, Node_2={n4,n5}, Node_3={n6,n7}

步骤1：计算目标
  avg = (80+200+50+400+270+90+30) / 4 = 280GB

步骤2：贪心切分（按哈希顺序累加）
  Node_1: n1(80) + n2(200) = 280GB ✅ 正好达到目标
  Node_2: n3(50) + n4(400) = 450GB ⚠️ 超了
  Node_3: n5(270) + n6(90) = 360GB
  Node_4: n7(30)

步骤3：对比新旧归属，找出迁到新节点 Node_4 的 Namespace
  n7(shop): 旧归属=Node_3, 新归属=Node_4  ← 需要迁移

  其他 Namespace 虽然也有归属变化（如 n5: Node_2→Node_3），
  但目标是"哪些 Namespace 需要迁到新节点"——只需找出新归属==Node_4 的那些。

步骤4：去重（n7 无父节点被标记，无需去重）

输出：
  [{Node_3 → Node_4, n7, 30GB}]
```

**注意：** 迁移计算只关心"哪些 Namespace 需要迁到新节点"。其他节点之间的归属变化（如 n5 从 Node_2 换到 Node_3）是贪心切分的副作用，不需要执行物理搬迁——因为实际上只有新节点缺数据，老节点之间不需要互相搬。

### 6.2 去重逻辑验证

```
场景：
  标记待迁移: ["user", "user/profile", "user/setting", "order"]

去重：
  "user" → 父节点未被标记 → 加入迁移列表
  "user/profile" → 父节点 "user" 也被标记 → 跳过
  "user/setting" → 父节点 "user" 也被标记 → 跳过
  "order" → 父节点未被标记 → 加入迁移列表

最终迁移列表：
  [{Node_1 → Node_4, "user"}]
  [{Node_2 → Node_4, "order"}]

物理结果：
  Node_1 搬走 /data/storage/user/          → 子树全跟着走
  Node_2 搬走 /data/storage/order/         → 单独搬
```

**验证通过。** 去重逻辑的成立不依赖任何哈希计算，只依赖 Namespace 路径的字符串前缀关系——这是确定性、零歧义的。

### 6.3 边界情况

| 场景 | 行为 | 正确性 |
|------|------|--------|
| 空集群 | 无 Namespace，无计算 | ✅ |
| 单 Namespace | 按 4.3 约束二，此部署无意义，不应使用本系统 | 🚫 禁止部署 |
| 单节点集群加节点 | 部分 Namespace 迁到新节点 | ✅ |
| 父 Namespace 单独迁 | 子 Namespace 自动跟随目录迁移 | ✅ |
| 子 Namespace 单独迁（父不动） | 只搬子目录 | ✅ |
| 多节点同时加入 | Coordinator(Raft) 串行处理 | ✅ |
| 加机器后负载不变（阈值内） | 不触发迁移 | ✅ |

## 7. 添加节点的完整流程

### 7.1 无停机方案（加只读锁）

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
    └── 写请求 → 引擎拒绝（返回"暂时只读"错误），由客户端重试或跳过
                  注：具体行为由 StorageEngine 实现决定，单线程引擎通常直接拒绝

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

### 7.2 时序图

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

## 8. 接口抽象

### 8.1 StorageEngine 接口

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

### 8.2 NamespaceNode 定义

```go
type NamespaceNode struct {
    Path     string           // 命名空间路径，如 "user/profile"
    Size     int64            // 数据量（字节）
    Children []*NamespaceNode // 子 Namespace
}
```

### 8.3 迁移任务定义

```go
type MigrationTask struct {
    SrcNode   *Node  // 源节点
    DstNode   *Node  // 目标节点（新加入的节点）
    Namespace string // 要迁移的 Namespace 路径
    Size      int64  // 数据量大小
}
```

---

## 9. 可插拔的存储引擎

### 9.1 分布式 Redis 引擎

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

### 9.2 分布式 LSM-Tree / RocksDB 引擎

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

### 9.3 分布式文件存储引擎

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

### 9.4 KV Cache（AI Agent 记忆存储）引擎

```
┌──────────────────────────────────────────────┐
│  KvCacheEngine (对应 quqiufeng/my_db)         │
│                                              │
│  ListNamespaces() → 扫描 /data/cache/*.bin   │
│   每个 .bin 文件对应一个 Namespace             │
│                                              │
│  LockNamespace("user") →                    │
│    ● 该 Namespace 的 bin 文件标记为只读        │
│    ● 单线程引擎直接拒绝写入请求                │
│                                              │
│  MigrateOut("user", dst) →                   │
│    ● 关闭 user.bin 的 mmap                    │
│    ● sendfile user.bin 到 dst                │
│                                              │
│  MigrateIn("user", src) →                    │
│    ● 接收 user.bin                           │
│    ● mmap 加载到内存                          │
│    ● 重建 hash 索引 + 排序数组                │
│                                              │
│  DeleteNamespace("user") →                    │
│    ● munmap + unlink user.bin               │
└──────────────────────────────────────────────┘
```

### 9.5 不同引擎的特点对比

| 引擎 | 迁移方式 | 热点数据 | 适用场景 |
|------|---------|---------|----------|
| Redis | dump + scp + load | 纯内存，快 | 缓存、会话、计数器 |
| RocksDB | SST rsync / sendfile | 磁盘顺序 IO | 时序、IoT、KV 存储 |
| 文件系统 | rsync / cp | 无额外开销 | 图片、对象、日志 |
| **kvCache** | **bin sendfile** | **mmap 零拷贝** | **AI Agent 记忆、层级知识库** |

---

## 10. 关键设计决策

### 10.1 为什么用 Namespace 做路由单位，而不是用 Key？

- 迁移精度 O(Namespace数) vs O(Key数)，差几个数量级
- 目录级搬迁，物理操作简单可靠
- 父→子、子→父的树形搜索天然支持
- 业务上 Namespace 是有意义的隔离边界

### 10.2 为什么用负载感知切分，而不是 vnode 统计均匀？

- vnode 只能保证哈希均匀，不能保证数据量均匀
- 10 个 Namespace 中有一个 500GB，vnode 再多也扛不住
- 实际负载切分能确保各节点磁盘使用率接近

### 10.3 为什么迁移时只加读锁，不停写？

- 加锁方案实现简单（一个 RWMutex）
- 读完全不受影响（缓存一直可用）
- 写暂停时间 ≈ 搬目录时间，秒级到分钟级
- 相比双写+WAL 方案，无数据不一致风险

### 10.4 为什么父迁子不迁？

- 目录结构天然保证包含关系
- 搬父目录时，子目录在物理上已经跟着走了
- 避免重复计算和重复搬迁

---

## 11. 部署结构

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

## 12. 总结

Newhash 方案以 **Namespace 目录树** 为核心抽象，在一致性哈希环上做 **负载感知切分**，实现了：

1. **精确到 Namespace 级别的迁移计算** — 加机器时 O(K log K) 算出要搬什么
2. **目录级物理搬迁** — 不需要逐 Key 扫描和重哈希
3. **可插拔的 StorageEngine 接口** — Redis、RocksDB、文件系统统一接入
4. **迁移期间缓存读可用** — 加读锁，读不受影响
5. **父迁子不迁** — 目录结构带来的天然优化
6. **确定性算法** — 相同输入永远产生相同输出，可验证、可测试
