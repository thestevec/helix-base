# Logging System (ix.log)

> **Reference**: `gamemode/core/libs/sh_log.lua`

The logging system provides centralized event logging with customizable message formats, severity levels, and multiple output handlers including console, file storage, and live admin notifications.

## ⚠️ Important: Use Built-in Logging System

**Always use Helix's logging system** rather than creating custom logging implementations. The framework provides:
- Automatic file logging with timestamps and date organization
- Live log streaming to admins with CAMI permission integration
- Customizable log types with formatting functions
- Colored console output by severity level
- Extensible handler system for custom storage backends
- Network synchronization for client notification

## Core Concepts

### What is the Logging System?

The logging system records server events, player actions, and administrative activities. It supports:
- **Log Types**: Categorized events with custom formatting
- **Severity Flags**: Color-coded importance levels (normal, success, warning, danger, server, dev)
- **Handlers**: Pluggable storage backends (file, database, external services)
- **Live Streaming**: Real-time log delivery to connected admins

### Key Terms

- **Log Type**: A named category for specific events (e.g., "charCreate", "itemAction", "playerDeath")
- **Log Flag**: Severity level determining color and importance
- **Handler**: System for processing/storing log entries
- **Format Function**: Converts event data into readable log messages

### Severity Flags

**Reference**: `gamemode/core/libs/sh_log.lua:16-21`

```lua
FLAG_NORMAL = 0    -- Gray - routine events
FLAG_SUCCESS = 1   -- Green - successful actions
FLAG_WARNING = 2   -- Yellow - concerning events
FLAG_DANGER = 3    -- Red - critical actions
FLAG_SERVER = 4    -- Light blue - server events
FLAG_DEV = 5       -- Light blue - development info
```

## Using the Logging System

### ix.log.AddType

**Reference**: `gamemode/core/libs/sh_log.lua:58`

```lua
ix.log.AddType(logType, format, flag)
```

Registers a new log type with custom formatting. Must be called on server before logging events.

**Parameters**:
- `logType` (string): Unique identifier for this log type
- `format` (string or function): Static message or function that returns formatted string
- `flag` (number, optional): Severity flag (defaults to FLAG_NORMAL)

**Complete Example**:
```lua
-- In your plugin's SERVER section
if (SERVER) then
    -- Simple string format
    ix.log.AddType("doorLock", "Player locked a door", FLAG_NORMAL)

    -- Dynamic format with function
    ix.log.AddType("itemCraft", function(client, itemName, materials)
        return string.format("%s crafted '%s' using %s",
            client:Name(),
            itemName,
            table.concat(materials, ", ")
        )
    end, FLAG_SUCCESS)

    -- Format with conditional logic
    ix.log.AddType("trade", function(client, target, amount, isGiving)
        if isGiving then
            return string.format("%s gave %s to %s",
                client:Name(),
                ix.currency.Get(amount),
                target:Name()
            )
        else
            return string.format("%s received %s from %s",
                client:Name(),
                ix.currency.Get(amount),
                target:Name()
            )
        end
    end, FLAG_NORMAL)

    -- Complex log with multiple arguments
    ix.log.AddType("adminAction", function(client, action, targetName, reason)
        return string.format("%s used '%s' on %s (Reason: %s)",
            client:SteamName(),
            action,
            targetName,
            reason or "None"
        )
    end, FLAG_DANGER)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't print to console manually
print(client:Name() .. " performed action")  -- Won't be logged!

-- WRONG: Don't use MsgN/Msg for logging
Msg("[LOG] Player did something\n")  -- No admin streaming, no file storage!

-- WRONG: Don't create custom log files
file.Append("my_custom_log.txt", logMessage)  -- Bypass framework!
```

### ix.log.Add

**Reference**: `gamemode/core/libs/sh_log.lua:107`

```lua
ix.log.Add(client, logType, ...)
```

Records a log entry of the specified type. Sends to console, file, and connected admins.

**Parameters**:
- `client` (Player or nil): Player who initiated the action
- `logType` (string): Registered log type identifier
- `...` (any): Arguments passed to the format function

**Complete Example**:
```lua
-- Simple event logging
function PLUGIN:PlayerSpawnedProp(client, model, entity)
    ix.log.Add(client, "spawnProp", model, entity:EntIndex())
end

-- Character action logging
function PLUGIN:OnCharacterCreate(client, character)
    ix.log.Add(client, "charCreate", character:GetName())
end

-- Item interaction logging
function PLUGIN:OnItemTransferred(item, inventory, oldInv)
    local owner = inventory.GetOwner and inventory:GetOwner()

    if IsValid(owner) and owner:IsPlayer() then
        ix.log.Add(owner, "itemTransfer", item:GetName(), item:GetID())
    end
end

-- Combat logging
function PLUGIN:PlayerDeath(victim, weapon, killer)
    if IsValid(killer) and killer:IsPlayer() then
        local weaponName = IsValid(weapon) and weapon:GetClass() or "unknown"
        ix.log.Add(victim, "playerDeath", killer:Name(), weaponName)
    else
        ix.log.Add(victim, "playerDeath", "Environment", nil)
    end
end

-- Administrative action logging
function PLUGIN:OnPlayerKicked(admin, target, reason)
    ix.log.Add(admin, "adminKick", target:Name(), reason)
end

-- Trading system logging
function PLUGIN:OnPlayerTrade(client, target, itemsGiven, itemsReceived)
    local giveList = table.concat(itemsGiven, ", ")
    local receiveList = table.concat(itemsReceived, ", ")

    ix.log.Add(client, "trade", target:Name(), giveList, receiveList)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't log undefined types
ix.log.Add(client, "myUndefinedType", data)  -- Error! Must call AddType first

-- WRONG: Don't pass client as string
ix.log.Add(client:Name(), "action")  -- Pass player entity, not name!

-- WRONG: Don't use for frequent events
hook.Add("Think", "MyLogThink", function()
    ix.log.Add(nil, "think")  -- Will spam logs every frame!
end)
```

### ix.log.AddRaw

**Reference**: `gamemode/core/libs/sh_log.lua:90`

```lua
ix.log.AddRaw(logString, bNoSave)
```

Adds a pre-formatted log message without using log types. Useful for direct logging.

**Parameters**:
- `logString` (string): Fully formatted log message
- `bNoSave` (boolean, optional): If true, don't save to file

**Complete Example**:
```lua
-- Direct formatted logging
function PLUGIN:OnDatabaseError(err)
    ix.log.AddRaw("Database error: " .. err, false)
end

-- Temporary debug logging (not saved)
function PLUGIN:DebugInfo(info)
    if ix.config.Get("debugMode") then
        ix.log.AddRaw("[DEBUG] " .. info, true)  -- Console only, no file
    end
end

-- Custom formatted message
function PLUGIN:OnCustomEvent(data)
    local message = string.format("Custom Event: %s at %s",
        data.type,
        os.date("%X")
    )
    ix.log.AddRaw(message)
end
```

### Custom Log Handlers

**Reference**: `gamemode/core/libs/sh_log.lua:136-156`

```lua
ix.log.RegisterHandler(name, handlerTable)
```

Registers a custom log handler for storing logs in databases, external services, etc.

**Handler Table**:
- `Load()`: Called when log tables/directories are loaded
- `Write(client, message, flag, logType, args)`: Called when log is written

**Complete Example**:
```lua
-- Database log handler
if (SERVER) then
    local HANDLER = {}

    function HANDLER.Load()
        -- Create database table
        ix.db.Query([[
            CREATE TABLE IF NOT EXISTS ix_logs (
                id INT NOT NULL AUTO_INCREMENT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                steamid VARCHAR(32),
                type VARCHAR(64),
                message TEXT,
                flag TINYINT,
                PRIMARY KEY (id),
                INDEX (steamid),
                INDEX (type)
            )
        ]])
    end

    function HANDLER.Write(client, message, flag, logType, args)
        local steamID = IsValid(client) and client:SteamID() or "CONSOLE"

        ix.db.Query(
            "INSERT INTO ix_logs (steamid, type, message, flag) VALUES (?, ?, ?, ?)",
            steamID, logType or "raw", message, flag or 0
        )
    end

    ix.log.RegisterHandler("Database", HANDLER)
end

-- Discord webhook handler
if (SERVER) then
    local HANDLER = {}
    local webhookURL = "https://discord.com/api/webhooks/..."

    function HANDLER.Load()
        print("[Logging] Discord webhook handler initialized")
    end

    function HANDLER.Write(client, message, flag, logType, args)
        -- Only send important logs to Discord
        if flag == FLAG_DANGER or flag == FLAG_WARNING then
            local color = flag == FLAG_DANGER and 16711680 or 16776960
            local data = util.TableToJSON({
                embeds = {{
                    title = "Server Log",
                    description = message,
                    color = color,
                    timestamp = os.date("!%Y-%m-%dT%H:%M:%S")
                }}
            })

            HTTP({
                url = webhookURL,
                method = "POST",
                body = data,
                type = "application/json"
            })
        end
    end

    ix.log.RegisterHandler("Discord", HANDLER)
end
```

## Complete Example

### Full Logging Plugin

```lua
PLUGIN.name = "Advanced Logging"
PLUGIN.author = "YourName"
PLUGIN.description = "Comprehensive logging for server events"

if (SERVER) then
    -- Register log types
    function PLUGIN:InitializedPlugins()
        -- Property damage
        ix.log.AddType("propDamage", function(client, prop, damage)
            return string.format("%s damaged prop %s for %d HP",
                client:Name(),
                prop:GetModel(),
                damage
            )
        end, FLAG_WARNING)

        -- Money transactions
        ix.log.AddType("moneyDrop", function(client, amount, position)
            return string.format("%s dropped %s at %s",
                client:Name(),
                ix.currency.Get(amount),
                tostring(position)
            )
        end, FLAG_NORMAL)

        -- Admin commands
        ix.log.AddType("adminTP", function(client, targetName, location)
            return string.format("%s teleported %s to %s",
                client:Name(),
                targetName,
                location or "their position"
            )
        end, FLAG_DANGER)
    end

    -- Hook into events
    function PLUGIN:EntityTakeDamage(entity, dmgInfo)
        if entity:GetClass() == "prop_physics" then
            local attacker = dmgInfo:GetAttacker()

            if IsValid(attacker) and attacker:IsPlayer() then
                ix.log.Add(attacker, "propDamage", entity, dmgInfo:GetDamage())
            end
        end
    end

    function PLUGIN:OnCharacterMoneyDropped(client, amount, position)
        ix.log.Add(client, "moneyDrop", amount, position)
    end

    -- Custom admin command logging
    function PLUGIN:OnAdminTeleport(admin, target)
        ix.log.Add(admin, "adminTP", target:Name(), nil)
    end
end
```

## Best Practices

### ✅ DO

- Use `ix.log.AddType()` to register log types before using them
- Provide descriptive format functions that include relevant context
- Use appropriate severity flags (FLAG_DANGER for admin actions, FLAG_WARNING for exploits)
- Log important player actions (character creation, item trades, admin commands)
- Include player names, item IDs, and timestamps in log messages
- Create custom handlers for external logging services
- Use `ix.log.AddRaw()` for pre-formatted messages

### ❌ DON'T

- Don't bypass logging with print() or Msg()
- Don't log high-frequency events (Think hook, every frame, etc.)
- Don't forget to register log types before using them
- Don't log sensitive data (passwords, IP addresses without consent)
- Don't create duplicate log files manually
- Don't pass client names instead of player entities
- Don't spam logs with redundant information

## Common Patterns

### Pattern 1: Character Action Logging

```lua
-- Log character-related actions
function PLUGIN:InitializedPlugins()
    ix.log.AddType("charPurchase", function(client, itemName, cost)
        return string.format("%s purchased '%s' for %s",
            client:GetCharacter():GetName(),
            itemName,
            ix.currency.Get(cost)
        )
    end, FLAG_SUCCESS)
end

function PLUGIN:OnCharacterPurchase(client, itemName, cost)
    ix.log.Add(client, "charPurchase", itemName, cost)
end
```

### Pattern 2: Admin Action Tracking

```lua
-- Track administrative actions
ix.log.AddType("adminBan", function(client, targetName, duration, reason)
    return string.format("%s banned %s for %s (Reason: %s)",
        client:SteamName(),
        targetName,
        duration,
        reason
    )
end, FLAG_DANGER)

function PLUGIN:BanPlayer(admin, target, duration, reason)
    ix.log.Add(admin, "adminBan", target:Name(), duration, reason)
    -- ... ban logic
end
```

### Pattern 3: Item Tracking

```lua
-- Log item creation and destruction
ix.log.AddType("itemSpawn", function(client, itemName, itemID)
    return string.format("%s spawned item '%s' (#%d)",
        client:Name(),
        itemName,
        itemID
    )
end, FLAG_NORMAL)

function PLUGIN:OnItemSpawned(client, item)
    ix.log.Add(client, "itemSpawn", item:GetName(), item:GetID())
end
```

### Pattern 4: Conditional Logging

```lua
-- Log only when conditions are met
function PLUGIN:PlayerSpawn(client)
    local char = client:GetCharacter()

    if char and char:GetFaction() == FACTION_ADMIN then
        ix.log.Add(client, "adminSpawn", char:GetName())
    end
end
```

## Common Issues

### Issue: Log Type Not Found Error

**Cause**: Attempting to log before registering log type
**Fix**: Register log types in plugin initialization

```lua
-- Register in InitializedPlugins hook
function PLUGIN:InitializedPlugins()
    ix.log.AddType("myLogType", function(client, data)
        return string.format("%s did something: %s", client:Name(), data)
    end, FLAG_NORMAL)
end

-- Then use it
function PLUGIN:SomeHook(client)
    ix.log.Add(client, "myLogType", "test data")
end
```

### Issue: Logs Not Showing to Admins

**Cause**: Admin doesn't have "Helix - Logs" permission
**Fix**: Grant permission via CAMI usergroup

```lua
-- Check if player has log access
CAMI.PlayerHasAccess(client, "Helix - Logs", function(hasAccess)
    if hasAccess then
        client:Notify("You have log access!")
    end
end)
```

### Issue: Format Function Errors

**Cause**: Wrong number of arguments or nil values
**Fix**: Validate arguments in format function

```lua
ix.log.AddType("safeLog", function(client, arg1, arg2)
    arg1 = arg1 or "unknown"
    arg2 = arg2 or "none"

    return string.format("%s performed action with %s and %s",
        IsValid(client) and client:Name() or "Console",
        tostring(arg1),
        tostring(arg2)
    )
end, FLAG_NORMAL)
```

## See Also

- [Commands System](../systems/commands.md) - Logging command usage
- [Character System](../systems/character.md) - Character action logging
- [Items System](../systems/items.md) - Item transaction logging
- [Plugin System](../plugins/plugin-system.md) - Creating logging plugins
- Source: `gamemode/core/libs/sh_log.lua`
- Example: `plugins/logging.lua`
