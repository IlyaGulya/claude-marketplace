# Research Findings: Context File Effectiveness

Summary of "Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?" (Gloaguen, Mundler, Muller, Raychev, Vechev — ETH Zurich / LogicStar.ai, 2026).

## Study Design

- **Benchmark:** AGENTBENCH — 138 instances from 12 real open-source repos with developer-committed context files
- **Also tested on:** SWE-BENCH LITE — 300 tasks from 11 popular Python repos
- **Agents tested:** Claude Code (Sonnet-4.5), Codex (GPT-5.2), Codex (GPT-5.1 Mini), Qwen Code (Qwen3-30B)
- **Three settings:** No context file, LLM-generated context file, developer-written context file
- **Metric:** Task success rate (% of patches that pass all tests)

## Key Quantitative Results

### LLM-Generated Context Files

| Metric | Impact |
|--------|--------|
| Success rate (SWE-BENCH LITE) | -0.5% average |
| Success rate (AGENTBENCH) | -2% average |
| Steps per task | +2.45 (SWE-BENCH), +3.92 (AGENTBENCH) |
| Inference cost | +20% (SWE-BENCH), +23% (AGENTBENCH) |

**Verdict: LLM-generated context files are worse than no context file.**

### Developer-Written Context Files

| Metric | Impact |
|--------|--------|
| Success rate (AGENTBENCH) | +4% average (3 out of 4 agents improved) |
| Steps per task | +3.34 average |
| Inference cost | up to +19% |

**Verdict: Marginal improvement, but at significant cost.**

### Cost Breakdown (Average USD per Instance)

| Agent | No Context | LLM-Generated | Developer-Written |
|-------|-----------|---------------|-------------------|
| Sonnet-4.5 | $1.15 | $1.33 (+16%) | $1.30 (+13%) |
| GPT-5.2 | $0.38 | $0.57 (+50%) | $0.54 (+42%) |
| GPT-5.1 Mini | $0.18 | $0.20 (+11%) | $0.19 (+6%) |
| Qwen3-30B | $0.13 | $0.15 (+15%) | $0.15 (+15%) |

## Key Behavioral Findings

### 1. Codebase Overviews Don't Work

- 100% of Sonnet-4.5-generated files contained codebase overviews
- 95-99% of other LLM-generated files contained overviews
- **Zero measurable improvement** in how quickly agents found relevant files
- Agents with context files did NOT interact with PR-relevant files any faster

### 2. Context Files ARE Followed

- When `uv` was mentioned in context files: used 1.6 times/instance vs 0.01 times when not mentioned
- When repo-specific tools were mentioned: used 2.5 times/instance vs 0.05 times when not mentioned
- **This is the core problem: instructions are followed, so bad instructions actively hurt**

### 3. More Instructions = More Thinking

- GPT-5.2 reasoning tokens: +22% with LLM-generated files (SWE-BENCH LITE)
- GPT-5.1 Mini reasoning tokens: +14% with LLM-generated files
- Developer-written files also increased reasoning by 2-20%
- The agent spends tokens reasoning about compliance instead of solving the task

### 4. Context Files Are Redundant With Docs

- When all documentation (.md files, docs/, examples) was REMOVED from repos, context files became helpful (+2.7%)
- This proves: context files duplicate existing docs, and that duplication hurts performance
- LLM-generated files even outperformed developer-written ones when docs were removed

### 5. Stronger Models Don't Generate Better Context Files

- GPT-5.2-generated context files: +2% on SWE-BENCH LITE but -3% on AGENTBENCH vs own-model generation
- No consistent improvement from using a stronger model to generate context files

### 6. Prompt Choice Doesn't Matter Much

- Claude Code prompt vs Codex prompt for generating context files: no consistent winner
- Sensitivity to different good prompts is generally small

## What Works (The Only Effective Instructions)

Based on the research, the only context file content with measurable positive impact:

1. **Non-obvious tooling requirements** — e.g., "use `uv` instead of `pip`", "run tests with `pdm run pytest`"
2. **Project-specific commands** — specific build/test/deploy commands that aren't in standard config
3. **Domain-critical rules** — correctness requirements an agent couldn't infer from code
4. **Unconventional patterns** — where the project deliberately deviates from language/framework norms

## What Hurts (Instructions to Avoid)

1. **Codebase overviews / directory listings** — zero benefit, adds tokens
2. **Information already in README/configs** — redundancy hurts
3. **Generic coding best practices** — the model already knows them, and following them wastes steps
4. **Excessive testing/verification mandates** — agents already test, extra mandates add cost
5. **Style/formatting rules** — should be in linter configs, not context files
6. **Obvious process instructions** — "read the code first", "understand the problem" etc.

## Recommended Approach

> "We suggest omitting LLM-generated context files for the time being, contrary to agent developers' recommendations, and including only minimal requirements." — Paper conclusion

1. **Do NOT auto-generate context files** — they are worse than nothing
2. **Write manually, if at all** — only developer-written files showed any benefit
3. **Keep it minimal** — average effective file was ~641 words, but shorter is better
4. **Only non-discoverable information** — if the agent can find it in existing files, don't repeat it
5. **Test the impact** — remove the context file and see if output quality actually decreases
