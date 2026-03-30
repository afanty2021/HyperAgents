# Paper Review 模块 - 论文评审能力

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **paper_review**

> **更新时间**: 2026-03-30 11:34:52
>
> **模块类型**: Domain Implementation
>
> **主要语言**: Python 3.12+

---

## 模块职责

Paper Review 模块实现了对 AI 智能体论文评审能力的评估基准。该模块评估智能体判断论文是否应该被学术会议或期刊接受的能力，基于论文全文内容做出接受/拒绝决策。

**核心功能**：
- 生成平衡的训练/验证/测试集划分
- 提供统一的评估接口（通过 `domains/harness.py`）
- 支持多种论文评审场景的评估

---

## 数据集

### 数据集来源

该模块使用内部论文数据集，通过 `domains/harness.py` 的 `get_dataset()` 函数加载。

### 数据格式

每个样本包含：
- `paper_text`: 论文全文内容
- `outcome`: 评审结果（`accept`/`reject`）
- `question_id`: 唯一标识符

### 数据预处理

**过滤规则**：
1. 仅保留明确接受或拒绝的样本（`outcome` ∈ `["accept", "reject"]`）
2. 确保数据平衡（接受和拒绝样本数量相等）

**数据划分**：
- **训练集**: 100 个样本（50 个接受，50 个拒绝）
- **验证集**: 100 个样本（50 个接受，50 个拒绝）
- **测试集**: 100 个样本（50 个接受，50 个拒绝）

---

## 入口与启动

### 数据集准备

```bash
# 生成数据集划分
python domains/paper_review/curate_subsets.py
```

**输出文件**：
- `domains/paper_review/dataset_filtered_100_train.csv` - 训练集
- `domains/paper_review/dataset_filtered_100_val.csv` - 验证集
- `domains/paper_review/dataset_filtered_100_test.csv` - 测试集

### 评估接口

```bash
# 通过 harness 运行评估
python -m domains.harness \
    --domain paper_review \
    --run_id <run_id> \
    --num_samples <num_samples>

# 生成报告
python -m domains.report \
    --domain paper_review \
    --dname ./outputs/<run_id>
```

---

## 对外接口

### utils.py

**常量**：
```python
QUESTION_ID = "question_id"
GROUND_TRUTH_KEY = "outcome"
MODEL = "gpt-4o"
```

**函数**：
```python
def format_input_dict(row):
    """
    从数据集行提取输入字典

    Args:
        row: pandas DataFrame 行，包含 paper_text, outcome

    Returns:
        dict: {
            "domain": "paper_review",
            "paper_text": "..."
        }
    """
```

### curate_subsets.py

**主要函数**：
```python
def make_balanced_splits(
    df,
    num_samples=100,
    shuffle_seed=42,
    prefix=""
):
    """
    创建平衡的训练/验证/测试集划分

    Args:
        df: 源数据集
        num_samples: 每个划分的样本总数（默认 100）
        shuffle_seed: 随机种子（默认 42）
        prefix: 输出文件名前缀

    Returns:
        None（直接保存 CSV 文件）
    """
```

**辅助函数**：
```python
def _largest_remainder(target, sizes):
    """
    使用最大余数法分配名额

    Args:
        target: 总名额
        sizes: 各部分大小

    Returns:
        list: 分配结果
    """

def _print_split_stats(name, df_split):
    """
    打印数据集统计信息

    Args:
        name: 数据集名称（train/val/test）
        df_split: 数据集 DataFrame
    """

def _leak_check(a_df, b_df, a_name, b_name):
    """
    检查两个数据集之间的重叠

    Args:
        a_df, b_df: 两个数据集
        a_name, b_name: 数据集名称
    """
```

---

## 关键依赖与配置

### 依赖项

```python
# requirements.txt
pandas
```

### 配置文件

无独立配置文件，使用 `domains/harness.py` 的统一配置。

---

## 数据模型

### 输入格式

```python
{
    "domain": "paper_review",
    "paper_text": "Full paper text here..."
}
```

### 输出格式

```python
{
    "outcome": "accept",  # 或 "reject"
    "question_id": 12345
}
```

### 评估指标

- **overall_accuracy**: 预测与真实结果匹配的比例

---

## 测试与质量

### 数据质量保证

1. **重叠检查**: 确保 train/val/test 之间无重复 question_id
2. **平衡性检查**: 每个划分中接受和拒绝样本数量相等
3. **数据充足性检查**: 确保有足够的数据生成指定大小的划分

### 验证脚本

`curate_subsets.py` 内置验证：
```python
# 重叠检查
_leak_check(splits["train"], splits["val"], "train", "val")
_leak_check(splits["train"], splits["test"], "train", "test")
_leak_check(splits["val"], splits["test"], "val", "test")

# 统计信息打印
_print_split_stats("train", splits["train"])
_print_split_stats("val", splits["val"])
_print_split_stats("test", splits["test"])
```

---

## 常见问题 (FAQ)

### Q1: 如何使用自定义数据集？

**A**: 修改 `curate_subsets.py` 主函数中的数据加载：

```python
# 原代码
df = get_dataset(domain="paper_review")

# 修改为
df = pd.read_csv("your-custom-dataset.csv")
```

### Q2: 如何调整样本数量？

**A**: 修改 `curate_subsets.py` 主函数中的参数：

```python
make_balanced_splits(
    df,
    num_samples=200,  # 每个划分 200 个样本
    shuffle_seed=42
)
```

### Q3: 如何处理三类评审结果？

**A**: 当前仅支持二分类（accept/reject）。如需支持第三类（如 `weak_accept`），需修改：

1. 更新过滤条件：
```python
df = df[df['outcome'].isin(['accept', 'reject', 'weak_accept'])].copy()
```

2. 修改平衡逻辑以支持三类

---

## 相关文件清单

### 核心文件
- `domains/paper_review/utils.py` - 工具函数和常量
- `domains/paper_review/curate_subsets.py` - 数据集划分生成

### 数据文件
- `domains/paper_review/dataset.csv` - 原始数据集（通过 harness 加载）
- `domains/paper_review/dataset_filtered_100_train.csv` - 训练集
- `domains/paper_review/dataset_filtered_100_val.csv` - 验证集
- `domains/paper_review/dataset_filtered_100_test.csv` - 测试集

### 配置文件
- 无（使用统一 harness 接口）

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 paper_review 模块文档
- ✅ 记录数据集结构和处理流程
- ✅ 记录评估接口和工具函数
- 📝 待添加：多类别评审支持
- 📝 待添加：更细粒度的评审指标

---

*此模块文档由 PAI Architecture Agent 自动生成*
