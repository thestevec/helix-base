# Helix Framework - AI Assistant Guide

> This document provides guidance for AI assistants (like Claude) when working with the Helix framework codebase.

## Documentation Knowledgebase

The complete Helix framework documentation is available in **[docs/llms.txt](docs/llms.txt)**. This file contains a comprehensive index of all framework systems, APIs, and development guides organized for efficient LLM consumption.

**Important**: Always reference the llms.txt file first to understand available documentation before making assumptions about framework functionality.

## Quick Navigation by Task Type

### New to Helix?
Start with these core documents:
- [Getting Started](docs/manual/getting-started.md) - Installation, setup, and MySQL configuration
- [Architecture Overview](docs/architecture.md) - Framework structure and initialization flow
- [Best Practices](docs/best-practices.md) - Coding standards and development guidelines

### Plugin Development
Essential references:
- [Plugin System](docs/plugins/plugin-system.md) - Architecture and loading mechanism
- [Creating Plugins](docs/plugins/creating-plugins.md) - Step-by-step development guide
- [Plugin Structure](docs/plugins/plugin-structure.md) - Directory organization and file layout
- [Hook System](docs/plugins/hooks.md) - Available hooks and event handling
- [Plugin Best Practices](docs/plugins/best-practices.md) - Quality guidelines

### Character & Player Systems
Key documentation:
- [Character System](docs/systems/character.md) - Character creation and lifecycle
- [Inventory System](docs/systems/inventory.md) - Grid-based inventory management
- [Item System](docs/systems/items.md) - Item registration and functionality
- [Attributes](docs/systems/attributes.md) - Character progression and stats
- [Faction System](docs/systems/factions.md) - Team/faction management
- [Class System](docs/systems/classes.md) - Job/role assignment

### UI Development
Interface references:
- [Derma Overview](docs/ui/derma-overview.md) - UI component framework
- [Menu System](docs/ui/menus.md) - Main menu and navigation
- [Inventory UI](docs/ui/inventory.md) - Drag-and-drop interface
- [Character Creation](docs/ui/character-creation.md) - Character screens
- [HUD System](docs/ui/hud.md) - Status bars and displays

### Database & Persistence
Data handling:
- [Database System](docs/libraries/database.md) - MySQL/SQLite abstraction
- [Storage System](docs/libraries/storage.md) - Persistent data with JSON/YAML
- [Data System](docs/libraries/data.md) - Key-value storage
- [Database Operations Guide](docs/guides/database.md) - Practical examples

### Commands & Chat
Communication systems:
- [Command System](docs/systems/commands.md) - Command registration and parsing
- [Chat System](docs/systems/chat.md) - IC/OOC/LOOC modes
- [Creating Commands Guide](docs/guides/commands.md) - Tutorial

### Networking
Client-server communication:
- [Networking Library](docs/libraries/networking.md) - Network message handling
- [Networking Guide](docs/guides/networking.md) - Communication patterns

### Schema Development
Creating custom game modes:
- [Schema Structure](docs/schema/structure.md) - Organization and files
- [Schema Configuration](docs/schema/configuration.md) - Schema-specific settings
- [Creating Factions](docs/schema/factions.md) - Faction definitions
- [Creating Classes](docs/schema/classes.md) - Class configuration
- [Creating Items](docs/schema/items.md) - Schema-specific items

### Configuration
Server and client settings:
- [Server Configuration](docs/config/server.md) - Server-side settings
- [Character Settings](docs/config/characters.md) - Character options
- [Database Configuration](docs/config/database.md) - Database setup
- [Inventory Settings](docs/config/inventory.md) - Inventory configuration

## API Quick Reference

### Most Commonly Used Libraries
- **ix.util** - Helper functions and utilities ([docs](docs/libraries/util.md), [api](docs/api/util.md))
- **ix.char** - Character management ([api](docs/api/character.md))
- **ix.inventory** - Inventory operations ([api](docs/api/inventory.md))
- **ix.item** - Item registration ([api](docs/api/item.md))
- **ix.command** - Command creation ([api](docs/api/command.md))
- **ix.config** - Configuration variables ([api](docs/api/config.md))
- **ix.faction** - Faction management ([api](docs/api/faction.md))
- **ix.db** - Database queries ([api](docs/api/database.md))
- **ix.net** - Network messages ([api](docs/api/networking.md))

### Full API Reference
See [docs/llms.txt](docs/llms.txt) lines 69-101 for the complete library reference index.

## Development Workflow Best Practices

### Before Making Changes
1. **Read llms.txt** - Understand the documentation structure
2. **Review Architecture** - Check [architecture.md](docs/architecture.md) for system design
3. **Check Best Practices** - Follow [best-practices.md](docs/best-practices.md) guidelines
4. **Search for Examples** - Look in [docs/examples/](docs/examples/) for similar implementations

### When Creating New Features
1. **Plugin-based approach** - Prefer plugins over core modifications
2. **Hook utilization** - Use existing hooks before creating new ones
3. **Network efficiency** - Minimize network messages, batch when possible
4. **Database optimization** - Use prepared statements, avoid queries in loops
5. **UI responsiveness** - Use derma properly, avoid blocking operations

### Code Quality Standards
- Follow Lua best practices
- Use proper indentation (tabs)
- Add comments for complex logic
- Use ix.util helper functions
- Implement proper error handling
- Test on both client and server realms

## Troubleshooting Resources

When encountering issues:
1. [Common Issues](docs/troubleshooting/common-issues.md) - Frequently encountered problems
2. [Database Issues](docs/troubleshooting/database.md) - Connection and query problems
3. [Plugin Conflicts](docs/troubleshooting/conflicts.md) - Resolving conflicts
4. [Performance Issues](docs/troubleshooting/performance.md) - Optimization tips

## Converting from Other Frameworks

If migrating from Clockwork:
- [Converting from Clockwork](docs/manual/converting-from-clockwork.md) - Migration guide

## Community Resources

- Official Website: https://gethelix.co
- Documentation: https://docs.gethelix.co
- Plugin Center: https://plugins.gethelix.co
- Discord: https://discord.gg/2AutUcF
- GitHub: https://github.com/nebulouscloud/helix

## Framework Information

- **Framework**: Helix
- **Base**: NutScript 1.1
- **Platform**: Garry's Mod
- **Language**: Lua
- **Supported Databases**: MySQL, SQLite

## How to Use This Guide

1. **For general questions**: Reference the llms.txt overview first
2. **For specific systems**: Navigate directly to the relevant section above
3. **For code examples**: Check the [docs/examples/](docs/examples/) directory
4. **For API details**: Use the API Quick Reference section
5. **For troubleshooting**: Start with the Troubleshooting Resources section

---

**Note for AI Assistants**: This framework is complex with many interconnected systems. Always consult the documentation before suggesting changes. The llms.txt file provides the most comprehensive overview and should be your primary reference point.
