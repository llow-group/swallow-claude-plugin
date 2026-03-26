---
name: swallow-docs
description: Load the Swallow Pricing Engine documentation and JSON schema as reference context
tools:
  - Read
  - MCP
---

# Swallow Documentation

Load the Swallow Pricing Engine reference documentation and schema to answer questions or inform model building.

1. **Read the agent rules** — Read `docs/swallow_Agent_rules.md` first. This is the authoritative set of constraints for building, validating, and testing Swallow configs. All agents and prompts must follow these rules.

2. **Read the documentation** — Read `docs/swallow_readme.md` for the full engine reference covering step types, expressions, input/output blocks, query language, and configuration patterns.

3. **Read the schema** — Read `docs/swallow_schema.json` for the JSON schema defining the structure of a Swallow project, including all step types, input formats, output blocks, and test definitions.

4. **Optionally fetch live schema** — Call `schema_swallow_project` via MCP to get the latest schema directly from the engine if you need to verify the local schema is current.

Use this documentation to:
- Check the agent rules before building or modifying any config
- Answer questions about Swallow syntax, step types, and expressions
- Validate configuration patterns against the schema
- Look up the query language (1liner/QL) operators and syntax
- Check available fields for input format properties (`key`, `label`, `sub_label`, `type`, `def`, `exp`, `static`, `index`)
- Understand step-specific configuration requirements
