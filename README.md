# .agenthelpers

A collection of skills and agents to enhance user workflows.

## Structure

```
.agenthelpers/
├── agents/     # Agent definitions with specific capabilities
├── commands/   # Reusable command templates
└── skills/     # Specialized skill modules
```

## Components

### Agents

Agents are specialized assistants with defined tools and behaviors.

- **planner** - An expert software architect that clarifies requirements and generates prioritized todo lists. Uses the `question` tool to gather details before creating actionable plans.

### Commands

Pre-defined command templates for common operations.

- **git-commit** - Creates conventional commit messages, handles version bumping, and updates documentation.

### Skills

Specialized skill modules for enhancing agent capabilities.

- **teach** - A structured teaching framework that creates lessons, missions, learning records, glossaries, and curates resources for skill acquisition. Designed for interactive, stateful learning sessions.

## Usage

Place your custom agents, commands, and skills in their respective directories. Each component is defined as a markdown file with frontmatter configuration.

### Agent Format

```markdown
---
description: What this agent does
mode: primary
tools:
  write: true
  edit: true
  question: true
  bash: false
---

Agent instructions...
```

### Command Format

```markdown
---
description: What this command does
---

Command steps...
```
