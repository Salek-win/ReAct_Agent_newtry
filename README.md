# ReAct Agent

基于 ReAct（Reasoning + Acting） 范式的 LLM 智能体，通过"思考 → 行动 → 观察"循环赋予大模型自主调用外部工具的能力。

---

## 快速开始

### 环境准备

- Python >= 3.12
- [uv](https://docs.astral.sh/uv/) 包管理器

### 安装依赖

```bash
uv sync
```

### 配置 API Key

在项目根目录创建 `.env` 文件：

```bash
DASHSCOPE_API_KEY=sk-your-key-here
BASE_URL=**********
MODEL=**********  
```

### 运行

```bash
uv run agent.py <任务>
```

示例：

```bash
uv run agent.py .
<打开对话框>
```

---

## 项目结构

```
.
├── agent.py              # ReAct 引擎核心（ReActAgent 类 + 工具函数 + CLI 入口）
├── prompt_template.py    # 系统提示模板（XML 标签规范 + Few-Shot 示例）
├── pyproject.toml        # 项目配置与依赖声明
├── uv.lock               # 依赖锁定文件
└── .env                  # API 密钥等敏感配置
```

---

## 技术架构

```
用户输入
    │
    ▼
┌──────────────────────────────────────────┐
│  ReActAgent.run()                        │
│  ├── 构造 messages [system + user]       │
│  ├── while True:                         │
│  │   ├── call_model() → LLM              │
│  │   ├── 解析 <thought>                   │
│  │   ├── 检测 <final_answer> ? → 返回    │
│  │   ├── 解析 <action> → parse_action()   │
│  │   ├── 执行工具函数                     │
│  │   └── 回传 <observation> → messages    │
│  └── end while                           │
└──────────────────────────────────────────┘
    │
    ▼
通义千问 qwen-plus（阿里云 DashScope）
```

---

## 核心模块

### 1. ReAct 循环引擎（`agent.py`）

| 组件 | 说明 |
|------|------|
| `ReActAgent` 类 | 封装完整的 ReAct 推理循环 |
| `run(user_input)` | 单次任务入口，自动迭代 Thought → Action → Observation |
| `call_model()` | 调用 DashScope API，将 assistant 回复追加到 messages |
| `parse_action()` | 自定义参数字符串解析器，支持多行字符串、嵌套括号、转义字符 |
| `_parse_single_arg()` | 处理字符串字面量转义（`\n` → 换行，`\\` → 反斜杠等） |

### 2. 动态工具系统

通过 `inspect` 模块自动提取函数签名和 DocString，实现工具的动态注册与发现：

```python
def read_file(file_path):
    """用于读取文件内容"""
    ...

def write_to_file(file_path, content):
    """将指定内容写入指定文件"""
    ...

def run_terminal_command(command):
    """用于执行终端命令"""
    ...
```

新增工具只需编写一个带 DocString 的函数，在 `main()` 中注册即可，零侵入核心逻辑。

### 3. 结构化 Prompt Engineering（`prompt_template.py`）

系统提示模板包含：

- **ReAct 流程规范**：问题分解 → 工具调用 → 结果评估 → 最终答案
- **2 组 Few-Shot 示例**：单工具查询、多工具串联任务
- **严格格式约束**：XML 标签（`<thought>` / `<action>` / `<observation>` / `<final_answer>`）、文件路径必须使用绝对路径、多行参数用 `\n` 转义
- **动态变量注入**：`${tool_list}`（可用工具列表）、`${operating_system}`（操作系统信息）、`${file_list}`（当前目录文件列表）

### 4. 安全机制

- **终端命令执行前强制 Y/N 确认**：防止 Agent 误操作对本地环境造成破坏
- **API Key 环境变量管理**：通过 `.env` 文件加载，避免硬编码到代码中


## 依赖

| 包 | 版本 | 用途 |
|----|------|------|
| `openai` | >=1.91.0 | OpenAI SDK（复用为通用 HTTP 客户端，对接 DashScope 兼容端点） |
| `click` | >=8.2.1 | CLI 命令行框架 |
| `python-dotenv` | >=1.1.1 | `.env` 环境变量加载 |

---

## License

MIT
