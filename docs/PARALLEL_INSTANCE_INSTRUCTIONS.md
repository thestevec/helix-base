# Instructions for Parallel Claude Code Instances

## Quick Start

You are helping to create comprehensive documentation for the Helix Garry's Mod framework. All documentation must reference actual Helix source code and emphasize using built-in functionality.

## Your Task Format

When you receive a task, it will look like:

```
Create [filename.md] documenting [system name] from Helix gamemode.

Priority: [HIGH/MEDIUM/LOW]
Source files: gamemode/core/libs/[filename].lua
Topics: [list of topics]
```

## Documentation Template

Use this exact structure for every file:

```markdown
# [System/Library Name]

> **Reference**: `gamemode/core/libs/sh_filename.lua`

Brief description (1-2 sentences) of what this system does.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's built-in [system name]** rather than creating custom [alternatives]. The framework provides:
- Automatic [feature 1]
- Built-in [feature 2]
- Integration with [feature 3]

## Core Concepts

### What is [System Name]?

Explain the purpose and role in the framework.

### Key Terms

Define important terms.

## Using [System Name]

### Main Function Name

**Reference**: `gamemode/core/libs/sh_file.lua:123`

```lua
-- Show actual function signature
ix.system.Function(arguments)

-- Complete working example
local result = ix.system.Function(arg1, arg2)
if result then
    -- Use result
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Show what NOT to do
local custom = CreateCustomSystem()  -- Don't bypass framework!
```

### Additional Functions

[Repeat for each major function]

## Complete Example

```lua
-- Full, working example from actual usage
function PLUGIN:SomeHook(client)
    local system = ix.system.Get()
    -- ... complete implementation
end
```

## Best Practices

### ✅ DO

- Use `ix.system.Function()` for operations
- Let framework handle persistence
- Check return values
- Validate input
- Use provided hooks

### ❌ DON'T

- Don't create custom implementations
- Don't bypass framework functions
- Don't access internal tables directly
- Don't forget realm checks
- Don't implement custom sync

## Common Patterns

### Pattern 1: [Common Use Case]

```lua
-- Show how to do common task correctly
```

### Pattern 2: [Another Use Case]

```lua
-- Another common pattern
```

## Common Issues

### Issue Name

**Cause**: Why it happens
**Fix**: How to resolve it

```lua
-- Code showing fix
```

## See Also

- [Related System 1](../path/to/doc.md)
- [Related System 2](../path/to/doc.md)
- Source: `gamemode/core/libs/sh_file.lua`
```

## Critical Rules

### 1. Always Read Source Code First

```bash
# Read the actual Helix file
Read file: /home/user/helix-base/gamemode/core/libs/[filename].lua

# Or search for the system
Glob pattern: gamemode/**/*[systemname]*.lua
```

### 2. Reference Actual Code

**DO:**
```markdown
**Reference**: `gamemode/core/libs/sh_inventory.lua:38`

The framework's `ix.inventory.Restore()` function automatically:
- Queries database for items
- Creates inventory instances
- Loads item data
```

**DON'T:**
```markdown
The inventory system loads items.
```

### 3. Show Built-in Usage

**DO:**
```markdown
## ⚠️ Important: Use ix.inventory Functions

**Always use `ix.inventory.Restore()`** to load inventories from database.
Don't write custom SQL queries. Framework handles:
- Database queries
- Item instantiation
- Position validation
- Network sync
```

**DON'T:**
```markdown
You can load inventories from the database.
```

### 4. Provide Complete Examples

**DO:**
```lua
-- Complete, working example
function PLUGIN:CreateStorageChest(position)
    local chest = ents.Create("prop_physics")
    chest:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
    chest:SetPos(position)
    chest:Spawn()

    ix.inventory.New(0, "storage", function(inventory)
        if not inventory then return end
        chest.ixInventoryID = inventory:GetID()
    end)

    return chest
end
```

**DON'T:**
```lua
-- Incomplete snippet
local chest = ents.Create("prop_physics")
-- ... etc
```

### 5. Show Anti-Patterns

**Always include a "⚠️ Do NOT" section:**

```markdown
**⚠️ Do NOT**:
```lua
-- WRONG: Don't create manual inventory tables
local inventory = {
    w = 6,
    h = 4,
    slots = {}
}
-- This won't save to database or sync to clients!
```
Use `ix.inventory.Create()` instead.
```

## Research Process

### Step 1: Find Source Files

```bash
# Search for the system
Glob: gamemode/core/libs/*[systemname]*.lua
Glob: gamemode/core/meta/*[systemname]*.lua
Glob: plugins/*/[systemname]*.lua
```

### Step 2: Read Source Code

```bash
# Read the main file
Read: gamemode/core/libs/sh_[systemname].lua

# Read first 200 lines to understand structure
Read with limit: 200
```

### Step 3: Find Examples

```bash
# Search for usage examples
Grep: "ix\.[systemname]\." output_mode: "content"
Grep: "PLUGIN.*[SystemName]" output_mode: "content"
```

### Step 4: Check Existing Docs

```bash
# See if there's existing documentation
Read: docs/hooks/[systemname].lua
Read: docs/manual/*.md
```

## Example Workflow

### Task: Create libraries/data.md

**Step 1: Research**
```
Read: /home/user/helix-base/gamemode/core/libs/sh_data.lua (if exists)
Glob: gamemode/**/*data*.lua
Grep: "ix\.data\." output_mode: "content" -n
```

**Step 2: Analyze**
- Identify main functions (ix.data.Set, ix.data.Get, etc.)
- Note parameters and return values
- Find usage examples in codebase
- Check for hooks or callbacks

**Step 3: Create Documentation**

```markdown
# Data System (ix.data)

> **Reference**: `gamemode/core/libs/sh_data.lua`

The data system provides key-value storage for persistent data.

## ⚠️ Important: Use ix.data Functions

**Always use ix.data.Set/Get** rather than creating custom storage systems.

[... continue with template ...]
```

**Step 4: Include Examples**

```lua
-- From actual codebase usage
local data = ix.data.Get("myKey", defaultValue)
```

**Step 5: Commit**

```
git add docs/libraries/data.md
git commit -m "Add data system documentation

- Reference: gamemode/core/libs/sh_data.lua
- Covers: Set/Get operations, persistence
- Emphasizes: Using built-in data storage"
```

## File Naming Conventions

- Systems: `docs/systems/[name].md`
- Libraries: `docs/libraries/[name].md`
- Plugins: `docs/plugins/[name].md`
- Guides: `docs/guides/[name].md`
- API: `docs/api/[name].md`
- Examples: `docs/examples/[name].md`

All lowercase, use hyphens for spaces:
- ✅ `character-creation.md`
- ❌ `Character_Creation.md`
- ❌ `characterCreation.md`

## Quality Checklist

Before committing, verify:

- [ ] Read actual Helix source code
- [ ] Included source file reference at top
- [ ] Has "⚠️ Important" section
- [ ] Shows complete working examples
- [ ] Includes ✅ DO and ❌ DON'T sections
- [ ] References actual functions/methods with paths
- [ ] Follows template structure
- [ ] No grammar/spelling errors
- [ ] Code examples are valid Lua
- [ ] Cross-references related docs
- [ ] Includes "See Also" section

## Parallel Work Coordination

### Claim Your Task

1. Check DOCUMENTATION_TASKS.md for available files
2. Post comment: "Working on [filename.md]"
3. Create the file
4. Commit and push
5. Mark task as complete

### Avoid Conflicts

- Don't work on files others are doing
- Commit frequently
- Pull before starting new file
- Use descriptive commit messages

## Common Mistakes to Avoid

### ❌ Mistake 1: Generic Descriptions

```markdown
The inventory system manages items.
```

### ✅ Correct:

```markdown
**Reference**: `gamemode/core/libs/sh_inventory.lua:38`

The inventory system provides grid-based item storage with:
- Automatic database persistence (line 38-131)
- Network synchronization (line 45-48)
- Item position validation (line 85-116)
```

### ❌ Mistake 2: Missing Examples

```markdown
Use ix.inventory.Add() to add items.
```

### ✅ Correct:

```markdown
```lua
-- Add item to inventory
local inventory = character:GetInventory()
inventory:Add("item_medkit", 1, nil, nil, function(item)
    if item then
        client:Notify("Added medkit!")
    else
        client:Notify("Inventory full!")
    end
end)
```
```

### ❌ Mistake 3: No Warnings

```markdown
You can create your own inventory system.
```

### ✅ Correct:

```markdown
## ⚠️ Important: Use ix.inventory Functions

**Always use Helix's inventory system** rather than creating custom storage.
Framework automatically handles:
- Database saving
- Network sync
- Position validation

**⚠️ Do NOT** create custom inventory tables or implement your own storage.
```

## Getting Help

### If Source File Doesn't Exist

```
The file gamemode/core/libs/sh_[name].lua doesn't exist.

Search alternatives:
Glob: gamemode/core/**/*[name]*.lua
Glob: gamemode/**/*[name]*.lua

If not found, check if it's:
- In a plugin: plugins/*/
- In meta: gamemode/core/meta/
- Removed/renamed in newer versions
```

### If System is Complex

```
This system has many functions and I'm not sure what to prioritize.

Focus on:
1. Most commonly used functions (search codebase)
2. Functions called by developers (not internal)
3. Public API functions (not private helpers)
4. Main use cases (creation, retrieval, modification)
```

### If Examples Needed

```
I need examples but can't find usage in codebase.

Search for:
Grep: "ix\.[system]\." output_mode: "content"
Grep: "[SystemName]" path: "plugins/" output_mode: "content"
Grep: "PLUGIN:[SystemName]" output_mode: "content"

Check:
- Plugin files for usage examples
- Schema files in helix-hl2rp
- Hook documentation in docs/hooks/
```

## Summary

**Your mission**: Document Helix systems by:
1. Reading actual source code
2. Showing built-in functionality
3. Providing complete examples
4. Warning against custom implementations
5. Following the template structure

**Remember**: The goal is to help developers use Helix correctly, not create workarounds.
