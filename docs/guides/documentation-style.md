# Documentation Style

## Principles

- Write for a new teammate. Assume context, not memory.
- Prefer short pages with clear headings over one long page.
- Show the command/code, then explain it.
- Document what the code actually does — if you're not sure, say so rather than guessing (e.g. the [Quality Measures](../guides/mom_backend/modules/quality-measures.md) doc marks a few measure names as "not specified in code" instead of inventing them).

## Conventions

- One H1 (`#`) per page — it becomes the page title and nav label.
- Use relative links between docs so they survive moves.
- Use admonitions (`!!! note`, `!!! warning`, `!!! tip`) for callouts.
- Put diagrams in [Mermaid](https://squidfunk.github.io/mkdocs-material/reference/diagrams/) so they live in version control.
- For API/module reference pages, use a consistent shape: an **Endpoints** table (method, path, purpose), a **Data model** section, and a **Notes** section for non-obvious behavior — see any page under [Feature Modules](../guides/mom_backend/modules/index.md) for the pattern.
- Wrap file paths, headers, and code identifiers in backticks so they render as `code`, not prose.
