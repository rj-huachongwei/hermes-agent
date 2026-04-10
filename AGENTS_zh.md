# Hermes Agent - 开发指南

面向 AI 编程助手及在 hermes-agent 代码库上工作的开发者。

## 开发环境

```bash
source venv/bin/activate  # 运行 Python 前务必先激活虚拟环境
```

## 项目结构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 — 核心对话循环
├── model_tools.py        # 工具编排，_discover_tools()，handle_function_call()
├── toolsets.py           # 工具集定义，_HERMES_CORE_TOOLS 列表
├── cli.py                # HermesCLI 类 — 交互式 CLI 编排器
├── hermes_state.py       # SessionDB — SQLite 会话存储（FTS5 全文搜索）
├── agent/                # Agent 内部模块
│   ├── prompt_builder.py     # 系统提示词组装
│   ├── context_compressor.py # 自动上下文压缩
│   ├── prompt_caching.py     # Anthropic 提示词缓存
│   ├── auxiliary_client.py   # 辅助 LLM 客户端（视觉、摘要）
│   ├── model_metadata.py     # 模型上下文长度、Token 估算
│   ├── models_dev.py         # models.dev 注册表集成（感知 provider 的上下文）
│   ├── display.py            # KawaiiSpinner、工具预览格式化
│   ├── skill_commands.py     # 技能斜杠命令（CLI/gateway 共享）
│   └── trajectory.py         # 轨迹保存辅助函数
├── hermes_cli/           # CLI 子命令与设置
│   ├── main.py           # 入口点 — 所有 `hermes` 子命令
│   ├── config.py         # DEFAULT_CONFIG、OPTIONAL_ENV_VARS、迁移逻辑
│   ├── commands.py       # 斜杠命令定义 + SlashCommandCompleter
│   ├── callbacks.py      # 终端回调（clarify、sudo、approval）
│   ├── setup.py          # 交互式安装向导
│   ├── skin_engine.py    # 皮肤/主题引擎 — CLI 视觉定制
│   ├── skills_config.py  # `hermes skills` — 按平台启用/禁用技能
│   ├── tools_config.py   # `hermes tools` — 按平台启用/禁用工具
│   ├── skills_hub.py     # `/skills` 斜杠命令（搜索、浏览、安装）
│   ├── models.py         # 模型目录、provider 模型列表
│   ├── model_switch.py   # 共享 /model 切换流程（CLI + gateway）
│   └── auth.py           # Provider 凭证解析
├── tools/                # 工具实现（每个工具一个文件）
│   ├── registry.py       # 中央工具注册表（schema、handler、dispatch）
│   ├── approval.py       # 危险命令检测
│   ├── terminal_tool.py  # 终端编排
│   ├── process_registry.py # 后台进程管理
│   ├── file_tools.py     # 文件读/写/搜索/补丁
│   ├── web_tools.py      # 网页搜索/抓取（Parallel + Firecrawl）
│   ├── browser_tool.py   # Browserbase 浏览器自动化
│   ├── code_execution_tool.py # execute_code 沙箱
│   ├── delegate_tool.py  # 子 Agent 委托
│   ├── mcp_tool.py       # MCP 客户端（约 1050 行）
│   └── environments/     # 终端后端（local、docker、ssh、modal、daytona、singularity）
├── gateway/              # 消息平台网关
│   ├── run.py            # 主循环、斜杠命令、消息分发
│   ├── session.py        # SessionStore — 对话持久化
│   └── platforms/        # 适配器：telegram、discord、slack、whatsapp、homeassistant、signal
├── acp_adapter/          # ACP 服务器（VS Code / Zed / JetBrains 集成）
├── cron/                 # 调度器（jobs.py、scheduler.py）
├── environments/         # RL 训练环境（Atropos）
├── tests/                # Pytest 测试套件（约 3000 个测试）
└── batch_runner.py       # 并行批处理
```

**用户配置：** `~/.hermes/config.yaml`（设置项），`~/.hermes/.env`（API 密钥）

## 文件依赖链

```
tools/registry.py  （无依赖 — 被所有工具文件导入）
       ↑
tools/*.py  （每个文件在 import 时调用 registry.register()）
       ↑
model_tools.py  （导入 tools/registry 并触发工具发现）
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## AIAgent 类（run_agent.py）

```python
class AIAgent:
    def __init__(self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,           # "cli"、"telegram" 等
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        # ... 以及 provider、api_mode、callbacks、routing 参数
    ): ...

    def chat(self, message: str) -> str:
        """简单接口 — 返回最终响应字符串。"""

    def run_conversation(self, user_message: str, system_message: str = None,
                         conversation_history: list = None, task_id: str = None) -> dict:
        """完整接口 — 返回包含 final_response 和 messages 的字典。"""
```

### Agent 循环

核心循环位于 `run_conversation()` 内部，完全同步执行：

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

消息遵循 OpenAI 格式：`{"role": "system/user/assistant/tool", ...}`。推理内容存储在 `assistant_msg["reasoning"]` 中。

---

## CLI 架构（cli.py）

- **Rich** 用于横幅/面板渲染，**prompt_toolkit** 用于带自动补全的交互输入
- **KawaiiSpinner**（`agent/display.py`）— API 调用期间的动态表情动画，`┊` 前缀用于工具结果的活动流
- `cli.py` 中的 `load_cli_config()` 合并硬编码默认值与用户 YAML 配置
- **皮肤引擎**（`hermes_cli/skin_engine.py`）— 数据驱动的 CLI 主题系统；启动时从配置的 `display.skin` 键初始化；皮肤可定制横幅颜色、spinner 表情/动词/翅膀、工具前缀、响应框、品牌文字
- `process_command()` 是 `HermesCLI` 的方法 — 通过中央注册表的 `resolve_command()` 解析规范命令名后进行分发
- 技能斜杠命令：`agent/skill_commands.py` 扫描 `~/.hermes/skills/`，以**用户消息**（而非系统提示词）方式注入，以保留提示词缓存

### 斜杠命令注册表（`hermes_cli/commands.py`）

所有斜杠命令在 `COMMAND_REGISTRY`（`CommandDef` 对象列表）中集中定义，所有下游消费者自动从此注册表派生：

- **CLI** — `process_command()` 通过 `resolve_command()` 解析别名，按规范名称分发
- **Gateway** — `GATEWAY_KNOWN_COMMANDS` 冻结集用于钩子触发，`resolve_command()` 用于分发
- **Gateway 帮助** — `gateway_help_lines()` 生成 `/help` 输出
- **Telegram** — `telegram_bot_commands()` 生成 BotCommand 菜单
- **Slack** — `slack_subcommand_map()` 生成 `/hermes` 子命令路由
- **自动补全** — `COMMANDS` 扁平字典提供给 `SlashCommandCompleter`
- **CLI 帮助** — `COMMANDS_BY_CATEGORY` 字典提供给 `show_help()`

### 添加斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中添加 `CommandDef` 条目：
```python
CommandDef("mycommand", "命令功能描述", "Session",
           aliases=("mc",), args_hint="[arg]"),
```
2. 在 `cli.py` 的 `HermesCLI.process_command()` 中添加处理器：
```python
elif canonical == "mycommand":
    self._handle_mycommand(cmd_original)
```
3. 如果该命令在 gateway 中也可用，在 `gateway/run.py` 中添加处理器：
```python
if canonical == "mycommand":
    return await self._handle_mycommand(event)
```
4. 对于需要持久化的设置，使用 `cli.py` 中的 `save_config_value()`

**CommandDef 字段说明：**
- `name` — 不含斜杠的规范名称（例如 `"background"`）
- `description` — 人类可读的描述
- `category` — 取值为 `"Session"`、`"Configuration"`、`"Tools & Skills"`、`"Info"`、`"Exit"` 之一
- `aliases` — 备用名称元组（例如 `("bg",)`）
- `args_hint` — 帮助信息中显示的参数占位符（例如 `"<prompt>"`、`"[name]"`）
- `cli_only` — 仅在交互式 CLI 中可用
- `gateway_only` — 仅在消息平台中可用
- `gateway_config_gate` — 配置点路径（例如 `"display.tool_progress_command"`）；当设置在 `cli_only` 命令上时，若配置值为真则该命令在 gateway 中也可用。`GATEWAY_KNOWN_COMMANDS` 始终包含配置门控命令以便 gateway 可以分发；帮助/菜单仅在门控开启时显示它们。

**添加别名**只需将其加入现有 `CommandDef` 的 `aliases` 元组即可。无需修改其他文件 — 分发、帮助文本、Telegram 菜单、Slack 映射和自动补全均会自动更新。

---

## 添加新工具

需要修改 **3 个文件**：

**1. 创建 `tools/your_tool.py`：**
```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**2. 在 `model_tools.py` 的 `_discover_tools()` 列表中添加 import。**

**3. 添加到 `toolsets.py`** — 加入 `_HERMES_CORE_TOOLS`（所有平台）或新建工具集。

注册表负责 schema 收集、分发、可用性检查和错误包装。所有 handler **必须返回 JSON 字符串**。

**工具 schema 中的路径引用**：如果 schema 描述中提到文件路径（例如默认输出目录），请使用 `display_hermes_home()` 以兼容多 profile。schema 在 import 时生成，此时 `_apply_profile_override()` 已设置 `HERMES_HOME`。

**状态文件**：如果工具需要持久化状态（缓存、日志、检查点），使用 `get_hermes_home()` 作为基础目录 — 永远不要使用 `Path.home() / ".hermes"`。这确保每个 profile 拥有独立的状态。

**Agent 级别的工具**（todo、memory）：在 `run_agent.py` 中被 `handle_function_call()` 之前拦截处理。参见 `todo_tool.py` 了解此模式。

---

## 添加配置项

### config.yaml 选项：
1. 在 `hermes_cli/config.py` 的 `DEFAULT_CONFIG` 中添加
2. 递增 `_config_version`（当前为 5）以触发现有用户的迁移

### .env 变量：
1. 在 `hermes_cli/config.py` 的 `OPTIONAL_ENV_VARS` 中添加元数据：
```python
"NEW_API_KEY": {
    "description": "用途说明",
    "prompt": "显示名称",
    "url": "https://...",
    "password": True,
    "category": "tool",  # provider、tool、messaging、setting
},
```

### 配置加载器（两套独立系统）：

| 加载器 | 使用方 | 位置 |
|--------|---------|----------|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | `hermes tools`、`hermes setup` | `hermes_cli/config.py` |
| 直接 YAML 加载 | Gateway | `gateway/run.py` |

---

## 皮肤/主题系统

皮肤引擎（`hermes_cli/skin_engine.py`）提供数据驱动的 CLI 视觉定制。皮肤是**纯数据** — 添加新皮肤无需修改代码。

### 架构

```
hermes_cli/skin_engine.py    # SkinConfig dataclass、内置皮肤、YAML 加载器
~/.hermes/skins/*.yaml       # 用户安装的自定义皮肤（直接放入即可）
```

- `init_skin_from_config()` — CLI 启动时调用，从配置读取 `display.skin`
- `get_active_skin()` — 返回当前皮肤的缓存 `SkinConfig`
- `set_active_skin(name)` — 运行时切换皮肤（由 `/skin` 命令使用）
- `load_skin(name)` — 先从用户皮肤加载，再查找内置皮肤，最后回退到 default
- 缺失的皮肤值会自动从 `default` 皮肤继承

### 皮肤可定制的元素

| 元素 | 皮肤键 | 使用方 |
|---------|----------|---------|
| 横幅面板边框 | `colors.banner_border` | `banner.py` |
| 横幅面板标题 | `colors.banner_title` | `banner.py` |
| 横幅章节标题 | `colors.banner_accent` | `banner.py` |
| 横幅暗色文字 | `colors.banner_dim` | `banner.py` |
| 横幅正文 | `colors.banner_text` | `banner.py` |
| 响应框边框 | `colors.response_border` | `cli.py` |
| Spinner 等待表情 | `spinner.waiting_faces` | `display.py` |
| Spinner 思考表情 | `spinner.thinking_faces` | `display.py` |
| Spinner 动词 | `spinner.thinking_verbs` | `display.py` |
| Spinner 翅膀（可选） | `spinner.wings` | `display.py` |
| 工具输出前缀 | `tool_prefix` | `display.py` |
| 各工具 emoji | `tool_emojis` | `display.py` → `get_tool_emoji()` |
| Agent 名称 | `branding.agent_name` | `banner.py`、`cli.py` |
| 欢迎消息 | `branding.welcome` | `cli.py` |
| 响应框标签 | `branding.response_label` | `cli.py` |
| 提示符符号 | `branding.prompt_symbol` | `cli.py` |

### 内置皮肤

- `default` — 经典 Hermes 金色/Kawaii 风格（当前外观）
- `ares` — 深红/青铜战神主题，带自定义 spinner 翅膀
- `mono` — 简洁灰度单色
- `slate` — 冷蓝色开发者风格主题

### 添加内置皮肤

在 `hermes_cli/skin_engine.py` 的 `_BUILTIN_SKINS` 字典中添加：

```python
"mytheme": {
    "name": "mytheme",
    "description": "简短描述",
    "colors": { ... },
    "spinner": { ... },
    "branding": { ... },
    "tool_prefix": "┊",
},
```

### 用户皮肤（YAML）

用户创建 `~/.hermes/skins/<name>.yaml`：

```yaml
name: cyberpunk
description: 霓虹浸染的终端主题

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  banner_accent: "#FF1493"

spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "

tool_prefix: "▏"
```

通过 `/skin cyberpunk` 或在 config.yaml 中设置 `display.skin: cyberpunk` 来激活。

---

## 重要规范

### 不可破坏提示词缓存

Hermes-Agent 确保缓存在整个对话过程中保持有效。**禁止实现以下会破坏缓存的变更：**
- 在对话中途修改历史上下文
- 在对话中途更改工具集
- 在对话中途重新加载记忆或重建系统提示词

破坏缓存会导致成本急剧上升。**唯一**允许修改上下文的时机是在上下文压缩期间。

### 工作目录行为
- **CLI**：使用当前目录（`.` → `os.getcwd()`）
- **消息平台**：使用 `MESSAGING_CWD` 环境变量（默认：home 目录）

### 后台进程通知（Gateway）

当使用 `terminal(background=true, check_interval=...)` 时，gateway 会运行一个 watcher，将状态更新推送到用户聊天。通过 config.yaml 中的 `display.background_process_notifications`（或 `HERMES_BACKGROUND_NOTIFICATIONS` 环境变量）控制通知详细程度：

- `all` — 运行中输出更新 + 最终消息（默认）
- `result` — 仅最终完成消息
- `error` — 仅当退出码不为 0 时的最终消息
- `off` — 不发送任何 watcher 消息

---

## Profiles：多实例支持

Hermes 支持 **profiles** — 多个完全隔离的实例，每个实例拥有独立的 `HERMES_HOME` 目录（配置、API 密钥、记忆、会话、技能、gateway 等）。

核心机制：`hermes_cli/main.py` 中的 `_apply_profile_override()` 在任何模块导入之前设置 `HERMES_HOME`。所有 119+ 处 `get_hermes_home()` 调用自动作用于当前激活的 profile。

### Profile 安全代码规范

1. **所有 HERMES_HOME 路径使用 `get_hermes_home()`。** 从 `hermes_constants` 导入。
   绝对不要在读写状态的代码中硬编码 `~/.hermes` 或 `Path.home() / ".hermes"`。
   ```python
   # 正确
   from hermes_constants import get_hermes_home
   config_path = get_hermes_home() / "config.yaml"

   # 错误 — 会破坏 profiles
   config_path = Path.home() / ".hermes" / "config.yaml"
   ```

2. **面向用户的消息使用 `display_hermes_home()`。** 从 `hermes_constants` 导入。
   默认 profile 返回 `~/.hermes`，其他 profile 返回 `~/.hermes/profiles/<name>`。
   ```python
   # 正确
   from hermes_constants import display_hermes_home
   print(f"Config saved to {display_hermes_home()}/config.yaml")

   # 错误 — profile 下会显示错误路径
   print("Config saved to ~/.hermes/config.yaml")
   ```

3. **模块级常量没问题** — 它们在 import 时缓存 `get_hermes_home()`，
   此时 `_apply_profile_override()` 已经设置好了环境变量。直接使用 `get_hermes_home()`，
   不要使用 `Path.home() / ".hermes"`。

4. **Mock 了 `Path.home()` 的测试还必须设置 `HERMES_HOME`** — 因为代码现在使用
   `get_hermes_home()`（读取环境变量），而非 `Path.home() / ".hermes"`：
   ```python
   with patch.object(Path, "home", return_value=tmp_path), \
        patch.dict(os.environ, {"HERMES_HOME": str(tmp_path / ".hermes")}):
       ...
   ```

5. **Gateway 平台适配器应使用 token 锁** — 如果适配器使用唯一凭证（bot token、API key）连接，
   在 `connect()`/`start()` 方法中从 `gateway.status` 调用 `acquire_scoped_lock()`，
   在 `disconnect()`/`stop()` 中调用 `release_scoped_lock()`。
   这可防止两个 profile 使用同一凭证。
   参见 `gateway/platforms/telegram.py` 中的规范模式。

6. **Profile 操作以 HOME 为锚点，而非 HERMES_HOME** — `_get_profiles_root()`
   返回 `Path.home() / ".hermes" / "profiles"`，而**不是** `get_hermes_home() / "profiles"`。
   这是有意为之 — 使得 `hermes -p coder profile list` 无论哪个 profile 处于激活状态，都能看到所有 profile。

## 已知陷阱

### 禁止硬编码 `~/.hermes` 路径
代码路径使用 `hermes_constants` 中的 `get_hermes_home()`。面向用户的打印/日志消息使用 `display_hermes_home()`。
硬编码 `~/.hermes` 会破坏 profiles — 每个 profile 有独立的 `HERMES_HOME` 目录。这是 PR #3575 中修复的 5 个 bug 的根源。

### 禁止使用 `simple_term_menu` 创建交互式菜单
在 tmux/iTerm2 中存在渲染 bug — 滚动时出现残影。请改用 `curses`（标准库）。参见 `hermes_cli/tools_config.py` 中的模式。

### 禁止在 spinner/display 代码中使用 `\033[K`（ANSI 清除到行尾）
在 `prompt_toolkit` 的 `patch_stdout` 下会泄漏为字面量 `?[K` 文本。请使用空格填充：`f"\r{line}{' ' * pad}"`。

### `_last_resolved_tool_names` 是 `model_tools.py` 中的进程全局变量
`delegate_tool.py` 中的 `_run_single_child()` 在子 Agent 执行前后保存和恢复此全局变量。如果你添加了读取此全局变量的新代码，请注意在子 Agent 运行期间它可能暂时是过期值。

### 禁止在 schema 描述中硬编码跨工具引用
工具 schema 描述中不能按名称提及其他工具集中的工具（例如 `browser_navigate` 中写"优先使用 web_search"）。那些工具可能不可用（缺少 API key、工具集被禁用），导致模型产生对不存在工具的幻觉调用。如果确实需要跨工具引用，请在 `model_tools.py` 的 `get_tool_definitions()` 中动态添加 — 参见 `browser_navigate` / `execute_code` 后处理块的模式。

### 测试不得写入 `~/.hermes/`
`tests/conftest.py` 中的 `_isolate_hermes_home` autouse fixture 会将 `HERMES_HOME` 重定向到临时目录。测试中永远不要硬编码 `~/.hermes/` 路径。

**Profile 测试**：测试 profile 功能时，还需 mock `Path.home()`，使得
`_get_profiles_root()` 和 `_get_default_hermes_home()` 在临时目录中解析。
使用 `tests/hermes_cli/test_profiles.py` 中的模式：
```python
@pytest.fixture
def profile_env(tmp_path, monkeypatch):
    home = tmp_path / ".hermes"
    home.mkdir()
    monkeypatch.setattr(Path, "home", lambda: tmp_path)
    monkeypatch.setenv("HERMES_HOME", str(home))
    return home
```

---

## 测试

```bash
source venv/bin/activate
python -m pytest tests/ -q          # 完整测试套件（约 3000 个测试，约 3 分钟）
python -m pytest tests/test_model_tools.py -q   # 工具集解析
python -m pytest tests/test_cli_init.py -q       # CLI 配置加载
python -m pytest tests/gateway/ -q               # Gateway 测试
python -m pytest tests/tools/ -q                 # 工具层测试
```

推送变更前务必运行完整测试套件。

