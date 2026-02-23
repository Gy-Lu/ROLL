# roll/pipeline/base_pipeline.py

`BasePipeline` 是 ROLL 框架中所有训练/推理流水线（Pipeline）的基类。它定义了分布式训练任务的核心生命周期管理，包括资源初始化、状态维护、模型更新同步以及 Checkpoint 的保存与清理。

## 核心名词

*   **Pipeline (流水线)**: 整个训练任务的编排者，负责协调不同 Cluster 之间的协作。
*   **WorkerState (工作状态)**: 记录当前流水线的进度（如 global_step）和日志历史。
*   **ModelUpdateGroup (权重同步组)**: 定义了模型权重从“生产集群”（如 ActorTrain）同步到“消费集群”（如 ActorInfer）的策略。

## 核心功能实现

### 1. 生命周期管理 (`__init__`)
初始化资源管理器（ResourceManager）、状态机、Checkpoint 管理器和追踪器（Tracker）。支持从 Checkpoint 恢复（Resume）。

### 2. 权重同步控制 (`set_model_update_pair` & `model_update`)
强化学习中通常有训练节点和采样节点。`BasePipeline` 允许将这些节点配对，并在每个 step 调用 `model_update` 将最新的训练权重下发到采样集群，保证采样策略的实时更新。

### 3. Checkpoint 机制 (`do_checkpoint`)
*   **分布式保存**: 协调各个 Cluster 同时执行保存操作。
*   **分层存储**: 在保存各 Cluster 内部状态的同时，保存 Pipeline 自身的元数据（如 RNG 状态、当前步数）。
*   **自动清理**: 通过 `_cleanup_old_checkpoints` 维护 `max_ckpt_to_keep`，自动删除过旧的检查点以节省存储空间。

### 4. 并行模型下载 (`download_models`)
利用 Ray 的分布式特性，在所有计算节点上并行下载所需模型，极大地缩短了大规模集群启动时的等待时间。

## 调用关系图

```ascii
+-----------------------+
|    BasePipeline       |
+-----------+-----------+
            |
            +--- initialize() ---> ResourceManager
            |
            +--- run() [Abstract]
            |
            +--- model_update() ---> ModelUpdateGroup ---> Cluster (Workers)
            |
            +--- do_checkpoint() ---> CheckpointManager & Cluster.do_checkpoint()
            |
            +--- _cleanup_old_checkpoints()
```

## 关键代码片段逻辑

```python
def do_checkpoint(self, global_step, is_last_step=None):
    # 1. 收集各 Cluster 状态
    for cluster in self.checkpoint_clusters:
        ckpt_metrics_refss.append(cluster.do_checkpoint(...))
    # 2. 保存 Pipeline 自身状态
    self.state.save_to_json(save_dir=save_dir, tag="pipeline")
    # 3. 异步上传/持久化
    self.checkpoint_manager.upload(ckpt_id=ckpt_id, local_state_path=pipeline_save_dir)
```
