# roll/distributed/scheduler/protocol.py

`DataProto` 是 ROLL 框架内标准化的数据交换协议，它定义了分布式环境下数据传输的格式和接口。

## 核心名词

*   **DataProto (数据原语)**: 系统间流动的数据包。
*   **TensorDict (张量字典)**: `batch` 的核心容器，支持批量处理张量。
*   **non_tensor_batch (非张量批数据)**: 存储不可通过张量化的数据（如原始文本、UUID、UUID 列表等）。
*   **meta_info (元数据)**: 存储控制信息（如 global_step、generation_config、metrics）。

## 核心功能实现

### 1. 数据容器设计
*   **`batch`**: 基于 `tensordict` 实现，允许像操作单个张量一样操作整个字典（支持 `.to(device)`, `.clone()`, `.slice()`）。
*   **`non_tensor_batch`**: 使用 `numpy` 的 object 数组存储。
*   **`meta_info`**: 存储模型超参数或统计指标。

### 2. 分布式数据操作
*   **`materialize_concat` (合并)**: 从 Ray 的分布式对象引用（`ObjectRef`）中获取数据并合并成一个完整的 `DataProto`。
*   **`chunk` (切分)**: 将大的数据包按照 Worker 数量均匀切分成小包，用于数据并行分发。
*   **`union` (求并)**: 合并两个 `DataProto` 的字段，常用于将采样结果（Responses）与原始提示词（Prompts）组合。

### 3. 数据生命周期
*   **序列化 (`__getstate__` / `__setstate__`)**: 专门优化了跨节点传输时的内存共享和序列化开销，确保在 Ray 节点间传递时的高效性。
*   **内存平衡**: 集成了 `batch_balance`（来自 `utils`），确保每个 GPU 处理的 Token 数量均衡。

## 数据包结构示意图 (Table)

| 字段 | 类型 | 典型内容 | 用途 |
| :--- | :--- | :--- | :--- |
| `batch` | `TensorDict` | `input_ids`, `attention_mask` | 训练/推理的张量数据 |
| `non_tensor_batch` | `Dict[str, np.ndarray]` | `raw_prompts`, `raw_responses` | 记录、日志和非张量处理 |
| `meta_info` | `Dict[str, Any]` | `global_step`, `metrics`, `config` | 控制信号、性能监控 |

## 名词解释：collate_fn
`DataProto` 提供了一个自定义的 `collate_fn`。与常规 PyTorch collate 不同，它能同时处理张量对齐、非张量聚合以及元数据继承，是衔接 `DataLoader` 与 `Cluster` 执行的关键。
