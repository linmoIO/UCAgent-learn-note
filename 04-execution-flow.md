# 执行流程深度追踪

**省流版**：

本文追踪 UCAgent 从启动到完成的完整调用链，标注每个关键函数的位置和作用：
- **启动流程**：`cli.py:main()` → 参数解析 → VerifyAgent 初始化
- **初始化阶段**：
  - 加载配置、初始化 LLM
  - 创建工具列表、初始化 StageManager
  - 创建 ReAct Agent
- **主循环执行**：`run_loop()` → `one_loop()` → `get_current_tips()` → `do_work()`
  - **ReAct 循环**：推理 → 行动 → 观察（循环往复）
- **Checker 执行**：Check 工具调用 → `do_check()` → 返回结果
- **Complete 流程**：最终检查 → 完成当前阶段 → 推进到下一阶段
- **消息管理**：消息压缩机制、消息统计

---

## 1. 启动流程

### 1.1 命令行入口

**位置**：`cli.py:566`

```python
if __name__ == "__main__":
    main()
```

**调用链**：
```
main() (cli.py:549)
  └─ run() (cli.py:463)
      └─ VerifyAgent.__init__() (verify_agent.py:43)
          └─ agent.run() (verify_agent.py:522)
```

### 1.2 参数解析

**位置**：`cli.py:110-372`

```python
def get_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="UCAgent - UnityChip Verification Agent",
        prog="ucagent",
    )

    # 核心参数
    parser.add_argument("workspace", type=str)
    parser.add_argument("dut", type=str)
    parser.add_argument("--config", type=str)
    parser.add_argument("--interaction-mode", choices=["standard", "enhanced", "advanced"])
    # ... 更多参数
```

**关键参数**：
- `workspace`: 工作目录
- `dut`: 被测设计名称
- `--config`: 配置文件路径
- `--interaction-mode`: 交互模式（standard/enhanced/advanced）
- `--loop`: 立即开始循环
- `--mcp-server`: 启动MCP服务器

---

## 2. VerifyAgent 初始化

### 2.1 初始化流程概览

**位置**：`verify_agent.py:43-304`

```python
class VerifyAgent:
    def __init__(self, workspace, dut_name, output, ...):
        # 1. 加载配置
        self.cfg = get_config(config_file, cfg_override)

        # 2. 初始化LLM模型
        self.model = get_chat_model(self.cfg, callbacks)

        # 3. 初始化工具列表
        self.tool_list_base = [ReadTextFile, RoleInfo, ...]
        self.tool_list_file = [PathList, EditTextFile, ...]
        self.tool_list_task = stage_manager.new_tools()

        # 4. 初始化StageManager
        self.stage_manager = StageManager(...)
        self.stage_manager.init_stage()

        # 5. 创建ReAct Agent
        self.agent = create_react_agent(
            model=self.model,
            tools=self.test_tools,
            checkpointer=MemorySaver(),
            pre_model_hook=message_manage_node,
        )
```

### 2.2 配置加载

**位置**：`util/config.py`

```python
def get_config(config_file, cfg_override):
    # 1. 加载默认配置
    default_cfg = load_yaml("ucagent/lang/zh/config/default.yaml")

    # 2. 加载用户配置（如果存在）
    if config_file:
        user_cfg = load_yaml(config_file)
        default_cfg.update(user_cfg)

    # 3. 应用命令行覆盖
    if cfg_override:
        default_cfg.update(cfg_override)

    return default_cfg
```

**配置内容**：
- `mission`: 任务定义（名称、System Prompt）
- `stage`: 阶段配置（10个验证阶段）
- `tools`: 工具配置
- `conversation_summary`: 消息压缩配置

### 2.3 工具初始化

**位置**：`verify_agent.py:163-207`

```python
# 基础工具
self.tool_list_base = [
    ReadTextFile(workspace),              # 读文件
    RoleInfo(system_prompt),              # 角色信息
    SemanticSearchInGuidDoc(...),         # 语义搜索
    MemoryPut(), MemoryGet(),             # 记忆管理
]

# 文件操作工具
self.tool_list_file = [
    PathList(workspace),                  # 列出路径
    GetFileInfo(workspace),               # 获取文件信息
    SearchText(workspace),                # 搜索文本
    EditTextFile(workspace),              # 编辑文件
    DeleteFile(workspace),                # 删除文件
    # ... 更多文件工具
]

# 任务管理工具（由StageManager提供）
self.tool_list_task = [
    ToolCurrentTips(),                    # 获取当前任务提示
    ToolStatus(),                         # 获取状态
    ToolDoCheck(),                        # 执行检查
    ToolDoComplete(),                     # 完成阶段
    ToolRunTestCases(),                   # 运行测试
    ToolGoToStage(),                      # 跳转阶段
    ToolDoExit(),                         # 退出
]
```

---

## 3. StageManager 初始化

### 3.1 Stage 解析流程

**位置**：`stage/vmanager.py:261-310`

```python
def init_stage(self):
    # 1. 解析Stage配置
    self.root_stage = get_root_stage(self.cfg, self.workspace, self.tool_read_text)

    # 2. 扁平化Stage树（只保留叶子节点）
    self.stages = self.root_stage.get_substages()

    # 3. 添加参考文件
    if self.reference_files:
        for si, flist in self.reference_files.items():
            self.stages[si].add_reference_files(flist)

    # 4. 恢复历史状态
    for stage_idx, stage_info in stages_info.items():
        stage.set_fail_count(stage_info["fail_count"])
        stage.set_time_prev_cost(stage_info["time_cost"])

    # 5. 设置当前阶段
    self.stage_index = force_stage_index
    self.stages[self.stage_index].on_init()
```

### 3.2 Stage 树结构

**配置示例**（`default.yaml:93-725`）：

```yaml
stage:
  - name: requirement_analysis_and_planning
    desc: "需求分析与验证规划"
    task: [...]
    checker: [...]

  - name: dut_function_understanding
    desc: "DUT功能理解"
    stage:  # 嵌套子阶段
      - name: interface_analysis
        desc: "接口分析"
      - name: behavior_analysis
        desc: "行为分析"
```

**解析后的结构**：
```
Root Stage (组)
├── 1. requirement_analysis_and_planning (叶子)
├── 2. dut_function_understanding (组)
│   ├── 2.1 interface_analysis (叶子)
│   └── 2.2 behavior_analysis (叶子)
└── 3. ...

扁平化后 → [1, 2.1, 2.2, 3, ...]
```

---

## 4. 主循环执行

### 4.1 run() 方法

**位置**：`verify_agent.py:522-552`

```python
def run(self):
    self.pre_run()      # 预处理
    self.run_loop()     # 主循环

def pre_run(self):
    info("Verify Agent started at: " + time_str)
    self.check_pdb_trace()  # 检查是否需要进入调试模式
    return self

def run_loop(self, msg=None):
    while not self.is_exit():
        self.one_loop()              # 执行一轮
        if self.is_exit():
            break
        if self.is_break():          # 用户中断
            return
        if self._need_human:         # 需要人工输入
            return
        self.check_pdb_trace()

    info("Verify Agent finished")
    info(f"Total time taken: {time_cost}")
```

### 4.2 one_loop() 方法

**位置**：`verify_agent.py:554-592`

```python
def one_loop(self, msg=None):
    # 根据交互模式选择执行逻辑
    if self.interaction_mode == "advanced":
        return self.advanced_logic.advanced_one_loop(msg)
    elif self.interaction_mode == "enhanced":
        return self.enhanced_logic.enhanced_one_loop(msg)

    # 标准模式（默认）
    if msg:
        self.set_continue_msg(msg)

    # 执行一轮对话（带重试）
    while True:
        tips = self.get_current_tips()      # 获取当前任务提示
        if self.is_exit():
            return
        self.do_work(tips, self.get_work_config())  # 执行工作
        if not self._tool__call_error:      # 没有工具调用错误
            break
        if self.is_break():
            return

    self.invoke_round += 1
    return self
```

---

## 5. ReAct 循环详解

### 5.1 get_current_tips()

**位置**：`verify_agent.py:451-466`

```python
def get_current_tips(self):
    # 如果有工具调用错误，返回错误消息
    if self._tool__call_error:
        return {"messages": copy.deepcopy(self._tool__call_error)}

    # 获取当前阶段的任务提示
    tips = self._continue_msg or yam_str(self.stage_manager.get_current_tips())
    self._continue_msg = None

    # 构造消息
    msg = []
    if self._system_message:
        msg.append(SystemMessage(content=self._system_message))
        self._system_message = None
    msg.append(HumanMessage(content=tips))

    return {"messages": msg}
```

**StageManager.get_current_tips()**（`vmanager.py:349-368`）：

```python
def get_current_tips(self):
    if self.stage_index >= len(self.stages):
        return "Your mission is completed..."

    cstage = self.stages[self.stage_index]
    tips = OrderedDict()
    tips["mission"] = self.mission.name
    tips["current_stage"] = {
        "index": self.stage_index,
        **cstage.detail(),  # 包含task、reference_files、output_files
    }

    # 提示需要读取的参考文件
    ref_files = [k for k, v in cstage.reference_files.items() if not v]
    if ref_files:
        tips["notes"] = f"You need use tool: {self.tool_read_text.name} to read..."

    tips["process"] = f"{self.stage_index}/{len(self.stages)}"
    return make_llm_tool_ret(tips)
```

### 5.2 do_work()

**位置**：`verify_agent.py:684-690`

```python
def do_work(self, instructions, config):
    self._tool__call_error = []
    if self.stream_output:
        self.do_work_stream(instructions, config)
    else:
        self.do_work_values(instructions, config)
```

**do_work_values()**（`verify_agent.py:755-767`）：

```python
def do_work_values(self, instructions, config):
    last_msg_index = None
    # 流式处理Agent的输出
    for _, step in self.agent.stream(instructions, config, stream_mode=["values"]):
        if self._need_break:
            break

        index = len(step["messages"])
        if index == last_msg_index:
            continue
        last_msg_index = index

        msg = step["messages"][-1]
        self.check_tool_call_error(msg)     # 检查工具调用错误
        self.state_record_mesg(msg)         # 记录消息统计
        self.message_echo(msg.pretty_repr())  # 输出消息
```

---

## 6. Checker 执行流程

### 6.1 Check 工具调用

**位置**：`vmanager.py:145-177`

```python
class ToolDoCheck(ManagerTool):
    name: str = "Check"
    description: str = "执行当前阶段的验证检查..."

    def _run(self, timeout=0, run_manager=None) -> str:
        if timeout <= 0:
            timeout = self.get_call_time_out()
        return self.function(timeout)
```

**StageManager.tool_check()**（`vmanager.py:599-602`）：

```python
def tool_check(self, timeout):
    ret = make_llm_tool_ret(self.check(timeout))
    info("ToolCheck:\n" + ret)
    return ret

def check(self, timeout):
    # 执行当前阶段的检查
    ck_pass, ck_info = self.stages[self.stage_index].do_check(timeout=timeout)

    ret_data = OrderedDict({
        "check_info": ck_info,
        "check_pass": ck_pass,
    })

    if not ck_pass:
        ret_data["action"] = "Please fix the issues..."
    else:
        ret_data["message"] = "Congratulations! You can use 'Complete' tool."

    return ret_data
```

### 6.2 VerifyStage.do_check()

**位置**：`stage/vstage.py`（需要查看具体实现）

```python
def do_check(self, timeout, is_complete=False):
    # 遍历所有checker
    for checker in self.checkers:
        pass_flag, info = checker.do_check(timeout=timeout)
        if not pass_flag:
            return False, info

    return True, "All checks passed"
```

### 6.3 Checker 类型

**UnityTestChecker**（`checkers/base.py`）：
- 执行 pytest 测试
- 检查测试是否通过
- 返回测试结果和覆盖率

**BashScriptChecker**：
- 执行 bash 脚本
- 检查脚本返回值

**HumanChecker**：
- 等待人工审查
- 设置 `_need_human = True`

---

## 7. Complete 流程

### 7.1 Complete 工具调用

**位置**：`vmanager.py:179-200`

```python
class ToolDoComplete(ManagerTool):
    name: str = "Complete"
    description: str = "告诉管理器你已完成当前阶段..."

    def _run(self, timeout=0, run_manager=None) -> str:
        return self.function(timeout)
```

**StageManager.complete()**（`vmanager.py:522-558`）：

```python
def complete(self, timeout):
    # 1. 执行最终检查
    ck_pass, ck_info = self.stages[self.stage_index].do_check(
        timeout=timeout,
        is_complete=True
    )

    if ck_pass:
        # 2. 完成当前阶段
        self.stages[self.stage_index].on_complete()

        # 3. 进入下一阶段
        self.next_stage()

        if self.stage_index >= len(self.stages):
            # 所有阶段完成
            message = "All stages completed successfully..."
            self.all_completed = True
        else:
            # 初始化下一阶段
            message = f"Current stage index is now {self.stage_index}..."
            self.stages[self.stage_index].set_reached(True)
            self.stages[self.stage_index].on_init()
    else:
        message = "Stage not completed. Please check requirements."

    return {"complete": ck_pass, "message": message}
```

---

## 8. 消息历史管理

### 8.1 消息压缩机制

**位置**：`verify_agent.py:222-246`

```python
# 初始化消息管理节点
if self.use_uc_mode:
    message_manage_node = UCMessagesNode(
        max_summary_tokens=self.max_summary_tokens,
        max_keep_msgs=self.max_keep_msgs,
        model=self.sumary_model
    )
else:
    message_manage_node = SummarizationAndFixToolCall(
        max_tokens=self.max_token,
        max_summary_tokens=self.max_summary_tokens,
    )
```

**UCMessagesNode 工作原理**（`message/`）：
1. 监控消息总数
2. 超过 `max_keep_msgs` 时触发压缩
3. 保留最近 `tail_keep_msgs` 条消息
4. 将旧消息压缩为摘要

### 8.2 消息统计

**位置**：`verify_agent.py:692-699`

```python
def state_record_mesg(self, msg):
    if isinstance(msg, AIMessage):
        self._stat_msg_count_ai += 1
    elif isinstance(msg, ToolMessage):
        self._stat_msg_count_tool += 1
    elif isinstance(msg, SystemMessage):
        self._stat_msg_count_system += 1

    self.message_statistic.update_message(msg)
```

---

## 9. 完整执行流程图

```
启动
  ↓
cli.main()
  ↓
VerifyAgent.__init__()
  ├─ 加载配置 (default.yaml)
  ├─ 初始化 LLM 模型
  ├─ 初始化工具列表
  ├─ 初始化 StageManager
  │   ├─ 解析 Stage 配置
  │   ├─ 扁平化 Stage 树
  │   └─ 恢复历史状态
  └─ 创建 ReAct Agent
  ↓
agent.run()
  ↓
run_loop()
  ↓
┌─────────────────────────────────────┐
│ one_loop() - 主循环                  │
├─────────────────────────────────────┤
│ 1. get_current_tips()               │
│    └─ StageManager.get_current_tips()│
│       └─ 返回当前阶段任务            │
│                                     │
│ 2. do_work()                        │
│    └─ agent.stream()                │
│       └─ LangGraph ReAct 循环       │
│          ├─ LLM 推理                │
│          ├─ 工具调用                │
│          │  ├─ ReadTextFile         │
│          │  ├─ EditTextFile         │
│          │  ├─ Check                │
│          │  └─ Complete             │
│          └─ 观察结果                │
│                                     │
│ 3. invoke_round += 1                │
└─────────────────────────────────────┘
  ↓
Check 通过 → Complete → 下一阶段
  ↓
所有阶段完成 → Exit → 结束
```

---

## 下一章

[05-core-classes.md](./05-core-classes.md) - 核心类详解