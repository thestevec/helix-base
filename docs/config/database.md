# Database Configuration

> **Reference**: `helix.yml`, `gamemode/core/libs/sv_database.lua`

Configuration for database connections, including SQLite and MySQL setup for persistent data storage.

## ⚠️ Important: Use Built-in Database System

**Always use Helix's database system** rather than creating custom database connections. The framework provides:
- Automatic table creation and schema management
- Query builder for safe SQL operations
- Support for SQLite (default) and MySQL/MySQLOO
- Connection pooling and error handling
- Automatic type conversion and sanitization
- Migration system for schema changes

## Database Adapters

Helix supports two database adapters:

1. **SQLite** (Default) - Built into Garry's Mod, no configuration needed
2. **MySQLOO** - External MySQL database for multi-server setups

---

## SQLite Configuration (Default)

SQLite is the default database adapter and requires **no configuration**.

### Features

- ✅ Built into Garry's Mod (no external dependencies)
- ✅ Automatic database file creation
- ✅ Works out-of-the-box
- ✅ Perfect for single-server setups
- ❌ Cannot be shared across multiple servers
- ❌ No external access (web integration)

### Usage

Simply install Helix - SQLite is automatically used if no `helix.yml` file exists.

**Database file location**: `garrysmod/sv.db`

---

## MySQL Configuration

MySQL allows multiple servers to share one database and enables web integration.

### Prerequisites

1. **MySQLOO Module**: Install the [MySQLOO](https://github.com/FredyH/MySQLOO) binary module
2. **MySQL Server**: Access to a MySQL database (version 5.7+ or MariaDB)
3. **Database Credentials**: hostname, username, password, database name

### Installing MySQLOO

**Reference**: See [Getting Started Guide](../manual/getting-started.md#mysql-usage)

1. Download MySQLOO for your server's OS:
   - Windows: `gmsv_mysqloo_win32.dll`
   - Linux: `gmsv_mysqloo_linux.dll`

2. Place the file in: `garrysmod/lua/bin/`

3. Restart your server

### Configuration File

Create `helix.yml` in the `garrysmod/gamemodes/helix/` directory.

**Example Configuration**:

```yaml
database:
  adapter: "mysqloo"
  hostname: "myexampledatabase.com"
  username: "myusername"
  password: "mypassword"
  database: "helix"
  port: 3306
```

### Configuration Options

#### adapter

**Type**: String
**Options**: `"sqlite"`, `"mysqloo"`
**Default**: `"sqlite"`
**Description**: Database adapter to use

```yaml
adapter: "mysqloo"
```

---

#### hostname

**Type**: String
**Required**: Yes (for MySQL only)
**Description**: Database server hostname or IP address

**Examples**:
```yaml
# Domain name
hostname: "db.example.com"

# IP address
hostname: "192.168.1.100"

# Localhost (if MySQL is on same machine)
hostname: "localhost"
```

---

#### username

**Type**: String
**Required**: Yes (for MySQL only)
**Description**: Database username for authentication

```yaml
username: "helix_user"
```

**⚠️ Security**: Create a dedicated MySQL user with limited privileges. Don't use root!

---

#### password

**Type**: String
**Required**: Yes (for MySQL only)
**Description**: Database password for authentication

```yaml
password: "secure_password_here"
```

**⚠️ Security**: Use a strong, unique password. Never commit `helix.yml` to version control!

---

#### database

**Type**: String
**Required**: Yes (for MySQL only)
**Description**: Name of the database to use

```yaml
database: "helix"
```

**Note**: Database must already exist. Helix will create tables automatically.

---

#### port

**Type**: Number (Integer)
**Default**: `3306`
**Description**: MySQL server port

```yaml
port: 3306
```

**Note**: Only change if your MySQL server uses a non-standard port.

---

## Complete Examples

### Example 1: SQLite (Default)

No configuration file needed. Helix automatically uses SQLite.

```lua
-- In Lua code, just use ix.db functions
ix.db.Query("SELECT * FROM ix_characters WHERE id = 1", function(data)
    PrintTable(data)
end)
```

---

### Example 2: Local MySQL Server

`helix.yml`:
```yaml
database:
  adapter: "mysqloo"
  hostname: "localhost"
  username: "helix_local"
  password: "localpass123"
  database: "helix_dev"
  port: 3306
```

---

### Example 3: Remote MySQL Server

`helix.yml`:
```yaml
database:
  adapter: "mysqloo"
  hostname: "db.myserver.com"
  username: "helix_prod"
  password: "prod_secure_password"
  database: "helix_production"
  port: 3306
```

---

### Example 4: Custom MySQL Port

`helix.yml`:
```yaml
database:
  adapter: "mysqloo"
  hostname: "192.168.1.50"
  username: "helix_user"
  password: "password123"
  database: "helix"
  port: 3307  # Custom port
```

---

## YAML Formatting Rules

**CRITICAL**: YAML is whitespace-sensitive. Follow these rules:

### ✅ Correct Formatting

```yaml
database:
  adapter: "mysqloo"
  hostname: "example.com"
  username: "user"
  password: "pass"
  database: "helix"
  port: 3306
```

**Rules**:
- `database:` has NO spaces before it
- All other lines have EXACTLY 2 spaces before them
- Use quotes around string values
- Use colons followed by space: `key: "value"`

### ❌ Incorrect Formatting

```yaml
  database:              # ❌ Extra spaces before database
adapter: "mysqloo"       # ❌ No indentation
    hostname: "test"     # ❌ Too many spaces (4 instead of 2)
username:"user"          # ❌ No space after colon
password: pass           # ❌ Missing quotes
```

---

## Database Tables

Helix automatically creates these tables:

### ix_schema
Tracks database schema versions and column definitions

### ix_characters
Stores all character data (name, description, faction, etc.)

### ix_inventories
Stores inventory definitions and associations

### ix_items
Stores individual item instances and their positions

### ix_players
Stores player data (Steam ID, ban status, etc.)

---

## Using the Database in Code

### Query Execution

```lua
-- Basic query
ix.db.Query("SELECT * FROM ix_characters WHERE id = ?", {characterID}, function(data)
    if data and data[1] then
        print("Character name:", data[1].name)
    end
end)
```

### Query Builder

```lua
-- Insert character
local query = mysql:Insert("ix_characters")
    query:Insert("name", "John Doe")
    query:Insert("description", "A mysterious person")
    query:Insert("faction", 1)
    query:Callback(function(result, status, lastID)
        print("New character ID:", lastID)
    end)
query:Execute()

-- Select with conditions
local query = mysql:Select("ix_characters")
    query:Select("name")
    query:Select("description")
    query:Where("faction", 1)
    query:Limit(10)
    query:Callback(function(result)
        PrintTable(result)
    end)
query:Execute()

-- Update character
local query = mysql:Update("ix_characters")
    query:Update("money", 500)
    query:Where("id", characterID)
query:Execute()
```

---

## Best Practices

### ✅ DO

- Use SQLite for development and single-server setups
- Use MySQL for production multi-server environments
- Use prepared statements (?) to prevent SQL injection
- Use the query builder instead of raw SQL
- Create database backups regularly
- Use dedicated MySQL user with limited privileges
- Keep `helix.yml` secure (don't commit to git)
- Test database connection before going live
- Use connection pooling for better performance

### ❌ DON'T

- Don't use MySQL root user for Helix
- Don't commit `helix.yml` with real credentials
- Don't write raw SQL queries with string concatenation
- Don't skip MySQLOO installation for MySQL setups
- Don't use tabs in YAML files (use 2 spaces)
- Don't share database between incompatible Helix versions
- Don't forget to backup before major updates
- Don't expose MySQL port to public internet

---

## Common Patterns

### Pattern 1: Conditional Database Usage

```lua
-- Check which database adapter is being used
if ix.db.config.adapter == "mysqloo" then
    -- MySQL-specific optimizations
    print("Using MySQL")
elseif ix.db.config.adapter == "sqlite" then
    -- SQLite-specific handling
    print("Using SQLite")
end
```

### Pattern 2: Safe Character Data Query

```lua
-- Always use prepared statements
function GetCharacterByName(name, callback)
    ix.db.Query("SELECT * FROM ix_characters WHERE name = ?", {name}, function(data)
        if data and data[1] then
            callback(data[1])
        else
            callback(nil)
        end
    end)
end
```

### Pattern 3: Transaction-Safe Operations

```lua
-- Perform multiple operations safely
function TransferMoney(fromChar, toChar, amount)
    -- Start transaction
    ix.db.Query("START TRANSACTION")

    -- Deduct from sender
    local query = mysql:Update("ix_characters")
        query:Update("money", fromChar:GetMoney() - amount)
        query:Where("id", fromChar:GetID())
    query:Execute()

    -- Add to receiver
    query = mysql:Update("ix_characters")
        query:Update("money", toChar:GetMoney() + amount)
        query:Where("id", toChar:GetID())
    query:Execute()

    -- Commit transaction
    ix.db.Query("COMMIT")
end
```

---

## Common Issues

### Issue: "MySQLOO module not found"

**Cause**: MySQLOO binary not installed or in wrong location
**Fix**:
1. Download correct MySQLOO version for your OS
2. Place in `garrysmod/lua/bin/`
3. Ensure filename is correct: `gmsv_mysqloo_win32.dll` or `gmsv_mysqloo_linux.dll`
4. Restart server

---

### Issue: "Can't connect to MySQL server"

**Cause**: Incorrect credentials, hostname, or port
**Fix**:
1. Verify credentials are correct
2. Check MySQL server is running
3. Verify firewall allows connection on port 3306
4. Test connection with MySQL client first
5. Check hostname/IP is correct

```bash
# Test MySQL connection from server
mysql -h hostname -u username -p database
```

---

### Issue: YAML parsing error

**Cause**: Incorrect YAML formatting
**Fix**:
1. Ensure exactly 2 spaces for indentation
2. No tabs in file
3. Quotes around string values
4. Space after colons
5. Validate YAML online: http://www.yamllint.com/

**Correct format**:
```yaml
database:
  adapter: "mysqloo"
```

---

### Issue: "Access denied for user"

**Cause**: MySQL user doesn't have permissions
**Fix**: Grant privileges to database user

```sql
-- On MySQL server, grant privileges
GRANT ALL PRIVILEGES ON helix.* TO 'helix_user'@'%';
FLUSH PRIVILEGES;
```

---

### Issue: Data not persisting

**Cause**: Database not connected or queries failing silently
**Fix**: Enable error logging

```lua
-- Check connection status
hook.Add("DatabaseConnected", "CheckConnection", function()
    print("[Helix] Database connected:", ix.db.config.adapter)
end)

-- Log query errors
hook.Add("DatabaseQueryError", "LogErrors", function(error)
    print("[Helix] Database error:", error)
end)
```

---

## Security Best Practices

### Create Dedicated MySQL User

```sql
-- Create user with limited permissions
CREATE USER 'helix_user'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER ON helix.* TO 'helix_user'@'%';
FLUSH PRIVILEGES;
```

### Secure helix.yml

```bash
# Add to .gitignore
echo "helix.yml" >> .gitignore

# Set restrictive permissions (Linux)
chmod 600 garrysmod/gamemodes/helix/helix.yml
```

### Use Environment Variables (Advanced)

```lua
-- Instead of hardcoding in YAML, use environment variables
ix.db.config = {
    adapter = "mysqloo",
    hostname = os.getenv("HELIX_DB_HOST"),
    username = os.getenv("HELIX_DB_USER"),
    password = os.getenv("HELIX_DB_PASS"),
    database = os.getenv("HELIX_DB_NAME"),
    port = tonumber(os.getenv("HELIX_DB_PORT")) or 3306
}
```

---

## Migration from SQLite to MySQL

To migrate existing SQLite data to MySQL:

1. **Backup SQLite database**: Copy `garrysmod/sv.db`
2. **Install MySQLOO** on server
3. **Create MySQL database**:
   ```sql
   CREATE DATABASE helix CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```
4. **Configure helix.yml** with MySQL settings
5. **Use migration tool** (if available) or manually export/import data
6. **Test thoroughly** before going live

---

## See Also

- [Getting Started Guide](../manual/getting-started.md) - Initial setup
- [Database Library](../libraries/database.md) - Database API reference
- [Configuration System](../systems/configuration.md) - Config overview
- Source: `gamemode/core/libs/sv_database.lua`
- Example Config: `helix.example.yml`
