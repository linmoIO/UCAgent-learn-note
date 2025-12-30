# Prompt 工程深度解析

**省流版**：

本文深入解析 UCAgent 的 Prompt 设计，分为两层：
- **System Prompt**（约 800 词）：
  - 定义 LLM 的角色、目标、工作方式
  - 强调发现 bug 而不是掩盖 bug
  - 正向激励机制："发现越多满足感越强"
- **Stage Task Prompt**（动态生成）：
  - 每个 Stage 的具体任务描述
  - 批处理技巧：动态内容、进度提示

Bug 发现机制：优先怀疑设计，不修改 DUT；保留 Fail 用例作为证据；基于源码进行深度分析

核心设计：通过工具动态获取任务，Prompt 不包含 Stage 列表。

---

## 📋 1. Prompt 的整体结构

UCAgent 的 Prompt 分为两个层次：

1. **System Prompt**：定义 LLM 的角色、目标和工作方式（约 800 词）
2. **Stage Task Prompt**：定义每个阶段的具体任务（动态生成）

### 1.1 Prompt 统计

**配置文件位置**：`lang/zh/config/default.yaml`

- **System Prompt**：第 19-87 行（约 800 词）
- **Stage 配置**：第 93-725 行（10 个阶段，约 2000 词）
- **工具描述**：动态生成（约 140 词）

---

## 🎭 2. System Prompt 完整解析

System Prompt 是 UCAgent 的核心，它定义了 LLM 的"人格"和工作方式。

### 2.1 角色定义（第1-2段）

**位置**：`default.yaml:19-24`

```
你是一位资深的芯片验证工程师和AI测试专家，具有INTJ和INTP人格，
专门从事数字电路的功能验证工作，非常擅长使用python进行验证。
你具备深厚的硬件设计理解能力，还具有软件测试方法论知识，
以及基于现代验证框架的实践经验。
```

**设计意图**：
- 让 LLM 进入专家模式,提高输出质量
- 人格特质（INTJ/INTP）强调理性、逻辑、追求完美、注重细节
- 技能栈：Python + 硬件验证 + 测试方法论

### 2.2 核心目标与动机（第3段）

**位置**：`default.yaml:25-31`

```
你非常优秀，能发现验证中的所有bug和潜在隐患，
能基于源代码进行bug详细分析并给出修复建议。
你不惧怕测试用例Fail，因为Fail可能意味着bug，
这是发现{DUT}中bug的基础。
发现bug是你一直追求的目标，发现的越多你获得的满足感越强。
```

**设计要点**：
- 强调 bug 发现为核心目标,避免 LLM 掩盖问题
- 不怕 Fail,避免 LLM 为了通过测试而修改逻辑
- 要求基于源码进行深度分析,不是简单修复
- 正向激励机制:"发现越多满足感越强"

这段 Prompt 解决了如何让 LLM 主动发现 bug 而不是掩盖 bug 的问题。

### 2.3 工作流组织（第6-7段，最重要）

**位置**：`default.yaml:38-57`

```
工作流（Mission）组织结构：
- 工作流由多个stage组成，每个stage包含具体的task描述，需要按顺序完成

子stage处理机制:
- 如果stage中包含子stage，必须按顺序逐一完成每个子stage
- 子stage完成后，使用Check工具检测当前子stage是否达到完成标准
- 所有子stage完成后，再检测父stage（upper_stage）是否达到完成标准
- 父stage完成后，使用Complete工具进入下一个主要阶段

完成顺序举例:
- 如果stage 3包含子阶段，完成顺序是：3.1 → 3.2 → 3.3 → 3（父stage最后完成）
- 如果stage 3只是分组容器，则只有子任务：3.1、3.2、3.3，没有独立的任务3
```

**设计要点**：
- 解释 Stage 嵌套机制和父子关系的执行顺序
- 明确工具使用时机：Check 和 Complete 的调用时机
- 提供具体示例帮助 LLM 理解抽象概念

### 2.4 工作原则（第8段）

**位置**：`default.yaml:58-75`

```
工作原则：
- 按步骤有序进行，每步完成后用Check工具验证
- 你会根据CurrentTips的要求完成该阶段的所有任务，不会提前进行后续阶段的工作
- 测试失败时，优先怀疑是芯片设计问题，不是测试问题
- 深入分析发现的问题
- 触发bug对应的测试用例必须 Fail，不能误报为 Pass
- 发现bug要基于源码进行详细分析：什么问题、为什么出现、如何修复
```

**设计要点**：
- 不提前工作：避免跳过阶段
- 优先怀疑设计：避免掩盖 bug
- 保留 Fail 用例：记录 bug 证据
- 源码分析：要求基于 Verilog/Scala 等源码进行深度分析

### 2.5 工具引导（第9段）

**位置**：`default.yaml:76-86`

```
必须使用的工具：
- `CurrentTips`: 获取当前步骤的具体指导
- `Check`: 检查当前步骤是否完成
- `Complete`: 完成当前步骤，进入下一步
- `ReadTextFile`: 读取文件内容
- 其他文件操作和搜索工具按需使用

现在调用`CurrentTips`，开始你的验证工作！
```

**设计意图**：
- **明确告知有工具可用**：引导 LLM 调用工具（Acting）
- **要求按步骤执行**：引导 LLM 推理（Reasoning）
- **强调工具反馈**：引导 LLM 观察结果（Observation）
- **最后一句**：直接要求 LLM 调用工具，启动 ReAct 循环的第一步

---

## 📝 3. Stage Task Prompt 设计

每个 Stage 的 Prompt 都是动态生成的，包含当前阶段的具体任务。

### 3.1 典型 Stage 结构

**示例**：API 测试阶段（`default.yaml:448-472`）

```yaml
- name: basic_api_implementation
  desc: "基础API实现"
  task:
    - "请参考文档：Guide_Doc/dut_api_instruction.md创建API"
    - "所有API以'api_'前缀开头"
    - "至少实现1个基础API函数"
    - "原则：让测试人员使用简单的API，而不是复杂的底层信号"
    - "每个API函数都必须要有注释"
    - "把结果写入{OUT}/tests/{DUT}_api.py"
    - "注意："
    - "  API函数操作的主要对象为env，非必要情况下不能直接操作dut"
    - "  针对时序电路，API函数需要考虑超时和等待机制"
  reference_files:
    - "Guide_Doc/dut_api_instruction.md"
  checker:
    - clss: "UnityChipCheckerDutApi"
```

**Prompt 设计要点**：

1. **明确目标**："基础API实现"
2. **分步指导**：参考文档 → 命名规范 → 实现要求
3. **约束条件**：前缀、注释、输出位置
4. **注意事项**：操作对象、超时机制
5. **参考文档**：提供详细指导
6. **验证方式**：Checker 自动检查

### 3.2 批处理 Stage 的 Prompt 技巧

**示例**：分批实现测试用例（`default.yaml:592-630`）

```yaml
task:
  - "第2步：将以下空的测试模板填充为可执行代码": "{LIST_CURRENT_CASES}"
  - "第6步：完成上述所有测试用例后，立即用RunTestCases('{TEST_BATCH_RUN_ARGS}')检查运行结果"
  - "完成进度：{COMPLETED_CASES}/{TOTAL_CASES}个测试用例已完成"
```

**关键设计**：
- **动态内容**：`{LIST_CURRENT_CASES}` 由 Checker 动态生成
- **进度提示**：`{COMPLETED_CASES}/{TOTAL_CASES}` 让 LLM 知道进度
- **立即验证**：完成后立即运行测试

这种设计允许 LLM 分批完成大量测试用例，避免一次性生成过多代码。

---

## 🐛 4. Prompt 引导 bug 发现的机制

### 4.1 核心 Prompt 设计

**位置**：`default.yaml:562-577`

```
重要：测试失败时，优先怀疑是芯片设计问题，不要急于修改测试。
不要尝试修复{DUT}中的bug，而是记录和分析bug。
触发bug对应的测试用例必须 Fail，不能误报为 Pass。
发现bug要基于源码进行详细分析：什么问题、为什么出现、如何修复。
```

### 4.2 设计原理

LLM 的默认行为倾向于"让测试通过",可能会修改测试用例来掩盖 bug、修改 DUT 代码或误报 bug 为正常行为。

**解决方案**：通过 Prompt 明确告知优先怀疑设计（测试失败 → 先怀疑 DUT 有 bug）、不要修复 DUT（只记录和分析，不修改）、保留 Fail 用例（作为 bug 证据）、源码分析（基于 Verilog/Scala 源码进行深度分析）。

### 4.3 实际效果

通过这种 Prompt 设计，LLM 会发现测试失败后，读取 DUT 源码，分析失败原因，生成详细的 bug 报告（包括什么问题、为什么出现、如何修复）。

---

## 📝 5. 总结

### 5.1 Prompt 的核心作用

1. **定义角色**：让 LLM 成为"验证工程师"
2. **设定目标**：发现 bug，而不是掩盖 bug
3. **控制流程**：通过工具动态获取任务，按顺序执行
4. **引导行为**：明确工作原则和注意事项

### 5.2 设计特点

1. **动态获取 Stage**：Prompt 不包含所有 Stage，通过工具动态获取
2. **正向激励**：强调发现 bug 的满足感
3. **明确约束**：不修改 DUT，保留 Fail 用例
4. **源码分析**：要求基于源码进行深度分析

---

**下一章**： [03-key-algorithms.md](./03-key-algorithms.md)
