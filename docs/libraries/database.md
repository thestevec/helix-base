# Database Library (ix.db)

> **Reference**: `gamemode/core/libs/sv_database.lua`

The database library provides MySQL and SQLite database connectivity with a query builder interface. It manages Helix's core tables and schema system for character, inventory, and item persistence.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's database system** through Character, Inventory, and Item methods rather than writing raw queries. The framework provides:
- Automatic table creation and schema management
- Safe query building with prepared statements
- Connection pooling and error handling
- Automatic migrations when fields are added
- Support for MySQL (mysqloo/tmysql4) and SQLite
- Protection against SQL injection

## Core Concepts

### What is ix.db?

The database library is Helix's server-side persistence layer. It:
- Manages database connections (MySQL or SQLite)
- Creates and maintains core tables (characters, inventories, items, players)
- Provides a query builder interface through the `mysql` library
- Handles schema migrations automatically
- Should **rarely be used directly** - use Character/Inventory/Item APIs instead

### Database Adapters

Helix supports multiple database backends:
- **SQLite** (default) - File-based database, no setup required
- **MySQL via mysqloo** - Recommended for production servers
- **MySQL via tmysql4** - Alternative MySQL module

### Core Tables

**Reference**: `gamemode/core/libs/sv_database.lua:69-112`

Helix creates these tables automatically:

| Table | Purpose | Key Fields |
|-------|---------|-----------|
| `ix_schema` | Tracks table schema versions | table, columns |
| `ix_characters` | Character data | id, name, steamid, faction, money, data |
| `ix_inventories` | Inventory containers | inventory_id, character_id, inventory_type |
| `ix_items` | Item instances | item_id, inventory_id, unique_id, x, y, data |
| `ix_players` | Player accounts | steamid, steam_name, play_time, last_join_time, data |

## Database Configuration

Configuration is stored in `garrysmod/settings/helix/server_config.txt`:

```lua
-- config/server.lua or server_config.txt
config.database = {
    adapter = "mysqloo",        -- or "tmysql4" or "sqlite"
    hostname = "localhost",
    username = "helix_user",
    password = "secure_password",
    database = "helix_db",
    port = 3306
}
```

**SQLite (Default)**:
```lua
config.database = {
    adapter = "sqlite"
}
```

## Database Connection

### ix.db.Connect

**Reference**: `gamemode/core/libs/sv_database.lua:18`

Connects to the database using configured adapter. Called automatically on server start.

```lua
-- Automatically called by InitPostEntity hook (line 160-163)
-- You should NOT call this manually

-- Connection happens automatically:
-- 1. Server starts
-- 2. InitPostEntity hook fires
-- 3. ix.db.Connect() called
-- 4. Database connected
```

## Query Building

Helix uses the `mysql` library for query building. **DO NOT** write raw SQL queries.

### SELECT Queries

```lua
-- Select all characters for a player
local query = mysql:Select("ix_characters")
    query:Select("id")
    query:Select("name")
    query:Select("faction")
    query:Where("steamid", client:SteamID64())
    query:Callback(function(result)
        if istable(result) then
            for _, row in ipairs(result) do
                print(row.name, row.faction)
            end
        end
    end)
query:Execute()

-- Select with multiple conditions
local query = mysql:Select("ix_characters")
    query:Where("steamid", steamID)
    query:Where("schema", Schema.folder)
    query:OrderByDesc("last_join_time")
    query:Limit(10)
    query:Callback(function(result)
        -- Handle results
    end)
query:Execute()
```

**From sh_character.lua:118-130**:
```lua
local query = mysql:Select("ix_characters")
    query:Where("steamid", steamID64)
    query:Where("schema", Schema and Schema.folder or "helix")
    if (id) then
        query:Where("id", id)
    end
    query:Callback(function(result)
        -- Character loading logic
    end)
query:Execute()
```

### INSERT Queries

```lua
-- Insert new character
local query = mysql:Insert("ix_characters")
    query:Insert("name", "John Doe")
    query:Insert("description", "A citizen")
    query:Insert("model", "models/player/group01/male_01.mdl")
    query:Insert("steamid", client:SteamID64())
    query:Insert("faction", "citizen")
    query:Insert("money", 100)
    query:Callback(function(result, status, lastID)
        print("Created character with ID: " .. lastID)
    end)
query:Execute()
```

**From sh_character.lua:53-88**:
```lua
local query = mysql:Insert("ix_characters")
    query:Insert("name", data.name or "")
    query:Insert("description", data.description or "")
    query:Insert("model", data.model or "models/error.mdl")
    query:Insert("schema", Schema and Schema.folder or "helix")
    query:Insert("create_time", data.createTime)
    query:Insert("steamid", data.steamID)
    query:Insert("faction", data.faction or "Unknown")
    query:Insert("money", data.money)
    query:Callback(function(result, status, lastID)
        -- Handle newly created character ID
    end)
query:Execute()
```

### UPDATE Queries

```lua
-- Update character data
local query = mysql:Update("ix_characters")
    query:Update("money", 500)
    query:Update("last_join_time", os.time())
    query:Where("id", characterID)
query:Execute()

-- Update with JSON data
local query = mysql:Update("ix_characters")
    query:Update("data", util.TableToJSON({
        achievements = {...},
        stats = {...}
    }))
    query:Where("id", characterID)
query:Execute()
```

**From sv_hooks.lua:259**:
```lua
local query = mysql:Update("ix_characters")
    query:Update("last_join_time", os.time())
    query:Where("id", character:GetID())
query:Execute()
```

### DELETE Queries

```lua
-- Delete character
local query = mysql:Delete("ix_characters")
    query:Where("id", characterID)
    query:Callback(function()
        print("Character deleted")
    end)
query:Execute()
```

**From sh_character.lua:1038-1060**:
```lua
local query = mysql:Delete("ix_characters")
    query:Where("id", id)
    query:Callback(function()
        -- Delete associated inventories and items
        local invQuery = mysql:Select("ix_inventories")
            invQuery:Where("character_id", id)
            invQuery:Callback(function(result)
                -- Handle cascading delete
            end)
        invQuery:Execute()
    end)
query:Execute()
```

## Schema Management

### ix.db.AddToSchema

**Reference**: `gamemode/core/libs/sv_database.lua:32`

Adds a field to a database table schema. Used when registering custom character variables.

```lua
-- Add custom field to ix_characters table
ix.db.AddToSchema("ix_characters", "customField", ix.type.string)

-- Automatically called by ix.char.RegisterVar
ix.char.RegisterVar("reputation", {
    field = "reputation",
    fieldType = ix.type.number,
    default = 0,
    isLocal = false  -- Saved to database
})
-- This calls ix.db.AddToSchema internally
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually ALTER tables
ix.db.Query("ALTER TABLE ix_characters ADD COLUMN mycol VARCHAR(255)")

-- CORRECT: Use ix.db.AddToSchema
ix.db.AddToSchema("ix_characters", "mycol", ix.type.string)
```

## Type Mappings

**Reference**: `gamemode/core/libs/sv_database.lua:5-13`

Helix types map to SQL types:

| ix.type | SQL Type | Usage |
|---------|----------|-------|
| `ix.type.string` | VARCHAR(255) | Short text (names, titles) |
| `ix.type.text` | TEXT | Long text (descriptions) |
| `ix.type.number` | INT(11) | Integer values |
| `ix.type.steamid` | VARCHAR(20) | Steam IDs |
| `ix.type.bool` | TINYINT(1) | Boolean flags |

## Advanced Usage (DANGEROUS)

### Direct Queries - Use With Caution

**⚠️ WARNING**: Only use these patterns if you absolutely cannot use Character/Inventory/Item APIs.

```lua
-- Custom table for plugin
function PLUGIN:LoadData()
    local query = mysql:Create("myplugin_data")
        query:Create("id", "INT(11) UNSIGNED NOT NULL AUTO_INCREMENT")
        query:Create("character_id", "INT(11) UNSIGNED NOT NULL")
        query:Create("custom_data", "TEXT")
        query:PrimaryKey("id")
    query:Execute()
end

-- Insert custom data
function PLUGIN:SaveCustomData(character, data)
    local query = mysql:Insert("myplugin_data")
        query:Insert("character_id", character:GetID())
        query:Insert("custom_data", util.TableToJSON(data))
    query:Execute()
end

-- Load custom data
function PLUGIN:LoadCustomData(character, callback)
    local query = mysql:Select("myplugin_data")
        query:Where("character_id", character:GetID())
        query:Callback(function(result)
            if istable(result) and #result > 0 then
                local data = util.JSONToTable(result[1].custom_data)
                callback(data)
            else
                callback(nil)
            end
        end)
    query:Execute()
end
```

## Complete Example

```lua
-- Plugin with custom database table
PLUGIN.name = "Quest System"
PLUGIN.description = "Tracks player quests"

-- Create custom table
function PLUGIN:InitPostEntity()
    timer.Simple(1, function()  -- Wait for database connection
        local query = mysql:Create("quest_progress")
            query:Create("id", "INT(11) UNSIGNED NOT NULL AUTO_INCREMENT")
            query:Create("character_id", "INT(11) UNSIGNED NOT NULL")
            query:Create("quest_id", "VARCHAR(60) NOT NULL")
            query:Create("progress", "INT(11) DEFAULT 0")
            query:Create("completed", "TINYINT(1) DEFAULT 0")
            query:PrimaryKey("id")
        query:Execute()
    end)
end

-- Save quest progress
function PLUGIN:SaveQuest(character, questID, progress, completed)
    -- Check if record exists
    local query = mysql:Select("quest_progress")
        query:Where("character_id", character:GetID())
        query:Where("quest_id", questID)
        query:Callback(function(result)
            if istable(result) and #result > 0 then
                -- Update existing
                local updateQuery = mysql:Update("quest_progress")
                    updateQuery:Update("progress", progress)
                    updateQuery:Update("completed", completed and 1 or 0)
                    updateQuery:Where("id", result[1].id)
                updateQuery:Execute()
            else
                -- Insert new
                local insertQuery = mysql:Insert("quest_progress")
                    insertQuery:Insert("character_id", character:GetID())
                    insertQuery:Insert("quest_id", questID)
                    insertQuery:Insert("progress", progress)
                    insertQuery:Insert("completed", completed and 1 or 0)
                insertQuery:Execute()
            end
        end)
    query:Execute()
end

-- Load quest progress
function PLUGIN:LoadQuests(character, callback)
    local query = mysql:Select("quest_progress")
        query:Where("character_id", character:GetID())
        query:Callback(function(result)
            local quests = {}

            if istable(result) then
                for _, row in ipairs(result) do
                    quests[row.quest_id] = {
                        progress = tonumber(row.progress),
                        completed = row.completed == 1
                    }
                end
            end

            callback(quests)
        end)
    query:Execute()
end

-- Load when character spawns
function PLUGIN:PlayerLoadedCharacter(client, character)
    self:LoadQuests(character, function(quests)
        character.quests = quests
        client:ChatPrint("Loaded " .. table.Count(quests) .. " quests")
    end)
end

-- Save when character is saved
function PLUGIN:PreCharacterSave(character)
    if character.quests then
        for questID, data in pairs(character.quests) do
            self:SaveQuest(character, questID, data.progress, data.completed)
        end
    end
end
```

## Best Practices

### ✅ DO

- Use Character, Inventory, and Item methods instead of raw queries
- Use the query builder (mysql:Select, Insert, Update, Delete)
- Always use callbacks for SELECT queries
- Always use `query:Where()` for UPDATE and DELETE to prevent mass updates
- Use `util.TableToJSON()` for complex data storage
- Create custom tables for plugin-specific data
- Use transactions for multi-query operations (when supported)
- Validate data before inserting

### ❌ DON'T

- Don't write raw SQL strings (vulnerable to SQL injection)
- Don't modify core Helix tables directly (use Character/Inventory/Item APIs)
- Don't forget to call `query:Execute()`
- Don't forget callbacks on SELECT queries
- Don't execute queries on every frame or in tight loops
- Don't store binary data directly (use base64 encoding)
- Don't assume queries succeed (always check results in callbacks)
- Don't call `ix.db.Connect()` manually

## Common Patterns

### Pattern 1: Upsert (Update or Insert)

```lua
-- Check if record exists, update or insert accordingly
function UpdateOrInsert(characterID, data)
    local query = mysql:Select("mytable")
        query:Where("character_id", characterID)
        query:Callback(function(result)
            if istable(result) and #result > 0 then
                -- Record exists, update it
                local updateQuery = mysql:Update("mytable")
                    updateQuery:Update("data", util.TableToJSON(data))
                    updateQuery:Where("id", result[1].id)
                updateQuery:Execute()
            else
                -- Record doesn't exist, insert it
                local insertQuery = mysql:Insert("mytable")
                    insertQuery:Insert("character_id", characterID)
                    insertQuery:Insert("data", util.TableToJSON(data))
                insertQuery:Execute()
            end
        end)
    query:Execute()
end
```

### Pattern 2: Cascading Delete

```lua
-- Delete related records before deleting parent
function DeleteWithRelations(characterID)
    -- Delete child records first
    local query = mysql:Delete("child_table")
        query:Where("character_id", characterID)
        query:Callback(function()
            -- Then delete parent
            local parentQuery = mysql:Delete("parent_table")
                parentQuery:Where("id", characterID)
            parentQuery:Execute()
        end)
    query:Execute()
end
```

### Pattern 3: Batch Loading

```lua
-- Load multiple records efficiently
function LoadPlayerData(steamID, callback)
    local data = {}
    local completed = 0
    local total = 2

    local function CheckComplete()
        completed = completed + 1
        if completed >= total then
            callback(data)
        end
    end

    -- Load characters
    local charQuery = mysql:Select("ix_characters")
        charQuery:Where("steamid", steamID)
        charQuery:Callback(function(result)
            data.characters = result or {}
            CheckComplete()
        end)
    charQuery:Execute()

    -- Load player data
    local playerQuery = mysql:Select("ix_players")
        playerQuery:Where("steamid", steamID)
        playerQuery:Callback(function(result)
            data.player = result and result[1] or nil
            CheckComplete()
        end)
    playerQuery:Execute()
end
```

## Common Issues

### Issue: Query not executing

**Cause**: Forgot to call `query:Execute()`
**Fix**: Always call Execute after building query

```lua
-- WRONG
local query = mysql:Select("ix_characters")
    query:Where("id", 1)
-- Query never runs!

-- CORRECT
local query = mysql:Select("ix_characters")
    query:Where("id", 1)
query:Execute()  -- Must call Execute!
```

### Issue: Results always nil in callback

**Cause**: Database not connected yet, or query syntax error
**Fix**: Wait for database connection, check console for errors

```lua
-- WRONG: Called immediately on server start
function PLUGIN:Initialize()
    local query = mysql:Select("mytable")  -- DB not connected yet!
    query:Execute()
end

-- CORRECT: Wait for database
function PLUGIN:InitPostEntity()
    timer.Simple(1, function()
        local query = mysql:Select("mytable")
        query:Execute()
    end)
end
```

### Issue: WHERE clause not working

**Cause**: Forgot to add WHERE before Execute
**Fix**: Always add WHERE conditions before Execute

```lua
-- WRONG
local query = mysql:Update("ix_characters")
    query:Update("money", 500)
query:Execute()  -- Updates ALL characters!

-- CORRECT
local query = mysql:Update("ix_characters")
    query:Update("money", 500)
    query:Where("id", characterID)  -- Specify which record
query:Execute()
```

### Issue: Cannot modify core tables

**Cause**: Trying to manually ALTER Helix tables
**Fix**: Use ix.char.RegisterVar or ix.db.AddToSchema

```lua
-- WRONG
RunConsoleCommand("sql_exec", "ALTER TABLE ix_characters ADD COLUMN mycol TEXT")

-- CORRECT
ix.char.RegisterVar("myvar", {
    field = "mycol",
    fieldType = ix.type.text,
    default = "",
    isLocal = false
})
```

## When to Use Database vs ix.data

### Use Database (ix.db / Character methods) for:
- ✅ Character data
- ✅ Player accounts
- ✅ Inventory systems
- ✅ Large datasets (hundreds/thousands of records)
- ✅ Queryable data (search, filter, sort)
- ✅ Relational data with foreign keys

### Use File Storage (ix.data) for:
- ✅ Plugin configuration
- ✅ Server settings
- ✅ Small key-value data
- ✅ Map-specific data
- ✅ Schema metadata

## Database Maintenance

### Wipe Database (Console Only)

**Reference**: `gamemode/core/libs/sv_database.lua:167-188`

```
> ix_wipedb
[Helix] WIPING THE DATABASE WILL PERMANENTLY REMOVE ALL DATA.
[Helix] RUN 'ix_wipedb' AGAIN WITHIN 3 SECONDS TO CONFIRM.

> ix_wipedb
[Helix] DATABASE WIPE IN PROGRESS...
[Helix] DATABASE WIPE COMPLETED!
```

**⚠️ WARNING**: This permanently deletes ALL characters, inventories, items, and players.

## MySQL Setup Example

```sql
-- Create database
CREATE DATABASE helix_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user
CREATE USER 'helix_user'@'localhost' IDENTIFIED BY 'secure_password';

-- Grant permissions
GRANT ALL PRIVILEGES ON helix_db.* TO 'helix_user'@'localhost';
FLUSH PRIVILEGES;
```

Then configure in `garrysmod/settings/helix/server_config.txt`:
```lua
config.database = {
    adapter = "mysqloo",
    hostname = "localhost",
    username = "helix_user",
    password = "secure_password",
    database = "helix_db",
    port = 3306
}
```

## See Also

- [Character System](../systems/character.md) - Use Character methods instead of raw queries
- [Inventory System](../systems/inventory.md) - Inventory persistence
- [Items System](../systems/items.md) - Item persistence
- [Data Library](data.md) - For simple file-based storage
- [Storage Library](storage.md) - For entity-based storage
- Source: `gamemode/core/libs/sv_database.lua`
