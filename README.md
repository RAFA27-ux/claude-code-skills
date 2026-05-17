# claude-code-skills
Claude Code Skills 
# claude-code-skills

> Stop babysitting Claude Code. These skills make it finish what it starts.

Most Claude Code sessions end the same way: Claude writes something, you run it, it breaks, Claude tries again, it breaks differently, you fix it yourself.

This repo is a set of **Skills** — structured instruction files that change how Claude Code *behaves*, not just what it knows. Each skill enforces a discipline: read before writing, test before shipping, debug with a limit, verify before handing off.

They were built out of real frustration, on a real project, after one too many "should be working now" moments that weren't.

---

## The problem with Claude Code (and most AI coding tools)

Claude Code is powerful. It's also undisciplined by default:

- It writes code without reading the codebase first
- It duplicates logic that already exists three files away
- It runs tests zero times before telling you it's done
- When something breaks, it guesses — and keeps guessing indefinitely
- It hands you back broken code and calls it finished

These aren't model failures. They're **behavioral failures**. The model is capable — it just doesn't have rules that force it to slow down and do things right.

Skills fix that.

---

## What's in this repo

Four skills, designed to work as a pipeline:

```
large-python-projects → pytest-discipline → self-verification → debugging-loop
```

| Skill | What it enforces |
|---|---|
| `large-python-projects` | Read the codebase before writing. Match existing patterns. No duplicates, no parallel abstractions. |
| `pytest-discipline` | Write tests. Run them. Fix them. Max 5 attempts — then hand off to human with a full report. |
| `self-verification` | Before declaring done: syntax check, import check, None safety, no debug artifacts, no secrets, no raw SQL. |
| `debugging-loop` | State root cause before touching anything. One fix per attempt. Max 3 attempts — then hand off with full context. |

---

## How to use

### Option 1: Project-level (recommended)

Copy the skills into your project's `.claude/skills/` folder:

```bash
git clone https://github.com/yourusername/claude-code-skills
cp -r claude-code-skills/skills/ your-project/.claude/skills/
```

Claude Code will automatically pick them up.

### Option 2: Global

Copy to your home directory so all projects benefit:

```bash
cp -r claude-code-skills/skills/ ~/.claude/skills/
```

### Option 3: Cherry-pick

Only want the debugging discipline? Just copy that one file:

```bash
cp claude-code-skills/skills/debugging-loop.md your-project/.claude/skills/
```

---

## Skill details

### `large-python-projects`

The root cause of most Claude Code mistakes: it writes code without understanding the codebase it's writing into.

This skill forces a mandatory orientation phase before any code is written:

1. Read `pyproject.toml`, `README`, architecture docs
2. Map the package structure — entry points, core abstractions, utilities, test layout
3. Find existing base classes, mixins, shared models — inherit, don't reinvent
4. Read the 2–5 files most relevant to the task

Then pattern extraction: match the project's type annotation style, docstring format, error handling convention, async patterns, and logging setup. Your new code should be indistinguishable from the existing code.

Then a duplicate check: does this logic already exist? Before creating anything new, search.

Only then: write.

---

### `pytest-discipline`

**Core rule: never output `[TASK_DONE]` while any test is failing.**

This skill governs the full test lifecycle:

- One test file per source file, mirrored directory structure
- Minimum bar per function: happy path + one error path + one edge case
- Match the project's test conventions (`conftest.py`, `mocker` vs `patch`, `asyncio` vs `anyio`)
- No placeholder tests (`assert True` is not a test)
- No mocking your own code to force a pass

The run-verify-fix loop has a **hard cap of 5 attempts**. On attempt 6, it stops and produces a structured handoff report — what failed, what was tried, what it needs from you — instead of continuing to guess.

---

### `self-verification`

The last gate before the human sees the work.

Six sections, checked in order before any `[TASK_DONE]`:

1. **Syntax and imports** — `py_compile` on every new file, import resolution, circular import check
2. **Interface correctness** — argument counts, return types, async/sync boundary
3. **Logic and edge cases** — None safety, collection safety, off-by-one, error path handling
4. **Code quality** — no dead code, no placeholders, no debug `print()`, no `breakpoint()`
5. **Security basics** — no hardcoded secrets, no `shell=True`, no raw string SQL
6. **Final checklist** — all boxes must be checked before handoff

If a check fails and the fix is out of scope, it surfaces the issue explicitly rather than silently leaving it.

---

### `debugging-loop`

Debugging is not guessing. This skill enforces a read-understand-fix discipline.

**Hard cap: 3 fix attempts per bug.**

Before touching any code:
- Read the full traceback, bottom to top
- Locate the origin — not where it was caught, where it was caused
- State the root cause in one specific sentence before writing any fix

Each attempt: one root cause → one targeted fix → one verification run.

On attempt 4, it stops immediately and produces a handoff report:
- Original error (full output)
- Current error (full output)
- What was tried, what happened, what the current hypothesis is
- What specific information or decision is needed from you

No 4th attempt. No silent retry. No `[TASK_DONE]` with an open bug.

---

## Why a hard cap on attempts?

Most AI coding tools loop until they either accidentally succeed or you give up. That's not discipline — that's slot machine behavior.

A hard cap forces an honest admission: *"I don't know how to fix this, and more attempts won't help. Here's everything I know. You decide."*

That handoff, done well, is more useful than 10 more guesses.

---

## Philosophy

These skills are built around one idea: **Claude Code should behave like a disciplined junior developer, not a confident guesser.**

A disciplined junior:
- Reads the codebase before writing
- Asks "does this already exist?" before creating something new
- Writes tests as part of the work, not after
- Says "I've tried 3 things and I'm stuck" instead of trying a 4th guess
- Does a self-review before handing anything off

That's what these skills enforce.

---

## Contributing

If you've built a skill that enforces a specific discipline — not just a style preference, but a behavioral rule with clear pass/fail conditions — PRs are welcome.

Good skills:
- Solve a real, recurring failure mode
- Have clear "do this, not that" rules
- Include a hard stop condition (don't loop forever)
- Were tested on a real project, not written speculatively

---

## License

MIT
