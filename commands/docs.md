# Swallow Pricing Engine

A configurable pricing engine that takes a project definition (JSON config) and a quote payload and produces a price with associated decision rules (exclusions, excesses, endorsements, refers).

Used widely for fiscal products in Banking and Insurance.

## Installation

```bash
npm install @llow-group/swa_llow_pricing_engine
```

## Quick Start

```javascript
const { engine } = require('@llow-group/swa_llow_pricing_engine');

const project = {
    meta: {
        name: 'Simple Project',
        description: 'Simple motor premium'
    },
    input: {
        format: {
            base: {
                label: 'Base value',
                def: 1000.00,
                type: 'decimal',
                exp: '{{base}}',
                static: true
            }
        }
    },
    steps: [{
        step: 'calculation',
        name: 'Premium',
        key: 'premium',
        def: 0,
        formula: '{{base}} * 10.91'
    }],
    output: {
        format: {},
        result: {
            key: 'result',
            formula: 'round({{premium}}, 2)',
            type: 'decimal',
            def: 0
        },
        valid: {
            key: 'output',
            exp: '{{exclusions.count()}} === 0',
            def: true,
            type: 'boolean'
        }
    }
};

const result = await engine({
    project,
    quote: { base: 500 },
});

// Output (always the same schema)
// {
//     result: 5455,
//     valid: true,
//     debug: {
//         errors: [],
//         exclusions: [],
//         refers: [],
//         excesses: [],
//         endorsements: [],
//         indicators: {},
//         meta: { ... },
//         timer: [ ... ],
//         steps: { ... }
//     }
// }
```

## How It Works

The engine processes data through a pipeline:

```
Quote Payload
    |
Input Block  (validate, map, type-check the incoming payload)
    |
Steps []     (transform, calculate, enrich, apply rules - in sequence)
    |
Output Block (format result, calculate final price, determine validity)
    |
Standardised Response
```

Properties flow through each step. The result of the input block is passed to the first step, and each step's result is passed to the next. All the way through to the output block.

When creating steps, reference previous results with template variables: `{{property_name}}`.

## Engine Parameters

```javascript
const result = await engine({
    project,           // Project config (JSON) - required
    quote,             // Quote payload (object) - required
    environment: {     // API config for external/modular steps - optional
        root_url: 'https://api.llow.io/sw/quotes',
        client_id: '',
        client_secret: '',
    },
    is_test: false,    // Skip input mapping when true - optional
    source: 'api',     // 'api', 'form', or 'test' - optional
    log: {},           // External log state - optional
});
```

## Project Config

A project config has four top-level sections: `meta`, `input`, `steps`, and `output`.

### Meta

```javascript
{
    meta: {
        name: 'My Project',
        description: 'Description of the project',
        project_reference: 'uuid',
        updated_at: '2024-01-01T00:00:00Z'
    }
}
```

### Input Block

The input block sets up the data contract with the quote payload. It validates types and maps incoming data to internal property names.

```javascript
{
    input: {
        // Optional - transform payload before format processing
        mapping: `
            let result = {
                name: proposer.name,
                age: proposer.age,
                total_claims: drivers.map(d => d.claims).reduce((sum, c) => sum + c, 0)
            };
            return result;
        `,
        // Optional - set true when payload is an array
        batch: false,
        // Optional - child format definitions for array-type inputs
        format_children: {
            driver_format: {
                ncd: { label: 'NCD', exp: '{{ncd}}', type: 'integer', def: 0 },
                dob: { label: 'DOB', exp: '{{dob}}', type: 'date', def: '2000-01-01' }
            }
        },
        // Required - defines the input properties
        format: {
            proposer_name: {
                label: 'Name',
                exp: '{{proposer.name}}',
                type: 'string',
                def: 'Unknown',
                index: true
            },
            proposer_age: {
                label: 'Age',
                exp: '{{proposer.dob.age(YY)}}',
                type: 'integer',
                def: 18
            },
            additional_drivers: {
                label: 'Drivers',
                exp: '{{additional_drivers}}',
                type: 'array',
                def: [],
                format_children: 'driver_format'
            },
            base_price: {
                label: 'Base price',
                type: 'decimal',
                def: 1000.00,
                static: true
            }
        }
    }
}
```

**Format property fields:**

| Field | Description |
|-------|-------------|
| `label` | Human-readable description |
| `type` | Data type: `string`, `integer`, `decimal`, `number`, `currency`, `boolean`, `date`, `array`, `object` |
| `exp` | Expression to extract value from payload. Uses `{{path}}` syntax with query language operators |
| `def` | Default value if not found or expression fails |
| `index` | If `true`, value is stored as a searchable indicator |
| `static` | If `true`, value is internal (not from payload) and cannot be overwritten |
| `format_children` | Reference to a child format definition (for array types) |

**Mapping block** (optional): A JavaScript string that pre-processes the raw payload before the format block runs. Must return an object or array. Only runs when `source` is `'api'` (not `'form'`).

**Batch mode** (optional): Set `batch: true` when the incoming payload is an array. Each item in the array is processed through the input format independently.

### Steps

Steps are an ordered array of processors. Each step receives the accumulated result of all previous steps.

```javascript
{
    steps: [
        {
            id: 'unique_id',
            step: 'transform',       // Step type
            name: 'Step Name',       // Display name
            description: 'What it does',
            key: 'step_key',         // Result property name
            placeholder: false,      // If true, skip execution and retain data
            retain: true,            // If true, merge output with previous data
            // ... step-specific config
        }
    ]
}
```

All steps support `placeholder: true` which skips execution and passes data through unchanged. This is useful for disabling steps without removing them.

#### Step Types

---

#### `transform`

Maps and transforms properties into new values.

```javascript
{
    step: 'transform',
    key: 'age_transform',
    retain: true,
    format: {
        age_months: {
            label: 'Age in months',
            exp: '{{age_years}} * 12',
            type: 'integer',
            def: 0
        },
        postcode_area: {
            label: 'Postcode area',
            exp: '{{postcode.regex(^[A-Za-z]{1,2})}}',
            type: 'string',
            def: 'X'
        }
    }
}
```

When `retain: true`, the output is merged with the input so all previous properties remain available to subsequent steps.

---

#### `calculation`

Evaluates a mathematical formula. Uses [mathjs](https://mathjs.org/) for evaluation.

```javascript
{
    step: 'calculation',
    key: 'premium',
    def: 0,
    formula: 'round({{base_price}} * (25 / {{age}}) * {{factor}}, 2)'
}
```

Calculations always retain previous data. If the formula fails (e.g. division by zero, non-numeric input), the `def` value is used and an error is logged.

---

#### `factors`

Rating factor tables that produce weights, exclusions, refers, excesses, and endorsements. Factors map input dimensions to lookup tables (hashes).

```javascript
{
    step: 'factors',
    key: 'driving_factors',
    result: {
        key: 'driving_factors',
        formula: 'round({{age_factor}} * {{ncd_factor}}, 4)',
        type: 'decimal',
        def: 0
    },
    factors: {
        age_factor: {
            def: { weight: 1, exclude: false, refer: false, excess: 0, endorsement: '' },
            dimensions: {
                age: {
                    value: ['18', '25', '35', '50'],
                    value_to: ['24', '34', '49', '65']  // Optional range upper bounds
                }
            },
            hashes: {
                '18': { weight: 1.8, exclude: false, refer: true, excess: 500, endorsement: 'Y1' },
                '25': { weight: 1.2, exclude: false, refer: false, excess: 0, endorsement: '' },
                '35': { weight: 1.0, exclude: false, refer: false, excess: 0, endorsement: '' },
                '50': { weight: 0.9, exclude: false, refer: false, excess: 0, endorsement: '' }
            }
        }
    }
}
```

Multi-dimensional factors use `::` as a separator in hash keys (e.g. `'W::1'` for postcode `W` and children `1`).

Each factor hash can produce:
- `weight` - numeric multiplier used in the result formula
- `exclude` - if `true`, adds an exclusion
- `refer` - if `true`, adds a referral
- `excess` - if non-zero, adds an excess
- `endorsement` - if non-empty, adds an endorsement

---

#### `collection`

Looks up values from a data collection (array of objects) using filter/map query syntax.

```javascript
{
    step: 'collection',
    key: 'postcode_lookup',
    collection: [
        { key: 'W', amount: 500, rate: 1.2 },
        { key: 'E', amount: 300, rate: 1.0 },
        { key: 'N', amount: 400, rate: 1.1 }
    ],
    format: {
        postcode_amount: {
            label: 'Postcode amount',
            exp: 'collection.filter(key={{postcode_area}}).map(amount).first()',
            type: 'integer',
            def: 0
        }
    }
}
```

---

#### `links`

Maps a single identifier to a set of enrichment values. Like a dictionary/lookup table.

```javascript
{
    step: 'links',
    key: 'vehicle_enrichment',
    exp: '{{vehicle_body_type}}',
    format: {
        vehicle_weight: { label: 'Weight', def: 1000, type: 'integer' },
        vehicle_rating: { label: 'Rating', def: 1, type: 'integer' }
    },
    links: {
        '2DH': { vehicle_weight: 1000, vehicle_rating: 1 },
        '4DH': { vehicle_weight: 1400, vehicle_rating: 4 },
        'MPV': { vehicle_weight: 2000, vehicle_rating: 5 }
    }
}
```

The `exp` is evaluated to get a key, which is then looked up in the `links` map. If not found, format defaults are used.

---

#### `label`

Creates a category/label from conditional rules. The last matching item wins.

```javascript
{
    step: 'label',
    key: 'age_band',
    def: 'unknown',
    items: [
        { name: 'Young', exp: '{{age}} < 25', value: 'young' },
        { name: 'Mid', exp: '({{age}} >= 25 && {{age}} < 50)', value: 'mid' },
        { name: 'Senior', exp: '{{age}} >= 50', value: 'senior' }
    ]
}
```

---

#### `exclusion`

Boolean tests that exclude (reject) a quote. If any item evaluates to `true`, the exclusion is triggered.

```javascript
{
    step: 'exclusion',
    key: 'my_exclusions',
    items: [
        { name: 'Too young', exp: '{{age}} < 17', def: false },
        { name: 'Banned driver', exp: '{{is_banned}}', def: false }
    ]
}
```

Triggered exclusions are collected in the output's `exclusions` array.

---

#### `refer`

Same structure as exclusions but for flagging quotes for manual review rather than rejecting them.

```javascript
{
    step: 'refer',
    key: 'my_refers',
    items: [
        { name: 'Expensive vehicle', exp: '{{vehicle_value}} > 75000', def: false }
    ]
}
```

Triggered refers are collected in the output's `refers` array.

---

#### `endorsements`

Conditional string labels added to a quote. Each matching item appends its `value` to the endorsements list.

```javascript
{
    step: 'endorsements',
    key: 'my_endorsements',
    def: [],
    items: [
        { name: 'Not parked at home', exp: '{{home_postcode}} != {{vehicle_postcode}}', value: 'A1' },
        { name: 'West London', exp: '({{postcode}}).startsWith("W")', value: 'W' }
    ]
}
```

Duplicate endorsement values are automatically removed (unique only).

---

#### `excesses`

Conditional numeric values added as compulsory excess amounts.

```javascript
{
    step: 'excesses',
    key: 'my_excesses',
    def: [200.00],
    items: [
        { name: 'Young driver', exp: '{{age}} < 25', value: 100.00 },
        { name: 'Expensive car', exp: '{{vehicle_value}} > 100000', value: 400.00 }
    ]
}
```

---

#### `external`

Calls an external API and maps the response into properties.

```javascript
{
    step: 'external',
    key: 'credit_check',
    options: {
        url: 'https://api.example.com/credit',
        method: 'post',
        timeout: 5000,
        headers: { auth: 'secret_key' },
        data: { id: 150 },
        mock: false  // Set true in tests to skip the API call and use format defaults
    },
    format: {
        credit_score: {
            label: 'Credit score',
            exp: '{{json.score}}',
            type: 'integer',
            def: 0
        }
    }
}
```

For testing, set `options.mock: true` to bypass the API and return format defaults. You can also use `mock::<format_key>` in the quote input to inject specific mock values per field.

---

#### `code`

Executes custom JavaScript code. The code has access to all current properties and must return an object.

```javascript
{
    step: 'code',
    key: 'custom_check',
    code: `
        if (premium > 100) {
            return { is_high: true };
        } else {
            return { is_high: false };
        }
    `,
    freeform: false,
    format: {
        is_high: {
            label: 'Is high premium',
            exp: '{{is_high}}',
            type: 'boolean',
            def: false
        }
    }
}
```

The code string runs in a sandboxed context with access to lodash (`_`) and mathjs functions. When `freeform: true`, the code receives the raw object; otherwise property names are sanitised.

---

#### `batch`

Sends an array of quotes to another Swallow project via API and collects the results as a collection.

```javascript
{
    step: 'batch',
    key: 'batch_quotes',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid'
    },
    format: {
        total_premium: {
            label: 'Total',
            exp: 'collection.map(result).sum()',
            type: 'decimal',
            def: 0
        }
    }
}
```

Quotes are chunked into batches of 100 and processed in parallel.

---

#### `modular`

Calls another Swallow project via API and maps the response.

```javascript
{
    step: 'modular',
    key: 'sub_project',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid',
        timeout: 5000
    },
    format: {
        sub_result: {
            label: 'Sub-project result',
            exp: '{{result}}',
            type: 'decimal',
            def: 0
        }
    }
}
```

Set `timeout: 0` to skip the API call and return format defaults.

---

#### `modules`

Runs an embedded project (provided in the project's `modules` array) against the current data. Like `modular` but without an API call.

```javascript
{
    step: 'modules',
    key: 'embedded_module',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid',
        root_property: '',   // Optional - send a sub-property instead of entire object
        debug: false         // Optional - include debug info from module run
    },
    format: {
        module_result: {
            label: 'Module result',
            exp: 'collection.map(result).first()',
            type: 'decimal',
            def: 0
        }
    }
}
```

### Output Block

The output block calculates the final price, determines validity, and formats additional output properties.

```javascript
{
    output: {
        // Optional - additional properties to include in the response
        format: {
            commission: {
                label: 'Commission',
                exp: '{{commission_amount}}',
                type: 'decimal',
                def: 0
            },
            tax: {
                label: 'Tax',
                exp: '{{tax_amount}}',
                type: 'decimal',
                def: 0
            }
        },
        // Required - calculates the final price
        result: {
            key: 'result',
            formula: 'round({{premium}} + {{tax}} + {{commission}}, 2)',
            type: 'decimal',
            def: 0
        },
        // Required - determines if the quote is valid
        valid: {
            key: 'output',
            exp: '{{exclusions.count()}} === 0',
            def: true,
            type: 'boolean'
        },
        // Optional - JavaScript code to post-process the output
        mapping: `
            return {
                ...result,
                formatted_price: '£' + result.toFixed(2)
            };
        `
    }
}
```

The `valid` expression has access to `{{exclusions}}`, `{{refers}}`, `{{endorsements}}`, and `{{excesses}}` arrays collected during step processing.

## Expressions and Query Language

Expressions use `{{path}}` syntax powered by [Swallow QL](https://github.com/llow-group/swa_llow_ql). They support:

**Basic property access:**
```
{{age}}                          // top-level property
{{proposer.name}}                // nested property
{{proposer.dob.age(YY)}}        // age in years from current date
{{proposer.dob.age(MM)}}        // age in months
{{proposer.dob.age(YY, start_date)}} // age relative to another date field
```

**Array operations:**
```
{{drivers.count()}}              // array length
{{drivers.sum(claims)}}          // sum a property
{{drivers.map(ncd).min()}}       // minimum value
{{drivers.map(ncd).max()}}       // maximum value
{{drivers.filter(employment=E).count()}}  // filter then count
```

**String operations:**
```
{{postcode}}.regex(^[A-Za-z]{1,2})     // regex match
{{postcode}}.postcode(area)            // postcode area extraction
({{name}}).includes("Bob")             // string includes
({{postcode}}).startsWith("W")         // starts with
```

**Boolean expressions:**
```
({{age}} > 25 && {{ncd}} >= 2)         // AND
({{age}} < 18 || {{is_banned}})        // OR
(!{{has_exclusions}})                  // NOT
```

**Aggregation helpers:**
```
min([{{drivers.map(age).min()}}, {{proposer_age}}])    // min of values
max([{{val_a}}, {{val_b}}])                            // max of values
```

**Excel formulas** are also supported as expressions:
```
SUM({{array}})
VLOOKUP({{key}}, {{table}}, 2, false)
IF({{age}} > 25, "adult", "young")
```

## Exported Utilities

The package exports many utilities beyond the engine:

```javascript
const {
    // Core engine
    engine,

    // Query Language
    QL,

    // Expression & calculation helpers
    evaluateExpression,
    evaluateCalculation,
    evaluateFactors,
    evaluateExternal,
    evaluateCollection,
    evaluateCustomCode,
    evaluateExcelCode,
    excelFormulas,
    evaluateTests,
    generateTests,
    evaluateDiff,
    evaluateLists,
    evaluateURL,
    evaluateData,

    // Command mapping (UI helpers)
    commandToExpression,
    expressionToCommand,

    // Compression
    compressProject,
    decompressProject,
    isProjectCompressed,
    compressCollection,
    decompressCollection,

    // Schema validation
    evaluateSchema,
    buildFullSchema,
    projectSchema,
    inputSchema,
    outputSchema,
    // ... step-specific schemas

    // Templates
    bikeInsuranceTemplate,
    petInsuranceTemplate,

    // Prompt generation
    buildPrompt,

    // Test fixtures
    baseProject,
    testProject,
    testQuote,
} = require('@llow-group/swa_llow_pricing_engine');
```

## Project Compression

Projects can be compressed for storage/transfer:

```javascript
const { compressProject, decompressProject, isProjectCompressed } = require('@llow-group/swa_llow_pricing_engine');

const compressed = compressProject(project);
const original = decompressProject(compressed);

// The engine accepts compressed projects directly
const result = await engine({ project: compressed, quote });
```

## Schema Validation

Every part of a project config has a corresponding JSON schema defined in `__schemas__/`. These are used to validate project structure before running the engine.

### `evaluateSchema`

The main validation function. Validates a full project (or an individual step) against its schema.

```javascript
const { evaluateSchema, testProject, testQuote } = require('@llow-group/swa_llow_pricing_engine');

// Validate a full project
const result = evaluateSchema({ project: myProject });
// { valid: true, errors: [] }

// Validate with input mapping check (pass a quote to test the mapping code runs)
const result = evaluateSchema({ project: myProject, quote: { age: 25 } });

// Validate a single step against its step schema
const { stepCalculationSchema } = require('@llow-group/swa_llow_pricing_engine');
const result = evaluateSchema({
    project: myStep,
    schema: stepCalculationSchema,
});
```

**Return value:**

```javascript
{
    valid: true | false,
    errors: [
        {
            path: 'input.format.name.label',  // JSON path to the problem
            keyword: 'required',               // Schema keyword that failed
            message: 'This is required',       // Human-readable message
            type: 'input' | 'output' | 'step', // Section type
            key: 'input',                       // Step key or section name
            step: 'transform',                  // Step type (for step errors)
        }
    ]
}
```

**What it validates:**

1. **Project structure** - `meta`, `input`, `steps`, and `output` are present with correct types
2. **Input schema** - `format` properties have required fields (`label`, `type`, `def`), property names match `^[A-Za-z_][A-Za-z0-9_]*$`
3. **Output schema** - `result` (with `formula`), `valid` (with `exp`), and `format` are valid
4. **Each step** - Validated against the step-specific schema (e.g. `calculation` requires `formula`, `exclusion` requires `items`)
5. **Input mapping** - If a `quote` is passed, the mapping code is executed to check for runtime errors
6. **Factors consistency** - `key` must match `result.key`, factor names cannot be empty
7. **Placeholder steps** - Steps with `placeholder: true` skip validation

### `buildFullSchema`

Returns the complete composed JSON schema with all step types inlined. This is useful for external tooling, editor autocompletion, or generating documentation.

```javascript
const { buildFullSchema } = require('@llow-group/swa_llow_pricing_engine');

const fullSchema = buildFullSchema();
// Returns a JSON Schema object with steps.items.oneOf containing all step schemas
```

The generated schema is also written to `dist/schema/index.json` when you run:

```bash
npm run generate:schema
```

### Available Schemas

Individual schemas are exported for validating specific sections or step types:

```javascript
const {
    // Top-level sections
    projectSchema,          // Full project structure
    inputSchema,            // Input block
    outputSchema,           // Output block
    testsSchema,            // Embedded test cases
    diffSchema,             // Diff/comparison
    processSchema,          // Process config

    // Step schemas (one per step type)
    stepTransformSchema,
    stepLabelSchema,
    stepLinksSchema,
    stepExclusionSchema,
    stepReferSchema,
    stepExcessesSchema,
    stepEndorsementsSchema,
    stepCalculationSchema,
    stepExternalSchema,
    stepFactorsSchema,
    stepCollectionSchema,
    stepCodeSchema,
    stepModularSchema,
    stepModulesSchema,
    stepBatchSchema,
} = require('@llow-group/swa_llow_pricing_engine');
```

Source schemas live in `__schemas__/` as JS modules. The full composed schema is generated to `dist/schema/index.json`.

## Templates

Starter templates are included for common use cases:

```javascript
const { bikeInsuranceTemplate, petInsuranceTemplate } = require('@llow-group/swa_llow_pricing_engine');

const result = await engine({
    project: bikeInsuranceTemplate,
    quote: { /* ... */ },
});
```

## Testing

```bash
npm test              # run all tests
npm run test:watch    # watch mode
```

Projects can include embedded test cases:

```javascript
{
    tests: [{
        id: 'default',
        key: 'defaults',
        name: 'Default Test',
        input: {},              // Override input values (skips input mapping)
        output: {
            result: 1000,
            valid: true
        },
        mocks: {},              // Mock values for external steps
        exclusions: [],
        excesses: [],
        endorsements: [],
        refers: []
    }]
}
```

Run embedded tests programmatically:

```javascript
const { evaluateTests } = require('@llow-group/swa_llow_pricing_engine');

const results = await evaluateTests({ project });
// Returns pass/fail for each test case
```

## Development

```bash
npm test                    # run tests
npm run test:watch          # watch mode
npm run generate:types      # generate TypeScript definitions
npm run generate:schema     # generate JSON schemas
```
