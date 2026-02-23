# roll/utils/functionals.py

`functionals.py` 是 ROLL 框架中强化学习算法核心数学逻辑的实现库。它封装了所有关键的张量计算函数，如优势估计（Advantage Estimation）、奖励后处理和数据重排逻辑。

## 核心名词

*   **Advantage (优势)**: 策略梯度算法（如 PPO）中的核心概念，代表了某个动作相对于平均水平的“好坏”。
*   **GAE (Generalized Advantage Estimation)**: 广义优势估计，通过引入 $\lambda$ 参数在偏差和方差之间进行权衡。
*   **KL Divergence (KL 散度)**: 用于衡量训练模型与参考模型（Ref Model）之间的差异，作为惩罚项防止模型过度偏离。
*   **Whiten (白化)**: 对数据进行均值移除和标准差归一化，通常用于稳定优势值的分布。

## 核心功能实现

### 1. 优势估计算法 (`compute_advantage`)
实现了多种主流算法的优势计算：
- **GAE**: 使用 `compute_gae_advantage_return`，结合 `values` 预估值计算 TD-Error。
- **Reinforce**: 简单的蒙特卡洛回报累计。
- **GRPO (Group Relative Policy Optimization)**: 专门支持 DeepSeek 所提倡的组内相对优势计算。

### 2. 奖励逻辑处理 (`compute_token_reward`)
- **Token 级扩展**: 将序列最后的奖励值（如通过/失败）分发到所有 Token，或仅分发给 EOS Token。
- **KL 惩罚注入**: 在计算 Token 级 Reward 时，自动减去 KL 散度惩罚项。

### 3. 数据重排与负载均衡 (`batch_balance`)
在大规模训练中，不同样本的长度差异巨大（如 Math 题目推理过程的长短）。`batch_balance` 使用 **Karmarkar-Karp 算法** 对 Batch 内的数据进行重新分区（Partitioning），确保每个数据并行（DP）秩（Rank）处理的总 Token 数尽可能接近，从而最大化 GPU 利用率并减少 Bubble。

### 4. 采样后处理 (`postprocess_generate`)
负责将采样输出（左填充 Left Pad）转换为训练所需的格式（右填充 Right Pad），并生成对应的 `attention_mask` 和 `response_mask`。

## 数学逻辑对比 (Table)

| 算法模式 | 是否需要 Critic | 优势来源 | 典型用途 |
| :--- | :--- | :--- | :--- |
| PPO (GAE) | 是 | $Reward + \gamma V(s') - V(s)$ | 通用 RL 训练 |
| Reinforce++ | 否 | 累计打分回报 | 显存敏感型训练 |
| GRPO | 否 | $\frac{Score - Group\_Mean}{Group\_Std}$ | 逻辑推理/数学训练 |

## 关键代码逻辑：agg_loss
支持多种 Loss 聚合模式（`token-mean`, `seq-mean`），这是由于不同 RL 算法对样本权重理解不同所致。例如 GRPO 论文中提到的序列级平均与 Token 级求和的组合。
