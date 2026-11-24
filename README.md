# CS593 HW3: Fixing Jinja issue #2069 with a coding agent

This repository is a copy of [pallets/jinja](https://github.com/pallets/jinja)
at version **3.1.6**, with a fix for
[issue #2069](https://github.com/pallets/jinja/issues/2069):
`meta.find_undeclared_variables()` reported a variable as undeclared even when
every branch of an `{% if %}` block assigns it.

```python
from jinja2 import Environment, meta

ast = Environment().parse(
    "{% if c %}{% set out = 1 %}{% else %}{% set out = 2 %}{% endif %}{{ out }}"
)
meta.find_undeclared_variables(ast)  # 3.1.6: {'c', 'out'}   fixed: {'c'}
```

The original Jinja README follows the HW3 sections below.

## Commits

All work is on the `fix-2069` branch (the default branch), on top of the
unmodified `3.1.6` tag. Run `git diff 3.1.6 fix-2069` to see every change.

| Commit | Description |
|---|---|
| `15206881` | `release version 3.1.6`: the unmodified upstream starting point (tag `3.1.6`) |
| `b8bea6af` | Ignore a local report directory (`.gitignore` only) |
| `287b0df7` | **Reproduction tests only.** These fail on the unmodified 3.1.6 source |
| `5990c929` | **The fix** (`src/jinja2/idtracking.py`) plus a `CHANGES.rst` entry |
| `6c76da4c` | Additional validation tests (metadata, rendering, symbol tracking) |

## Setup

These steps were tested with Python 3.11 on Windows. Any Python version that
Jinja 3.1.6 supports should work.

```sh
git clone https://github.com/farbs03/CS593-jinja-bugfix.git
cd CS593-jinja-bugfix
python -m venv .venv
# Windows:      .venv\Scripts\activate
# macOS/Linux:  source .venv/bin/activate
pip install -e . -r requirements/tests.txt
```

## Reproduce the original failure

Check out the commit that adds the reproduction tests but not the fix:

```sh
git checkout 287b0df7
pytest tests/test_api.py::TestMeta -v
# 4 failed, 3 passed, e.g.  AssertionError: assert {'c', 'output'} == {'c'}
git checkout fix-2069
```

To run *all* of the new tests against the original 3.1.6 implementation, swap
in the old file temporarily:

```sh
git checkout 3.1.6 -- src/jinja2/idtracking.py
pytest tests/test_api.py::TestMeta tests/test_core_tags.py::TestIfCondition tests/test_idtracking.py
# 11 failed, 33 passed
git checkout HEAD -- src/jinja2/idtracking.py
```

## Run the tests with the fix

```sh
# reproduction tests and additional tests
pytest tests/test_api.py::TestMeta tests/test_core_tags.py::TestIfCondition tests/test_idtracking.py -v
# 44 passed

# relevant regression tests (includes the #1253 test in test_regression.py)
pytest tests/test_api.py tests/test_idtracking.py tests/test_core_tags.py tests/test_regression.py tests/test_compile.py tests/test_async.py
# 267 passed

# full suite (the unmodified 3.1.6 suite has 909 tests, all passing)
pytest
# 938 passed
```

## What was wrong and what changed

`find_undeclared_variables` reports every name whose symbol-table load
instruction is `resolve`, meaning "look it up in the render context". For
assignments inside an `{% if %}`, that decision is made in
`Symbols.branch_update` in `src/jinja2/idtracking.py`.

- Jinja 3.1.5 fixed #1253 by making `branch_update` resolve **every** name
  assigned in any branch. That was correct for rendering, but it made every
  conditional assignment look undeclared.
- The older 3.1.4 approach (skip names assigned in all branches) couldn't
  simply be restored. `visit_If` put all `elif`s into a single symbol table and
  always passed an empty `elif` table, so "assigned in every branch" was never
  computed correctly. Restoring it brings back #1253 and two rendering bugs
  with `elif` chains.

The fix, about 20 lines in `src/jinja2/idtracking.py`:

1. `FrameSymbolVisitor.visit_If` analyzes the `if` body, **each** `elif`
   (including its test), and the `else` (empty if absent) as separate branches.
2. `Symbols.branch_update` keeps a name `undefined` (local, not undeclared)
   only when **every** branch assigns it **without reading it first**.
   Otherwise it resolves or aliases the name exactly as before.

New tests:

- `tests/test_api.py::TestMeta`: 4 reproduction tests plus 16 parametrized
  scenarios. These cover names assigned in all, some, or none of the branches;
  `elif` chains; a missing `else`; nested `if`s; reads before the `if` or before
  the assignment; `elif` tests; tuple and block `set`; `import`; and `for` and
  `macro` scopes.
- `tests/test_core_tags.py::TestIfCondition`: 7 rendering tests that check the
  context fallback is kept where it's needed.
- `tests/test_idtracking.py`: 2 symbol-table tests.

---

# Jinja

Jinja is a fast, expressive, extensible templating engine. Special
placeholders in the template allow writing code similar to Python
syntax. Then the template is passed data to render the final document.

It includes:

-   Template inheritance and inclusion.
-   Define and import macros within templates.
-   HTML templates can use autoescaping to prevent XSS from untrusted
    user input.
-   A sandboxed environment can safely render untrusted templates.
-   AsyncIO support for generating templates and calling async
    functions.
-   I18N support with Babel.
-   Templates are compiled to optimized Python code just-in-time and
    cached, or can be compiled ahead-of-time.
-   Exceptions point to the correct line in templates to make debugging
    easier.
-   Extensible filters, tests, functions, and even syntax.

Jinja's philosophy is that while application logic belongs in Python if
possible, it shouldn't make the template designer's job difficult by
restricting functionality too much.


## In A Nutshell

```jinja
{% extends "base.html" %}
{% block title %}Members{% endblock %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href="{{ user.url }}">{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}
```

## Donate

The Pallets organization develops and supports Jinja and other popular
packages. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today][].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/
