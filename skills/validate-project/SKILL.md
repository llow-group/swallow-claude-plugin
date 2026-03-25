---
name: validate-project
description: Validate an existing Swallow project configuration against the schema
tools:
  - MCP
  - Read
---

# Validate Project

The user wants to validate a Swallow project configuration.

1. **Load the project** — Read the JSON file the user provides or references.

2. **Validate** — Call `validate_swallow_project` via MCP with the project JSON.

3. **Report results**:
   - If `valid: true` — confirm the project is schema-valid.
   - If `valid: false` — list each error with its path and message. Suggest specific fixes for each error, referencing the Swallow constraints.

4. **Fix if asked** — Apply fixes and re-validate until the project passes.
