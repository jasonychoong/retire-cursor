# Implementation Tasks — Data Completeness Tools

## Overview

This task list documents the implementation work for the **data-completeness tools** that support the retirement-planning chat prototype, as specified in `design-completeness-tools.md`.

Conventions:
- Lead Agent per task (Design, Prompt, Coding, Web, Cloud, Tool)
- Dependencies listed using task numbers
- Each numbered task may contain one or more sub-tasks (e.g., 1.1, 1.2) with clear acceptance criteria
- **Unit tests** for core logic should generally be implemented as part of each major functional task
- Design review is expected at the completion of each major numbered task before moving on to the next

---

## Task 1: Canonical Topics and Shared Persistence Structures

Objective: Establish shared Python structures for canonical topics and JSONL-based persistence so that all three tools behave consistently and align with `design-completeness-tools.md` §3–4.

### 1.1 Topic Enum and Record Schemas

- [x] Status: **Completed** (Nov 25, 2025, 1:00PM)
- Lead: Coding Agent
- Scope:
  - Introduce a small shared module under `server/agents/tools/` (e.g., `completeness_common.py`) that defines:
    - A canonical topic enum or equivalent constants for:
      - `income_cash_flow`, `healthcare_medicare`, `housing_geography`,
        `tax_efficiency_rmds`, `longevity_inflation`, `long_term_care`,
        `lifestyle_purpose`, `estate_planning`.
    - Lightweight data structures (e.g., `dataclass` or typed dicts) that mirror:
      - The **information record** schema (`topic`, `subtopic`, `fact_type`, `value`, `confidence`, `session_id`, `created_at`, optional `id`).
      - The **completeness snapshot** schema (`session_id`, `scores[]`, `created_at`).
  - Provide simple helpers to:
    - Validate topic IDs against the canonical set (raising a clear error for invalid topics).
    - Resolve the current session’s base directory under `server/tools/.chat/sessions/<session-id>/` using existing session/persistence conventions.
    - Append a JSON-serializable object as a single line of JSON to a given `*.jsonl` file, creating directories/files on first write.
- Deliverables:
  - A reusable module containing:
    - Canonical topic definitions used by all completeness tools.
    - Helper functions for topic validation and JSONL append operations.
  - Basic unit tests covering:
    - Topic validation (accept valid, reject invalid).
    - JSONL append behavior (file creation on first write, append-only semantics).
- Dependencies: Existing session storage layout from main `tasks.md` Task 1 (already implemented).

---

## Task 2: Information Capture and Query Tools

Objective: Implement the `information` and `information_query` tools that persist and retrieve structured user information for the current session.

### 2.1 Implement `information` Tool

- [x] Status: **Completed** (Nov 25, 2025, 2:10PM)
- Lead: Coding Agent
- Scope:
  - Implement `server/agents/tools/information.py` as described in `design-completeness-tools.md` §5:
    - Define the `@tool`-decorated `information(...)` function with signature:
      - `topic: str`, `value: str`, `subtopic: str | None = None`,
        `fact_type: str | None = None`, `confidence: float = 0.9`.
    - Use the shared helpers from Task 1 to:
      - Validate `topic` against the canonical enum.
      - Infer the current session’s directory and open/append to `information.jsonl`.
      - Stamp `session_id` and `created_at` on each record server-side.
    - Ensure **one call per distinct fact** is the default behavior; do not attempt to coalesce multiple facts into one record.
    - Treat corrections as new records with updated `value` and (optionally) `fact_type`.
  - Make the return value a short, user-agnostic confirmation string or record identifier (suitable for logging/tool traces, not for direct user display).
- Deliverables:
  - Working `information` tool that:
    - Writes valid records to `information.jsonl` for the current session.
    - Rejects invalid topics with a clear error.
  - Unit tests that:
    - Verify correct record shape and JSONL writing behavior.
    - Exercise corrections/updates by writing multiple records for the same conceptual fact.
- Dependencies: 1.1.

### 2.2 Implement `information_query` Tool

- [x] Status: **Completed** (Nov 25, 2025, 2:10PM)
- Lead: Coding Agent
- Scope:
  - Implement `server/agents/tools/information_query.py` as described in `design-completeness-tools.md` §6:
    - Define the `@tool`-decorated `information_query() -> list[dict]` function with **no arguments**.
    - Use the shared helpers from Task 1 to:
      - Locate `information.jsonl` for the current session.
      - If the file does not exist, return an empty list rather than raising.
      - Read each line, parse JSON, and return a list of records.
    - Keep this tool strictly **read-only**; no writes or transformations should occur.
  - Ensure returned objects closely match the stored schema (including `session_id`, `topic`, `value`, `confidence`, timestamps), so the LLM can reason about them.
- Deliverables:
  - Working `information_query` tool that:
    - Returns an empty list when no information has been captured.
    - Returns all existing information records when present.
  - Unit tests that:
    - Cover the empty-file / missing-file case.
    - Cover the case with multiple records and verify correct parsing.
- Dependencies: 1.1, 2.1.

---

## Task 3: Completeness Scoring Tool and Integration

Objective: Implement the `completeness` tool and wire all three tools into the existing tool registry so they are available to the agent under the `completeness.txt` system prompt.

### 3.1 Implement `completeness` Tool

- [x] Status: **Completed** (Nov 26, 2025, 8:49AM)
- Lead: Coding Agent
- Scope:
  - Implement `server/agents/tools/completeness.py` as described in `design-completeness-tools.md` §7:
    - Define the `@tool`-decorated `completeness(scores: List[Dict[str, Any]]) -> str` function.
    - Reuse the shared helpers from Task 1 to:
      - Validate each `topic` in `scores` against the canonical enum.
      - Validate that `score` is an integer in the range 0–100 (inclusive).
      - Append a completeness snapshot entry to `completeness.jsonl` that includes:
        - `session_id` (from context),
        - the raw `scores` list,
        - `created_at` timestamp.
    - Do not compute deltas or enforce thresholds inside the tool; assume the LLM/system prompt already decided when to call it.
  - Make the return value a short confirmation string or identifier for the snapshot, suitable for logging but not intended for direct user display.
- Deliverables:
  - Working `completeness` tool that:
    - Validates inputs rigorously and fails fast on invalid topics or scores.
    - Writes well-formed snapshot entries to `completeness.jsonl`.
  - Unit tests that:
    - Cover valid multi-topic snapshots.
    - Cover invalid topic IDs and out-of-range scores.
- Dependencies: 1.1.

### 3.2 Tool Registry Wiring and Smoke Test

- [x] Status: **Completed** (Nov 26, 2025, 8:49AM)
- Lead: Coding Agent
- Scope:
  - Register the three new tools (`information`, `information_query`, `completeness`) in the existing tool registry file (e.g., `server/tools/.chat/tools.yaml`) following the established pattern from Task 4 in the main `tasks.md`.
  - Confirm that, when the chat agent is started with the `completeness.txt` system prompt:
    - The LLM can see these tools in its tool list.
    - Tool calls are correctly logged in `history.json` and `metadata.json` per the existing tool-logging behavior.
  - Add a lightweight integration test (or documented manual test) that:
    - Simulates a short conversation where the model (or a stub) calls `information`, `information_query`, and `completeness`.
    - Verifies that the corresponding JSONL files are created and updated under the current session.
- Deliverables:
  - Updated tool registry including all three completeness-related tools.
  - A passing smoke test (automated or manual) demonstrating end-to-end use of the completeness tools within a session.
- Dependencies: 2.1, 2.2, 3.1, existing tool registry infrastructure from main `tasks.md` Task 4.

---

## Task 4: Refinement and Design Review Feedback

Objective: Incorporate feedback from the Design Agent and ensure the tools behave as intended in realistic conversations.

### 4.1 Review-Driven Refinements

- [ ] Status: **Not Started**
- Lead: Coding Agent (in collaboration with Design Agent)
- Scope:
  - After Tasks 1–3 are implemented, conduct a review with the Design Agent focusing on:
    - Alignment of tool behavior with `design-completeness-tools.md` and `completeness.txt`.
    - Whether the information records and completeness snapshots are sufficiently expressive for downstream UI/analytics.
    - Any gaps in topic coverage, logging, or error handling observed in trial conversations.
  - Apply small, targeted refinements based on this feedback, such as:
    - Adjusting field names or types in stored records (within reason and with migration notes if needed).
    - Tweaking validation behavior or default confidence values.
    - Adding or refining tests to cover newly identified edge cases.
- Deliverables:
  - Agreed-upon refinements merged, with notes (in comments or a short markdown changelog) documenting any intentional deviations from the original design.
  - Updated tests reflecting the final expected behavior.
- Dependencies: 1–3.

---

## Task 5: Completeness and Profile Monitor CLIs

Objective: Implement two read-only CLI tools (`server/tools/completeness` and `server/tools/profile`) that provide live views of completeness scores and captured user information for a given session, as described in `design-completeness-tools.md` §10.

### 5.1 Implement `server/tools/completeness` Monitor

- [x] Status: **Completed** (Nov 27, 2025, 8:28AM)
- Lead: Coding Agent
- Scope:
  - Implement an executable CLI script `server/tools/completeness` that:
    - Accepts an optional `--session <UUID>` argument.
    - Resolves the target session as follows:
      - If `--session` is provided, use that ID.
      - Otherwise, load `server/tools/.chat/sessions/index.json` and pick the entry with `is_current: true` (the same “current” session that `chat --list-sessions` highlights).
    - Enters a polling loop (e.g., every 2 seconds) that:
      - Looks for `<chat_dir>/sessions/<session_id>/completeness.jsonl`.
      - If the file is missing or contains no records:
        - Clears the terminal (via ANSI) and prints `awaiting data...`.
      - If records exist:
        - Reads all entries, merges them to compute the **latest score per topic** (one canonical score per topic ID).
        - Clears the screen and prints one line per canonical topic, in the fixed order:
          1. `income_cash_flow`
          2. `healthcare_medicare`
          3. `housing_geography`
          4. `tax_efficiency_rmds`
          5. `longevity_inflation`
          6. `long_term_care`
          7. `lifestyle_purpose`
          8. `estate_planning`
        - Renders each line as a **text arrow** based on the score:
          - A `|` character aligned across all lines.
          - Then zero or more `=` characters plus a final `>` character, where each character (either `=` or `>`) represents **5 completeness points**.
          - After the arrow (or just `|` if score is 0), prints a space and the numeric score.
          - Example:
            - `1. income_cash_flow     |================> 85`
            - `2. healthcare_medicare  |========> 45`
            - `3. housing_geography    |===========> 60`
            - `4. tax_efficiency_rmds  | 0`
            - ...
      - After the topic lines, prints:
        - `Help me explore a specific topic (enter number 1-8)> `
      - If the user enters `1`–`8` and presses Enter:
        - Looks up the corresponding canonical topic ID based on the number (1 → `income_cash_flow`, 2 → `healthcare_medicare`, etc.).
        - Loads the JSON file `server/tools/.chat/user-prompts/explore-topic.json`, which maps topic IDs to **recommended user prompts**.
        - Prints the mapped prompt text for that topic to stdout so the user can copy/paste it into `server/tools/chat`.
        - Does **not** attempt to inject into an existing chat process or manage sessions beyond printing the suggestion.
    - Continues polling and updating until terminated by the user with Ctrl-C or Ctrl-D.
- Deliverables:
  - A working `server/tools/completeness` CLI that:
    - Correctly resolves the target session.
    - Displays per-topic arrows with aligned formatting.
    - Handles missing/empty `completeness.jsonl` with an `awaiting data...` message.
    - Prints a topic-specific recommended prompt when the user selects a topic number.
  - Basic tests (or a small manual test script/README section) describing how to run `chat` and `completeness` side by side and what to look for.
- Dependencies: 1.1, 3.1–3.2, main `tasks.md` Task 1 (session store) and Task 3 (chat CLI/session management).

### 5.2 Implement `server/tools/profile` Monitor

- [x] Status: **Completed** (Nov 27, 2025, 8:28AM)
- Lead: Coding Agent
- Scope:
  - Implement an executable CLI script `server/tools/profile` that:
    - Accepts an optional `--session <UUID>` argument and resolves the target session using the same rules as `server/tools/completeness`.
    - Enters a polling loop (e.g., every 2 seconds) that:
      - Looks for `<chat_dir>/sessions/<session_id>/information.jsonl`.
      - If the file is missing or contains no records:
        - Clears the screen and prints `awaiting data...`.
      - If records exist:
        - Reads all entries, groups them by:
          - `topic` (canonical ID) in a stable order,
          - then `subtopic` (string), and
          - chronological order within each subtopic.
        - Clears the screen and renders a hierarchical view:
          - Top-level: topic name (e.g., `income_cash_flow`).
          - Second-level: subtopic (e.g., `retirement_timing`, `ages`, `spending`, `social_security`, `assets`, `asset_breakdown`).
          - Third-level: indented bullet or prefixed line for each record, with a label based on `fact_type` or simple heuristics:
            - e.g., `Goal: ...`, `Fact: ...`, `Preference: ...`.
        - The formatting can follow the example given in the Prompt document (grouped and indented text), using monospaced indentation and stdout-only printing (no external TUI libraries).
    - Continues running and refreshing until terminated by the user (Ctrl-C/D).
- Deliverables:
  - A working `server/tools/profile` CLI that:
    - Resolves the target session.
    - Correctly groups and displays information records by topic and subtopic.
    - Handles empty/missing `information.jsonl` with an `awaiting data...` message.
  - Basic tests or manual instructions demonstrating how to run `chat` and `profile` together and interpret the output.
- Dependencies: 1.1, 2.1–2.2, main `tasks.md` Task 1 (session store) and Task 3 (chat CLI/session management).




