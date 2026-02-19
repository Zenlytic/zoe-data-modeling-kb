# Zoë Data Modeling Knowledge Base

A comprehensive knowledge base for working with **Zenlytic's semantic layer** and **Zoë**, Zenlytic's AI data analyst. This document serves as the authoritative reference for building, maintaining, and troubleshooting data models across Zenlytic customer workspaces.

## What This Is

`CLAUDE.md` contains the complete knowledge base covering:

- **YAML schema** for views, dimensions, measures, dimension groups, topics, and models
- **Join architecture** — identifiers, topics, relationships, and how Zoë interprets them
- **Zoë AI context** — how Zoë ingests descriptions, zoe_descriptions, synonyms, searchable fields, memories, and system prompts to generate SQL
- **Git operations** for managing Zenlytic customer repositories
- **Permissions and access controls** — user roles, attributes, row-level and column-level security
- **Practical lessons** — real-world patterns, pitfalls, and workflows discovered across customer engagements

## Who This Is For

Anyone using an AI coding assistant to work with Zenlytic data models — whether that's Claude, ChatGPT, Copilot, Cursor, or any other AI-powered tool.

## How to Use It

### With Claude Code (automatic)

Place `CLAUDE.md` in your project's root directory or a parent directory. Claude Code automatically detects and loads files named `CLAUDE.md` as project instructions — no additional setup required.

### With Other AI Coding Assistants

Load the knowledge base into your AI tool's context before starting work:

- **ChatGPT / GPT-based tools**: Attach `CLAUDE.md` as a file or paste its contents into your system prompt or custom instructions
- **Cursor**: Place the file in your project root or add it to your `.cursorrules` / docs context
- **GitHub Copilot Chat**: Reference the file with `#file:CLAUDE.md` in your prompt
- **Any other AI tool**: Copy the contents into whatever context, system prompt, or knowledge base mechanism your tool supports

The content is plain Markdown with no tool-specific syntax. It works as a reference document for any LLM.

### What to Expect

Once loaded, your AI assistant will understand:

- How to read and write Zenlytic YAML files (views, topics, models)
- The correct schema for dimensions, measures, dimension groups, and identifiers
- How Zoë generates SQL and what context sources influence its behavior
- Join best practices and common pitfalls (fan-out, redundant identifiers, chained joins)
- When to use `description` vs `zoe_description` vs system prompts vs memories
- Data discovery workflows for investigating new customer workspaces

## Versioning

The knowledge base is versioned internally. Check the **Changelog** section at the bottom of `CLAUDE.md` for the history of changes. When making updates, increment the version number (minor changes: 1.4 → 1.5, major restructures: 1.x → 2.0) and add a changelog entry.

## Repository Structure

```
/
├── README.md       # This file
└── CLAUDE.md       # The knowledge base
```

## Maintenance

This knowledge base is actively maintained and updated as new patterns are discovered during customer engagements. Lessons learned from real-world data modeling work are added to **Part 15: Practical Lessons & Common Pitfalls**.
