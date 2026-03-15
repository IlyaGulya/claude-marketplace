---
name: skill-creator-plugin
description: >-
  Guide for packaging Claude Code skills as distributable plugins.
  Use when creating a plugin, packaging skills for sharing, setting up
  a plugin marketplace, or when the user says "make a plugin", "package
  my skills", "distribute skills", or "create a marketplace".
---

# Plugin Packaging Guide

A plugin bundles one or more skills into a distributable package. This guide covers how to structure, configure, and distribute plugins for Claude Code.

## Plugin Structure

```
your-plugin/
  .claude-plugin/
    plugin.json          # Required — plugin metadata
  skills/
    skill-one/
      SKILL.md           # Required per skill
      references/        # Optional
      scripts/           # Optional
    skill-two/
      SKILL.md
      PITFALLS.md        # Optional supporting file
    skill-three/
      SKILL.md
```

## plugin.json

The plugin metadata file lives at `.claude-plugin/plugin.json`:

```json
{
  "name": "your-plugin-name",
  "description": "What the plugin provides — list all skills and capabilities",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

**Fields:**
- `name`: Plugin identifier (kebab-case recommended, no spaces)
- `description`: Complete description of what the plugin provides. Mention all included skills and their topics.
- `version`: Semantic versioning (MAJOR.MINOR.PATCH)
- `author.name`: Creator name

## Skill Naming in Plugins

Plugin skills use a `plugin-name:skill-name` namespace. This means they cannot conflict with personal or project skills.

Convention for this marketplace: prefix skill names with the plugin name:

```
Plugin: xstate
Skills: xstate-machine-modeling, xstate-actions-and-effects, xstate-testing
```

This makes skills self-documenting and avoids collisions even without the namespace prefix.

## Marketplace Structure

If distributing multiple plugins via a marketplace:

```
your-marketplace/
  .claude-plugin/
    marketplace.json     # Marketplace metadata
  plugins/
    plugin-a/
      .claude-plugin/
        plugin.json
      skills/
        ...
    plugin-b/
      .claude-plugin/
        plugin.json
      skills/
        ...
```

### marketplace.json

```json
{
  "name": "your-marketplace-name",
  "owner": {
    "name": "Your Name"
  },
  "metadata": {
    "description": "Description of your marketplace"
  },
  "plugins": [
    {
      "name": "plugin-a",
      "source": "./plugins/plugin-a",
      "description": "What plugin-a provides"
    },
    {
      "name": "plugin-b",
      "source": "./plugins/plugin-b",
      "description": "What plugin-b provides"
    }
  ]
}
```

## Creating a New Plugin: Step by Step

### Step 1: Plan Your Skills

Identify 3-15 skills that form a coherent set around one topic or technology.

Group by:
- **Technology** (e.g., all XState skills, all SolidJS skills)
- **Workflow** (e.g., all deployment skills, all code review skills)
- **Domain** (e.g., all financial analysis skills, all DevOps skills)

### Step 2: Create Directory Structure

```bash
mkdir -p your-plugin/.claude-plugin
mkdir -p your-plugin/skills/skill-one
mkdir -p your-plugin/skills/skill-two
```

### Step 3: Write plugin.json

Include a comprehensive description listing all skills and topics covered.

### Step 4: Write Each Skill

For each skill:
1. Create the skill directory under `skills/`
2. Write `SKILL.md` with proper frontmatter
3. Add supporting files if needed (PITFALLS.md, references/, scripts/)

### Step 5: Validate

For each skill:
- Run through the skill creation checklist
- Test triggering with relevant queries
- Test functional correctness

For the plugin:
- Verify plugin.json is valid JSON
- Verify all skill folders have SKILL.md
- Verify no README.md files inside skill folders
- Verify all skill names match their folder names

### Step 6: Add to Marketplace (if applicable)

Update the marketplace's `marketplace.json` to include the new plugin:

```json
{
  "name": "your-plugin",
  "source": "./plugins/your-plugin",
  "description": "Concise description of what the plugin provides"
}
```

## Distribution Methods

### As a Plugin (Claude Code)

Users add your plugin by referencing it in their Claude Code configuration. Place the plugin in a git repository and share the URL.

### As Individual Skills (Claude.ai)

For Claude.ai users:
1. Zip individual skill folders
2. Upload via Settings > Capabilities > Skills

### Via Git Repository

Host on GitHub with:
- A repo-level README (for humans) — this is separate from skill folders
- Clear installation instructions
- Example usage with screenshots

### Organization-Wide (Managed Settings)

Admins can deploy skills workspace-wide through managed settings for centralized management and automatic updates.

## Best Practices

**Cohesion:** All skills in a plugin should relate to the same topic. Don't mix unrelated skills in one plugin.

**Completeness:** Cover the full topic. A TypeScript plugin should cover types, generics, utility types, configuration, etc. — not just one aspect.

**Consistency:** Use the same style across all skills:
- Same frontmatter format
- Same section structure
- Same code example style
- Same naming convention for folders

**Progressive disclosure:** Keep each SKILL.md focused. Use supporting files for detailed references that aren't needed on every invocation.

**Independence:** Each skill should work on its own. Don't require users to read skill A before skill B can be useful.

**Versioning:** Update `version` in plugin.json when making changes. Follow semantic versioning:
- PATCH (0.0.x): fixes and minor improvements
- MINOR (0.x.0): new skills added
- MAJOR (x.0.0): breaking changes to existing skills
