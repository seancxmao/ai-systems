# Pytest Essentials

> Master the 90% of pytest you use every day in modern Python projects.

## Why Pytest?

Pytest is the de facto standard testing framework for modern Python development. Compared with `unittest`, it offers a simpler assertion style, a powerful fixture system, parameterized testing, and an extensive plugin ecosystem.

This guide focuses on the features you will use in almost every project, especially AI infrastructure and LLM serving systems.

## Mental Model

Unlike application frameworks, pytest is not something you "build on". Think of it as a toolbox.

When you need to...

| I want to...                            | Use...                 |
| --------------------------------------- | ---------------------- |
| Write a test                            | `assert`               |
| Reuse setup code                        | `fixture`              |
| Run the same test with different inputs | `parametrize`          |
| Test async code                         | `pytest-asyncio`       |
| Verify an exception                     | `pytest.raises`        |
| Replace external dependencies           | `mock` / `monkeypatch` |
| Create temporary files                  | `tmp_path`             |
| Skip tests                              | `skip` / `skipif`      |
| Organize tests                          | `markers`              |

Don't memorize everything. Learn which tool solves which problem.

## Installation

```bash
uv add --dev pytest
```

For async projects:

```bash
uv add --dev pytest pytest-asyncio
```

Useful plugins:

```bash
uv add --dev pytest-cov
uv add --dev pytest-mock
```

## Project Structure

```
project/

src/
    llm_serving/

tests/
    conftest.py
    test_scheduler.py
    test_engine.py
    test_api.py
```

Pytest automatically discovers

```
test_*.py
*_test.py
```

## Writing Tests

The basic unit is simply a function.

```python
def add(a, b):
    return a + b


def test_add():
    assert add(1, 2) == 3
```

No `TestCase`.

No `assertEqual()`.

Just use Python's `assert`.

## Running Tests

Run everything:

```bash
pytest
```

Run one file:

```bash
pytest tests/test_scheduler.py
```

Run one test:

```bash
pytest tests/test_scheduler.py::test_add_request
```

Run tests matching a keyword:

```bash
pytest -k scheduler
```

Frequently used options:

```bash
pytest -q      # quiet
pytest -vv     # verbose
pytest -x      # stop after first failure
```

## Fixtures ★★★★★

Use fixtures to create reusable test resources.

```python
import pytest

@pytest.fixture
def scheduler():
    return Scheduler()


def test_add_request(scheduler):
    ...
```

Instead of repeatedly writing setup code, pytest injects the fixture automatically.

Typical uses:

* Scheduler
* Engine
* Tokenizer
* Fake Request
* Temporary configuration

Prefer fixtures over manually constructing objects in every test.

## Parameterized Tests ★★★★★

Test multiple inputs with one function.

```python
@pytest.mark.parametrize(
    "batch_size",
    [1, 4, 8, 16],
)
def test_batch(batch_size):
    ...
```

Very common for

* scheduler policies
* sampling parameters
* request lengths
* timeout values

## Testing Exceptions

```python
import pytest

def test_invalid_request():
    with pytest.raises(ValueError):
        parse_request(None)
```

Use `pytest.raises()` instead of try/except.

## Async Tests ★★★★★

Install

```
pytest-asyncio
```

Example:

```python
import pytest

@pytest.mark.asyncio
async def test_stream():
    result = await engine.generate(...)
    assert result.finished
```

This is essential for

* FastAPI
* asyncio
* streaming
* queues
* locks
* events

## Mocking

Use mocks when the real dependency is

* slow
* expensive
* unavailable

Example:

```python
from unittest.mock import Mock

tokenizer = Mock()
tokenizer.encode.return_value = [1, 2, 3]
```

Or use `pytest-mock`:

```python
def test_x(mocker):
    tokenizer = mocker.Mock()
```

Typical targets:

* HTTP requests
* LLM Engine
* Tokenizer
* GPU interfaces
* External services

## Temporary Files

Need a temporary directory?

```python
def test_config(tmp_path):
    config = tmp_path / "config.json"
    config.write_text("{}")
```

Everything is cleaned up automatically.

## Marks

Skip a test:

```python
@pytest.mark.skip
```

Conditional skip:

```python
@pytest.mark.skipif(
    sys.platform == "win32",
    reason="Linux only",
)
```

Custom markers:

```python
@pytest.mark.gpu

@pytest.mark.integration

@pytest.mark.slow
```

Run only GPU tests:

```bash
pytest -m gpu
```

Exclude slow tests:

```bash
pytest -m "not slow"
```

## Shared Fixtures

Create shared fixtures in

```
tests/conftest.py
```

Example:

```python
@pytest.fixture
def fake_request():
    ...
```

Every test can use it.

## Coverage

Install

```bash
uv add --dev pytest-cov
```

Run

```bash
pytest --cov=src
```

Generate HTML report

```bash
pytest --cov=src --cov-report=html
```

## Testing Strategy

For an LLM serving project:

```
tests/

    test_scheduler.py
    test_engine.py
    test_sampling.py
    test_tokenizer.py
    test_api.py
```

Recommended split:

```
Unit Tests
    Scheduler
    Sampling
    Tokenizer

Integration Tests
    Engine
    FastAPI

End-to-End Tests
    Full request lifecycle
```

Start with unit tests. Add integration tests where component interactions matter.

## Best Practices

* One test should verify one behavior.
* Prefer fixtures over duplicated setup code.
* Keep tests independent.
* Use parameterized tests instead of copy-paste.
* Mock external dependencies.
* Name tests after behavior, not implementation.
* Test public interfaces rather than private methods.

## Cheat Sheet

| Task               | Tool                            |
| ------------------ | ------------------------------- |
| Write a test       | `assert`                        |
| Reuse setup        | `fixture`                       |
| Multiple inputs    | `parametrize`                   |
| Async code         | `pytest.mark.asyncio`           |
| Expected exception | `pytest.raises`                 |
| Mock dependency    | `pytest-mock` / `unittest.mock` |
| Temporary files    | `tmp_path`                      |
| Skip tests         | `skip` / `skipif`               |
| Organize tests     | `markers`                       |
| Shared setup       | `conftest.py`                   |
| Coverage           | `pytest-cov`                    |
