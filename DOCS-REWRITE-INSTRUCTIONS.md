# Zenlytic Documentation Site Rewrite — Full Instruction Set

**Purpose:** Complete overhaul of docs.zenlytic.com. This document contains everything needed to execute the rewrite in an independent session.

---

## 1. Context

### What Zenlytic Is
Zenlytic is a BI platform whose core differentiator is **Zoë**, an AI data analyst. Users ask business questions in natural language → Zoë generates SQL against a governed semantic layer (called the "Cognitive Layer") → returns charts, tables, and insights. The semantic layer is defined in YAML files (views, topics, models) that describe database tables, their fields, how they join, and business context.

### What the Docs Site Is For
docs.zenlytic.com is the public documentation for Zenlytic customers and implementation teams. Primary audiences:
1. **Data modelers** building/maintaining the YAML semantic layer
2. **Workspace admins** configuring roles, permissions, security
3. **Zoë optimizers** tuning descriptions, system prompts, and memories to improve AI accuracy
4. **Embedded analytics developers** integrating Zenlytic into their products
5. **End users** learning how to interact with Zoë

### Why the Rewrite
The current site is a YAML property reference manual. It documents *what* each property does but not *why* you'd use it, *how* things connect, or *what goes wrong*. Critical concepts are entirely missing. The most valuable content is buried under "Tips & Tricks" — a section name that signals optional, when it's actually core.

---

## 2. Source of Truth: The Knowledge Base

The canonical knowledge base is at:
```
/Users/gregpeters/Library/CloudStorage/GoogleDrive-greg@zenlytic.com/My Drive/Claude Code Projects/Git-Repo-Work/Zenlytic-Customers/CLAUDE.md
```

This is a v1.8 document (Parts 1-16) built from Zenlytic's official docs + real customer engagement learnings. **It contains everything the new docs should cover.** Use it as the authoritative source for:
- Complete YAML schema (all properties, all types, all valid combinations)
- How Zoë works (agentic architecture, SQL generation, context ingestion)
- Join strategy (identifiers, topics, model-level relationships, and when to use each)
- Common pitfalls (Part 15 — all discovered through production customer work)
- LLM model behavioral differences (Part 16)
- Permissions model and how it interacts with Zoë
- System prompt patterns
- Memory management

**Important:** The KB was written for AI coders working on customer workspaces. The docs site is for customers. The content must be rewritten for a customer audience — less internal process, more "here's how to do this in your workspace."

---

## 3. Current Docs Site Structure (What Exists Today)

```
docs.zenlytic.com/
├── Tips & Tricks/
│   ├── zoe_tips_and_tricks        — Core Zoë optimization (mislabeled as "tips")
│   ├── zoe_context_ingestion      — What Zoë sees (has misleading priority hierarchy)
│   ├── entity-drills              — Drill field configuration
│   ├── time-metrics               — default_date & canon_date
│   └── data-indexing              — searchable fields
├── Zenlytic UI/
│   ├── zoe                        — Zoë overview (features, inputs, integrations)
│   ├── memories                   — Creating/managing memories
│   ├── data_model_editor          — Branch workflow, adding tables, deploying
│   ├── user_attributes            — User attribute setup
│   ├── user_roles                 — 8 roles, 13 permissions
│   └── workspace_groups_and_permissions — Groups + dashboard-level access
├── Data Modeling/
│   ├── data_modeling              — Overview + naming conventions
│   ├── view                       — View properties
│   ├── dimension                  — Dimension properties
│   ├── dimension_group            — Time & duration dimension groups
│   ├── measure                    — Measure properties + aggregation types
│   └── topic                      — Topic/join configuration
├── Workflows/                     — (minimal content)
├── Embedding/
│   └── embedding_overview         — Private vs Signed embedding
├── Data Sources/
│   └── integrations               — Supported warehouses list
├── Authentication & Security/
│   └── microsoft_entra_zenlytic   — Entra SSO setup guide
└── Legal & Support/               — Support policy, legal
```

### What's Wrong With It

1. **"Tips & Tricks" contains core concepts** — searchable fields, time metrics, entity drills, and Zoë context ingestion are not tips. They're required knowledge for building a working data model.

2. **Zoë is buried** — The AI analyst is Zenlytic's differentiator but gets one UI page and scattered references. No cohesive section on how to optimize for Zoë.

3. **Missing entirely from the current site:**
   - Model-level `relationships` property
   - System prompt writing guidance
   - LLM model selection and behavioral differences (four-model tier framework with structured test evidence)
   - Common pitfalls / troubleshooting
   - Poisoned memory auditing
   - Fan-out / double-counting prevention
   - Redundant identifier risks
   - Passive vs forceful description language
   - `join_as` on identifiers (barely mentioned)
   - `non_additive_dimension` (mentioned but poorly explained)
   - How permissions enforcement interacts with Zoë's SQL generation
   - Derived table alias mismatch risks
   - View Selection Guide pattern for system prompts
   - Row limit guardrails for detail queries

4. **Misleading content:**
   - Context ingestion page presents a strict 1-2-3-4 priority hierarchy. In reality, Zoë weighs all context based on relevance to the query — there is no strict hierarchy. Replace with a role-based table (what each source is best for).

5. **No concept of "how things work together"** — Views, dimensions, measures, topics, and identifiers are documented as isolated concepts. No workflow showing how they combine into a working Zoë experience.

6. **No audience segmentation** — A data modeler and a workspace admin read the same flat structure.

---

## 4. New Documentation Structure

### Top-Level Navigation

```
Getting Started
├── What is Zenlytic?
├── Quick Start Guide
├── Architecture Overview
│   (semantic layer → Zoë → SQL → results diagram)
└── Glossary

Building Your Data Model
├── Overview & Repository Structure
│   (zenlytic_project.yml, models/, views/, topics/)
├── Models & Connections
├── Views
│   ├── Basic View Setup (sql_table_name, label, description)
│   ├── Derived Tables (sql, caveats, always_filter doesn't apply)
│   ├── Filters (always_filter vs access_filters — clear distinction)
│   └── Identifiers (primary, foreign, join, join_as)
├── Dimensions
│   ├── Standard Dimensions (string, number, yesno, tier)
│   ├── Dimension Groups — Time (timeframes, datatype, convert_tz)
│   ├── Dimension Groups — Duration (sql_start, sql_end, intervals)
│   └── Data Indexing / searchable Fields
│       (include allow_higher_searchable_max, capitalization rules)
├── Measures
│   ├── Standard Aggregations (sum, average, count, count_distinct, max, min, median)
│   ├── Distinct Aggregations (sum_distinct, average_distinct + sql_distinct_key)
│   ├── Cumulative Measures
│   ├── Number Type (calculated/static)
│   └── Non-Additive Dimensions (with clear MRR example)
├── Topics & Joins
│   ├── Topic Basics (base_view, views, label, description, zoe_description)
│   ├── Join Types & Relationships (left_outer, inner, full_outer + cardinality)
│   ├── Explicit Joins (sql_on with single and multi-column examples)
│   ├── Chained Joins / Bridge Tables
│   ├── Joining the Same View Twice (from + alias pattern)
│   ├── Model-Level Relationships ← NEW PAGE
│   │   (the relationships property on the model, when to use vs topics vs identifiers)
│   └── Join Strategy Guide ← NEW PAGE
│       (decision tree: when to use identifiers vs topics vs relationships,
│        identifier cleanup, redundant identifier risks)
├── Time Configuration
│   ├── default_date (view level)
│   ├── canon_date (measure level)
│   ├── Cross-Table Date References
│   └── Fiscal Calendars
├── Entity Drills (drill_fields, tags, primary key drills)
└── Common Patterns
    ├── Star Schema
    ├── Slowly Changing Dimensions
    └── dbt MetricFlow Integration

Optimizing for Zoë ← NEW TOP-LEVEL SECTION
├── How Zoë Works
│   (agentic architecture, SQL generation is non-deterministic,
│    user prompt → LLM → SQL → results, the "talented analyst on day one" test)
├── What Zoë Sees
│   ├── Overview (all sources weighed by relevance — NO strict hierarchy)
│   ├── System Prompts
│   ├── Topics, Identifiers & Relationships
│   ├── View & Field Descriptions (description vs zoe_description — where each is valid)
│   ├── Memories
│   ├── Searchable Fields & Synonyms
│   └── Access Controls (what Zoë can/cannot see based on permissions)
├── Writing Effective Descriptions ← NEW PAGE
│   ├── description vs zoe_description (valid locations for each)
│   ├── Passive vs Forceful Language (with before/after examples)
│   ├── View-Level Join Warnings
│   └── Topic zoe_description Patterns (when to use, field disambiguation, NULL warnings, date guidance)
├── Writing System Prompts ← NEW PAGE
│   ├── What Goes in a System Prompt (workspace-wide rules, NOT field-level guidance)
│   ├── View Selection Guide Pattern
│   ├── Row Limit Guardrails (LIMIT 1001 + COUNT-first alternative)
│   ├── Column Avoidance Rules (e.g., raw internal columns)
│   ├── Embedded Environment Rules
│   ├── Time Period Interpretation Rules
│   └── User Attribute Handling
├── Memories
│   ├── Creating Memories (from chat + from portal)
│   ├── Managing Memories
│   ├── When to Use Memories vs Other Context Types (decision table)
│   └── Auditing for Poisoned Memories ← NEW
│       (memories with bad SQL persist and poison future queries; audit after model fixes)
├── LLM Model Selection ← NEW PAGE
│   ├── Available Models (Anthropic Claude: Sonnet 4.5, Sonnet 4.6, Opus 4.6; OpenAI GPT 5.1)
│   ├── Four-Model Tier Framework
│   │   (Tier 1: Opus 4.6, Tier 2: Sonnet 4.6, Tier 3: Sonnet 4.5, Tier 4: GPT 5.1)
│   ├── Behavioral Differences (Tested)
│   │   (autonomy, error recovery [strongest differentiator], domain awareness,
│   │    proactive interpretation, methodology transparency, instruction adherence,
│   │    SQL complexity ceiling, complete comparison table)
│   ├── Analytical Capability Parity
│   │   (same conclusions, different working style — autonomy is the variable)
│   ├── Data Model Implications by Tier
│   │   (universal baseline + tier-specific compensations)
│   ├── Zenlytic Recommendations (Opus 4.6 ideal, Sonnet 4.5 min, Sonnet 4.6 note)
│   └── Testing Across Models (include messy-data tests for error recovery)
└── Troubleshooting Zoë ← NEW PAGE
    ├── Zoë Picks the Wrong Join Path
    ├── Zoë Generates Invalid SQL
    ├── Zoë Misses a Required Filter
    ├── Fan-Out / Double Counting
    ├── Redundant Identifiers Causing Unexpected Joins
    ├── Derived Table Alias Mismatches
    ├── Wrong Column from Raw Table Instead of Dimension Group
    └── Slow Queries / Timeouts on Detail Queries

Security & Permissions
├── User Roles & Permissions Matrix (8 roles, 13 permissions — clean table)
├── User Attributes
│   (setup, assignment, reserved attributes: email, zenlytic_connection_name, zenlytic_connection_database)
├── Access Filters (Row-Level Security)
├── Required Access Grants (Column-Level / OR logic)
├── Workspace Groups & Dashboard Permissions
└── How Permissions Interact with Zoë ← NEW PAGE
    (admins vs explore users, enforce-permissions toggle,
     what Zoë can query vs what users can see)

Zenlytic UI
├── Zoë (AI Analyst)
│   ├── Asking Questions (text, voice, file attachments)
│   ├── Understanding Results (charts, citations, drill-down)
│   ├── Dynamic Fields (create personal measures/dimensions)
│   ├── Integrations (Slack, Microsoft Teams)
│   └── Workflows
├── Explore Interface
├── Dashboards
└── Data Model Editor
    ├── Branch Workflow
    ├── Adding Tables from Database
    ├── Validation & Error Handling
    └── Deploying to Production

Embedding
├── Overview (Private vs Signed)
├── Private Embedding
├── Signed Embedding
└── Embedded User Roles

Data Sources & Connections
├── Supported Warehouses
│   (Snowflake, BigQuery, Redshift, Postgres, MySQL, Databricks,
│    Druid, DuckDB/MotherDuck, Trino, SQL Server, Azure Synapse)
├── Connection Configuration
└── dbt MetricFlow Integration

Authentication & Security
├── SSO / Microsoft Entra Setup
└── (other auth methods as applicable)

Legal & Support
├── Support Policy
└── Legal
```

---

## 5. Content Migration Map

This maps every current page to its new location and flags what's new.

| Current Page | New Location | Notes |
|---|---|---|
| Tips & Tricks / zoe_tips_and_tricks | **Optimizing for Zoë / How Zoë Works** + distributed across "Writing Effective Descriptions" and other Zoë pages | Break apart — it covers 7 different concepts in one page |
| Tips & Tricks / zoe_context_ingestion | **Optimizing for Zoë / What Zoë Sees / Overview** | Rewrite: remove misleading priority hierarchy, replace with relevance-based model |
| Tips & Tricks / entity-drills | **Building Your Data Model / Entity Drills** | Move as-is, expand with examples |
| Tips & Tricks / time-metrics | **Building Your Data Model / Time Configuration** | Move, expand with cross-table date examples |
| Tips & Tricks / data-indexing | **Building Your Data Model / Dimensions / Data Indexing** | Move, add allow_higher_searchable_max and capitalization guidance |
| Zenlytic UI / zoe | **Zenlytic UI / Zoë** | Keep, reorganize into sub-pages |
| Zenlytic UI / memories | **Optimizing for Zoë / Memories** | Move, expand with poisoned memories + decision table |
| Zenlytic UI / data_model_editor | **Zenlytic UI / Data Model Editor** | Keep location |
| Zenlytic UI / user_attributes | **Security & Permissions / User Attributes** | Move from UI section to Security |
| Zenlytic UI / user_roles | **Security & Permissions / User Roles** | Move, add clean permission matrix |
| Zenlytic UI / workspace_groups | **Security & Permissions / Workspace Groups** | Move |
| Data Modeling / data_modeling | **Building Your Data Model / Overview** | Rewrite intro, add repo structure diagram |
| Data Modeling / view | **Building Your Data Model / Views / Basic View Setup** | Split into sub-pages (basic, derived, filters, identifiers) |
| Data Modeling / dimension | **Building Your Data Model / Dimensions / Standard Dimensions** | Split from dimension_group |
| Data Modeling / dimension_group | **Building Your Data Model / Dimensions / Dimension Groups** | Split into Time and Duration sub-pages |
| Data Modeling / measure | **Building Your Data Model / Measures** | Split into sub-pages by type |
| Data Modeling / topic | **Building Your Data Model / Topics & Joins** | Expand massively — 7 sub-pages |
| N/A — **NEW** | Optimizing for Zoë / Writing Effective Descriptions | Source: KB Part 7 (Zoë properties table, description vs zoe_description), Part 15 (passive vs forceful language, topic zoe_description patterns) |
| N/A — **NEW** | Optimizing for Zoë / Writing System Prompts | Source: KB Part 15 (system prompt as operational manual, view selection guide, row limits, column avoidance) |
| N/A — **NEW** | Optimizing for Zoë / LLM Model Selection | Source: KB Part 16 (entire section) |
| N/A — **NEW** | Optimizing for Zoë / Troubleshooting Zoë | Source: KB Part 15 (identifier validation, redundant identifiers, derived table alias mismatches, fan-out, etc.) |
| N/A — **NEW** | Building Your Data Model / Topics & Joins / Model-Level Relationships | Source: KB Part 7 (model-level relationships example) |
| N/A — **NEW** | Building Your Data Model / Topics & Joins / Join Strategy Guide | Source: KB Part 15 (identifier cleanup workflow, redundant identifiers, test-before-remove) |
| N/A — **NEW** | Security & Permissions / How Permissions Interact with Zoë | Source: KB Part 7 (Zoë Permissions section) |

---

## 6. Writing Guidelines

### Voice & Audience
- Write for a **data modeler who just joined a team using Zenlytic**. They know SQL and analytics but not Zenlytic-specific concepts.
- Use second person ("you") and active voice.
- Lead with *why*, then *how*, then *reference*. Every page should explain the concept's purpose before showing properties.
- The guiding question for every page: **"Could a talented data analyst understand this on their first day?"**

### Page Structure Pattern
Every content page should follow this structure:
1. **One-paragraph overview** — What is this concept? Why does it matter?
2. **When to use it** — Practical guidance on when/why you'd configure this
3. **How to configure it** — YAML examples with annotations
4. **Complete property reference** — Table of all properties (required + optional)
5. **Common patterns** — Real-world examples (obfuscated, no customer names)
6. **Common mistakes** — What goes wrong and how to fix it (where applicable)

### YAML Examples
- Every YAML example must be valid and complete (not fragments)
- Use realistic but generic field names (orders, customers, products — not customer-specific)
- Annotate with inline comments explaining non-obvious choices
- Show both simple and advanced configurations

### Cross-Linking
- Link generously between related pages (e.g., from the dimension `searchable` property to the full Data Indexing page)
- Every "Optimizing for Zoë" page should link back to the relevant "Building Your Data Model" page and vice versa

### What NOT to Include
- No internal process documentation (Git workflows for Claude Code, customer engagement patterns)
- No customer names or customer-identifying examples
- No references to the internal knowledge base or CLAUDE.md
- No "Tips & Tricks" framing — everything is core documentation

---

## 7. Critical Corrections to Make During Rewrite

These are specific factual errors or misleading content on the current site that must be fixed:

### 1. Context Ingestion Priority Hierarchy (WRONG)
**Current:** Presents a strict 1-2-3-4 priority ordering (system prompt → structural relationships → descriptions → memories)
**Correct:** Zoë weighs all context sources based on relevance to the query. There is no strict hierarchy. Replace with a table showing what each source is best for:

| Context Source | Best For |
|---|---|
| Custom System Prompt | Organization-wide rules, terminology, calculation logic |
| Topics / Relationships | Structural join context, analytical domain scoping |
| description / zoe_description | Field-level and view-level business context |
| Memories | Reinforcing correct response patterns for specific question types |

### 2. `zoe_description` Valid Locations (INCOMPLETE)
**Current:** Mentioned on some field pages but not clearly stated where it's valid vs invalid.
**Correct:** `zoe_description` is valid on: dimensions, measures, dimension_groups, topics. It is NOT valid on views. On views, use `description` (which is sent to both the UI and Zoë).

### 3. Memories Limitations (INCOMPLETE)
**Current:** Says memories are opt-in and for data interpretation only.
**Missing:** Memories created from Zoë responses that contained incorrect SQL will persist and poison future queries. After fixing data model issues, always audit memories (Settings → Memory) for entries created before the fix.

### 4. Identifier Auto-Discovery (UNDOCUMENTED RISK)
**Current:** Identifiers are documented as join helpers.
**Missing:** Identifiers remain auto-discoverable by Zoë even when a topic defines an explicit sql_on for the same join. A redundant identifier gives Zoë a second, less-controlled join path. Rule: if a topic defines an explicit sql_on, remove overlapping foreign identifiers.

### 5. `always_filter` and Derived Tables (UNCLEAR)
**Current:** Mentioned in passing that always_filter doesn't apply to derived tables.
**Correct:** This must be prominently called out — it's a common source of confusion. When using `derived_table`, the SQL statement becomes the data foundation and `always_filter` has no effect.

### 6. Model-Level `relationships` (MISSING)
**Current:** Not documented at all.
**Correct:** The `relationships` property on the model is the recommended location for storing non-obvious joins. Zoë sees all of these alongside identifiers and topics.

---

## 8. New Pages — Content Source Mapping

For each new page, here's exactly where to find the content in the KB:

### Optimizing for Zoë / How Zoë Works
- KB Part 7: "How Zoë Works" table, "How Zoë Weighs Context Sources", "Zoë AI Data Analyst Details"
- KB Part 7: "Guiding Principle for Data Model Quality"

### Optimizing for Zoë / Writing Effective Descriptions
- KB Part 7: "Zoë-Specific Properties" table, "When to Use Each Context Type" table
- KB Part 15: "Passive vs. Forceful Description Language" (full section with before/after patterns)
- KB Part 15: "Topic `zoe_description` Patterns" (full section with template)
- KB Part 7: "Example Zoë-Optimized View with Join Warnings"

### Optimizing for Zoë / Writing System Prompts
- KB Part 15: "System Prompt as Zoë's Operational Manual" (full section with all proven patterns)
- KB Part 15: "View Selection Guide in System Prompt"
- KB Part 15: "Row Limits for Detail Queries"
- KB Part 15: "Don't Band-Aid SQL Dialect Issues in the Data Model" (exception: column avoidance IS appropriate)

### Optimizing for Zoë / LLM Model Selection
- KB Part 16: Entire section (Available Models, Four-Model Tier Framework, Observable Behavioral Differences, Complete Behavioral Comparison Table, Practical Implications by Tier, Model Selection Guidance, Impact on System Prompt Strategy)
- Important: The **four-model tier framework** is the centerpiece — Opus 4.6 (Tier 1), Sonnet 4.6 (Tier 2), Sonnet 4.5 (Tier 3), GPT 5.1 (Tier 4). Behavioral differences exist WITHIN the Anthropic family, not just between Anthropic and GPT.
- **Error recovery is the strongest practical differentiator** — include the null-data test scenario (all models hit the same data quality issue; each handled it differently: Opus self-corrected through ~10 queries, Sonnet 4.6 self-corrected in ~3, Sonnet 4.5 presented broken data, GPT stopped and asked)
- Include the price elasticity parity example (analytical capability is comparable across models — autonomy is the variable)
- Include the complete behavioral comparison table from Part 16
- Note that Sonnet 4.6 is newly released and should be retested after stabilization — don't present its tier placement as final
- Tier-specific data model implications are critical for the docs audience: show customers what to do differently based on which model their users select

### Optimizing for Zoë / Troubleshooting Zoë
- KB Part 15: "Identifier Validation" (verify identifiers against actual database)
- KB Part 15: "Redundant Identifiers as Latent Risks"
- KB Part 15: "Derived Table Alias Mismatches"
- KB Part 15: "Memories with Bad SQL Persist and Poison Future Queries"
- KB Part 15: "Don't Band-Aid SQL Dialect Issues" (the __time column exception)
- KB Part 7: "How Zoë Sees Join Information" (fan-out warning)
- KB Part 15: "SQL Comparison Is Essential for Topic Removal Validation"

### Building Your Data Model / Topics & Joins / Model-Level Relationships
- KB Part 7: "Model-Level Relationships" (full YAML example)
- KB Part 7: "Where to Store Joins (Recommended)" section

### Building Your Data Model / Topics & Joins / Join Strategy Guide
- KB Part 15: "Identifier Cleanup Workflow" (6-step reusable pattern)
- KB Part 15: "Redundant Identifiers as Latent Risks"
- KB Part 15: "Multi-Column Joins in Topics"
- KB Part 15: "Chained Joins (Bridge Tables)"
- KB Part 15: "Test-Before-Remove Workflow for Multi-View Topics"
- KB Part 15: "Topics Provide More Than Join Context"
- KB Part 6: All topic/join content

### Security & Permissions / How Permissions Interact with Zoë
- KB Part 7: "Zoë Permissions" (full section — admin vs explore, enforce toggle, what's enforced by system vs what's Zoë's responsibility)

---

## 9. Page Priority Order

Write pages in this order (highest impact first):

**Phase 1 — Foundation**
1. Getting Started / Architecture Overview
2. Building Your Data Model / Overview & Repository Structure
3. Building Your Data Model / Views / Basic View Setup
4. Building Your Data Model / Dimensions (all sub-pages)
5. Building Your Data Model / Measures (all sub-pages)
6. Building Your Data Model / Topics & Joins (all sub-pages including new ones)

**Phase 2 — Zoë (the differentiator)**
7. Optimizing for Zoë / How Zoë Works
8. Optimizing for Zoë / What Zoë Sees (with corrected context model)
9. Optimizing for Zoë / Writing Effective Descriptions
10. Optimizing for Zoë / Writing System Prompts
11. Optimizing for Zoë / Memories (expanded)
12. Optimizing for Zoë / LLM Model Selection
13. Optimizing for Zoë / Troubleshooting Zoë

**Phase 3 — Security & Admin**
14. Security & Permissions (all pages)
15. Zenlytic UI (all pages)

**Phase 4 — Specialized**
16. Embedding
17. Data Sources & Connections
18. Authentication & Security
19. Building Your Data Model / Time Configuration
20. Building Your Data Model / Entity Drills
21. Building Your Data Model / Common Patterns

---

## 10. Final Checklist

Before publishing any page, verify:

- [ ] Page follows the structure pattern (overview → when to use → how to configure → reference → patterns → mistakes)
- [ ] All YAML examples are valid and complete
- [ ] `zoe_description` is never shown as valid on views
- [ ] Context ingestion does NOT present a strict priority hierarchy
- [ ] No customer names or identifying examples anywhere
- [ ] Cross-links to related pages are included
- [ ] Every property table distinguishes required vs optional
- [ ] "Common mistakes" section exists for complex topics (joins, identifiers, measures)
- [ ] Zoë-related guidance links back to "Optimizing for Zoë" section
- [ ] Model-level `relationships` are mentioned wherever join strategies are discussed
