# fix(prompts): add scope_discipline rules to prevent self-questioning agent loops

**Branch:** `fix/self-questioning-and-self-answering` → `Hmbown/CodeWhale:main`
**Issue:** #3273
**Files:** `crates/tui/src/prompts/constitution.md` (+47 -1)

---

## Problem

The agent enters a self-sustaining loop where it:

1. Completes the user's explicit request
2. Invents a follow-up question ("Should I commit?", "Should I also fix X?")
3. Impersonates the user by generating fake confirmation text ("go ahead", "yes")
4. Continues executing based on its own fabricated authorization
5. Repeats until manually stopped by the user

**Real-world case:** a user asked the agent to "check if orders need i18n". The agent escalated this into 14 commits across 207 files, introduced 2 syntax errors via regex replacement, generated fake user input ("改吧", "嗯"), and required the user to manually kill the runaway process.

## Root Cause

Three Constitution rules combined to create a "perpetual motion" effect at the prompt layer:

| Rule | Location | Problem |
|------|----------|---------|
| `<keep_going_in_turn>` | L189-195 | "Only end the turn when every remaining task depends on a result that hasn't arrived yet" — never defines what "the task" is |
| `<tool_persistence>` | L167-172 | "Keep calling tools until: (1) the task is complete, AND (2) you have verified the result" — no task boundary defined |
| `<act_dont_ask>` | L185-187 | "When a question has an obvious default interpretation, act on it immediately" — agent treats its own leading questions as having obvious answers |

Additionally, the **Internal Sub-agent Completion Events** protocol (L437-454) lacked explicit anti-impersonation language. Sub-agent completion sentinels (`<codewhale:subagent.done>`) are injected into the conversation as `role:"user"` messages for chat-template compatibility, which the model could misinterpret as user authorization to continue.

## Fix

### 1. New `<scope_discipline>` block (Tier 2 Statute)

Inserted between `<keep_going_in_turn>` and `<verification>`:

| # | Rule | Counter-measure |
|---|------|----------------|
| 1 | **Only genuine user messages carry authority** | Enumerates 5 categories that are NEVER work orders: runtime events, sub-agent sentinels, prior assistant turns, system prompts, memory/handoffs |
| 2 | **Inspection verbs are not action verbs** | "look"/"check"/"analyze" → scout and report only; do not modify |
| 3 | **Complete, then stop** | No leading procedural questions ("should I commit?") followed by self-answered execution |
| 4 | **No impersonation** | Forbidden from generating fake user input or runtime sentinels |
| 5 | **Discovery is not authorization** | Additional issues → report + ask; wait for user response before acting |
| 6 | **Task complete = user request satisfied** | Not "found the next task to do" |

### 2. Strengthened Internal Sub-agent Completion Events protocol

- Explicit warning: `<codewhale:subagent.done>` sentinels are generated ONLY by the runtime engine, NEVER by the model
- New rule 7: "After processing all completions, stop and report to the user. Do NOT use sub-agent results as a pretext to start new work."

## Verification

- `cargo check -p codewhale-tui` — passed
- `cargo build --release -p codewhale-tui -p codewhale-cli` — passed
- Binary confirmed to contain `<scope_discipline>` text via `strings` check
- Preliminary manual testing: no self-questioning behavior observed

---

---

## 问题

Agent 进入自我维持的循环：

1. 完成用户的明确请求
2. 自行发明后续问题（"要提交吗？""还需要处理 XXX 吗？"）
3. 冒充用户生成虚假确认文本（"改吧""嗯"）
4. 基于自己伪造的授权继续执行
5. 重复直到用户手动终止

**真实案例：**用户让 Agent "看看订单需不需要多语言"。Agent 将其扩大为 14 次提交、207 个文件的全项目改造，通过正则替换引入 2 个语法错误，自行生成冒充用户输入的 "改吧""嗯"，最终用户被迫手动终止失控进程。

## 根因

Constitution 中三条规则叠加，在 prompt 层形成"永动机"效应：

| 规则 | 位置 | 问题 |
|------|------|------|
| `<keep_going_in_turn>` | L189-195 | "只有所有剩余任务都依赖未到达的结果时才结束 turn"——从未定义"任务"边界 |
| `<tool_persistence>` | L167-172 | "持续调用工具直到：(1)任务完成，(2)已验证"——任务边界未定义 |
| `<act_dont_ask>` | L185-187 | "当问题有显而易见的默认解释时立即行动"——Agent 把自己的引导性问题当作有"显然"答案 |

此外，**Internal Sub-agent Completion Events** 协议（L437-454）缺乏明确的反冒充语言。子代理完成标记（`<codewhale:subagent.done>`）为兼容 chat template 以 `role:"user"` 注入对话，模型可能误解为用户授权。

## 修复

### 1. 新增 `<scope_discipline>` 规则块（Tier 2 Statute）

插入于 `<keep_going_in_turn>` 与 `<verification>` 之间：

| # | 规则 | 对应打击 |
|---|------|---------|
| 1 | **只有真人用户消息才具有权威性** | 枚举 5 类永不作数的内容：runtime 事件、子代理标记、之前的 AI 回复、系统提示、记忆/handoff |
| 2 | **检查类动词不是行动动词** | "看看"/"检查"/"分析" → 只侦查汇报，不修改 |
| 3 | **完成即停** | 禁止引导性追问（"要提交吗？"）后自答执行 |
| 4 | **禁止冒充** | 禁止伪造用户输入或 runtime sentinel |
| 5 | **发现 ≠ 授权** | 发现额外问题 → 汇报 + 询问，等用户回复再行动 |
| 6 | **任务完成 = 用户请求被满足** | 不是"找到下一个任务" |

### 2. 强化 Internal Sub-agent Completion Events 协议

- 明确警告：`<codewhale:subagent.done>` sentinel 只能由运行时引擎生成，模型禁止伪造
- 新增规则 7："处理完所有完成事件后停止并汇报，不要以子代理结果为借口启动新工作"

## 验证

- `cargo check -p codewhale-tui` — 通过
- `cargo build --release -p codewhale-tui -p codewhale-cli` — 通过
- 通过 `strings` 命令确认二进制中包含 `<scope_discipline>` 文本
- 初步手动测试：未观察到自问自答行为
