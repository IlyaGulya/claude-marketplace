---
name: skill-creator-troubleshooting
description: >-
  Diagnoses and fixes common problems with Claude Code skills.
  Use when a skill is not working, not triggering, triggering too often,
  instructions are ignored, or when the user says "my skill doesn't work",
  "skill not loading", "fix my skill", or "debug this skill".
---

# Skill Troubleshooting Guide

Diagnose the issue by matching the symptom below, then apply the fix.

## Skill Won't Load / Upload

### Error: "Could not find SKILL.md in uploaded folder"

**Cause:** File is not named exactly `SKILL.md`.

**Fix:**
- Rename to `SKILL.md` (case-sensitive)
- Verify: `ls -la` should show exactly `SKILL.md`
- Common mistakes: `skill.md`, `SKILL.MD`, `Skill.md`

### Error: "Invalid frontmatter"

**Cause:** YAML formatting issue.

**Common mistakes and fixes:**

```yaml
# WRONG — missing delimiters
name: my-skill
description: Does things

# WRONG — unclosed quotes
---
name: my-skill
description: "Does things
---

# WRONG — tab indentation (YAML requires spaces)
---
name: my-skill
description:	Does things
---

# CORRECT
---
name: my-skill
description: Does things
---
```

For multi-line descriptions, use `>-` to avoid YAML parsing issues:
```yaml
---
name: my-skill
description: >-
  This is a long description that spans
  multiple lines without issues.
---
```

### Error: "Invalid skill name"

**Cause:** Name has spaces, capitals, or forbidden characters.

```yaml
# WRONG
name: My Cool Skill
name: my_cool_skill
name: MyCoolSkill
name: claude-helper

# CORRECT
name: my-cool-skill
```

Rules: lowercase letters, numbers, and hyphens only. Max 64 characters. Cannot start with "claude" or "anthropic".

## Skill Doesn't Trigger

**Symptom:** Skill never loads automatically when it should.

### Diagnosis Steps

1. Check if `disable-model-invocation: true` is set. If so, the skill only responds to `/skill-name` — it will never auto-trigger.

2. Ask Claude: "When would you use the [skill-name] skill?" Claude will quote the description back. If it doesn't mention your skill, the description may not be loaded into context.

3. Check description quality against this checklist:
   - Is it too generic? ("Helps with projects" won't match specific queries)
   - Does it include trigger phrases users would actually say?
   - Does it mention relevant file types or technologies?
   - Is it under 1024 characters?

### Fixes

**Add specific trigger phrases:**
```yaml
# BEFORE — too abstract
description: Manages project workflows.

# AFTER — includes what users actually say
description: >-
  Manages project workflows including sprint planning, task creation,
  and status tracking. Use when user mentions "sprint", "tasks",
  "project planning", or asks to "create tickets".
```

**Add technology keywords:**
```yaml
# BEFORE
description: Generates documentation.

# AFTER
description: >-
  Generates API documentation from TypeScript and Python source code.
  Use when documenting REST endpoints, GraphQL schemas, or when user
  says "document this API" or "generate docs".
```

**Check skill budget:** If you have many skills enabled, descriptions may exceed the character budget (2% of context window, ~16,000 chars fallback). Run `/context` to check for warnings about excluded skills. Set `SLASH_COMMAND_TOOL_CHAR_BUDGET` to override.

## Skill Triggers Too Often

**Symptom:** Skill loads for unrelated queries.

### Fixes

**1. Add negative triggers:**
```yaml
description: >-
  Advanced statistical analysis for CSV files. Use for regression,
  clustering, and hypothesis testing. Do NOT use for simple data
  viewing or chart creation (use data-viz skill instead).
```

**2. Narrow the scope:**
```yaml
# TOO BROAD
description: Processes documents.

# SPECIFIC
description: >-
  Processes PDF legal documents for contract review. Use when
  reviewing contracts, extracting clauses, or analyzing legal terms.
```

**3. Add scope boundaries:**
```yaml
description: >-
  PayFlow payment processing for e-commerce. Use specifically for
  online payment workflows, not for general financial queries.
```

**4. Use `disable-model-invocation: true`** if over-triggering is unacceptable and you prefer manual invocation.

## Instructions Not Followed

**Symptom:** Skill loads but Claude ignores or partially follows instructions.

### Cause 1: Instructions too verbose

SKILL.md is too long, and critical instructions get lost in the noise.

**Fix:** Move detailed reference to supporting files. Keep SKILL.md under 500 lines. Put the most important instructions at the top.

### Cause 2: Instructions buried

Key requirements appear deep in the document.

**Fix:** Use prominent headers for critical sections:
```markdown
## CRITICAL: Validation Requirements
Before calling create_project, verify:
- Project name is non-empty
- At least one team member assigned
- Start date is not in the past
```

### Cause 3: Ambiguous language

```markdown
# BAD
Make sure to validate things properly.

# GOOD
CRITICAL: Before calling create_project, verify:
- Project name is non-empty string
- At least one team member ID in the members array
- Start date is today or later (format: YYYY-MM-DD)
```

### Cause 4: Competing instructions

Multiple skills loaded simultaneously may give conflicting guidance.

**Fix:** Make instructions self-contained. Don't assume Claude has read other skills.

### Advanced technique

For critical validations, bundle a script that performs checks programmatically:
```markdown
Before proceeding, run the validation script:
`python ${CLAUDE_SKILL_DIR}/scripts/validate.py --input $ARGUMENTS`
```
Code is deterministic; language interpretation is not.

## Performance Issues

**Symptom:** Skill seems slow or responses are degraded.

### Cause 1: SKILL.md too large

**Fix:**
- Move detailed docs to `references/` and link to them
- Keep SKILL.md under 500 lines / 5,000 words
- Use progressive disclosure (Level 1: frontmatter, Level 2: SKILL.md body, Level 3: linked files)

### Cause 2: Too many skills enabled

**Fix:**
- Evaluate if you have more than 20-50 skills enabled simultaneously
- Disable skills you don't use regularly
- Consider grouping related skills into focused sets

### Cause 3: Missing `context: fork` for heavy tasks

Skills that do extensive research or exploration should use `context: fork` to avoid polluting the main conversation context.

## MCP-Related Issues

**Symptom:** Skill loads but MCP tool calls fail.

### Diagnosis

1. **Test MCP independently:** Ask Claude to call the MCP tool directly without the skill. If this fails, the issue is the MCP connection, not the skill.

2. **Verify tool names:** Skill must reference exact MCP tool names (case-sensitive). Check your MCP server documentation.

3. **Check authentication:** API keys valid? OAuth tokens refreshed? Proper scopes granted?

4. **Check MCP server status:** Settings > Extensions > [Your Service] should show "Connected".

## Quick Diagnostic Flow

```
Skill not working
  |
  +-- Won't load?
  |     +-- Check file name (SKILL.md)
  |     +-- Check frontmatter YAML syntax
  |     +-- Check skill name format
  |
  +-- Doesn't trigger?
  |     +-- Check disable-model-invocation
  |     +-- Improve description keywords
  |     +-- Check skill budget (/context)
  |
  +-- Triggers too much?
  |     +-- Add negative triggers
  |     +-- Narrow description scope
  |     +-- Use disable-model-invocation
  |
  +-- Instructions ignored?
  |     +-- Shorten SKILL.md
  |     +-- Move critical rules to top
  |     +-- Be specific, not vague
  |
  +-- Slow/degraded?
        +-- Reduce SKILL.md size
        +-- Use context: fork
        +-- Reduce enabled skill count
```
