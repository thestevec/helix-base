# Config API (ix.config)

> **Reference**: `gamemode/core/sh_config.lua`

The config API manages server configuration options that admins can adjust through the F1 menu. Config options automatically save to disk, sync across clients, and support callbacks when values change.

## ⚠️ Important: Use Built-in Helix Config System

**Always use Helix's built-in config system** rather than creating custom ConVars or config files. The framework automatically provides:
- Automatic disk persistence in `data/helix/`
- Network synchronization to all clients
- Admin panel GUI for editing (F1 menu)
- Type validation and conversion
- Change callbacks
- Schema-specific and global config options
- CAMI permission integration
- Default value fallbacks

## Core Concepts

### What is Config?

The config system stores server settings that can be changed at runtime:
- Options appear in the admin panel (F1 > Config)
- Values persist across server restarts
- Changes automatically sync to all clients
- Each option has a type, default, description, and optional callback
- Can be global (all schemas) or schema-specific

### Key Terms

**Config**: A registered setting (e.g., "chatRange", "maxMoney")
**Default**: The initial value if not modified
**Global**: Config shared across all schemas
**Schema-Only**: Config specific to current schema
**Callback**: Function called when config value changes

## Library Tables

### ix.config.stored

**Reference**: `gamemode/core/sh_config.lua:6`

**Realm**: Shared

Table of all registered config options with their metadata.

```lua
-- Access config metadata
local cfg = ix.config.stored["chatRange"]
print("Type:", cfg.type)
print("Default:", cfg.default)
print("Current:", cfg.value)

-- Iterate all configs
for key, config in pairs(ix.config.stored) do
    print(key, config.value, config.description)
end
```

## Library Functions

### ix.config.Add

**Reference**: `gamemode/core/sh_config.lua:32`

**Realm**: Shared

```lua
ix.config.Add(key, value, description, callback, data, bNoNetworking, bSchemaOnly)
```

Registers a new config option.

**Parameters**:
- `key` (string) - Unique config identifier
- `value` (any) - Default value
- `description` (string) - Description shown in config panel
- `callback` (function, optional) - Called when value changes: `callback(oldValue, newValue)`
- `data` (table, optional) - Additional settings (category, min/max, etc.)
- `bNoNetworking` (bool, optional) - Don't sync to clients
- `bSchemaOnly` (bool, optional) - Schema-specific config (default: false = global)

**Example - Simple Config**:
```lua
ix.config.Add("maxMoney", 10000, "Maximum money a character can have")
```

**Example - With Callback**:
```lua
ix.config.Add("chatRange", 280, "Range for IC chat in units", function(oldValue, newValue)
    print("Chat range changed from", oldValue, "to", newValue)

    -- Update chat classes if needed
    if ix.chat.classes["ic"] then
        ix.chat.classes["ic"].CanHear = newValue
    end
end)
```

**Example - With Category**:
```lua
ix.config.Add("walkSpeed", 120, "Walking speed multiplier", nil, {
    category = "performance",  -- Group in config panel
    data = {min = 50, max = 200}  -- Slider bounds
})
```

**Example - Schema-Specific**:
```lua
-- Only applies to current schema
ix.config.Add("factionLimit", 50, "Max members per faction", nil, nil, false, true)
```

**Example - Server-Only (No Networking)**:
```lua
ix.config.Add("databaseHost", "localhost", "Database hostname", nil, nil, true, false)
```

### ix.config.Get

**Reference**: `gamemode/core/sh_config.lua:133`

**Realm**: Shared

```lua
local value = ix.config.Get(key, default)
```

Retrieves the current value of a config option.

**Parameters**:
- `key` (string) - Config key
- `default` (any, optional) - Value to return if config doesn't exist

**Returns**: (any) Current config value or default

**Example**:
```lua
-- Get chat range
local range = ix.config.Get("chatRange", 280)

-- Use in code
if player:GetPos():Distance(speaker:GetPos()) <= ix.config.Get("chatRange") then
    player:ChatPrint(message)
end

-- With fallback
local maxMoney = ix.config.Get("maxMoney", 999999)
if character:GetMoney() > maxMoney then
    character:SetMoney(maxMoney)
end
```

### ix.config.Set

**Reference**: `gamemode/core/sh_config.lua:104`

**Realm**: Shared (but prefer server)

```lua
ix.config.Set(key, value)
```

Changes a config option's value and saves to disk.

**Parameters**:
- `key` (string) - Config key
- `value` (any) - New value

**Example**:
```lua
-- Change config value
ix.config.Set("chatRange", 500)

-- Admin command
ix.command.Add("SetChatRange", {
    adminOnly = true,
    arguments = ix.type.number,
    OnRun = function(self, client, range)
        ix.config.Set("chatRange", range)
        return "Chat range set to " .. range
    end
})
```

**Note**: Automatically triggers callback, saves to disk, and syncs to clients.

### ix.config.SetDefault

**Reference**: `gamemode/core/sh_config.lua:74`

**Realm**: Shared

```lua
ix.config.SetDefault(key, value)
```

Sets the default value for a config option. Used by schemas to override framework defaults.

**Parameters**:
- `key` (string) - Config key
- `value` (any) - New default value

**Example**:
```lua
-- Schema changing framework default
ix.config.SetDefault("maxAttributes", 100)  -- Framework default is usually 30
ix.config.SetDefault("chatRange", 500)      -- Override default chat range
```

**Note**: Must be called before the config is first accessed. Usually in schema initialization.

### ix.config.Load

**Reference**: `gamemode/core/sh_config.lua:151`

**Realm**: Shared (internal)

```lua
ix.config.Load()
```

Loads all saved config values from disk.

**Note**: Called automatically by framework at startup. Should not be called manually.

### ix.config.Save

**Reference**: `gamemode/core/sh_config.lua:200`

**Realm**: Server (internal)

```lua
ix.config.Save()
```

Saves all changed config values to disk.

**Note**: Called automatically when config changes. Should not be called manually.

## Config Data Options

The `data` parameter in `ix.config.Add` supports these options:

```lua
data = {
    category = "categoryName",  -- Group in config panel ("server", "chat", "performance", etc.)
    min = 0,                    -- Minimum value for numbers
    max = 100,                  -- Maximum value for numbers
    decimals = 2,               -- Decimal places for numbers
    hidden = true               -- Don't show in config panel
}
```

## Complete Examples

### Chat System Config

```lua
-- Register chat-related configs
ix.config.Add("chatRange", 280, "Range for IC chat in units", nil, {
    category = "chat",
    data = {min = 50, max = 1000}
})

ix.config.Add("chatAutoFormat", true, "Automatically format chat messages", nil, {
    category = "chat"
})

ix.config.Add("oocDelay", 10, "Delay between OOC messages in seconds", nil, {
    category = "chat",
    data = {min = 0, max = 60}
})
```

### Economy Config with Callback

```lua
ix.config.Add("maxMoney", 100000, "Maximum money a character can have", function(old, new)
    -- Enforce new limit on all characters
    for _, character in pairs(ix.char.loaded) do
        if character:GetMoney() > new then
            character:SetMoney(new)

            local client = character:GetPlayer()
            if IsValid(client) then
                client:Notify("Money capped at new limit: " .. ix.currency.Get(new))
            end
        end
    end
end, {
    category = "economy",
    data = {min = 1000, max = 1000000}
})

ix.config.Add("startMoney", 500, "Starting money for new characters", nil, {
    category = "economy",
    data = {min = 0, max = 10000}
})
```

### Performance Configs

```lua
ix.config.Add("saveInterval", 600, "Character save interval in seconds", function(old, new)
    -- Update timer
    timer.Remove("ixAutoSave")
    timer.Create("ixAutoSave", new, 0, function()
        hook.Run("SaveData")
    end)
end, {
    category = "performance",
    data = {min = 60, max = 3600}
})

ix.config.Add("maxSlots", 5, "Maximum character slots per player", nil, {
    category = "server",
    data = {min = 1, max = 20}
})
```

### Schema Defaults Override

```lua
-- In schema/sh_schema.lua
function Schema:InitializedConfig()
    -- Override framework defaults
    ix.config.SetDefault("chatRange", 500)
    ix.config.SetDefault("maxAttributes", 100)
    ix.config.SetDefault("sprintSpeed", 250)
end
```

### Dynamic Config Based on Map

```lua
-- Different chat ranges per map
hook.Add("InitializedConfig", "MapBasedConfig", function()
    local map = game.GetMap()

    if map == "rp_downtown_v4c_v2" then
        ix.config.SetDefault("chatRange", 400)  -- Larger map
    else
        ix.config.SetDefault("chatRange", 280)  -- Default
    end
end)
```

### Command to Modify Config

```lua
ix.command.Add("ConfigSet", {
    description = "Set a config value",
    superAdminOnly = true,
    arguments = {
        ix.type.string,  -- Config key
        ix.type.text     -- New value
    },
    OnRun = function(self, client, key, value)
        local config = ix.config.stored[key]

        if not config then
            return "Config '" .. key .. "' does not exist"
        end

        -- Convert value to correct type
        if config.type == "number" then
            value = tonumber(value)
            if not value then
                return "Value must be a number"
            end
        elseif config.type == "boolean" then
            value = tobool(value)
        end

        ix.config.Set(key, value)
        return "Set " .. key .. " to " .. tostring(value)
    end
})
```

## Best Practices

### ✅ DO

- Use descriptive config keys (e.g., "chatRange" not "cr")
- Provide clear descriptions for admin panel
- Set appropriate min/max values for numbers
- Use categories to organize related configs
- Use callbacks for configs that need immediate effect
- Make configs schema-specific if they don't apply globally
- Use `ix.config.Get` with fallback defaults
- Test config changes in-game before deploying

### ❌ DON'T

- Don't use ConVars - use ix.config instead
- Don't store sensitive data in configs (use `bNoNetworking`)
- Don't call `ix.config.Save()` manually - it's automatic
- Don't modify `ix.config.stored` directly
- Don't forget to check if config exists before using
- Don't use configs for frequently changing data
- Don't create configs without descriptions
- Don't bypass the config system with custom files

## Common Patterns

### Config with Validation

```lua
ix.config.Add("maxPlayers", 32, "Maximum players on server", function(old, new)
    -- Validate range
    if new < 1 then
        ix.config.Set("maxPlayers", 1)
        return
    end
    if new > 128 then
        ix.config.Set("maxPlayers", 128)
        return
    end

    -- Apply change
    RunConsoleCommand("maxplayers", tostring(new))
end)
```

### Feature Toggle Config

```lua
ix.config.Add("enableStamina", true, "Enable stamina system", function(old, new)
    if new then
        hook.Run("StaminaEnabled")
    else
        hook.Run("StaminaDisabled")

        -- Reset all stamina
        for _, client in ipairs(player.GetAll()) do
            client:SetNWFloat("stamina", 100)
        end
    end
end, {category = "gameplay"})
```

### Color Config

```lua
ix.config.Add("chatColor", Color(255, 255, 175), "Default chat message color", nil, {
    category = "chat"
})

-- Usage
local color = ix.config.Get("chatColor", Color(255, 255, 255))
chat.AddText(color, "Message")
```

## Common Issues

### Config Not Persisting

**Cause**: Server crashing before save or file permission issues.

**Fix**: Configs auto-save. Check `data/helix/` folder permissions.

### Config Not Syncing to Clients

**Cause**: Using `bNoNetworking = true` or client hasn't received sync.

**Fix**: Don't use `bNoNetworking` unless intentional:
```lua
-- This WON'T sync
ix.config.Add("key", value, "desc", nil, nil, true)

-- This WILL sync
ix.config.Add("key", value, "desc", nil, nil, false)
```

### Callback Not Firing

**Cause**: Callback only runs on server when value changes.

**Fix**: Callbacks are server-only:
```lua
ix.config.Add("key", value, "desc", function(old, new)
    -- This only runs on SERVER
    if SERVER then
        print("Value changed")
    end
end)
```

### Type Mismatch Error

**Cause**: Trying to set config to wrong type.

**Fix**: Maintain type consistency:
```lua
ix.config.Add("maxMoney", 1000, "desc")  -- Type: number

-- WRONG
ix.config.Set("maxMoney", "5000")  -- Type mismatch!

-- CORRECT
ix.config.Set("maxMoney", 5000)
```

## Related Hooks

### InitializedConfig

Called after all configs are loaded.

```lua
hook.Add("InitializedConfig", "MyHook", function()
    -- Safe to access configs here
    print("Chat range:", ix.config.Get("chatRange"))

    -- Override defaults
    ix.config.SetDefault("maxMoney", 50000)
end)
```

### SaveData

Called periodically to save data.

```lua
hook.Add("SaveData", "MyHook", function()
    -- Config saves automatically
    -- Save your custom data here
end)
```

## See Also

- [Data API](data.md) - File storage system used by config
- [Plugin API](plugin.md) - Creating plugins with configs
- [Option API](option.md) - Client-side preference system
- Source: `gamemode/core/sh_config.lua`
