# Implementation Tasks — Prompt Design Chat Prototype

## Overview

This task list documents the implementation work for the **local Strands Agent–based chat prototype** used to investigate prompt designs for a retirement-planning assistant.

Conventions:
- Lead Agent per task (Design, Prompt, Coding, Web, Cloud, Tool)
- Dependencies listed using task numbers
- Each numbered task may contain one or more sub-tasks (e.g., 1.1, 1.2) with clear acceptance criteria
- **Unit tests** for core logic should generally be implemented as part of each major functional task; **Task 7** focuses on consolidating coverage and filling remaining gaps.

---

## Task 1: Session Persistence and Configuration Foundations

Objective: Establish the local persistence layer and configuration mechanics needed to support session-based conversations across CLI invocations.

### 1.1 Session Storage Layout and Index

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement the on-disk layout under `server/tools/.chat/`:
    - `config.yaml` (placeholder file to be populated in Task 1.2).
    - `sessions/` directory containing:
      - `sessions/index.json` for tracking:
        - `id` (UUIDv4), `created_at`, `description`, `is_current`.
      - Per-session directories: `sessions/<session_id>/`.
      - Empty `history.json` and `metadata.json` files for new sessions.
  - Implement a small **session store abstraction** in Python (e.g., `SessionStore` class in `server/agents/chat` or a dedicated module) to:
    - Create sessions with new UUIDv4 IDs.
    - Read/write `index.json`, `history.json`, and `metadata.json`.
    - Enforce the structure described in `design.md` §4.2.
  - Implement CLI-level session listing and management operations:
    - `--list-sessions`: read `index.json` and print a simple table.
    - `--description="<text>" --session=<UUID>`: update description without switching.
    - `--delete-session=<UUID>`: delete all per-session files and remove from `index.json`.
- Deliverables:
  - Session directory structure created on first use if missing.
  - `sessions/index.json` maintained accurately as sessions are created, described, and deleted.
  - Unit tests for `SessionStore` basic operations (create, list, describe, delete).
- Dependencies: None (foundational for later tasks).

### 1.2 Config File Handling and Validation

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Define the expected schema for `server/tools/.chat/config.yaml` consistent with `design.md` §5.1, including:
    - `model`, `window_size`, `should_truncate_results`, and any other required fields.
  - Implement a configuration loader that:
    - Fails fast with clear error messages when `config.yaml` is missing or required keys are absent.
    - Returns a strongly-typed configuration object (e.g., Pydantic or dataclass).
  - For **new sessions**:
    - Load and validate `config.yaml`.
    - Merge CLI overrides (e.g., `--model`, `--window-size`, `--should-truncate-results`, system prompt options) on top.
    - Persist the effective configuration into `metadata.json` as the session baseline.
  - For **existing sessions**:
    - Load baseline configuration from `metadata.json`.
    - Apply CLI overrides, and persist any changes back to `metadata.json` as the new baseline.
- Deliverables:
  - Robust configuration loader with clear error messages.
  - Config precedence behavior matching `design.md` §5.2.
  - Unit tests for:
    - Missing/invalid `config.yaml`.
    - New vs existing session config resolution.
- Dependencies: 1.1.

---

## Task 2: Chat Agent Layer (Strands, Models, and System Prompts)

Objective: Implement the Strands-based chat agent layer that the CLI will invoke, including model mapping, conversation manager, and system prompt handling.

### 2.1 Model Mapping and Provider Integration

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement a **model registry/mapping** in `server/agents/chat` that:
    - Maps logical model codes (e.g., `gpt-5.1-mini`, `gpt-5.1`, `gemini-2.5`, `gemini-2.5-flash`) to:
      - Provider (OpenAI vs Gemini).
      - Concrete API model IDs and client configuration (e.g., API base URLs).
  - Implement a small factory or helper that:
    - Creates an appropriate Strands `model` object for a given model code.
    - Reads API keys and other sensitive configuration from environment variables (not from `config.yaml`).
  - Ensure at least **one real integration** is wired end-to-end (e.g., `gpt-5.1-mini`), so the prototype can talk to a real LLM when configured properly.
- Deliverables:
  - Model mapping module with a documented set of supported model codes.
  - Unit tests for mapping logic and provider selection.
- Dependencies: 1.2.

### 2.2 Conversation Manager and Strands Agent Construction

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement a helper in `server/agents/chat` that:
    - Constructs a `SlidingWindowConversationManager` using:
      - `window_size` and `should_truncate_results` from the effective session config.
  - Implement a function (e.g., `build_agent(config, tools, session_manager, system_prompt)`) that:
    - Instantiates a Strands `Agent` with:
      - The mapped model from Task 2.1.
      - The conversation manager described above.
      - The current effective system prompt.
      - A set of tools resolved from the tool registry (Task 4.2).
  - Ensure the agent can be reused across turns within a session or recreated cheaply as needed.
- Deliverables:
  - A reusable agent-construction function with a clear signature for the CLI to call via a thin wrapper.
  - Unit tests that verify agent construction using test/dummy models.
- Dependencies: 2.1, 4.2 (tools can be stubbed initially if needed).

### 2.3 System Prompt Persistence and Telemetry

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement logic for handling system prompts in accordance with `design.md` §6 and §4.2:
    - Accept system prompts via:
      - `--system-prompt-file <filename>` (relative or absolute).
      - `--system-prompt="<text>"`.
    - Enforce that **only one** of these options is allowed per invocation; otherwise, fail with a clear error.
  - For each session:
    - Maintain current effective system prompt metadata in `metadata.json`:
      - `system_prompt_text`, `system_prompt_source`, and `system_prompt_file_path` when applicable.
    - Log system prompt changes in `history.json`:
      - Append a `role: "system"` entry when a system prompt is first set.
      - Append another `role: "system"` entry **only when** the prompt changes.
    - Mirror system prompt changes into per-turn telemetry (configuration changes) for that turn in `metadata.json`.
- Deliverables:
  - Consistent system prompt handling (source enforcement, persistence, and history logging).
  - Unit tests covering:
    - Error on both system prompt options being used together.
    - Proper history and metadata updates when the system prompt changes.
- Dependencies: 1.1, 1.2.

---

## Task 3: CLI Core Behavior (Sessions, Modes, and Help)

Objective: Implement the main CLI entrypoint (`server/tools/chat`) and core behaviors for interactive and single-turn modes, including session resolution and help output.

### 3.1 CLI Entry Script and Argument Parsing

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement `server/tools/chat` as an executable Python script (shebang + `chmod a+x`).
  - Use a standard CLI parsing library (e.g., `argparse` or `typer`) to support:
    - Modes:
      - Default interactive mode.
      - `--single` for single-turn mode.
    - Session options:
      - `--new-session`, `--session=<UUID>`, `--list-sessions`, `--description="<text>"`, `--delete-session=<UUID>`.
    - Configuration options:
      - `--model`, `--window-size`, `--should-truncate-results`, `--system-prompt`, `--system-prompt-file`.
    - Logging:
      - `--verbose`.
    - Help:
      - `--help` and `-h` to print usage and examples, then exit.
  - Integrate with the `SessionStore` abstraction and configuration loader from Task 1.
- Deliverables:
  - A single entry script providing clear, discoverable options with `--help`.
  - Unit tests for CLI argument parsing and basic routing logic (can be limited to parsing + behavior flags).
- Dependencies: 1.1, 1.2.

### 3.2 Session Resolution and Lifecycle in CLI

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement session resolution logic in the CLI as described in `design.md` §3.2 and §4.1:
    - Identify whether an invocation is for a **new** or **existing** session based on:
      - `RETIRE_CURRENT_SESSION_ID`.
      - `--new-session`.
      - `--session=<UUID>`.
  - For new sessions:
    - Load and validate `config.yaml`, apply CLI overrides, persist effective config to `metadata.json`.
  - For existing sessions:
    - Load baseline config from `metadata.json`, apply CLI overrides, persist updated config.
  - Ensure `RETIRE_CURRENT_SESSION_ID` is appropriately set/updated/cleared as sessions are created or deleted.
- Deliverables:
  - Consistent behavior for session creation, switching, and reuse across invocations.
  - Unit tests for:
    - First-run behavior with no `RETIRE_CURRENT_SESSION_ID`.
    - `--new-session` and `--session=<UUID>` combinations.
    - Delete-session behavior when deleting the current session.
- Dependencies: 3.1, 1.1, 1.2.

### 3.3 Interactive and Single-Turn Control Flow (Non-TUI)

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement the business logic for:
    - Interactive mode:
      - Load prior `history.json` and display a simple textual summary of previous turns.
      - Loop reading user input until EOF (Ctrl-D) or explicit exit.
      - For each input:
        - Invoke the Chat Agent API with the current session ID and config.
        - Stream responses token-by-token to stdout (a basic implementation is acceptable here; detailed TUI handled in Task 4.1).
        - Append new messages and telemetry to `history.json` and `metadata.json`.
      - On normal exit, print the session UUID.
    - Single-turn mode:
      - Read exactly one user input (from stdin or a prompt).
      - Execute a single round-trip and persist the new turn.
      - Print the session UUID on completion.
  - This task focuses on **core behavior**, using simple stdout printing; richer TUI comes later.
- Deliverables:
  - A working CLI that supports end-to-end interaction (with at least one real LLM wired via Task 2.1), even before the TUI enhancements.
  - Integration tests (can be light) to confirm a minimal chat flow succeeds when valid API keys are set.
- Dependencies: 2.2, 2.3, 3.2.

---

## Task 4: Tool Registry and Initial Tools

Objective: Implement the tool registry and wire tools into the Strands Agent so that system prompts can refer to callable tools.

### 4.1 Tool Registry File and Loader

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Design and implement a tool registry file under `server/tools/.chat/` (e.g., `tools.yaml` or `tools.json`):
    - Represented as a **simple list of Python module (or module:object) paths**, such as:
      - `"server.agents.tools.calculators.retirement_savings"`
      - `"server.agents.tools.market_data.fetch_latest_rates"`.
  - Implement a loader that:
    - Reads the registry file at startup.
    - Imports the specified modules/objects.
    - Validates that they adhere to Strands’ expected tool signatures.
    - Returns a list of tool objects to be passed into the Strands `Agent`.
  - Handle basic error cases gracefully (e.g., missing module paths) with clear log output.
- Deliverables:
  - Tool registry file with at least one example tool module path.
  - Loader function with corresponding unit tests.
- Dependencies: 1.1.

### 4.2 Tool Integration with Strands Agent

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Integrate the tool loader from Task 4.1 into the agent construction function (Task 2.2):
    - Ensure that all sessions use a **single global tool registry** (no per-session variants).
    - If the tool registry is modified and the CLI is restarted, new tools should become available without changing session metadata.
  - Provide at least one simple tool implementation (e.g., a basic calculator or placeholder financial function) under `server/agents/tools/` for testing purposes.
- Deliverables:
  - Tools visible to the LLM (via Strands’ tool mechanism) in actual chat interactions.
  - Unit/integration tests verifying that a tool can be invoked by the model (can use a mock model that always calls a known tool).
- Dependencies: 2.2, 4.1.

---

## Task 5: TUI and Streaming UX Enhancements

Objective: Enhance the CLI experience to more closely resemble a chat UI, including scrollable history and token-level streaming.

### 5.1 TUI Implementation (Textual or Alternative)

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement a richer TUI for interactive sessions, preferably using **Textual**:
    - Scrollable area for prior user/assistant/system messages.
    - Fixed input field at the bottom for entering user messages.
    - Live updating of the assistant’s current response as tokens stream in.
  - If Textual proves too heavy or problematic:
    - Implement a simpler fallback using `prompt_toolkit` or raw ANSI control codes.
    - Document the chosen approach in a brief comment or note.
  - Ensure that TUI remains optional/fallback-friendly:
    - If TUI cannot be initialized (e.g., non-TTY environment), revert gracefully to the basic stdout behavior implemented in Task 3.3.
- Deliverables:
  - Interactive chat UI within the terminal that feels closer to a browser chatbox experience.
  - Manual testing instructions or small examples to exercise the TUI.
- Dependencies: 3.3.

### 5.2 Token-Level Streaming Integration

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Ensure that both the basic CLI flow and the TUI use **token-by-token streaming** (or the closest approximation supported by the model APIs).
  - Implement streaming callbacks/hooks in the Chat Agent layer that:
    - Emit tokens incrementally to the caller.
    - Allow the TUI/CLI to update the display in near real time.
  - Confirm that streaming works for at least one real model integration (e.g., `gpt-5.1-mini`).
- Deliverables:
  - Visible streaming behavior in the terminal during assistant responses.
  - Tests (can be limited to mocked streaming behavior) to verify the callback pipeline.
- Dependencies: 2.2, 3.3, 5.1.

---

## Task 6: Logging and Diagnostics

Objective: Provide file-based logging and optional verbose output to aid debugging during the spike.

### 6.1 Logging Infrastructure

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Implement a logging setup that:
    - Writes logs to `server/tools/.chat/logs/chat.log` (creating directories as needed).
    - Includes timestamps, session IDs, and key operations (session creation, switching, deletion, model selection, errors).
  - Integrate logging with:
    - Session store operations (Task 1.1).
    - Config loading and overrides (Task 1.2).
    - Chat agent invocations and tool usage (Tasks 2 and 4).
  - Implement `--verbose` behavior:
    - When present, mirror debug-level logs to stdout.
    - When absent, keep stdout quiet aside from chat content and critical errors.
- Deliverables:
  - Consistent log output across the system, with minimal configuration.
  - Unit tests verifying that logs are written to file and that `--verbose` enables console logging.
- Dependencies: 1.1, 3.1.

---

## Task 7: Testing and End-to-End Validation

Objective: Ensure the prototype is testable and that a minimal end-to-end flow works with a real LLM.

### 7.1 Unit Test Suite

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Ensure there is a cohesive test suite under `server/tests/` that, together with per-task tests, covers the unit-test requirements outlined in `design.md` §10:
    - Session management (create/list/describe/delete + `RETIRE_CURRENT_SESSION_ID`).
    - Config resolution and precedence.
    - System prompt behavior and logging.
    - Tool registry loading and error handling.
    - Logging configuration and `--verbose`.
  - Fill any gaps left by earlier tasks by adding or refining tests.
  - Use mocks/fakes for model calls and tools where appropriate to keep tests deterministic.
- Deliverables:
  - A passing unit test suite runnable via a standard test command (e.g., `pytest`).
- Dependencies: 1–6 (tests may be staged and expanded as earlier tasks complete).

### 7.2 End-to-End Smoke Test (Manual or Scripted)

- [ ] Status: **Not Started**
- Lead: Coding Agent
- Scope:
  - Document and/or script a simple end-to-end smoke test that:
    - Configures a real model (e.g., `gpt-5.1-mini`) with valid credentials.
    - Starts a new session via the CLI.
    - Runs:
      - A short interactive chat (2–3 turns).
      - A single-turn chat using `--single`.
    - Confirms that:
      - Responses are streamed to the terminal.
      - Session files (`history.json`, `metadata.json`, `index.json`) are updated.
      - Logs are written.
  - Optionally implement a minimal automated smoke test that can be run locally (skipping in CI if real credentials are unavailable).
- Deliverables:
  - A concise smoke test procedure (in comments or a short markdown file) and, optionally, a script.
- Dependencies: 2.1, 3.3, 6.1.


