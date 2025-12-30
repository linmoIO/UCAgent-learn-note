# UCAgent 核心概念

**省流版**：

本文讲解 UCAgent 的三大核心概念：
- **Stage**（定义"做什么"）：通过 YAML 配置验证流程，支持嵌套结构
- **Agent**（执行"怎么做"）：LLM 通过 ReAct 循环（推理 → 行动 → 观察）调用工具完成任务
- **Checker**（验证"做完了吗"）：执行 pytest 等检查确认完成

以及配置说明：
  - 必须提供：工作目录、LLM API 配置
  - 已提供：System Prompt、Stage 配置、工具、Checker
  - 可定制：Stage 流程、Prompt、Checker、工具

核心设计：配置驱动、动态获取任务、完全自动化闭环。

---

## 1. UCAgent 是什么？

UCAgent 是一个基于大语言模型的自动化硬件验证系统,让 LLM 像人类验证工程师一样自动完成芯片验证的全流程。

### 1.1 技术栈概览

UCAgent 采用 5 层架构：

```
应用层: UCAgent (Stage配置 + Checker验证)
   ↓
Agent层: LangGraph ReAct Agent (推理-行动-观察循环)
   ↓
LLM层: ChatOpenAI (大语言模型推理)
   ↓
验证框架层: pytest + toffee (测试执行和硬件仿真)
   ↓
仿真器层: Verilator (RTL硬件行为模拟)
```

**核心依赖**：
- **LangGraph**: `create_react_agent` 提供 ReAct 框架
- **pytest + toffee**: Python 测试框架 + 硬件验证接口
- **Verilator**: 开源高性能 RTL 仿真器

详细说明参见 [01-technology-stack.md](./01-technology-stack.md)

### 1.2 工作方式对比

**传统人工验证**：
```
人类工程师读文档 → 编写测试 → 执行测试 → 分析覆盖率 → 补充测试 → 循环...
```

**UCAgent自动验证**：
```
配置验证流程 → LLM自动完成所有步骤 → 输出验证报告
```

LLM 自动完成：
1. 读取设计文档，理解芯片功能
2. 生成测试用例代码
3. 执行测试并收集覆盖率
4. 分析结果，补充测试
5. 发现bug并生成分析报告

### 1.3 使用 UCAgent 的配置说明

#### 1.3.1 必须由使用者提供的配置

**工作目录结构**：
```
workspace/
├── README.md              # 必须：DUT 功能说明文档
├── {DUT}/                 # 必须：DUT 源码目录（Verilog/Scala等）
│   ├── *.v / *.scala      # RTL 设计文件
│   └── __init__.py        # Python 接口文件（可选）
└── Guide_Doc/             # 可选：额外的指导文档
    └── *.md
```

**命令行参数**：
```bash
ucagent <workspace> <dut_name> [选项]
```
- `workspace`: 工作目录路径（必须）
- `dut_name`: 被测设计名称（必须）
- `--config`: 自定义配置文件路径（可选）
- `--interaction-mode`: 交互模式（可选，默认 standard）

**LLM API 配置**（必须在配置文件或环境变量中提供）：
```yaml
# OpenAI 配置
model_type: "openai"
openai:
  openai_api_key: "your-key"
  openai_api_base: "https://api.openai.com/v1"
  model_name: "gpt-4"

# 或 Anthropic 配置
model_type: "anthropic"
anthropic:
  api_key: "your-key"
  model: "claude-3-5-sonnet-20241022"
```

#### 1.3.2 UCAgent 已经提供的配置

**默认配置文件**：`UCAgent/ucagent/lang/zh/config/default.yaml`

**已提供的内容**：
1. **System Prompt**（第 19-87 行）
   - 定义 LLM 的角色和工作方式
   - 无需修改，开箱即用

2. **Stage 配置**（第 93-725 行）
   - 10 个标准验证阶段
   - 每个阶段的任务描述、参考文件、输出文件
   - 适用于大多数芯片验证场景

3. **工具配置**
   - 文件操作工具（ReadTextFile, WriteTextFile 等）
   - 任务管理工具（Check, Complete 等）
   - 测试执行工具（RunTestCases 等）

4. **Checker 配置**
   - UnityTestChecker（pytest 测试）
   - BashScriptChecker（脚本执行）
   - HumanChecker（人工审查）

#### 1.3.3 支持自定义的配置

**可定制的配置项**：

1. **Stage 流程定制**
   ```yaml
   # 在自定义配置文件中覆盖默认 Stage
   stage:
     - name: custom_stage_1
       desc: "自定义阶段描述"
       task:
         - "自定义任务步骤1"
         - "自定义任务步骤2"
       reference_files:
         - "custom_doc.md"
       output_files:
         - "custom_output/*.py"
       checker:
         - clss: "CustomChecker"
   ```

2. **System Prompt 定制**
   ```yaml
   # 覆盖默认的 System Prompt
   mission:
     system_prompt: |
       你是一个专门针对 RISC-V 处理器的验证专家...
   ```

3. **Checker 定制**
   ```python
   # 创建自定义 Checker
   from ucagent.checkers.base import UCChecker

   class CustomChecker(UCChecker):
       def do_check(self, *args, **kwargs):
           # 自定义检查逻辑
           return True, "检查通过"
   ```

4. **工具定制**
   ```python
   # 添加自定义工具
   from ucagent.tools.uctool import UCTool

   class CustomTool(UCTool):
       name = "CustomTool"
       description = "自定义工具描述"

       def _run(self, *args, **kwargs):
           # 自定义工具逻辑
           return "工具执行结果"
   ```

5. **模型参数定制**
   ```yaml
   # 调整 LLM 参数
   openai:
     temperature: 0.7        # 创造性（0-1）
     max_tokens: 4096        # 最大输出长度
     top_p: 0.9              # 采样参数
   ```

**配置优先级**：
```
命令行参数 > 自定义配置文件 > 默认配置文件
```

---

## 2. 三大核心概念

UCAgent 的架构基于三个核心概念：**Stage**、**Agent**、**Checker**。

### 2.1 Stage（阶段）- "做什么"

**Stage 定义了验证流程的每一步需要做什么**。

#### 配置示例

```yaml
# 配置文件：default.yaml
stage:
  - name: requirement_analysis_and_planning
    desc: "需求分析与验证规划"
    task:
      - "第1步：读取README.md，理解验证要求"
      - "第2步：确定验证目标"
      - "第3步：识别风险点"
    reference_files:
      - "README.md"
    output_files:
      - "unity_test/verification_plan.md"
    checker:
      - clss: "UnityTestChecker"
```

#### Stage 的组成部分

1. **name**: 阶段名称（唯一标识）
2. **desc**: 阶段描述（告诉LLM这个阶段的目标）
3. **task**: 任务列表（具体的执行步骤）
4. **reference_files**: 必须读取的参考文件
5. **output_files**: 必须生成的输出文件
6. **checker**: 验证完成的检查器

#### Stage 的嵌套结构

Stage 支持嵌套，可以包含子 Stage：

```yaml
stage:
  - name: test_implementation
    desc: "测试实现"
    stage:  # 子阶段
      - name: basic_test
        desc: "基础测试"
      - name: corner_case_test
        desc: "边界测试"
```

**执行顺序**：子Stage → 父Stage
- 先完成 `basic_test`
- 再完成 `corner_case_test`
- 最后完成 `test_implementation`

### 2.2 Agent（执行者）- "怎么做"

**Agent 是 LLM + LangGraph ReAct 框架，通过 ReAct 循环来完成 Stage 定义的任务**。

#### Agent 的构成

**源码位置**: `verify_agent.py:255-261`

```python
from langgraph.prebuilt import create_react_agent

self.agent = create_react_agent(
    model=self.model,              # LLM 模型 (ChatOpenAI)
    tools=self.test_tools,         # 工具列表
    checkpointer=MemorySaver(),    # 对话历史保存
    pre_model_hook=message_manage_node,  # 消息管理
    state_schema=State,            # 状态模式
)
```

**关键点**:
- `create_react_agent` 是 LangGraph 提供的**预构建 ReAct Agent**
- Agent 的行为模式由 **LangGraph 框架内部实现**，不是 UCAgent 自己写的
- UCAgent 只负责提供工具列表和 System Prompt

#### ReAct 循环机制

ReAct = **Rea**soning（推理）+ **Act**ing（行动）

```
┌─────────────────────────────────────┐
│  1. Reasoning (推理)                 │
│     LLM: "我需要先读README了解功能"   │
│     [由 LLM 根据 Prompt 决定]        │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  2. Acting (行动)                    │
│     调用工具: ReadTextFile("README") │
│     [由 LangGraph 自动执行]          │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  3. Observation (观察)               │
│     收到: README内容                 │
│     [由 LangGraph 返回给 LLM]        │
└─────────────────────────────────────┘
              ↓
         回到步骤1，继续推理...
```

**Agent 行为的控制层次**:
- **ReAct 循环本身**: 由 LangGraph 框架控制（固化在 `create_react_agent` 中）
- **具体行动内容**: 由 LLM 根据 System Prompt 决定
- **可用工具**: 由 UCAgent 提供（Check, Complete, ReadTextFile 等）

#### 实际案例

假设当前 Stage 的任务是"生成API测试"：

```
轮次1：
  Reasoning: 我需要先了解芯片功能
  Acting: ReadTextFile("README.md")
  Observation: [README内容]

轮次2：
  Reasoning: 我需要了解API接口定义
  Acting: ReadTextFile("Adder/__init__.py")
  Observation: [接口定义]

轮次3：
  Reasoning: 现在可以生成测试代码了
  Acting: WriteTextFile("tests/test_api.py", code)
  Observation: "写入成功"

轮次4：
  Reasoning: 需要检查测试是否通过
  Acting: Check()
  Observation: "测试通过"

轮次5：
  Acting: Complete()
  Observation: "进入下一阶段"
```

#### pytest 测试代码的生成机制

pytest 测试代码由 LLM 自动生成。

**完整流程**:
```
1. Stage 配置定义任务
   ↓
2. LLM 根据 Prompt 和任务描述生成测试代码
   ↓
3. LLM 调用 WriteTextFile 工具写入测试文件
   ↓
4. LLM 调用 Check 工具验证
   ↓
5. Checker 通过 subprocess 执行 pytest
   ↓
6. pytest 加载测试文件并执行
   ↓
7. toffee 提供硬件仿真接口
   ↓
8. Verilator 执行 RTL 仿真
```

**源码证据**:
- **测试代码生成**: LLM 通过 `WriteTextFile` 工具写入 (由 Prompt 引导)
- **pytest 执行**: `UnityTestChecker.do_check()` 调用 `subprocess.run(["pytest", ...])`
- **配置位置**: `default.yaml` 中的 Stage 配置指定使用哪个 Checker

### 2.3 Checker（检查器）- "做完了吗"

**Checker 验证当前 Stage 是否完成**。

#### Checker 的工作流程

```python
class UnityTestChecker(Checker):
    def do_check(self):
        # 1. 检查 reference_files 是否都已读取
        if not all_files_read:
            return False, "还有文件未读取"

        # 2. 检查 output_files 是否都已生成
        if not all_files_generated:
            return False, "还有文件未生成"

        # 3. 执行 pytest 测试
        result = run_pytest()

        # 4. 返回结果
        if result.passed:
            return True, "测试通过"
        else:
            return False, "测试失败：详细信息..."
```

#### Checker 的类型

1. **UnityTestChecker**: 执行 pytest 测试
2. **BashScriptChecker**: 执行 bash 脚本
3. **ToffeeReportChecker**: 检查覆盖率报告
4. **HumanChecker**: 需要人工确认

#### Checker 的反馈机制

Checker 的返回值会直接反馈给 LLM：

```python
# 失败时
{
  "error": "输出文件未生成",
  "failed_patterns": ["unity_test/verification_plan.md"]
}

# 成功时
{
  "success": "检查通过",
  "message": "所有要求已满足"
}
```

LLM 根据反馈决定下一步行动：
- 如果失败 → 分析原因 → 修复 → 重新 Check
- 如果成功 → 调用 Complete → 进入下一个 Stage

---

## 3. 三者的协同工作

### 3.1 完整工作流程

```
启动 UCAgent
    ↓
加载配置，解析所有 Stage
    ↓
┌─────────────────────────────────────┐
│  进入 Stage 1: 需求分析              │
└─────────────────────────────────────┘
    ↓
StageManager 返回当前 Stage 的任务
    ↓
┌─────────────────────────────────────┐
│  Agent (LLM) 开始 ReAct 循环         │
│  - 推理：需要读 README               │
│  - 行动：ReadTextFile("README")     │
│  - 观察：收到 README 内容            │
│  - 推理：需要生成验证计划             │
│  - 行动：WriteTextFile("plan.md")   │
│  - 推理：需要检查是否完成             │
│  - 行动：Check()                    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Checker 执行检查                    │
│  - 检查文件是否读取                  │
│  - 检查文件是否生成                  │
│  - 返回结果给 LLM                    │
└─────────────────────────────────────┘
    ↓
如果通过 → Agent 调用 Complete()
    ↓
StageManager 推进到 Stage 2
    ↓
重复上述流程...
```

### 3.2 Stage 的加载和执行顺序控制机制

#### Stage 的加载机制

Stage 由配置文件 (`UCAgent/ucagent/lang/zh/config/default.yaml`) 定义,源码负责解析。

**加载流程**:
```
1. UCAgent 启动
   ↓
2. 加载配置文件 default.yaml (verify_agent.py:112)
   ↓
3. StageManager.init_stage() 解析配置 (vmanager.py:261-310)
   ↓
4. parse_vstage() 递归解析 Stage 树 (vstage.py:367-404)
   ↓
5. get_substages() 扁平化为线性列表 (vstage.py:300-306)
   ↓
6. 得到可执行的 Stage 列表 (只包含叶子节点)
```

**源码位置**:
- 配置加载: `verify_agent.py:112` → `get_config()`
- Stage 解析: `vmanager.py:263` → `get_root_stage()`
- 递归解析: `vstage.py:367-404` → `parse_vstage()`
- 扁平化: `vstage.py:300-306` → `get_substages()`

#### Stage 执行顺序的控制机制

StageManager 通过 `stage_index` 维护当前进度,控制 Stage 的执行顺序。

**控制机制**:
```
StageManager 维护:
- self.stages: Stage 列表 (扁平化后的)
- self.stage_index: 当前执行到第几个 Stage

Agent 通过工具动态获取:
- CurrentTips: 获取当前 Stage 的任务
- Complete: 完成当前 Stage，推进 stage_index
```

**关键设计**: Prompt **不包含** 具体的 Stage 列表！

System Prompt 只告诉 LLM:
```
工作流由多个 stage 组成，每个 stage 包含具体的 task 描述，需要按顺序完成。
使用工具 `CurrentTips` 获取当前阶段的详细任务指导。
```

#### 动态获取机制

**源码位置**: `vmanager.py:349-368`

```python
def get_current_tips(self):
    # 获取当前 Stage
    cstage = self.stages[self.stage_index]

    # 返回当前 Stage 的任务
    tips = OrderedDict()
    tips["mission"] = self.mission.name
    tips["current_stage"] = {
        "index": self.stage_index,
        **cstage.detail(),  # task, reference_files, output_files
    }
    tips["process"] = f"{self.stage_index}/{len(self.stages)}"
    return tips
```

#### 推进机制

**源码位置**: `vmanager.py:522-558`

```python
def complete(self, timeout):
    # 1. 执行最终检查
    ck_pass, ck_info = self.stages[self.stage_index].do_check(...)

    if ck_pass:
        # 2. 完成当前阶段
        self.stages[self.stage_index].on_complete()

        # 3. 推进到下一阶段
        self.next_stage()  # stage_index += 1

        # 4. 初始化下一阶段
        self.stages[self.stage_index].on_init()
```

#### 设计优势

1. **解耦**: Agent 不需要知道所有 Stage，只关注当前任务
2. **灵活**: 可以修改配置文件，无需修改代码或 Prompt
3. **可控**: StageManager 完全控制顺序，Agent 无法跳过
4. **可恢复**: 保存 stage_index 可实现断点续传

---

## 4. 总结

### 核心要点

1. **技术栈**: UCAgent = LLM + LangGraph + pytest + toffee + Verilator
2. **三大核心**：
   - **Stage**: 定义"做什么"（配置在 `default.yaml` 中）
   - **Agent**: 执行"怎么做"（LangGraph ReAct Agent + LLM）
   - **Checker**: 验证"做完了吗"（继承自 `base.Checker`）

3. **关键控制机制**：
   - **Agent 行为**: 由 LangGraph 框架控制 ReAct 循环，Prompt 引导具体行动
   - **Stage 加载**: 配置文件定义，源码解析（递归 + 扁平化）
   - **Stage 顺序**: StageManager 维护 stage_index，Agent 动态获取
   - **测试代码**: LLM 生成，Checker 执行 pytest

4. **设计特点**：
   - Prompt 不包含 Stage 列表，通过工具动态获取
   - 配置与代码分离，灵活可扩展
   - 完整的自动化闭环：生成 → 执行 → 验证 → 反馈

---

## 下一章

[01-technology-stack.md](./01-technology-stack.md) - 技术栈详解