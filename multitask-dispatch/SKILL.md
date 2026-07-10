---
name: multitask-dispatch
description: Detect multi-task prompts ("do A, B, and C") and fan them out to parallel subagents instead of running them one at a time on the main thread. Each subtask gets model/effort picked automatically (via adaptive-effort's tier table) unless the user pins model/effort in the prompt. Use whenever a prompt contains 2+ independent asks — separated by "and", commas, numbered/bulleted lists, or multiple imperative verbs — and those asks don't depend on each other's output.
---

# Multitask Dispatch

## Purpose

User prompt "do A, B, and C" run sequential on main thread = slow, wastes token budget on trivial parts, no isolation between subtasks. This skill: split into subtasks, check independence, fire independent ones as parallel `Agent` calls in one message, pick model/effort per subtask (or honor user-pinned model/effort), merge results.

## Trigger

2+ distinct asks in one prompt: "and" chains, comma lists, numbered/bulleted lists, multiple imperative verbs ("fix X, add Y, refactor Z"). Not for a single task with sub-steps that depend on each other in sequence — dependency chains still run sequentially (see Step 2).

## Step 1 — Split

Break prompt into discrete tasks. Each task = one deliverable, independently describable without the others.

## Step 2 — Dependency check

For each pair of tasks: does B need A's output/files to start? If yes, sequential (dependent tasks run in order, independent ones parallel). Don't force parallelism onto a real dependency chain — that produces wrong results, not speed.

## Step 3 — Model/effort per task

Default: classify each task with [[adaptive-effort]]'s tier table (complexity/risk/reversibility/ambiguity/repetition → low/medium/high/xhigh/max → haiku/sonnet/opus + thinking keyword). Don't re-derive the table here — reuse it.

**User override wins.** If the prompt names a model or effort ("use opus for all of these", "low effort is fine", "think hard on the security one"), apply it verbatim to the scoped task(s) and skip auto-classification for those. Global override ("all haiku") applies to every dispatched task; scoped override ("careful on B") applies to just that task.

## Step 3b — Shared-context check

Before dispatching, check if 2+ tasks need the same groundwork first (same research, same file inventory, same design decision). If so, run that shared step once (inline or single agent), feed its result into each dependent task's prompt, then fan out the rest. Don't make every parallel agent re-derive the same context — that's N redundant sub-explorations.

## Step 4 — Dispatch

One message, multiple `Agent` tool calls — this is what makes it parallel, not multiple messages. Each call gets a self-contained prompt (the subagent has no conversation history): task description, relevant file paths already known, expected output form, and the result-format contract below.

Sequential tasks (from Step 2) get their own `Agent` call after the dependency resolves, not bundled into the same parallel batch.

State each dispatch as it happens: `→ dispatching B to sonnet, high` — one line per task, so the user can track what's running without polling.

**Result-format contract.** Every dispatched subagent prompt must ask for, at minimum: status (`done` / `blocked` / `needs-input`), files touched, and a one-line summary. This keeps Step 5's merge mechanical instead of re-reading full agent output to guess what happened.

## Step 5 — Merge

Collect each agent's result, reconcile into single response to user. Flag conflicts (two agents touched the same file) before reporting done.

**Partial failure.** If a subagent comes back `blocked`, errors, or its output doesn't satisfy the task: retry once with clarified instructions. Still failing → surface that specific task to the user as unresolved, report the others as done. Never silently drop a failed task from the summary.

## Skip conditions

- Single task, however large — decompose internally if needed but no multi-agent fan-out required just because a task has many steps.
- Tasks are trivial one-liners (adaptive-effort's one-liner exception) — do inline, dispatch overhead not worth it.
- Tasks are tightly coupled / all touch the same file — parallel edits collide, do sequential.

## Anti-patterns

- Splitting a single coherent task into fake "subtasks" to look parallel.
- Parallel-dispatching tasks that write to the same file.
- Ignoring a user's explicit model/effort instruction and auto-classifying anyway.
