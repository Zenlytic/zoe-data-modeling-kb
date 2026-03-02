# Zenlytic Zoë Data Model - Operational Knowledge Base

**Version:** 3.0 | **Last Updated:** 2026-03-02

Use this document when working with Zenlytic customer workspaces via Git repositories. This covers Git operations, YAML schema, joins, dimensions, measures, and how Zoë (the AI analyst) uses the semantic layer.

> **Version Note:** This KB is versioned. Check the changelog at the bottom for what changed between versions. When making updates, increment the version (minor changes: 3.0 → 3.1, major restructures: 3.x → 4.0) and add a changelog entry.

> **Companion Document:** LLM model behavioral testing data (tier framework, error recovery patterns, model-specific data modeling compensations) is maintained separately in `LLM_Model_Testing_Reference.md`. That document is for internal Zenlytic use — it is NOT an operational instruction for Claude Code and should NOT be included in CLAUDE.md.

---

## Critical Constraints (Read First)

These rules apply to every task, every repo, every customer. Violations cause real production problems.

### 1. Never Commit CLAUDE.md to Customer Repositories

CLAUDE.md files must NEVER be committed to a customer's Git repository. These are internal working files for Claude Code and must not appear in the Zenlytic Data Model Editor, be visible to customers, or be ingested by Zoë.

**Required setup for every customer repo:**
1. Add a `.gitignore` file at the repo root containing `CLAUDE.md`
2. If CLAUDE.md was previously committed, remove it from tracking: `git rm --cached CLAUDE.md`
3. Commit the `.gitignore` (and the CLAUDE.md removal if applicable)

When committing changes to a customer repo, never use `git add -A` or `git add .` without first verifying that CLAUDE.md is in `.gitignore`. Prefer adding specific files by name.

### 2. Be Conservative — Fix What's Broken, Don't Over-Enrich

- **Stay within the scope of what was asked.** If the task identified specific bugs (broken SQL, wrong value formats, comma-separated synonyms), fix those. Don't escalate to deleting files, restructuring the architecture, or adding descriptions to every field unless explicitly instructed.
- **Never delete views, topics, or model files without explicit customer approval.** Even if a file appears redundant or orphaned, the customer may have plans for it. Flag it as a recommendation and ask — don't act on your own judgment.
- **Don't over-enrich the semantic layer.** Too much guidance slows Zoë down. Every `zoe_description`, `description`, synonym, and `searchable` flag adds to the context Zoë processes on every query. Only add enrichment where Zoë is demonstrably failing. If chats already work, leave it alone.
- **Enrichment should be reactive, not proactive.** The pattern is: (1) observe Zoë failing on a question, (2) diagnose why, (3) add the minimum enrichment needed to fix it. Don't preemptively add `zoe_description` to every field "just in case."

### 3. Topics Are Being Deprecated — Avoid Creating New Ones

Topics will be removed from Zenlytic in the future. For new workspaces and migrations, **do not create new topics** unless absolutely necessary. Use this approach instead:

- **Structured joins** (table-to-table relationships with explicit ON conditions) → Put in the **model file** using the `relationships` property. This is the canonical location for all structured join logic.
- **Unstructured join context** (business rules about when/how to join, fan-trap warnings, field routing guidance, domain scoping) → Put in the **system prompt**, **view `description`**, or **field-level `zoe_description`** as appropriate.

If an existing workspace has topics, they still work — but when migrating or refactoring, move structured join logic to model `relationships` and move contextual guidance to descriptions/system prompt rather than creating new topics.

### 4. Always Run Before/After Tests on Major Changes

Before making any significant data model change (adding/removing joins, restructuring views, migrating from topics to relationships, changing field definitions), capture a baseline:

1. **Before the change:** Run 3-5 representative test questions. Save both the results AND the generated SQL.
2. **Make the change.**
3. **After the change:** Run the same questions. Compare results AND SQL to the baseline.
4. **If regression:** Diagnose whether the change caused the issue, and either fix the semantic layer or revert.

This applies to all changes, not just topic removals. SQL comparison is essential — two queries can return superficially similar results while one is fundamentally wrong (see Part 17 for details).

### 5. Never Invent Customer Terminology

Only use terms that exist in the customer's data model, memory definitions, or system prompts. If you need a label, ask the customer what they call it. Don't assume field names match business concepts — always verify with `SELECT DISTINCT`.

---

## Part 1: Git Repository Operations

### Repository Structure
```
/
├── zenlytic_project.yml    # Project configuration
├── models/
│   └── base_model.yml      # Database connection/model definitions
├── views/
│   └── *.yml               # View definitions (dimensions, measures, identifiers)
├── topics/
│   └── *.yml               # Topic definitions (join configurations)
└── dashboards/
    └── *.yml               # Dashboard configurations (optional)
```

### Common Git Operations

#### Clone and Connect to a Branch
```bash
git clone https://github.com/Zenlytic/<repo-name>.git
cd <repo-name>
git fetch origin
git checkout <branch-name>
git pull origin <branch-name>
```

#### Copy Content Between Repos/Branches
```bash
git clone https://github.com/Zenlytic/<target-repo>.git
cd <target-repo>
rm -rf models views topics dashboards zenlytic_project.yml

# Copy from source repo (exclude .git)
cp -r /path/to/source-repo/models /path/to/source-repo/views /path/to/source-repo/topics /path/to/source-repo/zenlytic_project.yml /path/to/target-repo/

git add -A
git commit -m "Description of changes"
git push origin <branch>
```

#### Search for References
Use Grep tool: `Grep pattern="search_term" path="/path/to/repo"`

### Branch Naming Conventions
- `data-sci-<n>` — Data scientist working branches
- `master` / `main` — Production branch

### Git Tips
- Always fetch before checkout to get latest remote branches
- Check git status before committing to review changes
- Use descriptive commit messages explaining what was migrated/changed
- The .git folder should NOT be copied between repos

---

## Part 2: Project Configuration

### zenlytic_project.yml
```yaml
name: project_name              # Project name
profile: connection_profile     # Database connection profile
version: 1

dashboard-paths:
  - dashboards
model-paths:
  - models
view-paths:
  - views
topic-paths:
  - topics

# Optional: For dbt MetricFlow integration
use_default_topics: false       # Set false for custom topic join logic
```

### Model File (models/base_model.yml)
```yaml
version: 1
type: model
name: model_name                # Unique model identifier
connection: connection_name     # Database connection reference
week_start_day: monday          # Optional: Week start for reporting
timezone: America/New_York      # Optional: Default timezone
```

---

## Part 3: Views (Core Building Block)

Views represent database tables and contain dimensions, measures, and identifiers.

### View File Structure
```yaml
version: 1
type: view
name: view_name                           # Unique identifier (letters, numbers, _ only)
model_name: model_name                    # Reference to model
sql_table_name: schema.table_name         # Direct table reference
# OR use derived_table for transformed data:
# derived_table:
#   sql: SELECT * FROM table WHERE condition

label: "Display Name"                     # Optional: User-facing name
description: "View description"           # Optional: Shown in UI and to Zoë
default_date: created_at                  # Optional: Default date dimension group

# Optional security
always_filter:                            # Filters applied to all queries
  - field: is_deleted
    value: "No"
access_filters:                           # Row-level security
  - field: region
    user_attribute: allowed_regions
required_access_grants: [grant_name]      # Access control (OR logic)

identifiers:                              # For joins (see Part 5)
  - name: identifier_name
    type: primary
    sql: ${field_name}

fields:                                   # Dimensions and measures
  # ... (see Parts 4a-4c)
```

### Key View Rules
- The `name` field must contain only letters, numbers, or `_` and remain unique within collections
- All names are automatically lowercased in Zenlytic
- When using `derived_table`, the SQL statement becomes the data foundation, and `always_filter` does not apply
- `always_filter` can join external fields by specifying the view name (e.g., `customers.is_churned`)
- Views support a `sets` property: lists of grouped fields
- **`zoe_description` is NOT valid at the view level** — use `description` instead (which is sent to both the UI and Zoë). `zoe_description` is valid on: dimensions, measures, dimension groups, and topics.

---

## Part 4a: Dimensions

Dimensions represent columns for grouping and filtering.

### Basic Dimension
```yaml
- name: field_name                        # Required: Unique identifier
  field_type: dimension                   # Required: Must be "dimension"
  type: string                            # Required: string, number, yesno, tier
  sql: ${TABLE}.COLUMN_NAME               # Required: SQL expression

  # Optional display properties
  label: "Display Name"                   # User-facing label
  description: "Description text"         # Shown in UI and to Zoë
  zoe_description: "AI-specific context"  # Overrides description for Zoë only
  group_label: "Sidebar Group"            # Groups fields in sidebar
  hidden: false                           # Hide from UI (still referenceable)

  # Optional behavior properties
  searchable: true                        # Enable natural language search indexing (see Part 12)
  allow_higher_searchable_max: true       # Optional: Raise index limit from 10K to 500K categories
  synonyms:                               # Alternative names for NLP queries
    - alternative_name
    - another_term
  primary_key: true                       # Marks as unique identifier
  value_format_name: "#,##0"              # Display formatting

  # Optional advanced properties
  tags: ['customer', 'orders']            # Special identifiers for Zenlytic
  drill_fields: [field1, field2]          # Fields shown in drill-down
  link: https://example.com/{{value}}     # URL template for drill-through
  filters:                                # Field-level filters
    - field: status
      value: "active"
  tiers: [0, 20, 40, 60, 80, 100]        # For tier type only
  window: true                            # Indicates window function in SQL
  extra:                                  # Custom metadata
    custom_key: custom_value
```

### Dimension Types
| Type | Description |
|------|-------------|
| `string` | Text data (default) |
| `number` | Numeric values |
| `yesno` | Boolean/Yes-No |
| `tier` | Categorized value ranges |

---

## Part 4b: Dimension Groups (Time-based)

Dimension groups auto-generate multiple time aggregations from a single column.

### Time Dimension Group
```yaml
- name: order                             # Base name (generates order_date, order_week, etc.)
  field_type: dimension_group             # Required
  type: time                              # Required: time or duration
  sql: ${TABLE}.ORDER_DATE                # Required for time type
  datatype: timestamp                     # timestamp, datetime, or date

  timeframes:                             # Required: List of time buckets to generate
    - raw                                 # Original value
    - date                                # YYYY-MM-DD
    - week                                # Week number
    - month                               # Month (1-12)
    - quarter                             # Quarter (1-4)
    - year                                # Year
    - day_of_week                         # Day name
    - day_of_month                        # Day (1-31)
    - day_of_year                         # Day (1-366)
    - week_of_year                        # Week (1-52)
    - month_of_year                       # Month (1-12)
    - hour_of_day                         # Hour (0-23)
    - fiscal_month                        # Fiscal month
    - fiscal_quarter                      # Fiscal quarter
    - fiscal_year                         # Fiscal year

  convert_tz: true                        # Optional: Convert timezone
  label: "Order Date"
  description: "When the order was placed"
  synonyms: [transaction_date, sale_date]
```

### Duration Dimension Group
```yaml
- name: fulfillment_time
  field_type: dimension_group
  type: duration                          # Calculates interval between dates
  sql_start: ${TABLE}.order_date          # Required for duration
  sql_end: ${TABLE}.ship_date             # Required for duration

  intervals:                              # Required: Time units to calculate
    - day
    - week
    - month
    - hour
```

---

## Part 4c: Measures (Metrics)

Measures represent aggregations/calculations.

### Basic Measure
```yaml
- name: total_revenue                     # Required: Unique identifier
  field_type: measure                     # Required: Must be "measure"
  type: sum                               # Required: Aggregation type
  sql: ${TABLE}.revenue                   # Required: SQL expression

  label: "Total Revenue"
  description: "Sum of all revenue"
  zoe_description: "Total sales revenue including all channels"
  group_label: "Revenue Metrics"
  hidden: false
  value_format_name: "$#,##0.00"

  synonyms: [total_sales, income, sales_total]

  filters:
    - field: is_cancelled
      value: "No"

  canon_date: order_at                    # Override default trending date
  extra:
    metric_type: revenue
```

### Measure Types
| Type | Description | Notes |
|------|-------------|-------|
| `sum` | Sum values | Basic aggregation |
| `average` | Calculate mean | Basic aggregation |
| `count` | Count rows | **Cannot have `filters` unless `sql` references a specific column** |
| `count_distinct` | Count unique values | Basic aggregation |
| `max` | Maximum value | Basic aggregation |
| `min` | Minimum value | Basic aggregation |
| `median` | Median value | Database-dependent |
| `sum_distinct` | Sum without duplication | Requires `sql_distinct_key` |
| `average_distinct` | Average unique values | Requires `sql_distinct_key` |
| `cumulative` | Aggregate over time | Requires `measure` property |
| `number` | Static/calculated value | No aggregation |

### Count Measure Filter Constraint
**Zenlytic does NOT support `filters` on a `count` measure that doesn't reference a specific column in its `sql` property.** If you need a filtered count, the `sql` must reference a column (e.g., `sql: ${TABLE}.id`), not just `sql: 1` or omit `sql` entirely. This is a platform constraint, not a modeling choice.

### Distinct Aggregations (Prevent Double-Counting)
```yaml
- name: total_order_value
  field_type: measure
  type: sum_distinct
  sql_distinct_key: ${order_id}           # Ensures uniqueness
  sql: ${order_total}
  description: "Sum of order totals without double-counting line items"
```

### Cumulative Measures
```yaml
- name: cumulative_revenue
  field_type: measure
  type: cumulative
  measure: total_revenue                  # References another measure
```

### Non-Additive Dimensions (e.g., MRR)
```yaml
- name: mrr
  field_type: measure
  type: max
  sql: ${TABLE}.mrr_amount
  non_additive_dimension:
    name: customers.customer_id           # Fully qualified field reference
    window_choice: max                    # max or min
    window_aware_of_query_dimensions: true
    window_groupings: [customer_id]       # Optional grouping fields
```

---

## Part 5: Identifiers and Joins

Joins between views are defined through **identifiers** (in views) and **topics** (join configuration).

### Identifiers in Views
```yaml
identifiers:
  # Primary Key - uniquely identifies records in this view
  - name: customer_id                     # Name must match across views to join
    type: primary
    sql: ${customer_id}

  # Foreign Key - references another view
  - name: store_id
    type: foreign
    sql: ${store_id}

  # Custom Join - explicit point-to-point join
  - name: discount_join
    type: join
    sql: ${discount_code}
    relationship: many_to_one             # Required for join type
    join_view: discounts                  # View to join to
```

### Identifier Types
| Type | Description |
|------|-------------|
| `primary` | Unique key for this view |
| `foreign` | References a primary key in another view |
| `join` | Custom explicit join definition |

### Join-As (Same Table Multiple Times)
```yaml
identifiers:
  - name: billing_address_id
    type: primary
    sql: ${address_id}
    join_as: billing_address
    join_as_label: "Billing Address"
    join_as_field_prefix: billing_

  - name: shipping_address_id
    type: primary
    sql: ${address_id}
    join_as: shipping_address
    join_as_label: "Shipping Address"
    join_as_field_prefix: shipping_
```

---

## Part 6: Topics (Join Configuration) — DEPRECATED FOR NEW WORKSPACES

> **Topics are being deprecated.** Do not create new topics for new workspaces or migrations. Use model-level `relationships` for structured joins and view `description` / system prompt for unstructured context. This section documents topic syntax for reference when working with existing workspaces that still use topics.

Topics organize views and define how they join together.

### Topic File Structure
```yaml
version: 1
type: topic
name: orders_topic
model_name: model_name
label: "Orders"
description: "Order transaction data with customer and product details"
zoe_description: "Use this topic for questions about orders, customers, and products"

base_view: orders                         # Primary/anchor view

always_filter:
  - field: is_test
    value: "No"
access_filters:
  - field: region
    user_attribute: allowed_regions
required_access_grants: [orders_access]

views:
  # Simple join (uses identifier matching)
  customers: {}

  # Explicit join configuration
  order_lines:
    join:
      join_type: left_outer               # left_outer, inner, full_outer
      relationship: one_to_many           # many_to_one, one_to_one, one_to_many, many_to_many
      sql_on: ${orders.order_id} = ${order_lines.order_id}

  # Multi-column join
  products:
    join:
      join_type: left_outer
      relationship: many_to_one
      sql_on: ${order_lines.product_id} = ${products.product_id} AND ${order_lines.variant_id} = ${products.variant_id}

  # Same view joined multiple times
  billing_addresses:
    from: addresses                       # Source view name
    join:
      join_type: left_outer
      relationship: many_to_one
      sql_on: ${orders.billing_address_id} = ${billing_addresses.address_id}

  shipping_addresses:
    from: addresses
    join:
      join_type: left_outer
      relationship: many_to_one
      sql_on: ${orders.shipping_address_id} = ${shipping_addresses.address_id}

  # Override access filters
  admin_data:
    override_access_filters: true
    join:
      join_type: left_outer
      relationship: many_to_one
      sql_on: ${orders.admin_id} = ${admin_data.id}
```

### Join Types
| Type | Description |
|------|-------------|
| `left_outer` | Include all from base, matched from joined |
| `inner` | Only matching records |
| `full_outer` | All records from both sides |

### Relationship Types
| Type | Description |
|------|-------------|
| `many_to_one` | Multiple base records → one joined record |
| `one_to_many` | One base record → multiple joined records |
| `one_to_one` | One-to-one match |
| `many_to_many` | Multiple on both sides |

### Joins Best Practice
- Don't join views only because it's possible — separate topics can be queried independently with results merged afterward
- Topics are the primary organizational mechanism for joins
- Default joining: If a view in a topic lacks explicit join configuration, pre-existing foreign key relationships (identifiers) are automatically applied

---

## Part 7: How Zoë Uses the Semantic Layer

Zoë writes SQL to answer user questions, using the semantic layer as context to understand the data model.

### How Zoë Works

| Aspect | How Zoë Operates |
|--------|-----------------|
| **Process** | User prompt → LLM → SQL query → Results |
| **SQL Generation** | Non-deterministic — the same prompt may yield different SQL each run |
| **Semantic Layer** | Used as **context** for the LLM to generate SQL |
| **Join Logic** | LLM interprets topic definitions, relationships, and identifiers to construct joins |
| **Permissions** | Row and column-based access controls are enforced by the system |

> **Exploratory Mode** is the primary mode for all new workspaces. It uses LLM semantic reasoning to generate SQL from the data model context. Some existing customers may still be on **Clarity mode** (deterministic field compilation) and require migration — see Part 16 for the migration workflow.

### How Zoë Sees Join Information

**Topics**: Zoë sees topics and their associated joins. She uses topic join definitions to inform the SQL she writes.

**Warning**: A potential issue can arise if the topic implies illogical or counter-intuitive joins. If a topic says it's valid to make many-to-many joins between large fact tables, Zoë may be misled into thinking it's a safe join when it will actually cause a fan-out.

**Best Practice**: Any `one_to_many` or `many_to_many` join in a topic *can* potentially degrade performance — but it won't necessarily if the join makes logical sense to Zoë. The key factor is whether Zoë can understand what the join represents, not just the cardinality.

**Identifiers**: Zoë sees identifiers (primary keys, foreign keys, and join-type identifiers). Identifiers provide join context alongside topics and model-level relationships.

**Relationships**: The `relationships` property on the model is how the Context Manager in Self-Service stores joins. **Zoë sees all of these.** Along with identifiers and topics, relationships provide another way to define join logic for Zoë.

**Where to Store Joins:**

Join context belongs in two places, based on its nature:

1. **Structured joins** (explicit table-to-table relationships with ON conditions) → **Model-level `relationships`**. This is the canonical, preferred location for all structured join logic. See the Model-Level Relationships example below.

2. **Unstructured join context** (business rules, fan-trap warnings, field routing guidance, "don't join X to Y" instructions, domain scoping) → **System prompt** (for workspace-wide rules), **view `description`** (for table-specific guidance), or **field-level `zoe_description`** (for field-specific instructions).

Obvious joins (e.g., matching foreign key names between tables) don't need to be stored anywhere — Zoë discovers these through identifiers.

> **Topics are being deprecated.** Existing topics still work, but when migrating or building new workspaces, use model `relationships` + descriptions/system prompt instead of creating topics. See Critical Constraint #3.

### Zoë Permissions

- `always_filter` — Appended to every SQL query on the view (not visible to Zoë's LLM reasoning)
- `access_filters` — Row-level security via user attributes
- `required_access_grants` — Column/view-level access control using OR logic across grants

### Context Sources Zoë Sees

| Context Source | Role |
|--------------|--------|
| **Custom System Prompt** | Organization-wide rules, terminology, calculation logic |
| **Structural Relationships** | Topics, joins, identifiers, topic descriptions |
| **Field & View Descriptions** | `zoe_description` (if set) overrides `description` for Zoë |
| **Memories** | Reinforced response patterns from previous queries |
| **Patterns** | Historical query patterns from the warehouse (see Part 14) |
| **Attachments** | Ad-hoc files and query results provided by the user during a conversation |

### When to Use Each Context Type

| Context Type | Best For | Use When |
|---|---|---|
| **Custom System Prompt** | Company terminology, business rules, calculation methodologies | Need organization-wide consistency across all Zoë interactions |
| **`zoe_description`** (field/topic level) | AI-specific guidance that differs from user-facing copy | Standard `description` is too technical or misleading for the AI |
| **`description`** (view/field/topic level) | Business context explaining how data should be used | Applies to both users in the UI and Zoë |
| **Memory** | Reinforcing desired response patterns for specific questions | User confirms Zoë's answer is correct and wants it repeated |
| **Topic Structure** | Understanding data connections and analytical context | Need Zoë to understand which fields relate to each domain |
| **Patterns** | Supplementing governed model with proven query patterns | Warehouse has rich query history; Snowflake connection available |

### Zoë-Specific Properties

| Property | Valid On | Purpose |
|----------|----------|---------|
| `zoe_description` | dimensions, measures, dimension_groups, topics | AI-specific context (overrides `description` for Zoë). **NOT valid on views.** |
| `description` | views, dimensions, measures, dimension_groups, topics | Shown in UI and sent to Zoë. On views, this is the only way to provide Zoë context. |
| `synonyms` | dimensions, measures, dimension_groups | Alternative names for NLP matching |
| `searchable: true` | dimensions | Index field categories for search |
| `hidden: true` | views, dimensions, measures, dimension_groups, topics | Exclude from Zoë's available fields |
| `tags` | dimensions, measures | Special identifiers for semantic meaning |

### Guiding Principles

> **"Could a talented data analyst answer questions using this data model on their first day?"**
>
> This is the foundational test for every data model. If the naming, descriptions, and structure would confuse a skilled analyst who is new to the domain, Zoë will also struggle.

> **"Give Zoë just enough context that chats work — not more."**
>
> Too much guidance, too many descriptions, and over-enriched metadata slow Zoë down. Only add enrichment where Zoë is demonstrably failing or confused.

### Best Practices for Zoë

1. **Use `zoe_description`** on fields and topics for AI-specific context. **Do NOT use `zoe_description` on views** — use `description` instead.
2. **Use `description`** on views to provide join guidance and context.
3. **Add synonyms** for fields with multiple common names.
4. **Set `searchable: true`** on categorical dimensions users will filter on (see Part 12).
5. **Write clear topic descriptions** explaining analytical purpose.
6. **Use meaningful, distinguishing field names** — "Gross Revenue" and "Net Revenue" instead of "Revenue 1" and "Revenue 2".
7. **Hide technical fields** that shouldn't appear in queries.
8. **Document fan trap risks** in view-level `description`.
9. **Avoid many-to-many joins** in topics unless the join makes clear logical sense.
10. **Store structured joins in model-level `relationships`** (preferred). Use view `description` and system prompt for unstructured join context. Avoid creating new topics.
11. **Set `default_date`** on every view with time-series measures, and use `canon_date` on measures that trend by a different date.
12. **Configure entity drills** with `drill_fields` on primary key dimensions.
13. **Use Zoë Memories** for company-specific data interpretation patterns.
14. **Maintain consistent database capitalization** in searchable fields.
15. **Design for model-agnostic robustness** — different users may use different LLM backends; build the semantic layer so it works across all supported models, not just the most capable one.

### Example Zoë-Optimized View with Join Warnings
```yaml
version: 1
type: view
name: order_lines
model_name: my_model
sql_table_name: prod.order_lines
# NOTE: zoe_description is NOT valid at the view level. Use description instead.
description: |
  Order line items - one row per product in an order.
  JOIN GUIDANCE: This table has a many-to-one relationship with orders (many lines per order).
  When joining to orders, be careful not to double-count order-level metrics.
  NEVER join directly to other line-level tables (e.g., inventory_movements)
  without going through the orders table first - this will cause fan-out.
  Use sum_distinct with order_id as the distinct key for order-level aggregations.

fields:
  - name: marketing_channel
    field_type: dimension
    type: string
    sql: ${TABLE}.utm_source
    label: "Marketing Channel"
    description: "Source of customer acquisition"
    zoe_description: "The marketing channel that brought this customer. Use this to analyze marketing performance, attribution, and ROI by channel."
    searchable: true
    synonyms:
      - acquisition channel
      - traffic source
      - utm source
      - marketing source
```

### Model-Level Relationships
```yaml
version: 1
type: model
name: my_model
connection: my_connection

# Model-level relationships provide join context to Zoë alongside identifiers and topics
relationships:
  - from_table: orders
    join_table: customers
    join_type: left_outer
    relationship: many_to_one
    sql_on: ${orders.customer_id} = ${customers.customer_id}

  - from_table: order_lines
    join_table: orders
    join_type: left_outer
    relationship: many_to_one
    sql_on: ${order_lines.order_id} = ${orders.order_id}

  - from_table: order_lines
    join_table: products
    join_type: left_outer
    relationship: many_to_one
    sql_on: ${order_lines.product_id} = ${products.product_id}
```

---

## Part 8: Common Patterns

### Star Schema (Fact + Dimensions)
```
                    +-------------+
                    |  customers  |
                    |  (dimension)|
                    +------+------+
                           |
+-------------+    +-------+-------+    +-------------+
|   stores    |----+    orders     +----+  products   |
| (dimension) |    |    (fact)     |    | (dimension) |
+-------------+    +-------+-------+    +-------------+
                           |
                    +------+------+
                    |    dates    |
                    | (dimension) |
                    +-------------+
```

### Slowly Changing Dimension (SCD Type 2)
```yaml
fields:
  - name: dw_scd_is_current
    field_type: dimension
    type: yesno
    sql: ${TABLE}.is_current

  - name: dw_scd_effective_dttm
    field_type: dimension_group
    type: time
    sql: ${TABLE}.effective_date
    timeframes: [raw, date]

  - name: dw_scd_expire_dttm
    field_type: dimension_group
    type: time
    sql: ${TABLE}.expire_date
    timeframes: [raw, date]
```

### Fiscal Calendar Support
```yaml
- name: order
  field_type: dimension_group
  type: time
  sql: ${TABLE}.order_date
  timeframes:
    - raw
    - date
    - fiscal_month
    - fiscal_quarter
    - fiscal_year
```

### dbt MetricFlow Integration
- Semantic models map to Zenlytic views; dbt measures and metrics map to Zenlytic measures
- Dimensions are converted with appropriate time granularities
- Join logic and entity relationships are preserved and mapped to topics
- Set `use_default_topics` to `false` in `zenlytic_project.yml` for custom topics

---

## Part 9: Zoë Memories

Memories allow users to train Zoë with company-specific knowledge and desired response patterns. Zoë automatically retrieves relevant memories when answering questions.

### Creating Memories

**From Chat:** Click "Add to memory" beneath a helpful Zoë response.

**From Memory Portal (Settings → Memory):**
- **Text Memory**: Pair a user question with the desired Zoë response
- **Explore Memory**: Configure questions with specific time ranges, metrics, filters, and slices

### Key Rules
- Memories work for **data interpretation only**, not personality/behavioral changes
- For tone or style adjustments, use custom system prompts instead
- Memories are opt-in only — Zoë does not learn from conversation history automatically
- Memories remember the query, not the visualization or Python code used for advanced analysis
- **Memories with bad SQL persist and poison future queries** — when fixing data model issues, always audit memories (Settings → Memory) for entries created before the fix

### When to Use Memories vs Other Context Types

| Scenario | Use This |
|----------|----------|
| Company-wide business rules | **Custom System Prompt** |
| AI-specific guidance on a field | **`zoe_description`** |
| General business context for a table or field | **`description`** |
| Reinforcing a correct answer pattern | **Memory** |
| Join guidance or fan-trap warnings | **`description`** on view + **`zoe_description`** on topic |

---

## Part 10: Data Model Editor & Workspace Manager

### Data Model Editor
1. **Create a branch** — Zenlytic will not allow editing production live
2. **Edit YAML files** — Define metrics, joins, access controls in the browser
3. **Real-time validation** — Errors and warnings display with pinpointed locations
4. **Deploy to production** — Click "Deploy to Production" to merge and update Zoë (~10 second ingestion delay)

### Adding Tables
1. Click "+" → "Create View from Table"
2. Choose database and table
3. Click "Create View" — auto-generates boilerplate YAML

### Workspace Manager
The Workspace Manager is a self-service admin interface for **Organization Admins** to create, manage, and provision workspaces.

**Creating a New Workspace:**
1. **Name & SSO** — Name the workspace and optionally enable SSO User Provisioning
2. **Database Connections** — Copy from existing workspaces or create new ones (auto-tested)
3. **GitHub Repository** — Link the data model repo (Managed or Connected). Must be configured before Zoë can be used.

---

## Part 11: User Attributes, Roles & Permissions

### User Attributes
Assigned to users/groups. Drive `access_filters` and `required_access_grants` in views/topics.

**Reserved Attributes:**

| Attribute | Behavior |
|-----------|----------|
| `email` | Auto-populated with logged-in user's email; cannot be overridden |
| `zenlytic_connection_name` | Overrides which credential/connection executes queries |
| `zenlytic_connection_database` | Overrides database selection (applies to Postgres, MySQL, Redshift, Azure Synapse, SQL Server, Trino, DuckDB, Snowflake; NOT BigQuery, Druid, or Databricks) |

### User Roles

| Role | Key Characteristics |
|------|-------------------|
| **Admin** | All permissions — full workspace control |
| **Develop** | All admin permissions except workspace settings |
| **Develop without Deploy** | Cannot push data model changes to production |
| **Explore** | Most common role — content, download, analytics, Zoë chat access |
| **View** | Explore minus unlimited downloads |
| **Restricted** | View content only — requires data access controls |
| **Embed** | Specialized embedded user role |
| **Embed with SQL** | Embedded with SQL query access |
| **Embedded with Scheduling** | Embedded with dashboard scheduling |

---

## Part 12: Data Indexing, Entity Drills & Time Metrics

### Data Indexing (searchable Fields)
Setting `searchable: true` indexes distinct values (up to 10,000 categories). This enables Zoë to use specific filter values in queries.

```yaml
- name: status
  field_type: dimension
  type: string
  sql: ${TABLE}.status
  searchable: true
```

**High-Cardinality:** Use `allow_higher_searchable_max: true` to increase threshold to 500,000.

**Best Practices:**
- Set `searchable: true` on categorical dimensions users will filter on
- Don't index high-cardinality IDs unless needed
- Maintain consistent database capitalization — Zoë applies indexed values directly to filters
- Don't index sensitive fields (SSN, email)

### Entity Drills
Define `drill_fields` on primary key dimensions to create "Drill into [entity]" dropdown options:

```yaml
- name: sales_rep_id
  field_type: dimension
  type: string
  sql: ${TABLE}.SALES_REP_ID
  primary_key: true
  tags: ['Sales Rep']
  drill_fields: [first_name, last_name, email, status]
```

### Time Metrics (canon_date & default_date)

**`default_date`** (view level): Sets the default time dimension for all measures in a view.
```yaml
default_date: created_at                  # References a dimension_group name
```

**`canon_date`** (measure level): Overrides `default_date` for a specific measure.
```yaml
- name: canceled_subscriptions
  field_type: measure
  type: count
  sql: ${TABLE}.id
  canon_date: canceled_at                 # Trends by cancellation date, not creation date
```

**Cross-Table Dates:** Reference other views using dot notation: `default_date: google_record_stats.recorded_at`

**Rules:**
- Both reference the `name` of a `dimension_group` field (type: time)
- `canon_date` takes precedence over `default_date`
- Always set `default_date` on views with time-series measures

---

## Quick Reference: SQL Expression Syntax

| Syntax | Meaning |
|--------|---------|
| `${TABLE}` | Current table |
| `${field_name}` | Field in same view |
| `${view_name.field_name}` | Field in another view |
| `{{value}}` | Dynamic value (for links) |

---

## Part 13: New Customer Setup

When creating a new customer folder under Zenlytic-Customers/:
1. Create the customer subfolder
2. Clone the customer's Git repository into the subfolder
3. Create a customer-specific CLAUDE.md with customer name, repo URL, branch info, and any special notes
4. **Add a `.gitignore` to the customer repo** that excludes `CLAUDE.md`
5. The shared knowledge in this parent CLAUDE.md is automatically inherited

---

## Part 14: Patterns & Attachments

### Patterns (Query History Indexing)
Patterns index a team's historical analytical queries from the data warehouse (Snowflake only during beta), making them available to Zoë as supplemental context.

**Setup:** Settings → External Context → Query History tab. Toggle on connections, click Sync. Auto-syncs every 24 hours.

**What Gets Indexed:** SELECT and WITH statements referencing model tables. Excludes DDL/DML, ETL, system queries, staging tables. Up to 10,000 patterns per workspace.

**Data Modeling Implications:**
- Patterns supplement but do not replace the governed semantic layer
- Patterns can bridge gaps during Clarity→Exploratory migrations
- If Zoë consistently relies on patterns instead of governed fields, the semantic layer needs enrichment
- Audit patterns periodically — old patterns may reference renamed columns

### Attachments
Users can attach files (CSV, images, PDFs — up to 5 files per message, 25 MB each) and structured query results to Zoë conversations via the **+** icon.

When users frequently attach CSVs to supplement Zoë answers, it may signal the governed data model is missing measures or dimensions.

---

## Part 15: Greenfield Exploratory Workspace Setup

When setting up a brand-new workspace in Exploratory mode (no Clarity baseline to compare against), follow this structured approach. The goal is to import the data model, assess its readiness for Zoë, and prepare informed questions for the customer — not to make unsolicited changes.

### Phase 1: Import and Analyze Views

**1.1 Read all view files systematically**
```bash
# List all views
ls views/

# Read each view file
cat views/*.yml
```

For each view, document:
- `name`, `model_name`, `sql_table_name` (or `derived_table` SQL)
- Whether `default_date` is set
- Number of dimensions, measures, dimension_groups
- Whether identifiers are defined (and their types)
- Whether `description` is populated
- Whether any fields have `zoe_description`, `searchable`, or `synonyms`

**1.2 Read all topic files**
For each topic, document:
- `base_view` and which views are joined
- Whether joins use explicit `sql_on` or rely on identifier matching
- Whether `zoe_description` is populated
- Whether `always_filter` or `access_filters` are defined

**1.3 Read model files**
- Note the `connection` and any `relationships` defined
- Check for `week_start_day` and `timezone` settings
- Flag multiple models pointing to the same connection with no differentiating configuration

**1.4 Verify tables exist and match YAML**
```sql
SELECT * FROM schema.table_name LIMIT 5
```
- Confirm column names in the database match `sql:` expressions in the YAML
- For derived tables, verify the SQL executes without errors
- For Druid connections, verify column casing matches quoted references

### Phase 2: Flag Anomalies

Scan for these specific issues and document them. Do NOT fix them unilaterally — flag them for discussion.

**Structural Anomalies:**
- Views with no `default_date` that contain time-series measures
- Views whose `model_name` references a model that doesn't exist or has no working connection
- Existing topics that should be migrated to model-level `relationships` (flag for migration, not new topic creation)
- Derived table alias mismatches — aliases in SELECT don't match dimension `sql:` references
- Composite key identifiers (`CONCAT(...)`) that may not match actual database columns
- Redundant identifiers that overlap with existing join definitions
- Multiple models with identical connections and no differentiating settings
- Missing model-level `relationships` for non-obvious joins between views

**Content Anomalies:**
- Measures that share natural-language keywords without distinguishing `zoe_description` (e.g., both a dollar margin and percentage margin measure, where only one has a description claiming "Margin")
- Wide fact tables that contain customer/product/region fields but also have dimension table joins defined — Zoë may wander unnecessarily
- Dimensions storing cryptic or technical values (system codes, API object names) that users won't phrase naturally
- Fields with `searchable: true` on high-cardinality columns (>10K distinct values) without `allow_higher_searchable_max`
- `zoe_description` or `description` with stale references to renamed columns or deprecated tables

**Data Quality Signals:**
- Tables with archival snapshots or multiple temporal versions (needs explicit filter guidance)
- Known zero/null patterns (promotional items with $0 price, internal transfers)
- Fiscal calendar usage without fiscal timeframes in dimension_groups

### Phase 3: Prepare Customer Clarification Questions

Based on the anomaly scan, prepare a structured list of questions to ask the customer. Organize by priority.

**High Priority (blocks correct Zoë behavior):**

- **Terminology mappings:** "Field X stores values like `[sample_values]`. What do your users call these? How would they phrase a filter on this field?"
- **Compound calculation definitions:** "I see fields for [field_a], [field_b], [field_c]. How does your team define [business_concept] — which numerator and denominator?"
- **Active vs. legacy domains:** "The workspace has views referencing [schema_a] and [schema_b]. Which schemas contain your active production data?"
- **Fiscal calendar rules:** "Does your organization use a fiscal calendar? If so, what's the fiscal year start date and week pattern (e.g., 4-4-5, 5-4-4)?"
- **Critical filter requirements:** "Does [table] contain multiple snapshots or archival records that require filtering to the latest version?"

**Medium Priority (improves Zoë accuracy):**

- **Disambiguation between related measures:** "You have [measure_a] and [measure_b]. When a user asks about [concept], which should Zoë use by default?"
- **Region/geography groupings:** "How does your team define regions like 'EMEA' or 'North America' in terms of the values in [field]?"
- **Default scoping:** "When users ask about [domain], should Zoë default to a specific time range, status filter, or business unit?"

**Low Priority (nice-to-have refinements):**

- **Drill-down preferences:** "When users drill into a [entity_type], what fields are most useful to see?"
- **Redundant topic consolidation:** "I see [N] similar topics for [domain]. Would it make sense to consolidate into one topic with a filter dimension?"
- **Hidden fields:** "Are there fields in [view] that are internal/technical and should be hidden from Zoë?"

### Phase 4: Minimum Viable Semantic Layer Checklist

After customer clarification, implement the minimum needed for Zoë to work. Apply the "just enough context" principle — don't over-build.

- [ ] Every view with time-series measures has `default_date` set
- [ ] Every view has at least a basic `description` (even one sentence)
- [ ] Primary key dimensions are marked with `primary_key: true`
- [ ] Categorical filter dimensions have `searchable: true`
- [ ] Measures that share keywords have distinguishing `zoe_description`
- [ ] Wide fact tables have `description` guidance to prevent unnecessary joins
- [ ] Critical data quality filters are documented in field-level `zoe_description` (not just view `description`)
- [ ] Non-obvious joins are defined in model-level `relationships` (do NOT create new topics)
- [ ] Unstructured join context (fan-trap warnings, field routing) is in view `description` or system prompt
- [ ] System prompt includes View Selection Guide (if workspace has many views)
- [ ] System prompt includes fiscal calendar rules (if applicable)
- [ ] System prompt includes row limit guardrail for detail queries
- [ ] `.gitignore` excludes CLAUDE.md

### Phase 5: Validate with Test Questions

Write 3-5 test questions that exercise the core data paths:

1. **Simple aggregation with critical filters** — Tests whether Zoë applies required filters (archival dates, status filters)
2. **Cross-table query** — Tests whether Zoë follows the right join path
3. **Time-based trending** — Tests whether Zoë uses the correct date fields (fiscal vs calendar)
4. **Disambiguation** — Tests whether Zoë selects the right measure when multiple candidates exist
5. **Natural language filter** — Tests whether searchable dimensions resolve correctly

Run each question, inspect the generated SQL (not just results), and iterate on the semantic layer based on failures.

---

## Part 16: Clarity → Exploratory Migration

For customers transitioning from Clarity to Exploratory mode.

### Mode Comparison

| Aspect | Clarity Mode | Exploratory Mode |
|--------|-------------|-----------------|
| **Resolution** | Deterministic field compilation from governed model | LLM semantic reasoning over data model context |
| **Output** | Structured JSON query → compiled SQL | Raw SQL generated directly by the LLM |
| **Field discovery** | Precise — maps to exact governed fields | Probabilistic — LLM interprets field names, descriptions, synonyms |
| **Join behavior** | Uses only defined topic joins | May follow identifier chains, foreign keys, or relationships beyond the topic |
| **Sensitivity to descriptions** | Low — descriptions are metadata, not instructions | High — descriptions and system prompts directly guide SQL generation |
| **Error mode** | Field-not-found errors if governed model is incomplete | Plausible but wrong SQL if semantic layer is ambiguous |

### Clarity as a Diagnostic Tool

When Exploratory produces wrong results for a question that Clarity answers correctly, the Clarity output reveals exactly what the semantic layer is missing:

1. **Run the question in Clarity** — Note which fields, filters, and calculations Clarity used.
2. **Run the same question in Exploratory** — Compare table references, WHERE clause, JOINs, and field selections.
3. **Identify the gaps:**
   - **Field routing:** Clarity mapped correctly; Exploratory used a wrong field or joined unnecessarily
   - **Filter values:** Clarity applied correct filter; Exploratory missed it
   - **Date interpretation:** Clarity used fiscal periods; Exploratory used calendar dates
   - **Measure selection:** Clarity used right measure; Exploratory couldn't find it
4. **Enrich the semantic layer** — Add `zoe_description`, `synonyms`, `searchable`, or view `description` guidance
5. **Re-test in Exploratory** — Compare SQL to the Clarity baseline

### Migration Workflow

**Step 1: Inventory the Clarity surface area**
- Identify the top questions users ask (from usage logs, dashboards, or customer interviews)
- Run each in Clarity and save the JSON output as baselines

**Step 2: Test each question in Exploratory**
- Run the same questions and note failures
- For each failure, use the Clarity JSON to diagnose the gap

**Step 3: Enrich the semantic layer iteratively**
- Prioritize by impact: fix the most commonly asked questions first
- Common enrichments needed:
  - Add `zoe_description` to measures that exist but are undiscoverable
  - Add wide-table guidance to view `description` to prevent unnecessary joins
  - Add `searchable: true` and `synonyms` to filter dimensions
  - Add region/geography definitions to system prompt or view `description`
  - Add fiscal calendar rules to system prompt
  - Migrate structured join logic from topics to model-level `relationships`
  - Migrate unstructured topic `zoe_description` content to view `description` or system prompt

**Step 4: Re-test and iterate**
- **Before each batch of enrichments, save baseline SQL from current behavior**
- After enrichments, re-test affected questions in Exploratory
- Compare generated SQL to the Clarity baselines AND to the pre-enrichment baseline (not just results)
- If enrichment causes regression on a previously-working question, diagnose and fix before proceeding

**Step 5: Switch production mode**
- Once the top question set passes in Exploratory, switch the workspace
- Monitor user feedback and Zoë chat logs for new failure patterns

### Key Patterns Observed During Migration Testing

**Pattern 1: Clarity stays on one table; Exploratory wanders**
- Clarity used a single fact table with zero joins; Exploratory joined to dimension tables unnecessarily because the fact table already contained those fields
- Fix: Wide fact table guidance in view description

**Pattern 2: Existing measures are invisible without zoe_description**
- A measure existed with the correct SQL but no `zoe_description`; a related measure claimed the keyword
- Fix: Add `zoe_description` to every measure that could be requested by name

**Pattern 3: Region/geography resolution differs between modes**
- Clarity correctly mapped "EMEA" to a LIKE filter; Exploratory couldn't resolve it
- Fix: Define region groupings explicitly in topic `zoe_description` or system prompt

**Pattern 4: Fiscal vs calendar date confusion**
- Clarity used fiscal period fields; Exploratory defaulted to calendar dates
- Fix: Document fiscal calendar rules prominently

### Migration Is Not All-or-Nothing
- Migration can be incremental — enrich progressively, testing questions as you go
- Start with the most common question types and expand
- Patterns (Part 14) can bridge the gap — historical Clarity-generated queries become indexed patterns that guide Exploratory mode

---

## Part 17: Practical Lessons & Common Pitfalls

These are real-world patterns discovered during customer workspace work. Apply these lessons across all customer engagements.

### Identifier Validation
- **Always verify identifiers against the actual database table.** Run `SELECT * FROM table LIMIT 5` to confirm column names before trusting the YAML.
- **Composite key identifiers are fragile.** If a view defines a computed identifier like `CONCAT(store, '-', order_nbr, '-', date)`, verify the target table actually has a matching column.
- **When a join returns no results, check the join keys first.** The most common cause is mismatched key formats.

### Redundant Identifiers as Latent Risks
- An identifier that points to a real column is still a latent risk if a topic `sql_on` already handles the same join more precisely.
- **Rule: If a topic defines an explicit `sql_on` for a join, remove any overlapping foreign identifiers on the same view.**
- Only keep identifiers that serve a purpose NOT covered by a topic.

### Don't Band-Aid SQL Dialect Issues in the Data Model
- Zoë knows the database warehouse through the `connection` property. SQL syntax errors are Zoë's SQL generation responsibility, not a data model problem.
- **Don't add SQL dialect guidance** to descriptions or system prompts.
- **Exception:** If Zoë persistently selects a raw internal column (e.g., Druid's `__time`) instead of a defined dimension group, system prompt guidance to steer toward the correct field IS appropriate.

### Identifier Cleanup Workflow
1. Catalog all identifiers across view files
2. Cross-reference against topics — flag identifiers whose join is covered by `sql_on`
3. Generate test questions that exercise each join path
4. Run tests and inspect which join path Zoë chose
5. Remove redundant identifiers
6. Re-test after removal

### Multi-Column Joins in Topics
```yaml
views:
  bridge_table:
    join:
      join_type: left_outer
      relationship: many_to_one
      sql_on: >-
        ${base_view.col_a} = ${bridge_table.col_a}
        AND ${base_view.col_b} = ${bridge_table.col_b}
        AND CAST(${base_view.col_c} AS DATE) = CAST(${bridge_table.col_c} AS DATE)
```

### Chained Joins (Bridge Tables)
The second join references the intermediate view, not the base view:
```yaml
views:
  bridge_tbl:
    join:
      sql_on: ${base_view.key} = ${bridge_tbl.key}
  metrics_tbl:
    join:
      sql_on: ${bridge_tbl.order_id} = ${metrics_tbl.order_id}
```

### Topic zoe_description Patterns (Existing Workspaces Only)
> **For new workspaces, do not create topics.** This guidance applies when working with existing workspaces that still have topics.

Write as operational instructions, not marketing copy. Include:
1. **When to use this topic** — what questions it answers
2. **Field disambiguation** — which fields to use (and NOT use) for common queries
3. **NULL/placeholder warnings** — fields that are empty for certain record types
4. **Date field guidance** — which dimension_group generates which columns

Example:
```yaml
zoe_description: |
  Use this topic for questions about [domain].

  FIELD_CATEGORY_1:
  - Use field_a for X. Do NOT use field_b — it contains Y values only.
  - field_c is NULL for [subset]. Use field_d via [bridge_table] instead.

  DATE FIELDS (CRITICAL):
  - [view] does NOT have a column called 'order_date'. Do NOT reference [view].order_date.
  - For date trending on [view], use [dimension_group_name].
```

### Zoë SQL Generation Behavior
- **Zoë discovers fields beyond the topic.** She can follow foreign key chains to tables not explicitly in the topic. Document risky join paths in the topic `zoe_description`.
- **Memories remember the query, not the visualization.** SQL logic and labeling are preserved, but visualization type and Python code are not.

### Data Discovery Workflow
1. Read all view files to understand the table landscape
2. Run `SELECT * LIMIT 5` on key tables to verify columns match YAML
3. Run `SELECT DISTINCT` on categorical fields to discover actual values
4. Check for NULL patterns on key fields across record subsets
5. Test joins with coverage queries before adding to topics

### Derived Table Alias Mismatches
- Aliases in derived table SQL must exactly match column names referenced by dimensions
- This is a silent, latent bug — YAML validation doesn't catch it
- **Audit:** Compare each alias in the derived table SELECT against every `${TABLE}."column_name"` reference in the view's dimensions

### Single-View Topics — Remove During Migration
Topics that wrap a single view with no joins serve no purpose and should be removed during migration. Migrate any `description`/`zoe_description` to the view itself (using `description` since `zoe_description` isn't valid on views). For multi-view topics, use the "Test-Before-Remove Workflow" above to migrate joins to model `relationships`.

### View Selection Guide in System Prompt
When a workspace has many views, add a routing table to the system prompt mapping question domains to specific views. Include negative guidance for cross-domain confusion.

### Row Limits for Detail Queries
- Add a system prompt instruction to include `LIMIT 1001` on detail/row-level queries
- If exactly 1,001 rows return, Zoë should inform the user results were capped
- Alternative: COUNT-first approach for expensive warehouses

### Consolidating Duplicate Topic/View Variants
Consolidate near-identical per-location topics into a single parameterized topic+view where the user filters by dimension. Reduces confusion and maintenance burden.

### Case Sensitivity in Druid Column Quoting
- Druid column names are case-sensitive. SQL references must use quoted names matching exact casing.
- Zenlytic auto-lowercases field `name` properties, but `sql:` must preserve original casing.

### System Prompt as Zoë's Operational Manual
Effective sections (proven patterns):
- **Database Dialect:** Which connections use which SQL dialect
- **View Selection Guide:** Domain-to-view routing table
- **Row Limits:** Performance guardrails for detail queries
- **Time Period Rules:** How to interpret relative date phrases
- **User Attribute Handling:** Instructions to use automatically-provided attributes

Don't put field-level guidance in the system prompt — use `zoe_description` instead.

### Flag Non-Business-Friendly Field Values
During data discovery, flag any dimension whose stored values use non-business-friendly naming. Ask the customer what their users call those values. Once clarified, apply `searchable: true`, `zoe_description` with explicit mappings, and `synonyms`.

### Test-Before-Remove Workflow for Migrating Topics to Relationships
When migrating a topic's joins to model-level `relationships` (see Critical Constraint #3 and #4):
1. Identify the candidate — document the topic's `sql_on` joins and any `zoe_description` content
2. Write a test question that exercises the cross-view join AND field-usage guidance
3. Test with topic in place (baseline) — save results AND SQL
4. Add the equivalent `relationships` entry to the model file, and migrate `zoe_description` content to view `description` or system prompt
5. Remove or hide the topic — compare results AND SQL to baseline
6. Re-test after removal
7. If regression persists — restore the topic and investigate what context was lost

**Critical:** Compare the actual SQL, not just whether results "look right."

### When Migrating a Topic, Audit Its Content
When removing a topic (migrating to relationships), audit its `zoe_description` for three types of content:
1. **Join context** — redundant if identifiers handle it. Safe to drop.
2. **Field-usage guidance** — MUST be migrated to the view `description`.
3. **Domain scoping** — useful for topic selection, irrelevant once topic is gone.

### Passive vs. Forceful Description Language
- Passive language ("Filter by the appropriate category") may be ignored
- Forceful language ("IMPORTANT: you MUST filter by category. Without this filter, results will be incorrect") gets consistent compliance
- When migrating guidance from topics to views, upgrade language intensity
- Effective patterns: `IMPORTANT -`, `CRITICAL -`, `you MUST`, state the negative consequence

### SQL Comparison Is Essential for Validation
Always compare generated SQL — not just results. Two queries can return superficially similar results while one is fundamentally wrong. Check: WHERE clause, JOIN clause, GROUP BY, table references.

### Measure Discoverability — Keyword Conflicts
Every measure that could be requested by name MUST have its own `zoe_description` with explicit keyword routing. For related measures (dollar vs percentage, gross vs net), clearly distinguish when to use each.

```yaml
- name: dsv_margin_usd
  zoe_description: >
    IMPORTANT: Use this measure for any question about margin in dollar terms,
    margin improvement, margin contribution, or margin change between periods.
    For margin as a percentage, use dsv_margin_percentage instead.

- name: dsv_margin_percentage
  zoe_description: >
    Use this measure when asked about margin rate, margin percentage, or
    margin %. For margin in dollar terms, use dsv_margin_usd instead.
```

### Wide Fact Table Awareness
When a view contains all fields needed to answer a question, Exploratory models may still wander to dimension tables via joins. Fix with explicit guidance:

```yaml
description: |
  This view contains all fields needed for sales billing analysis.
  IMPORTANT: Do NOT join to external dimension tables for customer, product,
  region, or date information. All fields are already present on this table.
  Joining to dim_customer, dim_product, or similar tables is unnecessary
  and will produce incorrect results.
```

### Procedural vs Declarative Guidance in Descriptions
- Procedural step-by-step instructions ("Step 1: Identify metrics. Step 2: Run a query.") may not be followed reliably by Exploratory models
- **Declarative guidance about the data** (field routing, filter defaults, date rules) is more reliably followed
- This applies to all description fields — view `description`, field `zoe_description`, and system prompt
- Best practice: Focus descriptions on data-specific rules; use system prompt for workspace-wide behavioral instructions

### View Description Alone Is Insufficient for Critical Filters
- View-level `description` warnings are not reliably read when generating SQL
- **Field-level `zoe_description` on the specific dimension that implements the filter works immediately**
- Why: View descriptions are read at topic selection time; field `zoe_description` is encountered when Zoë writes the WHERE clause
- **Rule:** For data quality rules requiring a specific filter, put guidance on the dimension, not only on the view
- Include exact SQL patterns in `zoe_description` when possible

### Data Model Validation Test Suite
Five tests covering distinct SQL generation capabilities:

1. **Simple Aggregation with Critical Filters** — Tests mandatory filter application. Run first after every data model change.
2. **Cross-Table Comparison** — Tests join correctness and consistent scoping.
3. **Year-over-Year Growth** — Tests fiscal calendar awareness and threshold filtering.
4. **Statistical/Analytical Python** — Tests methodology selection (hardest test).
5. **Multi-Source Data Reconciliation** — Tests source table selection when multiple tables have overlapping data.

Run each on at least two models. Compare SQL across models, not just results. Save baseline SQL from first passing run.

### Orphaned Views Cause Zoë Misrouting
- Deleting topics/models without their views leaves orphans that Zoë routes to
- Topics, models, and views form a three-layer stack — audit all together
- **Never delete without customer approval.** `hidden: true` is a safer interim step.

### View Descriptions for Unstructured Join Context (Preferred Approach)
For new workspaces, unstructured join context belongs in view `description` fields (and system prompt for workspace-wide rules) rather than topics. Write join guidance as: table name, ON condition with both column names. Example: `JOIN to DIM_PRODUCT ON FK_PRODUCT_ID = PK_PRODUCT_CD`. For non-obvious join keys where column names don't match, also add the structured join to model-level `relationships` — don't rely solely on description text for these cases. Always test with a question that forces the join to verify Zoë follows the guidance.

### Don't Assume Measures Are Needed
- Zoë can improvise simple aggregations (SUM, COUNT, AVG) from numeric dimensions
- The risk area is compound ratio measures (e.g., margin = operating profit / sales) with multiple valid options
- Don't create measures proactively — flag compound calculations for customer clarification
- Alternatives: Memories, view `description`, field-level `zoe_description`

### Dead Domain Cleanup
1. Identify which domains have working data (`SELECT * LIMIT 5`)
2. Flag all three layers (topics, models, views) for dead domains
3. Present findings to customer — let them decide
4. Consolidate redundant models with identical connections and no differentiating settings

---

## Documentation Sources

### Data Modeling
- [Views](https://docs.zenlytic.com/data-modeling/view)
- [Dimensions](https://docs.zenlytic.com/data-modeling/dimension)
- [Measures](https://docs.zenlytic.com/data-modeling/measure)
- [Dimension Groups](https://docs.zenlytic.com/data-modeling/dimension_group)
- [Topics/Joins](https://docs.zenlytic.com/data-modeling/topic)

### Zoë & Tips
- [Zoë AI Analyst](https://docs.zenlytic.com/zenlytic-ui/zoe)
- [Zoë Tips & Tricks](https://docs.zenlytic.com/tips-and-tricks/zoe_tips_and_tricks)
- [Zoë Context Ingestion](https://docs.zenlytic.com/tips-and-tricks/zoe_context_ingestion)
- [Zoë Memories](https://docs.zenlytic.com/zenlytic-ui/memories)
- [Entity Drills](https://docs.zenlytic.com/tips-and-tricks/entity-drills)
- [Time Metrics](https://docs.zenlytic.com/tips-and-tricks/time-metrics)
- [Data Indexing](https://docs.zenlytic.com/tips-and-tricks/data-indexing)
- [Patterns](https://docs.zenlytic.com/zenlytic-ui/patterns)
- [Attachments](https://docs.zenlytic.com/zenlytic-ui/attachments)

### UI & Administration
- [Data Model Editor](https://docs.zenlytic.com/zenlytic-ui/data_model_editor)
- [User Attributes](https://docs.zenlytic.com/zenlytic-ui/user_attributes)
- [User Roles](https://docs.zenlytic.com/zenlytic-ui/user_roles)
- [Workspace Groups & Permissions](https://docs.zenlytic.com/zenlytic-ui/workspace_groups_and_permissions)
- [Workspace Manager](https://docs.zenlytic.com/zenlytic-ui/workspace-manager)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 3.0 | 2026-03-02 | Major restructure: Moved Critical Constraints to top of document for immediate visibility. Added Critical Constraint #3: Topics are being deprecated — avoid creating new ones; use model-level `relationships` for structured joins, system prompt/view description/field `zoe_description` for unstructured context (per CTO guidance). Added Critical Constraint #4: Always run before/after tests on major changes — capture baseline SQL before changes, compare after. Added Part 15 (Greenfield Exploratory Workspace Setup) with phased approach: import/analyze views, flag anomalies, prepare customer clarification questions, minimum viable checklist, validation testing. Extracted Part 16 (LLM Model Considerations) into separate `LLM_Model_Testing_Reference.md` for internal use — reduces CLAUDE.md context footprint. Updated Part 6 (Topics) with deprecation notice. Updated Part 7 join architecture to reflect two-location model: structured → model `relationships`, unstructured → descriptions/system prompt. Reframed topic-related lessons in Part 17 as migration guidance. Removed stale Sonnet 4.6 "early release" hedging. Renumbered parts. |
| 2.5 | 2025-02-27 | Added "just enough context" guiding principle from CTO guidance. Added Part 15 lesson "Be Conservative — Fix What's Broken, Don't Over-Enrich." Moderated orphan cleanup to require customer approval. |
| 2.4 | 2025-02-25 | Corrected model-level relationships schema to match CTO canonical spec. Validated via SBD Production workspace migration. |
| 2.3 | 2025-02-24 | Added 4 new Part 15 lessons from Eaton workspace migration. Added count measure filter platform constraint. |
| 2.2 | 2025-02-23 | Rewrote Part 16 "Statistical & Analytical Capability" with production evidence from price elasticity testing. |
| 2.1 | 2025-02-23 | Added view description insufficiency lesson. Added Data Model Validation Test Suite. |
| 2.0 | 2025-02-23 | Added Patterns, Attachments, Clarity vs Exploratory migration. Corrected Clarity mode framing. |
| 1.8 | 2025-02-21 | Major Part 16 rewrite: four-model tier framework. |
| 1.7 | 2025-02-20 | Added Part 16: LLM Model Considerations. |
| 1.6 | 2025-02-20 | Added CLAUDE.md exclusion rule. |
| 1.5 | 2025-02-20 | Added 5 new Part 15 lessons from production engagement. |
| 1.4 | 2025-02-18 | Added flag non-business-friendly field values lesson. |
| 1.3 | 2025-02-17 | Added 8 new Part 15 lessons. |
| 1.2 | 2025-02-13 | Added redundant identifiers, SQL dialect, identifier cleanup lessons. |
| 1.1 | 2025-02-13 | Reframed Part 7 for exploratory-only Zoë. |
| 1.0 | 2025-02-12 | Initial complete KB — Parts 1-15. |
