# Agent 模块 - 核心智能体框架

[根目录](../CLAUDE.md) > **agent/**

> **更新时间**: 2026-03-30 11:14:03
>
> **模块类型**: Core Framework
>
> **主要语言**: Python 3.12+

---

## 模块职责

Agent 模块是 HyperAgents 框架的核心，提供了所有智能体的抽象基类和具体实现，包括：

- **抽象接口**: `AgentSystem` - 所有智能体的基类
- **Meta智能体**: `MetaAgent` - 递归式自我改进智能体
- **Task智能体**: `TaskAgent` - 任务执行智能体
- **LLM集成**: 统一的 LLM 接口，支持多个模型提供商
- **工具系统**: 可扩展的工具集（bash、编辑等）

---

## 入口与启动

### 核心类层次

```python
# 抽象基类
agent/base_agent.py: AgentSystem (ABC)
    ├── agent/meta_agent.py: MetaAgent
    └── agent/task_agent.py: TaskAgent
```

### 初始化示例

```python
from agent.base_agent import AgentSystem
from agent.meta_agent import MetaAgent
from agent.task_agent import TaskAgent

# Meta智能体初始化
meta_agent = MetaAgent(
    model="anthropic/claude-sonnet-4-5-20250929",
    chat_history_file='./outputs/chat_history.md'
)

# Task智能体初始化
task_agent = TaskAgent(
    model="openai/gpt-4o",
    chat_history_file='./outputs/task_chat.md'
)
```

### 使用流程

1. **Meta智能体改进循环**:
```python
# 在 generate_loop.py 中调用
prediction = meta_agent.forward(
    repo_path="/path/to/repo",
    eval_path="/path/to/eval/results",
    iterations_left=5
)
```

2. **Task智能体执行任务**:
```python
# 在领域评估中调用
inputs = {
    'domain': 'balrog_minihack',
    'task': 'Navigate to the goal',
    'observation': '...'
}
prediction, msg_history = task_agent.forward(inputs)
```

---

## 对外接口

### AgentSystem 抽象接口

```python
class AgentSystem(ABC):
    def __init__(
        self,
        model=OPENAI_MODEL,  # 默认使用 GPT-4o
        chat_history_file='./outputs/chat_history.md'
    ):
        """
        初始化智能体系统

        Args:
            model: LLM 模型标识符
            chat_history_file: 对话历史文件路径
        """
        self.model = model
        self.logger_manager = ThreadLoggerManager(log_file=chat_history_file)
        self.log = self.logger_manager.log

    @abstractmethod
    def forward(self, *args, **kwargs):
        """抽象方法：子类必须实现"""
        pass
```

### MetaAgent 接口

```python
class MetaAgent(AgentSystem):
    def forward(self, repo_path, eval_path, iterations_left=None):
        """
        递归式自我改进

        Args:
            repo_path (str): 要修改的代码库路径
            eval_path (str): 评估结果路径
            iterations_left (int): 剩余迭代次数

        Returns:
            str: 改进建议（差异格式）
        """
```

### TaskAgent 接口

```python
class TaskAgent(AgentSystem):
    def forward(self, inputs):
        """
        执行任务

        Args:
            inputs (dict): 任务输入，包含:
                - domain: 领域名称
                - task: 任务描述
                - observation: 观察数据
                - ... (领域特定字段)

        Returns:
            tuple: (prediction, new_msg_history)
                - prediction (str): 预测结果
                - new_msg_history (list): 对话历史
        """
```

### LLM 接口

```python
# agent/llm.py
def get_response_from_llm(
    msg: str,
    model: str = OPENAI_MODEL,
    temperature: float = 0.0,
    max_tokens: int = MAX_TOKENS,  # 16384
    msg_history=None
) -> Tuple[str, list, dict]:
    """
    调用 LLM 获取响应

    Args:
        msg: 提示消息
        model: 模型标识符
        temperature: 温度参数
        max_tokens: 最大token数
        msg_history: 对话历史

    Returns:
        tuple: (response_text, updated_history, metadata)
    """
```

---

## 关键依赖与配置

### 依赖包

**requirements.txt**:
```
litellm==1.74.9          # 统一LLM接口
backoff==2.2.1           # 重试机制
requests==2.32.4         # HTTP客户端
dotenv==0.9.9            # 环境变量管理
```

### 环境变量

**.env 文件**:
```bash
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=...
```

### 支持的模型

**agent/llm.py**:
```python
# Claude 系列
CLAUDE_MODEL = "anthropic/claude-sonnet-4-5-20250929"
CLAUDE_HAIKU_MODEL = "anthropic/claude-3-haiku-20240307"
CLAUDE_35NEW_MODEL = "anthropic/claude-3-5-sonnet-20241022"

# OpenAI 系列
OPENAI_MODEL = "openai/gpt-4o"
OPENAI_MINI_MODEL = "openai/gpt-4o-mini"
OPENAI_O3_MODEL = "openai/o3"
OPENAI_O3MINI_MODEL = "openai/o3-mini"
OPENAI_GPT52_MODEL = "openai/gpt-5.2"
OPENAI_GPT5_MODEL = "openai/gpt-5"

# Gemini 系列
GEMINI_3_MODEL = "gemini/gemini-3-pro-preview"
GEMINI_MODEL = "gemini/gemini-2.5-pro"
GEMINI_FLASH_MODEL = "gemini/gemini-2.5-flash"
```

---

## 数据模型

### 消息格式

**LLM 对话历史**:
```python
msg_history = [
    {
        "role": "system",  # 或 "user", "assistant"
        "content": "..."   # 或 "text" (兼容旧格式)
    },
    # ... 更多消息
]
```

### JSON 提取格式

TaskAgent 期望 LLM 返回 JSON 格式的响应：

```
<json>
{
    "response": "..."
}
</json>
```

或

```
```json
{
    "response": "..."
}
```
```

---

## 测试与质量

### 测试覆盖
- ❌ **单元测试**: 未发现
- ❌ **集成测试**: 未发现
- ✅ **手动测试**: 通过 `generate_loop.py` 端到端测试

### 错误处理

**重试机制**:
```python
@backoff.on_exception(
    backoff.expo,
    (requests.exceptions.RequestException, json.JSONDecodeError, KeyError),
    max_time=600,
    max_value=60,
)
def get_response_from_llm(...):
    # 指数退避重试，最长10分钟
```

**日志记录**:
- 使用 `ThreadLoggerManager` 支持多线程日志
- 所有对话记录到 Markdown 文件
- 关键错误记录但继续执行

---

## 常见问题 (FAQ)

### Q1: 如何切换 LLM 模型？

**A**: 在初始化时传入 `model` 参数：

```python
# 使用 Claude
agent = MetaAgent(model="anthropic/claude-sonnet-4-5-20250929")

# 使用 GPT-4o
agent = MetaAgent(model="openai/gpt-4o")

# 使用 Gemini
agent = MetaAgent(model="gemini/gemini-2.5-pro")
```

### Q2: MetaAgent 如何生成改进？

**A**: MetaAgent 的流程：

1. 读取评估结果 (`eval_path`)
2. 分析表现不佳的案例
3. 生成代码修改建议（差异格式）
4. 返回改进方案供应用

### Q3: 如何添加新工具？

**A**: 在 `agent/tools/` 下创建新文件：

```python
# agent/tools/mytool.py
def my_tool(args):
    """工具实现"""
    pass

# 在 agent/tools/__init__.py 中注册
```

然后在 `agent/llm_withtools.py` 中添加工具定义。

### Q4: 如何调试智能体行为？

**A**: 检查日志文件：

```bash
# 查看对话历史
cat outputs/chat_history.md

# 查看评估日志
cat outputs/eval.log
```

---

## 相关文件清单

### 核心文件
- `agent/base_agent.py` - 抽象基类
- `agent/meta_agent.py` - Meta智能体
- `agent/task_agent.py` - Task智能体
- `agent/llm.py` - LLM接口
- `agent/llm_withtools.py` - 带工具的LLM接口

### 工具
- `agent/tools/__init__.py` - 工具注册
- `agent/tools/bash.py` - Bash命令执行
- `agent/tools/edit.py` - 文件编辑

### 使用示例
- `meta_agent.py` - 根目录Meta智能体入口
- `task_agent.py` - 根目录Task智能体入口
- `generate_loop.py` - 整体循环入口

---

## 变更记录 (Changelog)

### 2026-03-30 - 初始化文档
- ✅ 创建模块 CLAUDE.md
- ✅ 记录核心接口和数据模型
- ✅ 文档化 LLM 集成
- 📝 待添加：单元测试示例
- 📝 待添加：工具开发指南

---

*此模块文档由 PAI Architecture Agent 自动生成*
