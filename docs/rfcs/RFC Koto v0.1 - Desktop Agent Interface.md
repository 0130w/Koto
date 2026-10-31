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

Koto uses Tauri 2 as its desktop framework, React and TypeScript for the frontend, and Rust for the backend. The backend uses Tokio for asynchronous execution and SQLite for local persistence.
### 3.1.1 Frontend

The frontend runs inside the system WebView and is responsible for:

- Displaying conversation tabs and the history sidebar.
- Managing prompt input and display saved drafts.
- Rendering agent messages, tool execution updates, and task states.
- Sending user actions to the backend, including submitting prompts, stopping tasks, and opening or closing conversations.

Execution events are handled at the application level, independently of individual tab components. Switching or closing a tab does not interrupt event handling.
### 3.1.2 Backend

The Rust backend is responsible for:

- Managing conversations, workspaces, and execution states.
- Starting and stopping Codex CLI subprocesses.
- Reading stdout and stderr, parsing JSONL events, and forwarding execution updates to the frontend.
- Persisting conversation data in SQLite.
- Enforcing one active task per conversation while allowing different conversations to execute concurrently.

Task lifetimes are independent of tab lifetimes. Closing a tab does not stop its running task. Exiting the application interrupt all running tasks and saves their available state.
### 3.1.3 Communication

The frontend invokes backend operations through Tauri Commands. The backend streams execution updates to the frontend through Tauri Channels.

The Rust backend communicates with Codex CLI through subprocess standard streams and operating-system process controls.

No separate HTTP or WebSocket server is required for v0.1.
### 3.1.4 Agent Execution

Codex CLI owns the agent execution loop, including model interaction, tool execution, and Codex conversation context.

Koto owns the desktop interaction, subprocess lifecycle, and locally persisted display history. It uses the `thread_id` returned by Codex to continue an existing conversation.
## 3.2 Conversation and Execution Identifiers

Koto distinguishes a locally stored conversation from a Codex thread, an individual execution, and the items produced during that execution.
### 3.2.1 conversation_id

> TODO: for example, when I click `new conversation` button, I hope to use the same new conversation window instead of open multiple new windows. Here, the reason of why we need `conversation_id` and `thread_id` at the same time is that out agent harness is wrapping another agent harness instead of LLM Engine itself which causes we can not manage the lifecycle of a prompt directly. In fact, we can remove `conversation_id` after we use our own LLM Engine implementation.

Koto generates a unique conversation_id when a conversation is created.

This identifier associates the conversation with its workspace, draft, title, history, and execution records. It exists before the first prompt is submitted and remains unchanged throughout the conversation's lifetime.

A tab displays a conversation. Closing or reopening the tabs does not change its conversation_id or delete its data.

The conversation_id is internal and is not displayed to the user.
### 3.2.2 thread_id

The thread_id is an optional identifier returned by Codex through a `thread.start` event.

A new conversation has no thread_id. Koto stores and displays it only after receiving it from the backend. No placeholder is displayed before that.

Receiving a thread_id does not mean the execution has succeeded. Once received, it is retained even if the execution later fails or is interrupted.

Subsequent prompts use this identifier to resume the Codex thread. If resuming fails, Koto reports the error rather than silently creating a replacement thread.
### 3.2.3 run_id

> TODO: Codex does not have `run_id`?

Koto generates a unique run_id for each submitted prompt execution.

Each run belongs to one conversation and records its prompt, execution state, timestamps, output items, and errors.

Rerunning a prompt creates a new run_id and preserves the previous execution record. Switching, closing, or reopening a tab does not create a new run.
### 3.2.4 item_id

Codex supplies an item identifier for each execution item, such as an agent message or a command execution. Koto refers to the id field inside the event's item object as item_id.

Koto identifies an item by the pair (run_id, item_id). It does
### 3.2.5 Event Routing

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

