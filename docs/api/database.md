# Database API (ix.db)

> **Reference**: `gamemode/core/libs/sv_database.lua`

The database API provides MySQL/SQLite database operations with query building, prepared statements, and connection management.

## Core Concepts

The database system:
- Supports MySQL and SQLite
- Uses prepared statements (secure)
- Provides query builder
- Auto-connects on startup
- Character/inventory persistence

## Configuration

Database configured in `helix.yml`:
```yaml
database:
  adapter: mysqloo  # or sqlite
  hostname: localhost
  username: root
  password: ""
  database: helix
  port: 3306
```

## Functions

### ix.db.Query

**Realm**: Server

```lua
ix.db.Query(query, callback, ...)
```

Executes SQL query with prepared statement support.

**Example**:
```lua
-- Simple query
ix.db.Query("SELECT * FROM ix_characters WHERE _steamID = ?", function(data)
    PrintTable(data)
end, client:SteamID64())

-- Insert
ix.db.Query("INSERT INTO my_table (name, value) VALUES (?, ?)", nil, "test", 123)

-- Update
ix.db.Query("UPDATE ix_characters SET _money = ? WHERE _id = ?", nil, newMoney, charID)
```

### ix.db.Select

Query builder for SELECT statements.

**Example**:
```lua
ix.db.Select("ix_characters")
    :Select("_name")
    :Select("_money")
    :Where("_faction", FACTION_POLICE)
    :Limit(10)
    :Callback(function(data)
        for _, row in ipairs(data) do
            print(row._name, row._money)
        end
    end)
    :Execute()
```

### ix.db.Insert

Query builder for INSERT statements.

**Example**:
```lua
ix.db.Insert("my_table")
    :Insert("name", "John")
    :Insert("value", 100)
    :Callback(function(result, lastID)
        print("Inserted row ID:", lastID)
    end)
    :Execute()
```

### ix.db.Update

Query builder for UPDATE statements.

**Example**:
```lua
ix.db.Update("ix_characters")
    :Update("_money", 5000)
    :Where("_id", characterID)
    :Execute()
```

## Best Practices

### ✅ DO
- Use prepared statements (?)
- Use query builders for safety
- Handle callbacks properly
- Check for nil data

### ❌ DON'T
- Don't use string concatenation for queries
- Don't forget to escape user input
- Don't block main thread with queries
- Don't query in loops (use batch operations)

## See Also

- [Character API](character.md) - Character persistence
- [Data API](data.md) - File-based storage
- Source: `gamemode/core/libs/sv_database.lua`
