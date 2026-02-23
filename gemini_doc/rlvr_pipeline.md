# roll/pipeline/rlvr/rlvr_pipeline.py

`RLVRPipeline` 是 ROLL 框架中处理强化学习可变奖励（Reinforcement Learning for Variable Rewards）的核心引擎。它实现了完整的强化学习训练循环，包括分布式采样、多种奖励模型的集成计算、优势估计以及模型更新。

## 核心引擎功能

`RLVRPipeline` 的 `run` 方法是整个系统的动力核心，执行 PPO/GRPO 等算法的主循环。

### 1. 引擎执行流程 (Run Loop)

每个 step 的典型执行路径如下：

1.  **准备阶段**: 
    *   通过 `model_update` 同步 Actor 权重。
    *   卸载或加载模型状态（针对显存优化）。
2.  **采样阶段 (Rollout)**:
    *   调用 `generate_schedulers` 从环境中采集样本。
    *   通过 `ActorInfer` 集群进行模型推理生成文本。
3.  **奖励计算 (Reward Calculation)**:
    *   将生成的样本分发到不同的 `Reward Clusters`。
    *   收集各种奖励信号（Rule-based, Model-based）。
4.  **后处理与算法计算**:
    *   `reward_postprocess`: 奖励标准化（Normalization）和裁剪（Clipping）。
    *   `compute_token_reward`: 将序列级奖励扩展到 Token 级，并计算 KL 惩罚。
    *   `compute_advantage`: 计算优势函数（GAE, GRPO, Reinforce++ 等）。
5.  **训练阶段 (Train Step)**:
    *   对数据进行负载均衡重排（`batch_balance`）。
    *   调用 `ActorTrain` 执行梯度反向传播和参数更新。

## 名词解释

*   **Domain (领域)**: 支持多领域混合训练（如数学、代码、通用逻辑）。每个 Domain 可以有不同的采样比例和奖励策略。
*   **DynamicSamplingScheduler**: 动态采样调度器，负责在 Ray 集群中异步、高效地获取训练样本。
*   **Advantage Estimator (优势估计器)**: 算法的核心，如 GAE（广义优势估计）用于 PPO，或简单的 Reinforce 返回值。

## 执行序列示意 (ASCII Art)

```ascii
[ Step N ]
    |
    v
+------------------+     +------------------+
| Model Sync       | --> | ActorInfer Init  |
+------------------+     +------------------+
    |                             |
    v                             v
+------------------+     +------------------+
| Rollout / Gen    | <---| vLLM / SGLang    |
+------------------+     +------------------+
    |                             |
    v                             v
+------------------+     +------------------+
| Reward Scorer    | <---| Multiple Rewards |
+------------------+     +------------------+
    |
    v
+------------------+     +------------------+
| Adv Estimation   | --> | GAE / GRPO / PP  |
+------------------+     +------------------+
    |
    v
+------------------+     +------------------+
| Train Step       | --> | DeepSpeed/Mcore  |
+------------------+     +------------------+
```

## 核心函数逻辑：compute_advantage

该函数决定了强化学习的策略改进方向：
- 如果是 `gae`，它结合模型 `values` 和 `token_level_rewards` 计算 TD 误差。
- 如果是 `grpo` 或 `reinforce`，它通过组内对比或序列累计回报来估计优势。
