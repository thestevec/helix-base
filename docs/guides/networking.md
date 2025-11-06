# Client-Server Networking Guide

> **Reference**: `gamemode/core/libs/sv_networking.lua`, `gamemode/core/libs/cl_networking.lua`

Guide to network communication between client and server in Helix using Garry's Mod's networking system.

## ⚠️ Important: Use Proper Networking

**Always use Garry's Mod's `net` library** with proper realm checks. The framework handles:
- Network message registration
- Data serialization
- Bandwidth management
- Security validation

## Basic Networking

### Server to Client

```lua
-- SERVER: Send to single client
util.AddNetworkString("MyMessage")

net.Start("MyMessage")
net.WriteString("Hello!")
net.WriteInt(42, 32)
net.Send(client)

-- CLIENT: Receive
net.Receive("MyMessage", function()
    local text = net.ReadString()
    local number = net.ReadInt(32)
    print(text, number)
end)
```

### Client to Server

```lua
-- CLIENT: Send to server
net.Start("RequestData")
net.WriteInt(characterID, 32)
net.SendToServer()

-- SERVER: Receive
util.AddNetworkString("RequestData")

net.Receive("RequestData", function(len, client)
    local charID = net.ReadInt(32)

    -- Validate client
    local character = client:GetCharacter()
    if not character or character:GetID() != charID then
        return  -- Security check!
    end

    -- Process request
end)
```

## Practical Examples

### Example 1: Open Custom Menu

```lua
-- SERVER
util.AddNetworkString("OpenVendorMenu")

function OpenVendorMenu(client, vendorID)
    net.Start("OpenVendorMenu")
    net.WriteInt(vendorID, 32)
    net.Send(client)
end

-- CLIENT
net.Receive("OpenVendorMenu", function()
    local vendorID = net.ReadInt(32)

    -- Open VGUI panel
    local frame = vgui.Create("VendorFrame")
    frame:SetVendor(vendorID)
end)
```

### Example 2: Update HUD Element

```lua
-- SERVER: Notify all players
util.AddNetworkString("UpdateObjective")

function BroadcastObjective(text)
    net.Start("UpdateObjective")
    net.WriteString(text)
    net.Broadcast()  -- Send to all clients
end

-- CLIENT: Display
net.Receive("UpdateObjective", function()
    local text = net.ReadString()
    HUD_OBJECTIVE = text
end)
```

### Example 3: Request-Response Pattern

```lua
-- CLIENT: Request player stats
net.Start("GetPlayerStats")
net.WriteString(player:SteamID())
net.SendToServer()

-- SERVER: Respond with stats
util.AddNetworkString("GetPlayerStats")
util.AddNetworkString("ReceivePlayerStats")

net.Receive("GetPlayerStats", function(len, client)
    local steamID = net.ReadString()

    -- Load from database
    LoadStats(steamID, function(stats)
        net.Start("ReceivePlayerStats")
        net.WriteTable(stats)
        net.Send(client)
    end)
end)

-- CLIENT: Receive stats
net.Receive("ReceivePlayerStats", function()
    local stats = net.ReadTable()
    DisplayStats(stats)
end)
```

## Data Types

```lua
-- Strings
net.WriteString("text")
local text = net.ReadString()

-- Numbers
net.WriteInt(42, 32)  -- 32-bit integer
local num = net.ReadInt(32)

net.WriteFloat(3.14)
local float = net.ReadFloat()

-- Booleans
net.WriteBool(true)
local bool = net.ReadBool()

-- Tables
net.WriteTable({key = "value"})
local tbl = net.ReadTable()

-- Entities
net.WriteEntity(ent)
local ent = net.ReadEntity()

-- Vectors
net.WriteVector(Vector(0, 0, 0))
local vec = net.ReadVector()

-- Colors
net.WriteColor(Color(255, 0, 0))
local col = net.ReadColor()
```

## Best Practices

### ✅ DO
- Register network strings on both realms
- Validate all client input on server
- Use specific data types (not just strings)
- Check entity validity after reading
- Limit message frequency (avoid spam)

### ❌ DON'T
- Don't trust client data without validation
- Don't send excessive data
- Don't net.Broadcast() frequently
- Don't forget to register network strings

## Security

```lua
-- SERVER: Always validate!
net.Receive("BuyItem", function(len, client)
    local itemID = net.ReadString()
    local character = client:GetCharacter()

    -- Validate character exists
    if not character then return end

    -- Validate item exists
    local item = ix.item.Get(itemID)
    if not item then return end

    -- Validate money
    if character:GetMoney() < item.price then
        client:Notify("Not enough money!")
        return
    end

    -- Process purchase
    character:TakeMoney(item.price)
    character:GetInventory():Add(itemID)
end)
```

## See Also

- [Database Guide](database.md)
- [UI Development](ui-development.md)
- [Commands Guide](commands.md)
