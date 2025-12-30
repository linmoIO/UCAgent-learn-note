# 关键算法深度解析

**省流版**：

本文讲解 UCAgent 的三个关键算法实现：
- **Stage 递归解析算法**：
  - 将 YAML 配置的嵌套 Stage 结构递归解析为 Stage 树
  - 通过 prefix 参数生成索引编号（如 "1.1"、"1.2"）
  - 源码位置：`stage/vstage.py:367-404`
- **Stage 树扁平化算法**：
  - 将树状结构扁平化为线性列表
  - 只保留叶子节点执行，跳过组节点
  - 通过后序遍历保证执行顺序
  - 源码位置：`stage/vstage.py:300-306`
- **批处理任务管理算法**：
  - 将大量测试用例分批生成，避免一次性生成过多代码
  - 四个列表管理任务状态：目标任务、已生成任务、待完成任务、已完成任务
  - 源码位置：`checkers/base.py:309-550`

---

## 1. Stage 递归解析算法

### 1.1 算法背景

UCAgent 的验证流程由多个 Stage 组成，这些 Stage 可以嵌套。例如：

```yaml
stage:
  - name: "测试实现"
    stage:
      - name: "基础测试"
      - name: "边界测试"
  - name: "覆盖率分析"
```

这种嵌套结构需要递归解析，将 YAML 配置转换为 Stage 树。

### 1.2 算法实现

**源码位置**：`stage/vstage.py:367-404`

```python
def parse_vstage(root_cfg, cfg, workspace, tool_read_text, prefix=""):
    """递归解析 Stage 配置"""
    if cfg is None:
        return []

    ret = []
    for i, stage in enumerate(cfg):
        # 计算当前 Stage 的索引
        index = i + 1

        # 递归解析子 Stage
        substages = parse_vstage(
            root_cfg,
            stage.get_value('stage', None),  # 获取子 Stage 配置
            workspace,
            tool_read_text,
            prefix + f"{index}."             # 拼接索引前缀
        )

        # 创建当前 Stage 对象
        ret.append(VerifyStage(
            cfg=root_cfg,
            workspace=workspace,
            name=stage.name,
            description=stage.desc,
            task=stage.task,
            checker=stage.get_value('checker', []),
            reference_files=stage.get_value('reference_files', []),
            output_files=stage.get_value('output_files', []),
            substages=substages,             # 绑定子 Stage
            prefix=prefix + f"{index}",      # 设置索引前缀
            skip=stage.get_value('skip', False),
        ))

    return ret
```

### 1.3 算法执行过程

**输入配置**：
```yaml
stage:
  - name: "API测试"
    stage:
      - name: "基础API"
      - name: "高级API"
  - name: "功能测试"
```

**递归调用树**：
```
parse_vstage(cfg, prefix="")
│
├─ i=0, index=1, prefix="1"
│  ├─ 创建 VerifyStage("API测试", prefix="1")
│  └─ 递归调用 parse_vstage(cfg[0].stage, prefix="1.")
│      │
│      ├─ i=0, index=1, prefix="1.1"
│      │  └─ 创建 VerifyStage("基础API", prefix="1.1")
│      │
│      └─ i=1, index=2, prefix="1.2"
│         └─ 创建 VerifyStage("高级API", prefix="1.2")
│
└─ i=1, index=2, prefix="2"
   └─ 创建 VerifyStage("功能测试", prefix="2")
```

**输出结果**：
```
VerifyStage("1-API测试")
├── VerifyStage("1.1-基础API")
└── VerifyStage("1.2-高级API")
VerifyStage("2-功能测试")
```

### 1.4 关键设计点

1. **prefix 参数**：用于生成 Stage 的索引编号（如 "1.1", "1.2"）
2. **递归终止条件**：`cfg is None` 或 `stage.get_value('stage', None)` 返回 None
3. **索引计算**：`index = i + 1`，从 1 开始编号（而非 0）
4. **父子关系**：通过 `substages` 参数建立父子关系

---

## 2. Stage 树扁平化算法

### 2.1 算法背景

解析后的 Stage 是树状结构，但 StageManager 需要按顺序执行 Stage。因此需要将树状结构扁平化为线性列表。

**关键问题**：哪些 Stage 需要执行？

**答案**：只执行**叶子节点**（有实际任务的 Stage），跳过**组节点**（仅用于组织结构的 Stage）。

### 2.2 组节点的判断

**源码位置**：`stage/vstage.py:308-309`

```python
def is_group(self):
    """判断是否为组节点"""
    return (self.check_size == 0 and           # 没有 Checker
            len(self.output_files) == 0 and    # 没有输出文件
            len(self.reference_files) == 0 and # 没有参考文件
            len(self.substages) > 0)           # 有子 Stage
```

**判断逻辑**：
- 组节点：只用于组织结构，没有实际任务
- 叶子节点：有实际任务需要执行

### 2.3 扁平化算法

**源码位置**：`stage/vstage.py:300-306`

```python
def get_substages(self) -> list[VerifyStage]:
    """递归收集所有叶子节点"""
    ret = []

    # 递归收集所有子 Stage 的叶子节点
    for s in self.substages:
        ret.extend(s.get_substages())

    # 如果当前节点不是组节点，添加自己
    if not self.is_group():
        ret.append(self)

    return ret
```

### 2.4 算法执行过程

**输入树**：
```
Root (组节点)
├── 1. API测试 (组节点)
│   ├── 1.1 基础API (叶子节点)
│   └── 1.2 高级API (叶子节点)
└── 2. 功能测试 (叶子节点)
```

**递归调用过程**：
```
Root.get_substages()
│
├─ 遍历 substages[0] (1. API测试)
│  └─ 1.get_substages()
│     │
│     ├─ 遍历 substages[0] (1.1 基础API)
│     │  └─ 1.1.get_substages()
│     │     ├─ substages 为空，跳过循环
│     │     ├─ 1.1 不是组节点 → 添加自己
│     │     └─ 返回 [1.1]
│     │
│     ├─ 遍历 substages[1] (1.2 高级API)
│     │  └─ 1.2.get_substages()
│     │     ├─ substages 为空，跳过循环
│     │     ├─ 1.2 不是组节点 → 添加自己
│     │     └─ 返回 [1.2]
│     │
│     ├─ ret = [1.1, 1.2]
│     ├─ 1 是组节点 → 不添加自己
│     └─ 返回 [1.1, 1.2]
│
├─ 遍历 substages[1] (2. 功能测试)
│  └─ 2.get_substages()
│     ├─ substages 为空，跳过循环
│     ├─ 2 不是组节点 → 添加自己
│     └─ 返回 [2]
│
├─ ret = [1.1, 1.2, 2]
├─ Root 是组节点 → 不添加自己
└─ 返回 [1.1, 1.2, 2]
```

**输出列表**：
```python
stages = [
    VerifyStage("1.1-基础API"),
    VerifyStage("1.2-高级API"),
    VerifyStage("2-功能测试")
]
```

### 2.5 关键设计点

1. **后序遍历**：先递归子节点，再处理当前节点
2. **组节点过滤**：组节点不添加到结果列表
3. **顺序保证**：深度优先遍历保证了执行顺序
4. **扁平化结果**：StageManager 使用扁平化后的列表，通过 `stage_index` 顺序执行

---

## 3. 批处理任务管理算法

### 3.1 算法背景

在验证过程中，经常需要生成大量测试用例（例如 30 个）。如果一次性让 LLM 生成所有测试用例，会导致：
1. 输出过长，容易出错
2. 难以追踪进度
3. 失败后需要重新生成所有用例

因此，UCAgent 采用**批处理**策略：将任务分成多个批次，每次只生成一小部分（例如 10 个）。

### 3.2 核心数据结构

**源码位置**：`checkers/base.py:309-550`

```python
class UnityChipBatchTask:
    """批处理任务管理器"""

    def __init__(self, name: str, checker: Checker):
        self.name = name                # 任务类型名称
        self.checker = checker          # 关联的 Checker

        # 四个核心列表
        self.source_task_list = []      # 目标任务（来自配置）
        self.gen_task_list = []         # 已生成任务（实际完成）
        self.tbd_task_list = []         # 待完成任务（当前批次）
        self.cmp_task_list = []         # 已完成任务（当前批次）

        self.batch_size = 10            # 批次大小
```

**四个列表的关系**：
```
source_task_list: [test1, test2, ..., test30]  # 目标：30个测试
gen_task_list:    [test1, ..., test10]         # 已生成：10个
tbd_task_list:    [test11, ..., test20]        # 当前批次：10个
cmp_task_list:    [test11, test12]             # 当前批次已完成：2个
```

---

## 下一章

[04-execution-flow.md](./04-execution-flow.md) - 执行流程分析