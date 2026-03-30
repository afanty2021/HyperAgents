# Balrog 模块 - 游戏强化学习环境

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **balrog/**

> **更新时间**: 2026-03-30 11:14:03
>
> **模块类型**: Reinforcement Learning Environment
>
> **主要语言**: Python 3.12+

---

## 模块职责

Balrog 模块实现了多个游戏强化学习环境，用于训练和评估 AI 智能体：

- **NetHack Learning Environment (NLE)**: 经典 roguelike 游戏
- **MiniHack**: NLE 的自定义任务集合
- **BabyAI**: 语言导航任务
- **Baba Is AI**: 规则推理游戏
- **Crafter**: 开放世界生存游戏
- **TextWorld**: 文本冒险游戏

每个环境都提供统一的 Gym/Gymnasium 接口，支持标准的 RL 循环。

---

## 入口与启动

### 评估命令

```bash
# MiniHack 评估
python -m domains.harness \
    --domain balrog_minihack \
    --run_id test_minihack \
    --num_samples 10 \
    --num_workers 4

# BabyAI Text 评估
python -m domains.harness \
    --domain balrog_babyai_text \
    --run_id test_babyai \
    --num_samples 10

# Crafter 评估
python -m domains.harness \
    --domain balrog_crafter \
    --run_id test_crafter \
    --num_samples 10
```

### 配置文件

**domains/balrog/config/config.yaml**:
```yaml
defaults:
  - balrog_task: minihack

balrog_task:
  name: minihack
  environment:
    id: MiniHack-CorridorBattle-v0
  evaluation:
    num_episodes: 10
    max_steps: 100
```

---

## 对外接口

### 环境创建

```python
from domains.balrog.environments import make_env

# 创建环境
env = make_env(
    env_id="MiniHack-CorridorBattle-v0",
    seed=42
)

# 标准RL循环
obs, info = env.reset()
done = False
while not done:
    action = agent.predict(obs)
    obs, reward, done, truncated, info = env.step(action)
```

### 评估器接口

```python
from domains.balrog.evaluator import EvaluatorManager

# 初始化评估器
evaluator = EvaluatorManager(
    config=config,
    output_dir="./outputs"
)

# 运行评估
results = evaluator.evaluate(agents)
```

---

## 关键依赖与配置

### 依赖包

**requirements.txt**:
```
gym==0.23.0
gymnasium==1.2.0
hydra-core==1.3.2
git+https://github.com/BartekCupial/Minigrid.git
git+https://github.com/balrog-ai/minihack.git
git+https://github.com/nacloos/baba-is-ai
```

### 环境列表

| 环境名称 | 描述 | 文件路径 |
|---------|------|---------|
| **MiniHack** | NLE 自定义任务 | `environments/minihack/` |
| **BabyAI Text** | 语言导航 | `environments/babyai_text/` |
| **Baba Is AI** | 规则推理 | `environments/babaisai/` |
| **Crafter** | 开放生存 | `environments/crafter/` |
| **NLE** | NetHack | `environments/nle/` |
| **TextWorld** | 文本冒险 | `environments/textworld/` |

---

## 数据模型

### 状态表示

**MiniHack/NLE**:
```python
observation = {
    "glyphs": np.array,      # 地形符号
    "colors": np.array,      # 颜色信息
    "blstats": np.array,     # 角色状态
    "message": str,          # 游戏消息
    "inv_glyphs": np.array,  # 物品符号
    "inv_strs": list,        # 物品名称
}
```

**BabyAI Text**:
```python
observation = {
    "mission": str,          # 任务描述
    "observation": str,      # 环境描述
}
```

**Crafter**:
```python
observation = {
    "inventory": dict,       # 物品栏
    "player_pos": tuple,     # 玩家位置
    "semantic": np.array,    # 语义地图
}
```

### 奖励信号

- **进度奖励**: 探索新区域、完成目标
- **时间惩罚**: 鼓励高效完成
- **死亡惩罚**: 失败的负反馈

---

## 环境子模块

### MiniHack

**文件**: `domains/balrog/environments/minihack/minihack_env.py`

特点：
- 基于 NetHack 的自定义任务
- 多样化挑战（战斗、导航、解谜）
- 支持程序化生成

### BabyAI Text

**文件**: `domains/balrog/environments/babyai_text/babyai_env.py`

特点：
- 自然语言指令
- 合成语言导航
- 语言与行动对齐

### Baba Is AI

**文件**: `domains/balrog/environments/babaisai/babaisai_env.py`

特点：
- 规则推理
- 动态游戏机制
- 逻辑推演

### Crafter

**文件**: `domains/balrog/environments/crafter/crafter_env.py`

特点：
- 开放世界探索
- 资源收集
- 长期规划

### NLE (NetHack)

**文件**: `domains/balrog/environments/nle/nle_env.py`

特点：
- 经典 roguelike
- 复杂状态空间
- 永久死亡

---

## 测试与质量

### 评估指标

**主要指标**: `average_progress`

```python
# 计算平均进度
progress = (
    completed_steps / max_steps +
    achievements_count / total_achievements
) / 2
```

### 数据集

**domains/balrog/dataset.py**:
```python
class InContextDataset:
    """用于上下文学习的数据集"""

    def __init__(self, config):
        # 加载演示数据
        self.demos = load_demonstrations(config)
```

---

## 常见问题 (FAQ)

### Q1: 如何选择合适的 Balrog 环境？

**A**: 根据任务需求：

- **导航任务**: MiniHack-CorridorBattle
- **语言理解**: BabyAI Text
- **规则推理**: Baba Is AI
- **长期规划**: Crafter
- **复杂决策**: NLE

### Q2: 如何自定义 MiniHack 任务？

**A**: 修改配置文件：

```yaml
balrog_task:
  name: minihack
  environment:
    id: MiniHack-Custom-v0  # 自定义任务ID
  custom_args:
    des_file: "path/to/des_file"
```

### Q3: 如何加速评估？

**A**: 使用并行worker：

```bash
python -m domains.harness \
    --domain balrog_minihack \
    --num_workers 8  # 8个并行评估
```

### Q4: 如何查看游戏过程？

**A**: 检查轨迹文件：

```bash
# 查看特定episode的轨迹
cat outputs/{run_id}/balrog_minihack/episode_0/seed_0/trajectory.json
```

---

## 相关文件清单

### 核心文件
- `domains/balrog/eval.py` - 评估逻辑
- `domains/balrog/evaluator.py` - 评估器管理
- `domains/balrog/dataset.py` - 数据集
- `domains/balrog/utils.py` - 工具函数
- `domains/balrog/config/config.yaml` - 配置

### 环境实现
- `domains/balrog/environments/minihack/` - MiniHack
- `domains/balrog/environments/babyai_text/` - BabyAI Text
- `domains/balrog/environments/babaisai/` - Baba Is AI
- `domains/balrog/environments/crafter/` - Crafter
- `domains/balrog/environments/nle/` - NLE
- `domains/balrog/environments/textworld/` - TextWorld

### 工具
- `domains/balrog/environments/wrappers/` - Gym兼容包装器
- `domains/balrog/environments/env_wrapper.py` - 统一环境接口

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 Balrog 模块文档
- ✅ 记录所有环境接口
- ✅ 文档化评估指标
- 📝 待添加：环境详细说明
- 📝 待添加：自定义任务指南

---

*此模块文档由 PAI Architecture Agent 自动生成*
