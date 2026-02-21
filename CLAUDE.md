# Zenlytic Zoë Data Model - Complete Knowledge Base

**Version:** 1.8 | **Last Updated:** 2025-02-21

Use this document when working with Zenlytic customer workspaces via Git repositories. This covers Git operations, YAML schema, joins, dimensions, measures, and how Zoë (the AI analyst) uses the semantic layer.

> **Version Note:** This KB is versioned. Check the changelog at the bottom for what changed between versions. When making updates, increment the version (minor changes: 1.1 → 1.2, major restructures: 1.x → 2.0) and add a changelog entry.

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
# Clone the repo
git clone https://github.com/Zenlytic/<repo-name>.git

# Checkout specific branch
cd <repo-name>
git fetch origin
git checkout <branch-name>
git pull origin <branch-name>
```

#### Copy Content Between Repos/Branches
```bash
# Clone target repo
git clone https://github.com/Zenlytic/<target-repo>.git

# Remove existing content (keep .git folder)
cd <target-repo>
rm -rf models views topics dashboards zenlytic_project.yml

# Copy from source repo (exclude .git)
cp -r /path/to/source-repo/models /path/to/source-repo/views /path/to/source-repo/topics /path/to/source-repo/zenlytic_project.yml /path/to/target-repo/

# Stage, commit, push
git add -A
git commit -m "Description of changes"
git push origin <branch>
```

#### Search for References
Use Grep tool: `Grep pattern="search_term" path="/path/to/repo"`

### Branch Naming Conventions
- `data-sci-<name>` - Data scientist working branches
- `master` / `main` - Production branch

### Key Commands Summary

| Action | Command |
|--------|---------|
| Clone repo | `git clone <url>` |
| Switch branch | `git checkout <branch>` |
| Update branch | `git pull origin <branch>` |
| Check status | `git status` |
| Stage all | `git add -A` |
| Commit | `git commit -m "message"` |
| Push | `git push origin <branch>` |
| Search content | Use Grep tool with pattern |

### Git Tips
- Always fetch before checkout to get latest remote branches
- Check git status before committing to review changes
- Use descriptive commit messages explaining what was migrated/changed
- When copying between repos, the .git folder should NOT be copied (it contains repo-specific history)

---

## Part 2: Project Configuration

### zenlytic_project.yml
```yaml
name: project_name              # Project name
profile: connection_profile     # Database connection profile
version: 1

# Directory paths
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
description: "View description"           # Optional: Shown in UI and to ZoE
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
  description: "Description text"         # Shown in UI and to ZoE
  zoe_description: "AI-specific context"  # Overrides description for ZoE only
  group_label: "Sidebar Group"            # Groups fields in sidebar
  hidden: false                           # Hide from UI (still referenceable)

  # Optional behavior properties
  searchable: true                        # Enable natural language search indexing (see Part 14)
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

  # Optional display properties
  label: "Total Revenue"
  description: "Sum of all revenue"
  zoe_description: "Total sales revenue including all channels"
  group_label: "Revenue Metrics"
  hidden: false
  value_format_name: "$#,##0.00"

  # Optional behavior properties
  synonyms: [total_sales, income, sales_total]

  # Optional filtering
  filters:
    - field: is_cancelled
      value: "No"

  # Optional advanced
  canon_date: order_at                    # Override default trending date
  extra:
    metric_type: revenue
```

### Measure Types
| Type | Description | Notes |
|------|-------------|-------|
| `sum` | Sum values | Basic aggregation |
| `average` | Calculate mean | Basic aggregation |
| `count` | Count rows | Basic aggregation |
| `count_distinct` | Count unique values | Basic aggregation |
| `max` | Maximum value | Basic aggregation |
| `min` | Minimum value | Basic aggregation |
| `median` | Median value | Database-dependent |
| `sum_distinct` | Sum without duplication | Requires `sql_distinct_key` |
| `average_distinct` | Average unique values | Requires `sql_distinct_key` |
| `cumulative` | Aggregate over time | Requires `measure` property |
| `number` | Static/calculated value | No aggregation |

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

Define primary and foreign keys for automatic join discovery:

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
    join_as: billing_address              # Alias for this join path
    join_as_label: "Billing Address"      # Display label
    join_as_field_prefix: billing_        # Prefix for joined fields

  - name: shipping_address_id
    type: primary
    sql: ${address_id}
    join_as: shipping_address
    join_as_label: "Shipping Address"
    join_as_field_prefix: shipping_
```

---

## Part 6: Topics (Join Configuration)

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

# Optional filters
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
| `many_to_one` | Multiple base records -> one joined record |
| `one_to_many` | One base record -> multiple joined records |
| `one_to_one` | One-to-one match |
| `many_to_many` | Multiple on both sides |

### Joins Best Practice
- Don't join views only because it's possible to join them — separate topics can be queried independently with results merged afterward
- Topics are the primary organizational mechanism for joins
- Default joining: If a view in a topic lacks explicit join configuration, pre-existing foreign key relationships (identifiers) are automatically applied

---

## Part 7: How Zoë Uses the Semantic Layer

Zoë (Zenlytic's AI analyst) writes SQL to answer user questions, using the semantic layer as context to understand the data model.

### How Zoë Works

| Aspect | How Zoë Operates |
|--------|-----------------|
| **Process** | User prompt → LLM → SQL query → Results |
| **SQL Generation** | Non-deterministic — the same prompt may yield different SQL each run |
| **Semantic Layer** | Used as **context** for the LLM to generate SQL (field definitions, joins, descriptions, etc.) |
| **Join Logic** | LLM interprets topic definitions, relationships, and identifiers to construct joins |
| **Permissions** | Row and column-based access controls are enforced by the system (see Permissions section below) |

> **Note (Legacy):** An older mode called "Clarity" used deterministic SQL compilation from a query JSON intermediate step. This mode is being phased out. All guidance in this document is written for Zoë's current SQL-generation behavior.

### How Zoë Sees Join Information

**Topics**: Zoë sees topics and their associated joins. She uses topic join definitions to inform the SQL she writes.

**Warning**: A potential issue can arise if the topic implies illogical or counter-intuitive joins. For example, if a topic says it's valid to make many-to-many joins between large fact tables, Zoë may be misled into thinking it's a safe join when it will actually cause a fan-out.

**Best Practice**: Any `one_to_many` or `many_to_many` join in a topic *can* potentially degrade performance — but it won't necessarily if the join makes logical sense to Zoë. For example, a one-to-many from `orders` to `discounts` is intuitive and unlikely to cause issues, while a one-to-many between obscure tables with non-obvious names (e.g., `tbl_fact_txn_ext` to `tbl_dim_entity_ext`) will confuse Zoë because it can't reason well about the actual concepts. The key factor is whether Zoë can understand what the join represents, not just the cardinality.

**Identifiers**: Zoë sees identifiers (primary keys, foreign keys, and join-type identifiers). Identifiers provide join context alongside topics and model-level relationships.

**Relationships**: The `relationships` property on the model is how the Context Manager in Self-Service stores joins. This is a single YAML property that stores all relationships between tables. **Zoë sees all of these.** Along with identifiers and topics, relationships provide another way to define join logic for Zoë.

**Where to Store Joins (Recommended)**: Non-obvious joins should be stored using **relationships** — this is the preferred location for structured join logic. Obvious joins (e.g., matching foreign key names between tables) don't need to be stored anywhere. In addition to relationships, add plain-text context in view-level `description` fields to explain valid or invalid join paths when Zoë makes mistakes in certain scenarios but not others.

**Coming Soon — Context Manager**: The Context Manager will unify all join management under the `relationships` concept, making the current multi-location approach (identifiers, topics, relationships) simpler. Once fully rolled out, all joins will be managed through relationships, eliminating the need to decide where to put join context.

### Zoë Permissions

Zoë writes raw SQL, but **access controls are enforced by the system** — row-level permissions (`access_filters`) and column-level permissions are applied regardless of the query Zoë generates.

**For Admin Users:**
- By default, admins can query any table the SQL role has access to (same as using the SQL editor)
- There is a toggle to **"Enforce permissions for admins"** — when enabled, admins get the same restrictions as Explore-level users
- With the toggle OFF, Zoë can try to query tables even if she doesn't know about them (if the role has access)

**For Explore-Level Users:**
- Permissions from the semantic layer are enforced:
  - `always_filter` — Always applied
  - `access_filters` — Row-based access control
  - Column-based access control
  - Table access restrictions

**Important**: While access controls are enforced by the system, Zoë (the LLM) is responsible for constructing valid queries — field existence, join validity, correct SQL syntax, etc. Descriptions, topic `zoe_description`, and system prompts are the primary tools for guiding Zoë toward correct queries.

### The Importance of Prompts for Join Context

**Critical Insight**: Just as important (or more important) than structured join definitions are the **prompts on the view or system level** that explain to Zoë:
- Concepts around the joins
- Potential fan traps
- Other scenarios that could cause problems

Use `description` at the view level and `zoe_description` at the field level and topic level to provide this guidance. Note: `zoe_description` is NOT a valid view-level property — only `description` is valid on views.

### What Zoë Ingests

**Context Sources (Zoë sees all of these and weighs them based on relevance):**
- **Custom System Prompt** — Domain-specific knowledge, business rules, terminology, calculation methodologies
- **Topics** — Join structure, relationships, and topic-level `description`/`zoe_description`
- **Views, Dimensions, Measures** — Field definitions, types, `description`, and `zoe_description` (latter takes precedence)
- **Identifiers** — Primary and foreign key definitions on views
- **Memories** — User-marked successful responses and desired answer patterns (see Part 9)
- **Relationships** — Model-level relationship definitions
- **Synonyms** — Alternative names for natural language queries
- **Searchable Fields** — Indexed categorical values for NLP discovery (see Part 14)
- **Access Controls** — Zoë cannot see or reference data that users don't have permission to view

### How Zoë Weighs Context Sources

Zoë sees all context sources simultaneously and uses them based on relevance to the query. There is no strict hierarchy — the model weighs each source as it sees fit depending on the question being asked. That said, each source type has a natural role:

| Context Source | Role |
|--------------|--------|
| **Custom System Prompt** | Organization-wide rules, terminology, calculation logic |
| **Structural Relationships** | Topics, joins, identifiers, topic descriptions |
| **Field & View Descriptions** | `zoe_description` (if set) overrides `description` for Zoë |
| **Memories** | Reinforced response patterns from previous queries |

### When to Use Each Context Type

| Context Type | Best For | Use When |
|---|---|---|
| **Custom System Prompt** | Company terminology, business rules, calculation methodologies | Need organization-wide consistency across all Zoë interactions |
| **`zoe_description`** (field/topic level) | AI-specific guidance that differs from user-facing copy | Standard `description` is too technical or misleading for the AI |
| **`description`** (view/field/topic level) | Business context explaining how data should be used | Applies to both users in the UI and Zoë |
| **Memory** | Reinforcing desired response patterns for specific questions | User confirms Zoë's answer is correct and wants it repeated |
| **Topic Structure** | Understanding data connections and analytical context | Need Zoë to understand which fields relate to each domain |

**Key Guidance:**
- Write **rich descriptions** with business context, not just technical definitions (e.g., "Total revenue expected from a customer over their entire relationship" vs bare field name)
- Use **clear semantic names** like `monthly_recurring_revenue` instead of abbreviations (`mrr`)
- **Document edge cases** — note data quality issues, calculation nuances, and special handling in descriptions
- **Review memories regularly** in Settings → Memory to remove undesired or outdated patterns

### Zoë-Specific Properties

| Property | Valid On | Purpose |
|----------|----------|---------|
| `zoe_description` | dimensions, measures, dimension_groups, topics | AI-specific context (overrides `description` for Zoë). **NOT valid on views.** |
| `description` | views, dimensions, measures, dimension_groups, topics | Shown in UI and sent to Zoë. On views, this is the only way to provide Zoë context. |
| `synonyms` | dimensions, measures, dimension_groups | Alternative names for NLP matching |
| `searchable: true` | dimensions | Index field categories for search |
| `hidden: true` | views, dimensions, measures, dimension_groups, topics | Exclude from Zoë's available fields |
| `tags` | dimensions, measures | Special identifiers for semantic meaning |

### Guiding Principle for Data Model Quality

> **"Could a talented data analyst answer questions using this data model on their first day?"**

This is the foundational test for every data model. If the naming, descriptions, and structure would confuse a skilled analyst who is new to the domain, Zoë will also struggle. Apply this lens when naming fields, writing descriptions, and organizing topics.

### Best Practices for Zoë

1. **Use `zoe_description`** on fields (dimensions, measures, dimension groups) and topics for AI-specific context that differs from user-facing descriptions. **Do NOT use `zoe_description` on views** — use `description` instead.
2. **Use `description`** on views to provide join guidance and context to Zoë (view `description` is sent to both the UI and Zoë)
3. **Add synonyms** for fields with multiple common names
4. **Set `searchable: true`** on categorical dimensions users will filter on (see Part 14 for indexing limits and `allow_higher_searchable_max`)
5. **Write clear topic descriptions** explaining analytical purpose (both `description` and `zoe_description` are valid on topics)
6. **Use meaningful, distinguishing field names** — Zoë needs to tell similar fields apart. Use "Gross Revenue" and "Net Revenue" instead of "Revenue 1" and "Revenue 2". Name fields the way a data analyst would describe them to a colleague.
7. **Hide technical fields** that shouldn't appear in queries
8. **Document fan trap risks** in view-level `description` to warn about dangerous joins
9. **Avoid many-to-many joins** in topics unless the join makes clear logical sense to Zoë (see "How Zoë Sees Join Information" above)
10. **Store non-obvious joins in model-level `relationships`** (preferred) — obvious joins don't need to be stored anywhere. Supplement with plain-text context in view descriptions for edge cases. Zoë sees identifiers, topics, and relationships.
11. **Add system-level prompts** explaining join concepts and potential pitfalls
12. **Set `default_date`** on every view with time-series measures, and use `canon_date` on measures that trend by a different date (see Part 13)
13. **Configure entity drills** with `drill_fields` on primary key dimensions to enable contextual navigation (see Part 12)
14. **Use Zoë Memories** for company-specific data interpretation patterns — create them from chat or the Memory Portal (see Part 9)
15. **Maintain consistent database capitalization** in searchable fields — Zoë applies indexed values directly to filter conditions
16. **Design for model-agnostic robustness** — different users may use different LLM backends; build the semantic layer so it works across all supported models, not just the most capable one (see Part 16)

### Zoë AI Data Analyst Details

**Core Functionality:**
- Zoë is an AI data analyst that interprets user questions and generates answers through data analysis
- Operates on an "agentic architecture" that enables planning, tool usage, and memory development
- Users can see exactly what steps she is taking, the reasoning behind her thinking, and the data she queries

**Semantic Layer Integration:**
- Zoë searches across the governed measures and dimensions to use existing fields and know when to create new ones
- The system respects governance by preventing automatic modifications to the global cognitive layer
- Dynamic Fields: Zoë can create user-specific measures and dimensions (for Develop+ roles when enabled), but cannot promote them to the global model — users must handle promotion manually through the UI

**Configuration Options:**
- Model Selection: Users switch between LLM models mid-conversation via dropdown (see Part 16 for how model choice affects SQL generation quality and data model design)
- Input Methods: Voice transcription (`cmd + i` to toggle recording), file attachments (images, CSVs, PDFs — limit 5)
- Integration: Slack (@Zenlytic tagging required) and Microsoft Teams support

### Example Zoë-Optimized View with Join Warnings
```yaml
version: 1
type: view
name: order_lines
model_name: my_model
sql_table_name: prod.order_lines
# NOTE: zoe_description is NOT valid at the view level. Use description instead.
# The view description is sent to both the UI and ZoE.
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
    # zoe_description IS valid on dimensions, measures, and dimension groups
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

# Model-level relationships provide join context to ZoE alongside identifiers and topics
relationships:
  - name: orders_to_customers
    from_view: orders
    to_view: customers
    type: many_to_one
    sql_on: ${orders.customer_id} = ${customers.customer_id}

  - name: orders_to_order_lines
    from_view: orders
    to_view: order_lines
    type: one_to_many
    sql_on: ${orders.order_id} = ${order_lines.order_id}

  - name: order_lines_to_products
    from_view: order_lines
    to_view: products
    type: many_to_one
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
- Semantic models are automatically mapped to Zenlytic views
- Both dbt measures and metrics map to Zenlytic measures
- Dimensions are converted with appropriate time granularities
- Join logic and entity relationships are preserved and mapped to topics
- Field descriptions transfer from dbt to provide business context
- Set `use_default_topics` to `false` in `zenlytic_project.yml` for custom topics beyond MetricFlow's capabilities

---

## Part 9: Zoë Memories

Memories allow users to train Zoë with company-specific knowledge, data preferences, and desired response patterns. Zoë automatically retrieves relevant memories when answering questions.

### Creating Memories

**From Chat:** Click "Add to memory" beneath a helpful Zoë response. Zoë will use this response pattern for similar future questions.

**From Memory Portal (Settings → Memory):**
- **Text Memory**: Pair a user question with the desired Zoë response
- **Explore Memory**: Configure questions with specific time ranges, metrics, filters, and slices, then preview before saving

### Managing Memories
Access via Settings → Memory to search, edit, or delete existing memories.

### Key Rules
- Memories work for **data interpretation only**, not personality/behavioral changes
- For tone or style adjustments, use custom system prompts instead
- Memories are opt-in only — Zoë captures only explicitly saved content
- Zoë does not learn from negative feedback or conversation history automatically
- Zoë sees memories alongside all other context sources (system prompt, topics, descriptions) and weighs them based on relevance (see Part 7)

### When to Use Memories vs Other Context Types

| Scenario | Use This |
|----------|----------|
| Company-wide business rules or terminology | **Custom System Prompt** (affects all queries) |
| AI-specific guidance on a field that differs from the UI label | **`zoe_description`** on the dimension/measure |
| General business context for a table or field | **`description`** on the view or field |
| Reinforcing a correct answer pattern for a specific question type | **Memory** (best for fine-tuning specific responses) |
| Join guidance or fan-trap warnings | **`description`** on the view + **`zoe_description`** on the topic |

**Rule of thumb:** Use the system prompt for broad rules, descriptions/zoe_descriptions for field-level and join-level context, and memories for fine-tuning specific answer patterns after Zoë gets them right.

---

## Part 10: Data Model Editor

The Data Model Editor enables teams to modify the Cognitive Layer (views, topics, measures, dimensions) directly within the Zenlytic UI.

### Workflow
1. **Create a branch** — Zenlytic will not allow editing the production branch live
2. **Edit YAML files** — Define metrics, joins, access controls directly in the browser
3. **Real-time validation** — Errors and warnings display in a left-side panel with pinpointed locations
4. **Deploy to production** — Click the green "Deploy to Production" button to merge and update Zoë

### Adding Tables
1. Click the "+" button
2. Select "Create View from Table"
3. Choose database and table
4. Click "Create View" — auto-generates boilerplate YAML

### Deployment Notes
- After deploying, wait ~10 seconds for Zoë to ingest the updated semantic layer
- Branch-based editing protects production stability
- All changes go through the same YAML schema documented in this knowledge base

---

## Part 11: User Attributes, Roles & Permissions

### User Attributes

User attributes are data points assigned to users that drive access controls and row-level security.

**Setup:** Workspace Settings → Team Members → Attributes

**Assignment levels:**
- Individual users
- User groups

**How they work with access controls:**
- Attributes are matched against `access_filters` and `required_access_grants` in views/topics
- If a user's attribute value doesn't match the allowed values in an access grant, they lose access to the restricted data

**Reserved/Special Attributes:**

| Attribute | Behavior |
|-----------|----------|
| `email` | Auto-populated with logged-in user's email; cannot be overridden |
| `zenlytic_connection_name` | Overrides which credential/connection executes queries |
| `zenlytic_connection_database` | Overrides the database selection (behavior varies by warehouse — applies to Postgres, MySQL, Redshift, Azure Synapse, SQL Server, Trino, DuckDB, Snowflake; does NOT apply to BigQuery, Druid, or Databricks) |

### User Roles

Zenlytic has 8 role types with 13 granular permissions:

| Role | Key Characteristics |
|------|-------------------|
| **Admin** | All permissions — full workspace control |
| **Develop** | All admin permissions except workspace settings |
| **Develop without Deploy** | Cannot push data model changes to production |
| **Explore** | Most common role — content, download, analytics, Zoë chat access |
| **View** | Explore minus unlimited downloads |
| **Restricted** | View content only — requires data access controls for proper security |
| **Embed** | Specialized embedded user role |
| **Embed with SQL** | Embedded with SQL query access |
| **Embedded with Scheduling** | Embedded with dashboard scheduling |

**Permission Categories:**
- **Content Management**: save, view, schedule dashboards
- **Data Access**: download (with/without limits), run SQL, see underlying SQL
- **Exploration**: explore from dashboards, chat with Zoë, create personal fields
- **Administration**: edit workspace settings, manage data model, deploy to production, change branches

**Key permissions for data model work:**
- `data_model_edit` — Can modify views/topics/measures in the Data Model Editor
- `deploy_to_production` — Can merge branch to production
- `edit_settings` — Can manage workspace configuration

### Workspace Groups & Permissions

Groups control dashboard-level access:
1. **Create groups** in Workspace Settings → Team Members
2. **Assign members** to groups
3. **Set dashboard permissions** per group via the Share button on each dashboard

This two-tier approach (workspace groups + dashboard permissions) avoids reconfiguring membership for each dashboard.

---

## Part 12: Entity Drills

Entity drills define groupings of related fields that represent business concepts (products, users, transactions, etc.). When configured, they create "Drill into [entity]" dropdown options on visualizations.

### Configuration
Use the `drill_fields` property on a dimension:

```yaml
- name: sales_rep_id
  field_type: dimension
  type: string
  sql: ${TABLE}.SALES_REP_ID
  primary_key: true
  tags: ['Sales Rep']
  drill_fields: [first_name, last_name, email, status]
```

### Best Practices
- Group logically related fields users would want to see together
- Use meaningful entity `tags` to organize the data model
- Include identifying information (IDs, names, contact details) as drill fields
- Apply drills at the primary key level to maximize utility across queries
- Drill fields enable Zoë to understand entity relationships and provide contextual navigation

---

## Part 13: Time Metrics (canon_date & default_date)

Time context is critical for trending and period-over-period analysis. Zenlytic uses two properties to establish temporal context for metrics.

### default_date (View Level)
Sets the default time dimension for all metrics in a view:

```yaml
version: 1
type: view
name: orders
default_date: created_at                  # References a dimension_group name
```

All measures in this view will trend by `created_at` unless overridden.

### canon_date (Measure Level)
Overrides the view's `default_date` for a specific measure:

```yaml
- name: new_subscriptions
  field_type: measure
  type: count
  sql: ${TABLE}.id
  # Uses view's default_date (created_at)

- name: canceled_subscriptions
  field_type: measure
  type: count
  sql: ${TABLE}.id
  canon_date: canceled_at                 # Trends by cancellation date, not creation date
  filters:
    - field: is_canceled
      value: "Yes"
```

### Cross-Table Dates
You can reference time dimensions from other views using dot notation:

```yaml
default_date: google_record_stats.recorded_at
```

This requires a joinable relationship between the views.

### Key Rules
- Both `default_date` and `canon_date` reference the `name` of a `dimension_group` field (type: time)
- `canon_date` on a measure takes precedence over the view's `default_date`
- If neither is set, Zoë may not know how to trend the metric over time
- Always set `default_date` on views that contain time-series measures

---

## Part 14: Data Indexing (searchable Fields)

Data indexing controls which categorical values Zoë can discover and use for natural language filtering. Indexing is opt-in to protect privacy and prevent indexing sensitive data.

### How It Works
Setting `searchable: true` on a dimension allows Zoë to index the distinct values in that column (up to 10,000 categories by default). This enables queries like "show me orders where status is 'shipped'" — Zoë knows "shipped" is a valid value.

### Configuration
```yaml
- name: status
  field_type: dimension
  type: string
  sql: ${TABLE}.status
  searchable: true
```

### High-Cardinality Columns
When a column exceeds 10,000 distinct values, indexing is skipped entirely. To increase the threshold to 500,000:

```yaml
- name: product_sku
  field_type: dimension
  type: string
  sql: ${TABLE}.product_sku
  searchable: true
  allow_higher_searchable_max: true
```

### Best Practices
- **Explicitly set `searchable: true`** on categorical dimensions users will filter on (status, category, region, channel, etc.)
- **Don't set searchable on high-cardinality IDs** (order_id, customer_id) unless needed — they bloat the index
- **Maintain consistent database capitalization** — Zoë applies indexed values directly to generated filters, so "Active" vs "active" matters
- **Use `allow_higher_searchable_max`** sparingly and only when the business need requires it
- **Don't index sensitive fields** (SSN, email, etc.) — indexing makes values discoverable through Zoë

---

## Quick Reference: SQL Expression Syntax

| Syntax | Meaning |
|--------|---------|
| `${TABLE}` | Current table |
| `${field_name}` | Field in same view |
| `${view_name.field_name}` | Field in another view |
| `{{value}}` | Dynamic value (for links) |

---

## New Customer Setup

When creating a new customer folder under Zenlytic-Customers/:
1. Create the customer subfolder
2. Clone the customer's Git repository into the subfolder
3. Create a customer-specific CLAUDE.md with customer name, repo URL, branch info, and any special notes
4. **Add a `.gitignore` to the customer repo** that excludes `CLAUDE.md` — see rule below
5. The shared knowledge in this parent CLAUDE.md is automatically inherited

### CRITICAL: Never Commit CLAUDE.md to Customer Repositories

**CLAUDE.md files must NEVER be committed to a customer's Git repository.** These are internal working files for Claude Code and must not appear in the Zenlytic Data Model Editor, be visible to customers, or be ingested by Zoë.

**The canonical CLAUDE.md lives in the `zoe-data-modeling-kb` repository** — that is where it is tracked, versioned, and shared. Customer-specific CLAUDE.md files live on disk in the customer subfolder but are excluded from the customer's Git repo.

**Required setup for every customer repo:**
1. Add a `.gitignore` file at the repo root containing `CLAUDE.md`
2. If CLAUDE.md was previously committed, remove it from tracking: `git rm --cached CLAUDE.md`
3. Commit the `.gitignore` (and the CLAUDE.md removal if applicable)

**When committing changes to a customer repo**, never use `git add -A` or `git add .` without first verifying that CLAUDE.md is in `.gitignore`. Prefer adding specific files by name.

---

## Part 15: Practical Lessons & Common Pitfalls

These are real-world patterns discovered during customer workspace work. Apply these lessons across all customer engagements.

### Identifier Validation
- **Always verify identifiers against the actual database table.** View YAML files may define identifiers (e.g., composite keys via `CONCAT(...)`) that don't exist as actual columns. Run `SELECT * FROM table LIMIT 5` to confirm column names before trusting the YAML.
- **Composite key identifiers are fragile.** If a view defines a computed identifier like `CONCAT(store, '-', order_nbr, '-', date)`, verify the target table actually has a matching column. Often the table has separate columns instead, requiring a multi-column join in the topic rather than a single identifier match.
- **When a join returns no results, check the join keys first.** The most common cause is mismatched key formats or non-existent columns, not missing data.

### Redundant Identifiers as Latent Risks
- **An identifier doesn't have to be actively wrong to be a problem.** Even if it points to a real column with the correct type, a foreign identifier is a latent risk if a topic `sql_on` already handles the same join more precisely.
- **Identifiers remain auto-discoverable by Zoë** for queries outside the topic context or on edge cases. A redundant identifier gives Zoë a second, less-controlled join path that it might choose unpredictably.
- **Rule: If a topic defines an explicit `sql_on` for a join, remove any overlapping foreign identifiers on the same view.** The topic join is more precise (multi-column, CAST-aware) and controlled. Keeping both creates ambiguity.
- **Only keep identifiers that serve a purpose NOT covered by a topic** — e.g., simple single-key joins used across multiple topics, or primary keys that identify the view's grain.

### Don't Band-Aid SQL Dialect Issues in the Data Model
- **Zoë knows the database warehouse** through the `connection` property on the model (e.g., `connection: my_connection` → Databricks). She has access to the connection config and knows what SQL dialect to use.
- **SQL syntax errors** (date functions, type casting, dialect-specific functions) are Zoë's SQL generation responsibility, not a data model problem.
- **Don't add SQL dialect guidance** to `zoe_description`, view `description`, or system prompts — it clutters the semantic layer with implementation details that Zoë should already know.
- **Fix at the root:** If Zoë consistently generates bad SQL for a specific warehouse, that's a Zoë platform issue to report, not a data model issue to work around.
- **Exception — Column selection vs syntax:** The above applies to SQL *syntax* (date functions, CAST, etc.). However, if Zoë persistently selects a raw internal column (e.g., Druid's `__time`) instead of a properly defined dimension group, that's a *column navigation* problem, not a dialect problem. System prompt guidance to steer Zoë toward the correct field is appropriate in this case. Be explicit and forceful — generic guidance like "don't use raw columns" is often insufficient. Name the specific column to avoid and the specific dimension group to use instead.

### Identifier Cleanup Workflow
Reusable pattern for auditing any customer's identifiers:
1. **Catalog all identifiers** — Read all view YAML files and list every identifier (name, type, sql expression)
2. **Cross-reference against topics** — Flag any identifier whose join is also covered by a topic `sql_on`
3. **Generate targeted Zoë test questions** — Write questions that exercise each join path (basic counts, cross-table lookups, chained joins, potential fan-out scenarios)
4. **Run tests and inspect the SQL** — Check not just whether results are correct, but which join path Zoë chose (topic `sql_on` vs identifier auto-discovery)
5. **Remove redundant identifiers** — Remove foreign identifiers where topic `sql_on` handles the same join
6. **Re-test after removal** — Confirm no regressions; the topic join should still work identically

### Multi-Column Joins in Topics
When tables don't share a single join key, use explicit `sql_on` with multiple conditions in the topic:
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
Note: Use `CAST(...AS DATE)` when one side is a timestamp and the other is a date to ensure the join matches.

### Chained Joins (Bridge Tables)
When a topic needs to traverse multiple tables (e.g., base → bridge → metrics), the second join references the intermediate view, not the base view:
```yaml
views:
  bridge_tbl:
    join:
      sql_on: ${base_view.key} = ${bridge_tbl.key}
  metrics_tbl:
    join:
      sql_on: ${bridge_tbl.order_id} = ${metrics_tbl.order_id}
```

### Topic `zoe_description` Patterns
Write topic `zoe_description` as operational instructions, not marketing copy. Include:
1. **When to use this topic** — what questions it answers
2. **Field disambiguation** — which fields to use (and which NOT to use) for common queries
3. **NULL/placeholder warnings** — fields that are empty or misleading for certain record types
4. **Date field guidance** — which dimension_group generates which date columns (e.g., `pos_order_date` generates `pos_order_date_date`, `pos_order_date_week`, etc.)

Example pattern:
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
- **Zoë discovers fields beyond the topic.** Zoë can follow foreign key chains to tables not explicitly in the topic. This is powerful but can lead to unexpected joins. Document risky join paths in the topic `zoe_description`.
- **Memories remember the query, not the visualization.** A memory stores the query Zoë ran (including column aliases), so the SQL logic and labeling are preserved. However, memories do **not** capture the visualization type (bar chart, line chart, etc.) or any Python code used for advanced analysis. If a specific visualization or analysis method matters, it must be re-specified by the user or prescribed in the system prompt.

### Data Discovery Workflow
When investigating a new customer's data model, follow this sequence:
1. **Read all view files** to understand the table landscape
2. **Run `SELECT *` with `LIMIT 5`** on key tables to verify columns match the YAML
3. **Run `SELECT DISTINCT` on categorical fields** to discover actual values (field names in YAML may not reflect what's actually stored)
4. **Check for NULL patterns** on key fields across different record subsets (e.g., customer_id may be populated for first-party but NULL for third-party orders)
5. **Test joins with coverage queries** before adding them to topics (e.g., count matched vs unmatched records)

### Terminology Discipline
- **Never invent customer terminology.** Only use terms that exist in the customer's data model, memory definitions, or system prompts. If you need a label, ask the customer what they call it.
- **Don't assume field names match business concepts.** A field called `channel` may not contain what business users call "channels." Always verify with `SELECT DISTINCT`.
- **Document field-to-concept mappings** in `zoe_description` when the field name doesn't match the business term (e.g., `source_org_uri` is actually the platform name like 'partner_platform.com').

### Derived Table Alias Mismatches
- **Aliases in derived table SQL must exactly match the column names referenced by dimensions.** When a derived table uses functions like `LATEST_BY`, `FIRST_VALUE`, or similar, the output alias can silently differ from the original column name. A dimension referencing `${TABLE}."full.column.name"` will fail at query time if the derived table aliased it as `"shortened.name"`.
- **This is a silent, latent bug.** YAML validation doesn't catch it because it doesn't execute the SQL. The error only surfaces when Zoë generates a query that touches the mismatched dimension.
- **Audit pattern:** For every derived table, compare each alias in the SELECT clause against every `${TABLE}."column_name"` reference in the view's dimensions. Any mismatch = runtime "column not found" error.

### Single-View Topics Are Redundant
- **Topics that wrap a single view with no joins serve no purpose.** They don't add join context, they don't add access control that couldn't live on the view itself, and they increase the number of topics Zoë must evaluate for every query.
- **Removing single-view topics reduces noise** and improves Zoë's topic selection accuracy.
- **Before deleting:** Migrate any `description`/`zoe_description` from the topic to the view itself (using `description` since `zoe_description` isn't valid on views). Then delete the topic file.
- **Test:** If a topic has only `base_view` and no additional views in the `views:` section, it's a candidate for removal.
- **Important distinction:** A topic with `base_view: X` and `views: { Y: {} }` is a **two-view topic** (base + joined), not single-view. The `base_view` is implicit. Only topics where `views:` is empty or absent are truly single-view. For multi-view topics, use the "Test-Before-Remove Workflow" (see below) instead of removing outright.

### View Selection Guide in System Prompt
- **When a workspace has many views spanning different domains**, Zoë may scan all views before selecting one, causing slow or incorrect responses. Adding a **View Selection Guide** to the system prompt — mapping question domains to specific views — dramatically improves Zoë's initial view selection.
- **Format:** A simple bulleted list mapping question types to view names (e.g., "Transaction questions → Use transactions_v2"). Include geographic or database scope notes (e.g., "Druid connection, regional data only").
- **This complements, not replaces, good `description` fields on views.** The system prompt guide gives Zoë a quick routing table before it reads individual view metadata.
- **Include negative guidance:** Explicitly state which views should NOT be used for certain question types to prevent cross-domain confusion.

### Row Limits for Detail Queries (Performance Guardrail)
- **Zoë can generate unbounded detail queries** (SELECT with many columns, no aggregation) that return hundreds of thousands of rows, causing timeouts.
- **Add a system prompt instruction to include `LIMIT 1001`** on detail/row-level queries. This doesn't prevent the query from running — it caps the result set so the query executes faster and returns.
- **Post-hoc notification:** If exactly 1,001 rows return, Zoë should inform the user that results were capped and offer alternatives (add filters, aggregate instead, or increase the limit). Note: Zoë still *executes* the query — the limit is a performance guardrail, not a pre-flight check.
- **For aggregated queries** (GROUP BY with SUM, COUNT, AVG), no limit is needed.
- **Alternative approach (COUNT-first):** For warehouses where even a capped query is expensive, instruct Zoë to run `SELECT COUNT(*)` with the same FROM/WHERE first. If the count exceeds a threshold, ask the user before running the detail query. This adds a trivial aggregation query but avoids executing the expensive detail query entirely. On columnar stores like Druid, the COUNT is near-free.

### Memories with Bad SQL Persist and Poison Future Queries
- **Memories created from Zoë responses that contained incorrect SQL** (wrong column names, invalid functions, broken syntax) will be served back to Zoë on similar future questions, perpetuating the error.
- **When fixing data model issues, always audit memories** (Settings → Memory) for entries created *before* the fix. Delete any that contain the old, broken patterns.
- **Common poisoned memory patterns:** References to renamed columns (e.g., `__time` after renaming to a semantic date field), wrong SQL dialect functions (Snowflake functions used on Druid), invalid LOOKUP syntax, or columns that no longer exist after a derived table refactor.

### Consolidating Duplicate Topic/View Variants
- **Workspaces can accumulate near-identical copies** of topics and views (e.g., 5 `supplier_dashboard_locationA` topics and 5 `new_parkers_locationA` views, each for a different location).
- **Consolidate into a single parameterized topic+view** where the user filters by a dimension (e.g., Operator ID, Location) rather than selecting a different topic.
- **Why:** Duplicate topics confuse Zoë (which one to pick?), increase maintenance burden, and create inconsistency when changes need to propagate to all copies.
- **Process:** Create the consolidated view with all locations' data, create one topic, then delete the per-location variants. Migrate any unique descriptions.

### Case Sensitivity in Druid Column Quoting
- **Druid column names are case-sensitive.** If a database column is `"MeterCode"` (PascalCase), the SQL reference must use quoted `"MeterCode"`, not `metercode` or `METERCODE`.
- **Zenlytic auto-lowercases field `name` properties**, but the `sql:` expression must preserve the original casing with double quotes.
- **A casing mismatch causes silent null returns or column-not-found errors** — not a validation-time error.
- **Audit pattern:** For any Druid view, compare `sql:` expressions against actual database column names (run `SELECT * FROM table LIMIT 1` to verify). Ensure quoted column names match exactly.

### System Prompt as Zoë's Operational Manual
- **The system prompt is the most powerful tool for guiding Zoë's behavior** across an entire workspace. It applies to every query, every user, every session.
- **Effective system prompt sections** (proven patterns):
  - **Database Dialect:** Which connections use which SQL dialect, with explicit column avoidance rules
  - **View Selection Guide:** Domain-to-view routing table (see above)
  - **Row Limits:** Performance guardrails for detail queries (see above)
  - **Embedded Environment Rules:** Context that Zoë should infer automatically (e.g., "the park" = user's pre-selected park)
  - **Time Period Rules:** How to interpret relative date phrases ("last month," "last week") anchored to a specific reference date
  - **User Attribute Handling:** Instructions to use automatically-provided attributes (language, timezone) without asking the user
- **Don't put field-level guidance in the system prompt** — use `zoe_description` on the field instead. The system prompt is for workspace-wide rules, not individual field documentation.

### Flag Non-Business-Friendly Field Values for Customer Clarification
- **Dimensions may store cryptic, technical, or integration-level values** (system identifiers, API object names, internal codes) that won't match how business users phrase questions to Zoë. These mappings generally **cannot be inferred from the data alone**.
- **During data discovery, flag any dimension whose stored values use non-business-friendly naming** — abbreviations, system codes, integration object names, or internal identifiers that a business user wouldn't recognize or use in a question.
- **Ask the customer what their users call those values.** Don't guess or invent mappings. The customer knows the business terminology; you don't.
- **Once clarified, apply the standard discoverability pattern:** `searchable: true` on the dimension, `zoe_description` with the explicit stored-value-to-business-name mapping, and `synonyms` with the common business names. This ensures Zoë can translate natural language queries into correct filter values.
- **Example:** A dimension stores `"electronic_funds_transfer"` but users ask about `"EFT"`. Without the mapping, Zoë can't connect the two. After customer clarification, add `zoe_description` documenting `EFT = 'electronic_funds_transfer'` and `synonyms: [EFT, wire transfer]`.

### Test-Before-Remove Workflow for Multi-View Topics
When evaluating whether a multi-view topic can be safely removed (because the join is auto-discoverable via identifiers), **always test before removing**. A topic that looks structurally redundant may still be providing critical field-usage guidance through its `description` or `zoe_description`.

**Complete workflow:**
1. **Identify the candidate** — The topic's join is auto-discoverable (matching identifier names across views, no explicit `sql_on` needed) and the topic has no `always_filter` or `access_filters`.
2. **Write a test question** — Craft a question that forces Zoë to exercise the cross-view join AND any field-usage guidance the topic provides. The question should be specific enough that incorrect behavior is detectable (e.g., wrong filter, wrong aggregation level).
3. **Test with the topic in place (baseline)** — Run the question, save both the results AND the generated SQL. This is your "known good" reference.
4. **Test without the topic** — If possible, temporarily hide the topic (`hidden: true`) or test on a branch without it. Compare both results AND SQL to the baseline.
5. **If it passes** — Proceed with removal: migrate useful `description`/`zoe_description` content to the view's `description`, delete the topic file, commit.
6. **Re-test after removal** — Run the same question again. Compare SQL to the baseline. If regression is found, **strengthen the view description** with more forceful language rather than restoring the topic.
7. **If regression persists** — The topic is providing value that can't be replicated in the view description. Restore it.

**Critical:** Compare the actual SQL generated, not just whether results "look right." Two queries can return superficially similar results while one is fundamentally wrong (e.g., all-category totals vs. filtered-category totals). The SQL WHERE clause is the ground truth.

### Topics Provide More Than Join Context
- **A topic's `zoe_description` often contains field-usage guidance** (which fields to filter by, default filter values, when filters are required) that materially affects query correctness — independently of the join itself.
- **When removing a topic, audit its `zoe_description` for three types of content:**
  1. **Join context** — "X joins to Y on Z." This is redundant if identifiers handle it. Safe to drop.
  2. **Field-usage guidance** — "Filter by category_l1 for broad categories." This MUST be migrated to the view `description` or Zoë will lose it.
  3. **Domain scoping** — "Use this for questions about [domain]." Useful for topic selection but becomes irrelevant once the topic is gone. Drop unless the view needs its own scoping guidance.
- **The most dangerous removal scenario** is when the topic appears structurally redundant (auto-discoverable join) but its `zoe_description` contains field-usage guidance that Zoë relies on. The join works fine without the topic, but Zoë stops applying critical filters.

### Passive vs. Forceful Description Language
- **Zoë responds to the intensity of instructions.** Passive language like "Filter spend by the appropriate category levels" may be ignored. Forceful language like "IMPORTANT: you MUST filter by the appropriate category level. Without this filter, spend totals will include ALL categories and produce incorrect results" gets consistent compliance.
- **When migrating guidance from a topic `zoe_description` to a view `description`, upgrade the language intensity.** Topic descriptions have natural prominence (they're the first thing Zoë reads when routing a query). View descriptions compete with many other fields for attention — so the language must be stronger to achieve the same effect.
- **Effective forceful patterns:**
  - Prefix critical instructions with `IMPORTANT -` or `CRITICAL -`
  - Use `you MUST` instead of passive suggestions
  - State the negative consequence: "Without this filter, results will be incorrect"
  - Be specific about which field to use and when (e.g., "Use category_level_1 for broad categories like 'Raw Materials'")
- **Test after strengthening.** Even forceful language may need iteration. Run the same test question and compare SQL output to the known-good baseline.

### Base View Is Implicit in Topic View Counts
- **A topic with `base_view: X` and `views: { Y: {} }` is a two-view topic**, not a single-view topic. The `base_view` is always implicit — it's the anchor table that all joined views connect to.
- **The "Single-View Topics Are Redundant" rule (see above) applies only when the `views:` section is empty or contains no additional views.** If the `views:` section lists even one view (e.g., `related_costs: {}`), the topic defines a join between the base view and that view — making it a multi-view topic that requires the test-before-remove workflow.
- **Common confusion:** A topic's `zoe_description` may mention two views (base + joined), leading you to think the `views:` section was a workaround for specifying multiple views. It's not — it's simply describing the full join path that the topic defines (`base_view` → `views` entry).

### SQL Comparison Is Essential for Topic Removal Validation
- **When validating topic removal, always compare the generated SQL — not just the query results.** Two queries can return results that look superficially similar or both "reasonable" while one is fundamentally wrong.
- **Example from a customer engagement:** With a procurement topic in place, Zoë's SQL included a category filter (e.g., `WHERE category_level = 'Target Category'`) and returned the correct top suppliers for that category. Without the topic, Zoë's SQL had no category filter at all and returned top suppliers across ALL categories — a completely different (and incorrect) answer. Both result sets were plausible lists of suppliers, but one was scoped correctly and the other wasn't.
- **What to compare in SQL:**
  1. **WHERE clause** — Are the same filters being applied? Missing filters are the #1 regression when removing topics.
  2. **JOIN clause** — Is Zoë using the same join path? Different join paths may produce different result sets.
  3. **GROUP BY / aggregation** — Is the granularity correct?
  4. **Table references** — Is Zoë querying the right tables?
- **Save the baseline SQL** from the "with topic" test. Use it as a diff target for all subsequent tests.

---

## Part 16: LLM Model Considerations

Zoë supports multiple LLM backends, and the choice of model materially affects SQL generation quality, error recovery, instruction adherence, and how effectively the semantic layer context is utilized. This section documents observable behavioral patterns — tested across four models on identical questions against a production workspace — to help data modelers build robust semantic layers that perform well regardless of which model is selected.

### Available Models and Zenlytic Recommendations

Zoë currently supports models from two providers:
- **Anthropic Claude** (Sonnet 4.5, Sonnet 4.6, Opus 4.6) — **Recommended.** Zenlytic recommends Opus 4.6 as the ideal choice for complex data models. Sonnet 4.5 is the minimum recommended baseline.
- **OpenAI GPT** (GPT-4, GPT-5 variants) — Offered for customers who prefer or require OpenAI models. Functional, but exhibits different behavioral patterns that affect data modeling strategy (see below).

> **Note on Sonnet 4.6:** This model was released recently (early 2025). Initial testing shows strong performance, but behavioral patterns may shift as the model stabilizes. Retest in a few weeks to confirm whether its observed behaviors are stable. The observations below reflect early testing results.

Users can switch models mid-conversation via a dropdown in the Zoë interface. Different users in the same workspace may use different models. When newer models are released and tested, recommendations may shift.

### Why Model Choice Matters for Data Modeling

The LLM model is the "interpreter" that reads the semantic layer context (descriptions, topics, identifiers, system prompts, memories) and generates SQL. Different models exhibit different strengths and weaknesses in:

| Capability | What It Affects |
|-----------|----------------|
| **Autonomy** | Whether the model executes immediately or asks clarifying questions first |
| **Error recovery** | How the model handles null data, broken queries, or unexpected results — does it self-correct, present broken results, or stop? |
| **Instruction adherence** | Whether Zoë follows forceful guidance in `zoe_description`, view `description`, and system prompts |
| **Context utilization depth** | How much of the semantic layer the model actually weighs — some models skim, others deeply reason about relationships |
| **Proactive interpretation** | Whether the model flags data quality anomalies, provides per-record narratives, or adds analytical context beyond what was asked |
| **Domain awareness** | Whether the model discovers and applies domain-specific context (fiscal calendars, business hierarchies, naming conventions) |
| **SQL dialect accuracy** | Correctness of generated SQL syntax for the target warehouse |
| **Join reasoning** | Ability to navigate multi-hop joins, avoid fan-outs, and select the right join path |
| **Complex query planning** | Performance on multi-step analytical queries (CTEs, window functions, subqueries) |

### Four-Model Behavioral Framework

> **Disclaimer:** Model capabilities change rapidly with new releases. The observations below reflect patterns seen in production customer workspaces as of early 2025, tested with identical questions across all four models. Re-evaluate when new model versions are released. Sonnet 4.6 is newly released and should be retested after it has had time to stabilize.

Testing revealed that behavioral differences exist **within** the Anthropic model family, not just between Anthropic and GPT. The four models form a clear tier structure:

| Tier | Model | Summary |
|------|-------|---------|
| **Tier 1** | **Opus 4.6** | Fully autonomous, deepest error recovery, domain-aware, delivers the most complete answers |
| **Tier 2** | **Sonnet 4.6** | Semi-autonomous (surfaces ambiguity but self-resolves), strong error recovery, proactive interpretation |
| **Tier 3** | **Sonnet 4.5** | Fully autonomous but shallow — executes immediately, minimal error recovery, may present broken results |
| **Tier 4** | **GPT 5.1** | Executes initial query but stops on errors — highest methodology transparency, lowest answer delivery rate |

### Observable Behavioral Differences (Tested)

The following observations are based on structured testing: each model received the same questions against the same production data, with results compared across all behavioral dimensions.

#### Autonomy (Most Important Initial Difference)

How each model handles ambiguous questions (e.g., "top 5 products by volume" — SKU vs product line? POS Qty vs GSV?):

- **Opus 4.6:** Fully autonomous. Makes reasonable assumptions immediately and executes. Never asks clarifying questions. Delivers results, then offers to refine.
- **Sonnet 4.6:** Semi-autonomous. Surfaces the ambiguity explicitly (e.g., "I'll interpret 'products' as distinct SKUs and 'volume' as POS Qty") but self-resolves without waiting for user confirmation. Proceeds to execute.
- **Sonnet 4.5:** Fully autonomous. Executes immediately without surfacing or acknowledging ambiguity. Fastest to first result.
- **GPT 5.1:** Mixed. May execute an initial query, but stops and asks clarifying questions when encountering complexity or errors. Adds round-trips to the conversation.

**Impact on user experience:** For business users who want quick answers, autonomy delivers immediate value — users see data and can iterate. Confirmation-seeking adds round-trips, which slows time-to-insight and can frustrate non-technical users who may not know how to answer clarifying questions (e.g., "Do you want forecast units or GSV?").

**Impact on data modeling:** With less autonomous models, the data model must pre-resolve more ambiguity — richer `zoe_description` guidance, more explicit `default_date` and `canon_date` settings, and clearer field naming — so the model doesn't need to ask.

#### Error Recovery (Most Important Practical Difference)

All four models were given questions where the underlying data contained null/zero values that would break naive queries (e.g., top-volume SKUs with $0.00 retail prices due to promotional items or internal transfers). How each model handled this:

- **Opus 4.6:** Systematic investigation. Ran ~10 queries autonomously: initial query → detected null/zero data → ran diagnostic queries to understand why → explored alternative data paths → resolved the issue → delivered clean results. Discovered and worked through 3+ distinct errors (query failures, brand value mismatches, NULL quantities) without ever asking the user for help.
- **Sonnet 4.6:** Self-correcting. Ran 3 queries: initial query → caught null/zero data mid-execution → ran diagnostic query → re-queried with exclusion filters → delivered clean results. Less thorough than Opus but caught the problem and fixed it autonomously.
- **Sonnet 4.5:** Presents broken data first. Executed the query, received $0.00 prices and NaN quantities, presented them to the user, then explained why the data looked wrong. Offered to re-run with exclusion filters. The user sees bad data before seeing good data.
- **GPT 5.1:** Stops on error. Executed the initial query, received broken results, then provided a structured explanation of why it couldn't answer (organized into sections: "Data pulled," "Issue in current data," "What I can't do reliably," "How we can tighten this up"). Most transparent methodology documentation, but no answer delivered. Asked the user 3 clarifying questions.

**Key insight:** Error recovery is the strongest practical differentiator between models. The question isn't just "did it execute?" — it's "what happened when reality didn't match expectations?" Opus works through problems like a senior analyst. Sonnet 4.6 catches and fixes problems like a mid-level analyst. Sonnet 4.5 presents the problem and offers to try again. GPT describes the problem and asks what to do.

**Data modeling implication:** For workspaces with messy data (null values, zero-price promotional items, inconsistent categorization), Opus and Sonnet 4.6 will navigate autonomously. Sonnet 4.5 and GPT will surface the problems to users, who may not understand the root cause. Richer `zoe_description` guidance about known data quality issues (e.g., "exclude records where retail_price = 0, as these are promotional/internal transfers") helps all models but is critical for Sonnet 4.5 and GPT.

#### Domain Awareness

- **Opus 4.6:** Actively discovers domain-specific context. In testing, it looked up the customer's 5-4-4 fiscal calendar boundaries (FY2025: Dec 29, 2024 → Jan 3, 2026) and used fiscal months for trending instead of calendar months. No other model did this.
- **Sonnet 4.6:** Uses calendar dates but does not discover or apply fiscal calendar logic unless explicitly guided.
- **Sonnet 4.5:** Uses calendar dates only.
- **GPT 5.1:** Uses calendar dates only.

**Data modeling implication:** For companies with fiscal calendars that differ from calendar months, Opus will often discover and apply the correct boundaries. For other models, document fiscal calendar rules in the system prompt or `zoe_description` (e.g., "This company uses a 5-4-4 fiscal calendar starting the last Sunday in December. Use dimension_group fiscal_year, fiscal_quarter, and fiscal_month for all time-based trending.").

#### Proactive Interpretation

Whether the model adds analytical context beyond the literal question asked:

- **Opus 4.6:** Consistently provides per-product narratives, flags data quality anomalies, notes trends, and offers drill-down suggestions. Delivered the most complete analytical package.
- **Sonnet 4.6:** Provides per-product narratives and flags anomalies (e.g., near-zero prices causing misleading percentage changes, mix shift vs. true price change). Consistent proactive interpretation.
- **Sonnet 4.5:** Inconsistent. Sometimes flags anomalies, sometimes presents results without commentary. In one test, flagged $0 prices after presenting them; in another test, did not flag any anomalies.
- **GPT 5.1:** Does not proactively interpret. Describes methodology and limitations but does not add analytical observations about the data. Most structured documentation of what it *tried* to do, but no insights about what the data *means*.

#### Methodology Transparency

- **GPT 5.1:** Highest transparency. Structures its response into explicit sections (data pulled, issues encountered, limitations, next steps). Documents formulas and calculation approaches. Users always know exactly what the model attempted and why it stopped.
- **Sonnet 4.6:** Medium. Shows its reasoning as it works through problems, but embeds it in the flow of analysis rather than structuring it separately.
- **Opus 4.6:** Medium-high. Comprehensive in its final delivery but doesn't always narrate intermediate steps.
- **Sonnet 4.5:** Low. Executes and presents results with minimal explanation of approach.

#### Analytical Capability (Comparable Across Models)

When all models are given the same well-defined analytical task (e.g., computing price elasticity from POS data using log-log regression), they arrive at **equivalent analytical conclusions** — same regression coefficients, same directional interpretation, same business recommendation.

The difference is in how they get there:
- **Opus 4.6** autonomously chose the price proxy (retail USD / quantity), scoped to all markets, pulled 6 months of weekly data, ran the regression, and returned the elasticity estimate, projected impact, and caveats — all without asking the user anything.
- **GPT 5.1** stopped to ask "What should I use as the price measure?" and "What market scope?" before executing anything. Once it received explicit answers, it produced an equally thorough analysis — arguably even more structured in presentation (numbered methodology sections, explicit formula documentation).

**Key insight:** The capability gap between models is not in analytical depth — it's in the **autonomy to make reasonable assumptions and start working.** GPT needs the user to pre-resolve ambiguity; Opus resolves it autonomously and delivers results immediately. Sonnet 4.6 surfaces the ambiguity but self-resolves. Sonnet 4.5 skips the ambiguity entirely (which is fast but occasionally leads to wrong assumptions).

### Complete Behavioral Comparison

Summary of all observed behaviors across both structured tests:

| Behavior | Sonnet 4.5 | Sonnet 4.6 | Opus 4.6 | GPT 5.1 |
|---|---|---|---|---|
| **Autonomy** | Executes immediately | Surfaces ambiguity, self-resolves | Executes immediately | Mixed — executes, then stops on errors |
| **Error recovery** | Presents broken data, explains after | Self-corrects mid-execution (~3 queries) | Systematic investigation, self-corrects through 3+ errors (~10 queries) | Stops, explains failure, asks user |
| **Domain awareness** | Calendar dates only | Calendar dates only | Discovers fiscal calendars, business hierarchies | Calendar dates only |
| **Proactive interpretation** | Inconsistent | Consistent (anomaly flagging, per-product narratives) | Consistent (deepest analytical context) | None — methodology documentation only |
| **Methodology transparency** | Low | Medium | Medium-high | Highest (structured sections) |
| **Gets user to an answer** | Yes (may include broken data) | Yes (clean) | Yes (clean, most complete) | Often no — blocked on user input |
| **Queries to resolve data issues** | 1 (presents as-is) | ~3 (detect, diagnose, fix) | ~10 (systematic exploration) | 1 (stops and reports) |
| **Instruction adherence** | Moderate | Strong | Strongest | Moderate (ignores long descriptions) |
| **SQL complexity ceiling** | Moderate | High | Highest | Moderate |

### Practical Implications for Data Model Design

The key insight: **a well-built semantic layer should work across all supported models, but the margin for error varies dramatically by model tier.**

#### Design for the Lowest Common Denominator, Optimize for the Best

1. **Universal baseline (all models):**
   - Clear, unambiguous field names (no abbreviations, no duplicates)
   - `searchable: true` on all categorical filter dimensions
   - `default_date` on every view with time-series measures
   - `hidden: true` on all technical/internal fields
   - Explicit `sql_on` joins in topics for any non-obvious relationships
   - At least basic `description` on every view and key fields
   - Document known data quality issues in `zoe_description` (null values, zero-price records, inconsistent categorization)

2. **Tier 1-2 models (Opus 4.6, Sonnet 4.6) unlock more from the semantic layer:**
   - Multi-paragraph `zoe_description` with conditional logic is followed reliably
   - Complex system prompt rules (row limits, view selection guides, dialect-specific column avoidance) are adhered to consistently
   - These models can navigate identifier auto-discovery + topic joins simultaneously without confusion
   - Nuanced business logic in descriptions (e.g., "use field A for questions about X, but field B for questions about Y") works as intended
   - Opus discovers domain context (fiscal calendars, business hierarchies) autonomously; Sonnet 4.6 follows it when documented

3. **Tier 3 model (Sonnet 4.5) needs data quality guardrails:**
   - Executes fast but may present broken data if the semantic layer doesn't pre-filter known issues
   - Add `always_filter` or `zoe_description` guidance to exclude known bad records (e.g., "IMPORTANT: Exclude records where retail_price = 0 — these are promotional/internal transfers, not real pricing data")
   - Descriptions should be concise — Sonnet 4.5 doesn't always weigh long, conditional descriptions

4. **Tier 4 model (GPT 5.1) needs the most explicit structure:**
   - Keep descriptions shorter and more direct — lead with the most critical instruction
   - Pre-resolve ambiguity that the model would otherwise ask the user about (e.g., specify the default unit of measure in `zoe_description`, set `default_date` on every view)
   - Prefer explicit topic `sql_on` over relying on identifier auto-discovery
   - Use `hidden: true` more aggressively to reduce the field space the model must reason over
   - Add more memories for common query patterns to provide explicit SQL examples
   - Consider simplifying join topologies (fewer hop chains, fewer many-to-many relationships)
   - Document fiscal calendars, data quality exceptions, and default scopes explicitly — GPT will not discover these on its own

#### Testing Across Models

When validating data model changes:
- **Test with at least two model tiers** if the workspace's users use multiple models — ideally one Anthropic (Opus or Sonnet 4.6) and GPT
- **Compare the generated SQL across models**, not just the results — a lower-tier model may produce correct results on simple queries but fail on edge cases
- **Test with intentionally messy data** — null values, zero-price records, ambiguous field names — to validate error recovery behavior
- **Document model-specific regressions** — if a query works on one model but fails on another, that's a signal the semantic layer needs to be more explicit (which helps all models)
- **Save the generated SQL from Opus 4.6 as a baseline** — it consistently produces the most thorough SQL. Use it as the reference when evaluating other models' output.

### Model Selection Guidance for Customers

**Zenlytic's recommendation:** Opus 4.6 as the ideal choice for complex data models. Sonnet 4.5 as the minimum recommended baseline. Sonnet 4.6 as a strong middle option that balances speed with error recovery.

> **Sonnet 4.6 note:** This model was recently released and shows very strong initial performance (Tier 2). Retest after a few weeks of production use to confirm behavioral stability before making it a primary recommendation. Early signs are positive — it may prove to be the best balance of speed, cost, and capability for most workspaces.

When advising customers on model selection:
- **For complex data models with messy data**, recommend Opus 4.6. Its autonomous error recovery and domain awareness mean users get answers even when the data has quality issues.
- **For well-structured, clean data models**, Sonnet 4.5 or Sonnet 4.6 may be sufficient and faster/cheaper. If the semantic layer pre-resolves most ambiguities, the gap between tiers narrows.
- **For workspaces using GPT models**, account for the behavioral differences during data modeling: write shorter, more direct descriptions; pre-resolve ambiguity in field naming and `zoe_description`; add more memories for common query patterns; document data quality issues explicitly; and expect the user experience to include more clarification round-trips.
- **Recommend testing.** Have customers run their top 10-20 most common questions across available models and compare both results and SQL quality. Include at least one question that exercises messy data (nulls, edge cases) to test error recovery.
- **Cost and latency trade-offs exist.** More capable models are generally slower and more expensive per query. For simple, well-structured data models with straightforward queries, a faster model may be equally effective and provide a better user experience.
- **The data model quality is the dominant variable.** A well-built semantic layer with clear names, rich descriptions, and explicit joins will outperform a poorly-built one on any model. Model selection is a secondary optimization after the semantic layer is solid.
- **Re-evaluate with new releases.** Model capabilities evolve rapidly. When new model versions are released by either provider, test them against the same benchmark questions before updating recommendations. Sonnet 4.6 specifically should be retested after its initial release period stabilizes.

### Impact on System Prompt and Description Strategy

Because different models process instructions with different fidelity:

- **System prompt instructions should be written for the least capable model** — short, direct, unambiguous. If GPT can follow it, all Anthropic models will too.
- **Critical instructions should use forceful language regardless of model** — "IMPORTANT:", "you MUST", "NEVER" (see Part 15: Passive vs. Forceful Description Language). This costs nothing and dramatically improves compliance across all models.
- **Document data quality issues proactively.** All models benefit from knowing about null values, zero-price records, and other data quirks — but Tier 3-4 models need this guidance to avoid presenting broken data.
- **Avoid relying on implicit reasoning.** Don't assume the model will "figure out" that a join is dangerous — state it explicitly. Don't assume the model will infer the right date field — set `default_date` and `canon_date`. Don't assume the model will discover fiscal calendar boundaries — document them.
- **Memories become more valuable for lower-tier models.** Explicit query/response pairs give the model a concrete example to follow rather than requiring it to reason from first principles. This is especially important for reducing GPT's confirmation-seeking behavior and Sonnet 4.5's tendency to present broken data — a memory that shows "for this question type, here is the correct answer with these exclusion filters" provides a template for correct behavior.

---

## Documentation Sources

### Data Modeling
- [Zenlytic Docs - Data Modeling](https://docs.zenlytic.com/data-modeling/data_modeling)
- [Zenlytic Docs - Views](https://docs.zenlytic.com/data-modeling/view)
- [Zenlytic Docs - Dimensions](https://docs.zenlytic.com/data-modeling/dimension)
- [Zenlytic Docs - Measures](https://docs.zenlytic.com/data-modeling/measure)
- [Zenlytic Docs - Dimension Groups](https://docs.zenlytic.com/data-modeling/dimension_group)
- [Zenlytic Docs - Topics/Joins](https://docs.zenlytic.com/data-modeling/topic)

### Zoë & Tips
- [Zenlytic Docs - Zoë AI Analyst](https://docs.zenlytic.com/zenlytic-ui/zoe)
- [Zenlytic Docs - Zoë Tips & Tricks](https://docs.zenlytic.com/tips-and-tricks/zoe_tips_and_tricks)
- [Zenlytic Docs - Zoë Context Ingestion](https://docs.zenlytic.com/tips-and-tricks/zoe_context_ingestion)
- [Zenlytic Docs - Zoë Memories](https://docs.zenlytic.com/zenlytic-ui/memories)
- [Zenlytic Docs - Entity Drills](https://docs.zenlytic.com/tips-and-tricks/entity-drills)
- [Zenlytic Docs - Time Metrics](https://docs.zenlytic.com/tips-and-tricks/time-metrics)
- [Zenlytic Docs - Data Indexing](https://docs.zenlytic.com/tips-and-tricks/data-indexing)

### UI & Administration
- [Zenlytic Docs - Data Model Editor](https://docs.zenlytic.com/zenlytic-ui/data_model_editor)
- [Zenlytic Docs - User Attributes](https://docs.zenlytic.com/zenlytic-ui/user_attributes)
- [Zenlytic Docs - User Roles](https://docs.zenlytic.com/zenlytic-ui/user_roles)
- [Zenlytic Docs - Workspace Groups & Permissions](https://docs.zenlytic.com/zenlytic-ui/workspace_groups_and_permissions)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.8 | 2025-02-21 | Major Part 16 rewrite: replaced two-family (Anthropic vs GPT) framework with four-model tier framework (Opus 4.6, Sonnet 4.6, Sonnet 4.5, GPT 5.1) based on structured testing with identical questions. Added error recovery as the strongest practical differentiator, domain awareness (fiscal calendar discovery), proactive interpretation dimension, complete behavioral comparison table, tier-specific data modeling guidance, and Sonnet 4.6 early-release stability note. Evidence from two structured test scenarios across all four models. |
| 1.7 | 2025-02-20 | Added Part 16: LLM Model Considerations — documents how different LLM backends (Anthropic Claude, OpenAI GPT) affect Zoë's SQL generation, instruction adherence, and join reasoning. Includes Zenlytic model recommendations (Sonnet 4.5 min, Opus 4.6 ideal), autonomy vs. confirmation-seeking behavioral patterns with production evidence, analytical capability parity example (price elasticity), data exploration differences, GPT-specific data modeling compensations, and impact on description/system prompt strategy. Obfuscated all customer-identifying references throughout KB. |
| 1.6 | 2025-02-20 | Added CLAUDE.md exclusion rule to New Customer Setup: CLAUDE.md must never be committed to customer repos, .gitignore required, canonical copy lives in zoe-data-modeling-kb only. |
| 1.5 | 2025-02-20 | Added 5 new Part 15 lessons from production customer engagement: test-before-remove workflow for multi-view topics, topics provide more than join context (field-usage guidance), passive vs. forceful description language, base view is implicit in topic view counts, SQL comparison is essential for validation. |
| 1.4 | 2025-02-18 | Added Part 15 lesson from customer engagement: flag non-business-friendly field values for customer clarification (name translation pattern). |
| 1.3 | 2025-02-17 | Added 8 new Part 15 lessons from customer engagement: derived table alias mismatches, single-view topic removal, View Selection Guide pattern, row limits guardrail, poisoned memories, duplicate topic consolidation, Druid case sensitivity, system prompt as operational manual. Added __time nuance to SQL dialect lesson. |
| 1.2 | 2025-02-13 | Added Part 15 lessons: redundant identifiers as latent risks, don't band-aid SQL dialect issues, identifier cleanup workflow. |
| 1.1 | 2025-02-13 | Reframed Part 7 for exploratory-only Zoë (Clarity mode deprecated). Removed two-column comparison table, "both modes" language, renamed sections. |
| 1.0 | 2025-02-12 | Initial complete KB — Parts 1-15. Includes CTO corrections on join cardinality nuance, relationships as preferred join location, and Context Manager note. |
