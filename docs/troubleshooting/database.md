# Database Troubleshooting

> **Reference**: `gamemode/core/libs/sv_database.lua`, `gamemode/core/libs/thirdparty/sv_mysql.lua`

Comprehensive guide for diagnosing and fixing database connection and query issues in Helix.

## ⚠️ Important: Use Helix Database Functions

**Always use Helix's built-in `ix.db` and `mysql` wrappers** rather than direct SQL modules. The framework provides:
- Automatic connection management
- Query builder interface
- SQLite fallback support
- Schema migration system
- Error handling and logging

**Reference**: `gamemode/core/libs/sv_database.lua:18-30`

---

## Quick Diagnostic Commands

Run these in server console to diagnose database issues:

```lua
-- Check database adapter
lua_run print(ix.db.config.adapter)
-- Should print: "mysqloo" or "sqlite"

-- Check connection status
lua_run print(mysql:IsConnected())
-- Should print: true

-- Test simple query
lua_run ix.db.Query("SELECT 1 as test", function(result) PrintTable(result) end)
-- Should print table with result

-- Check database tables
lua_run ix.db.Query("SHOW TABLES", function(r) PrintTable(r) end)
-- Should show: ix_characters, ix_inventories, ix_items, ix_players, ix_schema
```

---

## Database Configuration

### Configuration File Location

Helix reads database config from:
```
garrysmod/data/helix/config.txt
```

### SQLite Configuration (Default)

**No configuration needed** - SQLite works out of the box:

```json
{
    "database": {
        "adapter": "sqlite"
    }
}
```

**Database file location**:
```
garrysmod/sv.db
```

### MySQL Configuration

**Reference**: `gamemode/core/libs/sv_database.lua:18-30`

**Complete MySQL config**:

```json
{
    "database": {
        "adapter": "mysqloo",
        "hostname": "127.0.0.1",
        "username": "garrysmod",
        "password": "your_password_here",
        "database": "helix",
        "port": 3306
    }
}
```

**⚠️ Common mistakes**:

❌ **WRONG**:
```json
{
    "database": {
        "adapter": "mysql",  // Wrong! Use "mysqloo"
        "host": "localhost"  // Wrong! Use "hostname"
    }
}
```

✅ **CORRECT**:
```json
{
    "database": {
        "adapter": "mysqloo",
        "hostname": "127.0.0.1"  // or "localhost"
    }
}
```

---

## Issue 1: Database Won't Connect

### Symptoms
- Server starts but no database messages in console
- Players can't create characters
- Console shows: "Database Type: sqlite" when you expect MySQL
- No characters or items loading

### Solution 1: Check MySQLOO Module Installed

**MySQLOO binary must be installed** for MySQL connections.

**Check if installed**:
```
garrysmod/lua/bin/gmsv_mysqloo_[platform].dll
```

**Platform files**:
- Windows: `gmsv_mysqloo_win32.dll` or `gmsv_mysqloo_win64.dll`
- Linux: `gmsv_mysqloo_linux.dll` or `gmsv_mysqloo_linux64.dll`

**If missing**:
1. Download MySQLOO from: https://github.com/FredyH/MySQLOO/releases
2. Extract to `garrysmod/lua/bin/`
3. Restart server

### Solution 2: Verify Configuration File Exists

**Check file exists**:
```bash
# Linux
ls -la garrysmod/data/helix/config.txt

# Windows
dir garrysmod\data\helix\config.txt
```

**If missing**, create it:
```bash
# Linux
mkdir -p garrysmod/data/helix
nano garrysmod/data/helix/config.txt

# Windows
mkdir garrysmod\data\helix
notepad garrysmod\data\helix\config.txt
```

**Add minimal config**:
```json
{
    "database": {
        "adapter": "sqlite"
    }
}
```

### Solution 3: Check JSON Syntax

**Common JSON errors**:

❌ **WRONG** - Trailing comma:
```json
{
    "database": {
        "adapter": "mysqloo",
    }
}
```

❌ **WRONG** - Missing quotes:
```json
{
    database: {
        adapter: mysqloo
    }
}
```

✅ **CORRECT**:
```json
{
    "database": {
        "adapter": "mysqloo",
        "hostname": "127.0.0.1"
    }
}
```

**Validate JSON** at https://jsonlint.com/ before saving

### Solution 4: MySQL Server Not Running

**Test MySQL is running**:

```bash
# Linux
sudo systemctl status mysql
# or
sudo systemctl status mariadb

# Windows
# Check Services for "MySQL" or "MariaDB"
```

**Start MySQL**:
```bash
# Linux
sudo systemctl start mysql

# Windows
net start MySQL
```

**Test connection from command line**:
```bash
mysql -h 127.0.0.1 -u garrysmod -p helix
# Enter password when prompted
```

If this fails, MySQL credentials are wrong or user doesn't exist.

### Solution 5: Create MySQL Database and User

**MySQL must have database and user created first**.

**Login to MySQL as root**:
```bash
mysql -u root -p
```

**Create database and user**:
```sql
-- Create database
CREATE DATABASE helix CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user
CREATE USER 'garrysmod'@'localhost' IDENTIFIED BY 'your_password_here';

-- Grant permissions
GRANT ALL PRIVILEGES ON helix.* TO 'garrysmod'@'localhost';

-- Apply changes
FLUSH PRIVILEGES;

-- Exit
EXIT;
```

**Test login**:
```bash
mysql -u garrysmod -p helix
# Should connect successfully
```

### Solution 6: Firewall Blocking Connection

If MySQL is on remote server:

**Check port 3306 is open**:
```bash
# Linux
sudo ufw allow 3306

# Test connection
telnet your-db-server.com 3306
# Should connect
```

**MySQL bind address** - Edit `/etc/mysql/mysql.conf.d/mysqld.cnf`:
```ini
# Change from:
bind-address = 127.0.0.1

# To:
bind-address = 0.0.0.0
```

**Restart MySQL**:
```bash
sudo systemctl restart mysql
```

---

## Issue 2: "Database Connected" But Data Not Saving

### Symptoms
- Console shows "Database Type: mysqloo" or "sqlite"
- Characters can be created
- After server restart, all data lost
- Items disappear
- Money resets

### Solution 1: Check DatabaseConnected Hook Running

**Reference**: `gamemode/core/hooks/sv_hooks.lua:956-968`

**Verify hook is called**:
```lua
-- Add to sv_plugin.lua temporarily
function PLUGIN:DatabaseConnected()
    print("DATABASE CONNECTED - TABLES LOADING")
end
```

If you don't see this message, database didn't connect.

### Solution 2: Tables Not Created

**Check tables exist**:

```lua
-- Server console
lua_run ix.db.Query("SHOW TABLES", function(result) PrintTable(result) end)
```

**Should show**:
- ix_characters
- ix_inventories
- ix_items
- ix_players
- ix_schema

**If missing**, tables didn't create. Check for errors during `ix.db.LoadTables()`.

**Manual table creation** (if needed):

```lua
-- Server console - Forces table recreation
lua_run ix.db.LoadTables()
```

### Solution 3: Using Wrong Query Function

**❌ WRONG** - Don't use `sql.` library with MySQL:
```lua
-- This only works with SQLite, data won't save to MySQL!
sql.Query("INSERT INTO characters ...")
```

**✅ CORRECT** - Use `ix.db.Query()` or `mysql:` functions:
```lua
-- Works with both SQLite and MySQL
ix.db.Query("INSERT INTO ix_characters ...")

-- Or use query builder
local query = mysql:Insert("ix_characters")
    query:Insert("name", "John Doe")
    query:Insert("faction", 1)
    query:Callback(function(result, status, lastID)
        print("Inserted character ID:", lastID)
    end)
query:Execute()
```

**Reference**: `gamemode/core/libs/sv_database.lua:2-14`

### Solution 4: Callback Not Firing

**Queries are asynchronous** - data isn't available immediately.

**❌ WRONG**:
```lua
local characterName = nil

ix.db.Query("SELECT name FROM ix_characters WHERE id = 1", function(result)
    characterName = result[1].name
end)

print(characterName)  -- Still nil! Query hasn't completed yet
```

**✅ CORRECT**:
```lua
ix.db.Query("SELECT name FROM ix_characters WHERE id = 1", function(result)
    if result and result[1] then
        local characterName = result[1].name
        print(characterName)  -- Correct! Inside callback
    end
end)
```

### Solution 5: Transaction Rollback

**MySQL transactions** can rollback on error.

**Check for SQL errors**:
```lua
local query = mysql:Select("ix_characters")
    query:Callback(function(result, status, lastID)
        if status == false then
            print("Query failed!")
        else
            print("Query succeeded")
        end
    end)
query:Execute()
```

---

## Issue 3: SQL Syntax Errors

### Common Syntax Mistakes

#### Missing Quotes Around Strings

**❌ WRONG**:
```lua
ix.db.Query("INSERT INTO ix_players (steamid, steam_name) VALUES (STEAM_0:1:12345, John Doe)")
-- ERROR: Unknown column 'John' in field list
```

**✅ CORRECT**:
```lua
ix.db.Query("INSERT INTO ix_players (steamid, steam_name) VALUES ('STEAM_0:1:12345', 'John Doe')")
-- Or use query builder which handles escaping:
local query = mysql:Insert("ix_players")
    query:Insert("steamid", "STEAM_0:1:12345")
    query:Insert("steam_name", "John Doe")
query:Execute()
```

#### SQL Injection Vulnerability

**❌ DANGEROUS**:
```lua
local name = client:GetName()
ix.db.Query("SELECT * FROM ix_characters WHERE name = '" .. name .. "'")
-- If name contains quotes, breaks query or allows injection!
```

**✅ CORRECT**:
```lua
local name = mysql:Escape(client:GetName())
ix.db.Query("SELECT * FROM ix_characters WHERE name = '" .. name .. "'")

-- Better: Use query builder
local query = mysql:Select("ix_characters")
    query:Where("name", client:GetName())  -- Automatically escaped
    query:Callback(function(result) end)
query:Execute()
```

**Reference**: `gamemode/core/libs/thirdparty/sv_mysql.lua:82-84`

#### Reserved Keywords as Column Names

**If using MySQL reserved words** (like "key", "order", "table"), use backticks:

**❌ WRONG**:
```lua
ix.db.Query("SELECT order FROM ix_items WHERE id = 1")
-- ERROR: Syntax error near 'order'
```

**✅ CORRECT**:
```lua
ix.db.Query("SELECT `order` FROM ix_items WHERE id = 1")
```

**Query builder handles this automatically**:
```lua
local query = mysql:Select("ix_items")
    query:Select("order")  -- Automatically adds backticks
query:Execute()
```

---

## Issue 4: Character Data Corruption

### Symptoms
- Characters exist but data is null/empty
- JSON decode errors in console
- Character attributes reset to 0
- Inventory items disappear

### Solution 1: Invalid JSON in Data Column

**Characters store data as JSON** in `data` column.

**Check for corruption**:
```lua
ix.db.Query("SELECT id, data FROM ix_characters", function(result)
    for k, v in pairs(result) do
        local success, decoded = pcall(util.JSONToTable, v.data)
        if not success then
            print("Character " .. v.id .. " has corrupt data!")
        end
    end
end)
```

**Fix corrupt data**:
```lua
-- Reset single character's data
ix.db.Query("UPDATE ix_characters SET data = '{}' WHERE id = 1")

-- Or manually set valid JSON
ix.db.Query("UPDATE ix_characters SET data = '{\"hunger\":100}' WHERE id = 1")
```

### Solution 2: Schema Mismatch

**Schema tracks custom fields** added to characters.

**Check schema**:
```lua
lua_run PrintTable(ix.db.schema)
```

**If schema is wrong**, it may not query correct columns.

**Reset schema** (advanced - backup first!):
```lua
-- Backup database first!
lua_run ix.db.Query("DELETE FROM ix_schema WHERE table = 'ix_characters'")
-- Restart server to rebuild schema
```

---

## Issue 5: Performance Issues

### Slow Queries

**Identify slow queries** by adding logging:

```lua
-- Add to sv_plugin.lua
local oldQuery = ix.db.Query
ix.db.Query = function(query, callback, ...)
    local startTime = SysTime()

    return oldQuery(query, function(...)
        local duration = (SysTime() - startTime) * 1000
        if duration > 100 then  -- Log queries over 100ms
            print(string.format("[SLOW QUERY] %.2fms: %s", duration, query))
        end

        if callback then
            return callback(...)
        end
    end, ...)
end
```

### Common Performance Problems

#### No Indexes on Queried Columns

**Add index to frequently queried columns**:

```sql
-- If you frequently query by steamid
ALTER TABLE ix_players ADD INDEX idx_steamid (steamid);

-- If you query items by inventory_id often
ALTER TABLE ix_items ADD INDEX idx_inventory (inventory_id);

-- If you query characters by faction
ALTER TABLE ix_characters ADD INDEX idx_faction (faction);
```

#### Querying in Loops

**❌ WRONG**:
```lua
for _, client in ipairs(player.GetAll()) do
    local steamID = client:SteamID()

    -- BAD! Queries in loop
    ix.db.Query("SELECT * FROM ix_players WHERE steamid = '" .. steamID .. "'", function(result)
        -- Process result
    end)
end
```

**✅ CORRECT**:
```lua
-- Batch query using WHERE IN
local steamIDs = {}
for _, client in ipairs(player.GetAll()) do
    table.insert(steamIDs, mysql:Escape(client:SteamID()))
end

local query = mysql:Select("ix_players")
    query:WhereIn("steamid", steamIDs)  -- Single query for all players
    query:Callback(function(result)
        -- Process all results
    end)
query:Execute()
```

#### Loading Entire Tables

**❌ WRONG**:
```lua
-- Loads ALL items from database!
ix.db.Query("SELECT * FROM ix_items", function(result)
    for k, v in pairs(result) do
        if v.unique_id == "item_medkit" then
            -- Found it
        end
    end
end)
```

**✅ CORRECT**:
```lua
-- Only load what you need
local query = mysql:Select("ix_items")
    query:Where("unique_id", "item_medkit")
    query:Callback(function(result)
        -- Much faster!
    end)
query:Execute()
```

---

## Issue 6: MySQL Connection Lost

### Symptoms
- Server runs fine, then suddenly stops saving
- Console: "Lost connection to MySQL server"
- Players report data not saving mid-session

### Solutions

#### Increase MySQL Timeout

**Edit MySQL config** (`/etc/mysql/mysql.conf.d/mysqld.cnf`):
```ini
[mysqld]
wait_timeout = 28800
interactive_timeout = 28800
max_allowed_packet = 64M
```

**Restart MySQL**:
```bash
sudo systemctl restart mysql
```

#### Keep-Alive Query

**Add to plugin to prevent timeout**:
```lua
-- sv_plugin.lua
if SERVER then
    timer.Create("MySQLKeepAlive", 300, 0, function()
        if mysql:IsConnected() then
            ix.db.Query("SELECT 1")  -- Ping query
        end
    end)
end
```

**Reference**: Helix already has `ixDatabaseThink` timer (`gamemode/core/hooks/sv_hooks.lua:963`)

---

## Issue 7: Database Wipe Not Working

### Using ix_wipedb Command

**Reference**: `gamemode/core/libs/sv_database.lua:167-188`

**⚠️ WARNING**: This **permanently deletes all data**!

**How to wipe**:
1. Run `ix_wipedb` in **server console** (not client)
2. Within 3 seconds, run `ix_wipedb` again to confirm
3. Server will wipe database and restart

**❌ Common mistakes**:
- Running from client console (doesn't work)
- Waiting too long between commands
- Canceling server restart

**Manual wipe** (if command fails):

**SQLite**:
```bash
# Stop server first
rm garrysmod/sv.db
# Start server - new database created
```

**MySQL**:
```sql
-- Login to MySQL
mysql -u garrysmod -p helix

-- Drop and recreate
DROP DATABASE helix;
CREATE DATABASE helix CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## Migration: SQLite to MySQL

### Step-by-Step Migration

**1. Backup SQLite database**:
```bash
cp garrysmod/sv.db garrysmod/sv.db.backup
```

**2. Create MySQL database** (see Solution 5 in Issue 1)

**3. Update config.txt**:
```json
{
    "database": {
        "adapter": "mysqloo",
        "hostname": "127.0.0.1",
        "username": "garrysmod",
        "password": "your_password",
        "database": "helix",
        "port": 3306
    }
}
```

**4. Export SQLite data**:
```bash
sqlite3 sv.db .dump > helix_dump.sql
```

**5. Convert SQLite syntax to MySQL**:
```bash
# Remove SQLite-specific commands
sed -i '/BEGIN TRANSACTION/d' helix_dump.sql
sed -i '/COMMIT/d' helix_dump.sql
sed -i '/sqlite_sequence/d' helix_dump.sql

# Fix quotes
sed -i 's/`/"/g' helix_dump.sql
```

**6. Import to MySQL**:
```bash
mysql -u garrysmod -p helix < helix_dump.sql
```

**7. Restart server** with new config

**Alternative**: Use third-party tools like SQLite to MySQL converter

---

## Debugging Tools

### Query Logging

**Enable all query logging**:
```lua
-- sv_plugin.lua
local oldExecute = QUERY_CLASS.Execute
function QUERY_CLASS:Execute()
    print("[QUERY]", self:_Build())
    return oldExecute(self)
end
```

### Database Integrity Check

**Check all tables exist and have data**:
```lua
-- Server console
lua_run hook.Add("DatabaseConnected", "CheckTables", function()
    local tables = {"ix_characters", "ix_inventories", "ix_items", "ix_players", "ix_schema"}

    for _, tbl in ipairs(tables) do
        ix.db.Query("SELECT COUNT(*) as count FROM " .. tbl, function(result)
            print(tbl .. " has " .. result[1].count .. " rows")
        end)
    end
end)
```

### Connection Test Script

**Test MySQL connection**:
```lua
-- Run in server console
lua_run_sv require("mysqloo"); local db = mysqloo.connect("127.0.0.1", "garrysmod", "password", "helix", 3306); db.onConnected = function() print("MySQL connection successful!") end; db.onConnectionFailed = function(db, err) print("MySQL connection failed:", err) end; db:connect()
```

---

## See Also

- [Common Issues](common-issues.md) - General troubleshooting
- [Performance Optimization](performance.md) - Query optimization
- [Storage Library](../libraries/storage.md) - Data persistence
- [Configuration Guide](../systems/configuration.md) - Server setup
