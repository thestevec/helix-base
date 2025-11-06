# Data API (ix.data)

> **Reference**: `gamemode/core/sh_data.lua`

The data API provides simple file-based storage in the `data/helix/` directory. It handles serialization, caching, and organization of persistent data for schemas and plugins.

## ⚠️ Important: Use Built-in Helix Data System

**Always use Helix's built-in data functions** rather than direct file operations. The framework automatically provides:
- Automatic JSON serialization and deserialization
- Organized directory structure (global, schema, map-specific)
- Read/write caching for performance
- Schema and map isolation
- Backwards compatibility with old formats
- Safe file operations

**When to Use**:
- Saving plugin/schema settings
- Storing non-character data (bans, logs, etc.)
- Map-specific data
- Global server data

**When NOT to Use**:
- Character data (use character:SetData)
- Config options (use ix.config)
- Database-appropriate data (use ix.db)

## Core Concepts

### What is Data Storage?

The data system stores arbitrary Lua tables to disk:
- Files saved in `data/helix/`
- Can be global, schema-specific, or map-specific
- Automatically handles serialization
- Values are cached in memory
- Three storage levels:
  - **Global**: `data/helix/key.txt`
  - **Schema**: `data/helix/schema_name/key.txt`
  - **Map**: `data/helix/schema_name/mapname/key.txt`

### Key Terms

**Key**: Filename without extension (e.g., "bans" → `bans.txt`)
**Global**: Data shared across all schemas
**Schema-Only**: Data specific to current schema
**Map-Specific**: Data specific to current map and schema
**Cache**: In-memory copy of data for fast access

## Library Tables

### ix.data.stored

**Reference**: `gamemode/core/sh_data.lua:6`

**Realm**: Shared

Cache of loaded data indexed by key.

```lua
-- Check cached data
if ix.data.stored["mydata"] then
    print("Data is cached")
end

-- Clear cache for key
ix.data.stored["mydata"] = nil
```

**Note**: Clearing cache doesn't delete the file, only memory copy.

## Library Functions

### ix.data.Set

**Reference**: `gamemode/core/sh_data.lua:19`

**Realm**: Shared

```lua
local path = ix.data.Set(key, value, bGlobal, bIgnoreMap)
```

Writes data to a file and caches it.

**Parameters**:
- `key` (string) - Filename (without .txt extension)
- `value` (any) - Data to save (must be table-serializable)
- `bGlobal` (bool, optional) - Save globally (default: false = schema-specific)
- `bIgnoreMap` (bool, optional) - Ignore map folder (default: false = use map folder)

**Returns**: (string) Path where file was saved

**Example - Schema-Specific**:
```lua
-- Save to data/helix/myschema/mapname/bans.txt
ix.data.Set("bans", {
    ["STEAM_0:1:12345"] = {
        reason = "Cheating",
        time = os.time()
    }
})
```

**Example - Global Data**:
```lua
-- Save to data/helix/admins.txt
ix.data.Set("admins", {
    "STEAM_0:1:12345",
    "STEAM_0:1:67890"
}, true)  -- bGlobal = true
```

**Example - Schema-Wide (Ignore Map)**:
```lua
-- Save to data/helix/myschema/config.txt
ix.data.Set("customConfig", {
    setting1 = true,
    setting2 = 50
}, false, true)  -- bGlobal = false, bIgnoreMap = true
```

**Example - Complex Data**:
```lua
-- Save plugin data
local pluginData = {
    version = "1.0.0",
    settings = {
        enabled = true,
        interval = 300
    },
    users = {
        ["STEAM_0:1:12345"] = {
            points = 100,
            lastSeen = os.time()
        }
    }
}

ix.data.Set("myplugin", pluginData, false, true)
```

### ix.data.Get

**Reference**: `gamemode/core/sh_data.lua:49`

**Realm**: Shared

```lua
local value = ix.data.Get(key, default, bGlobal, bIgnoreMap, bRefresh)
```

Reads data from a file (or cache).

**Parameters**:
- `key` (string) - Filename (without .txt)
- `default` (any, optional) - Value if file doesn't exist
- `bGlobal` (bool, optional) - Load from global folder
- `bIgnoreMap` (bool, optional) - Load from schema folder (ignore map)
- `bRefresh` (bool, optional) - Skip cache and reload from disk

**Returns**: (any) Loaded data or default value

**Example - Basic Load**:
```lua
-- Load map-specific data
local bans = ix.data.Get("bans", {})

-- Load with default
local settings = ix.data.Get("settings", {
    enabled = true,
    timeout = 60
})
```

**Example - Global Load**:
```lua
-- Load global admins list
local admins = ix.data.Get("admins", {}, true)
```

**Example - Force Refresh**:
```lua
-- Force reload from disk (skip cache)
local data = ix.data.Get("mydata", {}, false, false, true)
```

**Example - Check Existence**:
```lua
local data = ix.data.Get("bans", nil)
if data == nil then
    print("No bans file exists")
else
    print("Found", table.Count(data), "bans")
end
```

### ix.data.Delete

**Reference**: `gamemode/core/sh_data.lua:99`

**Realm**: Shared

```lua
local success = ix.data.Delete(key, bGlobal, bIgnoreMap)
```

Deletes a data file and clears its cache.

**Parameters**:
- `key` (string) - Filename to delete
- `bGlobal` (bool, optional) - Delete from global folder
- `bIgnoreMap` (bool, optional) - Delete from schema folder

**Returns**: (bool) Whether deletion succeeded

**Example**:
```lua
-- Delete map-specific file
if ix.data.Delete("tempdata") then
    print("Deleted successfully")
end

-- Delete global file
ix.data.Delete("olddata", true)

-- Delete schema-wide file
ix.data.Delete("cache", false, true)
```

## Storage Paths

### Path Examples

```lua
-- Map-specific (default)
ix.data.Set("bans", {})
-- → data/helix/myschema/gm_flatgrass/bans.txt

-- Schema-specific (ignore map)
ix.data.Set("config", {}, false, true)
-- → data/helix/myschema/config.txt

-- Global
ix.data.Set("admins", {}, true)
-- → data/helix/admins.txt
```

## Complete Examples

### Ban System

```lua
-- Ban management with persistent storage
local BanSystem = {}

function BanSystem:AddBan(steamID, reason, duration)
    local bans = ix.data.Get("bans", {}, false, true)

    bans[steamID] = {
        reason = reason,
        timestamp = os.time(),
        expires = duration and (os.time() + duration) or nil,
        permanent = not duration
    }

    ix.data.Set("bans", bans, false, true)
end

function BanSystem:RemoveBan(steamID)
    local bans = ix.data.Get("bans", {}, false, true)

    if bans[steamID] then
        bans[steamID] = nil
        ix.data.Set("bans", bans, false, true)
        return true
    end

    return false
end

function BanSystem:IsBanned(steamID)
    local bans = ix.data.Get("bans", {}, false, true)
    local ban = bans[steamID]

    if not ban then
        return false
    end

    -- Check if temporary ban expired
    if ban.expires and os.time() >= ban.expires then
        self:RemoveBan(steamID)
        return false
    end

    return true, ban.reason
end

-- Usage
hook.Add("CheckPassword", "BanSystem", function(steamID64)
    local steamID = util.SteamIDFrom64(steamID64)
    local banned, reason = BanSystem:IsBanned(steamID)

    if banned then
        return false, "Banned: " .. (reason or "No reason given")
    end
end)
```

### Player Statistics

```lua
-- Track player statistics
local Stats = {}

function Stats:Load(steamID)
    local allStats = ix.data.Get("playerstats", {}, false, true)
    return allStats[steamID] or {
        playtime = 0,
        kills = 0,
        deaths = 0,
        joinCount = 0
    }
end

function Stats:Save(steamID, stats)
    local allStats = ix.data.Get("playerstats", {}, false, true)
    allStats[steamID] = stats
    ix.data.Set("playerstats", allStats, false, true)
end

hook.Add("PlayerInitialSpawn", "LoadStats", function(client)
    client.stats = Stats:Load(client:SteamID())
    client.stats.joinCount = client.stats.joinCount + 1
end)

hook.Add("PlayerDisconnected", "SaveStats", function(client)
    if client.stats then
        Stats:Save(client:SteamID(), client.stats)
    end
end)

hook.Add("PlayerDeath", "TrackDeaths", function(victim, inflictor, attacker)
    if victim.stats then
        victim.stats.deaths = victim.stats.deaths + 1
    end

    if IsValid(attacker) and attacker:IsPlayer() and attacker.stats then
        attacker.stats.kills = attacker.stats.kills + 1
    end
end)
```

### Plugin Settings

```lua
-- Plugin-specific settings storage
PLUGIN.name = "My Plugin"
PLUGIN.uniqueID = "myplugin"

function PLUGIN:LoadSettings()
    return ix.data.Get(self.uniqueID .. "_settings", {
        enabled = true,
        interval = 300,
        maxPlayers = 10
    }, false, true)
end

function PLUGIN:SaveSettings(settings)
    ix.data.Set(self.uniqueID .. "_settings", settings, false, true)
end

function PLUGIN:Initialize()
    self.settings = self:LoadSettings()

    -- Use settings
    if self.settings.enabled then
        self:StartSystem()
    end
end

-- Command to modify settings
ix.command.Add("PluginSetInterval", {
    adminOnly = true,
    arguments = ix.type.number,
    OnRun = function(self, client, interval)
        local settings = PLUGIN:LoadSettings()
        settings.interval = interval
        PLUGIN:SaveSettings(settings)
        PLUGIN.settings = settings

        return "Set interval to " .. interval
    end
})
```

### Map-Specific Data

```lua
-- Track door ownership per map
local Doors = {}

function Doors:SetOwner(doorID, steamID)
    local data = ix.data.Get("doorowners", {})
    data[doorID] = {
        owner = steamID,
        timestamp = os.time()
    }
    ix.data.Set("doorowners", data)
end

function Doors:GetOwner(doorID)
    local data = ix.data.Get("doorowners", {})
    return data[doorID]
end

function Doors:RemoveOwner(doorID)
    local data = ix.data.Get("doorowners", {})
    data[doorID] = nil
    ix.data.Set("doorowners", data)
end

-- Automatically map-specific because not using bIgnoreMap
```

### Leaderboard System

```lua
-- Global leaderboard across all schemas
local Leaderboard = {}

function Leaderboard:AddScore(steamID, name, score)
    local scores = ix.data.Get("leaderboard", {}, true)  -- Global

    table.insert(scores, {
        steamID = steamID,
        name = name,
        score = score,
        timestamp = os.time()
    })

    -- Sort by score
    table.sort(scores, function(a, b)
        return a.score > b.score
    end)

    -- Keep only top 100
    while #scores > 100 do
        table.remove(scores)
    end

    ix.data.Set("leaderboard", scores, true)
end

function Leaderboard:GetTop(count)
    local scores = ix.data.Get("leaderboard", {}, true)
    local result = {}

    for i = 1, math.min(count or 10, #scores) do
        table.insert(result, scores[i])
    end

    return result
end
```

## Best Practices

### ✅ DO

- Use descriptive keys for files
- Provide default values in Get()
- Use appropriate storage level (global/schema/map)
- Store simple, serializable data
- Clean up old/temporary data
- Check if data exists before assuming it's loaded
- Use schema-wide storage (bIgnoreMap) for shared data

### ❌ DON'T

- Don't store functions or metatables (not serializable)
- Don't store player entities or complex objects
- Don't use for frequently changing data (use database)
- Don't forget bGlobal/bIgnoreMap parameters
- Don't assume data exists - always provide defaults
- Don't store character data (use character:SetData)
- Don't use for real-time data (file I/O is slow)
- Don't use special characters in keys

## Common Patterns

### Lazy Loading

```lua
local myData

function GetMyData()
    if not myData then
        myData = ix.data.Get("mydata", {}, false, true)
    end
    return myData
end

function SaveMyData()
    ix.data.Set("mydata", myData, false, true)
end
```

### Versioned Data

```lua
function LoadVersionedData()
    local data = ix.data.Get("mydata", nil, false, true)

    if not data or not data.version then
        -- Create new format
        data = {
            version = 2,
            settings = {}
        }
    elseif data.version == 1 then
        -- Migrate old format
        data = {
            version = 2,
            settings = data  -- Wrap old data
        }
    end

    return data
end
```

### Periodic Auto-Save

```lua
-- Save data every 10 minutes
hook.Add("SaveData", "AutoSavePlugin", function()
    ix.data.Set("plugindata", PLUGIN.dataCache, false, true)
end)
```

## Common Issues

### Data Not Persisting

**Cause**: Server crash before write or file permissions.

**Fix**: Data saves immediately. Check file permissions on `data/helix/` directory.

### Data Not Loading

**Cause**: Wrong bGlobal/bIgnoreMap parameters or corrupted file.

**Fix**: Use same parameters for Get and Set:
```lua
-- These must match
ix.data.Set("key", value, false, true)
local data = ix.data.Get("key", {}, false, true)
```

### "Table is not serializable"

**Cause**: Trying to save functions, entities, or metatables.

**Fix**: Only store pure data:
```lua
-- WRONG
ix.data.Set("bad", {
    func = function() end,
    ent = someEntity
})

-- CORRECT
ix.data.Set("good", {
    number = 5,
    string = "text",
    table = {nested = true}
})
```

### Cache Not Updating

**Cause**: Another script modified file directly.

**Fix**: Force refresh:
```lua
local data = ix.data.Get("key", {}, false, false, true)
```

## Related Hooks

### SaveData

Called periodically for auto-saving.

```lua
hook.Add("SaveData", "MySaveHook", function()
    -- Save your data here
    ix.data.Set("mydata", myDataTable)
end)
```

## See Also

- [Config API](config.md) - Config option storage
- [Character API](character.md) - Character data storage (character:SetData)
- [Database API](database.md) - SQL database for structured data
- Source: `gamemode/core/sh_data.lua`
