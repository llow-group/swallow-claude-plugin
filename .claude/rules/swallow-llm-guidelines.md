# LLM Guidelines for Generating Swallow Configurations

These are guidelines for when YOU (the LLM) are generating Swallow project configurations. The engine supports more than what's listed here — these rules keep generated configs simple, correct, and maintainable.

## Step types to use

Stick to these step types when generating new configurations:

- **transform** — data transformation and aggregation
- **exclusion** — quote validity rules
- **refer** — flagging quotes for manual review
- **excesses** — defining deductibles
- **endorsements** — policy endorsements
- **calculation** — mathematical computations (Math.js)
- **collection** — data lookups from tables (prioritise this for any lookup)
- **code** — custom JavaScript (only when no other step works)
- **external** — external API calls
- **modules** — reusing other Swallow projects

Avoid `factors`, `modular`, `batch`, `links`, and `label` steps in generated configs — they're powerful but complex and better built by hand in the Swallow UI.

## Expression rules

- Always wrap property references in `{{double_curly_braces}}`
- Use loose equality `==` not strict `===`
- In collection steps: always start with `collection`, always chain `.filter()`, data column first then `{{key}}` — e.g. `collection.items.filter(age > {{age}})`
- Use `round()` for monetary values

## Naming rules

- `id` and `key` must be identical within a step, unique across the project
- Never use a step type name as a key (e.g. never `key: "exclusion"`)
- Use `snake_case` for all keys and property names
- Input fields should use flat names (`{{driver_age}}` not `{{driver.age}}`) unless using input mapping

## Input rules

- Every input property must have `type`, `label`, `def`, and `exp`
- Match `exp` to the key: `key: "age"` → `exp: "{{age}}"`
- Use `static: true` for internal constants
- Use `index: true` for searchable fields

## Step rules

- Every `transform` step must have `retain: true`
- Every property must have a `def` value
- Calculation steps should use `integer` or `decimal` type inputs
- Endorsement step `def` should be an empty array `[]`
- Steps inherit from all previous steps — order matters

## Output rules

- `result` uses `formula` field (not `exp`), type must be `decimal`, `currency`, or `integer`
- `valid` uses `exp` field (not `formula`), type must be `boolean`
- Use `({{exclusions.count()}} === 0)` in `valid.exp` to check all exclusions

## Test rules

- Every test needs `id`, `name`, `key`, `input`, and `output`
- `output` must have `result` (number) and `valid` (boolean)
- Include at minimum: happy path, exclusion trigger, edge case

## Workflow

1. Build the project JSON
2. Call `validate_swallow_project` to check schema
3. Call `test_swallow_project` to run pricing
4. Iterate until all tests pass
