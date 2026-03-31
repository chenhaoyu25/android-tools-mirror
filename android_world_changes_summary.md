# Android World 代码修改总结

> 对比 `android_world_origin` 与 `android_world_revised` 两个版本之间的所有差异。

---

## 一、全局性修改

### 1.1 版权年份变更

几乎所有文件的版权头从 `Copyright 2026` 改为 `Copyright 2025`（个别文件改为 `Copyright 2024`，如 `actuation.py`、`new_json_action.py`、`infer_ma3.py` 等）。

### 1.2 删除多个 `__init__.py` 文件

Revised 版本删除了以下子包中的 `__init__.py`，改为使用 Python 隐式命名空间包：

- `agents/__init__.py`
- `env/__init__.py`
- `env/setup_device/__init__.py`
- `task_evals/__init__.py`
- `task_evals/common_validators/__init__.py`
- `task_evals/composite/__init__.py`
- `task_evals/information_retrieval/__init__.py`
- `task_evals/information_retrieval/proto/__init__.py`
- `task_evals/miniwob/__init__.py`
- `task_evals/robustness_study/__init__.py`
- `task_evals/single/__init__.py`
- `task_evals/single/calendar/__init__.py`
- `task_evals/utils/__init__.py`
- `utils/__init__.py`

---

## 二、新增文件（共 12 个）

### 2.1 新增 Agent 实现（8 个文件，共约 2495 行）

| 文件 | 行数 | 说明 |
|---|---|---|
| `agents/gui_owl.py` | 362 | **GUI-Owl-1.5 Agent** — 一个全新的 Android Agent，基于 `seeact_utils` 和自定义工具集，使用 `new_json_action` 进行动作表示 |
| `agents/mobile_agent_v3.py` | 519 | **Mobile-Agent-v3.5 Agent** — 另一个全新 Agent，采用多角色（Manager/Executor/Notetaker）协作架构 |
| `agents/mobile_agent_v3_agent.py` | 385 | Mobile-Agent-v3 的辅助模块，定义了 `InfoPool`（信息池）和各角色（Manager、Executor、Notetaker）的 Agent 基类及实现 |
| `agents/infer_ma3.py` | 158 | 为 Mobile-Agent-v3 提供的 LLM 推理接口，基于 **OpenAI 兼容 API**（而非 Google Generative AI），支持 `qwen_vl_utils.smart_resize` |
| `agents/mobile_agent_utils_new.py` | 272 | 为新 Agent 提供的工具函数集，包含图片转 Base64、Qwen Agent 的 function call prompt 组装、动作解析等 |
| `agents/function_call_mobile_answer.py` | 346 | 基于 `qwen_agent` 框架注册的 `mobile_use` 工具类，定义了 click/long_press/swipe/type/answer/system_button/wait/terminate 等动作的 schema |
| `agents/coordinate_resize.py` | 275 | 坐标缩放/变换工具，包含 `smart_resize`（图片尺寸缩放）和 `convert_point_format`（将模型输出的归一化坐标转换为实际屏幕坐标）|
| `agents/new_json_action.py` | 178 | 原 `env/json_action.py` 的修改版，作为新 Agent 的动作表示层（详见第三节）|

### 2.2 新增 Protobuf 编译产物（4 个文件）

- `task_evals/information_retrieval/proto/state_pb2.py`
- `task_evals/information_retrieval/proto/state_pb2_grpc.py`
- `task_evals/information_retrieval/proto/task_pb2.py`
- `task_evals/information_retrieval/proto/task_pb2_grpc.py`

这些是由 `.proto` 文件自动生成的 Python 代码（Protobuf v5.29.0），原版中仅有 `__init__.py`，缺少实际编译产物。

---

## 三、核心逻辑修改

### 3.1 `env/actuation.py` — 动作执行层重构

这是改动最大的文件之一，主要变更：

**导入路径更改：**
```python
# Origin
from android_world.env import json_action
# Revised
from android_world.agents import new_json_action as json_action
```

**`input_text` 动作流程重写：**
- 注释掉了原来的「先 click 聚焦再输入」逻辑
- 改为使用 `adb_utils.long_press` 先长按目标坐标聚焦，再长按固定坐标 `(1000, 2030)` （可能为全选/粘贴操作）
- **删除了 `clear_text` 相关逻辑**（原版会通过 Ctrl+A → Delete 清除已有文本）

**`swipe` 动作简化：**
- 原版根据 `direction`（字符串如 `'up'`/`'down'`）计算起止坐标
- Revised 版本直接从 `action.direction` 读取 `(start_x, start_y, end_x, end_y)` 四元组，将方向解析责任交给上层

**全局延迟：**
- 在 `execute_adb_action` 末尾增加了 `time.sleep(2)` 全局等待

**调试痕迹：**
- 多处有注释掉的 `import pdb; pdb.set_trace()` 断点

### 3.2 `env/adb_utils.py` — ADB 工具层变更

**删除 `json` 导入及相关函数：**
- 删除了 `_post_process_settings()` 和 `get_all_settings()` 两个函数（获取 Android 系统设置的功能）

**App 启动 Activity 映射表格式调整：**
- 大量条目从多行括号包裹的字符串简化为单行字符串（纯格式变更，不影响功能）
- 删除了 `'google voice'` 的 Activity 映射

**日志拼写变更：**
```python
# Origin
logging.info('Attempting to tap the screen at (%d, %d)', x, y)
# Revised（引入了拼写错误）
logging.info('Attemting to tap the screen at (%d, %d)', x, y)
```

### 3.3 `env/json_action.py` — 动作数据类修改

**移除 `Any` 类型导入**，只保留 `Optional`。

**`__repr__` 方法重写：**
- 原版调用 `self.as_dict(skip_none=True)`
- Revised 版本直接遍历 `self.__dict__` 并跳过 `None`

**删除 `as_dict()` 方法，简化 `json_str()`：**
- 原版有独立的 `as_dict(skip_none)` 方法返回字典，`json_str()` 调用它
- Revised 版本将逻辑内联到 `json_str()` 中

### 3.4 `agents/new_json_action.py` — 新版动作数据类

基于原 `json_action.py` 的修改副本，额外变更：

- **新增动作类型常量：** `OPEN = 'open_app'`、`TYPE = 'type'`、`SYSTEM_BUTTON = 'system_button'`、`TERMINATE = 'terminate'`
- 将这些新类型加入 `_ACTION_TYPES` 合法列表
- **删除 `_SCROLL_DIRECTIONS` 验证**（注释掉了滑动方向的合法性检查）
- **删除 `clear_text` 字段**
- 与 `json_action.py` 做了同样的 `as_dict` → 内联简化

### 3.5 `agents/infer.py` — 推理接口微调

- 文档字符串修正：`"Calling text-only LLM"` → `"Calling multimodal LLM"`
- 在 `GenerationConfig` 中新增 `max_output_tokens=1000` 限制

### 3.6 `utils/file_utils.py` — 临时文件策略变更

**引入固定临时目录：**
```python
TMP_LOCAL_LOCATION = convert_to_posix_path(
    get_local_tmp_directory(), "android_world"
)
```

**`tmp_file_from_device` 上下文管理器改造：**
- 原版每次调用 `tempfile.mkdtemp()` 创建临时目录，结束后用 `shutil.rmtree` 清理
- Revised 版本使用固定的 `TMP_LOCAL_LOCATION`，结束后只 `os.remove` 单个文件（不再删除整个临时目录）

**`copy_file_to_device` 格式调整：**
- `chmod 777` 命令的格式微调（纯风格变更）

### 3.7 `google/.../UniqueIdsGenerator.kt` — Kotlin 代码修复

```kotlin
// Origin（多余的非空断言）
return uniqueIdsByNode.computeIfAbsent(a, Function { _: A -> nextId.getAndIncrement() })!!
// Revised（移除 !!）
return uniqueIdsByNode.computeIfAbsent(a, Function { _: A -> nextId.getAndIncrement() })
```

---

## 四、修改分类统计

| 类别 | 数量 | 说明 |
|---|---|---|
| 新增文件 | 12 | 8 个 Agent 模块 + 4 个 Protobuf 编译产物 |
| 删除文件 | 14 | 全部为各子包的 `__init__.py` |
| 仅版权头变更 | ~80+ | 绝大多数文件仅修改了版权年份 |
| 实质性代码修改 | 7 | `actuation.py`, `adb_utils.py`, `json_action.py`, `new_json_action.py`, `infer.py`, `file_utils.py`, `UniqueIdsGenerator.kt` |

---

## 五、修改意图总结

Revised 版本的核心目的是在 Android World 评测框架中**集成新的 Agent 实现**，主要包括：

1. **GUI-Owl-1.5 Agent** (`gui_owl.py`) — 一个新的视觉语言模型 Agent
2. **Mobile-Agent-v3.5** (`mobile_agent_v3.py`) — 采用 Manager-Executor-Notetaker 多角色协作的 Agent

为支撑这两个新 Agent，修改了底层的动作执行层（`actuation.py`）、引入了新的动作表示（`new_json_action.py`）、新的 LLM 推理接口（`infer_ma3.py`，基于 OpenAI API）、以及一套围绕 Qwen Agent 框架的工具函数。

同时还做了一些环境适配工作，如临时文件管理策略调整、Kotlin 代码修复、Protobuf 编译产物补全等。部分修改中保留了调试痕迹（如注释掉的 `pdb` 断点和 TODO 标记），说明这是一个仍在开发中的版本。
