# IMO 模块 - 数学证明生成与评分

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **imo/**

> **更新时间**: 2026-03-30 11:14:03
>
> **模块类型**: Mathematical Reasoning
>
> **主要语言**: Python, Sympy

---

## 模块职责

IMO 模块实现了数学奥林匹克问题的证明生成和评分：

- **IMO Proof**: 生成数学问题证明
- **IMO Grading**: 评分数学证明
- **形式化验证**: 使用 ProofGrader 进行验证
- **多领域覆盖**: 代数、几何、组合、数论

---

## 入口与启动

### 评估命令

```bash
# IMO Proof 评估
python -m domains.harness \
    --domain imo_proof \
    --run_id test_imo_proof \
    --num_samples 10

# IMO Grading 评估
python -m domains.harness \
    --domain imo_grading \
    --run_id test_imo_grading \
    --num_samples 10
```

### ProofGrader 设置

```bash
# 安装 ProofGrader
python -m domains.imo.setup_proofgrader_repo

# 或在 Dockerfile 中
RUN pip install -e proofgrader_repo
```

---

## 对外接口

### Proof 评估

```python
from domains.imo.proof_eval import evaluate_proof

# 评估证明
result = evaluate_proof(
    problem_statement=problem,
    proof=generated_proof,
    grader_path="proofgrader_repo/"
)
```

### Grading 评估

```python
from domains.imo.grading_utils import grade_proof

# 评分证明
score = grade_proof(
    problem=problem,
    proof=proof,
    rubric=grading_rubric
)
```

---

## 关键依赖与配置

### 依赖包

**requirements.txt**:
```
sympy==1.14.0    # 符号计算
proofgrader      # 形式化验证
```

### 数据集

**IMO 问题集**:
- 代数问题
- 几何问题
- 组合问题
- 数论问题

---

## 数据模型

### 问题定义

```python
problem = {
    "id": str,
    "year": int,
    "country": str,
    "category": str,  # algebra, geometry, combinatorics, number_theory
    "problem_number": int,
    "statement": str,
    "hints": list,
    "difficulty": float
}
```

### 证明定义

```python
proof = {
    "problem_id": str,
    "proof_text": str,
    "steps": list,
    "lemmas": list,
    "formalization": dict
}
```

### 评分结果

```python
grading_result = {
    "problem_id": str,
    "proof": proof,
    "score": float,           # 0-7 分
    "points_percentage": float,  # 0-1
    "feedback": str,
    "errors": list
}
```

---

## 评估指标

### IMO Proof

**主要指标**: `points_percentage`

```python
# 满分7分，计算百分比
percentage = total_points / (7 * num_problems)
```

### IMO Grading

**主要指标**: `overall_accuracy`

```python
# 评分准确率
accuracy = correct_gradings / total_gradings
```

---

## 证明生成

### 工作流程

1. **问题理解**: 分析问题陈述
2. **策略选择**: 选择解题策略
3. **步骤生成**: 生成证明步骤
4. **验证**: 验证证明正确性
5. **形式化**: 转换为形式化表示

### 示例

```python
# domains/imo/proof_utils.py
def generate_proof(problem, agent):
    """
    生成数学证明

    Args:
        problem: IMO 问题
        agent: TaskAgent 实例

    Returns:
        proof: 生成的证明
    """
    instruction = f"""
    Solve the following IMO problem:

    {problem['statement']}

    Provide a step-by-step proof.
    """

    response, _ = agent.forward({"problem": problem})
    return parse_proof(response)
```

---

## 评分系统

### ProofGrader 集成

**domains/imo/proof_grading_utils.py**:
```python
def grade_with_proofgrader(proof, problem):
    """
    使用 ProofGrader 评分

    Args:
        proof: 待评分的证明
        problem: 问题陈述

    Returns:
        score: 评分 (0-7)
    """
    # 调用 ProofGrader API
    grader = ProofGrader()
    result = grader.grade(proof, problem)
    return result.score
```

### 评分标准

| 分数 | 标准 |
|-----|------|
| 7 | 完全正确 |
| 6- | 微小错误 |
| 5- | 部分正确，主要思路正确 |
| 2- | 有进展但未完成 |
| 0 | 无进展或错误 |

---

## 测试与质量

### 数据集划分

```python
# domains/imo/curate_subsets.py
subsets = {
    "train": ["2000-2020"],
    "val": ["2021-2022"],
    "test": ["2023-2024"]
}
```

### 质量保证

- **形式化验证**: 使用 ProofGrader 验证
- **人工审查**: 部分证明人工检查
- **一致性检查**: 多次运行验证稳定性

---

## 常见问题 (FAQ)

### Q1: 如何添加新的 IMO 问题？

**A**: 添加到数据集：

```python
# domains/imo/dataset.py
new_problem = {
    "id": "2024_1",
    "year": 2024,
    "statement": "...",
    "category": "algebra"
}
```

### Q2: 如何验证证明正确性？

**A**: 使用 ProofGrader：

```python
from domains.imo.proof_eval import verify_proof

is_valid = verify_proof(proof, problem)
```

### Q3: 如何调试评分？

**A**: 检查评分日志：

```bash
# 查看评分详情
cat outputs/{run_id}/imo_grading/episode_0/grading_details.json
```

### Q4: 如何自定义评分标准？

**A**: 修改评分规则：

```python
# domains/imo/grading_utils.py
def custom_grading_rubric(proof, problem):
    # 自定义评分逻辑
    pass
```

---

## 相关文件清单

### 核心文件
- `domains/imo/proof_eval.py` - 证明评估
- `domains/imo/grading_utils.py` - 评分工具
- `domains/imo/proof_utils.py` - 证明工具
- `domains/imo/proof_grading_utils.py` - 证明评分

### 数据准备
- `domains/imo/curate_subsets.py` - 数据子集
- `domains/imo/analyze_imo_proof.py` - 证明分析
- `domains/imo/setup_proofgrader_repo.py` - ProofGrader设置

### 基线
- `baselines/imo_proof/agent.py` - Proof智能体
- `baselines/imo_grading/proofautograder.py` - 评分器

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 IMO 模块文档
- ✅ 记录证明生成和评分流程
- ✅ 文档化评估指标
- 📝 待添加：新问题添加指南
- 📝 待添加：自定义评分指南

---

*此模块文档由 PAI Architecture Agent 自动生成*
