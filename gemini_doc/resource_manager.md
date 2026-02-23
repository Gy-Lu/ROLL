# roll/distributed/scheduler/resource_manager.py

`ResourceManager`（资源管家）是 ROLL 框架中负责与底层物理集群（Ray Cluster）通信的组件。它负责统一调度 GPU/CPU 资源，并根据流水线的逻辑需求（如 TP/PP 配置）构建分布式放置拓扑。

## 核心名词

*   **Placement Group (放置组)**: Ray 提供的逻辑算力容器，用于锁定具体的计算资源。
*   **Device Mapping (设备映射)**: 将逻辑 Worker 序号映射到具体的 GPU 硬件索引（GPU Rank）。
*   **Bundle (资源束)**: 描述单次资源申请的具体规格（如：1 个 GPU + N 个 CPU）。

## 核心功能实现

### 1. 资源探测与管理 (`__init__`)
初始化时，`ResourceManager` 会通过 Ray 接口探测当前集群的总节点数和各节点的 GPU 资源。它会自动识别硬件类型（GPU/NPU），并预先创建一批 `PlacementGroup` 以供后续 Cluster 初始化。

### 2. 分布式放置拓扑构建 (`allocate_placement_group`)
根据配置中的 `device_mapping`，将大规模的 GPU 集群切分为多个逻辑 Worker 组。
- 如果提供了显式的 `device_mapping`，它会精确地为每个 Worker 分配指定索引的 GPU 资源。
- 如果没有映射，它会采用“尽量分散”的策略将 Worker 分布在不同节点上，以避免 CPU 内存溢出（OOM）。

### 3. 多机环境支持
实现了跨机器的分布式上下文维护。它会记录每个分配组的 `node_rank` 和 `gpu_rank`，这些信息会被注入到 Worker 的环境变量中，作为 NCCL 初始化（`init_process_group`）的输入。

## 资源分配逻辑流程图

```ascii
[ Pipeline Start ]
        |
        v
[ ResourceManager ] <--- (Query Ray Cluster Resource)
        |
        +--- [ Create Placement Groups ] --- (Lock CPU/GPU Slots)
        |
        +--- [ allocate_placement_group ] -- (Input: Device Mapping)
                    |
                    +---> [ Worker 0 (PG 0, Rank 0) ]
                    +---> [ Worker 1 (PG 1, Rank 1) ]
                    +---> [ ... ]
```

## 关键代码解析：nodes_maybe_used
在复杂的多租户或异构环境中，`ResourceManager` 会智能筛选符合条件的节点（如 GPU 数量满足要求的节点），并按 CPU 资源排序，从而优先利用空闲节点，减少计算干扰。

## 硬件适配实现
通过 `current_platform.ray_device_key`，`ResourceManager` 可以无缝切换支持 NVIDIA GPU (GPU) 和 华为昇腾 (NPU) 资源调度，向上层业务逻辑屏蔽硬件差异。
