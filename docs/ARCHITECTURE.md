# GenericAgent Architecture Analysis

> **What it is**: A ~3K-line, minimal, self-evolving autonomous AI agent framework that grants any LLM system-level control over a local computer — covering browser, terminal, filesystem, keyboard/mouse, screen vision, and mobile devices (ADB).

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frontends (UI Layer)                        │
│  Streamlit │ Textual TUI │ pywebview Desktop │ IM Bots │ Conductor │
└────────────────────────────┬────────────────────────────────────────┘
                             │ put_task(query)
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     agentmain.py — GenericAgent                     │
│  (Orchestrator: task queue, LLM session management, history)        │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────────┐ │
│  │  LLM Core    │   │  Agent Loop  │   │   Handler (ga.py)       │ │
│  │ (llmcore.py) │◄──│(agent_loop.py)│◄──│ (GenericAgentHandler)   │ │
│  │              │   │              │   │                         │ │
│  │ Multi-LLM    │   │ ~100 lines   │   │ 9 Atomic Tools +       │ │
│  │ Session Mgmt │   │ Perceive →   │   │ Memory Management      │ │
│  │ Claude/OAI/  │   │ Reason →     │   │                         │ │
│  │ Native/OAI   │   │ Execute →    │   │ code_run │ file_read   │ │
│  │              │   │ Write Memory  │   │ file_write│ file_patch  │ │
│  │ ToolClient / │   │ → Loop       │   │ web_scan  │ web_exec_js │ │
│  │ Native*      │   │              │   │ ask_user  │ no_tool     │ │
│  └──────────────┘   └──────────────┘   │ update_working_checkpoint│ │
│                                         │ start_long_term_update   │ │
│                                         └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Browser Layer  │ │  Memory Layer   │ │  Reflect Layer  │
│  (TMWebDriver + │ │  (L0-L4 + SOPs) │ │  (Background    │
│   simphtml.py)  │ │                 │ │   Workers)      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 2. Core Module Map

| File | Lines | Role | Key Exports |
|------|-------|------|-------------|
| `agentmain.py` | ~280 | **Main orchestrator** — `GenericAgent` class, task queue, CLI entry points | `GenericAgent`, `get_system_prompt()` |
| `agent_loop.py` | ~125 | **Agent loop engine** — ~100-line perceive→reason→execute→memory cycle | `agent_runner_loop()`, `BaseHandler`, `StepOutcome` |
| `llmcore.py` | ~1000 | **LLM abstraction** — multi-provider session management, streaming, tool call parsing | `LLMSession`, `ClaudeSession`, `NativeClaudeSession`, `NativeOAISession`, `MixinSession`, `ToolClient`, `NativeToolClient` |
| `ga.py` | ~590 | **Tool handler** — 9 atomic tools + memory management tool implementations | `GenericAgentHandler`, `code_run()`, `web_scan()`, `web_execute_js()`, `file_read()`, `file_write()`, `file_patch()` |
| `simphtml.py` | ~870 | **HTML simplification engine** — DOM optimization, change detection, smart truncation | `get_html()`, `execute_js_rich()`, `smart_truncate()` |
| `TMWebDriver.py` | ~284 | **Browser driver** — WebSocket + HTTP bridge to real browser via CDP | `TMWebDriver`, `Session` |

---

## 3. Architectural Layers in Detail

### 3.1 LLM Abstraction Layer (`llmcore.py`)

Provides a unified interface across multiple LLM providers:

```
BaseSession (abstract)
├── ClaudeSession       — Anthropic Messages API
├── LLMSession          — OpenAI-compatible Chat Completions API
├── NativeClaudeSession — Claude Code native protocol (beta headers, metadata)
└── NativeOAISession   — OpenAI Responses API via native Claude session wrapper

ToolClient          — Wraps any BaseSession; injects tool-use protocol via text prompting
NativeToolClient    — Wraps NativeClaudeSession; uses native tool_use content blocks

MixinSession        — Multi-model fallback with spring-back to primary
```

**Key design decisions:**
- **Dual tool-calling mode**: `ToolClient` embeds tool schemas in the prompt text (for models without native tool support); `NativeToolClient` uses the Claude-native `tool_use` content block format.
- **Streaming SSE parsing**: Separate parsers for Anthropic SSE (`_parse_claude_sse`) and OpenAI SSE (`_parse_openai_sse`), plus a Responses API variant.
- **Automatic retry**: `_stream_with_retry` handles transient HTTP errors with exponential backoff.
- **Context window management**: `trim_messages_history()` auto-compresses older messages to stay within the configured context window budget.

### 3.2 Agent Loop Engine (`agent_loop.py`)

The core execution cycle — deliberately minimal (~100 lines of logic):

```python
def agent_runner_loop(client, system_prompt, user_input, handler, tools_schema, max_turns=40):
    while turn < max_turns:
        1. Send messages + tools to LLM client
        2. Parse response → extract tool_calls (or fallback to no_tool)
        3. For each tool_call: handler.dispatch(tool_name, args)
        4. Collect StepOutcome (data + next_prompt + should_exit)
        5. handler.turn_end_callback() — inject safety prompts, history, memory
        6. Append next_prompt as new user message → loop
```

**Key concepts:**
- `StepOutcome`: Dataclass with `data`, `next_prompt`, `should_exit` — every tool returns one.
- `no_tool`: Special pseudo-tool triggered when the LLM responds without calling any tool — handles empty responses, code-block-only responses, and plan-mode interception.
- The loop is a **generator** — yields streaming chunks to frontends while running.

### 3.3 Tool Handler (`ga.py` — `GenericAgentHandler`)

Implements all 9+ tools via the `do_<tool_name>` dispatch pattern:

| Tool Method | Atomic Tool | Purpose |
|-------------|-------------|---------|
| `do_code_run` | `code_run` | Execute Python/bash/PowerShell scripts with timeout and streaming output |
| `do_file_read` | `file_read` | Read file contents with line numbers, keyword search, fuzzy path matching |
| `do_file_write` | `file_write` | Write/append/prepend file content (extracts from `<file_content>` tags) |
| `do_file_patch` | `file_patch` | Surgical find-and-replace within files (uniqueness-gated) |
| `do_web_scan` | `web_scan` | Capture simplified HTML of current browser tab + tab list |
| `do_web_execute_js` | `web_execute_js` | Execute JavaScript in browser, capture DOM changes and transient elements |
| `do_ask_user` | `ask_user` | Interrupt loop for human confirmation/input |
| `do_update_working_checkpoint` | `update_working_checkpoint` | Update short-term working memory (key_info + related SOP) |
| `do_start_long_term_update` | `start_long_term_update` | Trigger long-term memory crystallization (L2/L3 update) |
| `do_no_tool` | `no_tool` | Handle LLM responses with no tool calls |

**Safety mechanisms embedded in the handler:**
- Turn-count warnings at multiples of 7/10/65 to prevent infinite loops
- Plan-mode verification gates (blocks premature completion claims)
- Working memory injection via `_get_anchor_prompt()` to maintain context across turns
- Master intervention injection (reading `_keyinfo` and `_intervene` files)

### 3.4 Browser Layer (`TMWebDriver.py` + `simphtml.py`)

A unique architecture that **injects into a real browser** (not headless/sandbox):

```
TMWebDriver (Python server)
├── WebSocket server (port 18765) — communicates with browser extension
├── HTTP API server (port 18766) — fallback for remote connections
├── Session management — multiple browser tabs as independent sessions
└── JS execution — send scripts, receive structured results

simphtml.py (Token-optimized HTML processing)
├── js_optHTML — In-browser DOM simplification engine
│   ├── Removes invisible/floating elements
│   ├── Preserves form values, checked states, selected options
│   ├── Handles iframes, shadow DOM
│   └── Smart element classification (main/secondary/overlay/covered)
├── js_findMainList — Detects repeating list structures for truncation
├── optimize_html_for_tokens — BeautifulSoup-based attribute stripping
├── smart_truncate — Recursive DOM truncation preserving document structure
├── find_changed_elements — Before/after HTML diff for change detection
└── execute_js_rich — Rich JS execution with DOM change monitoring
```

**Design philosophy**: Real browser injection preserves login sessions, cookies, and full rendering — critical for tasks like food delivery, banking, social media automation.

### 3.5 Memory System (5-Layer Hierarchy)

The memory system lives in `memory/` and is the foundation of self-evolution:

```
L0 — Meta Rules         memory/memory_management_sop.md
                          Rules for how memory should be updated
                          
L1 — Insight Index      memory/global_mem_insight.txt
                          Minimal routing index for fast recall
                          
L2 — Global Facts       memory/global_mem.txt
                          Stable, verified knowledge accumulated over time
                          
L3 — Task Skills/SOPs   memory/*_sop.md (20+ SOPs)
                          Reusable procedures for specific task types:
                          ├── autonomous_operation_sop.md
                          ├── code_review_principles.md
                          ├── web_setup_sop.md
                          ├── plan_sop.md
                          ├── vision_sop.md
                          ├── supervisor_sop.md
                          ├── verify_sop.md
                          └── ... (20+ more)
                          
L4 — Session Archive    memory/L4_raw_sessions/
                          Compressed task records from finished sessions
```

**How memory flows during execution:**
1. System prompt loads L0 rules + L1 insight index + L2 global facts
2. `update_working_checkpoint` sets short-term key_info for the current task
3. `start_long_term_update` triggers the agent to crystallize verified experience into L2/L3
4. SOPs are recalled via `file_read` when the agent encounters similar tasks
5. Scheduler (`reflect/scheduler.py`) periodically archives raw sessions to L4

### 3.6 Frontend Layer (`frontends/`)

Multiple independent UI adapters, all connecting to `GenericAgent` via `put_task()`:

| Frontend | File | Technology | Purpose |
|----------|------|------------|---------|
| Streamlit UI | `stapp.py` / `stapp2.py` | Streamlit | Web-based chat interface |
| Desktop App | `launch.pyw` | pywebview + Streamlit | Native window wrapping Streamlit |
| TUI v2 | `tuiapp_v2.py` | Textual | Terminal-based rich UI |
| Qt Desktop | `qtapp.py` | PyQt6 | Linux-native desktop app |
| Telegram | `tgapp.py` | python-telegram-bot | IM bot interface |
| WeChat | `wechatapp.py` | Custom | Personal WeChat integration |
| QQ | `qqapp.py` | qq-botpy | QQ bot |
| Feishu | `fsapp.py` | lark-oapi | Feishu/Lark bot |
| WeCom | `wecomapp.py` | wecom-aibot-sdk | Enterprise WeChat |
| DingTalk | `dingtalkapp.py` | dingtalk-stream | DingTalk bot |
| Conductor | `conductor.py` | FastAPI + WebSocket | Sub-agent orchestration dashboard |
| Desktop Pet | `desktop_pet.pyw` | tkinter | Floating desktop companion |
| Hub/Launcher | `hub.pyw` | tkinter | Service manager for launching multiple frontends |

**Common infrastructure**: `chatapp_common.py` provides shared utilities for all chat-based frontends.

### 3.7 Reflect Layer (Background Workers)

Background processes that monitor and trigger tasks autonomously:

| Worker | File | Purpose |
|--------|------|---------|
| Scheduler | `scheduler.py` | Cron-based task scheduling (daily, weekly, monthly, custom intervals) |
| Goal Mode | `goal_mode.py` | Continuous goal-oriented autonomous operation |
| Autonomous | `autonomous.py` | Self-directed exploration and task execution |
| Agent Team | `agent_team_worker.py` | Multi-agent team coordination |

These run via `agentmain.py --reflect <script>`, which loads the script and calls its `check()` function periodically. When `check()` returns a task string, it's submitted to the agent.

---

## 4. Data Flow

### 4.1 Request Lifecycle

```
User Input (any frontend)
    │
    ▼
GenericAgent.put_task(query) → display_queue
    │
    ▼
GenericAgent.run() [background thread]
    │
    ├── get_system_prompt() → loads L0 rules + L1 index + L2 facts + date
    │
    ├── Creates GenericAgentHandler (inherits key_info from previous handler)
    │
    └── agent_runner_loop(llmclient, sys_prompt, query, handler, tools_schema)
        │
        ├── [Turn N] client.chat(messages, tools) → LLM API call
        │   │
        │   ├── ToolClient: builds protocol prompt with tool schemas in text
        │   └── NativeToolClient: sends native tool_use blocks
        │
        ├── Parse response → extract tool_calls
        │
        ├── For each tool_call: handler.dispatch(name, args)
        │   │
        │   └── do_<tool_name>() → execute tool → return StepOutcome
        │       ├── data: tool result (JSON-serializable)
        │       ├── next_prompt: continuation prompt (with working memory)
        │       └── should_exit: True for ask_user / task completion
        │
        ├── handler.turn_end_callback()
        │   ├── Compress history info
        │   ├── Inject safety warnings (turn count limits)
        │   ├── Inject plan-mode hints
        │   ├── Check for master interventions
        │   └── Return modified next_prompt
        │
        └── Append next_prompt as new user message → next turn
```

### 4.2 LLM Session Management

```
mykey.py (user config)
    │
    ▼
reload_mykeys() → parses API configs
    │
    ├── Creates LLMSession/ClaudeSession/NativeClaudeSession per config
    ├── Wraps in ToolClient or NativeToolClient
    ├── MixinSession: composites multiple sessions with fallback
    │
    └── GenericAgent.llmclients[] — round-robin via next_llm()
```

### 4.3 Memory Update Flow

```
Task completes successfully
    │
    ▼
Agent calls start_long_term_update tool
    │
    ▼
Handler provides L0 SOP + current global memory as context
    │
    ▼
Agent reads existing memory files (file_read)
    │
    ▼
Agent identifies verified facts / useful SOPs
    │
    ▼
Agent writes to:
    ├── L2 (global_mem.txt) via file_patch — stable facts
    ├── L3 (*_sop.md) via file_write/file_patch — task procedures
    └── L1 (global_mem_insight.txt) via file_patch — update index
```

---

## 5. Key Design Patterns

### 5.1 Generator-Based Streaming

The entire execution pipeline is generator-based, enabling real-time streaming to any frontend:

```python
# agent_runner_loop is a generator
gen = agent_runner_loop(client, system_prompt, query, handler, tools)
for chunk in gen:
    # Stream to frontend
    display_queue.put({'next': chunk})
```

### 5.2 Handler Dispatch Pattern

Tools are dispatched via naming convention:

```python
class BaseHandler:
    def dispatch(self, tool_name, args, response):
        method = getattr(self, f"do_{tool_name}")
        return yield from method(args, response)
```

This enables `GenericAgentHandler` to inherit `BaseHandler` and register tools simply by defining `do_*` methods.

### 5.3 File-Based IPC (for background tasks)

The `--task` mode uses file-based inter-process communication:

```
temp/<task_id>/
├── input.txt        → Task prompt
├── output.txt       → Agent response
├── output1.txt      → Multi-round responses
├── reply.txt        → User follow-up (written externally)
├── _stop            → Abort signal
├── _keyinfo         → Inject key info into running agent
├── _intervene       → Inject arbitrary prompt into running agent
└── _history.json    → Session history for recovery
```

### 5.4 Token Budget Management

Multiple strategies to stay within ~30K context window:

1. **HTML simplification** (`simphtml.py`): DOM → simplified HTML → token-optimized
2. **List truncation**: Detect repeating structures, keep 3 samples + `[FAKE ELEMENT] N more items hidden`
3. **History compression** (`compress_history_tags`): Truncate older `<thinking>`, `<tool_use>`, `<tool_result>` blocks
4. **Tool schema caching**: Tool descriptions sent once, then abbreviated as "Tools still active"
5. **Smart truncation** (`smart_truncate`): Recursive DOM budget allocation preserving document structure

---

## 6. Configuration & Extensibility

### 6.1 LLM Configuration (`mykey.py`)

Each LLM session is configured as a dictionary:

```python
{
    'name': 'claude-sonnet',
    'apikey': 'sk-...',
    'apibase': 'https://api.anthropic.com',
    'model': 'claude-sonnet-4-20250514',
    'context_win': 30000,
    'stream': True,
    'temperature': 1,
    # ... optional: proxy, max_retries, timeout, reasoning_effort, thinking_type
}
```

Mixin configurations enable multi-model fallback with automatic spring-back.

### 6.2 Adding New Tools

1. Add tool definition to `assets/tools_schema.json`
2. Add `do_<tool_name>` method to `GenericAgentHandler` in `ga.py`
3. Method signature: `def do_<name>(self, args, response) -> StepOutcome`

### 6.3 Adding New Frontends

1. Create a new file in `frontends/`
2. Import `GenericAgent` from `agentmain`
3. Call `agent.put_task(query)` to submit tasks
4. Read from the returned `display_queue` for streaming responses

### 6.4 Adding New SOPs (Skills)

1. Create `memory/<skill_name>_sop.md` with the procedure
2. The agent discovers and reads it via `file_read` when encountering similar tasks
3. No registration or code changes required

---

## 7. Dependency Graph

```
agentmain.py
├── llmcore.py (LLM sessions, streaming parsers)
│   └── mykey.py (API keys)
├── agent_loop.py (execution loop, BaseHandler)
└── ga.py (GenericAgentHandler, tool implementations)
    ├── simphtml.py (HTML processing)
    │   └── beautifulsoup4
    └── TMWebDriver.py (browser driver)
        ├── simple-websocket-server
        └── bottle

frontends/ (UI adapters — all import from agentmain.py)
├── stapp.py (Streamlit)
├── tuiapp_v2.py (Textual)
├── tgapp.py (Telegram)
├── conductor.py (FastAPI sub-agent orchestrator)
└── ... (other IM frontends)

reflect/ (background workers — run via agentmain.py --reflect)
├── scheduler.py (cron tasks)
├── goal_mode.py (autonomous goals)
└── ...

launch.pyw (pywebview desktop wrapper around Streamlit)
hub.pyw (tkinter service manager)
```

---

## 8. Runtime Architecture

### 8.1 Single-Agent Mode (Default)

```
Frontend → GenericAgent (1 thread) → LLM API
                                  → Tool execution (local)
                                  → Browser (via TMWebDriver)
                                  → Memory (file reads/writes)
```

### 8.2 Conductor Mode (Sub-Agent Orchestration)

```
Conductor (FastAPI, port 8900)
├── Main GenericAgent (conductor agent)
├── SubAgentState[] (parallel sub-agents)
│   ├── Each sub-agent: independent GenericAgent instance
│   ├── Separate thread + LLM session
│   └── File-based monitoring (_stop, _intervene)
└── WebSocket → Browser dashboard (conductor.html)
```

### 8.3 Reflect Mode (Background Workers)

```
agentmain.py --reflect scheduler.py
├── GenericAgent (1 thread, long-running)
├── scheduler.check() called every INTERVAL seconds
├── When triggered: task → put_task() → agent processes
└── Results logged to sche_tasks/done/
```

---

## 9. Unique Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **~3K lines total** | Minimality as a feature — the agent grows capabilities through self-evolution, not pre-built modules |
| **9 atomic tools only** | Composability over specialization — `code_run` can dynamically install packages, write scripts, create new tools at runtime |
| **Real browser injection** | Preserves login sessions, cookies, full rendering — essential for real-world web automation |
| **File-based memory** | No database — markdown files are human-readable, LLM-editable, and version-controllable |
| **Generator-based streaming** | Single execution model serves all frontends (CLI, web, IM, desktop) without adapters |
| **Token budget as architecture** | Every design choice (HTML simplification, history compression, tool schema caching) optimizes for staying within ~30K context |
| **SOP-driven self-evolution** | Task experience crystallizes into SOPs that are recalled on similar future tasks — the agent gets better with use |
| **No external framework dependency** | No LangChain, no CrewAI, no AutoGen — every component is purpose-built and minimal |
