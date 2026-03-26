---
name: test-project
description: Run a Swallow project's tests through the pricing engine and analyse results
tools:
  - MCP
  - Read
---

# Test Project

The user wants to run tests against a Swallow project configuration.

1. **Load the project** — Read the JSON file the user provides or references.

2. **Check test format** — Before running, verify every test has an `output` object with `result` (number) and `valid` (boolean). If any test has `output: {}` or is missing these fields, fix it before submitting (use `result: 0, valid: true` as placeholders for discovery runs).

3. **Run tests** — Call `test_swallow_project` via MCP with the project JSON.

4. **Analyse results** — For each test case:
   - Show the test name, expected output, and actual output
   - Flag any mismatches between expected and actual `result` or `valid`
   - List any errors from the pricing engine (missing properties, expression failures, etc.)

5. **Summarise** — Report how many tests passed vs failed, and what the common issues are.

6. **Fix if asked** — Suggest specific changes to steps/expressions to fix failing tests. Re-run after each fix to confirm.
