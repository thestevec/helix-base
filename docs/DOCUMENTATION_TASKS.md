# Helix Documentation - Remaining Tasks

This document lists all remaining documentation files that need to be created from llms.txt, organized by priority and category.

## Instructions for Parallel Claude Code Instances

### Core Principles

**CRITICAL: Always follow these guidelines when creating documentation:**

1. **Reference Actual Helix Code**
   - Read the relevant source file from `gamemode/core/libs/` or `gamemode/core/meta/`
   - Include file path references (e.g., `> **Reference**: gamemode/core/libs/sh_util.lua`)
   - Cite specific line numbers when relevant

2. **Emphasize Built-in Functionality**
   - Start each doc with "⚠️ Important: Use Built-in Helix Functions" section
   - Explain what the framework provides automatically
   - Warn against creating custom implementations

3. **Show Correct Usage**
   - Provide complete, working examples from actual codebase
   - Include both ✅ DO and ❌ DON'T sections
   - Show common patterns and anti-patterns

4. **Structure Each File**
   ```markdown
   # System/Library Name

   > **Reference**: `gamemode/core/libs/sh_filename.lua`

   Brief description of what it does.

   ## ⚠️ Important: Use Built-in Helix Functions

   **Always use Helix's built-in X** rather than creating Y. Framework provides:
   - Feature 1
   - Feature 2

   ## Core Concepts

   ### What is X?

   ## Using the System

   ### Function Name

   **Reference**: `gamemode/core/libs/sh_file.lua:123`

   ```lua
   -- Code example
   ```

   **⚠️ Do NOT**:
   ```lua
   -- Wrong way
   ```

   ## Best Practices

   ### ✅ DO
   ### ❌ DON'T

   ## See Also
   ```

### Task Assignment Template

When starting a documentation task:

```
I need to create [filename.md] for the Helix documentation.

This file should document [system/library/feature name] from the Helix gamemode.

Requirements:
1. Read the actual source code from gamemode/core/libs/[filename].lua (or appropriate location)
2. Include explicit references to source files with paths
3. Start with "⚠️ Important: Use Built-in Helix Functions" section
4. Emphasize using Helix's built-in systems rather than creating custom implementations
5. Show complete, working examples from the actual codebase
6. Include ✅ DO and ❌ DON'T sections
7. Reference specific line numbers where relevant
8. Keep tone concise and practical

The documentation should help developers understand:
- What the system does
- How to use it correctly
- What NOT to do
- Common patterns
- Integration with other systems

Use the same style and format as existing files in docs/systems/ and docs/libraries/.
```

---

## Priority 1: Core Libraries (HIGH PRIORITY)

These are essential for developers to understand framework internals.

### libraries/util.md
**Source**: `gamemode/core/libs/sh_util.lua` (check actual location)
**Topics**:
- ix.util.Include() - File inclusion system
- ix.util.IncludeDir() - Directory loading
- Type checking functions
- Color manipulation
- String utilities
- Table utilities

### libraries/data.md
**Source**: `gamemode/core/libs/sh_data.lua` (check actual location)
**Topics**:
- ix.data.Set() - Key-value storage
- ix.data.Get() - Data retrieval
- Data persistence
- Network synchronization

### libraries/database.md
**Source**: `gamemode/core/libs/sv_database.lua`
**Topics**:
- ix.db.Query() - Query execution
- Prepared statements
- Database connection
- MySQL vs SQLite
- Query builder

### libraries/networking.md
**Source**: `gamemode/core/libs/sv_networking.lua`, `gamemode/core/libs/cl_networking.lua`
**Topics**:
- ix.net functions
- Network message registration
- Client-server communication
- Data serialization

---

## Priority 2: Gameplay Systems (HIGH PRIORITY)

Complete the remaining gameplay systems.

### systems/options.md
**Source**: `gamemode/core/libs/sh_option.lua`
**Topics**:
- ix.option.Add() - Client preference registration
- ix.option.Get() - Getting user preferences
- Option types
- Option categories

### systems/currency.md
**Source**: `gamemode/core/libs/sh_currency.lua`
**Topics**:
- ix.currency functions
- Money display formatting
- Currency symbol customization

### systems/business.md
**Source**: `gamemode/core/libs/sh_business.lua`
**Topics**:
- ix.business system
- Property ownership
- Business transactions

---

## Priority 3: Plugin Documentation (MEDIUM PRIORITY)

Complete the plugin development documentation.

### plugins/creating-plugins.md
**Step-by-step tutorial** for creating first plugin with complete example

### plugins/plugin-structure.md
**Detailed breakdown** of plugin directory structure and auto-loading

### plugins/hooks.md
**Reference**: `docs/hooks/plugin.lua` (existing hook documentation)
**Complete list** of available hooks with examples

### plugins/best-practices.md
**Guidelines** for quality plugin development

---

## Priority 4: Advanced Topics (MEDIUM PRIORITY)

### advanced/animations.md
**Source**: `gamemode/core/libs/sh_anims.lua`, `gamemode/core/libs/sh_animation.lua`

### advanced/logging.md
**Source**: `gamemode/core/libs/sh_log.lua`

### advanced/datetime.md
**Source**: `gamemode/core/libs/sh_date.lua`

### advanced/localization.md
**Source**: `gamemode/core/libs/sh_language.lua`

### advanced/meta-tables.md
**Source**: `gamemode/core/meta/sh_*.lua` files

### advanced/type-system.md
**Source**: `gamemode/core/libs/sh_command.lua` (type definitions)

### advanced/cami.md
**Source**: `gamemode/core/libs/thirdparty/sh_cami.lua`

---

## Priority 5: User Interface (MEDIUM PRIORITY)

### ui/derma-overview.md
**Source**: `gamemode/core/derma/` directory overview

### ui/hud.md
**Source**: `gamemode/core/libs/cl_hud.lua`, `gamemode/core/libs/cl_bar.lua`

### ui/menus.md
**Source**: `gamemode/core/libs/sh_menu.lua`

### ui/markup.md
**Source**: `gamemode/core/libs/cl_markup.lua`

### ui/inventory.md
**Source**: `gamemode/core/derma/cl_inventory.lua`

### ui/character-creation.md
**Source**: `gamemode/core/derma/cl_charcreate.lua`, `cl_charload.lua`

### ui/notifications.md
**Source**: `gamemode/core/libs/sh_notice.lua`, `gamemode/core/derma/cl_notice.lua`

---

## Priority 6: Schema Development (MEDIUM PRIORITY)

### schema/configuration.md
Schema-specific config setup

### schema/factions.md
Creating factions for schema (expand on systems/factions.md)

### schema/classes.md
Creating classes for schema (expand on systems/classes.md)

### schema/items.md
Creating schema items (expand on systems/items.md)

### schema/commands.md
Adding custom commands to schema

### schema/hooks.md
Implementing schema hooks

### schema/derma.md
Creating schema-specific UI

---

## Priority 7: Development Guides (LOW-MEDIUM PRIORITY)

### guides/setup.md
Development environment setup, tools, IDE configuration

### guides/first-plugin.md
Complete tutorial for beginners

### guides/custom-items.md
Walkthrough for creating custom items

### guides/commands.md
Tutorial for command system

### guides/inventories.md
Working with inventories programmatically

### guides/database.md
Database operations guide

### guides/networking.md
Client-server communication guide

### guides/ui-development.md
Creating Derma interfaces

### guides/hooks.md
Understanding and using hooks

### guides/debugging.md
Debugging techniques

---

## Priority 8: Item Types (LOW PRIORITY)

### items/base-items.md
**Source**: `gamemode/items/base/` directory

### items/weapons.md
**Source**: `gamemode/items/base/sh_weapons.lua`

### items/ammunition.md
**Source**: `gamemode/items/base/sh_ammo.lua`

### items/bags.md
**Source**: `gamemode/items/base/sh_bags.lua`

### items/outfits.md
**Source**: `gamemode/items/base/sh_outfit.lua`

### items/pac-outfits.md
**Source**: `gamemode/items/base/sh_pacoutfit.lua`

### items/custom-items.md
Guide for creating new item types

---

## Priority 9: Entities (LOW PRIORITY)

### entities/ix_item.md
**Source**: `entities/entities/ix_item.lua`

### entities/ix_money.md
**Source**: `entities/entities/ix_money.lua`

### entities/ix_shipment.md
**Source**: `entities/entities/ix_shipment.lua`

### entities/ix_vendor.md
**Source**: `plugins/vendor/entities/ix_vendor.lua`

### entities/custom-entities.md
Guide for creating custom entities

---

## Priority 10: Base Plugins (LOW PRIORITY)

Each plugin needs documentation covering:
- What it does
- How to configure it
- How to extend it
- Example usage

**List of 25 plugins:**
- plugins/3dpanel.md
- plugins/3dtext.md
- plugins/ammosave.md
- plugins/area.md
- plugins/business.md
- plugins/chatbox.md
- plugins/combining.md
- plugins/containers.md
- plugins/crosshair.md
- plugins/doors.md
- plugins/logging.md
- plugins/mapscene.md
- plugins/observer.md
- plugins/pac.md
- plugins/permakill.md
- plugins/persistence.md
- plugins/propprotect.md
- plugins/recognition.md
- plugins/saveitems.md
- plugins/spawns.md
- plugins/spawnsaver.md
- plugins/stamina.md
- plugins/strength.md
- plugins/thirdperson.md
- plugins/typing.md
- plugins/vendor.md
- plugins/wepselect.md

**Source**: `plugins/[pluginname]/` directories

---

## Priority 11: API Reference (LOW PRIORITY)

These are detailed API documentation files. Can be partially auto-generated from source comments.

**20 API files needed:**
- api/attribs.md
- api/character.md
- api/chat.md
- api/class.md
- api/command.md
- api/config.md
- api/currency.md
- api/data.md
- api/date.md
- api/database.md
- api/faction.md
- api/flag.md
- api/inventory.md
- api/item.md
- api/language.md
- api/log.md
- api/menu.md
- api/networking.md
- api/notice.md
- api/option.md
- api/plugin.md
- api/storage.md
- api/util.md

---

## Priority 12: Configuration Reference (LOW PRIORITY)

### config/server.md
Server-wide configuration options

### config/characters.md
Character-related config

### config/chat.md
Chat system config

### config/appearance.md
Visual customization config

### config/inventory.md
Inventory config

### config/database.md
Database configuration (expand on manual/getting-started.md)

---

## Priority 13: Examples (LOW PRIORITY)

Complete, working examples:

- examples/plugin.md
- examples/item.md
- examples/command.md
- examples/faction.md
- examples/class.md
- examples/derma.md
- examples/hooks.md

---

## Priority 14: Troubleshooting (LOW PRIORITY)

### troubleshooting/common-issues.md
FAQ and common problems

### troubleshooting/database.md
Database connection/query issues

### troubleshooting/conflicts.md
Plugin conflict resolution

### troubleshooting/performance.md
Optimization and profiling

---

## Summary Statistics

**Total files needed**: ~130+

**Breakdown by priority:**
- Priority 1 (Core Libraries): 4 files
- Priority 2 (Gameplay Systems): 3 files
- Priority 3 (Plugin Docs): 4 files
- Priority 4 (Advanced): 7 files
- Priority 5 (UI): 7 files
- Priority 6 (Schema): 7 files
- Priority 7 (Guides): 10 files
- Priority 8 (Item Types): 7 files
- Priority 9 (Entities): 5 files
- Priority 10 (Base Plugins): 27 files
- Priority 11 (API): 23 files
- Priority 12 (Config): 6 files
- Priority 13 (Examples): 7 files
- Priority 14 (Troubleshooting): 4 files

**Already completed**: 18 files

**Remaining**: ~112 files

---

## Parallel Instance Workflow

### Step 1: Claim a Task
Comment which file(s) you're working on to avoid duplication.

### Step 2: Research
Read the actual Helix source code for that system/library.

### Step 3: Create Documentation
Follow the structure template above.

### Step 4: Commit
Commit with descriptive message following the pattern:
```
Add [system/library] documentation

- Reference: gamemode/core/libs/[file]
- Covers: [topics]
- Emphasizes: Using built-in Helix functionality
```

### Step 5: Push
Push to the branch: `claude/helix-gamemode-documentation-011CUqVGEmneQQTp32QxpVwh`

---

## Quality Checklist

Before marking a file as complete, verify:

- [ ] Includes source file reference at top
- [ ] Has "⚠️ Important" section about using built-in functions
- [ ] Shows complete, working code examples
- [ ] Includes ✅ DO and ❌ DON'T sections
- [ ] References actual Helix functions/methods
- [ ] Follows existing documentation style
- [ ] No grammatical errors
- [ ] Code examples are tested/verified
- [ ] Cross-references related documentation
- [ ] Includes "See Also" section

---

## Recommended Priority Order for Parallel Work

**Group 1 (Immediate need):**
1. libraries/util.md
2. libraries/data.md
3. libraries/database.md
4. systems/options.md

**Group 2 (High value):**
5. plugins/creating-plugins.md
6. plugins/hooks.md
7. guides/first-plugin.md
8. advanced/meta-tables.md

**Group 3 (Complete subsystems):**
9. ui/derma-overview.md
10. ui/hud.md
11. advanced/animations.md
12. advanced/logging.md

**Group 4 (Schema developers):**
13-19. All schema/* files

**Group 5 (Item developers):**
20-26. All items/* files

**Group 6 (Plugin documentation):**
27-53. All plugins/* files (can be done in parallel)

**Group 7 (API Reference):**
54-76. All api/* files (can be largely templated)

**Group 8 (Guides):**
77-86. All guides/* files

**Group 9 (Polish):**
87-93. Examples
94-97. Troubleshooting
98-103. Config reference
