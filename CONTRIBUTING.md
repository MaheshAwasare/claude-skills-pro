# Contributing

Thanks for your interest. The repo is intentionally opinionated — PRs that match the bar will be merged quickly.

## What we accept

- **New skills** that fit one of the four buckets (Platform/Tool, Workflow/Meta, Migration, Compliance).
- **Improvements to existing skills** — better examples, fixed anti-patterns, clearer explanations.
- **Bug fixes** in code samples.

## What we don't accept

- Generic "X coding standards" or "X testing" skills — these are already covered by Anthropic's official `everything-claude-code` plugin, so adding more dilutes the repo.
- Thin prompt wrappers (under 100 lines, no concrete examples).
- Skills without an anti-patterns section. The anti-patterns are what make a skill load-bearing instead of decorative.

## Skill structure

```
skills/<skill-name>/
  SKILL.md
  examples/   # optional
  templates/  # optional
```

### SKILL.md template

```markdown
---
name: <skill-name>
description: <one line — used by Claude to decide when to load this skill, so be specific about triggers>
---

# <Skill Title>

## When to use
- ...

## When NOT to use
- ...

## Core concepts
<the mental model the user/Claude needs>

## <Concrete how-to sections — code, config, examples>

## Anti-patterns
- **<Anti-pattern name>** — why it's wrong, what to do instead

## Verify it worked
- [ ] ...
- [ ] ...
```

## Style rules

1. **No emoji in SKILL.md bodies** unless they're part of an example output (e.g. status icons in a CLI demo).
2. **Concrete file paths in examples**, not `<your-app>/...`. Use `apps/api/src/...` etc.
3. **Link out instead of duplicating** the official docs of a tool. The skill's job is the *opinionated path*, not the reference.
4. **Anti-patterns must give the reason**, not just the rule. "Don't do X — it breaks Y when Z" beats "Don't do X."

## Process

1. Open an issue first for new skills, so we can scope it together (avoids you writing 250 lines we'd want refactored).
2. Fork → branch → PR.
3. CI is minimal — markdownlint + frontmatter validation. Pass it locally before pushing.

## License

By contributing you agree your work is licensed under MIT.
