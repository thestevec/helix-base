# Helix Database & Data Systems - AI Assistant Guide

> Context-specific guidance for working with databases and data persistence in the Helix framework.

## 📚 Documentation Knowledgebase

**Primary Reference**: [docs/llms.txt](../../llms.txt) - Complete framework documentation index

## Data Systems Overview

### Core Documentation
1. **[Database System](../../libraries/database.md)** - MySQL/SQLite abstraction layer
2. **[Database Operations Guide](../../guides/database.md)** - Practical examples
3. **[Storage System](../../libraries/storage.md)** - Persistent data with JSON/YAML
4. **[Data System](../../libraries/data.md)** - Key-value storage
5. **[Database Configuration](../../config/database.md)** - Setup and configuration

### API References
- **[ix.db](../../api/database.md)** - Database query functions
- **[ix.data](../../api/data.md)** - Data storage/retrieval
- **[ix.storage](../../api/storage.md)** - Data persistence functions

## Database System (ix.db)

### Configuration

**Essential Reading**: [Database Configuration](../../config/database.md)

#### MySQL Setup
```lua
-- garrysmod/cfg/server.cfg or startup parameters
ix_mysql "1"
ix_mysql_address "localhost"
ix_mysql_username "username"
ix_mysql_password "password"
ix_mysql_database "database_name"
ix_mysql_port "3306"
```

#### SQLite (Default)
```lua
-- No configuration needed - works out of the box
-- Database stored in: garrysmod/sv.db
```

### Database Queries

**API Reference**: [ix.db](../../api/database.md)

#### Select Query
```lua
local query = ix.db.Select("table_name")
    :Select("column1")
    :Select("column2")
    :Where("id", characterID)
    :Callback(function(result)
        if result and #result > 0 then
            local row = result[1]
            print(row.column1, row.column2)
        end
    end)
    :Execute()
```

#### Insert Query
```lua
local query = ix.db.Insert("table_name")
    :Insert("column1", value1)
    :Insert("column2", value2)
    :Callback(function(result, lastID)
        print("Inserted with ID:", lastID)
    end)
    :Execute()
```

#### Update Query
```lua
local query = ix.db.Update("table_name")
    :Update("column1", newValue)
    :Where("id", recordID)
    :Callback(function(result)
        print("Updated successfully")
    end)
    :Execute()
```

#### Delete Query
```lua
local query = ix.db.Delete("table_name")
    :Where("id", recordID)
    :Callback(function(result)
        print("Deleted successfully")
    end)
    :Execute()
```

### Advanced Queries

#### Multiple Conditions
```lua
local query = ix.db.Select("ix_characters")
    :Select("id")
    :Select("name")
    :Where("faction", factionID)
    :WhereGT("money", 1000)  -- Greater than
    :WhereLT("playtime", 3600)  -- Less than
    :OrderByDesc("money")
    :Limit(10)
    :Callback(function(result)
        -- Process results
    end)
    :Execute()
```

#### Joins
```lua
local query = ix.db.Select("ix_characters")
    :Select("ix_characters.name")
    :Select("ix_inventories.id")
    :InnerJoin("ix_inventories", "ix_characters.id = ix_inventories.character_id")
    :Where("ix_characters.faction", factionID)
    :Callback(function(result)
        -- Process joined results
    end)
    :Execute()
```

#### Custom SQL
```lua
local query = ix.db.Query("SELECT * FROM table_name WHERE custom_condition")
query.onSuccess = function(query, result)
    -- Process results
end
query.onError = function(query, error)
    ErrorNoHalt("[DB Error] " .. error .. "\n")
end
query:Start()
```

### Creating Custom Tables

```lua
-- In sv_schema.lua or plugin's sv_plugin.lua
function SCHEMA:DatabaseConnected()
    -- Create custom table
    local query = ix.db.Query([[
        CREATE TABLE IF NOT EXISTS custom_table (
            id INT AUTO_INCREMENT PRIMARY KEY,
            character_id INT NOT NULL,
            data_field VARCHAR(255),
            number_field INT DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX(character_id)
        )
    ]])
    query:Start()
end
```

## Storage System (ix.storage)

**Essential Reading**: [Storage System](../../libraries/storage.md)

**API Reference**: [ix.storage](../../api/storage.md)

### Saving Data
```lua
-- Save table to JSON file
ix.storage.Save("filename", {
    key1 = "value1",
    key2 = {nested = "data"},
    key3 = 123
})
```

### Loading Data
```lua
-- Load data from file
local data = ix.storage.Load("filename")

if data then
    print(data.key1)  -- "value1"
end
```

### Format Options
```lua
-- JSON (default)
ix.storage.Save("data", table, "json")

-- YAML
ix.storage.Save("data", table, "yaml")

-- Binary
ix.storage.Save("data", table, "dat")
```

## Data System (ix.data)

**Essential Reading**: [Data System](../../libraries/data.md)

**API Reference**: [ix.data](../../api/data.md)

### Setting Data
```lua
-- Set key-value pair
ix.data.Set("key", "value", shared, global)

-- shared: true = shared between realms, false = realm-specific
-- global: true = global across all maps, false = map-specific
```

### Getting Data
```lua
local value = ix.data.Get("key", default, shared, global)
```

### Deleting Data
```lua
ix.data.Delete("key", shared, global)
```

### Common Use Cases

#### Player Preferences
```lua
-- Save player preference
ix.data.Set("player_" .. client:SteamID64() .. "_setting", value, false, true)

-- Load player preference
local setting = ix.data.Get("player_" .. client:SteamID64() .. "_setting", defaultValue, false, true)
```

#### Plugin Data
```lua
-- Save plugin data
ix.data.Set("myplugin_data", {
    setting1 = value1,
    setting2 = value2
}, true, true)
```

## Character Data

**Essential Reading**: [Character System](../../systems/character.md)

**API Reference**: [ix.char](../../api/character.md)

### Character Variables (Network Synced)
```lua
-- Set character variable (synced to client)
character:SetData("key", value)

-- Get character variable
local value = character:GetData("key", default)
```

### Local Character Data (Server Only)
```lua
-- Set local data (not synced)
character:SetLocalVar("key", value)

-- Get local data
local value = character:GetLocalVar("key", default)
```

### Persistent Character Data
Character variables are automatically saved to the database.

## Inventory Data

**Essential Reading**: [Inventory System](../../systems/inventory.md)

**API Reference**: [ix.inventory](../../api/inventory.md)

### Creating Inventory
```lua
ix.inventory.Create(width, height, inventoryID, function(inventory)
    -- Inventory created callback
    character:SetInventory(inventory)
end)
```

### Adding Items
```lua
local inventory = character:GetInventory()

inventory:Add("item_uniqueid", quantity, data)
-- Or
inventory:Add("item_uniqueid", quantity, data, x, y)  -- Specific position
```

### Querying Items
```lua
local items = inventory:GetItems()

for k, item in pairs(items) do
    print(item:GetName())
end

-- Find specific item
local item = inventory:GetItemByID(itemID)
```

## Best Practices

### Database Operations

#### Use Prepared Statements
```lua
-- Good: Uses prepared statement (safe from SQL injection)
ix.db.Select("table")
    :Where("column", userInput)
    :Execute()

-- Bad: Direct SQL with user input
ix.db.Query("SELECT * FROM table WHERE column = '" .. userInput .. "'")
```

#### Avoid Queries in Loops
```lua
-- Bad: Query per iteration
for k, v in ipairs(characters) do
    ix.db.Select("table"):Where("id", v.id):Execute()
end

-- Good: Single query with IN clause
local ids = {}
for k, v in ipairs(characters) do
    ids[#ids + 1] = v.id
end
ix.db.Select("table"):WhereIn("id", ids):Execute()
```

#### Use Callbacks Properly
```lua
-- Queries are asynchronous - use callbacks
local query = ix.db.Select("table")
    :Where("id", id)
    :Callback(function(result)
        -- Data available here
        ProcessData(result)
    end)
    :Execute()

-- Data NOT available here yet!
```

#### Error Handling
```lua
local query = ix.db.Select("table")
query.onSuccess = function(q, result)
    -- Handle success
end
query.onError = function(q, error)
    ErrorNoHalt("[Database Error] " .. error .. "\n")
    -- Handle error gracefully
end
query:Start()
```

### Storage Best Practices

#### Save Frequency
```lua
-- Bad: Save on every change
function UpdateData(key, value)
    data[key] = value
    ix.storage.Save("data", data)  -- Disk write every change!
end

-- Good: Batch saves
local saveTimer = nil
function UpdateData(key, value)
    data[key] = value

    timer.Remove("SaveData")
    timer.Create("SaveData", 30, 1, function()
        ix.storage.Save("data", data)
    end)
end
```

#### Data Validation
```lua
function LoadData()
    local data = ix.storage.Load("config")

    -- Validate loaded data
    if !data or !istable(data) then
        data = GetDefaultConfig()
    end

    -- Validate fields
    data.version = data.version or 1
    data.settings = data.settings or {}

    return data
end
```

### Character Data Best Practices

#### Use Appropriate Storage
```lua
-- Network synced (visible to client)
character:SetData("visibleStat", value)

-- Server only (not synced)
character:SetLocalVar("internalState", value)

-- Temporary data (not saved)
character.tempVar = value
```

#### Data Types
```lua
-- Supported types for SetData:
character:SetData("string", "text")
character:SetData("number", 123)
character:SetData("boolean", true)
character:SetData("table", {key = "value"})  -- Serialized automatically

-- Not supported (will cause issues):
character:SetData("function", function() end)  -- Don't do this
character:SetData("entity", entity)  -- Don't do this
```

## Common Patterns

### Caching Database Results
```lua
local cache = {}
local cacheTime = {}

function GetCachedData(key, callback)
    if cache[key] and (cacheTime[key] or 0) > CurTime() - 300 then
        -- Cache valid for 5 minutes
        callback(cache[key])
        return
    end

    -- Fetch from database
    ix.db.Select("table")
        :Where("key", key)
        :Callback(function(result)
            cache[key] = result
            cacheTime[key] = CurTime()
            callback(result)
        end)
        :Execute()
end
```

### Leaderboard Query
```lua
function GetTopPlayers(limit, callback)
    ix.db.Select("ix_characters")
        :Select("name")
        :Select("money")
        :OrderByDesc("money")
        :Limit(limit or 10)
        :Callback(function(result)
            callback(result)
        end)
        :Execute()
end
```

### Batch Operations
```lua
function SaveMultipleCharacters(characters)
    local queries = {}

    for k, char in ipairs(characters) do
        local query = ix.db.Update("ix_characters")
            :Update("money", char.money)
            :Where("id", char.id)

        table.insert(queries, query)
    end

    -- Execute all queries
    for k, query in ipairs(queries) do
        query:Execute()
    end
end
```

## Troubleshooting

**MySQL not connecting?**
- Check credentials in configuration
- Verify MySQL server is running
- Check firewall/port settings
- Look for connection errors in console

**Data not saving?**
- Check if DatabaseConnected hook fired
- Verify table exists
- Check for query errors in console
- Ensure callback is defined

**Data not loading?**
- Verify file exists (garrysmod/data/ix/)
- Check for JSON parsing errors
- Validate data structure

**Performance issues?**
- Reduce query frequency
- Add database indexes
- Cache frequently-accessed data
- Use batch operations

## Migration from SQLite to MySQL

```lua
-- Export from SQLite, import to MySQL
-- Use mysqldump for schema
-- Use data migration scripts for records
-- Test thoroughly before production switch
```

---

**Quick Links**:
- [Back to Main Guide](../../../CLAUDE.md)
- [Complete Documentation Index](../../llms.txt)
- [Database Guide](../../guides/database.md)
- [Database Configuration](../../config/database.md)
