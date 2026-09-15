---
title: Unit Testing Guide
date:  2026-09-14 22:00:"00 +0800
categories: [Knowledge, "Unit Testing"]
tags: ["unit testing", documentation, knowledge]
description: A guide on unit testing.
---
# What Is Unit Testing?

A **unit test** verifies a single, isolated piece of code — typically one function or method — against known inputs and expected outputs. It confirms that logic behaves correctly without relying on real external systems such as databases, APIs, files, or network services.

Unit tests differ from other forms of testing:

| Test Type                            | Scope                                | Depends on Real Systems? |
| ------------------------------------ | ------------------------------------ | ------------------------ |
| **Unit test**                        | Single function/logic branch         | No (uses mocks)          |
| **Integration test**                 | Multiple components working together | Sometimes                |
| **Smoke test** / **end-to-end test** | Full application or script run       | Yes                      |

# Why Unit Testing Matters

* **Speed** — Tests run in milliseconds since no network or disk I/O occurs.
* **Reliability** — Results don't depend on whether an external service happens to be online.
* **Safety** — Refactoring code is less risky when tests confirm behavior hasn't changed.
* **Cost** — No real API usage, database load, or rate limits are triggered during testing.
* **Precision** — Each logical branch (success, timeout, error, edge case) can be tested independently.
* **Early feedback** — Bugs are caught during development instead of in production.

# How Mocking Replaces Real Dependencies

Unit tests use a **mocking library** (in Python, the built-in `unittest.mock`) to replace real objects and functions with fake, in-memory substitutes. This applies to any external dependency, such as:

* An HTTP client making an **API call** (e.g., `requests.get`, `requests.post`).
* A **database driver** opening a connection (e.g., `connect()`, `cursor.execute()`).
* A **file system** operation (e.g., reading/writing a file).
* A **third-party SDK client** (e.g., a cloud provider or messaging service).

### Core Mocking Techniques

1. **Create a fake object** — Build an object that mimics the real one's shape (methods, attributes, return values) without doing real work.
2. **Patch the dependency** — Temporarily replace the real function/class with the fake object for the duration of a test, then restore it afterward.
3. **Simulate failures** — Configure the fake object to raise an exception instead of returning a value, to test error-handling logic (e.g., a timeout, a dropped connection, an invalid response).

### Key Rule: Patch Where the Code Looks It Up

Always patch the dependency as it's referenced **inside the module under test**, not where it was originally defined elsewhere. This ensures the function being tested actually receives the fake object instead of the real one.

### Example: Mocking an API Call

```python
from unittest.mock import Mock, patch
import my_module

def test_fetch_data_success():
    fake_response = Mock(status_code=200, json=lambda: {"value": 42})

    with patch.object(my_module.requests, "get", return_value=fake_response):
        result = my_module.fetch_data("https://api.example.com/data")

    assert result == {"value": 42}
```

### Example: Mocking a Database Connection

```python
from unittest.mock import Mock, patch
import my_module

def test_get_user_success():
    fake_connection = Mock()
    fake_connection.execute.return_value = [{"id": 1, "name": "Alice"}]

    with patch.object(my_module, "connect_to_db", return_value=fake_connection):
        result = my_module.get_user(1)

    assert result["name"] == "Alice"
```

### Example: Simulating a Failure

```python
from unittest.mock import patch
import requests
import my_module

def test_fetch_data_timeout():
    with patch.object(my_module.requests, "get", side_effect=requests.Timeout()):
        result = my_module.fetch_data("https://api.example.com/data")

    assert result["status"] == "timeout"
```

### What This Means in Practice

Because the real client or connection is swapped out, **no actual API request, database query, or network call ever occurs** during a unit test. The test only verifies the code's decision-making logic — how it handles success, errors, timeouts, and edge cases — using controlled, predictable fake data.

# Typical Test Cases to Cover for Any Function

Regardless of whether a function calls an API, a database, or performs pure computation, aim to cover:

1. **The happy path** — Normal input produces the expected output.
2. **Expected errors** — The function handles known failure modes gracefully (e.g., invalid input, a failed connection, an error response).
3. **Edge cases** — Empty input, boundary values, unexpected types, or empty responses.
4. **Unexpected/unhandled errors** — The function fails safely rather than crashing silently or corrupting data.

# Running Unit Tests (Python Example with `pytest`)

### 1. Install the Test Framework

```powershell
python -m pip install pytest pytest-cov
```

### 2. Run All Tests

```powershell
python -m pytest
```

### 3. Run a Specific Test File

```powershell
python -m pytest test_my_module.py
```

### 4. Run With Verbose Output

```powershell
python -m pytest -v
```

Verbose mode lists each test function name with a `PASSED` or `FAILED` result, useful for confirming exactly which cases succeeded.

# Reading the Test Results

### Passing Run

```text
test_my_module.py::test_fetch_data_success PASSED
test_my_module.py::test_fetch_data_timeout PASSED
============================== 2 passed in 0.15s ==============================
```

* **`PASSED`** — The assertions in that test held true.
* **Summary line** — Shows total passed/failed/skipped counts and total run time.

### Failing Run

```text
test_my_module.py::test_fetch_data_success FAILED

================================== FAILURES ===================================
_____________________________ test_fetch_data_success ________________________

    assert result == {"value": 42}
E   assert {"value": 0} == {"value": 42}

============================== 1 failed in 0.10s ===============================
```

* **`FAILED`** — At least one assertion did not match the expected value.
* The traceback shows the exact assertion that failed and the actual vs. expected values, pinpointing the broken logic.

# Test Coverage

**Test coverage** is a percentage metric indicating how much of your source code executed during testing (measured in lines, statements, or branches).

### What It Does and Doesn't Tell You

* **Does show:** Which lines of code ran during the test suite.
* **Does not show:** Whether the test made meaningful assertions or whether the code is actually correct.
* A line can be "covered" while still being poorly or meaninglessly tested.

### Measuring Coverage

1. Run tests with a coverage report:
   ```powershell
   python -m pytest --cov=my_module --cov-report=term-missing
   ```
2. Review the output:
   ```text
   Name             Stmts   Miss  Cover   Missing
   --------------------------------------------------
   my_module.py        45      4    91%   88-91
   ```
3. The **`Missing`** column lists exact line numbers never executed by any test — inspect these to find untested branches.

### Is a High Percentage (e.g., 90%) Always the Right Target?

No — coverage targets should be based on **risk and complexity**, not a fixed number.

| Code Type                                                  | Recommended Priority                                                          |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Core business/decision logic                               | High — test every branch (success, error, edge case)                          |
| Simple configuration loading                               | Low — rarely worth testing                                                    |
| Orchestration/entry-point code (e.g., a `main()` function) | Often excluded from unit tests; better suited to integration or smoke testing |
| Third-party library internals                              | Not your responsibility to cover                                              |

### Better Guidance Than a Fixed Percentage

* Prioritize covering **every distinct logical branch**, not just a raw percentage of lines.
* Use coverage reports as a **diagnostic tool** to find blind spots, not a target to chase artificially.
* Reserve full end-to-end runs (hitting real APIs/databases) for **integration or smoke tests**, separate from unit tests.

# Summary

* Unit tests validate logic in isolation using **mocks**, never touching real APIs, databases, or external services.
* Mocking replaces real dependencies (HTTP clients, database connections, file systems, SDKs) with configurable fake objects.
* Run tests with `python -m pytest -v` and check coverage with `python -m pytest --cov=my_module --cov-report=term-missing`.
* Coverage measures **execution**, not **correctness** — high coverage doesn't guarantee bug-free code.
* Apply coverage targets based on the **importance and complexity** of the code, not a blanket rule.
