---
name: skill-creator-patterns
description: >-
  Common patterns and architectures for building Claude Code skills.
  Use when deciding how to structure a skill, choosing between skill patterns,
  designing multi-step workflows, or when the user asks "what kind of skill should I build"
  or "how should I structure this skill".
---

# Skill Patterns

Choose the pattern that best fits your skill's purpose. Most skills use one primary pattern, sometimes combining elements from others.

## Pattern 1: Sequential Workflow

**Use when:** Users need multi-step processes executed in a specific order.

**Key techniques:**
- Explicit step ordering with numbered steps
- Dependencies between steps (output of step N feeds step N+1)
- Validation gates between steps
- Rollback instructions for failures

**Structure:**

```markdown
## Workflow: [Name]

### Step 1: [Action]
[What to do]
Expected output: [What success looks like]

### Step 2: [Action]
Requires: Output from Step 1
[What to do]

### Step 3: [Action]
[What to do]

## Rollback
If Step 2 fails:
1. [Undo Step 1]
2. [Report error]
```

**Real-world examples:** deployment pipelines, onboarding workflows, data migration scripts, release processes.

## Pattern 2: Reference Knowledge

**Use when:** Claude needs domain expertise to apply during regular work.

**Key techniques:**
- Core principles up front
- Decision tables for common choices
- Code examples showing correct vs incorrect patterns
- Anti-patterns section with WRONG/CORRECT comparisons
- Progressive disclosure (main concepts in SKILL.md, details in references/)

**Structure:**

```markdown
# [Domain] Reference

## Core Principles
1. [Most important rule]
2. [Second rule]
3. [Third rule]

## Decision Guide

| Situation | Approach | Why |
|-----------|----------|-----|
| [Case A] | [Do X] | [Reason] |
| [Case B] | [Do Y] | [Reason] |

## Patterns

### [Pattern Name]
When to use: [condition]

[Code example]

## Anti-Patterns

### [Bad Pattern]
WRONG:
[code]

CORRECT:
[code]

Why: [explanation]
```

**Real-world examples:** coding conventions, API design guides, testing standards, architectural patterns.

## Pattern 3: Iterative Refinement

**Use when:** Output quality improves with multiple passes.

**Key techniques:**
- Generate initial draft
- Quality check against explicit criteria
- Refinement loop with re-validation
- Clear stopping condition

**Structure:**

```markdown
## Phase 1: Initial Draft
1. [Generate first version]
2. [Save to file]

## Phase 2: Quality Check
Review against these criteria:
- [ ] [Quality criterion 1]
- [ ] [Quality criterion 2]
- [ ] [Quality criterion 3]

## Phase 3: Refinement
For each failing criterion:
1. Identify the specific issue
2. Fix it
3. Re-check

Repeat until all criteria pass or 3 iterations completed.

## Phase 4: Finalization
1. Apply final formatting
2. Generate summary of changes
```

**Real-world examples:** code review, report generation, documentation writing, test coverage improvement.

## Pattern 4: Context-Aware Routing

**Use when:** The same goal requires different approaches depending on context.

**Key techniques:**
- Decision tree based on input characteristics
- Different tool selection per branch
- Fallback options
- Transparency about choices made

**Structure:**

```markdown
## Decision Tree

### Check 1: [Condition]
- If [A]: proceed to Path A
- If [B]: proceed to Path B
- Otherwise: proceed to Default Path

### Path A: [Name]
[Specific instructions for this case]

### Path B: [Name]
[Specific instructions for this case]

### Default Path
[Safe fallback instructions]

## Reporting
Always explain which path was chosen and why.
```

**Real-world examples:** file storage routing, environment-specific deployment, language-specific code generation.

## Pattern 5: Multi-Tool Coordination

**Use when:** Workflows span multiple tools or MCP servers.

**Key techniques:**
- Clear phase separation by tool/service
- Data passing between phases
- Validation before moving to next phase
- Centralized error handling

**Structure:**

```markdown
## Phase 1: [Service A]
1. [Action using Service A tools]
2. Capture: [output to pass forward]

## Phase 2: [Service B]
Input: [data from Phase 1]
1. [Action using Service B tools]
2. Capture: [output to pass forward]

## Phase 3: [Service C]
Input: [data from Phase 2]
1. [Action using Service C tools]

## Error Handling
- Phase 1 failure: [recovery steps]
- Phase 2 failure: [recovery steps]
- Phase 3 failure: [recovery steps]
```

**Real-world examples:** design-to-development handoff (Figma + Drive + Linear + Slack), CI/CD pipeline orchestration, cross-platform content publishing.

## Pattern 6: Forked Research

**Use when:** Heavy exploration that should not pollute the main conversation context.

**Key techniques:**
- `context: fork` in frontmatter
- `agent: Explore` for read-only research
- Focused task in SKILL.md body
- Results summarized and returned to main conversation

**Structure:**

```yaml
---
name: deep-research
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Identify patterns and relationships
4. Summarize findings with specific file:line references
```

**Real-world examples:** codebase exploration, dependency analysis, architecture review, security audit.

## Choosing the Right Pattern

| Your skill needs to... | Best pattern |
|------------------------|-------------|
| Execute steps in order | Sequential Workflow |
| Teach Claude domain knowledge | Reference Knowledge |
| Produce high-quality output | Iterative Refinement |
| Adapt to different inputs | Context-Aware Routing |
| Coordinate multiple tools/services | Multi-Tool Coordination |
| Explore without polluting context | Forked Research |

Many skills combine patterns. A deployment skill might use Sequential Workflow as the primary pattern with Context-Aware Routing to handle different environments, and Iterative Refinement for the validation step.

## Skill Sizing Guidelines

| Skill Size | Lines in SKILL.md | Supporting Files | Use Case |
|------------|-------------------|------------------|----------|
| Small | < 100 | None | Single concept, simple reference |
| Medium | 100-300 | 0-2 | Workflow, moderate reference |
| Large | 300-500 | 2-5 | Complex workflow, extensive reference |

If SKILL.md exceeds 500 lines, split content into supporting files. Keep SKILL.md focused on core instructions and navigation.
