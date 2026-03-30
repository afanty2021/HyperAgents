# Polyglot 模块 - 代码生成基准测试

[根目录](../../CLAUDE.md) > [domains](../CLAUDE.md) > **polyglot/**

> **更新时间**: 2026-03-30 11:14:03
>
> **模块类型**: Code Generation Benchmark
>
> **主要语言**: Python, Docker, Git

---

## 模块职责

Polyglot 模块实现了代码生成基准测试，用于评估 AI 智能体的编程能力：

- **多语言支持**: Python、JavaScript、TypeScript 等
- **真实代码库**: 基于开源项目的实际任务
- **自动化测试**: 通过测试用例验证生成代码
- **Docker 隔离**: 安全执行生成的代码

---

## 入口与启动

### 评估命令

```bash
# 运行代码生成评估
python -m domains.polyglot.run_evaluation \
    --subset small \
    --num_samples 10

# 或通过 harness
python -m domains.harness \
    --domain polyglot \
    --run_id test_polyglot \
    --num_samples 10
```

### 数据集准备

```bash
# 准备 Polyglot 数据集
python -m domains.polyglot.prepare_polyglot_dataset \
    --subset small \
    --output_dir ./data/polyglot
```

---

## 对外接口

### 评估接口

```python
from domains.polyglot.harness import run_polyglot_evaluation

# 运行评估
results = run_polyglot_evaluation(
    subset="small",
    num_samples=10,
    model="openai/gpt-4o"
)
```

### 基准测试

```python
from domains.polyglot.benchmark import benchmark_agent

# 基准测试
scores = benchmark_agent(
    agent=task_agent,
    test_cases=test_cases,
    timeout=300  # 5分钟超时
)
```

---

## 关键依赖与配置

### 依赖包

**requirements.txt**:
```
docker==7.1.0           # Docker API
GitPython==3.1.44       # Git操作
datasets==3.6.0         # 数据集加载
```

### 数据集

**预定义子集**:
- `domains/polyglot/subsets/small.json` - 小型测试集
- `domains/polyglot/subsets/medium.json` - 中型测试集

### Docker 配置

**domains/polyglot/dockerfiles.py**:
```python
DOCKERFILE_TEMPLATE = """
FROM python:3.12-slim

# 安装依赖
RUN pip install -r requirements.txt

# 复制代码
COPY . /app

# 运行测试
CMD ["python", "-m", "pytest", "tests/"]
"""
```

---

## 数据模型

### 任务定义

```python
task = {
    "repo_url": "https://github.com/user/repo",
    "base_commit": "abc123",
    "problem_statement": "Add feature X",
    "test_files": ["tests/test_x.py"],
    "source_files": ["src/x.py"],
    "instructions": "..."
}
```

### 评估结果

```python
result = {
    "task_id": str,
    "prediction": str,           # 生成的代码
    "test_passed": bool,         # 测试是否通过
    "test_output": str,          # 测试输出
    "compilation_success": bool, # 编译是否成功
    "execution_time": float,     # 执行时间
    "accuracy": float            # 准确度分数
}
```

---

## Docker 隔离

### 容器构建

**domains/polyglot/docker_build.py**:
```python
def build_test_container(repo_path, dockerfile_path):
    """
    构建测试容器

    Args:
        repo_path: 代码库路径
        dockerfile_path: Dockerfile路径

    Returns:
        container_id: 容器ID
    """
    client = docker.from_env()
    image = client.images.build(path=repo_path, dockerfile=dockerfile_path)
    return image.id
```

### 容器执行

**domains/polyglot/docker_utils.py**:
```python
def run_tests_in_container(container_id, test_command):
    """
    在容器中运行测试

    Args:
        container_id: 容器ID
        test_command: 测试命令

    Returns:
        exit_code: 退出码
        output: 输出日志
    """
    client = docker.from_env()
    container = client.containers.get(container_id)
    exit_code, output = container.exec_run(test_command)
    return exit_code, output.decode('utf-8')
```

---

## 评估流程

### 步骤

1. **任务选择**: 从数据集中选择任务
2. **代码生成**: TaskAgent 生成代码
3. **差异应用**: 应用代码到代码库
4. **容器构建**: 构建 Docker 容器
5. **测试执行**: 在容器中运行测试
6. **结果收集**: 记录测试结果

### 伪代码

```python
# 伪代码流程
for task in tasks:
    # 生成代码
    prediction = task_agent.forward(task)

    # 应用差异
    repo = clone_repo(task['repo_url'])
    apply_diff(repo, prediction)

    # 构建容器
    container = build_container(repo)

    # 运行测试
    exit_code, output = run_tests(container)

    # 记录结果
    results.append({
        'task_id': task['id'],
        'test_passed': exit_code == 0,
        'output': output
    })
```

---

## 测试与质量

### 评估指标

**主要指标**: `accuracy_score`

```python
accuracy = num_passed_tests / total_tests
```

### 质量保证

- **超时限制**: 每个任务5分钟超时
- **资源限制**: CPU和内存限制
- **网络隔离**: 容器内无网络访问
- **安全限制**: 沙箱执行

---

## 常见问题 (FAQ)

### Q1: 如何添加新的编程语言？

**A**: 创建新的 Dockerfile 模板：

```python
# domains/polyglot/dockerfiles.py
DOCKERFILE_TEMPLATES['rust'] = """
FROM rust:1.70
WORKDIR /app
COPY . .
RUN cargo test
"""
```

### Q2: 如何调试代码生成？

**A**: 检查生成的差异：

```bash
# 查看生成的代码差异
cat outputs/{run_id}/polyglot/episode_0/diff.patch

# 查看测试输出
cat outputs/{run_id}/polyglot/episode_0/test_results.json
```

### Q3: 如何处理编译错误？

**A**: 检查容器日志：

```bash
# 查看容器构建日志
docker logs {container_id}

# 查看测试执行日志
docker logs {container_id} --tail 100
```

### Q4: 如何自定义测试子集？

**A**: 创建新的 JSON 文件：

```json
// domains/polyglot/subsets/custom.json
{
  "tasks": [
    {
      "repo_url": "...",
      "commit": "...",
      "problem": "..."
    }
  ]
}
```

---

## 相关文件清单

### 核心文件
- `domains/polyglot/run_evaluation.py` - 评估入口
- `domains/polyglot/harness.py` - 测试工具
- `domains/polyglot/benchmark.py` - 基准测试
- `domains/polyglot/utils.py` - 工具函数

### Docker 相关
- `domains/polyglot/docker_build.py` - 容器构建
- `domains/polyglot/docker_utils.py` - 容器工具
- `domains/polyglot/dockerfiles.py` - Dockerfile 模板

### Git 相关
- `domains/polyglot/git_utils.py` - Git 操作
- `domains/polyglot/testrepo_prompt.py` - 测试仓库提示

### 数据集
- `domains/polyglot/prepare_polyglot_dataset.py` - 数据准备
- `domains/polyglot/subsets/small.json` - 小型子集
- `domains/polyglot/subsets/medium.json` - 中型子集

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建 Polyglot 模块文档
- ✅ 记录 Docker 隔离机制
- ✅ 文档化评估流程
- 📝 待添加：新语言支持指南
- 📝 待添加：自定义任务指南

---

*此模块文档由 PAI Architecture Agent 自动生成*
