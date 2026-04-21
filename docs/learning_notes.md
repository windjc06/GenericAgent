# GenericAgent 初学者学习笔记

本文面向刚开始学习编程和 Agent 的读者，目标是用项目源码解释 GenericAgent 的技术原理。建议先读 `README.md` 和 `GETTING_STARTED.md`，再配合本文按模块阅读源码。

## 1. 一句话理解这个项目

GenericAgent 是一个本地自主 Agent 框架。用户说一句任务，程序把任务交给大语言模型，模型决定下一步要不要调用工具，工具真实地读写文件、执行命令、控制网页，执行结果再反馈给模型。这个过程不断循环，直到任务完成或需要询问用户。

可以把它理解成：

```text
用户输入
  -> Agent 组装系统提示词、记忆、工具说明
  -> LLM 思考并选择工具
  -> Python 执行工具
  -> 工具结果返回给 LLM
  -> 重复，直到完成
```

这个项目的特点不是预先写很多复杂业务插件，而是只提供少量“原子能力”，让模型在真实电脑环境里组合这些能力完成任务，并把成功经验沉淀到 `memory/`。

## 2. 项目主要目录

```text
GenericAgent/
├── agentmain.py              # 主入口：创建 Agent、任务队列、命令行/后台/reflect 模式
├── agent_loop.py             # 核心循环：让 LLM 和工具反复交互
├── ga.py                     # 工具实现：执行代码、读写文件、网页扫描、JS 控制等
├── llmcore.py                # LLM 适配层：OpenAI/Claude/Native 工具调用/故障转移
├── TMWebDriver.py            # 浏览器连接桥：和 Chrome 扩展通信，执行 JS
├── launch.pyw                # 桌面 GUI 启动器：启动 Streamlit + pywebview 窗口
├── frontends/                # 多种聊天入口：Streamlit、Telegram、飞书、微信等
├── memory/                   # 长期记忆、SOP、技能、会话归档
├── assets/                   # 系统提示词、工具 schema、浏览器扩展、图片和模板
├── reflect/                  # 反射/定时触发任务
└── tests/                    # 少量测试
```

新手最重要的源码阅读顺序：

1. `agentmain.py`
2. `agent_loop.py`
3. `ga.py`
4. `llmcore.py`
5. `frontends/stapp.py`
6. `TMWebDriver.py`

## 3. 从启动到执行任务的完整流程

### 3.1 启动方式

最简单的命令行启动：

```bash
python agentmain.py
```

图形界面启动：

```bash
python launch.pyw
```

`launch.pyw` 会先启动 Streamlit 前端 `frontends/stapp.py`，再用 `pywebview` 打开一个桌面窗口。本质上，GUI 只是一个更友好的输入输出界面，真正处理任务的仍然是 `agentmain.py` 里的 `GeneraticAgent`。

### 3.2 配置 LLM

项目不会把模型写死。你需要把 `mykey_template.py` 复制成 `mykey.py`，然后填入模型 API。

`agentmain.py` 会读取 `llmcore.mykeys`，根据变量名选择不同 Session：

```text
变量名包含 native + claude -> NativeClaudeSession
变量名包含 native + oai    -> NativeOAISession
变量名包含 claude          -> ClaudeSession
变量名包含 oai             -> LLMSession
变量名包含 mixin           -> MixinSession
```

这里的关键点是：变量名决定使用哪种协议，而不是模型名决定。

### 3.3 创建 Agent 实例

`agentmain.py` 里的 `GeneraticAgent.__init__()` 做几件事：

1. 创建 `temp/` 目录。
2. 读取 `mykey.py` 中的模型配置。
3. 为每个配置创建一个 LLM client。
4. 创建任务队列 `task_queue`。
5. 保存当前会话历史、当前模型编号、运行状态等。

所以 `GeneraticAgent` 可以理解为“应用层 Agent 对象”，它负责接收任务、选择模型、启动核心循环。

## 4. Agent Loop：项目最核心的机制

核心循环在 `agent_loop.py` 的 `agent_runner_loop()`。

它做的事情可以简化成下面的伪代码：

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": user_input},
]

while turn < max_turns:
    response = client.chat(messages, tools=tools_schema)

    if response has tool_calls:
        for tool_call in response.tool_calls:
            result = handler.dispatch(tool_name, args, response)
            collect tool result
    else:
        handler.dispatch("no_tool", {}, response)

    if task done:
        break

    messages = [{
        "role": "user",
        "content": next_prompt,
        "tool_results": tool_results,
    }]
```

这里最重要的是理解三个对象：

```text
client  -> 负责和 LLM 通信，来自 llmcore.py
handler -> 负责执行工具，来自 ga.py 的 GenericAgentHandler
tools_schema -> 告诉模型有哪些工具可以用，来自 assets/tools_schema.json
```

模型不是直接操作电脑。模型只输出“我要调用哪个工具，以及参数是什么”。Python 程序解析这个工具调用，再执行真实操作。

## 5. 工具系统：LLM 如何获得真实执行能力

工具定义在 `assets/tools_schema.json`，工具实现主要在 `ga.py`。

### 5.1 工具调用如何分发

`agent_loop.py` 里的 `BaseHandler.dispatch()` 会把工具名映射成方法名：

```text
code_run       -> GenericAgentHandler.do_code_run()
file_read      -> GenericAgentHandler.do_file_read()
file_write     -> GenericAgentHandler.do_file_write()
web_scan       -> GenericAgentHandler.do_web_scan()
web_execute_js -> GenericAgentHandler.do_web_execute_js()
```

也就是说，只要模型调用了 `file_read`，Python 端就会执行 `do_file_read()`。

### 5.2 主要工具解释

#### `code_run`

执行 Python 或 shell 命令。它是 GenericAgent 最强的底层能力，因为通过它可以：

- 安装依赖
- 执行脚本
- 调用系统命令
- 操作本地程序
- 调试和验证结果

`ga.py` 中的 `code_run()` 会创建临时脚本文件，使用 `subprocess.Popen()` 执行，并实时收集标准输出。

#### `file_read`

读取文件内容，支持从指定行开始读，也支持按关键字搜索上下文。Agent 在改文件前通常应该先读文件。

#### `file_write`

创建、覆盖、追加文件。适合写入大块内容。

#### `file_patch`

做精确局部替换。它要求 `old_content` 在文件中只出现一次，这样可以避免误改多个位置。

#### `web_scan`

扫描浏览器当前页面，把 HTML 简化后返回给模型。简化逻辑在 `simphtml.py`，目的是减少 token 消耗。

#### `web_execute_js`

向浏览器页面执行 JavaScript。它比单纯扫描网页更强，因为可以点击按钮、填写输入框、读取 DOM、跳转页面。

#### `ask_user`

当 Agent 需要人类确认、缺少信息或遇到无法自动解决的问题时，用这个工具中断任务并询问用户。

#### `update_working_checkpoint`

短期工作记忆。长任务容易丢上下文，这个工具把关键进度保存到 `handler.working`，下一轮自动注入提示词。

#### `start_long_term_update`

长期记忆更新入口。任务完成后，如果有值得长期保存的经验、路径、环境事实，Agent 会启动记忆沉淀流程。

## 6. LLM 适配层：为什么能兼容多种模型

`llmcore.py` 负责把不同模型 API 包装成统一接口。核心目标是：不管后端是 Claude、OpenAI 兼容服务、MiniMax 还是中转服务，Agent Loop 都只调用：

```python
client.chat(messages=messages, tools=tools_schema)
```

主要类：

```text
BaseSession         # 保存 API key、base url、model、历史、超时等通用配置
ClaudeSession       # Claude messages API，工具调用通过文本协议解析
LLMSession          # OpenAI 兼容 chat/completions 或 responses
NativeClaudeSession # Claude 原生 tool_use/tool_result
NativeOAISession    # OpenAI 原生 tools/function calling
ToolClient          # 非 native：把工具说明塞进 prompt，再解析 <tool_use>
NativeToolClient    # native：把 tools 作为 API 字段传给模型
MixinSession        # 多模型故障转移，一个失败就切备用模型
```

这里有两个概念很重要：

### 6.1 Native 工具调用

Native 表示使用模型 API 原生支持的工具调用字段，例如 OpenAI 的 `tools` 或 Claude 的 `tool_use`。这通常更稳定。

### 6.2 文本协议工具调用

非 Native 模式会把工具说明写进 prompt，让模型输出类似：

```xml
<tool_use>{"name": "file_read", "arguments": {"path": "README.md"}}</tool_use>
```

然后 `ToolClient._parse_mixed_response()` 用正则和 JSON 解析出工具调用。

这种方式兼容性强，但更依赖模型是否遵守格式。

## 7. 记忆系统：项目所谓“自我进化”的基础

GenericAgent 的记忆主要放在 `memory/`。README 把它分成 L0 到 L4：

```text
L0 元规则           -> 系统行为规则和约束
L1 insight index    -> 记忆索引，帮助快速找相关 SOP
L2 global facts     -> 长期稳定事实
L3 task skills/SOP  -> 可复用任务流程
L4 session archive  -> 历史会话压缩归档
```

源码中最直接相关的是：

- `ga.py::get_global_memory()`：读取 `memory/global_mem_insight.txt` 和固定结构模板，注入系统提示词。
- `GenericAgentHandler.do_update_working_checkpoint()`：保存本任务短期记忆。
- `GenericAgentHandler.do_start_long_term_update()`：触发长期记忆沉淀。
- `reflect/scheduler.py`：周期性调用 `memory/L4_raw_sessions/compress_session.py` 压缩历史会话。

新手可以先这样理解：

```text
短期记忆：当前任务里别忘事
长期记忆：以后类似任务可以复用经验
SOP：把成功步骤写成可重复执行的流程
```

## 8. 浏览器控制原理

浏览器能力由三部分配合：

```text
assets/tmwd_cdp_bridge/  # Chrome 扩展
TMWebDriver.py           # Python 侧 WebDriver 桥
simphtml.py              # 页面 HTML 简化和 JS 执行包装
```

流程大致是：

1. 用户安装 `assets/tmwd_cdp_bridge/` 里的浏览器扩展。
2. 扩展和本地 Python 服务建立通信。
3. `TMWebDriver.py` 维护浏览器标签页 session。
4. `web_scan` 读取当前页面结构。
5. `web_execute_js` 在页面中执行 JavaScript。

这个设计的优点是能操作真实浏览器，保留用户登录态。比如你已经登录了某个网站，Agent 控制的是你的浏览器页面，而不是一个全新的无登录 headless 浏览器。

## 9. 前端系统

项目支持多种入口，但它们最终都把任务交给 `GeneraticAgent.put_task()`。

### 9.1 Streamlit 前端

`frontends/stapp.py` 是默认 GUI 前端。它做的事情包括：

- 初始化 `GeneraticAgent`
- 启动后台线程 `agent.run`
- 用聊天 UI 接收用户输入
- 流式展示 Agent 输出
- 支持停止任务、切换模型、重置会话等

### 9.2 其他前端

`frontends/` 里还有：

```text
tgapp.py       # Telegram
fsapp.py       # 飞书
wechatapp.py   # 微信
qqapp.py       # QQ
wecomapp.py    # 企业微信
dingtalkapp.py # 钉钉
qtapp.py       # Qt 桌面应用
```

它们的本质都是“消息入口适配器”。区别只是用户从哪里发消息，后端 Agent 逻辑是同一套。

## 10. Reflect 和定时任务

`agentmain.py` 支持 `--reflect` 参数。它会加载一个 Python 脚本，定期调用脚本里的 `check()` 函数。如果 `check()` 返回任务文本，就把它交给 Agent 执行。

`reflect/scheduler.py` 就是一个例子：

1. 定时扫描 `sche_tasks/` 里的 JSON 任务。
2. 判断是否到执行时间。
3. 防止重复执行。
4. 返回一段任务 prompt。
5. Agent 执行后把报告写入 `sche_tasks/done/`。

这是一种简单的“外部事件触发 Agent”的机制。

## 11. 为什么说它“极简”

GenericAgent 的核心不是写很多业务代码，而是抽象出几个稳定组件：

```text
LLM 适配层  -> 让不同模型看起来一样
Agent Loop  -> 负责多轮思考和工具调用
工具层      -> 提供少量真实世界操作能力
记忆层      -> 把经验保存下来
前端层      -> 提供不同聊天入口
```

这五层组合起来，就能从少量代码扩展出很多能力。

## 12. 新手阅读源码建议

### 第一步：理解任务队列

读 `agentmain.py`：

- `put_task()`：用户任务怎么进入队列
- `run()`：任务怎么被取出并执行
- `GeneraticAgent.__init__()`：模型和状态怎么初始化

重点理解：前端不会直接调用模型，而是把任务放进队列。

#### 第一次共读记录：任务队列模型

`agentmain.py` 可以先理解成 GenericAgent 的“调度中心”。它负责启动 Agent、加载模型配置、接收用户任务、把任务放进队列、启动 Agent Loop，并把执行过程返回给前端或命令行。

文件底部的启动入口中，核心代码是：

```python
agent = GeneraticAgent()
agent.next_llm(args.llm_no)
agent.verbose = args.verbose
threading.Thread(target=agent.run, daemon=True).start()
```

这几行做了三件事：

1. `GeneraticAgent()` 创建 Agent 对象。
2. `agent.next_llm(...)` 选择当前使用哪个 LLM 配置。
3. `threading.Thread(target=agent.run, daemon=True).start()` 启动后台线程，让 Agent 一直等待任务。

新手要先记住：Agent 启动后像一个长期运行的服务，不是每输入一句话才临时创建完整流程。它会先启动后台线程，然后持续监听任务队列。

命令行模式下，用户输入来自：

```python
q = input('> ').strip()
dq = agent.put_task(q, source='user')
```

`put_task()` 是任务进入系统的入口：

```python
def put_task(self, query, source="user", images=None):
    display_queue = queue.Queue()
    self.task_queue.put({
        "query": query,
        "source": source,
        "images": images or [],
        "output": display_queue
    })
    return display_queue
```

这里有两个队列：

```text
self.task_queue  -> 输入队列：前端/命令行把任务塞进来
display_queue    -> 输出队列：Agent 把执行过程和最终结果塞回去
```

用户的一句话会被包装成一个任务对象：

```python
{
    "query": "用户输入的内容",
    "source": "user",
    "images": [],
    "output": display_queue
}
```

后台线程运行的是 `run()`：

```python
def run(self):
    while True:
        task = self.task_queue.get()
```

`self.task_queue.get()` 会等待任务。只要 `put_task()` 放入一个任务，`run()` 就会取出来，解析出：

```python
raw_query, source, images, display_queue = (
    task["query"],
    task["source"],
    task.get("images") or [],
    task["output"]
)
```

所以第一条主线是：

```text
用户输入
  -> input() 或前端聊天框
  -> agent.put_task()
  -> self.task_queue.put(...)
  -> agent.run() 后台线程取出任务
  -> 后面才进入 LLM 和工具循环
```

这个设计让不同前端共用同一套后端。命令行、Streamlit、Telegram、飞书、微信等入口都可以把消息统一转换成 `agent.put_task(...)`。

#### 第二次共读记录：`GeneraticAgent.__init__()` 初始化过程

`GeneraticAgent.__init__()` 是 Agent 对象创建时自动执行的初始化函数。它主要做两类事情：

```text
1. 把 mykey.py / mykey.json 里的模型配置变成可调用的 LLM client
2. 初始化 Agent 运行时需要的状态，例如任务队列、历史记录、停止标记
```

入口代码：

```python
class GeneraticAgent:
    def __init__(self):
        script_dir = os.path.dirname(os.path.abspath(__file__))
        os.makedirs(os.path.join(script_dir, 'temp'), exist_ok=True)
        from llmcore import mykeys
        llm_sessions = []
```

这里先确保 `temp/` 目录存在。`temp/` 是运行时临时目录，后面模型日志、任务输入输出、临时脚本等都可能放在这里。

`from llmcore import mykeys` 看起来像普通导入，但 `llmcore.py` 中没有直接定义 `mykeys` 变量，而是用 `__getattr__()` 懒加载：

```python
def _load_mykeys():
    try:
        import mykey
        return {k: v for k, v in vars(mykey).items() if not k.startswith('_')}
    except ImportError:
        pass
    p = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'mykey.json')
    if not os.path.exists(p):
        raise Exception('[ERROR] mykey.py or mykey.json not found, please create one from mykey_template.')
    with open(p, encoding='utf-8') as f:
        return json.load(f)
```

这表示配置优先来自 `mykey.py`；如果没有，就尝试读取 `mykey.json`。如果两者都没有，程序会报错，提示从 `mykey_template.py` 创建配置。

接下来，`__init__()` 遍历所有配置项：

```python
for k, cfg in mykeys.items():
    if not any(x in k for x in ['api', 'config', 'cookie']):
        continue
    try:
        if 'native' in k and 'claude' in k:
            llm_sessions += [NativeToolClient(NativeClaudeSession(cfg=cfg))]
        elif 'native' in k and 'oai' in k:
            llm_sessions += [NativeToolClient(NativeOAISession(cfg=cfg))]
        elif 'claude' in k:
            llm_sessions += [ToolClient(ClaudeSession(cfg=cfg))]
        elif 'oai' in k:
            llm_sessions += [ToolClient(LLMSession(cfg=cfg))]
        elif 'mixin' in k:
            llm_sessions += [{'mixin_cfg': cfg}]
    except:
        pass
```

这里的关键点是：变量名决定 Session 类型。不是模型名决定，而是配置变量名中的关键词决定。

```text
变量名包含 native + claude -> NativeClaudeSession，再包一层 NativeToolClient
变量名包含 native + oai    -> NativeOAISession，再包一层 NativeToolClient
变量名包含 claude          -> ClaudeSession，再包一层 ToolClient
变量名包含 oai             -> LLMSession，再包一层 ToolClient
变量名包含 mixin           -> 先保存 mixin_cfg，稍后再组合多个 session
```

可以先这样理解：

```text
Session      -> 负责和具体模型 API 通信
ToolClient   -> 负责把工具调用协议接到 Agent Loop 上
NativeToolClient -> 使用 API 原生工具调用字段
ToolClient       -> 使用文本协议解析 <tool_use>
```

所以 `__init__()` 的这一段是在做“模型适配器装配”。

Mixin 处理在第二个循环：

```python
for i, s in enumerate(llm_sessions):
    if isinstance(s, dict) and 'mixin_cfg' in s:
        try:
            mixin = MixinSession(llm_sessions, s['mixin_cfg'])
            if isinstance(mixin._sessions[0], (NativeClaudeSession, NativeOAISession)):
                llm_sessions[i] = NativeToolClient(mixin)
            else:
                llm_sessions[i] = ToolClient(mixin)
        except Exception as e:
            print(f'[WARN] Failed to init MixinSession with cfg {s["mixin_cfg"]}: {e}')
```

`MixinSession` 可以理解为“多模型故障转移”。比如第一个模型请求失败，就自动切换到备用模型。它本身也会被包成 `NativeToolClient` 或 `ToolClient`，这样后面的 Agent Loop 不需要知道底层到底是单模型还是多个模型组合。

最后初始化运行状态：

```python
self.llmclients = llm_sessions
self.lock = threading.Lock()
self.task_dir = None
self.history = []
self.task_queue = queue.Queue()
self.is_running = False
self.stop_sig = False
self.llm_no = 0
self.inc_out = False
self.handler = None
self.verbose = True
self.llmclient = self.llmclients[self.llm_no]
```

这些变量可以按用途分组：

```text
self.llmclients -> 所有可用模型客户端
self.llmclient  -> 当前正在使用的模型客户端
self.llm_no     -> 当前模型编号

self.task_queue -> 输入任务队列
self.history    -> 简化后的对话历史摘要
self.handler    -> 当前任务的工具处理器

self.is_running -> 当前是否正在执行任务
self.stop_sig   -> 是否收到停止信号
self.lock       -> 线程锁，预留给并发控制
self.task_dir   -> 一次性任务模式下的任务目录
self.inc_out    -> 输出是否增量显示
self.verbose    -> 是否显示详细执行过程
```

这一节要抓住的主线：

```text
mykey.py / mykey.json
  -> llmcore 懒加载成 mykeys
  -> 根据变量名创建 Session
  -> 用 ToolClient 或 NativeToolClient 包装
  -> 放进 self.llmclients
  -> self.llmclient 指向当前使用的模型
```

所以 `GeneraticAgent.__init__()` 的本质是：把“配置文件中的模型信息”变成“Agent Loop 可以统一调用的模型客户端”，同时准备好任务队列和运行状态。

### 第二步：理解循环

读 `agent_loop.py`：

- `agent_runner_loop()`
- `BaseHandler.dispatch()`
- `StepOutcome`

重点理解：模型每一轮都可能调用工具；工具返回结果后，再进入下一轮。

### 第三步：理解工具

读 `ga.py`：

- `do_code_run()`
- `do_file_read()`
- `do_file_write()`
- `do_file_patch()`
- `do_web_scan()`
- `do_web_execute_js()`

重点理解：模型只负责决策，Python 工具负责真实执行。

### 第四步：理解模型适配

读 `llmcore.py`：

- `BaseSession`
- `ToolClient`
- `NativeToolClient`
- `NativeClaudeSession`
- `NativeOAISession`
- `MixinSession`

重点理解：不同 API 最终都被包装成统一的 `chat()` 接口。

### 第五步：理解前端

读 `frontends/stapp.py` 和 `launch.pyw`：

- Streamlit 如何显示聊天
- pywebview 如何包装成本地窗口
- 用户输入如何进入 `agent.put_task()`

## 13. 一个具体例子：让 Agent 读文件

假设用户说：

```text
帮我总结 README.md
```

可能发生的流程：

```text
1. frontends/stapp.py 收到用户输入
2. 调用 agent.put_task("帮我总结 README.md")
3. agentmain.py 的后台线程取出任务
4. get_system_prompt() 拼接系统提示词和记忆
5. agent_runner_loop() 调用 LLM
6. LLM 决定调用 file_read
7. BaseHandler.dispatch("file_read") 找到 do_file_read()
8. ga.py 读取 README.md 内容
9. 读取结果返回给 LLM
10. LLM 生成总结
11. 没有新工具调用，任务结束
12. 前端显示最终回答
```

这就是 Agent 的基本工作模式。

## 14. 学习这个项目需要掌握的 Python 知识

建议按优先级学习：

1. 函数、类、对象、实例方法
2. 字典、列表、字符串、JSON
3. 文件读写
4. `subprocess` 执行外部命令
5. 生成器 `yield`
6. 线程和队列 `threading`、`queue`
7. HTTP 请求 `requests`
8. 正则表达式 `re`
9. Streamlit 基础
10. 浏览器 DOM 和 JavaScript 基础

其中 `yield` 对理解本项目很重要。`agent_runner_loop()` 和很多工具函数都是生成器，这样前端可以边执行边显示输出，而不是等全部完成后一次性显示。

## 15. 学习路线

### 第 1 天：能跑起来

- 复制 `mykey_template.py` 为 `mykey.py`
- 填 API key
- 运行 `python agentmain.py`
- 让它完成一个简单文件任务

### 第 2 天：读主链路

- 读 `agentmain.py`
- 读 `agent_loop.py`
- 画出“用户输入 -> LLM -> 工具 -> LLM -> 输出”的流程图

### 第 3 天：读工具层

- 读 `ga.py`
- 重点看文件工具和 `code_run`
- 尝试让 Agent 创建、读取、修改一个测试文件

### 第 4 天：读模型层

- 读 `llmcore.py`
- 理解 Native 和非 Native 工具调用的差异
- 理解 `MixinSession` 为什么能故障转移

### 第 5 天：读前端和浏览器

- 读 `frontends/stapp.py`
- 读 `launch.pyw`
- 如有需要，再读 `TMWebDriver.py` 和 `memory/web_setup_sop.md`

## 16. 常见误区

### 误区 1：以为 Agent 直接“懂”电脑

不是。LLM 本身只会生成文本。真正操作电脑的是 Python 工具。

### 误区 2：以为工具越多越好

GenericAgent 的思路相反：工具少而通用。复杂能力通过 `code_run`、文件读写、网页 JS 控制组合出来。

### 误区 3：以为记忆就是聊天历史

聊天历史只是短期上下文。GenericAgent 更强调把稳定经验写成 SOP 或长期记忆，减少以后重复探索。

### 误区 4：以为前端是核心

前端只是入口。核心是 `agentmain.py + agent_loop.py + ga.py + llmcore.py`。

## 17. 可以动手做的小练习

1. 在项目里创建一个 `temp/hello.txt`，让 Agent 读取并总结。
2. 修改 `assets/tools_schema.json` 中某个工具描述，观察模型调用风格是否变化。
3. 在 `ga.py` 里给 `do_file_read()` 加一行调试输出，观察前端显示。
4. 阅读 `frontends/stapp.py`，找出“强行停止任务”按钮调用了哪个方法。
5. 阅读 `llmcore.py`，找出 OpenAI 兼容接口的请求在哪里发出。

## 18. 总结

GenericAgent 的核心思想可以压缩成一句话：

```text
用 LLM 做决策，用少量原子工具执行真实操作，用记忆系统沉淀可复用经验。
```

如果你是初学者，不需要一开始就理解所有细节。先抓住主线：

```text
agentmain.py 接任务
agent_loop.py 跑循环
llmcore.py 调模型
ga.py 执行工具
memory/ 保存经验
frontends/ 提供入口
```

只要这条主线清楚，再逐个模块深入，整个项目就会变得容易理解。

## 19. 完整源码导读

本节把项目主要源码按“从用户输入到真实执行”的顺序串起来。它不是逐行翻译，而是解释每个文件为什么存在、核心函数做什么、它和其他模块如何协作。初学者阅读时建议先理解调用链，再回到源码看细节。

### 19.1 总调用链

GenericAgent 的完整运行链路可以概括为：

```text
前端/命令行收到用户输入
  -> agent.put_task() 把任务放入队列
  -> agent.run() 后台线程取出任务
  -> get_system_prompt() 组装系统提示词和记忆
  -> 创建 GenericAgentHandler 工具处理器
  -> agent_runner_loop() 开始 LLM-工具循环
  -> llmcore client.chat() 调用模型
  -> 模型返回自然语言和工具调用
  -> handler.dispatch() 找到 do_xxx 工具方法
  -> ga.py 执行真实操作
  -> 工具结果返回给模型
  -> 任务完成后把最终输出传回 display_queue
```

这个链路中，最核心的四个文件是：

```text
agentmain.py   -> 应用调度中心
agent_loop.py  -> LLM 和工具反复交互的循环
ga.py          -> 真实工具能力
llmcore.py     -> 不同模型 API 的统一适配层
```

### 19.2 `agentmain.py`：应用调度中心

`agentmain.py` 是主入口。它负责初始化语言、工具 schema、记忆文件、模型客户端、任务队列，以及命令行/后台/reflect 等运行模式。

#### 语言和工具 schema

文件开头会根据系统语言设置 `GA_LANG`：

```python
os.environ.setdefault(
    'GA_LANG',
    'zh' if any(k in (locale.getlocale()[0] or '').lower() for k in ('zh', 'chinese')) else 'en'
)
```

这个变量影响系统提示词、记忆模板和部分工具描述的语言。

`load_tool_schema()` 读取工具定义：

```python
def load_tool_schema(suffix=''):
    global TOOLS_SCHEMA
    TS = open(os.path.join(script_dir, f'assets/tools_schema{suffix}.json'), 'r', encoding='utf-8').read()
    TOOLS_SCHEMA = json.loads(TS if os.name == 'nt' else TS.replace('powershell', 'bash'))
```

工具 schema 是给模型看的“工具说明书”。它告诉模型有哪些工具、每个工具有什么参数。Linux/macOS 下会把 `powershell` 替换成 `bash`，让同一份工具定义适配不同系统。

#### 记忆文件初始化

启动时会确保 `memory/` 存在，并创建基础记忆文件：

```text
memory/global_mem.txt
memory/global_mem_insight.txt
```

如果 `global_mem_insight.txt` 不存在，就从 `assets/global_mem_insight_template*.txt` 复制模板。这样即使新克隆项目，也有最基本的记忆结构。

#### `get_system_prompt()`

`get_system_prompt()` 负责生成每次任务开头的系统提示词：

```python
def get_system_prompt():
    with open(os.path.join(script_dir, f'assets/sys_prompt{lang_suffix}.txt'), 'r', encoding='utf-8') as f:
        prompt = f.read()
    prompt += f"\nToday: {time.strftime('%Y-%m-%d %a')}\n"
    prompt += get_global_memory()
    return prompt
```

它由三部分组成：

```text
assets/sys_prompt*.txt  -> Agent 基础行为规则
Today                   -> 当前日期
get_global_memory()     -> 记忆索引和长期记忆摘要
```

这说明模型每次执行任务时，不只是看到用户输入，还会看到项目预设规则和记忆系统。

#### `GeneraticAgent.__init__()`

这一段前面已经共读过。它的核心任务是：

```text
读取 mykey.py / mykey.json
  -> 根据变量名创建不同 Session
  -> 包装成 ToolClient 或 NativeToolClient
  -> 保存到 self.llmclients
  -> 初始化任务队列和运行状态
```

新手要记住：GenericAgent 通过“配置变量名”判断协议类型。例如 `native_oai_config` 会走 OpenAI 原生工具调用，`oai_config` 会走文本协议工具调用。

#### `next_llm()`

`next_llm()` 用于切换当前模型：

```python
def next_llm(self, n=-1):
    self.llm_no = ((self.llm_no + 1) if n < 0 else n) % len(self.llmclients)
    lastc = self.llmclient
    self.llmclient = self.llmclients[self.llm_no]
    self.llmclient.backend.history = lastc.backend.history
    self.llmclient.last_tools = ''
```

这里有两个细节：

1. 切换模型时会把旧模型的 `history` 复制给新模型，避免上下文断掉。
2. `last_tools` 清空，强制下一轮重新注入工具说明，避免新模型不知道可用工具。

后面根据模型名决定加载普通工具 schema 还是中文工具 schema：

```python
if 'glm' in name or 'minimax' in name or 'kimi' in name:
    load_tool_schema('_cn')
else:
    load_tool_schema()
```

这是对部分中文模型的兼容处理。

#### `put_task()` 和 `run()`

`put_task()` 是所有前端共用的任务入口：

```python
def put_task(self, query, source="user", images=None):
    display_queue = queue.Queue()
    self.task_queue.put({"query": query, "source": source, "images": images or [], "output": display_queue})
    return display_queue
```

`run()` 是后台线程持续执行的主循环。它会：

1. 从 `self.task_queue` 取任务。
2. 处理 `/session.xxx=yyy`、`/resume` 等斜杠命令。
3. 创建系统提示词。
4. 创建 `GenericAgentHandler`。
5. 调用 `agent_runner_loop()`。
6. 把中间输出和最终结果放回 `display_queue`。

其中最关键的调用是：

```python
gen = agent_runner_loop(
    self.llmclient,
    sys_prompt,
    user_input,
    handler,
    TOOLS_SCHEMA,
    max_turns=40,
    verbose=self.verbose
)
```

这里把五个核心对象交给 Agent Loop：

```text
self.llmclient -> 当前模型客户端
sys_prompt     -> 系统提示词和记忆
user_input     -> 用户当前任务
handler        -> 工具执行器
TOOLS_SCHEMA   -> 工具定义
```

#### 命令行、任务模式、Reflect 模式

`if __name__ == '__main__'` 下支持三种主要运行方式：

```text
默认命令行模式：循环 input('> ')
--task          ：文件 IO 一次性任务模式，适合外部进程调用
--reflect       ：定期调用某个脚本的 check()，返回任务就执行
```

这说明 `agentmain.py` 不只服务聊天界面，也可以作为自动化任务执行器。

### 19.3 `agent_loop.py`：Agent Loop 核心循环

`agent_loop.py` 是最值得新手反复看的文件，因为它展示了 Agent 的基本工作方式。

#### `StepOutcome`

```python
@dataclass
class StepOutcome:
    data: Any
    next_prompt: Optional[str] = None
    should_exit: bool = False
```

每个工具执行后都返回一个 `StepOutcome`：

```text
data        -> 工具执行结果
next_prompt -> 下一轮喂给模型的提示；为空表示当前任务可以结束
should_exit -> 是否直接退出，比如 ask_user 需要等待用户
```

#### `BaseHandler.dispatch()`

`dispatch()` 根据工具名找到对应方法：

```python
method_name = f"do_{tool_name}"
```

例如：

```text
file_read      -> do_file_read()
code_run       -> do_code_run()
web_execute_js -> do_web_execute_js()
```

这是一种简单但有效的工具分发机制。工具 schema 里的名字和 `GenericAgentHandler` 里的 `do_xxx` 方法形成约定。

#### `agent_runner_loop()`

核心循环结构可以简化为：

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": user_input}
]

while turn < handler.max_turns:
    response = client.chat(messages=messages, tools=tools_schema)
    tool_calls = parse response.tool_calls
    for tool_call in tool_calls:
        outcome = handler.dispatch(tool_name, args, response)
    messages = [{"role": "user", "content": next_prompt, "tool_results": tool_results}]
```

新手要注意：`messages` 每轮只设置成新的 user 消息，但注释里说：

```python
# just new message, history is kept in *Session
```

也就是说，完整历史不是一直塞在 `agent_runner_loop()` 的局部变量里，而是保存在 `llmcore` 的 Session 对象中。这样可以减少每轮显式拼装历史的复杂度。

#### 没有工具调用时的 `no_tool`

如果模型没有返回工具调用：

```python
if not response.tool_calls:
    tool_calls = [{'tool_name': 'no_tool', 'args': {}}]
```

`no_tool` 是内部特殊工具，不在 `tools_schema` 里。它用于判断模型是否已经给出最终回答，或者是否需要强制模型重新调用工具。

#### 工具结果如何回传

工具结果会被包装成：

```python
tool_results.append({'tool_use_id': tid, 'content': datastr})
```

下一轮会作为 `tool_results` 传给模型：

```python
messages = [{"role": "user", "content": next_prompt, "tool_results": tool_results}]
```

这就是“工具执行结果反馈给模型”的关键。

### 19.4 `ga.py`：工具实现层

`ga.py` 是 Agent 能真正操作电脑的地方。LLM 不会直接读写文件或执行命令，它只是请求调用工具，真正执行发生在这里。

#### `code_run()`

`code_run()` 支持 Python 和 shell 命令：

```text
python -> 写入临时 .ai.py 文件，用当前 Python 解释器执行
bash   -> Linux/macOS 下用 bash -c 执行
powershell -> Windows 下用 PowerShell 执行
```

它使用 `subprocess.Popen()` 启动进程，并开线程读取 stdout。这样前端可以看到流式输出，而不是等命令完成后一次性显示。

它还支持：

```text
timeout     -> 超时终止
stop_signal -> 用户点击停止时终止
smart_format -> 长输出截断，避免撑爆上下文
```

这是最通用的工具，也是 Agent 扩展能力的基础。通过它，Agent 可以安装包、运行脚本、检查环境、调用系统命令。

#### `file_read()`

`file_read()` 支持：

```text
按路径读取文件
从指定行号开始
按关键字搜索上下文
显示行号
长文件截断
```

如果文件不存在，它还会在常见根目录中扫描同名文件，给出候选路径。这对模型很有用，因为模型经常只知道文件名，不知道准确路径。

#### `file_patch()`

`file_patch()` 做精确替换：

```python
count = full_text.count(old_content)
if count == 0: return error
if count > 1: return error
updated_text = full_text.replace(old_content, new_content)
```

它要求 `old_content` 只能出现一次。这是为了避免模型误把多个相同片段一起改掉。

#### `file_write()`

`do_file_write()` 要求模型把写入内容放在 `<file_content>` 标签或代码块中。这样可以避免工具参数里塞超长字符串导致 JSON 转义问题。

它支持三种模式：

```text
overwrite -> 覆盖
append    -> 追加
prepend   -> 前插
```

#### `web_scan()` 和 `web_execute_js()`

`web_scan()` 读取浏览器标签页和当前页面的简化 HTML：

```text
TMWebDriver -> 连接浏览器扩展
simphtml    -> 简化 DOM，减少 token
```

`web_execute_js()` 在当前浏览器页面执行 JavaScript。它可以读取 DOM、点击按钮、填写输入框、跳转页面，也可以把长结果保存到文件。

新手要理解：GenericAgent 的网页自动化不是只靠截图，而是通过真实浏览器中的 JS 直接操作网页结构。

#### `GenericAgentHandler`

`GenericAgentHandler` 继承 `BaseHandler`，把通用分发机制和具体工具实现连接起来。

主要方法：

```text
do_code_run()
do_ask_user()
do_web_scan()
do_web_execute_js()
do_file_patch()
do_file_write()
do_file_read()
do_update_working_checkpoint()
do_no_tool()
do_start_long_term_update()
turn_end_callback()
```

其中 `do_no_tool()` 很重要。它会处理这些情况：

```text
模型空回复 -> 要求重试
响应不完整 -> 要求重试
只输出大代码块但没调用工具 -> 要求模型说明或调用工具
Plan 模式未验证就声称完成 -> 拦截
真正最终回答 -> 返回 StepOutcome(response, next_prompt=None)
```

`turn_end_callback()` 每轮结束都会把 `<summary>` 里的内容写入 `history_info`。如果模型没有写 `<summary>`，它会根据工具调用生成摘要，并在下一轮提示模型遵守协议。

#### 记忆相关工具

`do_update_working_checkpoint()` 保存当前任务短期记忆：

```text
关键用户需求
已发现的路径
当前进度
失败后的新策略
```

`do_start_long_term_update()` 用于任务结束后的长期记忆沉淀，会引导模型读取记忆管理 SOP，再更新 L2/L3 记忆。

`get_global_memory()` 读取 `memory/global_mem_insight.txt` 和固定结构模板，并注入系统提示词。

### 19.5 `llmcore.py`：模型适配层

`llmcore.py` 的目标是把不同模型 API 包装成统一接口，让 Agent Loop 不关心底层是 Claude、OpenAI、MiniMax，还是中转服务。

#### 配置加载

`_load_mykeys()` 优先导入 `mykey.py`，失败后读取 `mykey.json`。`__getattr__()` 用懒加载方式生成 `mykeys` 和 `proxies`。

这意味着只有真正访问 `llmcore.mykeys` 时才读取配置。

#### 历史压缩和裁剪

`compress_history_tags()` 会压缩旧消息里的 `<thinking>`、`<tool_use>`、`<tool_result>` 等标签内容，减少 token 消耗。

`trim_messages_history()` 根据 `context_win` 裁剪历史，防止上下文过长。

这些函数是项目“省 token”的基础之一。

#### SSE 解析

`_parse_claude_sse()` 和 `_parse_openai_sse()` 负责解析流式响应。

它们会把不同 API 的返回统一成 content blocks：

```text
{"type": "text", "text": "..."}
{"type": "tool_use", "id": "...", "name": "...", "input": {...}}
```

这一步很关键。只有统一成相同结构，后面的 `MockResponse` 和 Agent Loop 才能复用。

#### `BaseSession`

`BaseSession` 保存通用配置：

```text
api_key
api_base
model
context_win
history
proxy
timeout
temperature
max_tokens
reasoning_effort
thinking_type
```

它的 `ask()` 方法负责：

1. 把用户消息加入历史。
2. 裁剪历史。
3. 调用具体子类的 `raw_ask()`。
4. 把模型回复加入历史。

#### `ClaudeSession` 和 `LLMSession`

`ClaudeSession` 调 Claude messages API。

`LLMSession` 调 OpenAI 兼容接口，包括 `chat_completions` 和 `responses` 两种模式。

这两个属于“非 Native”路径，工具调用通常通过文本协议解析。

#### `NativeClaudeSession` 和 `NativeOAISession`

Native 表示工具调用走 API 原生字段：

```text
Claude -> tools + tool_use/tool_result
OpenAI -> tools/function calling
```

`NativeClaudeSession.raw_ask()` 会把工具 schema 转成 Claude 工具格式，并发送到 API。模型返回的 `tool_use` block 会被转成 `MockToolCall`。

`NativeOAISession` 复用 NativeClaudeSession 的结构，但底层调用 `_openai_stream()`。

#### `ToolClient`

`ToolClient` 用于非 Native 模式。它会把工具定义写进 prompt：

```text
### Tools (mounted, always in effect):
...
Format: <tool_use>{"name": "...", "arguments": {...}}</tool_use>
```

模型需要按文本协议输出 `<tool_use>`，然后 `_parse_mixed_response()` 用正则和 JSON 解析工具调用。

这条路径兼容性强，但要求模型严格遵守格式。

#### `NativeToolClient`

`NativeToolClient` 用于 Native 模式。它把 `tools` 直接设置到 backend 上，让 API 原生处理工具调用。

它还维护 `_pending_tool_ids`，确保上一轮的 tool_use 有对应 tool_result。Claude 这类 API 对 tool_use/tool_result 配对比较严格，所以这里要做补齐。

#### `MixinSession`

`MixinSession` 是多模型故障转移层。它包装多个 Session：

```text
主模型失败 -> 切备用模型
备用模型成功 -> 可在一段时间后 spring back 回主模型
```

它还会广播这些属性：

```text
system
tools
temperature
max_tokens
reasoning_effort
history
```

这样切换模型时，工具、系统提示词、历史不会丢。

### 19.6 `assets/tools_schema.json`：工具说明书

这个文件不是 Python 源码，但它决定模型“知道”哪些工具。里面每个工具都是 JSON schema：

```text
name        -> 工具名
description -> 给模型看的说明
parameters  -> 参数结构
```

工具 schema 和 `ga.py` 的关系是：

```text
tools_schema.json 中的 code_run
  -> 模型调用 code_run
  -> agent_loop.py dispatch()
  -> ga.py GenericAgentHandler.do_code_run()
```

所以新增工具通常要同时做两件事：

1. 在 `ga.py` 增加 `do_xxx()`。
2. 在 `assets/tools_schema.json` 增加工具描述。

### 19.7 `launch.pyw`：桌面 GUI 启动器

`launch.pyw` 的作用是启动默认桌面体验。

主要流程：

```text
find_free_port() 找一个本地端口
start_streamlit(port) 启动 frontends/stapp.py
可选启动 Telegram/QQ/飞书/企业微信/钉钉/调度器
启动 idle_monitor()
用 pywebview 打开本地 Streamlit 页面
```

`inject(text)` 会通过 JS 找到 Streamlit 聊天输入框并自动提交内容。`idle_monitor()` 用它在用户长时间离开后注入自主任务。

`launch.pyw` 不处理 Agent 核心逻辑，它只是“桌面外壳 + 多服务启动器”。

### 19.8 `frontends/stapp.py`：默认 Streamlit 前端

`frontends/stapp.py` 是默认聊天 UI。

#### 初始化

```python
@st.cache_resource
def init():
    agent = GeneraticAgent()
    threading.Thread(target=agent.run, daemon=True).start()
    return agent
```

这里和命令行一样，也是创建 `GeneraticAgent`，然后启动后台线程。

#### 侧边栏

`render_sidebar()` 提供：

```text
切换备用链路
强行停止任务
重新注入工具
启动桌面宠物
允许/禁止自主行动
```

这些按钮本质上是在调用 `agent.next_llm()`、`agent.abort()` 或修改 session state。

#### 聊天流式输出

`agent_backend_stream(prompt)`：

```python
display_queue = agent.put_task(prompt, source="user")
while True:
    item = display_queue.get(timeout=1)
    if 'next' in item: yield item['next']
    if 'done' in item: yield item['done']; break
```

这正好对应前面讲过的输出队列。前端不直接调用模型，只消费 `display_queue`。

`fold_turns()` 会把每一轮 `LLM Running (Turn N)` 折叠起来，让界面更清爽。

### 19.9 其他前端：统一入口适配器

`frontends/` 里有多种平台入口：

```text
tgapp.py       -> Telegram
fsapp.py       -> 飞书
wechatapp.py   -> 微信
qqapp.py       -> QQ
wecomapp.py    -> 企业微信
dingtalkapp.py -> 钉钉
qtapp.py       -> Qt 桌面应用
stapp2.py      -> 另一个 Streamlit UI
```

它们的共同点是：收到平台消息后，最终都调用 `agent.put_task(...)`。

例如：

```text
Telegram: agent.put_task(prompt, source="telegram")
飞书:     agent.put_task(user_input, source="feishu", images=image_paths)
微信:     agent.put_task(prompt, source="wechat")
Qt:       self.agent.put_task(full_prompt, source="user")
```

所以它们只是“消息协议适配层”。平台 SDK、鉴权、消息格式、文件下载各不相同，但后端 Agent 逻辑统一。

`frontends/chatapp_common.py` 抽象了部分聊天平台通用逻辑：

```text
清理回复
提取生成文件
拆分长文本
权限检查
单实例端口锁
AgentChatMixin
```

`frontends/continue_cmd.py` 负责 `/continue` 相关功能：扫描 `temp/model_responses`，列出历史会话，恢复 backend history。

### 19.10 `TMWebDriver.py`：浏览器通信桥

`TMWebDriver.py` 负责 Python 和浏览器扩展通信。

项目的浏览器控制不是单纯 headless 自动化，而是通过 `assets/tmwd_cdp_bridge/` 这个 Chrome 扩展连接真实浏览器。

核心概念：

```text
Session     -> 一个浏览器标签页/连接
TMWebDriver -> 管理 session，发送命令，等待结果
```

`execute_js()` 会把 JS 发送给指定标签页执行，并等待浏览器扩展返回结果。

`jump()` 和 `newtab()` 是对常见网页动作的封装：

```python
def jump(self, url, timeout=10):
    self.execute_js(f"window.location.href='{url}'", timeout=timeout)
```

浏览器链路大致是：

```text
ga.py web_execute_js()
  -> simphtml.execute_js_rich()
  -> TMWebDriver.execute_js()
  -> Chrome 扩展执行 JS
  -> 返回结果给 Python
```

### 19.11 `simphtml.py`：网页内容简化和 JS 执行增强

网页原始 HTML 往往非常大，不适合直接塞给模型。`simphtml.py` 的作用就是把网页简化成模型更容易理解的内容。

主要能力：

```text
optimize_html_for_tokens() -> 删除噪声属性和无用节点
get_main_block()           -> 获取主要页面区域
get_html()                 -> 返回简化后的 HTML 或文本
smart_truncate()           -> 按预算截断
execute_js_rich()          -> 执行 JS，并监控页面变化
find_changed_elements()    -> 比较执行前后 DOM 差异
```

`execute_js_rich()` 很关键。它不只是执行 JS，还会：

1. 执行前记录页面 HTML。
2. 执行用户 JS。
3. 等待页面变化。
4. 再次获取 HTML。
5. 返回变化摘要。

这样模型点击按钮后，不需要立刻全量扫描页面，也能知道页面发生了什么变化。

### 19.12 `reflect/`：外部触发和计划任务

`agentmain.py --reflect SCRIPT` 会加载一个脚本，定期调用 `check()`。

#### `reflect/scheduler.py`

这是计划任务调度器。它扫描 `sche_tasks/` 目录下的 JSON 文件，判断：

```text
enabled 是否开启
repeat 是 daily/weekly/monthly/once 还是 every_x
schedule 是否到点
max_delay 是否还在允许延迟窗口内
done/ 中是否已有近期执行记录
```

如果满足条件，就返回一段任务 prompt 给 Agent 执行。

它还会每 12 小时尝试调用 L4 会话归档：

```text
memory/L4_raw_sessions/compress_session.py
```

#### `reflect/autonomous.py`

这个文件是自主行动触发器示例。它定期返回一个提示，让 Agent 阅读自主操作 SOP 并执行自动任务。

### 19.13 `memory/`：记忆、SOP 和增强工具

`memory/` 既有 Markdown SOP，也有一些 Python 辅助工具。

#### SOP 文件

常见 SOP：

```text
memory/web_setup_sop.md              -> 浏览器工具初始化
memory/memory_management_sop.md      -> 记忆更新流程
memory/scheduled_task_sop.md         -> 定时任务执行流程
memory/autonomous_operation_sop.md   -> 自主操作流程
memory/plan_sop.md                   -> 计划模式流程
```

这些文件不是普通文档，它们会被 Agent 在任务中读取，变成可执行操作指南。

#### Python 工具

部分辅助脚本：

```text
memory/ocr_utils.py       -> OCR 图片/屏幕/窗口
memory/vision_api.template.py -> 视觉模型调用模板
memory/adb_ui.py          -> Android UI dump 和 tap
memory/ljqCtrl.py         -> Windows 鼠标键盘和窗口截图
memory/procmem_scanner.py -> 进程内存扫描
memory/ui_detect.py       -> UI 元素检测
memory/keychain.py        -> 简单密钥访问封装
```

这些脚本体现了 GenericAgent 的扩展方式：核心工具保持少量，但遇到具体能力时，可以把辅助脚本沉淀到 `memory/`，以后通过 SOP 调用。

### 19.14 `hub.pyw`：服务启动中心

`hub.pyw` 是一个 GUI 启动中心，用于发现和管理多个服务。它可以看作 `launch.pyw` 的更完整服务管理版本。

主要结构：

```text
acquire_singleton() -> 单实例锁
discover_services() -> 发现可启动服务
ServiceManager      -> 管理服务进程
LauncherApp         -> GUI 应用
```

它不是 Agent 核心链路的一部分，但对桌面端使用体验有帮助。

### 19.15 `tests/`：测试

当前测试主要集中在 MiniMax 集成相关：

```text
tests/test_minimax.py
tests/test_minimax_integration.py
```

测试覆盖并不完整。新手不要误以为没有测试的模块不重要；这个项目很多逻辑是运行时自动化和外部 API，传统单元测试覆盖较少。

如果以后要增强测试，可以优先测：

```text
agent_loop.py 的 StepOutcome/dispatch/循环分支
ga.py 的 file_read/file_patch/smart_format
llmcore.py 的工具调用解析
frontends/continue_cmd.py 的会话恢复解析
```

### 19.16 新手最终心智模型

读完整个项目后，可以把 GenericAgent 记成五层：

```text
入口层：frontends/、launch.pyw、agentmain.py 命令行
调度层：GeneraticAgent、任务队列、display_queue
循环层：agent_runner_loop、StepOutcome、BaseHandler.dispatch
能力层：ga.py 工具、TMWebDriver、simphtml、memory 辅助脚本
模型层：llmcore Session、ToolClient、NativeToolClient、MixinSession
```

一条任务的生命线是：

```text
用户说一句话
  -> 前端调用 put_task
  -> run 取出任务
  -> agent_runner_loop 调模型
  -> 模型选择工具
  -> ga.py 执行工具
  -> 工具结果回到模型
  -> 模型继续或结束
  -> display_queue 返回给用户
```

这就是理解 GenericAgent 源码的主轴。后续阅读任何文件，都可以问三个问题：

```text
它处在这条链路的哪一层？
它接收什么输入，输出什么结果？
它是给模型看的，还是给 Python 运行时用的？
```

只要能回答这三个问题，源码就不再是一堆散乱文件，而是一套清晰的 Agent 执行系统。
