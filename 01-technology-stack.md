# UCAgent 技术栈深度解析

**省流版**：

本文详细讲解 UCAgent 的技术栈选型和实现细节，从上到下分为 5 层：
- **UCAgent 应用层**：Stage 配置 + Checker 验证
- **LangGraph 层**：提供 ReAct Agent 框架
- **LLM 层**：支持 OpenAI/Anthropic/Google 多模型切换
- **验证框架层**：pytest 组织测试 + toffee 提供硬件仿真接口
- **仿真器层**：Verilator 执行 RTL 仿真

另外，UCAgent 可选集成 MCP Server，暴露给 Claude Code/Qwen Code 等调用。

重点说明各层的技术选型理由、核心功能、源码位置、以及层与层之间的交互方式。

---

## 1. 技术栈全景图

```
┌─────────────────────────────────────────────────────────────┐
│                    UCAgent 验证系统                          │
│              (应用层: Stage配置 + Checker验证)               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  LangGraph ReAct Agent                       │
│                  (推理-行动-观察循环)                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    LLM 大语言模型                            │
│          (OpenAI / Anthropic / Google 多模型支持)            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  pytest + toffee 验证框架                    │
│              (测试组织 + 硬件仿真接口)                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  Verilator 仿真器                            │
│                  (RTL 硬件行为模拟)                          │
└─────────────────────────────────────────────────────────────┘
```

**可选集成**：MCP Server (FastMCP) 可将 UCAgent 暴露给 Claude Code / Qwen Code 等调用。

---

## 2. 核心技术栈详解

### 2.1 UCAgent 应用层

**作用**: 定义验证流程和检查逻辑,是整个系统的最上层

**核心组件**:
1. **Stage 配置系统** (源码: `stage/vstage.py`)
2. **Checker 验证系统** (源码: `checkers/base.py`)
3. **工具系统** (源码: `tools/`)

**Stage 配置示例** (`lang/zh/config/default.yaml:93-725`):
```yaml
stages:
  - name: "理解规格"
    prompt: "请仔细阅读规格文档..."
    checker: "SpecChecker"

  - name: "编写测试"
    prompt: "根据规格编写pytest测试..."
    checker: "TestChecker"
```

**Checker 基类** (`checkers/base.py:14-260`):
```python
class Checker:
    def check(self) -> bool:
        """检查当前Stage是否完成"""
        raise NotImplementedError
```

**关键特性**:
- **配置驱动**: 通过YAML定义验证流程
- **可扩展**: 支持自定义Stage和Checker
- **多语言**: 支持中文/英文配置

---

### 2.2 LangGraph (Agent 框架) 层

**作用**: 提供 ReAct Agent 的实现框架,管理推理-行动循环

**技术选型**: `langgraph` (LangChain 生态的图执行引擎)

**核心函数**: `create_react_agent`

**源码位置**: `verify_agent.py:255-261`

```python
from langgraph.prebuilt import create_react_agent

self.agent = create_react_agent(
    model=self.model,              # LLM 模型
    tools=self.test_tools,         # 工具列表
    checkpointer=MemorySaver(),    # 对话历史保存
    pre_model_hook=message_manage_node,  # 消息管理钩子
    state_schema=State,            # 状态模式
)
```

**ReAct 循环机制**:
```
1. Reasoning (推理)
   ↓
   LLM 分析当前状态,决定下一步行动
   ↓
2. Acting (行动)
   ↓
   调用工具 (ReadTextFile, Check, Complete 等)
   ↓
3. Observation (观察)
   ↓
   接收工具返回结果
   ↓
   回到步骤 1,继续循环...
```

**关键特性**:
- **工具绑定**: 自动将工具列表转换为 LLM 可调用的函数
- **错误处理**: 自动处理工具调用失败并重试
- **消息压缩**: 通过 `pre_model_hook` 管理上下文长度

---

### 2.3 LLM (大语言模型) 层

**作用**: 提供智能推理能力,理解任务、生成代码、分析结果

**支持的LLM提供商** (源码: `util/models.py:94-116`):

| 提供商 | model_type | LangChain包 | 支持模型示例 |
|--------|-----------|-------------|-------------|
| **OpenAI** | `openai` | `langchain-openai` | GPT-4, GPT-3.5, GPT-4o |
| **Anthropic** | `anthropic` | `langchain-anthropic` | Claude 3.5 Sonnet, Claude 3 Opus |
| **Google** | `google_genai` | `langchain-google-genai` | Gemini Pro, Gemini Ultra |

**模型切换方式**:
```yaml
# config.yaml 中配置
model_type: "openai"  # 或 "anthropic" / "google_genai"

# OpenAI 配置
openai:
  openai_api_key: "your-key"
  openai_api_base: "https://api.openai.com/v1"
  model_name: "gpt-4"

# Anthropic 配置
anthropic:
  api_key: "your-key"
  model: "claude-3-5-sonnet-20241022"

# Google GenAI 配置
google_genai:
  google_api_key: "your-key"
  model: "gemini-pro"
```

**源码实现** (`util/models.py:94-116`):
```python
def get_chat_model(cfg: Config, callbacks: Any = None) -> Any:
    model_type = cfg.get_value("model_type", "openai")
    func = "get_chat_model_%s" % model_type  # 动态调用

    # 支持: get_chat_model_openai / anthropic / google_genai
    if func in globals():
        return globals()[func](cfg, callbacks, rate_limiter)
```

**关键特性**:
- **统一接口**: 所有LLM通过LangChain统一调用
- **速率限制**: 内置 `InMemoryRateLimiter` 防止API超限
- **流式输出**: 支持 streaming 模式实时显示推理过程

---

### 2.4 验证框架层

#### 2.4.1 pytest (测试框架)

**作用**: 组织和执行测试用例,提供断言和报告机制

**技术选型**: `pytest` (Python 标准测试框架)

**核心功能**:
- **测试发现**: 自动发现 `test_*.py` 文件中的测试函数
- **Fixture 机制**: 提供测试环境的初始化和清理
- **参数化测试**: 支持同一测试用例的多组输入
- **断言增强**: 提供详细的断言失败信息

**pytest测试代码的生成机制** (关键):
- **由LLM生成**: Agent在执行过程中,根据SPEC和Stage Prompt生成测试代码
- **写入文件**: 通过 `WriteTextFile` 工具写入 `test_*.py` 文件
- **Checker调用**: Checker通过 `subprocess` 调用 `pytest` 执行测试
- **结果反馈**: pytest输出返回给LLM,用于下一轮推理

**在 UCAgent 中的使用**:
```python
# 测试用例示例 (由LLM生成)
def test_adder_basic(env):
    """测试加法器基本功能"""
    # 设置输入
    env.io.a.value = 5
    env.io.b.value = 3
    env.dut.Step(1)

    # 断言输出
    assert env.io.sum.value == 8, f"Expected 8, got {env.io.sum.value}"
```

**Fixture 示例**:
```python
@pytest.fixture(scope="function")
def dut(request):
    """创建 DUT 实例"""
    dut = create_dut(request)
    yield dut
    dut.Finish()  # 清理
```

#### 2.4.2 toffee (硬件验证框架)

**作用**: 提供硬件仿真的 Python 接口和覆盖率收集

**技术选型**: `toffee` + `pytoffee` (专为硬件验证设计)

**核心功能**:
1. **信号访问**: 通过 Python 读写 RTL 信号
2. **时钟控制**: 驱动时序电路的时钟信号
3. **覆盖率收集**:
   - 代码行覆盖率 (Line Coverage)
   - 功能覆盖率 (Functional Coverage)
4. **Bundle 封装**: 将相关信号组织为 Python 对象

**信号访问示例**:
```python
# 读信号
value = dut.io.output.value

# 写信号
dut.io.input.value = 42

# 时钟驱动
dut.Step(1)  # 前进 1 个时钟周期
```

**功能覆盖率示例**:
```python
from toffee import CovGroup

# 定义覆盖组
cov_group = CovGroup("BasicOps")
cov_group.add_watch_point(
    "ADD",
    lambda: dut.io.op.value == 0,  # 检查条件
    lambda: dut.io.result.value == expected  # 验证结果
)
```

**Bundle 封装示例**:
```python
import toffee

class IOBundle(toffee.Bundle):
    a, b, sum = toffee.Signals(3)

# 绑定到 DUT
io = IOBundle.from_prefix(dut, "io_")
io.a.value = 5  # 等价于 dut.io_a.value = 5
```

---

### 2.5 仿真器层

**作用**: 执行 RTL 代码,模拟硬件行为

**技术选型**:
- **主要**: Verilator (开源高性能仿真器)
- **支持**: VCS, ModelSim 等商业仿真器

**工作原理**:
1. 将 Verilog/SystemVerilog 编译为 C++
2. 编译为共享库 (.so/.dll)
3. Python 通过 `pytoffee` 加载并调用

**性能优势**:
- Verilator 比解释型仿真器快 10-100 倍
- 支持多线程并行仿真

---

## 3. 层间交互机制

**LLM ↔ LangGraph**:
- LLM 输出工具调用指令
- LangGraph 解析并执行工具
- 工具结果返回给 LLM

**LangGraph ↔ UCAgent**:
- UCAgent 提供工具列表 (Check, Complete 等)
- LangGraph 管理工具调用生命周期

**UCAgent ↔ pytest**:
- UCAgent 通过 `subprocess` 调用 pytest
- pytest 返回测试结果和覆盖率

**pytest ↔ toffee**:
- pytest 加载 toffee fixture
- toffee 初始化 DUT 并提供信号接口

**toffee ↔ Verilator**:
- toffee 通过 `pytoffee` 加载仿真器
- Verilator 执行 RTL 仿真

**MCP ↔ Code Agent**:
- Code Agent 通过HTTP调用MCP Server
- MCP Server 将请求转发给UCAgent工具
- 实时推送执行日志到Code Agent

---

## 4. 总结

### 4.1 技术栈优势

1. **多模型支持**: OpenAI/Anthropic/Google,灵活切换
2. **MCP集成**: 可与Claude Code/Qwen Code等深度协作
3. **完整工具链**: pytest + toffee + Verilator,覆盖全流程
4. **高性能仿真**: Verilator 提供接近原生速度的仿真性能

### 4.2 技术选型理由

- **LangGraph**: 成熟的 ReAct Agent 框架,开箱即用
- **pytest**: Python 生态标准测试框架,易于集成
- **toffee**: 专为硬件验证设计,提供覆盖率收集
- **Verilator**: 开源高性能,支持大规模设计

---

## 下一章

[02-prompt-engineering.md](./02-prompt-engineering.md) - 深入解析Prompt设计