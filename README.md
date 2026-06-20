# newhash

加机器时，自动算出哪些数据要迁到哪台机器，高效维持集群扩容平稳。

## 核心思想

数据按 **Namespace 目录树** 组织，用一致性哈希环做路由，用实际数据量做环的切分。

加机器时，哈希环自动算出哪些 Namespace 换了主人，直接下发目录级搬迁指令。不需要逐 Key 扫描，不需要全网重哈希。

```
                 哈希环（按 Namespace 实际大小切分）
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      ┌──────┐   ┌──────┐   ┌──────┐
      │Node_1│   │Node_2│   │Node_3│
      │ user │   │ order│   │ log  │
      │ shop │   │ pay  │   │backup│
      └──────┘   └──────┘   └──────┘
               +
         新节点 Node_4
               ↓
    CalcMigration() 算出：
      user   → Node_1 → Node_4
      backup → Node_3 → Node_4
```

## 核心算法

```
加机器 → 对每个 Namespace 算哈希 → 环上找新主人
       → 哪些换了主人 → 去重（父迁则子不迁）
       → 输出迁移列表 [{SrcNode, DstNode, Namespace}]
```

复杂度 O(Namespace 数量)，万级秒算。

## 架构

```
┌─────────────────────────────┐
│      Coordinator (Raft)     │  ← 控制面，不感知底层存储
│  HashRing.CalcMigration()   │
└────────────────┬────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌──────┐    ┌──────┐    ┌──────┐
│ Node │    │ Node │    │ Node │  ← 数据面，可插拔引擎
│Engine│    │Engine│    │Engine│
└──────┘    └──────┘    └──────┘
```

## 迁移流程（不停机）

```
1. 拓扑计算 → 算出哪些 Namespace 要搬
2. 加只读锁 → 源节点该 Namespace 停写，读仍正常服务
3. 物理搬迁 → 目录级传输（sendfile / rsync / dump）
4. 原子切换 → etcd 事务更新元数据
5. 清理     → 源节点删旧目录
```

迁移期间，读一直可用（缓存）。写短暂暂停，其他 Namespace 完全不受影响。

## 可插拔引擎

| 引擎 | 迁移方式 | 场景 |
|------|---------|------|
| Redis | dump + scp + load | 缓存、会话 |
| RocksDB | SST sendfile | 时序、KV 存储 |
| 文件系统 | rsync / cp | 图片、日志 |

## 设计文档

详细架构设计见 [docs/design.md](docs/design.md)。

## 许可证

Apache License 2.0，详见 [LICENSE](./LICENSE)。
