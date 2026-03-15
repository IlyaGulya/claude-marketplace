---
name: skill-creator-review
description: >-
  Reviews and validates existing Claude Code skills for correctness, quality, and best practices.
  Use when reviewing a skill, auditing SKILL.md files, checking skill quality,
  or when the user says "review this skill", "validate my skill", "check my SKILL.md",
  or "is this skill correct".
---

# Skill Review and Validation

Review the given skill by checking each category below. Report findings as PASS, WARN, or FAIL with specific fixes.

## 1. Structure Validation

Check these hard requirements. Any failure here means the skill will not work:

| Check | Requirement |
|-------|-------------|
| File name | Exactly `SKILL.md` (case-sensitive) |
| Folder name | kebab-case only (no spaces, underscores, capitals) |
| Frontmatter | Starts and ends with `---` on separate lines |
| Name field | kebab-case, matches folder name, max 64 chars |
| Reserved names | Does NOT start with "claude" or "anthropic" |
| No XML | No angle brackets (`<` `>`) in frontmatter values |
| No README.md | Skill folder does not contain a README.md |

## 2. Description Quality

The description is the most critical field — it determines when Claude loads the skill.

**FAIL conditions:**
- Missing description entirely
- Over 1024 characters
- Contains XML angle brackets

**WARN conditions:**
- Too vague (e.g., "Helps with projects")
- Missing trigger phrases (no "Use when..." clause)
- Missing capability list
- Too technical without user-facing language

**PASS criteria:**
- States WHAT the skill does
- States WHEN to use it (with specific trigger phrases)
- Lists KEY CAPABILITIES
- Under 1024 characters
- Includes words users would actually say

Evaluate against this formula:
```
[What it does] + [When to use it] + [Key capabilities]
```

## 3. Instruction Quality

Review the SKILL.md body for:

**Specificity** — Instructions must be actionable, not vague.
- FAIL: "Validate the data before proceeding"
- PASS: "Run `python scripts/validate.py --input {file}` to check data format"

**Structure** — Should use clear headers and ordered steps.
- WARN if no numbered steps for sequential workflows
- WARN if walls of text without headers or bullet points

**Examples** — Should include at least one concrete example.
- WARN if no examples section
- FAIL if task skill has no usage example

**Error handling** — Should address common failure modes.
- WARN if no troubleshooting section
- FAIL for MCP-dependent skills with no error handling

**Progressive disclosure** — SKILL.md should stay focused.
- WARN if over 500 lines without supporting files
- WARN if large reference tables inline instead of in references/

## 4. Invocation Configuration

Check that frontmatter fields match the skill's intended use:

| Skill Type | Expected Configuration |
|------------|----------------------|
| Side-effect task (deploy, send, delete) | `disable-model-invocation: true` |
| Background knowledge | `user-invocable: false` |
| Read-only exploration | `allowed-tools: Read, Grep, Glob` |
| Isolated heavy task | `context: fork` with `agent` specified |
| Accepts arguments | `$ARGUMENTS` used in body, `argument-hint` set |

WARN if a skill with destructive actions (deploy, delete, send) does not have `disable-model-invocation: true`.

## 5. Triggering Risk Assessment

Evaluate the description for triggering risks:

**Under-triggering risk (skill won't activate when it should):**
- Description is too specific or technical
- Missing common phrasings users would say
- No mention of relevant file types

**Over-triggering risk (skill activates when it shouldn't):**
- Description is too broad (e.g., "helps with code")
- Overlaps with other common skills
- No scope boundaries

Suggest negative triggers if over-triggering is likely:
```yaml
description: >-
  Advanced data analysis for CSV files. Use for statistical modeling,
  regression, clustering. Do NOT use for simple data viewing or chart creation.
```

## 6. Supporting Files

If supporting files exist, check:
- Each file is referenced from SKILL.md
- File names are descriptive
- No orphan files that SKILL.md never mentions
- Scripts use `${CLAUDE_SKILL_DIR}` for portable paths

## Output Format

Present the review as:

```
## Skill Review: [skill-name]

### Structure: [PASS/WARN/FAIL]
[Details]

### Description: [PASS/WARN/FAIL]
[Details + suggested improvement if not PASS]

### Instructions: [PASS/WARN/FAIL]
[Details]

### Invocation: [PASS/WARN/FAIL]
[Details]

### Triggering: [PASS/WARN/FAIL]
[Under/over-triggering assessment]

### Supporting Files: [PASS/WARN/FAIL/N/A]
[Details]

### Overall: [PASS/WARN/FAIL]
[Summary + top 3 action items if any]
```
