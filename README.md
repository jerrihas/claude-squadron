# claude-squadron

Claude Code skill: detect multi-task prompts ("do A, B, and C") and fan them out to parallel subagents instead of running them one at a time on the main thread.

## What it does

- Splits a multi-ask prompt into discrete subtasks
- Checks dependencies — independent tasks run in parallel, dependent ones run sequentially
- Picks model/effort per subtask (via the companion [adaptive-effort](https://github.com/anthropics) tier logic — haiku/sonnet/opus + thinking level) unless the user pins a model/effort in the prompt
- Dispatches independent subtasks as parallel `Agent` calls in a single message
- Merges results with a result-format contract (status/files-touched/summary per subagent) and surfaces partial failures instead of dropping them

## Install

Drop `claude-squadron/` into your `~/.claude/skills/` directory.

```
cp -r claude-squadron ~/.claude/skills/
```

## Requires

Works best alongside an `adaptive-effort`-style skill for the model/effort tier table. Without one, do the tier classification inline using: complexity, risk/blast-radius, reversibility, ambiguity, repetition → route to haiku (low) / sonnet (medium-high) / opus (xhigh-max).
