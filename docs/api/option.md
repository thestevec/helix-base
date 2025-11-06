# Option API (ix.option)

> **Reference**: `gamemode/core/libs/sh_option.lua`

The option API manages client-side user preferences that persist across sessions. Options appear in the F1 menu and can be synchronized to the server.

## Core Concepts

Options are client-side settings like language preference, UI scale, graphical settings, etc. They:
- Save to `data/helix/options.txt`
- Can be networked to server
- Auto-generate UI in options menu
- Support multiple types (bool, number, string, etc.)

## Functions

### ix.option.Add

**Realm**: Shared

```lua
ix.option.Add(key, optionType, default, data)
```

Registers a new option.

**Example**:
```lua
-- Client-only
ix.option.Add("showHUD", ix.type.bool, true, {
    category = "interface"
})

-- Networked to server
ix.option.Add("language", ix.type.array, "english", {
    category = "general",
    bNetworked = true,
    populate = function()
        return {
            ["english"] = "English",
            ["french"] = "Français"
        }
    end
})

-- Number with bounds
ix.option.Add("uiScale", ix.type.number, 1, {
    category = "interface",
    min = 0.5,
    max = 2,
    decimals = 1
})
```

### ix.option.Get

**Realm**: Client / Server

```lua
-- Client
local value = ix.option.Get(key, default)

-- Server (networked options only)
local value = ix.option.Get(client, key, default)
```

Gets option value.

**Example**:
```lua
-- Client
if ix.option.Get("showHUD", true) then
    -- Draw HUD
end

-- Server
local lang = ix.option.Get(client, "language", "english")
```

### ix.option.Set

**Realm**: Client

```lua
ix.option.Set(key, value, bNoSave)
```

Sets option value.

**Example**:
```lua
ix.option.Set("language", "french")
ix.option.Set("showHUD", false)
```

## Complete Example

```lua
-- Register custom options
if CLIENT then
    ix.option.Add("headbob", ix.type.bool, true, {
        category = "gameplay",
        OnChanged = function(oldValue, newValue)
            print("Headbob:", newValue)
        end
    })

    ix.option.Add("fov", ix.type.number, 90, {
        category = "gameplay",
        min = 75,
        max = 110,
        OnChanged = function(oldValue, newValue)
            LocalPlayer():SetFOV(newValue)
        end
    })
end
```

## See Also

- [Config API](config.md) - Server configuration
- [Language API](language.md) - Language preferences
- Source: `gamemode/core/libs/sh_option.lua`
