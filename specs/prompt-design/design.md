## Prompt Design Investigation – Local Chat Prototype (Strands Agent)

### 1. Overview and Goals

This design specifies a **local, CLI-based chat prototype** for a retirement-planning application. The goal of the prototype is to:

- **Explore prompt designs** (primarily system prompts) that:
  - Guide users through key aspects of a **retirement financial planning journey**.
  - Elicit and structure **personal financial details** needed for planning.
  - Provide **clear, actionable recommendations** and structured summaries back to the user.
- **Experiment with multiple LLM models** (OpenAI and Gemini) and compare behavior.
- **Exercise Strands Agent tooling** (conversation managers, tools) in a local setting that is a precursor to a cloud-hosted architecture.

This prototype is not production infrastructure; it is a **local spike** but should be designed such that:

- The **agent logic and prompt design** can be lifted into a cloud runtime (e.g., AgentCore Runtime + AgentCore Memory + DynamoDB) later.
- The **CLI UX layer** is clearly separated from the **agent/orchestration layer**, so the CLI can eventually be replaced by a web UI without rewriting the underlying agent.

### 2. Non‑Goals and Out of Scope

- No direct integration with **AgentCore Runtime** or **AgentCore Memory** in this spike (local-only persistence).
- No production-grade authorization/authentication.
- No advanced safety/guardrails around tools (basic Strands SDK defaults are sufficient).
- No concurrency or multi-user access guarantees beyond simple file-based session persistence.

---

### 3. High-Level Architecture

#### 3.1 Components

- **CLI Chat Tool** (`server/tools/chat`)
  - A Python executable script providing an interactive and single-turn chat interface via the terminal.
  - Responsible for:
    - Parsing CLI arguments and environment variables.
    - Resolving configuration (config file + CLI overrides).
    - Managing **sessions** (creation, switching, listing, description, deletion) and session IDs via `RETIRE_CURRENT_SESSION_ID`.
    - Invoking the **Chat Agent API** for each user message (interactive or single-turn).
    - Rendering responses in the terminal (with streaming).

- **Chat Agent Package** (`server/agents/chat/`)
  - Python package that encapsulates all **agent-related logic**:
    - Strands `Agent` configuration and instantiation.
    - Conversation manager configuration (initially `SlidingWindowConversationManager`).
    - Model selection and provider wiring for OpenAI and Gemini models.
    - Tool registration and integration with Strands.
    - A simple, stable API that the CLI can call, such as a function:
      - `run_chat_turn(session_id, config, messages, system_prompt, streaming_callback, session_store, logger, ...)`
    - Session persistence abstraction (read/write session history and metadata) mediated through a **session store** interface.

- **Session Persistence** (`server/tools/.chat/`)
  - Local on-disk storage for:
    - **Config**: `config.yaml` (required).
    - **Session data**: per-session JSON files for **history** and **metadata**.
    - **Session registry**: file(s) that allow listing sessions and tracking descriptions, creation times, and “current” status.
    - **Tool registry**.
    - **Logs**.

#### 3.2 Data Flow (Single Turn)

1. User runs the CLI with flags (e.g., `server/tools/chat --single --model gpt-5.1`).
2. CLI:
   - Determines whether this invocation is for a **new session** or an **existing session**:
     - New session if:
       - `RETIRE_CURRENT_SESSION_ID` is not set or empty, or
       - `--new-session` is provided.
     - Existing session if:
       - `RETIRE_CURRENT_SESSION_ID` is set and no `--new-session` / `--session=<UUID>` override is provided, or
       - `--session=<UUID>` is provided and that session exists.
   - For a **new session**:
     - Loads `server/tools/.chat/config.yaml` as defaults.
     - Validates that all required fields exist; if missing, fails with a clear error.
     - Applies CLI flags as **overrides** for this invocation.
   - For an **existing session**:
     - Loads the session’s baseline configuration from `sessions/<session_id>/metadata.json`.
     - Applies any CLI flags as overrides on top of that baseline and updates metadata if configuration changes.
   - Resolves or creates a **session** (using `RETIRE_CURRENT_SESSION_ID`, `--new-session`, or `--session=<UUID>`).
   - Records effective configuration into the session metadata (if it is a new session or configuration has changed).
   - Prompts the user for input (or reads from stdin for single-turn scenarios).
3. CLI calls into the **Chat Agent API** in `server/agents/chat/`, passing:
   - Session ID.
   - Effective config (model, window size, etc.).
   - Session history (loaded by the session store).
   - System prompt (current effective system prompt).
   - Hooks/callbacks for streaming partial output.
4. Chat Agent:
   - Instantiates or retrieves a Strands `Agent` configured with:
     - Selected model (OpenAI or Gemini) based on config.
     - `SlidingWindowConversationManager` with configured `window_size` and `should_truncate_results`.
     - Loaded tools from the tool registry.
     - System prompt.
   - Executes the agent call with streaming.
   - Returns final output and telemetry (token usage, latency, tool calls, model ID).
5. CLI:
   - Streams the assistant’s reply to the terminal.
   - Updates session JSON files with new messages and metadata (model name, system prompt file, tool usage, latencies, tokens, configuration changes).
   - On normal termination prints the session ID.

#### 3.3 Data Flow (Interactive Session)

Similar to single-turn, but:

- CLI enters an interactive loop:
  - Reads user input repeatedly until EOF (Ctrl-D) or explicit exit.
  - For each turn, calls the Chat Agent and streams output.
  - Session history and metadata are updated incrementally after each turn.
- The same session ID is used for all turns in the interactive run.

---

### 4. Session and Persistence Design

#### 4.1 Session Identity and Environment Variable

- A **session** is a logical chat conversation identified by a **UUIDv4**.
- The current session is tracked via the environment variable `RETIRE_CURRENT_SESSION_ID`.
- Rules:
  - On **first CLI call** when `RETIRE_CURRENT_SESSION_ID` is not set or is empty:
    - A new UUIDv4 is generated.
    - `RETIRE_CURRENT_SESSION_ID` is set to this new value in the environment of the current process.
    - The CLI creates appropriate session files for this new session.
  - `--new-session`:
    - Forces creation of a new session with a new UUIDv4.
    - Updates `RETIRE_CURRENT_SESSION_ID` to this new value.
  - `--session=<UUID>`:
    - Switches to an existing session with the provided UUID (if it exists), or errors if not found.
    - Optionally, `--switch` may be accepted as a semantic alias, but is not required for switching behavior.
  - `--delete-session=<UUID>`:
    - Deletes all data associated with that session (history + metadata).
    - If the deleted session is the current session, the CLI should clear `RETIRE_CURRENT_SESSION_ID` in the current process and require a new session on the next run.

#### 4.2 On-Disk Layout (Initial Version)

Under `server/tools/.chat/`, the prototype will use:

- `config.yaml`
  - Global default configuration file (see section 5).
- `sessions/`
  - `sessions/index.json`:
    - A JSON file containing:
      - An array or mapping of all known session IDs.
      - For each session:
        - `id`: UUIDv4.
        - `created_at`: ISO8601 timestamp.
        - `description`: optional string.
        - `is_current`: boolean flag for current session (used for listing convenience only; `RETIRE_CURRENT_SESSION_ID` remains the source of truth at runtime).
  - `sessions/<session_id>/history.json`
    - JSON file representing the chronological list of messages for this session.
    - Each entry should minimally contain:
      - `role`: `"user"` | `"assistant"` | `"system"`.
      - `content`: text content.
      - `timestamp`: ISO8601.
      - Optional: `tool_calls`, `tool_results` when relevant.
    - System prompt logging behavior:
      - When a system prompt is first set for a session, a `role: "system"` entry is appended capturing the full prompt text in `content`.
      - Thereafter, an additional `role: "system"` entry is appended **only when the effective system prompt changes** (even though the system prompt is sent on every LLM call); this avoids redundant entries while preserving a clear history of prompt changes.
  - `sessions/<session_id>/metadata.json`
    - JSON file storing the **current session configuration and telemetry**, including:
      - **Configuration used** for this session (current effective values):
        - Model identifier (`model` string such as `gpt-5.1` or `gemini-2.5-flash`).
        - Conversation manager type (e.g., `"sliding_window"`).
        - `window_size` and `should_truncate_results`.
        - Any other relevant configuration values drawn from `config.yaml` or CLI overrides.
      - **System prompt metadata (current effective prompt)**:
        - `system_prompt_text`: effective system prompt raw text.
        - `system_prompt_source`: `"file"` or `"inline"`.
        - If from file:
          - `system_prompt_file_path`: original path as provided.
      - **Per-turn telemetry and configuration changes**:
        - A list or mapping keyed by turn index or timestamp that captures:
          - Model name used on that turn (in case it changes mid-session).
          - Tool calls + arguments and results.
          - Latency (overall LLM call duration).
          - Tokens used (prompt + completion).
          - Any configuration overrides applied at that time (including system prompt changes).
        - When a system prompt changes:
          - The change is reflected in both:
            - A `role: "system"` entry in `history.json` (message-level record), and
            - An updated entry in the per-turn telemetry for that turn (metadata-level record).

Design note: The design should clearly separate **session history** and **metadata** in code, and document that a future evolution may move to SQLite for better concurrency/querying.

#### 4.3 Session Operations and CLI Options

- `--list-sessions`
  - Prints a **human-friendly table** of sessions including:
    - UUID.
    - Created timestamp.
    - Description (if present).
    - Whether it is the current session.
  - Assume the number of sessions stays small enough that pagination is not required.

- `--description="<description-text>" --session=<UUID>`
  - Sets or updates the description of the specified session without switching to it.
  - If `--description` is provided without `--session`, the CLI should error and explain usage.

- `--delete-session=<UUID>`
  - Deletes the specified session’s `history.json` and `metadata.json` and removes its entry from `sessions/index.json`.
  - If the deleted session is marked as current, clear its current flag and (for this process) treat the environment as having no active session.

---

### 5. Configuration and Model Selection

#### 5.1 Config File (`config.yaml`)

- Location: `server/tools/.chat/config.yaml`.
- This file is **required** for the CLI to run.
  - If `config.yaml` is missing or any required configuration keys are missing, the tool must:
    - Exit with a non-zero status.
    - Print a clear error message indicating which keys are missing.

- **Required keys** (initial set, can be extended in this design):
  - `model`: string (e.g., `gpt-5.1`, `gpt-5.1-mini`, `gemini-2.5-pro`, `gemini-2.5-flash`).
  - `window_size`: integer (default 20).
  - `should_truncate_results`: boolean (default `true`).
  - Any additional values needed by the Strands configuration (e.g., API keys may be read from environment rather than config).

Design note: The design should specify a reference structure for `config.yaml` and allow the Coding Agent to document the exact schema in code-level comments or a short README.

#### 5.2 CLI Overrides and Precedence

Configuration precedence for the **initial CLI execution for a new session**:

1. Load all required defaults from `config.yaml`.
2. If any required values are missing, error out.
3. Apply CLI flags as **overrides** for the current invocation.
   - E.g., `--model`, `--window-size`, `--should-truncate-results`, `--system-prompt` or `--system-prompt-file`, etc.
4. The effective configuration (after overrides) is persisted into the session’s `metadata.json` for that session.

For **existing sessions**:

- The configuration persisted in `metadata.json` is treated as the baseline for that session for subsequent invocations.
- On each new invocation targeting an existing session:
  - Start from the baseline configuration from `metadata.json`.
  - Apply any CLI flags as overrides on top of that baseline.
  - Persist any changes back to `metadata.json` so that the latest configuration becomes the new baseline for future invocations on that session.
- The design should explicitly document which CLI flags are intended to **update session-level configuration** versus those that are **one-off for that run**. A simple initial rule can be:
  - Flags that correspond to configuration keys (e.g., `--model`, `--window-size`, `--should-truncate-results`, system prompt options) update the session metadata.

#### 5.3 Model Identification

- A single **model string** is used to represent both provider and model ID (e.g., `gpt-5.1`, `gpt-5.1-mini`, `gemini-2.5-pro`, `gemini-2.5-flash`).
- The design should specify a mapping layer in `server/agents/chat/` that:
  - Maps the model string to:
    - Provider (OpenAI vs Gemini).
    - Concrete Strands model configuration (model class and `model_id`) used by the underlying Strands SDK.
  - This mapping should be centralized so new models can be added by editing a single configuration/location.

---

### 6. System Prompt Management

#### 6.1 Sources of System Prompt

- System prompt may be provided by either:
  - `--system-prompt-file <filename>`: a path (relative or absolute) to a text file containing the system prompt.
  - `--system-prompt="<system-prompt>"`: inline system prompt text.
- **Only one mechanism may be used per invocation**:
  - If both `--system-prompt-file` and `--system-prompt` are provided, the CLI should:
    - Exit with an error.
    - Explain that only one of the two may be used at a time.

#### 6.2 Persistence and Session Behavior

- Once a system prompt is provided for a session (either via file or inline), that system prompt becomes the **effective system prompt for that session** until changed by a subsequent CLI invocation with a new system prompt option.
- The design should:
  - Ensure that the effective system prompt text is stored in `metadata.json` as:
    - `system_prompt_text`.
    - `system_prompt_source` (`"file"` or `"inline"`).
    - `system_prompt_file_path` if applicable.
  - Record that changes to system prompt should be tracked in metadata so it is clear which prompt was active at any given phase of the conversation.

#### 6.3 Prompt Design Considerations (Retirement Planning)

While concrete prompt text is out-of-scope for this design, the structure should support:

- System prompts that describe:
  - The assistant’s **role** as a retirement-planning guide.
  - High-level **conversation stages** (e.g., discovery, data collection, planning, recommendations).
  - Expectations around:
    - Asking clarifying questions.
    - Explaining financial concepts clearly.
    - Avoiding regulated advice where inappropriate (if needed later).
- The ability to experiment with **alternate system prompts** over different sessions and models for A/B–type explorations.

---

### 7. Tools and Tool Registry

#### 7.1 Tool Registration

- Tools shall be supported by registering **Python files and functions** that the Strands Agent can call as tools.
- A **tool registry** will be maintained under `server/tools/.chat/`, for example:
  - `server/tools/.chat/tools.json` or `tools.yaml`.
- For v1, the registry structure can be:
  - A **simple list of Python module (or module:object) paths**, such as:
    - `"server.agents.tools.calculators.retirement_savings"`
    - `"server.agents.tools.market_data.fetch_latest_rates"`
- The design should:
  - Specify how the registry is loaded and validated at runtime.
  - Ensure that the Strands Agent is initialized with tools resolved from this registry for all sessions (i.e., one global tool set).

#### 7.2 Tool Semantics

- There is a single **global tool registry across all sessions**:
  - If the tool registry is modified while a session is active, any system prompts that direct the LLM to use a newly added tool should be allowed (no session-level tracking of registry changes).
- The onus is on the **tool creator** to:
  - Ensure that tools conform to Strands’ expected structure and signatures.
  - Provide docstrings sufficient for Strands Agent SDK to infer tool properties and provide them to the LLM.
- Strands’ standard behavior applies:
  - Missing tools → runtime errors.
  - Tools without proper docstrings → “blind tools” (still callable but with reduced guidance to the model).
- No additional **safety guardrails** around tools are required for this prototype beyond what Strands provides by default.

---

### 8. Conversation Manager Configuration

- For v1, only **`SlidingWindowConversationManager`** is supported and is treated as the default.
  - Later, support can be added for:
    - `SummarizingConversationManager`.
    - `SemanticSummarizingConversationManager`.
  - Those will be introduced via additional CLI options and configuration fields (out of scope for this design).

#### 8.1 Defaults and Overrides

- Defaults (set in `config.yaml`):
  - `window_size`: 20.
  - `should_truncate_results`: `true`.
- CLI options:
  - `--window-size=<int>`: override for `window_size`.
  - `--should-truncate-results=<bool>`: override for `should_truncate_results` (e.g., `--should-truncate-results=False`).
- The design should:
  - Ensure these options update the session-level configuration (stored in `metadata.json`) when used.
  - Ensure that the `SlidingWindowConversationManager` is constructed with the effective per-session configuration.

---

### 9. CLI Behavior and UX

#### 9.1 Modes

- **Interactive mode** (default):
  - No `--single` flag → interactive chat.
  - Behavior:
    - After configuration and session resolution, the CLI enters a loop:
      - Displays prior conversation context in a simple, readable way.
      - Prompts the user for input.
      - Sends the input to the Chat Agent, streams the agent’s response.
      - Stores the turn in the session’s history and metadata.
    - Loop ends on EOF (Ctrl-D) or explicit exit command (if implemented).

- **Single-turn mode**:
  - `--single` flag provided.
  - Behavior:
    - Prompts the user once (or reads input from stdin).
    - Executes exactly one user → assistant turn.
    - Updates the session history and metadata.
    - Prints the session UUID on completion.

#### 9.2 Terminal UI and Streaming

- The design should support a richer **TUI** while keeping a fallback path:
  - Prefer **Textual** (or similar modern TUI framework) for a more chat-like experience:
    - Scrollable display of past user/assistant turns.
    - A fixed input area at the bottom for user prompts.
    - Streaming assistant responses into a display pane.
  - If Textual proves too heavy for initial implementation, the Coding Agent may:
    - Use a simpler approach with `prompt_toolkit` or raw `stdin/stdout` and minimal ANSI controls.
    - The design should call out this tradeoff and allow discussion with the user if needed.

- **Streaming behavior**:
  - Responses should be streamed **token-by-token** (or the closest equivalent supported by the underlying model APIs).
  - The TUI should:
    - Show tokens as they arrive.
    - Clearly separate completed assistant messages from previous turns (e.g., blank lines, prefixes).

#### 9.3 Logging and Verbosity

- Logging destination:
  - **File** under `server/tools/.chat/` (e.g., `logs/chat.log`).
  - Logs should include:
    - Timestamps.
    - Session ID.
    - High-level operations (session create/switch/delete, model selection, config load, errors).
    - Optionally, references to prompt versions or model calls (but not necessarily full prompt text).
- CLI flag:
  - `--verbose`:
    - When provided, also emit **debug logs to stdout** (in addition to the log file).
    - Debug logs may include more detailed information (e.g., resolved config, per-turn telemetry summaries).
  - Without `--verbose`, stdout should remain relatively quiet, focusing on:
    - Chat content.
    - Essential session identifiers.
    - Critical errors.

#### 9.4 Help and Usage

- The CLI must support `--help` and `-h` options:
  - When invoked with either flag (and no other action flags), the tool should:
    - Print a summary of:
      - Supported modes (interactive vs `--single`).
      - Session-related options (`--new-session`, `--session=<UUID>`, `--list-sessions`, `--description=...`, `--delete-session=<UUID>`).
      - Configuration-related options (`--model`, `--window-size`, `--should-truncate-results`, system prompt options).
      - Tooling/logging options (`--verbose`).
    - Provide at least one concrete example of typical usage for:
      - Starting a new session.
      - Resuming an existing session.
      - Running a one-off single-turn interaction.
  - After printing help, the CLI should exit without starting a chat.

---

### 10. Testing Strategy

- **Unit tests are required** for this prototype.
  - Location: `server/tests/`.
  - Suggested focus areas:
    - Session management:
      - Creation, switching, listing, description updates, deletion.
      - Correct handling of `RETIRE_CURRENT_SESSION_ID`.
    - Config resolution:
      - Erroring when `config.yaml` is missing or incomplete.
      - Correct precedence of CLI flags over `config.yaml`.
      - Persistence of configuration changes to session metadata.
    - System prompt behavior:
      - Error when both `--system-prompt` and `--system-prompt-file` are provided.
      - Persistence and update of system prompt text and metadata.
    - Tool registry handling:
      - Loading from registry file.
      - Detection of invalid module paths.
    - Logging behavior:
      - Basic tests that logs are written to file and `--verbose` adds stdout logs.
  - Full integration tests of real OpenAI/Gemini calls are not required for unit testing; mocking or faking model clients is acceptable there.

- **End-to-end behavior**:
  - The implemented prototype must support at least one **real LLM integration** (e.g., `gpt-5.1-mini`) so that:
    - A user can run the CLI, send messages, and receive real model responses.
    - Basic interactive and single-turn flows work end-to-end without mocks when configured with valid credentials.

---

### 11. Extensibility and Future Considerations

While this design targets a local spike, it should acknowledge likely future directions:

- **Persistence backend evolution**:
  - Current: JSON files under `server/tools/.chat/sessions/`.
  - Next step: SQLite for better concurrency and querying.
  - Long term: Cloud-based persistence using AgentCore Memory + DynamoDB; the current session store should be abstracted behind a small interface to ease migration.

- **Conversation manager evolution**:
  - Add support for `SummarizingConversationManager` and `SemanticSummarizingConversationManager`, toggled via config/CLI.
  - Allow per-session selection of conversation manager type.

- **Model and evaluation extensions**:
  - Add support for more models by extending the model mapping table.
  - Introduce simple mechanisms for:
    - A/B testing system prompts and models across sessions.
    - Recording evaluation metadata (ratings, notes) per session or turn.

- **Web UI integration**:
  - Long-term goal is to replace the CLI with a web-based chat UI.
  - The separation between:
    - CLI/TUI.
    - Chat Agent API in `server/chat/`.
    - Session store.
  - Should allow a web server to call the same Chat Agent functions, reusing prompt logic, tools, and persistence.

This design is intended to be sufficiently detailed that a Coding Agent can implement the prototype with minimal additional clarification. Any deviations from this design should be surfaced back to the Design Agent for review and approval.


