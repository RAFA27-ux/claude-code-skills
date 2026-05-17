---
name: self-verification
description: Use this skill after any code is written but before declaring it done. Triggers automatically after completing any implementation task — before outputting [TASK_DONE]. Also triggers when asked to "double check", "review your own code", "make sure this works", or "verify this". This skill makes Claude Code run a structured self-review pass so the human does not have to catch basic errors.
---

# self-verification

Before declaring any task done, run this verification pass. Every item is checked in order. Nothing is skipped because "it's probably fine."

The goal: the human should not discover a basic error that could have been caught before handoff.

---

## When to run this

Run immediately after finishing an implementation, before `[TASK_DONE]`. If any check fails, fix it before proceeding. This is not a formality — it is the last gate before the human sees the work.

---

## Verification checklist

Work through every section. For each item: check it, fix it if broken, mark it resolved. Do not move to the next section until the current one is clean.

---

### Section 1: Syntax and imports

**1.1 — Parse check**

For every new or modified Python file, run:

```bash
python -m py_compile path/to/file.py
```

If this fails, the file has a syntax error. Fix it before anything else.

**1.2 — Import resolution**

For every `import` statement in new code, verify:

- The module exists at the path being imported
- Relative imports (`from .module import X`) are correct relative to the file's package location
- Nothing is imported that is not used
- Nothing that is used is missing from imports

Run:

```bash
python -c "import path.to.your.module"
```

If this raises `ImportError` or `ModuleNotFoundError`, the import is broken. Fix it.

**1.3 — No circular imports**

If new code imports from module A, and module A imports from new code, there is a circular import. Check for this. Resolve by moving shared code to a third module that neither imports.

---

### Section 2: Interface correctness

**2.1 — Signature consistency**

For every function or method call in new code, verify:

- The number of arguments matches the function definition
- Keyword argument names match exactly (no typos)
- Required arguments are not missing
- Optional arguments have correct defaults

**2.2 — Return type consistency**

For every function that declares a return type:

- Every code path returns a value of that type
- No code path returns `None` when the return type does not include `Optional` or `None`
- No code path falls off the end of the function without returning

**2.3 — Async consistency**

For every `async def` function:

- Every call to it uses `await`
- It is only called from within another `async def` or a properly set up event loop
- No blocking I/O (`open()`, `requests.get()`, `time.sleep()`) is inside it

For every non-async function:

- It does not call `await` (this is a `SyntaxError`)
- It does not call an async function without `asyncio.run()` or similar

---

### Section 3: Logic and edge cases

**3.1 — None safety**

For every variable that could be `None`:

- There is a `None` check before it is used as an object (`.attribute`, `[index]`, function call)
- Or the type annotation reflects that `None` is impossible at that point

**3.2 — Collection safety**

For every list, dict, or set operation:

- Index access (`list[0]`) is guarded against empty collections
- Dict key access (`dict["key"]`) uses `.get()` or a prior `in` check unless the key is guaranteed to exist
- Iteration over a collection that could be `None` is guarded

**3.3 — Off-by-one and boundary conditions**

For every loop, slice, or range:

- The start and end bounds are correct
- The loop body handles the empty case (zero iterations)
- The loop body handles the single-element case

**3.4 — Error paths are handled**

For every `try/except` block:

- The exception type caught is specific (not bare `except:`)
- The except block does something meaningful: logs, re-raises, or returns a safe value
- There is no silent swallowing: `except Exception: pass` is a bug

---

### Section 4: Code quality

**4.1 — No dead code**

Remove:

- Functions defined but never called (unless they are part of a public API)
- Variables assigned but never used
- Imports that are unused
- Commented-out code blocks

**4.2 — No placeholders**

Search new code for:

- `pass` inside a function body (acceptable only in abstract methods, not implementations)
- `...` as a function body
- `# TODO`, `# FIXME`, `# HACK` — if present, either fix now or surface to human
- `raise NotImplementedError` in non-abstract code

**4.3 — No debug artifacts**

Remove before handoff:

- `print()` statements used for debugging (not intentional CLI output)
- `import pdb; pdb.set_trace()`
- `breakpoint()`
- Temporary hardcoded values that were used for testing

**4.4 — Logging not printing**

Non-CLI output must use the project's logging system. If a line outputs information about program state and is not intentional user-facing output, it must be `logger.debug(...)` or similar, not `print(...)`.

---

### Section 5: Security basics

**5.1 — No secrets in code or logs**

Search new code for:

- Hardcoded API keys, tokens, passwords, or secrets
- Log statements that include `api_key`, `token`, `password`, `secret`, `auth` in variable names

If found: move to environment variable or config, and remove from logs.

**5.2 — No `shell=True`**

Search for `subprocess` calls. If any use `shell=True`, replace with list form:

```python
# Wrong
subprocess.run("ls -la /tmp", shell=True)

# Correct
subprocess.run(["ls", "-la", "/tmp"])
```

**5.3 — No raw string SQL**

If the project uses SQLite or any database, search for string-formatted SQL:

```python
# Wrong — SQL injection risk
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Correct — parameterized
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
```

If found, fix it.

---

### Section 6: Final confirmation

After all sections pass, confirm:

- [ ] `py_compile` passes on all new/modified files
- [ ] All imports resolve
- [ ] No broken function signatures
- [ ] Async/sync boundary is clean
- [ ] No `None` or empty collection crashes waiting to happen
- [ ] All error paths handled
- [ ] No dead code, placeholders, or debug artifacts
- [ ] No secrets, no `shell=True`, no raw SQL

Only when every box is checked: output `[TASK_DONE]`.

If any box cannot be checked and the fix is beyond the current task scope, surface it explicitly to the human before outputting `[TASK_DONE]`. Do not silently leave known issues.
