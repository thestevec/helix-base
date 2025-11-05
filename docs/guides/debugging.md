# Debugging Techniques

> **Reference**: Garry's Mod console, Helix logging system

Practical debugging techniques for finding and fixing issues in Helix development.

## Console Basics

### Enable Developer Mode

```
// In console
developer 1

// See more detailed errors
con_logfile console.log
```

### Viewing Errors

```
// Filter for errors only
con_filter_enable 1
con_filter_text "error"

// Or warnings
con_filter_text "warn"
```

## Print Debugging

### Basic Printing

```lua
-- Simple print
print("Value:", variable)

-- Print table
PrintTable(tableVariable)

-- Formatted print
print(string.format("Player %s has $%d", name, money))

-- Client vs Server
if SERVER then
    print("[SERVER]", "text")
else
    print("[CLIENT]", "text")
end
```

### Advanced Printing

```lua
-- Print with color (SERVER console)
MsgC(Color(255, 0, 0), "[ERROR] ", color_white, "Something broke!\n")

-- Print to specific client (SERVER)
client:ChatPrint("Debug: " .. tostring(value))

-- Print all players
for _, ply in ipairs(player.GetAll()) do
    print(ply:Name(), ply:Health())
end
```

## Common Issues

### Issue 1: "attempt to index nil value"

**Cause**: Trying to access something that doesn't exist

**Debug**:
```lua
-- BAD
local character = client:GetCharacter()
local money = character:GetMoney()  -- Error if no character!

-- GOOD
local character = client:GetCharacter()
if not character then
    print("ERROR: No character for", client:Name())
    return
end
local money = character:GetMoney()
```

### Issue 2: Hook Not Running

**Debug**:
```lua
function PLUGIN:PlayerSpawn(client)
    print("PlayerSpawn called for", client:Name())  -- Is this printing?

    if SERVER then
        print("Running on server")
    else
        print("Running on client")
    end
end
```

### Issue 3: Command Not Found

**Debug**:
```lua
function PLUGIN:InitializedPlugins()
    print("Registering command...")

    ix.command.Add("Test", {
        OnRun = function(self, client)
            print("Command ran!")
            return "Success"
        end
    })

    print("Command registered:", ix.command.list.test != nil)
end
```

### Issue 4: Item Not Spawning

**Debug**:
```lua
local inventory = character:GetInventory()

print("Inventory:", inventory)
print("Inventory ID:", inventory and inventory:GetID())

inventory:Add("sh_medkit", 1, nil, nil, function(item)
    print("Item added:", item)

    if not item then
        print("ERROR: Item is nil!")
        print("Does item exist?", ix.item.Get("sh_medkit"))
    end
end)
```

## Testing Tools

### In-Game Commands

```lua
// Reload schema
sh_rebuildschema

// Give item
ix_giveitem "Player" sh_medkit

// Set money
/setmoney "Player" 1000

// Make admin
sh_makeadmin

// Become superadmin
lua_run game.SinglePlayer() or Entity(1):SetUserGroup("superadmin")
```

### Lua Console Commands

```lua
// Run server code
lua_run print("Server:", player.GetCount())

// Run client code
lua_run_cl print("Client:", LocalPlayer():Name())

// Print variable
lua_run PrintTable(ix.faction.indices)

// Test function
lua_run local char = Entity(1):GetCharacter(); print(char:GetMoney())
```

## Logging System

### Using ix.log

```lua
-- SERVER ONLY
-- Log player action
ix.log.Add(client, "playerAction", "did something")

// View logs
lua_run PrintTable(ix.log.stored)
```

### Custom Logging

```lua
function PLUGIN:Log(message, level)
    level = level or "INFO"

    local timestamp = os.date("%H:%M:%S")
    local formatted = string.format("[%s][%s] %s", timestamp, level, message)

    print(formatted)

    -- Log to file
    file.Append("myplugin.log", formatted .. "\n")
end

// Usage
PLUGIN:Log("Player connected", "INFO")
PLUGIN:Log("Error occurred", "ERROR")
```

## Performance Debugging

### Profiling Code

```lua
local startTime = SysTime()

-- Your code here
for i = 1, 1000 do
    -- something
end

local endTime = SysTime()
print("Took:", (endTime - startTime) * 1000, "ms")
```

### Finding Lag Sources

```lua
-- Add to Think hook
local lastCheck = 0

function PLUGIN:Think()
    if CurTime() - lastCheck > 1 then
        lastCheck = CurTime()

        print("Players:", player.GetCount())
        print("Entities:", #ents.GetAll())
        print("Frame time:", FrameTime() * 1000, "ms")
    end
end
```

## Network Debugging

```lua
-- SERVER: Log when sending
net.Start("MyMessage")
print("Sending MyMessage to", client:Name())
net.Send(client)

-- CLIENT: Log when receiving
net.Receive("MyMessage", function()
    print("Received MyMessage")
    print("Data:", net.ReadString())
end)

-- Check if registered
lua_run print("Registered:", util.NetworkStringToID("MyMessage") != 0)
```

## Best Practices

### ✅ DO
- Add print statements strategically
- Check nil values before accessing
- Test in both singleplayer and multiplayer
- Use developer mode
- Log errors to file
- Test with multiple players

### ❌ DON'T
- Don't leave debug prints in release
- Don't print in loops/Think hooks
- Don't assume entities are valid
- Don't skip error checking
- Don't test only as admin

## Debugging Checklist

When something doesn't work:

1. **Check console** for errors
2. **Add prints** to see execution flow
3. **Verify realm** (SERVER/CLIENT)
4. **Check entity validity** (IsValid())
5. **Verify hook registration**
6. **Test in sandbox** mode first
7. **Check permissions** (flags, admin)
8. **Verify file locations** and names
9. **Check dependencies** (other plugins)
10. **Test with clean database**

## See Also

- [Setup Guide](setup.md)
- [First Plugin Tutorial](first-plugin.md)
- [Hooks Guide](hooks.md)
