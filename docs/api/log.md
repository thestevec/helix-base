# Log API (ix.log)

> **Reference**: `gamemode/core/libs/sh_log.lua`

The log API provides server logging with categorization, severity levels, and admin viewing. Logs are saved to `data/helix/logs/` and visible to admins in real-time.

## ⚠️ Important: Use Built-in Helix Logging

**Always use Helix's built-in logging** rather than `print()` or file operations. The framework automatically provides:
- Log types with formatting
- Severity levels with colors
- Automatic file writing
- Real-time admin notifications
- Custom log handlers
- Structured log messages

## Core Concepts

### What is Logging?

The log system records server events:
- Categorized by type (command, money, damage, etc.)
- Color-coded by severity
- Saved to daily log files
- Streamed to online admins
- Extensible with custom handlers

### Log Flags

```lua
FLAG_NORMAL   = 0  -- Gray (default)
FLAG_SUCCESS  = 1  -- Green (positive events)
FLAG_WARNING  = 2  -- Yellow (warnings)
FLAG_DANGER   = 3  -- Red (critical events)
FLAG_SERVER   = 4  -- Light blue (server events)
FLAG_DEV      = 5  -- Light blue (development)
```

## Library Tables

### ix.log.types

**Reference**: `gamemode/core/libs/sh_log.lua:51`

**Realm**: Server

Registered log types with formats.

```lua
-- Check log type
local commandLog = ix.log.types["command"]
print(commandLog.format)  -- Format string
print(commandLog.flag)    -- Severity level
```

## Library Functions

### ix.log.AddType

**Reference**: `gamemode/core/libs/sh_log.lua:58`

**Realm**: Server

```lua
ix.log.AddType(logType, format, flag)
```

Registers a new log type.

**Parameters**:
- `logType` (string) - Unique log category
- `format` (string or function) - Message format or formatter function
- `flag` (number) - Severity flag (FLAG_*)

**Example**:
```lua
-- Simple format string
ix.log.AddType("trade", "%s traded with %s", FLAG_NORMAL)

-- With function formatter
ix.log.AddType("damage", function(client, target, amount)
    return string.format("%s dealt %d damage to %s",
        client:Name(), amount, target:Name())
end, FLAG_WARNING)

-- Admin action
ix.log.AddType("ban", "%s banned %s for: %s", FLAG_DANGER)
```

### ix.log.Add

**Reference**: `gamemode/core/libs/sh_log.lua:107`

**Realm**: Server

```lua
ix.log.Add(client, logType, ...)
```

Logs an event.

**Parameters**:
- `client` (Player or nil) - Player who triggered event
- `logType` (string) - Log category
- `...` - Arguments for format string

**Example**:
```lua
-- Log command execution
ix.log.Add(client, "command", "/givemoney", "100")

-- Log money transaction
ix.log.Add(client, "money", 500)

-- Log admin action
ix.log.Add(admin, "ban", target:Name(), reason)

-- Server event (no player)
ix.log.Add(nil, "serverStart", "Server started")
```

### ix.log.AddRaw

**Reference**: `gamemode/core/libs/sh_log.lua:90`

**Realm**: Server

```lua
ix.log.AddRaw(logString, bNoSave)
```

Logs a pre-formatted string.

**Parameters**:
- `logString` (string) - Complete log message
- `bNoSave` (bool, optional) - Skip writing to file

**Example**:
```lua
-- Direct log message
ix.log.AddRaw("Custom event occurred")

-- Temporary log (not saved)
ix.log.AddRaw("Debug message", true)
```

## Registering Log Handlers

### ix.log.RegisterHandler

**Reference**: `gamemode/core/libs/sh_log.lua:136`

**Realm**: Server

```lua
ix.log.RegisterHandler(name, data)
```

Registers custom log handler.

**Parameters**:
- `name` (string) - Handler name
- `data` (table) - Handler with Load/Write methods

**Example**:
```lua
local HANDLER = {}

function HANDLER.Load()
    -- Initialize handler
    print("Custom log handler loaded")
end

function HANDLER.Write(client, message, flag, logType, args)
    -- Handle log entry
    print("[CUSTOM]", message)

    -- Write to database
    -- Send to Discord webhook
    -- etc.
end

ix.log.RegisterHandler("Custom", HANDLER)
```

## Complete Examples

### Trading System Logging

```lua
-- Register log type
ix.log.AddType("trade", "%s traded %s to %s for %s", FLAG_SUCCESS)

-- Log trade
function CompleteTrade(client1, client2, item1, item2)
    ix.log.Add(client1, "trade", item1:GetName(), client2:Name(), item2:GetName())

    -- Perform trade...
end
```

### Admin Action Logging

```lua
-- Register types
ix.log.AddType("kick", "%s kicked %s: %s", FLAG_WARNING)
ix.log.AddType("ban", "%s banned %s (%s): %s", FLAG_DANGER)

ix.command.Add("Kick", {
    adminOnly = true,
    arguments = {ix.type.player, ix.type.text},
    OnRun = function(self, client, target, reason)
        ix.log.Add(client, "kick", target:Name(), reason)
        target:Kick(reason)
    end
})

ix.command.Add("Ban", {
    superAdminOnly = true,
    arguments = {ix.type.player, ix.type.number, ix.type.text},
    OnRun = function(self, client, target, duration, reason)
        ix.log.Add(client, "ban", target:Name(), duration, reason)
        -- Ban logic...
    end
})
```

### Database Log Handler

```lua
local HANDLER = {}

function HANDLER.Load()
    -- Create logs table
    ix.db.Query([[
        CREATE TABLE IF NOT EXISTS server_logs (
            id INT AUTO_INCREMENT PRIMARY KEY,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            player_id VARCHAR(20),
            log_type VARCHAR(50),
            message TEXT,
            flag INT
        )
    ]])
end

function HANDLER.Write(client, message, flag, logType, args)
    local steamID = client and client:SteamID() or "SERVER"

    ix.db.Query("INSERT INTO server_logs (player_id, log_type, message, flag) VALUES (?, ?, ?, ?)",
        steamID, logType, message, flag)
end

ix.log.RegisterHandler("Database", HANDLER)
```

### Discord Webhook Logging

```lua
local HANDLER = {}
local webhookURL = "https://discord.com/api/webhooks/..."

function HANDLER.Write(client, message, flag, logType, args)
    -- Only log important events
    if flag < FLAG_WARNING then return end

    local color = 0xFFFF00  -- Yellow
    if flag == FLAG_DANGER then
        color = 0xFF0000  -- Red
    end

    local data = util.TableToJSON({
        embeds = {{
            title = "Server Log",
            description = message,
            color = color,
            timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
        }}
    })

    HTTP({
        url = webhookURL,
        method = "POST",
        body = data,
        type = "application/json",
        success = function() end,
        failed = function() end
    })
end

ix.log.RegisterHandler("Discord", HANDLER)
```

### Character Action Logging

```lua
-- Log character-specific actions
ix.log.AddType("charCreate", "%s created character: %s", FLAG_SUCCESS)
ix.log.AddType("charDelete", "%s deleted character: %s", FLAG_WARNING)
ix.log.AddType("itemPickup", "%s picked up: %s", FLAG_NORMAL)

hook.Add("OnCharacterCreated", "LogCharCreate", function(client, character)
    ix.log.Add(client, "charCreate", character:GetName())
end)

hook.Add("OnCharacterDelete", "LogCharDelete", function(client, character)
    ix.log.Add(client, "charDelete", character:GetName())
end)

hook.Add("PlayerUse", "LogItemPickup", function(client, entity)
    if entity:GetClass() == "ix_item" then
        local item = entity:GetItemTable()
        if item then
            ix.log.Add(client, "itemPickup", item.name)
        end
    end
end)
```

## Best Practices

### ✅ DO

- Register log types for important events
- Use appropriate severity flags
- Include relevant player information
- Log admin actions
- Log economy transactions
- Use descriptive log type names
- Include context in log messages

### ❌ DON'T

- Don't log excessively (every movement, etc.)
- Don't log sensitive information (passwords)
- Don't use print() instead of logging
- Don't forget to pass client parameter
- Don't log client-side events (logs are server-only)
- Don't use same log type for different events

## Common Patterns

### Conditional Logging

```lua
if ix.config.Get("logDamage", true) then
    ix.log.Add(attacker, "damage", victim:Name(), dmgInfo:GetDamage())
end
```

### Formatted Log Messages

```lua
ix.log.AddType("inventory", function(client, action, item, amount)
    local amtStr = amount > 1 and (" x" .. amount) or ""
    return string.format("%s %s %s%s",
        client:Name(), action, item, amtStr)
end, FLAG_NORMAL)

-- Usage
ix.log.Add(client, "inventory", "added", "health_kit", 5)
-- "John added health_kit x5"
```

### Error Logging

```lua
local success, err = pcall(function()
    -- Risky operation
end)

if not success then
    ix.log.AddRaw("[ERROR] " .. err, false)
end
```

## Common Issues

### Logs Not Appearing

**Cause**: Player lacks "Helix - Logs" permission.

**Fix**: Grant permission via admin mod.

### Log Type Doesn't Exist

**Cause**: Using unregistered log type.

**Fix**: Register type first:
```lua
ix.log.AddType("myLogType", "%s did something", FLAG_NORMAL)
ix.log.Add(client, "myLogType", "test")
```

### Logs Not Saving to File

**Cause**: File permission issues or handler error.

**Fix**: Check `data/helix/logs/` permissions.

## See Also

- [Command API](command.md) - Commands automatically logged
- [Character API](character.md) - Character action logging
- [Currency API](currency.md) - Money transactions logged
- Source: `gamemode/core/libs/sh_log.lua`
