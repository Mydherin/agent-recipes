# agent-recipes

Reusable implementation recipes for coding agents. Each recipe is one Markdown file describing a technical approach, as technology-agnostic as possible, that an agent adapts to the target project.

## Layout

```
recipes/
  <snake_case_name>.md
```

## Recipe format

```markdown
---
name: Human readable name
description: One sentence on what the recipe implements.
---

# Human readable name

## Goal
## Requirements
### Functional
### Non-functional
## Approach
## Steps
## Acceptance criteria
```

- File name: semantic `snake_case`, `.md` extension.
- Front matter: `name` and `description` are required.
- Name libraries, frameworks or operating systems only when the approach depends on them.
