---
name: debugging-loop
description: Use this skill when code raises an error, a test fails unexpectedly, or behavior does not match expectation. Triggers include: any runtime exception, unexpected output, "this isn't working", "I'm getting an error", "fix this bug", or any situation where something that should work does not. Enforces a structured read-understand-fix loop with a hard cap of 3 attempts before human handoff.
---

# debugging-loop

Debugging is not guessing. It is reading, understanding, and acting on evidence. This skill enforces a disciplined loop: read the error fully, identify the root cause, apply one targeted fix, verify. Repeat up to 3 times. If not resolved, hand off to the human with full context.

---

## Core rule

**Maximum 3 fix attempts per bug. On attempt 4, stop and hand off.**

Attempting a fourth fix without human input means the root cause is not understood. More attempts will not help — more information is needed.

---

## Phase 1: Read before touching anything

When an error occurs, do not modify any code until Phase 1 is complete.

### 1.1 Read the full error output

Read the complete error — not just the last line. The last line names the exception. The traceback shows where it came from and why.

Read the traceback from **bottom to top**:

- **Bottom**: the exception type and message — what broke
- **Middle**: the call chain — how execution got there
- **Top**: the entry point — where the user's action triggered the chain

### 1.2 Locate the origin

Identify the exact file, line number, and expression where the failure originated. Not where it was caught — where it was caused.

Common misdirections:
- The error is raised in a library, but the cause is the argument you passed to it
- The error is on line N, but the bad value was assigned on line N-30
- The error is in a test, but the bug is in the implementation being tested

### 1.3 State the root cause before acting

Before writing any fix, write one sentence:

> "The root cause is: ___"

If you cannot complete that sentence with specificity, you do not understand the bug yet. Do not proceed to Phase 2. Re-read the traceback, read the relevant source code, and try again.

Acceptable:
> "The root cause is: `config` is `None` at line 47 because `load_config()` returns `None` when the file doesn't exist, and the caller doesn't check for it."

Not acceptable:
> "The root cause is: something is wrong with the config loading."

---

## Phase 2: The fix loop

```
attempt = 1

while attempt <= 3:
    state the root cause (one sentence)
    apply one targeted fix
    run the verification (test, script, or command that reproduces the error)
    
    if error is gone and behavior is correct:
        run self-verification checklist
        break → proceed to [TASK_DONE]
    
    read the new error output fully (it may be a different error now)
    attempt += 1

if attempt > 3:
    → STOP. Human handoff. (see Phase 3)
```

### What counts as one attempt

One attempt = one targeted fix + one verification run. If the fix involves changing three related lines that are all part of the same root cause, that is still one attempt. If after running you discover a second unrelated bug, that starts a new debugging session — do not chain them into the same attempt count.

### Targeted fixes only

Each fix must address the specific root cause identified. Do not:

- Change multiple unrelated things in one attempt hoping something works
- Rewrite the entire function when one line is wrong
- Add broad `try/except` to silence an error without fixing its cause

One root cause → one fix. Then verify.

### If the error changes between attempts

A new error after a fix is progress — the previous issue was resolved. But:

- Read the new error fully before acting
- State the new root cause before fixing
- This counts as a new attempt

Do not assume the new error is related to the old one without evidence.

---

## Phase 3: Human handoff (attempt 3 exhausted)

If 3 attempts pass and the bug is still present, stop immediately.

Do not attempt a 4th fix. Do not try a completely different approach without telling the human. Do not output `[TASK_DONE]`.

Output this handoff report:

```
[HANDOFF REQUIRED]

Attempts made: 3/3
Bug status: unresolved

Original error:
<paste the first error output in full>

Current error (after 3 attempts):
<paste the current error output in full>

What was tried:
1. Root cause identified: <your diagnosis>
   Fix applied: <what you changed>
   Result: <what happened>

2. Root cause identified: <your diagnosis>
   Fix applied: <what you changed>
   Result: <what happened>

3. Root cause identified: <your diagnosis>
   Fix applied: <what you changed>
   Result: <what happened>

Current hypothesis:
<your best current understanding of why the bug persists>

What is needed to proceed:
<specific information, decision, or context required from the human>
```

Then stop. Do not modify any more code. Wait for human input.

---

## Common bug patterns and their fixes

Use this as a reference during Phase 1 root cause analysis.

### `AttributeError: 'NoneType' object has no attribute 'X'`

Something that should return an object returned `None`. Find where the variable was assigned. Find why the assignment returned `None`. Fix the source, not the symptom — do not just add `if obj is not None`.

### `ImportError` / `ModuleNotFoundError`

The module path is wrong, the file doesn't exist, or the package isn't installed. Check: does the file exist at that path? Is it inside a package (has `__init__.py`)? Is the import relative or absolute — should it be the other?

### `TypeError: X() takes N positional arguments but M were given`

The call site and the function definition disagree on argument count. One of them changed and the other didn't. Find both, reconcile them.

### `KeyError: 'X'`

A dict is being accessed with a key that doesn't exist. Either use `.get('X')` with a default, or trace back to where the dict is built and fix why the key is missing.

### `asyncio.InvalidStateError` / `RuntimeError: no running event loop`

Async code is being called incorrectly. Either an async function is being called without `await`, or `asyncio.run()` is being called inside an already-running event loop. Find the async/sync boundary and fix it.

### Test passes locally, fails in CI (or vice versa)

Environment difference. Likely causes: different Python version, missing environment variable, file path assumptions, ordering dependency between tests. Run with `pytest -p no:randomly` to rule out ordering issues. Check env vars explicitly.

### `RecursionError`

A function is calling itself or triggering a cycle. Find the cycle. Add a base case or break the cycle.

---

## Hard rules

1. **Max 3 attempts.** Attempt 4 does not exist. Ever.
2. **State root cause before fixing.** No fix without a hypothesis.
3. **One fix per attempt.** No scattershot changes.
4. **No silencing errors with bare `except`.** That is not a fix.
5. **No `[TASK_DONE]` while the bug is present.**
6. **Handoff report is mandatory** at attempt 3 exhaustion. Do not skip it.
