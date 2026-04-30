# Contributing to TypeScript DDD Clean Architecture Skills

Thank you for taking the time to contribute!

## How to Add or Improve a Skill

### 1. Fork and clone

```bash
git clone https://github.com/YOUR_USERNAME/ts-ddd-clean-architecture.git
cd ts-ddd-clean-architecture
```

### 2. Create or edit a skill

Each skill lives in its own folder:

```
skills/
└── your-skill-name/
    └── SKILL.md
```

### SKILL.md format (required)

Every `SKILL.md` must follow this structure:

```markdown
---
name: your-skill-name
description: One sentence describing when Claude should use this skill.
---

# Title

Brief intro paragraph.

## When to Activate

- Bullet list of triggers

## Core Principles

1. **Principle** — explanation

## [Topic Sections with Code Examples]

```language
// real, working code
```

## Execution Workflow

Numbered step-by-step.

## Anti-Patterns

| ❌ Never Do | ✅ Instead |
|---|---|
| bad pattern | correct approach |
```

### 3. Rules for quality

- All code examples must be real, working TypeScript (no pseudocode)
- File paths in code comments must reflect the actual `src/` directory structure
- Anti-patterns table is mandatory
- "When to Activate" must have at least 4 bullet points

### 4. Open a Pull Request

- Branch name: `skill/your-skill-name` or `fix/skill-name-description`
- PR title: `Add skill: your-skill-name` or `Improve skill: skill-name`
- Describe what the skill covers and why it is useful

## Reporting Issues

Open a GitHub Issue if you find:
- Incorrect or outdated code examples
- Missing edge cases or anti-patterns
- A skill that conflicts with another skill

## Code of Conduct

Be respectful and constructive. All contributions are welcome.
