# Schema Configuration

> **Reference**: `gamemode/config/sh_config.lua`, `schema/sh_schema.lua`

This guide shows you how to configure your schema's settings, define custom config options, and set schema metadata.

## ⚠️ Important: Use Built-in Helix Config System

**Always use `ix.config.Add()` and `ix.option.Add()`** for schema settings. The framework provides:
- Automatic database persistence
- Admin UI for changing values at runtime
- Network synchronization between server and clients
- Type validation
- Change callbacks

## Core Concepts

### Schema Metadata

Every schema must define basic information in `schema/sh_schema.lua`:

```lua
Schema.name = "My Schema Name"
Schema.author = "Your Name"
Schema.description = "A description of your roleplay setting"
Schema.version = "1.0.0"

-- Optional: Load additional schema files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")
```

### Configuration Types

Helix provides two configuration systems:

1. **ix.config** - Server-wide settings (saved in database, admin-configurable)
2. **ix.option** - Per-player client preferences (saved locally on each client)

## Schema Metadata Configuration

### Basic Schema Information

**Reference**: `schema/sh_schema.lua`

```lua
-- File: schema/sh_schema.lua
Schema.name = "Half-Life 2 Roleplay"
Schema.author = "Community"
Schema.description = "Roleplay in the Half-Life 2 universe"

-- Optional version tracking
Schema.version = "2.1.4"

-- Optional: Include other schema files
ix.util.Include("sv_schema.lua")  -- Server-only schema code
ix.util.Include("cl_schema.lua")  -- Client-only schema code
```

### Schema Information Usage

```lua
-- Access schema information anywhere
print(Schema.name)     -- "Half-Life 2 Roleplay"
print(Schema.author)   -- "Community"
print(Schema.folder)   -- "hl2rp" (automatically set to schema folder name)
```

## Server Configuration (ix.config)

### Adding Server Config Options

**Reference**: `gamemode/core/libs/sh_config.lua`

Add server-wide configuration in `schema/sh_schema.lua` or your plugin files:

```lua
-- File: schema/sh_schema.lua
ix.config.Add("startingMoney", 100, "Money given to new characters", nil, {
    data = {min = 0, max = 10000},
    category = "characters"
})

ix.config.Add("enableVoice", true, "Allow voice chat in game", function(oldValue, newValue)
    if newValue then
        print("Voice chat enabled")
    else
        print("Voice chat disabled")
    end
end, {
    category = "gameplay"
})

ix.config.Add("serverName", "My Server", "Name shown in scoreboard", nil, {
    category = "general"
})
```

### Getting Config Values

```lua
-- Get config value with optional default
local startMoney = ix.config.Get("startingMoney", 100)
local voiceEnabled = ix.config.Get("enableVoice", true)

-- Use in character creation
function Schema:OnCharacterCreated(client, character)
    local money = ix.config.Get("startingMoney", 100)
    character:SetMoney(money)
end
```

### Config with Callbacks

**Reference**: `gamemode/core/libs/sh_config.lua:44`

```lua
-- Execute code when config changes
ix.config.Add("maxSpeed", 250, "Maximum run speed", function(oldValue, newValue)
    -- Update all players when config changes
    for _, client in ipairs(player.GetAll()) do
        client:SetRunSpeed(newValue)
    end
end, {
    data = {min = 100, max = 500},
    category = "gameplay"
})
```

### Config Categories

Organize configs into categories:

```lua
-- Gameplay settings
ix.config.Add("respawnTime", 5, "Respawn delay in seconds", nil, {
    data = {min = 0, max = 60},
    category = "gameplay"
})

-- Economy settings
ix.config.Add("salaryAmount", 50, "Default salary payment", nil, {
    data = {min = 0, max = 1000},
    category = "economy"
})

-- Character settings
ix.config.Add("maxCharacters", 5, "Max characters per player", nil, {
    data = {min = 1, max = 10},
    category = "characters"
})
```

## Client Options (ix.option)

### Adding Client Options

**Reference**: `gamemode/core/libs/sh_option.lua:44`

Define client-side preferences in `schema/cl_schema.lua`:

```lua
-- File: schema/cl_schema.lua
ix.option.Add("showHints", ix.type.bool, true, {
    category = "display"
})

ix.option.Add("hudScale", ix.type.number, 1, {
    category = "display",
    min = 0.5,
    max = 2,
    decimals = 1
})

ix.option.Add("chatTimestamps", ix.type.bool, false, {
    category = "chat"
})
```

### Getting Client Options

```lua
-- CLIENT
local showHints = ix.option.Get("showHints", true)
local hudScale = ix.option.Get("hudScale", 1)

-- Use in HUD rendering
hook.Add("HUDPaint", "DrawCustomHUD", function()
    if not ix.option.Get("showHUD", true) then
        return
    end

    local scale = ix.option.Get("hudScale", 1)
    -- Draw HUD with scale
end)
```

### Networked Options

**Reference**: `gamemode/core/libs/sh_option.lua:82`

Make client options available on server:

```lua
-- File: schema/sh_schema.lua (SHARED)
ix.option.Add("language", ix.type.string, "english", {
    category = "general",
    bNetworked = true,  -- Send to server
    populate = function()
        return {
            ["english"] = "English",
            ["spanish"] = "Spanish",
            ["french"] = "French"
        }
    end
})

-- SERVER - Get client's option value
local lang = ix.option.Get(client, "language", "english")
client:ChatPrint("Your language: " .. lang)
```

### Option with Callback

**Reference**: `gamemode/core/libs/sh_option.lua:85`

```lua
-- CLIENT
ix.option.Add("fov", ix.type.number, 90, {
    category = "display",
    min = 75,
    max = 110,
    OnChanged = function(oldValue, newValue)
        -- Update FOV when changed
        LocalPlayer():SetFOV(newValue, 0.5)
    end
})
```

## Complete Configuration Example

### Full Schema Config

```lua
-- File: schema/sh_schema.lua
Schema.name = "Fallout Roleplay"
Schema.author = "Schema Team"
Schema.description = "Post-apocalyptic roleplay in the Fallout universe"
Schema.version = "3.0.1"

-- Server-wide settings
ix.config.Add("startingMoney", 500, "Caps given to new characters", nil, {
    data = {min = 0, max = 5000},
    category = "economy"
})

ix.config.Add("radStormEnabled", true, "Enable radiation storms", nil, {
    category = "gameplay"
})

ix.config.Add("radStormInterval", 1800, "Seconds between rad storms", nil, {
    data = {min = 300, max = 7200},
    category = "gameplay"
})

ix.config.Add("startingRadiation", 0, "Starting radiation level", nil, {
    data = {min = 0, max = 100},
    category = "gameplay"
})

-- Client options (define in cl_schema.lua or with CLIENT check)
if CLIENT then
    ix.option.Add("enablePipboy", ix.type.bool, true, {
        category = "schema"
    })

    ix.option.Add("hudStyle", ix.type.string, "classic", {
        category = "schema",
        populate = function()
            return {
                ["classic"] = "Classic Fallout",
                ["modern"] = "Modern UI",
                ["minimal"] = "Minimal"
            }
        end,
        OnChanged = function(oldValue, newValue)
            -- Rebuild HUD with new style
            hook.Run("RebuildHUD", newValue)
        end
    })
end

-- Load other schema files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")
```

## Using Config in Schema Hooks

### Character Creation

```lua
-- File: schema/sv_schema.lua
function Schema:OnCharacterCreated(client, character)
    -- Use config for starting money
    local startMoney = ix.config.Get("startingMoney", 500)
    character:SetMoney(startMoney)

    -- Use config for starting radiation
    local startRads = ix.config.Get("startingRadiation", 0)
    character:SetData("radiation", startRads)

    -- Give starting items based on config
    if ix.config.Get("giveStarterKit", true) then
        local inventory = character:GetInventory()
        inventory:Add("item_water")
        inventory:Add("item_food")
    end
end
```

### Player Spawn

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerSpawn(client)
    -- Use config for spawn settings
    local maxSpeed = ix.config.Get("maxSpeed", 250)
    client:SetRunSpeed(maxSpeed)

    local maxHealth = ix.config.Get("maxHealth", 100)
    client:SetMaxHealth(maxHealth)
end
```

### HUD Rendering

```lua
-- File: schema/cl_schema.lua
function Schema:HUDPaint()
    -- Check client option
    if not ix.option.Get("showCustomHUD", true) then
        return
    end

    local scale = ix.option.Get("hudScale", 1)
    local style = ix.option.Get("hudStyle", "classic")

    -- Render HUD based on options
    self:DrawHUD(scale, style)
end
```

## Config Data Types

### Number Config

```lua
ix.config.Add("respawnTime", 5, "Respawn delay", nil, {
    data = {
        min = 0,      -- Minimum value
        max = 60,     -- Maximum value
        decimals = 1  -- Allow 1 decimal place
    },
    category = "gameplay"
})
```

### Boolean Config

```lua
ix.config.Add("pvpEnabled", false, "Enable PvP combat", nil, {
    category = "gameplay"
})
```

### String Config

```lua
ix.config.Add("motd", "Welcome to the server!", "Message of the day", nil, {
    category = "general"
})
```

### Array/Choice Config

```lua
-- For options (client-side)
ix.option.Add("language", ix.type.string, "english", {
    category = "general",
    populate = function()
        return {
            ["english"] = "English",
            ["spanish"] = "Spanish"
        }
    end
})
```

## Best Practices

### ✅ DO

- Define all configs in `schema/sh_schema.lua`
- Use meaningful category names
- Set appropriate min/max ranges for numbers
- Provide clear descriptions
- Use callbacks for dynamic updates
- Use `ix.config` for server settings
- Use `ix.option` for client preferences
- Test config changes before deployment

### ❌ DON'T

- Don't create custom config files (use `ix.config`)
- Don't hardcode values that should be configurable
- Don't forget to set default values
- Don't use server configs for client-only settings
- Don't access config tables directly (use `ix.config.Get()`)
- Don't create configs without categories
- Don't forget realm checks (`if CLIENT then`)

## Common Patterns

### Economy Settings

```lua
ix.config.Add("startingMoney", 100, "Starting money", nil, {
    data = {min = 0, max = 10000},
    category = "economy"
})

ix.config.Add("salaryInterval", 300, "Salary payment interval", nil, {
    data = {min = 60, max = 3600},
    category = "economy"
})

ix.config.Add("priceMultiplier", 1, "Item price multiplier", nil, {
    data = {min = 0.1, max = 10, decimals = 1},
    category = "economy"
})
```

### Gameplay Toggles

```lua
ix.config.Add("permadeath", false, "Enable permanent death", nil, {
    category = "gameplay"
})

ix.config.Add("friendlyFire", false, "Enable team damage", nil, {
    category = "gameplay"
})

ix.config.Add("dropWeapons", true, "Drop weapons on death", nil, {
    category = "gameplay"
})
```

### Display Options

```lua
-- CLIENT
ix.option.Add("showHints", ix.type.bool, true, {
    category = "display"
})

ix.option.Add("showCrosshair", ix.type.bool, true, {
    category = "display"
})

ix.option.Add("screenEffects", ix.type.bool, true, {
    category = "display"
})
```

## Config Admin Panel

Admins can change config values in-game:

1. Press **F1** to open character menu
2. Navigate to **Config** tab
3. Find config by category
4. Adjust value (changes saved to database)

**⚠️ Note**: Only configs added with `ix.config.Add()` appear in the admin panel.

## See Also

- [Configuration System](../systems/configuration.md) - Core config system reference
- [Schema Structure](structure.md) - Schema directory layout
- Source: `gamemode/core/libs/sh_config.lua`
- Source: `gamemode/core/libs/sh_option.lua`
