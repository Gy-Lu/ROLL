# examples/start_rlvr_pipeline.py

`start_rlvr_pipeline.py` 是 ROLL 框架的“点火钥匙”。它通过集成 Hydra 配置管理系统和 Ray 分布式初始化逻辑，将静态配置转化为动态运行的训练集群。

## 核心名词

*   **Hydra (配置管理)**: 一种强大的分层配置管理工具，支持 YAML 合并与命令行覆盖。
*   **OmegaConf (配置解析)**: 支持类型检查和变量插值的配置解析库。
*   **RLVRConfig (配置类)**: 强类型的配置类定义，确保配置文件的每个参数都符合预期的类型。

## 核心功能实现

### 1. 配置加载与解析
脚本利用 `hydra` 从 `examples/config`（或指定路径）加载主 YAML 文件，并递归合并子配置文件（如 `deepspeed_zero3.yaml`, `vllm_config.yaml`）。
- **类型校验**: 使用 `dacite.from_dict` 将字典形式的配置强制转换为 `RLVRConfig` 对象。

### 2. Ray 集群点火 (`init`)
调用 `roll.distributed.scheduler.initialize.init()` 方法。此方法不仅初始化 Ray，还会处理 Ray 的 Namespace 和各种环境变量，确保后续的 `Cluster` 能在一个干净的分布式上下文中运行。

### 3. 流水线实例化与启动
- **`RLVRPipeline` 实例化**: 将解析后的强类型配置对象传入流水线。
- **`pipeline.run()`**: 进入流水线的主执行循环。

## 启动流程图 (Call Flow)

```ascii
[ User CLI ] -- (python start_rlvr_pipeline.py --config_name=ppo_config)
    |
    v
[ Hydra / OmegaConf ] -- (Merge YAML files)
    |
    v
[ dacite.from_dict ] -- (Type-safe config object)
    |
    v
[ roll.init() ] -- (Ray Start & Storage setup)
    |
    v
[ RLVRPipeline.run() ] -- (Main Loop Starts)
```

## 关键代码解析：main() 函数
```python
def main():
    # 1. 解析命令行参数获取配置文件名
    args = parser.parse_args()
    # 2. Hydra 加载
    initialize(config_path=args.config_path)
    cfg = compose(config_name=args.config_name)
    # 3. 数据校验与转换
    ppo_config: RLVRConfig = from_dict(data_class=RLVRConfig, ...)
    # 4. 初始化分布式环境并运行
    init()
    pipeline = RLVRPipeline(pipeline_config=ppo_config)
    pipeline.run()
```

## 点火钥匙的作用
这个脚本是连接“用户需求”（YAML 描述）与“集群算力”（Ray Workers）的桥梁。它通过强类型的配置校验，极大地降低了由于配置项错误导致的大规模分布式任务崩坏的风险。
