# roll/distributed/scheduler/generate_scheduler.py

`GenerateScheduler` 是 ROLL 框架中采样逻辑（Rollout）的“指挥中心”。它实现了在大规模 GPU 集群上，高效、异步地从大模型中获取训练样本的复杂调度。

## 核心名词

*   **LoadBalancer (负载均衡器)**: 控制每个 DP（数据并行）节点上正在运行的请求数，避免显存溢出。
*   **ReplayBuffer (经验池/回放池)**: 管理正在运行和已完成的 Prompt，并提供事务接口供调度器获取数据。
*   **DynamicSamplingScheduler (动态采样调度器)**: 集成了异步采样、奖励计算、并发控制的业务逻辑层。
*   **RolloutContext (采样上下文)**: 封装了采样生命周期的辅助类，隐藏了底层负载均衡和缓存的细节。

## 核心功能实现

### 1. 异步采样流水线 (`sending_request`)
`DynamicSamplingScheduler` 使用异步协程（`asyncio`）维护一个持续的采样任务循环：
1.  **Poll (轮询)**: 从 `ReplayBuffer` 中获取新的 Prompt ID。
2.  **Acquire (获取资源)**: 向 `LoadBalancer` 申请执行槽位（Lease）。
3.  **Generate (生成)**: 异步调用 `ActorInfer` 集群生成回复文本。
4.  **Reward (奖励)**: 将生成的文本异步发送到 `Reward Clusters` 进行打分。
5.  **Commit (提交)**: 将带有分数的数据存回 `ReplayBuffer`。

### 2. 采样质量控制 (Filtering)
调度器支持多种过滤策略（如 `query_filter`），在将样本返回给训练引擎之前，剔除不符合要求的（如长度过短、逻辑完全错误的）低质量样本。

### 3. 动态配置更新
支持在运行过程中动态调整采样参数（如 `temperature`, `top_p`, `num_return_sequences`），以适应课程学习（Curriculum Learning）等高级训练策略。

## 调度拓扑示意图

```ascii
      [ ReplayBuffer ] <---+ (Store Finished)
             |             |
             v (Poll)      |
      [ LoadBalancer ] ----+ (Lease Slot)
             |
             v (Async Dispatch)
   +---------+----------+
   |                    |
   v                    v
[ ActorInfer Cluster ] [ Reward Cluster ]
(Ray remote calls)     (Ray remote calls)
```

## 执行逻辑：ReplayBuffer 的 GC 机制
为了防止异步采样产生过时的样本（Off-policy 过大），`ReplayBuffer` 会执行垃圾回收（GC），强制清理掉训练步数落后太多的请求。这保证了模型训练所使用的数据始终处于合理的策略分布内。

## 关键类：LoadBalancer.Lease
这是一个异步上下文管理器，通过 `lock` 机制确保在多轮对话或复杂 Agentic 采样中，同一个 Prompt 的多个回复始终在同一个计算节点上处理，从而利用 KV Cache 或减少数据移动开销。
