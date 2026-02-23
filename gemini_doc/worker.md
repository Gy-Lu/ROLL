# roll/distributed/executor/worker.py

`Worker` 类是 ROLL 框架中所有 Ray Actor 的基类。它是执行具体计算（如训练或推理策略）的最小容器，负责底层的硬件管理、环境初始化以及具体的计算逻辑调用。

## 核心名词

*   **Worker (工作节点)**: 封装了计算逻辑的 Ray 远端对象。
*   **Strategy (策略)**: 具体的执行后端实现（如 vLLM, SGLang, DeepSpeed, Megatron）。
*   **RankInfo**: 描述当前 Worker 在分布式拓扑中的位置（DP/TP/PP/CP rank）。

## 核心功能实现

### 1. 环境初始化 (`__init__`)
*   **分布式环境建立**: 每个 Worker 在被初始化时，会自动参与到一个分布式计算组中。
*   **Master 地址同步**: Rank 0 Worker 负责提供 `MASTER_ADDR` 和 `MASTER_PORT`，并将其同步到 `SharedStorage` 中供其他 Worker 发现。

### 2. 模型状态管理
*   **`load_states`**: 将模型权重加载到显存中。
*   **`offload_states`**: 将权重卸载到 CPU 内存或内存中，用于显存敏感场景（如采样后立即训练）。
*   **`process_weights_after_loading`**: 权重加载后的后处理逻辑（如显存优化、量化调整）。

### 3. 方法分发代理
`Worker` 类通常持有一个 `strategy` 实例。它通过将复杂的策略调用（如 `strategy.model_update`）暴露为远程可调用的接口，使得 `Cluster` 能够统一调度。

### 4. 性能监控 (`get_metrics`)
每个 Worker 负责收集其所在节点的性能指标（如 GPU 利用率、吞吐量），并通过 `DataProto` 反馈给调度中心。

## 执行引擎：Strategy 模式
`Worker` 本身不实现具体的模型逻辑，而是作为“外壳”存在。其内部的 `strategy` 决定了系统行为：
- 在 `ActorTrain` 集群中，`strategy` 是 `DeepSpeedStrategy` 或 `MegatronStrategy`。
- 在 `ActorInfer` 集群中，`strategy` 是 `vLLMStrategy` 或 `SGLangStrategy`。

## 代码结构示意图

```ascii
+-----------------------------+
|           Worker            |
| (Ray Actor / Context)       |
+--------------+--------------+
               |
               v
+--------------+--------------+
|          Strategy           |
| (Implementation Layer)      |
+--------+-----------+--------+
         |           |
         v           v
   [ Training ]  [ Inference ]
   (Backward)    (Generate)
```

## 关键装饰器：@register
使用 `@register` 装饰的方法可以被 `Cluster` 自动识别并进行分布式路由控制。它指明了该方法的调度模式（如 `ONE_TO_ALL`）。
