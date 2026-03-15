---
name: skill-creator-guide
description: >-
  Interactive guide for creating new Claude Code skills from scratch.
  Use when building a new skill, creating a SKILL.md, designing skill structure,
  or when the user says "create a skill", "build a skill", "make a new skill",
  or "help me write a skill".
---

# Skill Creator Guide

This is an interactive, step-by-step workflow for building a complete Claude Code skill.
Follow each phase in order. Do not skip steps.

## Phase 1: Define the Use Case

Before writing anything, clarify exactly what the skill should do.

Ask the user (if not already specified):
1. **What task does this skill accomplish?** (e.g., "generate API documentation", "review PRs", "deploy to staging")
2. **Who invokes it?** User-only (`disable-model-invocation: true`), Claude-only (`user-invocable: false`), or both (default)?
3. **What tools does it need?** Built-in (Read, Write, Bash, Grep, Glob), MCP tools, or none?
4. **Where should it live?** Personal (`~/.claude/skills/`), project (`.claude/skills/`), or plugin?

Write down 2-3 concrete use cases in this format:

```
Use Case: [Name]
Trigger: User says "[example phrase]"
Steps:
  1. [First action]
  2. [Second action]
  3. [Third action]
Result: [What the user gets]
```

## Phase 2: Choose Skill Type

Determine which category fits best:

**Reference content** — Knowledge Claude applies to current work. Conventions, patterns, style guides.
- Runs inline (no `context: fork`)
- No `disable-model-invocation`
- Example: API conventions, coding standards, domain knowledge

**Task content** — Step-by-step instructions for a specific action.
- Often uses `disable-model-invocation: true`
- May use `context: fork` for isolation
- Example: deploy, commit, generate report

**Hybrid** — Reference knowledge + actionable workflow.
- Default invocation settings
- Example: code review skill that knows your standards and applies them

## Phase 3: Design the Structure

Determine file layout:

```
your-skill-name/
  SKILL.md           # Required — main instructions
  references/        # Optional — detailed docs loaded on demand
  scripts/           # Optional — executable code
  assets/            # Optional — templates, fonts, icons
```

Rules:
- Folder name: kebab-case only (e.g., `my-cool-skill`)
- No spaces, underscores, or capitals in the folder name
- Do NOT include README.md inside the skill folder
- Keep SKILL.md under 500 lines; move detailed reference material to separate files

## Phase 4: Write the Frontmatter

Create the YAML frontmatter between `---` markers.

**Required fields:**
- `name`: kebab-case, matches folder name, max 64 characters
- `description`: What it does + when to use it + key capabilities. Under 1024 characters. No XML angle brackets.

**Optional fields** (add only if needed):
- `disable-model-invocation: true` — user-only invocation
- `user-invocable: false` — Claude-only invocation
- `allowed-tools` — restrict tool access (e.g., `Read, Grep, Glob`)
- `context: fork` — run in isolated subagent
- `agent` — subagent type when using `context: fork` (e.g., `Explore`, `Plan`)
- `argument-hint` — autocomplete hint (e.g., `[issue-number]`)
- `model` — model override

Description formula:
```
[What it does]. Use when [trigger conditions]. Covers [key capabilities].
```

Good example:
```yaml
description: >-
  Generates API documentation from source code with consistent formatting.
  Use when writing API docs, creating endpoint references, or when the user
  says "document this API" or "generate docs". Covers REST and GraphQL endpoints.
```

Bad example:
```yaml
description: Helps with documentation.
```

## Phase 5: Write the Instructions

Structure the SKILL.md body using this template:

```markdown
# Skill Name

## Instructions

### Step 1: [First Major Step]
Clear explanation of what happens.
- Specific, actionable instructions
- Include tool calls or commands when relevant

### Step 2: [Second Major Step]
...

## Examples

### Example 1: [Common scenario]
User says: "[example prompt]"
Actions:
1. [What Claude does]
2. [What Claude does]
Result: [What the user gets]

## Troubleshooting

### Error: [Common error]
Cause: [Why it happens]
Solution: [How to fix]
```

Best practices for instructions:
- Be specific and actionable — say exactly what to do, not "handle it properly"
- Use numbered lists for sequential steps
- Use bullet points for non-ordered items
- Include code examples where relevant
- Put critical instructions at the top
- Reference supporting files clearly: "For API patterns, see [references/api-patterns.md](references/api-patterns.md)"

## Phase 6: Add Supporting Files (if needed)

If SKILL.md exceeds ~300 lines, split into:
- `references/` — detailed API docs, specifications
- `PITFALLS.md` — common mistakes with WRONG/CORRECT examples
- `EXAMPLES.md` — extended examples
- `scripts/` — executable helpers

Reference them from SKILL.md:
```markdown
## Additional Resources
- For complete API reference, see [references/api-guide.md](references/api-guide.md)
- For common pitfalls, see [PITFALLS.md](PITFALLS.md)
```

## Phase 7: Add String Substitutions (if needed)

If the skill accepts arguments, use these placeholders:
- `$ARGUMENTS` — all arguments passed to the skill
- `$ARGUMENTS[0]`, `$ARGUMENTS[1]` or `$0`, `$1` — positional arguments
- `${CLAUDE_SESSION_ID}` — current session ID
- `${CLAUDE_SKILL_DIR}` — directory containing this SKILL.md

For dynamic context, use shell command injection:
- `!` followed by a backtick-wrapped command runs before the skill loads
- Example: `!`git branch --show-current`` injects the current branch name

## Phase 8: Validate

Run through this checklist before considering the skill complete.
For the full pre-flight checklist, see [CHECKLIST.md](CHECKLIST.md).

Quick validation:
1. Folder is kebab-case
2. File is exactly `SKILL.md` (case-sensitive)
3. Frontmatter has `---` delimiters
4. `name` is kebab-case, no spaces, no capitals
5. `description` includes WHAT and WHEN
6. No XML angle brackets anywhere
7. Instructions are specific and actionable
8. Name does not start with "claude" or "anthropic"

## Phase 9: Test

Test the skill in three ways:

**1. Triggering tests** — Does it activate on the right queries?
- Try 3-5 phrases that should trigger it
- Try 2-3 phrases that should NOT trigger it
- If under-triggering: add more keywords to description
- If over-triggering: make description more specific

**2. Functional tests** — Does it produce correct output?
- Run the skill on a real task
- Verify all steps execute correctly
- Check edge cases

**3. Comparison test** — Is it better than no skill?
- Do the same task without the skill
- Compare: fewer messages? Fewer errors? More consistent output?

## Output

After completing all phases, present the user with:
1. The complete SKILL.md file
2. Any supporting files
3. The directory structure
4. Installation instructions for their chosen location
