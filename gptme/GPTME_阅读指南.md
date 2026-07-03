# gptme 源码阅读指南(Java 背景版)

> 面向:Java 背景、Python 边读边学的贡献者。
> 目标:读懂项目 → 能修 bug → 能提 PR。
> 用法:按四步推进,每步末尾有「过关自查」,能用自己的话答出来再进下一步。
> 版本基准:本文基于本地仓库 gptme v0.31.0(master)编写,行号可能随更新漂移,以符号名(函数/类名)为准去搜。

---

## 第 1 步:项目全景(不看代码)

### 这是个什么项目

**gptme 是一个跑在终端里的通用 AI Agent(智能体)**,2023 年春就存在,是最早的 agent CLI 之一。定位类似 Claude Code / Codex CLI,但强调三点:

1. **Provider 无关**:接 Anthropic、OpenAI、Google、DeepSeek、OpenRouter,也能接本地 llama.cpp。
2. **本地优先**:对话记录、配置全在本地,不依赖云服务。
3. **无约束(unconstrained)**:自带 shell、Python 解释器、文件编辑、浏览器、视觉、computer use 等工具,agent 想干什么都有对应的"手"。

### 一句话说清输入/输出/核心价值

- **输入**:用户在终端敲的一句自然语言(如"帮我把这个脚本的 bug 修了"),外加当前工作目录里的文件上下文。
- **输出**:LLM 生成的回复,其中嵌着**代码块形式的工具调用**(比如一段 ```shell 块),gptme 解析并真实执行它们(跑命令、改文件),把执行结果再喂回 LLM,循环往复直到任务完成。
- **核心价值**:把"LLM 会说"变成"LLM 会做"——一个消息循环 + 一套可插拔工具系统,让模型能在你的机器上实际操作。

用 Java 的话类比:它像一个「REPL 驱动的工作流引擎」,LLM 是规则引擎,tools 是一组 `Command` 实现类,chat loop 是调度器。

### 跑通最小示例(quickstart)

```bash
# 安装(官方推荐 pipx,类似把 jar 装成全局命令)
pipx install gptme

# 配置 API key(首次运行会交互式引导,也可用环境变量)
export ANTHROPIC_API_KEY=<key>   # 或 OPENAI_API_KEY / OPENROUTER_API_KEY

# 最小交互
gptme "用 python 算一下 42 的阶乘"

# 非交互一次性执行(适合脚本/CI)
gptme --non-interactive "创建一个 hello.py 并运行它"
```

开发者模式(读源码时建议这样跑,改代码立即生效):

```bash
cd E:\projects\github\gptme
pipx install poetry        # 项目用 poetry 管理,相当于 Maven
poetry install             # ≈ mvn install(装依赖到独立 venv)
poetry run gptme "hello"   # 从源码启动
```

> Windows 注意:gptme 主力开发环境是 Linux/macOS,部分工具(tmux、pick)在 Windows 上不可用(pyproject.toml 里 `pick` 标了 `sys_platform != 'win32'`)。建议用 WSL 或 Git Bash,遇到"某工具不可用"先怀疑平台差异。

### 过关自查

1. gptme 和"直接在网页上用 ChatGPT"本质区别是什么?(答案关键词:工具执行、本地环境、循环反馈)
2. 一次 `gptme "修个 bug"` 的完整生命周期里,LLM 被调用了几次?(提示:不止一次——想想工具结果要不要回传)

---

## 第 2 步:工程骨架

### 顶层目录注释图

```
gptme/
├── pyproject.toml      # ≈ pom.xml:元数据、依赖、入口命令、lint/test 配置(全在这一个文件)
├── poetry.lock         # ≈ 依赖锁文件(Maven 没有直接对应,类似 Gradle 的 lockfile)
├── Makefile            # 常用命令入口:make test / make lint / make typecheck
├── AGENTS.md           # 给 AI agent(和你)看的贡献规范:分支命名、commit 格式、代码风格
├── gptme/              # ★ 主 Python 包(= Java 的 src/main/java 顶层包)
├── tests/              # pytest 测试(= src/test/java,但是平铺的 test_*.py 文件)
├── docs/               # Sphinx 文档源码(.rst/.md),gptme.org/docs 就是它构建的
├── scripts/            # 构建/运维脚本
├── webui/              # Web 前端(TypeScript,配合 gptme-server 用)
├── tauri/              # 桌面 App 壳
├── gptme-extension/    # VS Code 扩展
└── .github/workflows/  # CI:test/lint/eval/release 等
```

### 主包 `gptme/` 内部(重点)

按"分层架构"类比,大致是:

```
gptme/
├── cli/            # 【Controller 层】命令行入口。main.py 是 `gptme` 命令本体(click 框架)
├── chat.py         # 【Service 层核心】chat() 主循环 + step() 单步逻辑 ← 全项目最重要的文件
├── message.py      # 【领域模型】Message dataclass:role + content,一切数据的基本单位
├── logmanager/     # 【DAO 层】对话持久化:Log(不可变消息列表)、LogManager(读写 logdir)
├── tools/          # 【策略/插件层】所有工具:shell.py、python.py、patch.py、browser.py…
│   └── base.py     #   ToolSpec(工具定义)+ ToolUse(一次调用)——核心抽象,必读
├── llm/            # 【外部网关层】LLM 提供商适配:llm_anthropic.py / llm_openai.py
│   └── models.py   #   模型注册表(哪个模型多少钱、支持什么能力)
├── config/         # 配置加载(gptme.toml,分 user/project/chat 三级)
├── prompts/        # 系统提示词的组装(告诉 LLM 有哪些工具、怎么用)
├── commands/       # 用户斜杠命令(/undo /edit /tokens 等,不是 LLM 的工具)
├── hooks/          # 横切关注点:生命周期钩子(≈ Spring 的事件监听/AOP)
├── server/         # Flask REST API(gptme-server 命令,给 webui 用)
├── mcp/            # Model Context Protocol 客户端(接第三方工具服务器)
├── acp/            # Agent Client Protocol(接 Zed 等编辑器)
├── eval/           # 评测框架(gptme-eval,给模型跑基准测试)
├── agent/          # 持久化自主 agent 相关
├── context/        # 上下文压缩/管理
├── plugins/        # 插件系统(通过 entry_points 发现,≈ Java SPI)
└── util/           # 工具函数杂项
```

**先读哪些、跳过哪些**:第一遍只需要 `cli/main.py`、`chat.py`、`message.py`、`tools/base.py`、`tools/shell.py`、`llm/__init__.py`、`logmanager/`。`server/`、`eval/`、`acp/`、`mcp/`、`telemetry` 全部先跳过——它们是外围接入方式,不影响核心理解。

### pyproject.toml 对照 pom.xml 讲

| pyproject 片段 | Java 对应 | 说明 |
|---|---|---|
| `[project]` name/version | `<groupId>/<artifactId>/<version>` | 包元数据 |
| `[project.scripts]` `gptme = "gptme.cli.main:main"` | `Main-Class` + 打包出的可执行脚本 | **入口注册表**:装好后 `gptme` 命令 = 调用 `gptme/cli/main.py` 的 `main()` 函数。注意这里注册了近 20 个命令(gptme-server、gptme-eval…),都是同一个包的不同入口 |
| `[project.entry-points."gptme.plugins"]` | Java SPI(`META-INF/services`) | 插件发现机制:第三方包声明这个 entry point 就能被 gptme 加载 |
| `[tool.poetry.dependencies]` | `<dependencies>` | 关键依赖见下表 |
| `[tool.poetry.extras]` | Maven profiles / optional deps | `pip install gptme[server]` 才装 flask,按需安装 |
| `[tool.ruff]` / `[tool.mypy]` / `[tool.pytest.ini_options]` | checkstyle / 编译器 / surefire 配置 | lint、类型检查、测试配置都住在这个文件里 |

**关键运行时依赖**:

- `click`:CLI 框架(≈ picocli),用装饰器声明参数
- `rich`:终端彩色渲染
- `anthropic` / `openai`:两大官方 SDK,`llm/` 层就是包装它们
- `ipython`:python 工具的执行内核(agent 写的 Python 代码在 IPython 里跑)
- `bashlex`:解析 shell 命令做安全检查
- `mcp`:MCP 协议 SDK
- `tiktoken`:token 计数
- `flask`(可选):HTTP server
- `playwright`(可选):浏览器工具

### 测试怎么跑

```bash
poetry run pytest tests/ -m "not slow and not eval" -x   # 快速测试,失败即停
# 或
make test                # Makefile 里就是上面这句的封装
make typecheck           # mypy,≈ javac 的类型检查(Python 类型是可选的,靠 mypy 兜底)
make lint                # ruff,≈ checkstyle + 部分 spotbugs
```

pytest 约定(对照 JUnit):

- 测试文件 `tests/test_*.py`,函数 `def test_xxx()` —— 不需要类,不需要注解,**函数名前缀就是发现机制**
- `conftest.py` = 共享 fixture 的地方,fixture ≈ JUnit 的 `@BeforeEach` + 依赖注入的合体:测试函数声明一个同名参数,pytest 自动注入
- `-m "not slow"`:按 marker 过滤,≈ JUnit 的 `@Tag`
- 很多测试标了 `requires_api`(要真实 API key),本地没 key 时跳过它们是正常的

### 过关自查

1. 用户敲 `gptme "hi"`,操作系统层面是怎么找到 Python 代码执行的?(关键词:entry point、console script)
2. `tools/` 和 `commands/` 都是"命令",区别是什么?(一个给 LLM 用,一个给用户用)
3. `pip install gptme` 之后为什么没有 flask?(extras)

---

## 第 3 步:核心链路走读(最重要)

### 主流程:一次任务从输入到完成

先给结论——整条链就 6 跳,背下来:

```
gptme/cli/main.py: main()          # ① CLI 解析(click),组装初始消息
        │
        ▼
gptme/chat.py: chat()              # ② 会话初始化:init 工具/模型、加载 LogManager、切到 workspace
        │
        ▼
gptme/chat.py: _run_chat_loop()    # ③ 外层循环:取用户输入 → 反复调 step() 直到本轮结束
        │
        ▼
gptme/chat.py: step()              # ④ 单步:prepare_messages(裁剪上下文)
        │                          #        → reply() 调 LLM 拿回复
        │                          #        → execute_msg() 执行回复里的工具
        ├──▶ gptme/llm/__init__.py: reply()      # ⑤ 路由到 llm_anthropic / llm_openai,流式拿 Message
        └──▶ gptme/tools/__init__.py: execute_msg()
                     │
                     ▼
             gptme/tools/base.py: ToolUse.iter_from_content()  # 从回复文本里解析出工具调用
                     └─▶ ToolUse.execute()                     # ⑥ 找到对应 ToolSpec,真正执行
                          └─▶ 返回 role="system" 的结果 Message,append 回 Log
```

**循环的本质**:`step()` 是个**生成器**(generator,见下面语法专栏),每次 yield 出新消息(LLM 回复、工具输出)。外层循环把这些消息 append 进 Log;只要 LLM 的回复里还有未执行的工具调用,就继续 `step()`,LLM 看到工具输出后接着往下干——这就是 "agent loop"。直到 LLM 的回复里不再包含工具调用,才回头等用户输入。

### 核心抽象(项目的"领域模型")

| 抽象 | 位置 | 是什么 | Java 类比 |
|---|---|---|---|
| `Message` | message.py | `role`(user/assistant/system) + `content` + 时间戳等,**frozen dataclass** | 不可变 `record`;改字段用 `.replace()` 返回新对象,类似 `withX()` |
| `Log` | logmanager/ | 不可变的 `Message` 列表,append 返回新 Log | `List.copyOf` 风格的持久化数据结构 |
| `LogManager` | logmanager/manager.py | 负责把 Log 读写到磁盘 logdir(JSONL 文件) | DAO / Repository |
| `ToolSpec` | tools/base.py:306 | **工具的定义**:name、desc、instructions(进系统提示词)、`execute` 函数、`block_types`(认领哪种代码块) | 接口 + 元数据注解的合体;全部工具注册成一个列表 ≈ 策略模式的策略注册表 |
| `ToolUse` | tools/base.py:662 | **一次具体调用**:从 LLM 回复文本中解析出「工具名 + 参数 + 代码内容」,`.execute()` 分派给对应 ToolSpec | Command 模式:`ToolSpec` 是 handler,`ToolUse` 是 command 对象 |
| `reply()` | llm/__init__.py:263 | 按 model 前缀(`anthropic/...`、`openai/...`)路由到具体 provider 模块 | 简单工厂 + 适配器:`llm_anthropic.py`/`llm_openai.py` 是两个 Adapter |
| hooks | hooks/ | `HookType.SESSION_START` 等生命周期事件,工具/插件可挂钩子 | Spring 事件机制 / Servlet Filter |

**一个关键设计决策**:gptme 的默认工具调用格式不是各家 API 的原生 "function calling",而是**让 LLM 直接输出 markdown 代码块**(如 ```shell 块),由 `ToolUse.iter_from_content()` 正则/流式解析。好处是任何模型(包括不支持 function calling 的本地模型)都能用;`tool_format` 配置项可切换成原生 `tool` 格式。理解了这一点,`tools/base.py` 里大量解析代码就说得通了。

### 走读任务(自己动手,用搜索不用顺序读)

在项目根目录用 grep(或 IDE 的 Ctrl+点击)自己验证这条链:

```bash
# 1. 找入口:main() 在哪、它最后调了谁
grep -n "def main" gptme/cli/main.py
grep -n "chat(" gptme/cli/main.py        # 找到调用 chat() 的那一行

# 2. 看主循环怎么组织
grep -n "def _run_chat_loop\|def step\|def chat" gptme/chat.py

# 3. 看回复如何变成工具执行
grep -n "def execute_msg" gptme/tools/__init__.py
grep -n "iter_from_content\|def execute" gptme/tools/base.py

# 4. 挑一个最简单的工具看完整实现(推荐 save.py,不到百行)
cat gptme/tools/save.py
```

**产出要求**:合上本文,自己画出 ①→⑥ 的调用顺序图,标出「哪一步调 LLM、哪一步执行工具、循环在哪里回头」。画完对照上图检查。

### Python 语法专栏(本链路涉及,建议手敲)

1. **生成器 / `yield`**(chat.py 的 `step()`):
   ```python
   def step(...) -> Generator[Message, None, None]:
       yield msg_response                    # 像 Iterator,但函数执行到 yield 会"暂停"
       yield from execute_msg(msg_response)  # 委托给另一个生成器
   ```
   Java 里最接近的是 `Iterator` + 惰性求值,但 Python 的 generator 是语言级协程:调用方每 `next()` 一次,函数体推进到下一个 `yield`。gptme 靠它实现"边生成边处理"的流式管道。

2. **装饰器**(chat.py:46 `@trace_function(...)`):形似 Java 注解,**本质完全不同**——它是运行时高阶函数,`@f` 等价于 `chat = f(chat)`,直接包装/替换函数,不需要反射或字节码增强。

3. **frozen dataclass**(`@dataclass(frozen=True)` 的 `Message`、`ToolSpec`):≈ Java `record`。`frozen=True` 禁止赋值,所以 `ToolSpec.__init__` 里你会看到诡异的 `object.__setattr__(self, "name", name)`——这是 frozen 类自定义构造器时绕过冻结的标准写法。

4. **海象运算符 `:=`**(chat.py:96):`if x := f():` = 赋值同时判断,Java 里得写两行。

5. **`str | None`**(参数类型):= `Optional<String>`,Python 3.10+ 的联合类型写法。

6. **模块级函数与全局状态**:`message.py` 里 `_output_format` 全局变量 + `set_output_format()`,Python 常用模块本身当单例(模块只被 import 一次),不需要 Singleton 类。

### 过关自查

1. LLM 回复"我来运行这个命令: ```shell\nls\n``` "之后,到 `ls` 真正被执行,中间经过哪几个函数?
2. 为什么 `step()` 要写成生成器而不是返回 `List<Message>`?(想想流式输出和中断)
3. `ToolSpec` 和 `ToolUse` 各自的生命周期:哪个启动时创建一次,哪个每次调用创建?

---

## 第 4 步:定向深入(围绕 issue)

### 找候选 issue

```bash
gh issue list -R gptme/gptme --label "good first issue" --state open
gh issue list -R gptme/gptme --label "bug" --state open --limit 20
```

优先挑:**能本地复现的 bug** > 小工具改进 > 涉及 server/webui 的(先避开,链路长)。

### 每个 issue 的分析套路(固定四问)

1. **定位**:从 issue 描述里抠关键词,`grep -rn "关键词" gptme/` 找到模块;报错信息直接搜异常文本最快。
2. **复现**:写一个最小 `gptme` 命令或 pytest 用例触发它。修 bug 的第一个 commit 应该是"新增一个会失败的测试"。
3. **考古**:这段代码为什么长这样?
   ```bash
   git log --oneline -- gptme/tools/patch.py   # 这个文件的演进史
   git blame gptme/chat.py -L 540,580          # 这几行是谁、哪个 commit 改的
   git log -S "iter_from_content" --oneline    # 哪些 commit 增删过这个符号
   gh pr view <PR号> --comments                # 看当时的讨论
   ```
4. **评估**:改动面(1 个文件还是横跨 tools+llm?)、需不需要补的知识(async?流式解析?)、有没有现成测试可以照抄结构。

### 提 PR 前的规定动作(来自 AGENTS.md,项目硬性要求)

- 分支名:`fix/xxx`、`feat/xxx`;commit 用 Conventional Commits(`fix: ...`)
- 所有函数**必须有类型标注**;`make typecheck`(mypy)必须过
- `make lint`(ruff)、`make test` 必须过;不要 `git add .`,显式加文件
- 风格:KISS,小函数,少 mock 多集成测试

---

## 附:累积清单(每次会话后往这里追加)

### Python 新语法/概念

- [ ] 生成器与 `yield` / `yield from`
- [ ] 装饰器(vs Java 注解)
- [ ] `@dataclass(frozen=True)`(vs record)
- [ ] 海象运算符 `:=`
- [ ] `str | None` 联合类型、mypy
- [ ] entry_points(vs SPI)
- [ ] pytest fixture(vs JUnit 注解)
- [ ] 上下文管理器 `with`(vs try-with-resources)——chat.py 的 `with terminal_state_title(...)`

### gptme 项目知识

- [ ] agent loop 的本质:LLM 回复 → 解析工具块 → 执行 → 结果回喂 → 再问 LLM
- [ ] 默认用 markdown 代码块做工具调用(不是原生 function calling),`tool_format` 可切
- [ ] Message/Log 不可变,改动即新对象
- [ ] ToolSpec(定义,启动时注册)/ ToolUse(调用,每次解析)
- [ ] 一包多命令:20 个 `gptme-*` 入口都注册在 pyproject.toml `[project.scripts]`
