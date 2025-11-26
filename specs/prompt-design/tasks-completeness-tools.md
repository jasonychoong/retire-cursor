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

- [ ] Status: **Not Started**
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

- [ ] Status: **Not Started**
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




