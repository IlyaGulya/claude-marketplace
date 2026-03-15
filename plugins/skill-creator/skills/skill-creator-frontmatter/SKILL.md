---
name: skill-creator-frontmatter
description: >-
  Complete reference for Claude Code skill YAML frontmatter fields, their values,
  and best practices. Use when writing frontmatter for SKILL.md, configuring skill
  invocation, setting allowed-tools, or deciding on skill metadata fields.
---

# SKILL.md Frontmatter Reference

The YAML frontmatter between `---` markers at the top of SKILL.md controls how Claude discovers and runs your skill. The frontmatter is loaded into Claude's system prompt (Level 1 of progressive disclosure), so keep it concise.

## Required Format

```yaml
---
name: your-skill-name
description: What it does. Use when user asks to [specific phrases].
---
```

Both `---` delimiters must appear on their own lines. Everything between them is parsed as YAML.

## Field Reference

### name

Display name and slash-command identifier.

- **Type:** string
- **Required:** No (defaults to folder name)
- **Constraints:** kebab-case only, max 64 characters, lowercase letters/numbers/hyphens
- **Must match** the skill's folder name
- **Reserved prefixes:** "claude", "anthropic" (will be rejected)

```yaml
# CORRECT
name: deploy-staging
name: api-conventions
name: fix-issue

# WRONG
name: Deploy Staging     # spaces and capitals
name: deploy_staging     # underscores
name: DeployStaging      # camelCase
name: claude-helper      # reserved prefix
```

### description

What the skill does and when Claude should load it. This is the most impactful field.

- **Type:** string
- **Required:** Recommended (falls back to first paragraph of body)
- **Constraints:** max 1024 characters, no XML angle brackets (`<` `>`)
- **Formula:** `[What it does] + [When to use it] + [Key capabilities]`

```yaml
# GOOD — specific, with trigger phrases
description: >-
  Generates API documentation from source code with consistent formatting.
  Use when writing API docs, creating endpoint references, or when the user
  says "document this API" or "generate docs". Covers REST and GraphQL.

# GOOD — includes negative triggers to prevent over-triggering
description: >-
  Advanced statistical analysis for CSV datasets. Use for regression,
  clustering, and hypothesis testing. Do NOT use for simple data viewing
  or chart creation (use data-viz skill instead).

# BAD — too vague
description: Helps with projects.

# BAD — no trigger conditions
description: Creates sophisticated multi-page documentation systems.
```

### disable-model-invocation

Prevents Claude from automatically loading this skill.

- **Type:** boolean
- **Default:** `false`
- **Effect:** When `true`, only the user can invoke via `/skill-name`. Description is NOT loaded into context.

Use for skills with side effects: deploy, send messages, delete resources, commit code.

```yaml
disable-model-invocation: true
```

### user-invocable

Controls whether the skill appears in the `/` menu.

- **Type:** boolean
- **Default:** `true`
- **Effect:** When `false`, skill is hidden from the user menu. Only Claude can invoke it. Description IS loaded into context.

Use for background knowledge that isn't actionable as a command.

```yaml
user-invocable: false
```

### Invocation Matrix

| Configuration | User invokes | Claude invokes | Description in context |
|---------------|-------------|----------------|----------------------|
| (default) | Yes | Yes | Yes |
| `disable-model-invocation: true` | Yes | No | No |
| `user-invocable: false` | No | Yes | Yes |

### allowed-tools

Restricts which tools Claude can use when the skill is active.

- **Type:** string (comma-separated or space-separated tool names)
- **Default:** All tools available
- **Common values:** `Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, `Agent`, `WebFetch`, `WebSearch`
- **Bash patterns:** `Bash(python *)`, `Bash(npm *)`, `Bash(gh *)`

```yaml
# Read-only exploration
allowed-tools: Read, Grep, Glob

# Script execution only
allowed-tools: "Bash(python:*) Bash(npm:*) WebFetch"

# Full access (default — omit field entirely)
```

### context

Controls execution environment.

- **Type:** string
- **Values:** `fork` (only supported value)
- **Effect:** Runs the skill in an isolated subagent. The skill content becomes the subagent's task. No access to conversation history.

Only use `context: fork` when the skill has explicit task instructions. Reference-only skills should NOT use fork.

```yaml
context: fork
```

### agent

Which subagent type to use when `context: fork` is set.

- **Type:** string
- **Values:** `Explore`, `Plan`, `general-purpose`, or custom agent name from `.claude/agents/`
- **Default:** `general-purpose`
- **Requires:** `context: fork`

```yaml
context: fork
agent: Explore    # Read-only codebase exploration
```

### argument-hint

Hint shown during `/` autocomplete to indicate expected arguments.

- **Type:** string
- **Effect:** Displays in the autocomplete menu after the skill name

```yaml
argument-hint: "[issue-number]"
argument-hint: "[filename] [format]"
argument-hint: "[component-name] [from-framework] [to-framework]"
```

### model

Override the model used when this skill is active.

- **Type:** string

```yaml
model: claude-sonnet-4-6
```

### hooks

Hooks scoped to this skill's lifecycle.

- **Type:** object
- **See:** Claude Code hooks documentation for format

### license

License for open-source distribution.

- **Type:** string
- **Common values:** `MIT`, `Apache-2.0`

### compatibility

Environment requirements.

- **Type:** string (max 500 characters)
- **Use for:** required system packages, network access needs, intended product

### metadata

Custom key-value pairs for any additional information.

- **Type:** object

```yaml
metadata:
  author: Your Name
  version: 1.0.0
  mcp-server: server-name
  category: productivity
  tags: [automation, deployment]
```

## Security Restrictions

**Forbidden in frontmatter:**
- XML angle brackets (`<` `>`) — frontmatter appears in Claude's system prompt
- Skills with "claude" or "anthropic" in the name (reserved)

**Why:** Frontmatter is injected into the system prompt. Malicious content could be used for prompt injection.
