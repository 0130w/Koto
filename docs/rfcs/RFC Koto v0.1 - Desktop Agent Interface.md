> status: Draft

# 1 Target

Koto v0.1 aims to provide a user-friendly desktop interface for interacting with an agent in a selected local workspace.

Users can open multiple conversation tabs, submit plain-text prompts, follow agent messages and tool execution progress, and continue existing conversations with their context preserved.

The first version uses Codex CLI as its agent execution engine. A Rust backend manages subprocesses, handles execution events, and persists conversation data.

# 2 Scope

## 2.1 Included in v0.1

1. Multiple conversation tabs, each with its own workspace, input draft, conversation history, and execution state.
2. Plain-text prompt input with an `ENTER` button and keyboard submission.
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

1. Images, audio, video, or other attachments as prompt input.
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
- Managing prompt input and displaying saved drafts.
- Rendering agent messages, tool execution updates, and task states.
- Sending user actions to the backend, including submitting prompts, stopping tasks, and opening or closing conversation tabs.

Execution events are handled at the application level, independently of individual tab components. Switching or closing a tab does not interrupt event handling.

### 3.1.2 Backend

The Rust backend is responsible for:

- Managing conversations, workspaces, and execution states.
- Starting and stopping Codex CLI subprocesses.
- Reading stdout and stderr, parsing JSONL events, and forwarding execution updates to the frontend.
- Persisting conversation data in SQLite.
- Enforcing one active task per conversation while allowing different conversations to execute concurrently.

Task lifetimes are independent of tab lifetimes. Closing a tab does not stop its running task. When the application exits, the backend interrupts all running tasks and persists their received output and execution status.

### 3.1.3 Communication

The frontend invokes backend operations through Tauri Commands. The backend streams execution updates to the frontend through Tauri Channels.

The Rust backend communicates with Codex CLI through subprocess standard streams and operating-system process controls.

No separate HTTP or WebSocket server is required for v0.1.

### 3.1.4 Agent Execution

Codex CLI owns the agent execution loop, including model interaction, tool execution, and Codex conversation context.

Koto owns the desktop interaction, subprocess lifecycle, and locally persisted display history. It uses the `thread_id` returned by Codex to continue an existing conversation.

## 3.2 Conversation and Execution Identifiers

Koto distinguishes a locally stored conversation from a Codex thread, an individual execution, and the items produced during that execution.

### 3.2.1 `conversation_id`

Koto generates a unique `conversation_id` when a conversation is created.

This identifier associates the conversation with its workspace, draft, title, history, and execution records. It exists before the first prompt is submitted and remains unchanged throughout the conversation's lifetime.

A tab displays a conversation. Closing or reopening a tab does not change its `conversation_id` or delete its data.

The `conversation_id` is internal and is not displayed to the user.

Koto currently maintains a local `conversation_id` separately from the Codex `thread_id` because their creation and lifecycles are controlled by different systems.

A future native agent engine may unify these identifiers and let Koto decide when the shared conversation identifier is created.

### 3.2.2 `thread_id`

The `thread_id` is an optional identifier returned by Codex through a `thread.started` event.

A new conversation has no `thread_id`. Koto stores and displays it only after receiving it from the backend. No placeholder is displayed before that.

Receiving a `thread_id` does not mean the execution has succeeded. Once received, it is retained even if the execution later fails or is interrupted.

Subsequent prompts use this identifier to resume the Codex thread. If resuming fails, Koto reports the error rather than silently creating a replacement thread.

### 3.2.3 `run_id`

Koto generates a unique `run_id` for each prompt execution before starting the Codex subprocess.

The `codex exec --json` interface does not expose a run identifier in its documented turn lifecycle events. Koto uses its own `run_id` to associate startup failures, execution events, and cancellation requests with the same execution.

Each run belongs to one conversation and records its prompt, execution state, timestamps, output items, and errors.

Rerunning a prompt creates a new `run_id` and preserves the previous execution record. Switching, closing, or reopening a tab does not create a new run.

### 3.2.4 `item_id`

Codex supplies an item identifier for each execution item, such as an agent message or a command execution. Koto refers to the `id` field inside the event's item object as `item_id`.

Koto identifies an item by the pair `(run_id, item_id)`. It does not assume item IDs are unique across different runs.

Events referencing the same pair update the existing item rather than append a duplicate. An item may first appear in an `item.completed` event, without a preceding `item.started` event.

### 3.2.5 Event Routing

Each execution update forwarded to the frontend includes `conversation_id` and `run_id`.

The frontend first locates the conversation and run, then updates the corresponding item when the event contains an `item_id`.

## 3.3 Workspace

Users may select an existing local directory through a native folder picker or start a conversation without selecting one.

If no directory is selected, Koto creates a dedicated workspace under its persistent application data directory, using conversation_id as the directory name. The workspace is created before the first Codex subprocess starts, and its path is persisted with the conversation.

The selected workspace is displayed in the conversation view and persisted with the conversation. Users may change it before the first prompt is submitted. After the first submission, the workspace is fixed for that conversation in v0.1.

The Rust backend uses this directory as the working directory of every Codex subprocess started for the conversation, including subsequent prompts and manual reruns.

Before starting a subprocess, the backend verifies that the workspace exists, is a directory, and can be accessed as the process working directory with the current user's filesystem permissions. If validation fails, Koto displays an error in the conversation. The task is not started, and the workspace remains unchanged.

Permission errors encountered during execution are reported separately.

Closing a tab or restarting the application does not change the conversation's workspace.

Multiple conversations may use the same workspace. Koto does not create separate copies or Git worktrees for them in v0.1.
## 3.4 Codex calling

Koto starts a Codex CLI subprocess for each submitted prompt.
The subprocess runs in the conversation's persisted workspace.
### 3.4.1 New Conversations

If the conversation has no `thread_id`, Koto starts a new Codex thread:

```bash
codex exec --json "<prompt>"
```

When a `thread.started` event is received, Koto persists the returned `thread_id` with the conversation.
### 3.4.2 Continuing Conversations

If the conversation already has a `thread_id`, Koto resumes that thread:

```bash
codex exec resume --json <thread_id> "<prompt>"
```

The saved `thread_id` determines whether to create or resume a thread, regardless of whether the previous execution succeed, failed or was interrupted.

If resuming fails, Koto reports the error and retains the existing `thread_id`. It does not automatically create a replacement thread.
### 3.4.3 Subprocess Configuration

The commands above illustrate the invocation structure. The Rust backend passes arguments directly to the executable rather than constructing a shell command.

For each execution, the backend:

- Creates and persists a `run_id` before starting the subprocess.
- Sets the subprocess working directory to the conversation's workspace.
- Passes the prompt as a single argument, preserving its text and line breaks.
- Closes unused stdin so the subprocess does not wait for input.
- Captures stdout and stderr separately.
- Enables JSONL output on stdout.
- Configures full access without interactive approval prompts.
- Allows execution in workspaces that are not Git repositories.

The exact CLI options and minimum supported Codex version will be verified during implementation.
### 3.4.4 Startup Failures

If Codex cannot be found or started, Koto marks the run as failed and displays a Koto-generated error in the conversation.

The submitted prompt and failure record are retained. A startup failure does not create a `thread_id`.
### 3.4.5 Manual Reruns

A manual rerun submits the original prompt again with a new `run_id`, using the same workspace and the conversation's current `thread_id` if available.

The previous execution record and output are preserved. Rerunning does not restore the previous process or undo its side effects.
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

