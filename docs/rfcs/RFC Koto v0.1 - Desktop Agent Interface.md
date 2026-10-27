> status: Draft
# 1 Target

Koto v0.1 aims to provide a User-friendly desktop interface, let user talk to agent in specified local workspace.
User can create multiple chat tabs, submit plain text prompt, observe the messages agent output and the tools calling processes and continue talking to agent in preserved conversation.
The first version of Koto use `Codex CLI` as Agent execution engine, use Rust backend to spawn subprocesses, process executing events and persist data.
# 2 Scope
## 2.1 Koto v0.1 includes

1. Multiple chat tabs, each tab maintains its own conversation contents, draft and executing status.
2. Plain text prompt input box and submit button.
3. Call Codex CLI in specified workspace.
4. Accept and render Codex JsonL events, display event messages, executing processes and results of tool calling.
5. Support continued codex conversation after acquiring `thread_id`.
6. Long content fold and unfold.
7. Use SQLite to persist pages, draft, history and executing status.
8. Support recover pages and history after reloading applications.
9. Keep pages content after click closing chat tab, and support open it again.
## 2.2 Koto v0.1 not includes

1. Pictures, voices, videos or other attached files as input.
2. Use TTS or other voice to interact.
3. Implement LLM inference or tools execution loops.
4. Retry unfinished tasks automatically after reloading application.
# 3 Technical Solution

## 3.1 Architecture

TODO:
## 3.2 Page and Execution tags


## 3.3 Codex calling

When page not contains `thread_id`, create new session by:

```bash
codex exec --json "<prompt>"
```

When page contains `thread_id`, continued session by:

```bash
codex exec resume --json <thread_id> "<prompt>"
```

Rust backend will set specified workspace as the current dir of subprocess, and use argument list to pass prompt, avoiding use shell commands to concat commands.

## 3.4 Events Handling

TODO: type

## 3.5 Content Render

1. Render agent message as Markdown format by default.

## 3.6 SQLite Persistence


# 4 Core Interaction

## 4.1 Create and switch pages

## 4.2 Submit prompt

## 4.3 Continued Conversation

# 5 Acceptance Criteria

# 6 Koto v0.1 left

## 6.1 Unified Task Model

