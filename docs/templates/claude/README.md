# CLAUDE.md Templates

This directory contains context-specific CLAUDE.md templates for different development scenarios in the Helix framework.

## Purpose

These templates provide AI assistants (like Claude) with focused, context-specific guidance when working on different aspects of the Helix framework. Each template points to the comprehensive documentation in [docs/llms.txt](../../llms.txt) while highlighting the most relevant information for specific tasks.

## Available Templates

### 1. CLAUDE-PLUGIN.md
**Use for**: Plugin development projects

**Key Features**:
- Plugin structure and organization
- Hook system reference
- Command and item registration
- Network communication patterns
- Plugin-specific best practices

**When to use**:
- Creating new plugins
- Modifying existing plugins
- Understanding plugin architecture
- Implementing plugin-specific features

### 2. CLAUDE-SCHEMA.md
**Use for**: Schema (game mode) development

**Key Features**:
- Schema structure and setup
- Faction and class creation
- Schema-specific items and commands
- Character system integration
- Database configuration for schemas

**When to use**:
- Creating new game modes
- Developing HL2 RP, DarkRP-style, or custom schemas
- Implementing faction/class systems
- Schema customization

### 3. CLAUDE-UI.md
**Use for**: User interface development

**Key Features**:
- Derma component system
- Menu and HUD integration
- UI networking patterns
- Performance optimization
- Styling and theming

**When to use**:
- Creating custom UI panels
- Developing menu systems
- Implementing HUD elements
- Building inventory interfaces
- Character creation screens

### 4. CLAUDE-DATABASE.md
**Use for**: Database and data persistence work

**Key Features**:
- MySQL/SQLite operations
- Storage system usage
- Character data management
- Query optimization
- Caching strategies

**When to use**:
- Working with databases
- Implementing data persistence
- Creating custom tables
- Optimizing queries
- Data migration

## How to Use These Templates

### Option 1: Copy to Your Project Directory

Copy the relevant template to your plugin, schema, or project directory and rename it to `CLAUDE.md`:

```bash
# For plugin development
cp docs/templates/claude/CLAUDE-PLUGIN.md plugins/myplugin/CLAUDE.md

# For schema development
cp docs/templates/claude/CLAUDE-SCHEMA.md schema/CLAUDE.md

# For UI-focused work
cp docs/templates/claude/CLAUDE-UI.md myproject/CLAUDE.md
```

### Option 2: Reference in Your Instructions

When working with an AI assistant, reference the appropriate template:

```
"Please review docs/templates/claude/CLAUDE-PLUGIN.md for guidance on plugin development in this framework."
```

### Option 3: Include in Project Documentation

Add a reference to the appropriate template in your project's README or documentation:

```markdown
## Development Guide

For AI-assisted development, see [CLAUDE-PLUGIN.md](docs/templates/claude/CLAUDE-PLUGIN.md)
```

## Template Structure

Each template follows this structure:

1. **Documentation Knowledgebase** - Points to llms.txt as the primary reference
2. **Core Documentation** - Lists the most relevant documentation files
3. **Quick API Reference** - Commonly used APIs for the context
4. **Development Workflow** - Step-by-step guidance with checklists
5. **Common Patterns** - Code examples for typical scenarios
6. **Best Practices** - Context-specific best practices checklist
7. **Troubleshooting** - Common issues and solutions

## Customization

These templates are meant to be customized for your specific project:

1. **Add project-specific information** - Include your project's conventions
2. **Add custom sections** - Include project-specific requirements
3. **Update references** - Adjust file paths if your structure differs
4. **Add examples** - Include examples from your project

## Main CLAUDE.md

The root-level [CLAUDE.md](../../../CLAUDE.md) provides general framework guidance and should be the starting point for most projects. Use these templates for more focused, context-specific work.

## Relationship to llms.txt

All templates point to [docs/llms.txt](../../llms.txt) as the authoritative documentation index. The templates serve as:

- **Navigation aids** - Quick access to relevant sections
- **Context providers** - Focus on specific development areas
- **Workflow guides** - Step-by-step processes with checklists
- **Best practices** - Concentrated guidelines for specific contexts

## Creating Additional Templates

To create a new template for a specific context:

1. Copy an existing template as a starting point
2. Update the title and context description
3. List the most relevant documentation from llms.txt
4. Add context-specific API references
5. Include common patterns and examples
6. Add a workflow checklist
7. Include troubleshooting tips
8. Update the README (this file) with the new template

## Integration with Development Tools

These templates work well with:

- **Claude Code** - For interactive development assistance
- **GitHub Copilot** - As context for code completion
- **AI Chat Interfaces** - As system prompts or context
- **Documentation Systems** - As quick reference guides

## Feedback and Contributions

If you create useful templates for specific contexts or improve existing ones, consider contributing them back to the project.

---

**Quick Links**:
- [Main CLAUDE.md](../../../CLAUDE.md)
- [Complete Documentation Index](../../llms.txt)
- [Helix Documentation](https://docs.gethelix.co)
