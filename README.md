# CS593 Homework 3: Fixing `find_undeclared_variables` in Jinja

This repository is Jinja 3.1.6 with a fix for
[pallets/jinja issue 2069](https://github.com/pallets/jinja/issues/2069):
`meta.find_undeclared_variables()` reported a variable as undeclared even
when every branch of an `if` / `elif` / `else` assigns it.

```python
from jinja2 import Environment, meta

src = (
    "{% if c %}{% set output = 1 %}{% else %}{% set output = 2 %}{% endif %}"
    "{{ output }}"
)
meta.find_undeclared_variables(Environment().parse(src))
# before: {'c', 'output'}    after: {'c'}
```

## Changes

| Commit | Description |
|---|---|
| `15206881` | Original Jinja 3.1.6 release (tag `3.1.6`) |
| `c7324af2` | Reproduction tests, which fail on 3.1.6 |
| `f38847e0` | The fix in `src/jinja2/idtracking.py`, plus a `CHANGES.rst` entry |
| `c5ea9a1f` | Additional tests for related scenarios |

To see the whole patch: `git diff 3.1.6 main`

- `src/jinja2/idtracking.py`
  - `Symbols.branch_update`: a variable that every branch assigns before reading is no longer marked as looked up from the context.
  - `FrameSymbolVisitor.visit_If`: each `elif` is analyzed as its own branch, and a missing `else` counts as a branch that assigns nothing.
- `tests/test_api.py` (`TestMeta`)
  - `test_find_undeclared_variables_set_in_all_if_branches`: the reproduction tests.
  - `test_find_undeclared_variables_if_branch_scenarios`: related cases, including ones that must still be reported.
- `tests/test_core_tags.py` (`TestIfCondition`)
  - `test_set_in_branches_context_fallback`: checks that rendering still falls back to the context where it should.

## Setup

Requires Python 3.8+ and git.

```bash
git clone https://github.com/ObviAvi/CS593-HW3.git
cd CS593-HW3
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements/tests.txt -e .
```

## Reproduce the original failure

Check out the commit that adds the reproduction tests, before the fix:

```bash
git checkout c7324af2
python -m pytest tests/test_api.py -k TestMeta -v
# 3 failed: assert {'control', 'output'} == {'control'}
git checkout main
```

## Run the tests

```bash
# reproduction + additional tests, plus the existing regression test for the related issue 1253
python -m pytest tests/test_api.py::TestMeta tests/test_core_tags.py::TestIfCondition \
  tests/test_regression.py::test_load_parameter_when_set_in_all_if_branches -v

# full regression suite (929 passed)
python -m pytest
```

## Known limitation

If a branch reads the variable inside a nested scope, such as a `for` loop, before assigning it, the loop now sees an empty value instead of the context value. This matches how Jinja already treats the same pattern outside an `if`, but it is a change from 3.1.6.

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
