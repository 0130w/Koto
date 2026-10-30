> status: Draft
# 1 Target

Koto v0.1 aims to provide a User-friendly desktop interface for interacting with an agent in a selected local workspace.
Users can open multiple conversation tabs, submit plain-text prompts, follow agent messages and tool execution progress, and continue existing conversations with their context preserved.
The first version uses Codex CLI as its agent execution engine. A Rust backend manages subprocesses, handles execution events, and persists conversation data.
# 2 Scope
## 2.1 Included in v0.1

1. Multiple conversation tabs, each with its own workspace, input draft, conversation history, and execution state.
2. Plain-text prompt input with an ENTER button and keyboard submission.
3. Local workspace selection before the first prompt is submitted.
4. Codex CLI execution in the selected workspace, with JSONL events rendered as agent messages and tool execution updates.
5. Conversation continuation using the `thread_id` returned by Codex.
6. One active task per conversation, with parallel execution across different conversations.
7. A stop button for cancelling the current task.
8. Automatic title generation after the first conversation turn, and manual renaming through the tab context menu.
9. A left sidebar containing conversation history, including conversations whose tabs have been closed.
10. Background execution when a conversation tab is closed, with its running state displayed in the sidebar.
11. SQLite persistence for conversations, drafts, execution records, and open-tab state.
12. Restoration of saved tabs, drafts, and history when the application restarts.
13. Interruption of running tasks when the application exits, with a manual rerun action available afterward.
14. In-conversation error messages when Codex is unavailable or task execution fails.
## 2.2 Not Included in v0.1

1. Images, audio, video or other attachments as prompt input.
2. TTS or other voice interaction.
3. A custom LLM inference or tool execution loop.
4. Changing the workspace of an existing conversation.
5. Automatic retry of unfinished tasks after application restart.
6. Continued background execution after the application exits.
7. Independent control or cancellation of subtasks.
8. Collapsing and expanding long content.
9. Conversation archiving.
# 3 Technical Solution

## 3.1 Architecture

## 3.2 Page and Execution tags

## 3.3 Workspace

After starting new conversation, user can choose 
## 3.4 Codex calling

When page not contains `thread_id`, create new session by:

```bash
codex exec --json "<prompt>"
```

When page contains `thread_id`, continued session by:

```bash
codex exec resume --json <thread_id> "<prompt>"
```

Rust backend will set specified workspace as the current dir of subprocess, and use argument list to pass prompt, avoiding use shell commands to concat commands.

## 3.5 Events Handling

TODO: type

## 3.6 Content Render

1. Render agent message as Markdown format by default.

## 3.7 SQLite Persistence


# 4 Core Interaction

## 4.1 Create and switch pages

## 4.2 Submit prompt

## 4.3 Continued Conversation

# 5 Acceptance Criteria

# 6 Koto v0.1 left

## 6.1 Unified Task Model

