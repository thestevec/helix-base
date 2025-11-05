# Data Library (ix.data)

> **Reference**: `gamemode/core/sh_data.lua`

The data library provides simple key-value file storage for persistent data in the `data/helix/` folder. It handles automatic caching, schema-specific storage, and JSON serialization.

## ⚠️ Important: Use Built-in Helix Functions

**Always use ix.data.Set/Get** rather than creating custom file storage systems. The framework provides:
- Automatic JSON serialization/deserialization
- In-memory caching for fast reads
- Schema and map-specific storage paths
- Automatic folder creation
- Error handling for corrupted data
- Auto-save timer every 10 minutes (server-side)

## Core Concepts

### What is ix.data?

The data library is Helix's file-based persistence system for simple data storage. It's used for:
- **Plugin data** - Storing plugin-specific settings and state
- **Configuration** - Saving server configuration (used by `ix.config`, line 153-213 in sh_config.lua)
- **Client options** - Storing user preferences (used by `ix.option`, line 141-261 in sh_option.lua)
- **Game state** - Persistent world state like in-game time (used by `ix.date`, line 28-93 in sh_date.lua)

### Storage Hierarchy

Files are stored in `data/helix/` with this structure:

```
data/helix/
├── mykey.txt                    (global, map-ignored)
├── myglobalkey.txt              (global)
└── myschema/                    (schema-specific)
    ├── mykey.txt                (schema, map-ignored)
    └── gm_flatgrass/            (map-specific)
        └── mykey.txt            (schema + map)
```

### Key Terms

- **Key**: File name (without `.txt` extension)
- **Global**: Saves to `data/helix/` (shared across all schemas)
- **Schema-specific**: Saves to `data/helix/schemaname/` (per schema)
- **Map-specific**: Saves to `data/helix/schemaname/mapname/` (per map)
- **Cache**: In-memory storage (`ix.data.stored`) for fast access

## Using the Data Library

### ix.data.Set

**Reference**: `gamemode/core/sh_data.lua:19`

Saves data to a file in the data folder with automatic serialization.

```lua
-- Save simple value (schema + map specific by default)
ix.data.Set("myPluginData", {enabled = true, count = 5})

-- Load it back
local data = ix.data.Get("myPluginData", {enabled = false, count = 0})
print(data.enabled)  -- true
print(data.count)    -- 5

-- Save globally (shared across all schemas)
ix.data.Set("globalSettings", {version = "1.0"}, true)

-- Save for schema, ignore map (shared across all maps in schema)
ix.data.Set("schemaConfig", {difficulty = "hard"}, false, true)

-- Example: Save player's last position
function PLUGIN:PlayerDisconnected(client)
    local character = client:GetCharacter()
    if not character then return end

    local data = {
        position = client:GetPos(),
        angles = client:EyeAngles()
    }

    ix.data.Set("lastPos_" .. character:GetID(), data, false, true)
end
```

**Parameters:**
- `key` - File name (string, `.txt` added automatically)
- `value` - Data to save (any serializable type: tables, strings, numbers, booleans)
- `bGlobal` - Save globally across schemas (default: `false`)
- `bIgnoreMap` - Ignore map name in path (default: `false`)

**Returns:** File path where data was saved

### ix.data.Get

**Reference**: `gamemode/core/sh_data.lua:49`

Loads data from a file with automatic deserialization and caching.

```lua
-- Get with default value (returns default if file doesn't exist)
local data = ix.data.Get("myPluginData", {enabled = false})

-- Get global data
local globalData = ix.data.Get("globalSettings", {}, true)

-- Get schema data (ignoring map)
local schemaData = ix.data.Get("schemaConfig", {}, false, true)

-- Force refresh from disk (bypass cache)
local freshData = ix.data.Get("myPluginData", {}, false, false, true)

-- Example: Load player's last position
function PLUGIN:PlayerLoadedCharacter(client, character)
    local data = ix.data.Get("lastPos_" .. character:GetID(), nil, false, true)

    if data then
        client:SetPos(data.position)
        client:SetEyeAngles(data.angles)
        client:Notify("Restored your last position!")
    end
end
```

**Parameters:**
- `key` - File name (string)
- `default` - Value to return if file doesn't exist or is corrupted
- `bGlobal` - Load from global folder (default: `false`)
- `bIgnoreMap` - Ignore map name in path (default: `false`)
- `bRefresh` - Force reload from disk, bypass cache (default: `false`)

**Returns:** Loaded data or default value

**How caching works** (line 50-56):
- First call loads from disk and caches
- Subsequent calls return cached value instantly
- Use `bRefresh = true` to force disk read

### ix.data.Delete

**Reference**: `gamemode/core/sh_data.lua:99`

Deletes a saved file and removes it from cache.

```lua
-- Delete schema+map specific file
ix.data.Delete("myPluginData")

-- Delete global file
ix.data.Delete("globalSettings", true)

-- Delete schema file (ignoring map)
ix.data.Delete("schemaConfig", false, true)

-- Example: Clear player's saved position
function COMMAND:OnRun(client, arguments)
    local character = client:GetCharacter()
    local deleted = ix.data.Delete("lastPos_" .. character:GetID(), false, true)

    if deleted then
        return "Cleared saved position"
    else
        return "No saved position found"
    end
end
```

**Parameters:**
- `key` - File name (string)
- `bGlobal` - Delete from global folder (default: `false`)
- `bIgnoreMap` - Ignore map name in path (default: `false`)

**Returns:** `true` if file was deleted, `false` if file didn't exist

## Storage Scopes

### Schema + Map Specific (Default)

```lua
-- Saves to: data/helix/myschema/gm_flatgrass/mydata.txt
ix.data.Set("mydata", {value = 100})
local data = ix.data.Get("mydata", {value = 0})
```

**Use for:** Map-specific state like entity positions, door ownership

### Schema-Specific (Ignore Map)

```lua
-- Saves to: data/helix/myschema/mydata.txt
ix.data.Set("mydata", {value = 100}, false, true)
local data = ix.data.Get("mydata", {value = 0}, false, true)
```

**Use for:** Schema configuration that applies to all maps

### Global (All Schemas)

```lua
-- Saves to: data/helix/mydata.txt
ix.data.Set("mydata", {value = 100}, true)
local data = ix.data.Get("mydata", {value = 0}, true)
```

**Use for:** Settings that apply to entire server across all schemas

### Global + Map-Ignored

```lua
-- Same as global (saves to: data/helix/mydata.txt)
ix.data.Set("mydata", {value = 100}, true, true)
local data = ix.data.Get("mydata", {value = 0}, true, true)
```

**Use for:** Truly global data (bIgnoreMap has no effect when bGlobal is true)

## Auto-Save System

**Reference**: `gamemode/core/sh_data.lua:115`

The framework automatically calls the `SaveData` hook every 10 minutes on the server.

```lua
-- Hook called every 600 seconds (10 minutes)
function PLUGIN:SaveData()
    -- Save your plugin's data
    local data = {
        playerCount = #player.GetAll(),
        timestamp = os.time()
    }

    ix.data.Set("pluginStats", data, false, true)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create your own save timer
timer.Create("MyPluginSave", 600, 0, function()
    -- Save data
end)

-- CORRECT: Use SaveData hook
function PLUGIN:SaveData()
    -- Save data
end
```

## Complete Example

```lua
-- Plugin that tracks total money given by admins
PLUGIN.name = "Money Tracker"
PLUGIN.description = "Tracks admin money givings"
PLUGIN.author = "YourName"

-- Load data when plugin loads
function PLUGIN:InitializedPlugins()
    -- Load global tracking data (shared across schemas)
    self.moneyGiven = ix.data.Get("moneyTracking", {
        total = 0,
        perAdmin = {}
    }, true, true)

    print("Total money given: " .. self.moneyGiven.total)
end

-- Track when admin gives money
function PLUGIN:CharacterMoneyGiven(character, amount, client)
    -- Only track if given by admin
    if not client or not client:IsAdmin() then return end

    -- Update totals
    self.moneyGiven.total = self.moneyGiven.total + amount

    local steamID = client:SteamID()
    self.moneyGiven.perAdmin[steamID] = (self.moneyGiven.perAdmin[steamID] or 0) + amount

    -- Save immediately for important data
    ix.data.Set("moneyTracking", self.moneyGiven, true, true)

    print(string.format("%s gave $%d (total: $%d)",
        client:Name(), amount, self.moneyGiven.perAdmin[steamID]))
end

-- Command to view stats
ix.command.Add("MoneyStats", {
    adminOnly = true,
    OnRun = function(self, client)
        local data = PLUGIN.moneyGiven

        client:ChatPrint("=== Money Tracking Stats ===")
        client:ChatPrint("Total given: $" .. data.total)

        for steamID, amount in pairs(data.perAdmin) do
            local name = player.GetBySteamID(steamID)
            name = IsValid(name) and name:Name() or steamID

            client:ChatPrint(string.format("%s: $%d", name, amount))
        end
    end
})

-- Auto-save with the framework's save timer
function PLUGIN:SaveData()
    ix.data.Set("moneyTracking", self.moneyGiven, true, true)
    print("Money tracking data saved")
end
```

## Best Practices

### ✅ DO

- Use `ix.data.Set/Get` for simple persistent storage
- Provide sensible default values in `ix.data.Get()`
- Use schema-specific storage for gameplay data
- Use global storage for server-wide settings
- Use the `SaveData` hook for periodic saves
- Cache frequently accessed data in plugin variables
- Delete old data with `ix.data.Delete()` when no longer needed

### ❌ DON'T

- Don't use for large datasets (use database instead)
- Don't save every frame or very frequently (causes lag)
- Don't forget default values in `Get()` calls
- Don't use for character/player data (use database via `Character` methods)
- Don't create custom file.Write() implementations
- Don't save sensitive data unencrypted
- Don't assume data always exists (always provide defaults)

## Common Patterns

### Pattern 1: Plugin Configuration

```lua
function PLUGIN:InitializedPlugins()
    -- Load config with defaults
    self.config = ix.data.Get("myPluginConfig", {
        enabled = true,
        maxValue = 100,
        multiplier = 1.5
    }, false, true)  -- Schema-specific, map-ignored
end

function PLUGIN:SaveData()
    ix.data.Set("myPluginConfig", self.config, false, true)
end

-- Command to change config
ix.command.Add("ConfigSet", {
    adminOnly = true,
    arguments = {ix.type.string, ix.type.number},
    OnRun = function(self, client, key, value)
        if PLUGIN.config[key] == nil then
            return "Invalid config key"
        end

        PLUGIN.config[key] = value
        ix.data.Set("myPluginConfig", PLUGIN.config, false, true)

        return string.format("Set %s to %s", key, value)
    end
})
```

### Pattern 2: Per-Character Persistent Data

```lua
-- Save data for specific character
function PLUGIN:SaveCharacterData(character, data)
    local key = "charData_" .. character:GetID()
    ix.data.Set(key, data, false, true)
end

function PLUGIN:LoadCharacterData(character)
    local key = "charData_" .. character:GetID()
    return ix.data.Get(key, {
        achievements = {},
        stats = {}
    }, false, true)
end

-- Usage
function PLUGIN:PlayerLoadedCharacter(client, character)
    local data = self:LoadCharacterData(character)

    client:ChatPrint("You have " .. #data.achievements .. " achievements!")
end
```

### Pattern 3: Map-Specific Entity Data

```lua
-- Save entity positions for current map
function PLUGIN:SaveEntities()
    local entities = {}

    for _, ent in ipairs(ents.FindByClass("prop_physics")) do
        if ent.isPersistent then
            table.insert(entities, {
                pos = ent:GetPos(),
                ang = ent:GetAngles(),
                model = ent:GetModel()
            })
        end
    end

    -- Map-specific storage (default)
    ix.data.Set("persistentEntities", entities)
end

function PLUGIN:LoadEntities()
    local entities = ix.data.Get("persistentEntities", {})

    for _, data in ipairs(entities) do
        local ent = ents.Create("prop_physics")
        ent:SetModel(data.model)
        ent:SetPos(data.pos)
        ent:SetAngles(data.ang)
        ent:Spawn()
        ent.isPersistent = true
    end
end

-- Save on map change
function PLUGIN:ShutDown()
    self:SaveEntities()
end

-- Load on map start
function PLUGIN:LoadData()
    self:LoadEntities()
end
```

## Common Issues

### Issue: Data not persisting between sessions

**Cause**: Forgetting to call `ix.data.Set()` or not using `SaveData` hook
**Fix**: Ensure you're saving data before server shutdown

```lua
-- Use SaveData hook for automatic saves
function PLUGIN:SaveData()
    ix.data.Set("myData", self.data)
end

-- Or save immediately for critical data
ix.data.Set("importantData", data)
```

### Issue: Data returning nil unexpectedly

**Cause**: No default value provided to `ix.data.Get()`
**Fix**: Always provide a default value

```lua
-- WRONG
local data = ix.data.Get("myData")  -- Returns nil if file missing
data.value = 100  -- ERROR: attempt to index nil!

-- CORRECT
local data = ix.data.Get("myData", {value = 0})
data.value = 100  -- Safe!
```

### Issue: Wrong storage scope (global vs schema vs map)

**Cause**: Using wrong `bGlobal` or `bIgnoreMap` parameters
**Fix**: Choose appropriate scope for your data

```lua
-- Schema config (all maps in schema)
ix.data.Set("config", data, false, true)

-- Map-specific (unique per map)
ix.data.Set("entities", data, false, false)

-- Global (all schemas, all maps)
ix.data.Set("serverVersion", version, true, true)
```

### Issue: Performance problems with large data

**Cause**: Using ix.data for large datasets
**Fix**: Use database for large or frequently queried data

```lua
-- WRONG: Saving thousands of records
local bigData = {}
for i = 1, 10000 do
    bigData[i] = {lots = "of", data = "here"}
end
ix.data.Set("bigData", bigData)  -- Slow!

-- CORRECT: Use database for large data
ix.db.Query("INSERT INTO mytable VALUES (...)")
```

## When to Use ix.data vs Database

### Use ix.data for:
- ✅ Plugin configuration
- ✅ Server settings
- ✅ Small persistent state
- ✅ Client preferences
- ✅ Schema/map metadata
- ✅ Simple key-value data

### Use Database for:
- ✅ Character data (use `Character` methods)
- ✅ Inventory systems
- ✅ Large datasets
- ✅ Frequently queried data
- ✅ Relational data
- ✅ Player accounts

## Real Framework Usage

The data library is used extensively in Helix core:

### Configuration System
**Reference**: `gamemode/core/sh_config.lua:153-213`
```lua
-- Loads global and schema configs
local globals = ix.data.Get("config", nil, true, true)
local data = ix.data.Get("config", nil, false, true)
```

### Options System
**Reference**: `gamemode/core/libs/sh_option.lua:141-261`
```lua
-- Loads client options globally
local options = ix.data.Get("options", nil, true, true)

-- Saves client options
ix.data.Set("options", ix.option.client, true, true)
```

### Date System
**Reference**: `gamemode/core/libs/sh_date.lua:28-93`
```lua
-- Loads in-game date (schema-specific, map-ignored)
local currentDate = ix.data.Get("date", nil, false, true)

-- Saves in-game date
ix.data.Set("date", ix.date.lib.serialize(ix.date.current), false, true)
```

### Plugin System
**Reference**: `gamemode/core/libs/sh_plugin.lua:248-300`
```lua
-- Tracks unloaded plugins globally
ix.plugin.unloaded = ix.data.Get("unloaded", {}, true, true)
ix.data.Set("unloaded", unloaded, true, true)
```

## See Also

- [Storage Library](storage.md) - For entity-based storage systems
- [Database Library](database.md) - For large-scale data storage
- [Configuration System](../systems/configuration.md) - Uses ix.data internally
- [Plugin Development](../plugins/creating-plugins.md) - Using ix.data in plugins
- Source: `gamemode/core/sh_data.lua`
