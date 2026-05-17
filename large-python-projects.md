---
name: large-python-projects
description: Use this skill when working inside an existing Python codebase with multiple modules, packages, or layers — especially when the task involves adding features, fixing bugs, or refactoring without breaking existing patterns. Triggers include: "add a new feature to my project", "fix this in my codebase", "extend this module", "refactor this", or any task where Claude Code must write code that fits into an already-established Python project structure. Do NOT use for greenfield scripts, one-off utilities, or self-contained snippets with no surrounding codebase.
---

# Large Python Projects

This skill governs how Claude Code approaches tasks inside an existing, multi-file Python codebase. The core principle is: **understand before you write**. Every action must fit the project's existing patterns, conventions, and architecture. Never invent when you can inherit.

---

## Phase 1: Mandatory orientation (before writing any code)

Before touching a single file, build a mental model of the project. This is not optional.

### 1.1 Read the project definition files first

In order:

1. `pyproject.toml` or `setup.py` / `setup.cfg` — understand declared dependencies, Python version target, and project name
2. `requirements*.txt` if present — note any pinned versions
3. `README.md` or `ARCHITECTURE.md` if present — read the stated intent

Do not skip this step. These files tell you what the project is and what it depends on.

### 1.2 Map the package structure

Run or reason through the directory tree. Identify:

- Where the main application entry point is (`__main__.py`, `main.py`, `app.py`, `cli.py`, etc.)
- Where core abstractions live (often `core/`, `base/`, `models/`, `domain/`)
- Where utilities and helpers live (`utils/`, `helpers/`, `common/`)
- Where tests live and how they are structured (`tests/unit/`, `tests/integration/`, etc.)
- Whether there is a `conftest.py` and what fixtures it defines

### 1.3 Identify the base abstractions

Before writing any class, look for:

- Abstract base classes (`ABC`, `Protocol`, or informal base classes in `core/` or `base/`)
- Shared mixins or decorators used across the codebase
- Common data models (Pydantic `BaseModel` subclasses, dataclasses, named tuples)

If the project uses a `BaseHandler`, `BaseService`, `BaseModel`, or similar — your new code must inherit from or compose with it. Never create parallel abstractions.

### 1.4 Read the files directly relevant to the task

Identify the 2–5 files most relevant to the task. Read them fully before writing anything. Specifically look for:

- How imports are structured (relative vs absolute, `__init__.py` exports)
- How errors are raised and propagated
- How logging is done (stdlib `logging`, `loguru`, `structlog`, etc.)
- How configuration is passed (constructor injection, global config object, env vars)
- How async is used, if at all (`asyncio`, `aiohttp`, `anyio`) — and whether the task requires staying in the same async context

---

## Phase 2: Pattern extraction (match, don't invent)

Once orientation is complete, extract the patterns that govern how code is written in this project. Your new code must be indistinguishable from the existing code in style and structure.

### 2.1 Type annotation style

Check: does the codebase use type hints? If yes:

- All new functions must have fully annotated signatures: parameters and return types
- Use the same import style (`from typing import Optional` vs `str | None`, depending on Python version)
- If `TypeVar`, `Generic`, or `Protocol` are used — follow the same pattern

If the codebase does not use type hints, do not add them unless asked.

### 2.2 Docstring convention

Check what docstring format is used:

- Google style (`Args:`, `Returns:`, `Raises:`)
- NumPy style
- reStructuredText (`:param x:`, `:returns:`)
- Plain prose

Match it exactly. If no docstrings exist on similar functions, keep new docstrings minimal — do not introduce a new convention the project doesn't use.

Every public function and class in new code must have a docstring. Private helpers (`_name`) may omit if project convention does.

### 2.3 Error handling convention

Look at how the project raises and catches exceptions:

- Does it use custom exception classes? If so, inherit from the project's base exception, not Python built-ins directly
- Does it log before raising, or let callers handle logging?
- Does it use `try/except/else/finally` or context managers (`contextlib.suppress`, etc.)?

All I/O in new code (file reads, network calls, DB queries, subprocess calls) must be wrapped in `try/except`. No silent failures. Exceptions must either be re-raised with context (`raise NewError(...) from e`) or logged explicitly before being swallowed.

### 2.4 Async conventions

If the project uses `async/await`:

- New async functions must be declared `async def`
- Never call `asyncio.run()` inside a function that is already in an async context
- Never mix sync blocking I/O (e.g., `open()`, `requests.get()`) inside async functions — use the async equivalent (`aiofiles`, `aiohttp`, etc.)
- Check whether the project uses `asyncio.gather`, `asyncio.create_task`, or `anyio.create_task_group` — use the same pattern

### 2.5 Logging convention

Check how logging is initialized and used:

- Stdlib `logging`: look for `logger = logging.getLogger(__name__)` at module level — do the same
- `loguru`: look for `from loguru import logger` — do the same
- Do not introduce a new logging library
- Do not use `print()` for anything other than deliberate user-facing CLI output

---

## Phase 3: Duplicate check (before creating anything new)

Before writing a new function, class, or module, answer these questions:

1. **Does this logic already exist somewhere?** Search for function names, method names, or similar docstrings in `utils/`, `helpers/`, `core/`. If it exists, import and use it — do not rewrite it.
2. **Does a similar abstraction already exist?** If there is a `DatabaseManager`, do not create a `DBHelper`. Extend or use the existing one.
3. **Will this new file conflict with existing imports?** Check `__init__.py` files to understand what is already exported. Do not create a module with the same name as an existing one in the same package.

If in doubt, prefer extending an existing file over creating a new one.

---

## Phase 4: Writing code

Only after completing Phases 1–3, write the code.

### 4.1 Scope discipline

Write only what the task requires. Do not:

- Refactor surrounding code unless explicitly asked
- Add new dependencies unless the task cannot be done without them
- Change function signatures of existing public APIs
- Rename existing variables, classes, or modules

If you notice a problem in surrounding code, note it in a comment (`# NOTE: ...`) but do not fix it unless asked.

### 4.2 File placement

Place new files where they belong based on the project structure:

- New data model → `models/` or alongside existing models
- New utility function → `utils/` or an existing relevant utility module
- New service/handler → wherever existing services/handlers live
- New test → mirror the source file path under `tests/`

Do not create new top-level directories without explicit instruction.

### 4.3 Import hygiene

- Use the same import style as the rest of the file (relative vs absolute)
- Do not add wildcard imports (`from module import *`)
- Do not import inside functions unless the project does so intentionally (e.g., to avoid circular imports)
- Group imports in the standard order: stdlib → third-party → local, with a blank line between groups

### 4.4 No placeholders

Do not write:

```python
# TODO: implement this
pass

def process():
    ...  # implement later
```

Every function must be fully implemented. If a piece of logic is genuinely out of scope, say so explicitly in a comment and implement a safe no-op or raise `NotImplementedError` with a clear message — never silent `pass`.

---

## Phase 5: Testing

Every new function or class must have corresponding tests. No exceptions.

### 5.1 Mirror the test structure

If source is at `src/ghosttrace/proxy/interceptor.py`, tests go at `tests/unit/proxy/test_interceptor.py`. Mirror the directory structure exactly.

### 5.2 Match existing test style

Check `conftest.py` and existing test files:

- Do they use `pytest` fixtures? Use the same fixtures.
- Do they use `unittest.mock.patch` or `pytest-mock`'s `mocker`? Use the same.
- Do they use `pytest.mark.asyncio` or `anyio`? Use the same.
- Do they parametrize with `@pytest.mark.parametrize`? Follow the same pattern for similar cases.

### 5.3 What to test

At minimum, write tests for:

- The happy path (correct input → correct output)
- At least one error path (bad input, missing resource, network failure) — verify the exception type or error message
- Edge cases that are obvious from the function signature (empty list, zero, `None` if `Optional`)

### 5.4 Run the tests

After writing tests, run them:

```bash
pytest tests/unit/path/to/test_file.py -v
```

If tests fail, fix the implementation or the test — do not skip or mark as `xfail` without explanation. Do not proceed to `[TASK_DONE]` while tests are failing.

---

## Phase 6: Self-review before completion

Before declaring the task done, review your own output against this checklist:

- [ ] All new functions have type annotations (if the project uses them)
- [ ] All new public functions and classes have docstrings
- [ ] All I/O operations are wrapped in `try/except`
- [ ] No `print()` used for non-CLI output (use logging)
- [ ] No duplicate logic introduced — existing utilities were checked first
- [ ] No new top-level directories created without instruction
- [ ] No existing public API signatures changed
- [ ] No placeholders, `...`, or silent `pass` in production code
- [ ] Tests written and passing
- [ ] Imports are clean, grouped, and consistent with project style

Only after this checklist passes, output `[TASK_DONE]`.

---

## Hard rules (never violate)

These apply regardless of task complexity or time pressure:

1. **Never use `shell=True`** in `subprocess` calls. Use list form: `subprocess.run(["cmd", "arg"])`.
2. **Never log secrets, tokens, or API keys** — not in debug, not in error messages.
3. **Never use bare `except:`** — always catch a specific exception type or at minimum `except Exception as e`.
4. **Never swallow exceptions silently** — if you catch it, either re-raise or log it.
5. **Never write to files outside the project directory** without explicit user instruction.
6. **Never modify test fixtures in `conftest.py`** without explicit instruction — they affect the entire test suite.
