# Plugin Directory Structure

> **Reference**: `gamemode/core/libs/sh_plugin.lua:41-51`

This document details how plugin directories are organized and how Helix automatically loads different types of files.

## ⚠️ Important: Use Standard Structure

**Always follow Helix's plugin directory conventions** rather than creating custom loading systems. The framework provides:
- Automatic directory scanning and file inclusion
- Proper realm (client/server) handling based on file prefixes
- Correct loading order for dependencies
- Integration with the item, faction, class, and attribute systems

## Basic Plugin Structure

The **minimum required structure** for a plugin:

```
plugins/myplugin/
└── sh_plugin.lua          # Main plugin file (REQUIRED)
```

A **simple plugin** with organization:

```
plugins/myplugin/
├── sh_plugin.lua          # Main plugin file (required)
├── sh_config.lua          # Configuration options
├── sh_commands.lua        # Command definitions
├── sv_plugin.lua          # Server-only code
└── cl_plugin.lua          # Client-only code
```

## Complete Plugin Structure

A **full-featured plugin** utilizing all auto-loading directories:

```
plugins/myplugin/
├── sh_plugin.lua          # Main plugin file (REQUIRED)
│
├── sh_config.lua          # Configuration options
├── sh_commands.lua        # Command definitions
│
├── sv_plugin.lua          # Server-only logic
├── sv_hooks.lua           # Server-only hooks
├── sv_networking.lua      # Server networking
│
├── cl_plugin.lua          # Client-only logic
├── cl_hooks.lua           # Client-only hooks
├── cl_networking.lua      # Client networking
│
├── languages/             # Localization files (auto-loaded)
│   ├── sh_english.lua
│   ├── sh_spanish.lua
│   └── sh_french.lua
│
├── libs/                  # Custom libraries (auto-loaded)
│   ├── sh_mylib.lua
│   └── sv_database.lua
│
├── items/                 # Item definitions (auto-loaded)
│   ├── sh_custom_item.lua
│   └── sh_weapon.lua
│
├── factions/              # Faction definitions (auto-loaded)
│   └── sh_myfaction.lua
│
├── classes/               # Class definitions (auto-loaded)
│   └── sh_myclass.lua
│
├── attributes/            # Attribute definitions (auto-loaded)
│   └── sh_strength.lua
│
├── derma/                 # UI components (auto-loaded, client)
│   ├── cl_menu.lua
│   ├── cl_panel.lua
│   └── cl_frame.lua
│
├── entities/              # Custom entities (auto-loaded)
│   ├── entities/
│   │   ├── ent_custom/
│   │   │   ├── init.lua         # Server
│   │   │   ├── cl_init.lua      # Client
│   │   │   └── shared.lua       # Shared
│   │   └── ent_simple.lua       # Single-file entity
│   │
│   ├── weapons/
│   │   ├── weapon_custom/
│   │   │   ├── init.lua
│   │   │   ├── cl_init.lua
│   │   │   └── shared.lua
│   │   └── weapon_simple.lua
│   │
│   ├── effects/
│   │   └── my_effect.lua        # Client-only
│   │
│   └── tools/
│       └── sh_mytool.lua
│
└── plugins/               # Nested sub-plugins (auto-loaded)
    └── subplugin/
        └── sh_plugin.lua
```

## File Naming Conventions

### Realm Prefixes

**Reference**: `gamemode/core/libs/sh_util.lua` (ix.util.Include)

Files use prefixes to determine where they run:

| Prefix | Realm | Description |
|--------|-------|-------------|
| `sh_` | Shared | Runs on both client and server |
| `sv_` | Server | Runs only on server |
| `cl_` | Client | Runs only on client |

**Examples:**
- `sh_plugin.lua` - Runs on both realms
- `sv_hooks.lua` - Server-only hooks
- `cl_menu.lua` - Client-only UI code

### ⚠️ Important Naming Rules

**DO:**
```
sh_plugin.lua          ✓ Lowercase, realm prefix
sv_database.lua        ✓ Descriptive name
cl_notifications.lua   ✓ Clear purpose
```

**DON'T:**
```
Plugin.lua             ✗ No realm prefix
SV_Database.lua        ✗ Uppercase
client-menu.lua        ✗ Wrong separator
MyStuff.lua            ✗ Not descriptive
```

## Automatic Directory Loading

**Reference**: `gamemode/core/libs/sh_plugin.lua:41-51`

When your plugin loads, Helix automatically scans and loads these directories **in this specific order**:

### 1. Languages (`languages/`)

**Loaded first** - All language files are loaded before anything else.

```
languages/
├── sh_english.lua         # English translations
├── sh_spanish.lua         # Spanish translations
└── sh_french.lua          # French translations
```

**Reference**: `gamemode/core/libs/sh_plugin.lua:42`

Files are loaded via `ix.lang.LoadFromDir(path.."/languages")`.

### 2. Libraries (`libs/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:43`

Custom utility libraries loaded via `ix.util.IncludeDir(path.."/libs", true)`.

```
libs/
├── sh_utility.lua         # Shared helper functions
├── sv_database.lua        # Server database functions
└── cl_helpers.lua         # Client helpers
```

These are loaded **after languages** but **before** game content.

### 3. Attributes (`attributes/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:44`

Attribute definitions loaded via `ix.attributes.LoadFromDir(path.."/attributes")`.

```
attributes/
├── sh_strength.lua        # Strength attribute
└── sh_endurance.lua       # Endurance attribute
```

### 4. Factions (`factions/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:45`

Faction definitions loaded via `ix.faction.LoadFromDir(path.."/factions")`.

```
factions/
├── sh_rebels.lua          # Rebel faction
└── sh_combine.lua         # Combine faction
```

### 5. Classes (`classes/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:46`

Class definitions loaded via `ix.class.LoadFromDir(path.."/classes")`.

```
classes/
├── sh_medic.lua           # Medic class
└── sh_engineer.lua        # Engineer class
```

### 6. Items (`items/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:47`

Item definitions loaded via `ix.item.LoadFromDir(path.."/items")`.

```
items/
├── sh_health_kit.lua      # Health kit item
├── sh_weapon_pistol.lua   # Pistol item
└── sh_armor.lua           # Armor item
```

### 7. Nested Plugins (`plugins/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:48`

Sub-plugins loaded via `ix.plugin.LoadFromDir(path.."/plugins")`.

```
plugins/
└── subfeature/
    └── sh_plugin.lua      # Complete sub-plugin
```

### 8. Main Plugin File (`sh_plugin.lua`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:55`

Your main plugin file is loaded: `path.."/sh_plugin.lua"` (or `sv_plugin.lua`, `cl_plugin.lua`).

### 9. Derma UI (`derma/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:49`

Client-only UI components loaded via `ix.util.IncludeDir(path.."/derma", true)`.

```
derma/
├── cl_mainmenu.lua        # Main menu panel
├── cl_inventory.lua       # Custom inventory UI
└── cl_settings.lua        # Settings panel
```

### 10. Entities (`entities/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:50`, `sh_plugin.lua:108-244`

Last to load - custom entities, weapons, effects, and tools via `ix.plugin.LoadEntities(path.."/entities")`.

```
entities/
├── entities/
│   └── ent_custom.lua
├── weapons/
│   └── weapon_custom.lua
├── effects/
│   └── effect_custom.lua
└── tools/
    └── sh_tool.lua
```

## Loading Order Summary

**Reference**: `gamemode/core/libs/sh_plugin.lua:41-55`

```
1. languages/     → Translations loaded first
2. libs/          → Helper libraries
3. attributes/    → Attribute definitions
4. factions/      → Faction definitions
5. classes/       → Class definitions
6. items/         → Item definitions
7. plugins/       → Nested sub-plugins
8. sh_plugin.lua  → Main plugin file
9. derma/         → UI components (client)
10. entities/     → Entities, weapons, effects, tools
```

This order ensures dependencies are loaded before they're needed. For example:
- Libraries are available before plugin code runs
- Factions exist before classes (which may reference factions)
- Items can reference attributes in their definitions

## Manual File Loading

If you need custom files that **aren't** in auto-loaded directories, use `ix.util.Include()`:

**In `sh_plugin.lua`:**
```lua
PLUGIN.name = "My Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Description"

-- Load additional files manually
ix.util.Include("sh_config.lua")        # Loads config file
ix.util.Include("sh_commands.lua")      # Loads commands
ix.util.Include("sv_hooks.lua")         # Server hooks
ix.util.Include("cl_hooks.lua")         # Client hooks
ix.util.IncludeDir("libs")              # Load entire directory
```

**Reference**: `gamemode/core/libs/sh_util.lua` (ix.util.Include, ix.util.IncludeDir)

## Entity Directory Structure

**Reference**: `gamemode/core/libs/sh_plugin.lua:108-244`

Entities have special loading rules:

### Multi-File Entity

**Recommended for complex entities:**

```
entities/entities/ent_custom/
├── init.lua               # Server code (ENT table)
├── cl_init.lua            # Client code (ENT table)
└── shared.lua             # Shared code (ENT table)
```

**Loading order:**
1. Server loads `init.lua` and `shared.lua`
2. Server sends `cl_init.lua` to client
3. Client loads `cl_init.lua` and `shared.lua`

### Single-File Entity

**For simple entities:**

```
entities/entities/ent_simple.lua       # All code in one file
```

Use realm checks:
```lua
ENT.Type = "anim"
ENT.Base = "base_gmodentity"

if SERVER then
    function ENT:Initialize()
        -- Server code
    end
end

if CLIENT then
    function ENT:Draw()
        -- Client code
    end
end
```

### Weapons

**Same structure as entities:**

```
entities/weapons/weapon_custom/
├── init.lua
├── cl_init.lua
└── shared.lua
```

Or single-file:
```
entities/weapons/weapon_simple.lua
```

### Effects (Client-Only)

**Reference**: `gamemode/core/libs/sh_plugin.lua:238`

```
entities/effects/
└── my_particle_effect.lua     # Client-only, no prefix needed
```

### Tools

**Reference**: `gamemode/core/libs/sh_plugin.lua:227-235`

```
entities/tools/
└── sh_mytool.lua              # Shared prefix
```

## Real-World Examples

### Example 1: Simple Plugin (Ammo Saver)

**Reference**: `plugins/ammosave.lua`

```
plugins/ammosave.lua           # Single file plugin
```

Single-file plugins are perfect for simple functionality.

### Example 2: Medium Plugin (Doors)

**Reference**: `plugins/doors/`

```
plugins/doors/
├── sh_plugin.lua              # Main plugin
├── sh_commands.lua            # Door commands
├── sv_plugin.lua              # Server logic
├── cl_plugin.lua              # Client logic
└── derma/
    └── cl_door.lua            # Door UI
└── entities/
    └── weapons/
        └── ix_keys.lua        # Keys weapon
```

### Example 3: Complex Plugin (Vendor)

**Reference**: `plugins/vendor/`

```
plugins/vendor/
├── sh_plugin.lua              # Main plugin
├── entities/
│   └── entities/
│       └── ix_vendor.lua      # Vendor NPC entity
└── derma/
    ├── cl_vendor.lua          # Vendor interaction UI
    ├── cl_vendoreditor.lua    # Admin editor UI
    └── cl_vendorfaction.lua   # Faction vendor UI
```

### Example 4: Content Plugin (Stamina)

**Reference**: `plugins/stamina/`

```
plugins/stamina/
├── sh_plugin.lua              # Main plugin with hooks
└── attributes/
    ├── sh_stm.lua             # Stamina attribute
    └── sh_end.lua             # Endurance attribute
```

This plugin uses auto-loading for attributes - no need to manually include them!

## Best Practices

### ✅ DO

- **Use auto-loading directories**: Place files in `items/`, `factions/`, etc.
- **Follow naming conventions**: Use `sh_`, `sv_`, `cl_` prefixes
- **Organize by purpose**: Separate hooks, commands, config into different files
- **Keep related code together**: Group UI files in `derma/`, items in `items/`
- **Use single file for simple plugins**: Don't over-engineer
- **Document your structure**: Add comments explaining organization

### ❌ DON'T

**Don't create deeply nested structures:**
```
plugins/myplugin/
└── code/
    └── server/
        └── hooks/
            └── player/
                └── sv_spawn.lua    # ✗ Too deep
```

**Don't mix realms incorrectly:**
```
plugins/myplugin/
├── server_and_client.lua           # ✗ Ambiguous
└── everything.lua                  # ✗ Not descriptive
```

**Don't manually load auto-loaded directories:**
```lua
-- ✗ WRONG - items/ auto-loads
ix.util.IncludeDir("items")

-- ✗ WRONG - factions/ auto-loads
ix.util.IncludeDir("factions")
```

**Don't put files in wrong directories:**
```
plugins/myplugin/
├── items/
│   └── cl_menu.lua                 # ✗ UI in items folder
└── derma/
    └── sh_weapon.lua               # ✗ Item in derma folder
```

## Common Patterns

### Pattern 1: Organized Medium Plugin

```
plugins/roleplay/
├── sh_plugin.lua          # Metadata + includes
├── sh_config.lua          # Config options
├── sh_commands.lua        # Commands
├── sv_hooks.lua           # Server hooks
├── cl_hooks.lua           # Client hooks
└── derma/
    └── cl_menu.lua        # UI
```

### Pattern 2: Content-Heavy Plugin

```
plugins/customcontent/
├── sh_plugin.lua          # Minimal metadata
├── items/                 # Auto-loaded
│   ├── sh_item1.lua
│   ├── sh_item2.lua
│   └── sh_item3.lua
├── factions/              # Auto-loaded
│   └── sh_faction1.lua
└── classes/               # Auto-loaded
    ├── sh_class1.lua
    └── sh_class2.lua
```

No manual includes needed - all auto-loaded!

### Pattern 3: UI-Focused Plugin

```
plugins/customui/
├── sh_plugin.lua          # Plugin metadata
├── cl_plugin.lua          # Client initialization
└── derma/                 # Auto-loaded UI
    ├── cl_mainmenu.lua
    ├── cl_inventory.lua
    ├── cl_character.lua
    └── cl_settings.lua
```

## Troubleshooting

### Files Not Loading?

**Check these common issues:**

1. **Wrong directory**: Is the file in an auto-loaded folder?
2. **Wrong prefix**: Does the filename have `sh_`, `sv_`, or `cl_`?
3. **Syntax error**: Check console for Lua errors
4. **Load order**: Does this file depend on something not yet loaded?

### Manual Include Not Working?

```lua
-- ✗ Wrong - absolute path
ix.util.Include("/home/gmod/garrysmod/gamemodes/...")

-- ✗ Wrong - wrong directory
ix.util.Include("plugins/myplugin/sh_config.lua")

-- ✓ Right - relative to plugin root
ix.util.Include("sh_config.lua")

-- ✓ Right - subdirectory
ix.util.Include("config/sh_settings.lua")
```

### Entity Not Spawnable?

**Reference**: `gamemode/core/libs/sh_plugin.lua:210-218`

```lua
-- In entity file
ENT.Spawnable = true               # Allow spawning in sandbox
ENT.AdminOnly = true               # Admin-only spawn
```

## See Also

- [Creating Plugins](creating-plugins.md) - Step-by-step tutorial
- [Plugin System](plugin-system.md) - How the plugin system works
- [Hooks](hooks.md) - Available hooks
- [Best Practices](best-practices.md) - Plugin development guidelines
- Source: `gamemode/core/libs/sh_plugin.lua`
