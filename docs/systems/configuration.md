# Configuration System

> **Reference**: `gamemode/config/sh_config.lua`, `gamemode/core/libs/sh_option.lua`

Helix provides two configuration systems:
- **ix.config** - Server-wide settings (admin-configurable)
- **ix.option** - Per-player client settings

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Add()`** for server settings. Don't create custom config files. Framework handles:
- Database persistence
- Network synchronization
- Admin UI for changing values
- Type validation
- Callbacks on change

## Server Configuration (ix.config)

### Adding Config Options

```lua
ix.config.Add("configName", defaultValue, "Description", callback, options)

-- Example
ix.config.Add("maxCharacters", 5, "Maximum characters per player", nil, {
    data = {min = 1, max = 10},
    category = "characters"
})
```

### Getting Config Values

```lua
local value = ix.config.Get("configName", defaultValue)

-- Example
local maxChars = ix.config.Get("maxCharacters", 5)
```

### Config with Callback

```lua
ix.config.Add("respawnTime", 10, "Respawn delay in seconds", function(oldValue, newValue)
    print("Respawn time changed from", oldValue, "to", newValue)
end)
```

## Client Options (ix.option)

### Adding Options

```lua
ix.option.Add("optionName", ix.type.bool, defaultValue, {
    category = "Category Name"
})

-- Example
ix.option.Add("showHUD", ix.type.bool, true, {
    category = "display"
})
```

### Getting Option Values

```lua
-- CLIENT
local value = ix.option.Get("optionName", default)
```

## See Also

- Configuration: `gamemode/config/sh_config.lua`
- Options: `gamemode/core/libs/sh_option.lua`
