# Plugin API (ix.plugin)

> **Reference**: `gamemode/core/libs/sh_plugin.lua`

The plugin API manages loading and organization of plugins and schemas. Plugins extend Helix functionality through modular, self-contained packages.

## Core Concepts

Plugins are modular extensions that:
- Load from `helix/plugins/` or `schema/plugins/`
- Can include items, factions, classes, entities
- Have their own hooks and functions
- Can be enabled/disabled via config menu

## Plugin Structure

```
plugins/myplugin/
├── sh_plugin.lua          -- Main plugin file
├── items/                  -- Item definitions
├── factions/              -- Faction definitions
├── classes/               -- Class definitions
├── entities/              -- Custom entities
├── derma/                 -- UI files
├── languages/             -- Translations
└── libs/                  -- Libraries
```

## Plugin File

```lua
-- sh_plugin.lua
PLUGIN.name = "My Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Does something cool"

function PLUGIN:PlayerSay(client, text)
    if text == "/test" then
        client:Notify("Plugin works!")
        return ""
    end
end

function PLUGIN:OnLoaded()
    print("Plugin loaded!")
end
```

## Functions

### ix.plugin.Get

```lua
local plugin = ix.plugin.Get("pluginID")
```

Gets plugin by ID.

### Plugin Data Storage

```lua
-- Save data
PLUGIN:SetData(value, bGlobal, bIgnoreMap)

-- Load data
local data = PLUGIN:GetData(default, bGlobal, bIgnoreMap)
```

**Example**:
```lua
-- Save plugin settings
PLUGIN:SetData({
    enabled = true,
    value = 100
}, false, true)

-- Load plugin settings
local settings = PLUGIN:GetData({
    enabled = false,
    value = 0
}, false, true)
```

## Hooks

Plugins can use standard Garry's Mod hooks:

```lua
function PLUGIN:PlayerSpawn(client)
    client:SetHealth(150)
end

function PLUGIN:OnCharacterCreated(client, character)
    character:SetData("pluginData", {})
end
```

## Complete Example

```lua
PLUGIN.name = "Welcome System"
PLUGIN.author = "Me"
PLUGIN.description = "Welcomes players"

if SERVER then
    function PLUGIN:PlayerInitialSpawn(client)
        timer.Simple(5, function()
            if IsValid(client) then
                client:Notify("Welcome to the server!")
            end
        end)
    end

    function PLUGIN:SaveData()
        -- Save any persistent data
        self:SetData(self.playerVisits or {}, false, true)
    end

    function PLUGIN:LoadData()
        self.playerVisits = self:GetData({}, false, true)
    end
end
```

## See Also

- [Config API](config.md) - Plugin configuration
- [Data API](data.md) - Plugin data storage
- Source: `gamemode/core/libs/sh_plugin.lua`
