---
name: context-file-lint
description: >-
  Analyzes CLAUDE.md, AGENTS.md, and similar repository context files for effectiveness
  based on peer-reviewed research. Identifies instructions that hurt agent performance,
  redundant content, and unnecessary requirements. Use when reviewing a CLAUDE.md,
  auditing an AGENTS.md, optimizing context files, or when the user says "lint my CLAUDE.md",
  "review my AGENTS.md", "is my context file good", "optimize my context file",
  or "check my CLAUDE.md".
---

# Context File Linter

Analyze repository context files (CLAUDE.md, AGENTS.md, etc.) against findings from peer-reviewed research on context file effectiveness. This analysis is based on "Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?" (Gloaguen, Mundler, Muller, Raychev, Vechev — ETH Zurich / LogicStar.ai, 2026).

For the full research summary, see [RESEARCH-FINDINGS.md](RESEARCH-FINDINGS.md).

## How to Use

1. Read the target context file (CLAUDE.md, AGENTS.md, .cursorrules, etc.)
2. If available, also read the project's README.md, pyproject.toml/package.json, and linter configs to check for redundancy
3. Evaluate each section of the context file against the criteria below
4. Output the lint report

## Evaluation Criteria

Analyze every section of the context file using these 7 research-backed checks. Each check produces PASS, WARN, or FAIL.

### Check 1: Codebase Overview Bloat

**Research finding:** Codebase overviews (directory listings, component descriptions) do NOT help agents find relevant files faster. In 100% of tested LLM-generated files, overviews were present. They provided zero measurable benefit.

**FAIL if the section:**
- Lists directories and describes what each one contains
- Enumerates project components/modules with descriptions
- Provides a "project structure" or "architecture overview" tree
- Describes file organization patterns

**PASS if:**
- No codebase overview present, OR
- Overview is limited to one sentence about what the project does (not how it's organized)

**Recommendation:** Remove codebase overviews entirely. Agents discover structure faster on their own through file exploration tools.

### Check 2: Documentation Redundancy

**Research finding:** LLM-generated context files are highly redundant with existing documentation. When all other docs were removed, context files actually helped (+2.7%). This proves context files duplicate existing docs, and that duplication hurts.

**FAIL if the section:**
- Repeats information from README.md (setup instructions, usage, etc.)
- Duplicates what's in pyproject.toml / package.json / Cargo.toml (dependencies, scripts)
- Restates linter/formatter rules that are in .eslintrc / .prettierrc / ruff.toml / pyproject.toml [tool.ruff]
- Describes CI/CD pipelines already defined in .github/workflows or similar
- Repeats TypeScript config from tsconfig.json
- Describes test setup already configured in pytest.ini / vitest.config / jest.config

**PASS if:**
- Information is NOT available anywhere else in the repository
- Or explicitly clarifies something that's ambiguous in existing config

**Recommendation:** Before adding any instruction, check: "Can the agent discover this from existing files?" If yes, don't add it.

### Check 3: Unnecessary Requirements

**Research finding:** Context file instructions ARE followed by agents. This means unnecessary requirements actively make tasks harder. Every followed instruction costs extra steps, tokens, and reasoning. Across all models and agents, context files increased steps by 2-4 per task and cost by 20-23%.

**FAIL if the section:**
- Mandates processes the agent would do anyway (e.g., "read the code before making changes", "understand the codebase first")
- Requires generic best practices the model already knows (e.g., "write clean code", "follow SOLID principles", "use meaningful variable names")
- Demands excessive verification steps (e.g., "always run the full test suite before and after every change")
- Specifies obvious error handling patterns (e.g., "handle errors properly", "add try-catch blocks")
- Lists coding standards that are already enforced by linters/formatters

**WARN if the section:**
- Provides useful but verbose instructions that could be shorter
- Describes processes that are standard for the language/framework

**PASS if:**
- Each requirement is specific, non-obvious, and necessary for this particular project
- Requirements address things the agent would NOT know from the code alone

**Recommendation:** For each instruction, ask: "If I remove this, will the agent produce meaningfully worse results?" If the answer is unclear, remove it.

### Check 4: Generic Best Practices

**Research finding:** Agents follow instructions, so generic advice (code quality, security, testing) causes agents to spend more tokens reasoning about compliance rather than solving the task. Reasoning token usage increased 10-22% with context files.

**FAIL if the section:**
- Lists language-agnostic coding principles (DRY, KISS, YAGNI, etc.)
- Describes generic testing philosophy ("write tests for all new code")
- Provides security advice that applies to any project ("sanitize inputs", "don't expose secrets")
- Specifies documentation standards ("add docstrings to all public functions")
- Describes git workflow rules ("use conventional commits", "write clear commit messages")

**PASS if:**
- Advice is specific to THIS project's unique patterns or constraints
- Example: "We use a custom ORM — always use `db.query()` instead of raw SQL because it handles connection pooling"

**Recommendation:** Remove all generic advice. Only keep project-specific rules that an agent couldn't infer from the code.

### Check 5: LLM-Generated Content

**Research finding:** LLM-generated context files have a negative effect on task success (average -0.5% to -3%) compared to no context file at all. Auto-generated files via /init commands consistently performed worse than having no context file.

**WARN if the file:**
- Appears to be auto-generated (boilerplate structure, generic language, mentions of `/init`)
- Contains placeholder-like sections with obvious template language
- Has the characteristic structure of Claude's `/init` output or Codex's init

**Recommendation:** If the file was auto-generated, consider deleting it entirely and writing only the 3-5 instructions that actually matter.

### Check 6: Effective Minimal Requirements

**Research finding:** Developer-written context files provided marginal improvement (+4%) when they contained minimal, specific requirements — particularly non-obvious tooling choices (e.g., "use `uv` instead of `pip`", "run tests with `pdm run pytest`").

**PASS if the section contains:**
- Non-obvious tooling requirements (specific package manager, test runner, build tool)
- Project-specific conventions that differ from language defaults
- Required environment setup that isn't in standard config files
- Specific commands for non-trivial operations (e.g., database migrations, asset compilation)
- Domain rules that affect correctness (e.g., "monetary values must use Decimal, never float")

**These are the ONLY types of instructions research shows are helpful.**

### Check 7: Size and Token Cost

**Research finding:** Every instruction in a context file increases the agent's step count and inference cost. Files should be as short as possible.

**FAIL if:**
- File exceeds 500 words (the average developer-written file was 641 words, and even those provided only marginal benefit)
- File contains large code blocks that could be discovered from the codebase

**WARN if:**
- File is 200-500 words
- Could be shortened without losing information

**PASS if:**
- File is under 200 words
- Every sentence carries unique, non-obvious information

## Output Format

Present the analysis as:

```
## Context File Lint Report: [filename]

### Summary
- Total sections analyzed: [N]
- PASS: [N] | WARN: [N] | FAIL: [N]
- Estimated token overhead: [low/medium/high]
- Overall verdict: [EFFECTIVE / NEEDS WORK / COUNTERPRODUCTIVE]

### Section-by-Section Analysis

#### Section: "[section heading]"
[Quote the section or summarize it]

| Check | Result | Detail |
|-------|--------|--------|
| Codebase Overview | PASS/WARN/FAIL | [why] |
| Redundancy | PASS/WARN/FAIL | [why] |
| Unnecessary Requirements | PASS/WARN/FAIL | [why] |
| Generic Best Practices | PASS/WARN/FAIL | [why] |
| LLM-Generated | PASS/WARN/FAIL | [why] |
| Minimal Requirements | PASS/WARN/FAIL | [why] |

(repeat for each section)

### Token Cost Analysis
| Check | Result | Detail |
|-------|--------|--------|
| Size | PASS/WARN/FAIL | [word count, assessment] |

### Recommended Actions
1. **REMOVE:** [sections/instructions to delete entirely]
2. **KEEP:** [sections/instructions that are research-backed effective]
3. **REWRITE:** [sections that have useful content but need to be shortened/focused]

### Suggested Minimal Version
[Write a concise replacement that keeps only research-backed effective content]
```

## Key Principles (from the research)

When writing the suggested minimal version:

1. **Only include what the agent can't discover from existing files** — no README duplication, no config file summaries
2. **Only include project-specific non-obvious requirements** — unique tooling, unconventional patterns, critical domain rules
3. **Keep it under 200 words** — every word costs tokens and potentially hurts performance
4. **No codebase overviews** — agents navigate better without them
5. **No generic best practices** — the model already knows them
6. **Specific > general** — "use `uv run pytest` for tests" beats "make sure to test your changes"
7. **When in doubt, leave it out** — the research shows that no context file outperforms a bad one
