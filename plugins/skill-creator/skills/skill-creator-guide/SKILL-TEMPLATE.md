# Skill Template

Copy and adapt this template when creating a new skill.

## Minimal Skill

```yaml
---
name: your-skill-name
description: >-
  [What it does]. Use when [trigger conditions].
  Covers [key capabilities].
---

# Your Skill Name

## Instructions

### Step 1: [First Step]
[Clear, actionable instructions]

### Step 2: [Second Step]
[Clear, actionable instructions]

## Examples

### Example: [Scenario name]
User says: "[example prompt]"
Result: [Expected output]
```

## Task Skill (User-Only)

```yaml
---
name: your-task-skill
description: >-
  [What it does]. Use when [trigger conditions].
disable-model-invocation: true
argument-hint: "[required-argument]"
---

# Your Task Skill

## Instructions

### Step 1: Validate Input
Verify that $ARGUMENTS is provided and valid.

### Step 2: Execute Task
[Step-by-step instructions]

### Step 3: Report Result
[What to show the user]

## Troubleshooting

### Error: [Common error]
Cause: [Why]
Solution: [Fix]
```

## Reference Skill (Claude-Only)

```yaml
---
name: your-reference-skill
description: >-
  [Domain knowledge description]. Use when [relevant contexts].
user-invocable: false
---

# Your Reference Skill

## Core Principles

1. [Principle one]
2. [Principle two]
3. [Principle three]

## Patterns

### Pattern: [Name]
[When to use, how to apply, code example]

## Anti-Patterns

### [Bad pattern name]
[Why it's bad, what to do instead, WRONG/CORRECT examples]
```

## Forked Skill (Subagent)

```yaml
---
name: your-forked-skill
description: >-
  [What it does]. Use when [trigger conditions].
context: fork
agent: Explore
---

# Your Forked Skill

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

## Skill with Supporting Files

```
your-skill/
  SKILL.md
  PITFALLS.md
  references/
    api-guide.md
    examples.md
  scripts/
    validate.sh
```

SKILL.md references supporting files:
```markdown
## Additional Resources
- For common pitfalls, see [PITFALLS.md](PITFALLS.md)
- For API reference, see [references/api-guide.md](references/api-guide.md)
- For examples, see [references/examples.md](references/examples.md)
```

## Skill with Dynamic Context

```yaml
---
name: your-dynamic-skill
description: >-
  [What it does]. Use when [trigger conditions].
---

# Your Dynamic Skill

## Current Context
- Branch: !`git branch --show-current`
- Changed files: !`git diff --name-only`

## Instructions
Based on the context above, [do something specific].
```
