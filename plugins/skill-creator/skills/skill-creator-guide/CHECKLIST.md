# Skill Creation Checklist

Use this checklist to validate a skill before and after deployment.

## Before You Start

- [ ] Identified 2-3 concrete use cases
- [ ] Determined which tools are needed (built-in, MCP, or none)
- [ ] Decided on skill location (personal, project, or plugin)
- [ ] Planned file and folder structure

## During Development

- [ ] Folder named in kebab-case (no spaces, underscores, or capitals)
- [ ] `SKILL.md` file exists with exact casing
- [ ] YAML frontmatter has `---` delimiters on both sides
- [ ] `name` field is kebab-case, matches folder name
- [ ] `description` includes both WHAT the skill does and WHEN to use it
- [ ] No XML angle brackets (`<` or `>`) in frontmatter
- [ ] `name` does not start with "claude" or "anthropic" (reserved)
- [ ] Description is under 1024 characters
- [ ] Instructions are clear, specific, and actionable
- [ ] Error handling or troubleshooting section included
- [ ] Examples provided for common scenarios
- [ ] Supporting files referenced clearly from SKILL.md
- [ ] SKILL.md is under 500 lines (detailed content moved to supporting files)
- [ ] No `README.md` inside the skill folder

## Frontmatter Validation

- [ ] `disable-model-invocation` set correctly (true if user-only)
- [ ] `user-invocable` set correctly (false if Claude-only)
- [ ] `allowed-tools` restricts tools appropriately (if needed)
- [ ] `context: fork` set if skill should run in isolation
- [ ] `agent` specified if using `context: fork`
- [ ] `argument-hint` set if skill accepts arguments

## Before Deployment

- [ ] Tested triggering on 3-5 obvious queries
- [ ] Tested triggering on paraphrased requests
- [ ] Verified skill does NOT trigger on unrelated queries
- [ ] Functional tests pass (correct output on real tasks)
- [ ] Tool integration works (MCP connections, scripts, etc.)
- [ ] Tested with and without arguments (if applicable)

## After Deployment

- [ ] Tested in real conversations
- [ ] Monitored for under-triggering (skill doesn't load when it should)
- [ ] Monitored for over-triggering (skill loads for unrelated queries)
- [ ] Collected user feedback
- [ ] Iterated on description and instructions based on feedback
