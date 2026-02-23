# roll/distributed/executor/cluster.py

`Cluster` 类是 ROLL 框架的分布式执行核心抽象。它管理一组功能相同的 Ray Actor（`Worker`），并提供类似于单机操作的分布式方法代理。

## 核心名词

*   **Cluster (集群/节点组)**: 一个逻辑上的节点集合（如 `ActorTrain`），负责执行特定任务。
*   **Ray Actor (Worker)**: 分布在不同物理机器上的独立执行单元。
*   **Placement Group (放置组)**: Ray 的资源调度单元，用于精确控制 Actor 在 GPU 上的位置。
*   **Magic Method Binding (魔法绑定)**: `Cluster` 会自动探测 `Worker` 的方法，并将这些方法映射为 `Cluster` 的方法。

## 引擎工作原理：分布式方法分发

`Cluster` 最强大的特性是它的“魔法绑定”机制。当你在 `Cluster` 上调用一个方法时（例如 `actor_train.train_step(data)`），它会通过以下两种模式之一分发任务：

1.  **ONE_TO_ALL (广播)**: 将相同的数据分发给所有 Worker。
2.  **DP_MP_COMPUTE (切分/数据并行)**: 将输入数据（DataProto）均匀切分并分发给对应的数据并行（DP）节点。

### 核心方法分发实现 (`_bind_worker_method`)
通过探测 `Worker` 类中被 `@register` 装饰的方法，`Cluster` 动态生成代理函数。这些代理函数封装了 `ray.remote` 调用。

## 资源调度实现 (`_create_workers`)
1.  **分配资源**: 使用 `ResourceManager` 获取 `PlacementGroup`。
2.  **环境变量注入**: 为每个 Worker 设置 `RANK`, `LOCAL_RANK`, `MASTER_ADDR`, `MASTER_PORT` 等，支持 PyTorch 分布式环境（NCCL）。
3.  **异构调度**: 支持在不同的节点配置中灵活部署（CPU-only, Single GPU, Multi-GPU）。

## 分布式拓扑示意图

```ascii
      +------------------------+
      |        Cluster         |
      +-----------+------------+
                  | (Magic Proxy)
                  v
   +--------------+--------------+
   |              |              |
   v              v              v
+--------+     +--------+     +--------+
| Worker |     | Worker |     | Worker | (Ray Actor)
+--------+     +--------+     +--------+
   |              |              |
   +--------------+--------------+
                  | (Ray GCS / NCCL)
                  v
           [ GPU / NPU Nodes ]
```

## 关键代码逻辑：execute_all_async
`Cluster` 的 `execute_all_async` 方法实现了高效的任务分发：
- 如果参数是列表且长度等于 Worker 数量，则按位置分发。
- 否则，将同一参数广播给所有 Worker。
