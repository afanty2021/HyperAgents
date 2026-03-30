# Search Arena 模块 - 搜索能力评估

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **search_arena**

> **更新时间**: 2026-03-30 11:34:52
>
> **模块类型**: Domain Implementation
>
> **主要语言**: Python 3.12+

---

## 模块职责

Search Arena 模块实现了对 AI 智能体搜索能力的评估基准。该模块基于 [LM Arena Search Arena v1.7k](https://huggingface.co/datasets/lmarena-ai/search-arena-v1-7k) 数据集，评估智能体在搜索任务中的表现，特别是对比两个模型生成的搜索结果的优劣。

**核心功能**：
- 下载和预处理 Search Arena 数据集
- 生成平衡的训练/验证/测试集划分
- 提供统一的评估接口（通过 `domains/harness.py`）
- 支持多种搜索场景的评估

---

## 数据集

### 数据集来源

- **名称**: lmarena-ai/search-arena-v1-7k
- **平台**: Hugging Face Datasets
- **语言**: 英语（过滤后）
- **大小**: 约 7,000 个搜索查询对比样本

### 数据格式

每个样本包含：
- `question_id`: 唯一标识符
- `messages_a`: 模型 A 的搜索对话历史
- `messages_b`: 模型 B 的搜索对话历史
- `winner`: 对比结果（`a`/`b`/`tie`/`tie (bothbad)`）
- `language`: 问题语言

### 数据预处理

**过滤规则**：
1. 排除平局样本（`winner == "tie"` 或 `winner == "tie (bothbad)"`）
2. 仅保留英语样本（`language == "English"`）
3. 随机打乱（`random_state=42`）

**数据划分**：
- **训练集**: 100 个样本（50 个 A 胜，50 个 B 胜）
- **验证集**: 100 个样本（50 个 A 胜，50 个 B 胜）
- **测试集**: 100 个样本（50 个 A 胜，50 个 B 胜）

---

## 入口与启动

### 数据集准备

```bash
# 下载并预处理数据集
python domains/search_arena/curate_subsets.py
```

**输出文件**：
- `domains/search_arena/dataset.csv` - 原始数据集
- `domains/search_arena/dataset_filtered.csv` - 过滤后的数据集
- `domains/search_arena/dataset_filtered_100_train.csv` - 训练集
- `domains/search_arena/dataset_filtered_100_val.csv` - 验证集
- `domains/search_arena/dataset_filtered_100_test.csv` - 测试集

### 评估接口

```bash
# 通过 harness 运行评估
python -m domains.harness \
    --domain search_arena \
    --run_id <run_id> \
    --num_samples <num_samples>

# 生成报告
python -m domains.report \
    --domain search_arena \
    --dname ./outputs/<run_id>
```

---

## 对外接口

### utils.py

**常量**：
```python
QUESTION_ID = "question_id"
GROUND_TRUTH_KEY = "winner"
MODEL = "gpt-4o"
```

**函数**：
```python
def format_input_dict(row):
    """
    从数据集行提取输入字典

    Args:
        row: pandas DataFrame 行，包含 messages_a, messages_b, winner

    Returns:
        dict: {
            "domain": "search_arena",
            "messages_a": [...],
            "messages_b": [...]
        }
    """
```

### curate_subsets.py

**主要函数**：
```python
def download_dataset() -> pd.DataFrame:
    """
    通过 Hugging Face Datasets 下载 Search Arena 数据集
    保存为 CSV 并返回 DataFrame

    Returns:
        pd.DataFrame: 完整的数据集
    """

def make_split(df: pd.DataFrame, id_list: list[int]) -> pd.DataFrame:
    """
    根据 ID 列表创建数据子集，保持指定顺序

    Args:
        df: 源数据集
        id_list: 目标 ID 列表（决定顺序）

    Returns:
        pd.DataFrame: 过滤并排序后的数据集
    """
```

---

## 关键依赖与配置

### 依赖项

```python
# requirements.txt
pandas
datasets
```

### 配置文件

无独立配置文件，使用 `domains/harness.py` 的统一配置。

---

## 数据模型

### 输入格式

```python
{
    "domain": "search_arena",
    "messages_a": [
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
    ],
    "messages_b": [
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
    ]
}
```

### 输出格式

```python
{
    "winner": "a",  # 或 "b"
    "question_id": 12345
}
```

### 评估指标

- **overall_accuracy**: 预测与真实结果匹配的比例

---

## 测试与质量

### 数据质量保证

1. **重叠检查**: 确保 train/val/test 之间无重复 question_id
2. **平衡性检查**: 每个划分中 A 胜和 B 胜样本数量相等
3. **存在性检查**: 所有划分中的 ID 都存在于过滤后的数据集中

### 验证脚本

`curate_subsets.py` 内置验证：
```python
# 重叠检查
assert not (train_ids & val_ids), "train/val overlap in question_id"
assert not (train_ids & test_ids), "train/test overlap in question_id"
assert not (val_ids & test_ids), "val/test overlap in question_id"

# 大小检查
assert len(train_df) == len(train_list), "train_df rows != train_list length"
```

---

## 常见问题 (FAQ)

### Q1: 如何使用自定义数据集？

**A**: 替换 `download_dataset()` 函数中的数据集路径：

```python
ds = load_dataset("your-custom-dataset")
```

### Q2: 如何调整样本数量？

**A**: 修改 `curate_subsets.py` 中的 `num_samples` 参数：

```python
make_balanced_splits(
    df,
    num_samples=200,  # 每个划分 200 个样本
    shuffle_seed=42
)
```

### Q3: 如何评估其他语言？

**A**: 修改过滤条件：

```python
# 原代码
df = df[df["language"] == "English"].copy()

# 修改为（支持所有语言）
# df = df  # 不过滤语言

# 或指定其他语言
# df = df[df["language"].isin(["English", "Spanish"])].copy()
```

---

## 相关文件清单

### 核心文件
- `domains/search_arena/utils.py` - 工具函数和常量
- `domains/search_arena/curate_subsets.py` - 数据集下载和划分

### 数据文件
- `domains/search_arena/dataset.csv` - 原始数据集
- `domains/search_arena/dataset_filtered.csv` - 过滤后数据集
- `domains/search_arena/dataset_filtered_100_train.csv` - 训练集
- `domains/search_arena/dataset_filtered_100_val.csv` - 验证集
- `domains/search_arena/dataset_filtered_100_test.csv` - 测试集
- `domains/search_arena/dataset_filtered_100_ids.json` - 划分 ID 映射

### 配置文件
- 无（使用统一 harness 接口）

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 search_arena 模块文档
- ✅ 记录数据集结构和处理流程
- ✅ 记录评估接口和工具函数
- 📝 待添加：更多评估指标
- 📝 待添加：多语言支持

---

*此模块文档由 PAI Architecture Agent 自动生成*
