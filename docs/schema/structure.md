# Schema Structure

A schema is your gamemode - it derives from Helix and adds your unique content.

## ⚠️ Important: Follow Helix Schema Structure

**Always follow the standard schema structure**. Helix automatically loads files from specific directories.

## Directory Structure

```
myschema/
├── gamemode/
│   ├── init.lua              # Server entry (2 lines)
│   ├── cl_init.lua           # Client entry (1 line)
│   └── shared.lua            # Optional: gamemode info
├── schema/
│   ├── sh_schema.lua         # Schema definition (REQUIRED)
│   ├── sv_schema.lua         # Server schema code
│   ├── cl_schema.lua         # Client schema code
│   ├── items/                # Item definitions
│   │   ├── sh_medkit.lua
│   │   └── sh_weapon.lua
│   ├── factions/             # Faction definitions
│   │   ├── sh_citizen.lua
│   │   └── sh_police.lua
│   ├── classes/              # Class definitions
│   │   └── sh_officer.lua
│   ├── attributes/           # Attribute definitions
│   │   └── sh_strength.lua
│   ├── commands/             # Custom commands (optional)
│   │   └── sh_commands.lua
│   ├── derma/                # Custom UI (optional)
│   │   └── cl_menu.lua
│   └── languages/            # Localizations (optional)
│       └── sh_english.lua
├── plugins/                  # Schema-specific plugins
│   └── myplugin/
│       └── sh_plugin.lua
└── myschema.txt              # Gamemode info file
```

## Required Files

### gamemode/init.lua

```lua
AddCSLuaFile("cl_init.lua")
DeriveGamemode("helix")
```

### gamemode/cl_init.lua

```lua
DeriveGamemode("helix")
```

### schema/sh_schema.lua

```lua
Schema.name = "My Schema"
Schema.author = "Your Name"
Schema.description = "My awesome roleplay schema"

-- Optional: Include other schema files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")
```

### myschema.txt

```
"myschema"
{
    "base"        "helix"
    "title"       "My Schema"
    "author"      "Your Name"
    "menusystem"  "1"
}
```

## Auto-Loading Directories

**Framework automatically loads** from these folders:

### schema/items/

- Item definitions with `sh_` prefix
- Example: `sh_medkit.lua`, `sh_pistol.lua`

### schema/factions/

- Faction definitions with `sh_` prefix
- Example: `sh_citizen.lua`, `sh_police.lua`

### schema/classes/

- Class definitions with `sh_` prefix
- Example: `sh_officer.lua`, `sh_chief.lua`

### schema/attributes/

- Attribute definitions with `sh_` prefix
- Example: `sh_str.lua`, `sh_int.lua`

### schema/languages/

- Localization files with `sh_` prefix
- Example: `sh_english.lua`, `sh_spanish.lua`

## Schema Methods

Use PLUGIN-style methods in schema files:

```lua
-- schema/sh_schema.lua
function Schema:PlayerLoadedCharacter(client, character)
    client:ChatPrint("Welcome to " .. self.name)
end

function Schema:PlayerSpawn(player)
    player:SetRunSpeed(240)
end
```

## Schema vs Plugins

**Schema**: Core gamemode content
- Factions, items, classes specific to your setting
- Main gameplay hooks
- Schema-specific systems

**Plugins**: Reusable features
- Can work across multiple schemas
- Optional features
- Modular systems

## Starting New Schema

### Option 1: Use Skeleton (Recommended)

Download: https://github.com/nebulouscloud/helix-skeleton

```
1. Download skeleton
2. Rename folder to your schema name
3. Rename skeleton.txt to yourschema.txt
4. Edit sh_schema.lua metadata
5. Add your factions, items, etc.
```

### Option 2: From Scratch

```
1. Create folder structure
2. Create required files (init.lua, cl_init.lua, sh_schema.lua, schemaname.txt)
3. Set gamemode in server.cfg
4. Restart server
```

## Best Practices

### ✅ DO

- Follow standard directory structure
- Use `sh_` prefix for shared files
- Store content in appropriate folders
- Use schema methods for hooks
- Keep schema focused on your setting

### ❌ DON'T

- Don't modify Helix core files
- Don't create custom folder structures
- Don't bypass auto-loading
- Don't put everything in sh_schema.lua

## See Also

- [Getting Started](../manual/getting-started.md)
- [Quick Start Guide](../guides/quick-start.md)
- Skeleton Schema: https://github.com/nebulouscloud/helix-skeleton
