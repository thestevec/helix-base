# Database Operations Guide

> **Reference**: `gamemode/core/libs/sv_database.lua`

Practical guide to working with the Helix database system for custom queries and data persistence.

## ⚠️ Important: Use Helix's Database System

**Always use `ix.db` and `mysql:` query builder** rather than raw SQL queries. The framework provides:
- Automatic connection management
- SQLite/MySQL compatibility layer
- Prepared statement protection against SQL injection
- Query builder for safe, readable queries

## Quick Reference

```lua
-- Use query builder (recommended)
local query = mysql:Select("ix_characters")
query:Select("name")
query:Where("id", characterID)
query:Callback(function(result)
    if result and #result > 0 then
        print(result[1].name)
    end
end)
query:Execute()

-- Don't use raw SQL
mysql:Query("SELECT * FROM ix_characters WHERE id = " .. characterID)  -- UNSAFE!
```

## Basic Queries

### SELECT Query

```lua
-- Get character by ID
local query = mysql:Select("ix_characters")
query:Select("character_id")
query:Select("name")
query:Select("money")
query:Where("character_id", charID)
query:Callback(function(result)
    if result and #result > 0 then
        local data = result[1]
        print(data.name, data.money)
    end
end)
query:Execute()
```

### INSERT Query

```lua
-- Insert new record
local query = mysql:Insert("ix_plugin_data")
query:Insert("key", "playerStats")
query:Insert("value", util.TableToJSON({kills = 0, deaths = 0}))
query:Callback(function(result, status, lastID)
    print("Inserted with ID:", lastID)
end)
query:Execute()
```

### UPDATE Query

```lua
-- Update existing record
local query = mysql:Update("ix_characters")
query:Update("money", 1000)
query:Where("character_id", charID)
query:Callback(function()
    print("Updated character money")
end)
query:Execute()
```

### DELETE Query

```lua
-- Delete record
local query = mysql:Delete("ix_characters")
query:Where("character_id", charID)
query:Callback(function()
    print("Deleted character")
end)
query:Execute()
```

## Practical Examples

### Example 1: Player Statistics System

```lua
-- Save stats
function SavePlayerStats(client, stats)
    local steamID = client:SteamID()

    local query = mysql:Select("ix_plugin_stats")
    query:Where("steam_id", steamID)
    query:Callback(function(result)
        if result and #result > 0 then
            -- Update existing
            local update = mysql:Update("ix_plugin_stats")
            update:Update("kills", stats.kills)
            update:Update("deaths", stats.deaths)
            update:Where("steam_id", steamID)
            update:Execute()
        else
            -- Insert new
            local insert = mysql:Insert("ix_plugin_stats")
            insert:Insert("steam_id", steamID)
            insert:Insert("kills", stats.kills)
            insert:Insert("deaths", stats.deaths)
            insert:Execute()
        end
    end)
    query:Execute()
end

-- Load stats
function LoadPlayerStats(client, callback)
    local query = mysql:Select("ix_plugin_stats")
    query:Where("steam_id", client:SteamID())
    query:Callback(function(result)
        if result and #result > 0 then
            callback({
                kills = tonumber(result[1].kills) or 0,
                deaths = tonumber(result[1].deaths) or 0
            })
        else
            callback({kills = 0, deaths = 0})
        end
    end)
    query:Execute()
end
```

### Example 2: Ban System

```lua
-- Ban player
function BanPlayer(steamID, reason, duration, adminID)
    local query = mysql:Insert("ix_bans")
    query:Insert("steam_id", steamID)
    query:Insert("reason", reason)
    query:Insert("duration", duration)
    query:Insert("admin_id", adminID)
    query:Insert("timestamp", os.time())
    query:Execute()
end

-- Check if banned
function CheckBan(steamID, callback)
    local query = mysql:Select("ix_bans")
    query:Where("steam_id", steamID)
    query:Callback(function(result)
        if result and #result > 0 then
            local ban = result[1]
            local unbanTime = tonumber(ban.timestamp) + tonumber(ban.duration)

            if os.time() < unbanTime then
                callback(true, ban.reason, unbanTime - os.time())
            else
                callback(false)
            end
        else
            callback(false)
        end
    end)
    query:Execute()
end
```

### Example 3: Leaderboard System

```lua
-- Get top players by money
function GetTopPlayers(limit, callback)
    local query = mysql:Select("ix_characters")
    query:Select("name")
    query:Select("money")
    query:OrderByDesc("money")
    query:Limit(limit or 10)
    query:Callback(callback)
    query:Execute()
end

-- Usage
GetTopPlayers(5, function(result)
    if result then
        for i, player in ipairs(result) do
            print(i, player.name, "$" .. player.money)
        end
    end
end)
```

## Best Practices

### ✅ DO
- Use query builder for all queries
- Use callbacks for async operations
- Sanitize with prepared statements (automatic with query builder)
- Check callback results before accessing
- Use transactions for multiple related queries

### ❌ DON'T
- Don't use string concatenation in queries
- Don't block server waiting for queries
- Don't query in loops (batch instead)
- Don't store large blobs in database

## See Also

- [Networking Guide](networking.md)
- [Data System](../libraries/storage.md)
- Source: `gamemode/core/libs/sv_database.lua`
