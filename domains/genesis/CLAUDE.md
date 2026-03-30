# Genesis 模块 - 机器人控制仿真

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **genesis/**

> **更新时间**: 2026-03-30 11:14:03
>
> **模块类型**: Robotics Simulation
>
> **主要语言**: Python 3.12+, PyTorch

---

## 模块职责

Genesis 模块实现了基于 Genesis 仿真平台的机器人控制任务，主要用于：

- **四足机器人控制**: Unitree Go2 机器人的步态控制
- **强化学习训练**: 使用 PPO 算法训练策略
- **运动技能学习**: 行走、跳跃等运动技能
- **物理仿真**: 高保真物理交互

---

## 入口与启动

### 环境设置

**1. 检查 CUDA 版本**:
```bash
nvidia-smi
```

**2. 安装依赖** (参见 `domains/genesis/README.md`):
```bash
# 安装 PyTorch (根据 CUDA 版本选择)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130

# 安装 Genesis 和其他依赖
pip install git+https://github.com/Genesis-Embodied-AI/Genesis.git
pip install rsl-rl-lib==2.2.4 tensorboard==2.20.0
```

### 评估命令

```bash
# Go2 步态控制评估
python -m domains.harness \
    --domain genesis_go2walking \
    --run_id initial_genesis_go2walking_0 \
    --num_samples 1 \
    --num_workers 1

# 生成报告
python -m domains.report \
    --domain genesis_go2walking \
    --dname ./outputs/initial_genesis_go2walking_0
```

---

## 对外接口

### 环境创建

```python
from domains.genesis.environments import make_genesis_env

# 创建 Go2 步态环境
env = make_genesis_env(
    env_id="Go2WalkingCommand-v0",
    num_envs=4096,  # 并行环境数
    device="cuda:0"
)

# RL 训练循环
obs, info = env.reset()
for _ in range(max_steps):
    actions = policy(obs)
    obs, reward, done, truncated, info = env.step(actions)
```

### 评估接口

```python
from domains.genesis.evaluator import GenesisEvaluator

evaluator = GenesisEvaluator(
    env_id="Go2WalkingCommand-v0",
    num_eval_episodes=5
)

# 运行评估
fitness_scores = evaluator.evaluate(agent)
```

---

## 关键依赖与配置

### 依赖包

**requirements.txt**:
```
# 注意: torch 和 torchvision 在 Dockerfile 中单独安装
rsl-rl-lib==2.2.4       # RL 算法库
tensorboard==2.20.0     # 训练监控
git+https://github.com/Genesis-Embodied-AI/Genesis.git
```

### 支持的环境

| 环境ID | 描述 | 文件 |
|-------|------|------|
| `Go2WalkingCommand-v0` | 步态控制 | `environments/Go2WalkingCommand.py` |
| `Go2HoppingCommand-v0` | 跳跃控制 | `environments/Go2HoppingCommand.py` |

### 配置文件

**domains/genesis/config/config.yaml**:
```yaml
defaults:
  - genesis_task: go2walking

genesis_task:
  name: go2walking
  environment:
    id: Go2WalkingCommand-v0
    num_envs: 4096
    device: cuda:0
  training:
    num_epochs: 100
    learning_rate: 0.001
    batch_size: 4096
```

---

## 数据模型

### 观察空间

```python
observation = {
    "joint_positions": np.array,      # 关节位置 (12维)
    "joint_velocities": np.array,     # 关节速度 (12维)
    "base_lin_vel": np.array,         # 基座线速度 (3维)
    "base_ang_vel": np.array,         # 基座角速度 (3维)
    "projected_gravity": np.array,    # 投影重力 (3维)
    "commands": np.array,             # 速度命令 (3维)
}
```

### 动作空间

```python
action = {
    "joint_positions_target": np.array,  # 目标关节位置 (12维)
}
```

### 奖励函数

**domains/genesis/reward/default_walking_function.py**:
```python
def compute_walking_reward(obs, actions):
    """
    计算步态控制奖励

    组成部分:
    1. 速度跟踪奖励: 跟随目标速度
    2. 稳定性奖励: 保持机器人平衡
    3. 能量惩罚: 最小化关节扭矩
    4. 关节限制: 避免超出物理限制
    """
    reward = (
        velocity_reward * 1.0 +
        stability_reward * 0.5 +
        energy_penalty * -0.01 +
        joint_limit_penalty * -0.1
    )
    return reward
```

---

## 训练与评估

### RL 训练

**文件**: `domains/genesis/genesis_train/rl_trainer.py`

```python
class RLTrainer:
    def train(self, num_epochs):
        for epoch in range(num_epochs):
            for batch in dataloader:
                # 收集经验
                obs, actions, rewards = self.collect_rollouts()

                # 更新策略
                policy_loss = self.compute_policy_loss(obs, actions, rewards)
                self.update_policy(policy_loss)

            # 保存检查点
            self.save_checkpoint(epoch)
```

### RL 评估

**文件**: `domains/genesis/genesis_eval/rl_eval.py`

```python
def evaluate_agent(agent, env, num_episodes=5):
    """
    评估智能体性能

    Args:
        agent: 训练好的智能体
        env: 评估环境
        num_episodes: 评估回合数

    Returns:
        fitness_scores: 适应度分数列表
    """
    fitness_scores = []
    for episode in range(num_episodes):
        obs, info = env.reset()
        episode_reward = 0
        done = False
        while not done:
            actions = agent.predict(obs)
            obs, reward, done, truncated, info = env.step(actions)
            episode_reward += reward
        fitness_scores.append(episode_reward)
    return fitness_scores
```

---

## 输出结构

### 评估输出

```
outputs/
├── {run_id}/
│   └── genesis_go2walking/
│       ├── episode_0/
│       │   ├── chat_history.md          # Task智能体对话
│       │   ├── rl_eval_0/               # RL评估结果
│       │   │   ├── eval_log.json        # 评估日志
│       │   │   └── eval_100.mp4         # 评估视频
│       │   └── rl_train_0/              # RL训练结果
│       │       ├── model_0.pt           # 初始模型
│       │       ├── model_100.pt         # 100步模型
│       │       ├── events.out.tfevents  # TensorBoard日志
│       │       └── train_log.json       # 训练日志
```

### 适应度分数

**评估指标**: `average_fitness`

```python
# 适应度范围: 0-1 (1=最佳)
fitness_score = (
    velocity_tracking * 0.6 +
    stability * 0.3 +
    energy_efficiency * 0.1
)
```

---

## 测试与质量

### 评估设置

- **总仿真时间**: 20秒
- **回合时长**: 4秒 (200步 @ dt=0.02s)
- **速度命令**: 每回合随机采样
- **并行环境**: 4096个环境异步评估

### 正常行为

每个环境在20秒内完成5个回合 (20s ÷ 4s = 5)，产生5个数据点。

### 提前终止

如果机器人倾倒（roll/pitch > 10度），回合立即结束并开始新回合。因此可能观察到多于5个数据点。

---

## 常见问题 (FAQ)

### Q1: 如何选择 GPU？

**A**: 使用 `domains/genesis/gpu_selector.py`:

```python
from domains.genesis.gpu_selector import get_best_gpu

device = get_best_gpu()
print(f"Using GPU: {device}")
```

### Q2: 如何可视化训练？

**A**: 使用 TensorBoard:

```bash
# 启动 TensorBoard
tensorboard --logdir outputs/{run_id}/genesis_go2walking/episode_0/rl_train_0

# 在浏览器中查看
# http://localhost:6006
```

### Q3: 如何调整并行环境数？

**A**: 修改配置或环境创建参数：

```python
env = make_genesis_env(
    env_id="Go2WalkingCommand-v0",
    num_envs=2048,  # 减少到2048（显存不足时）
    device="cuda:0"
)
```

### Q4: 如何查看评估视频？

**A**: 检查生成的 MP4 文件：

```bash
# 播放评估视频
vlc outputs/{run_id}/genesis_go2walking/episode_0/rl_eval_0/eval_100.mp4

# 或使用 ffplay
ffplay outputs/{run_id}/genesis_go2walking/episode_0/rl_eval_0/eval_100.mp4
```

---

## 相关文件清单

### 核心文件
- `domains/genesis/eval.py` - 评估逻辑
- `domains/genesis/evaluator.py` - 评估器管理
- `domains/genesis/genesis_eval/rl_eval.py` - RL评估实现
- `domains/genesis/genesis_train/rl_trainer.py` - RL训练实现
- `domains/genesis/config/config.yaml` - 配置文件

### 环境实现
- `domains/genesis/environments/genesis_env.py` - 基础环境
- `domains/genesis/environments/Go2WalkingCommand.py` - 步态控制
- `domains/genesis/environments/Go2HoppingCommand.py` - 跳跃控制

### 工具
- `domains/genesis/genesis_utils.py` - 工具函数
- `domains/genesis/gpu_selector.py` - GPU选择
- `domains/genesis/reward/` - 奖励函数

### 基线
- `baselines/genesis_go2walking/reward_functions/` - 基线奖励函数

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 Genesis 模块文档
- ✅ 记录环境接口和配置
- ✅ 文档化训练和评估流程
- 📝 待添加：自定义奖励函数指南
- 📝 待添加：新任务开发指南

---

*此模块文档由 PAI Architecture Agent 自动生成*
