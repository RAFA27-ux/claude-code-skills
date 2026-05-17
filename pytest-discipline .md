---
name: pytest-discipline
description: Use this skill whenever Claude Code writes, runs, or fixes Python tests. Triggers include: "write tests for this", "fix the failing test", "add test coverage", "run the tests", or any task where pytest is involved. Also triggers automatically after any new function or class is written — tests are not optional. Enforces a strict run-verify-fix loop with a hard cap of 5 attempts before human handoff.
---

# pytest-discipline

Tests are not a finishing touch. They are part of the task. Code is not done until tests are written and passing. This skill governs the full test lifecycle: writing, running, fixing, and knowing when to stop.

---

## Core rule

**Never output `[TASK_DONE]` while any test is failing.**

No exceptions. Not "it's probably fine", not "the logic is correct even if the test fails", not "this is a minor issue". Failing test = unfinished task.

---

## Phase 1: Writing tests

### 1.1 One test file per source file

Mirror the source path under `tests/`:

- `src/myapp/proxy/interceptor.py` → `tests/unit/proxy/test_interceptor.py`
- `src/myapp/db/logger.py` → `tests/unit/db/test_logger.py`

If the `tests/` directory structure does not exist yet, create it with `__init__.py` files as needed.

### 1.2 What to test — minimum bar

For every new function or method, write at minimum:

- **Happy path** — correct input produces correct output
- **One error path** — bad input, missing resource, or forced exception triggers the expected behavior (raises the right exception, returns the right fallback, logs the right message)
- **One edge case** — empty string, empty list, zero, `None` for optional params, or the boundary condition most likely to break

More coverage is better. This is the floor, not the ceiling.

### 1.3 Match project test conventions

Before writing, check:

- Is `conftest.py` present? Read it. Use its fixtures — do not duplicate them.
- Does the project use `pytest-mock` (`mocker`) or `unittest.mock` (`patch`)? Use the same.
- Does the project use `pytest.mark.asyncio` or `anyio`? Use the same for async tests.
- Does the project parametrize with `@pytest.mark.parametrize`? Use it for similar cases.

### 1.4 Async tests

If testing an async function:

```python
import pytest

@pytest.mark.asyncio
async def test_my_async_function():
    result = await my_async_function()
    assert result == expected
```

Never call `asyncio.run()` inside a test. Never test async code with a sync test function.

### 1.5 No mocking what you own

Only mock:
- External I/O: network calls, file system, database, subprocesses, time (`datetime.now`)
- Third-party library calls you do not control

Never mock your own functions just to make a test pass. If your code is hard to test without mocking your own internals, the code needs to be refactored, not the test.

### 1.6 No placeholder tests

Do not write:

```python
def test_my_function():
    pass  # TODO

def test_my_function():
    assert True
```

Every test must make a real assertion about real behavior.

---

## Phase 2: The run-verify-fix loop

After writing tests, run them. Then follow this loop strictly.

### Loop structure

```
attempt = 1

while attempt <= 5:
    run: pytest <test_file> -v
    
    if all tests pass:
        break → proceed to [TASK_DONE]
    
    read the full failure output
    identify root cause (see Phase 3)
    apply fix
    attempt += 1

if attempt > 5:
    → STOP. Human handoff. (see Phase 4)
```

### What counts as an attempt

One attempt = one full `pytest` run. Fixing multiple failures in a single run before re-running counts as one attempt. Do not re-run after every single line change — batch your fixes, then run.

### How to run

Always run with `-v` for readable output:

```bash
pytest tests/unit/path/to/test_file.py -v
```

To run a single failing test by name:

```bash
pytest tests/unit/path/to/test_file.py::test_function_name -v
```

To see full tracebacks without truncation:

```bash
pytest tests/unit/path/to/test_file.py -v --tb=long
```

---

## Phase 3: Reading and fixing failures

### 3.1 Read the full traceback

Never guess the cause from the first line. Read the full traceback from bottom to top:

- Bottom: the actual assertion error or exception
- Middle: the call chain that led there
- Top: the test function that triggered it

### 3.2 Classify the failure before fixing

| Failure type | What it means | Fix |
|---|---|---|
| `ImportError` / `ModuleNotFoundError` | Wrong import path or missing file | Fix the import path or create the missing module |
| `AttributeError` | Wrong method/attribute name, or object is wrong type | Check the actual object returned |
| `AssertionError` | Output doesn't match expectation | Either the implementation is wrong or the test expectation is wrong — reason carefully |
| `TypeError` | Wrong argument count or type | Check function signature vs how it's called in the test |
| Async error (`RuntimeError: no running event loop`) | Async function called without async test | Add `@pytest.mark.asyncio` |
| Mock not called / called with wrong args | Mock target path is wrong | Check the full dotted path of what you're patching |

### 3.3 Fix the right thing

When a test fails, determine: is the implementation wrong, or is the test wrong?

- If the implementation doesn't match the spec → fix the implementation
- If the test expectation was wrong (misunderstood the behavior) → fix the test and document why
- If both are wrong → fix both

Never fix a test just to make it pass if the underlying implementation is still broken.

### 3.4 One fix at a time

Do not make speculative multi-part fixes in one attempt. Identify the most likely root cause, fix that, then run again. Scattershot fixes waste attempts.

---

## Phase 4: Human handoff (attempt 5 exhausted)

If 5 attempts pass and tests are still failing, stop immediately.

Do not attempt a 6th run. Do not try a different approach silently. Do not output `[TASK_DONE]`.

Output this handoff report:

```
[HANDOFF REQUIRED]

Attempts made: 5/5
Tests still failing: <list the failing test names>

Last failure output:
<paste the full pytest output from the last run>

Root cause analysis:
<what you believe the underlying problem is>

What was tried:
1. <fix attempted in attempt 1>
2. <fix attempted in attempt 2>
...

What is needed to proceed:
<what information, decision, or change is required from the human>
```

Then stop. Wait for human input before doing anything else.

---

## Hard rules

1. **Max 5 attempts.** Attempt 6 does not exist. Ever.
2. **No `[TASK_DONE]` with failing tests.** Not even one.
3. **No placeholder tests.** `assert True` is not a test.
4. **No mocking your own code** to force a pass.
5. **No skipping tests** with `@pytest.mark.skip` without explicit human instruction and a comment explaining why.
6. **No `xfail`** without explicit human instruction.
7. **Run tests yourself.** Do not ask the human to run them and report back. If you wrote it, you run it.
