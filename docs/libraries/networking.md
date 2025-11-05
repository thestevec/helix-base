# Networking Library (ix.net)

> **Reference**: `gamemode/core/libs/sv_networking.lua`, `gamemode/core/libs/cl_networking.lua`

The networking library provides synchronized variables (NetVars) for sharing data between server and client without manual net messages. It supports global, entity-specific, and player-specific (local) networked variables.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's NetVar system** rather than creating custom net messages for synchronization. The framework provides:
- Automatic synchronization to new players
- Type-safe variable transmission
- Efficient delta updates (only changed values sent)
- No manual net message registration needed
- Protection against sending functions or invalid types
- Three scopes: global, entity, and local (player-specific)

## Core Concepts

### What are NetVars?

NetVars (Networked Variables) are server-set values automatically synchronized to clients. They replace manual `net.Start()` / `net.Receive()` patterns for state synchronization.

### Three Scopes of NetVars

| Type | Set On | Visible To | Use Case |
|------|--------|-----------|----------|
| **Global** | Server | All clients | Server-wide state (round time, event flags) |
| **Entity** | Entity | All clients who can see entity | Entity state (health, name, owner) |
| **Local** | Player | Only that player | Private player data (money, stamina, secrets) |

### Key Features

- **Automatic sync**: New players automatically receive all current NetVars (line 62-90 in sv_networking.lua)
- **Type safety**: Cannot send functions or invalid types (line 18-31)
- **Efficient**: Only changed values are transmitted
- **Simple API**: No net message registration needed

## Global NetVars

**Reference**: `gamemode/core/libs/sv_networking.lua:39-54` (server), `gamemode/core/libs/cl_networking.lua:8-10` (client)

Global NetVars are server-wide variables visible to all clients.

### SetNetVar (Global)

```lua
-- Server-side only
SetNetVar(key, value, receiver)
```

**Parameters:**
- `key` - Variable name (string)
- `value` - Value to set (any type except functions)
- `receiver` - Player/table of players to send to (nil = broadcast to all)

```lua
-- Set global event flag (visible to all)
SetNetVar("eventActive", true)

-- Set round timer
SetNetVar("roundEndTime", CurTime() + 300)

-- Set to specific player
SetNetVar("serverMessage", "Welcome!", player.GetByID(1))

-- Set complex data
SetNetVar("eventData", {
    name = "Zombie Outbreak",
    startTime = os.time(),
    participants = {}
})
```

### GetNetVar (Global)

```lua
-- Shared (client and server)
local value = GetNetVar(key, default)
```

```lua
-- Check if event is active (client-side HUD)
if GetNetVar("eventActive", false) then
    draw.SimpleText("EVENT ACTIVE", "ixBigFont", ScrW()/2, 50, color_white, TEXT_ALIGN_CENTER)
end

-- Display round timer
local endTime = GetNetVar("roundEndTime", 0)
local remaining = math.max(0, endTime - CurTime())
print("Time remaining: " .. math.Round(remaining) .. " seconds")

-- Get complex data
local eventData = GetNetVar("eventData", {})
if eventData.name then
    print("Current event: " .. eventData.name)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't set global NetVars from client
SetNetVar("hackAttempt", true)  -- Only works on server!

-- WRONG: Don't send functions
SetNetVar("callback", function() end)  -- Blocked by type checking!
```

## Entity NetVars

**Reference**: `gamemode/core/libs/sv_networking.lua:138-181` (server), `gamemode/core/libs/cl_networking.lua:12-17` (client)

Entity NetVars are attached to entities and synchronized to all clients.

### entity:SetNetVar

```lua
-- Server-side only
entity:SetNetVar(key, value, receiver)
```

```lua
-- Set entity owner (character ID)
entity:SetNetVar("owner", character:GetID())

-- Set entity name
entity:SetNetVar("name", "John's Chest")

-- Set door locked state
door:SetNetVar("locked", true)

-- Set to specific players
entity:SetNetVar("secretData", {hidden = true}, specificPlayer)

-- Real example from ix_item.lua:77
self:SetNetVar("data", itemTable.data)
```

### entity:GetNetVar

```lua
-- Shared (client and server)
local value = entity:GetNetVar(key, default)
```

```lua
-- Check door locked state (client HUD)
local door = LocalPlayer():GetEyeTrace().Entity
if IsValid(door) and door:GetNetVar("locked", false) then
    draw.SimpleText("LOCKED", "ixMediumFont", ScrW()/2, ScrH()/2, color_white, TEXT_ALIGN_CENTER)
end

-- Get entity owner
local ownerID = entity:GetNetVar("owner", 0)
local owner = ix.char.loaded[ownerID]
if owner then
    print("Owned by: " .. owner:GetName())
end

-- Real example from ix_shipment.lua:41
if client:GetCharacter() and client:GetCharacter():GetID() == self:GetNetVar("owner", 0) then
    -- Player owns this shipment
end

-- Real example from ix_item.lua:195
item.data = self:GetNetVar("data", {})
```

### entity:ClearNetVars

**Reference**: `gamemode/core/libs/sv_networking.lua:187-199`

Clears all NetVars on an entity.

```lua
-- Clear all NetVars when entity is removed
function ENT:OnRemove()
    self:ClearNetVars()
end
```

## Local NetVars (Player-Specific)

**Reference**: `gamemode/core/libs/sv_networking.lua:101-125` (server), `gamemode/core/libs/cl_networking.lua:23-31` (client)

Local NetVars are player-specific and only visible to that player.

### player:SetLocalVar

```lua
-- Server-side only
player:SetLocalVar(key, value)
```

```lua
-- Set player's private money
client:SetLocalVar("money", 500)

-- Set stamina
client:SetLocalVar("stm", 100)

-- Set secret code only this player can see
client:SetLocalVar("secret", "12345678")

-- Set holding object state
client:SetLocalVar("bIsHoldingObject", true)

-- Complex data
client:SetLocalVar("stats", {
    strength = 10,
    intelligence = 5
})
```

### player:GetLocalVar

```lua
-- Shared (client can only get their own, server can get any player's)
local value = player:GetLocalVar(key, default)
```

**Client-side (own data only)**:
```lua
-- Get own money for HUD
local money = LocalPlayer():GetLocalVar("money", 0)
draw.SimpleText("$" .. money, "ixMediumFont", 50, 50, color_white)

-- Get stamina
local stamina = LocalPlayer():GetLocalVar("stm", 0)

-- Check if holding object (from ix_hands.lua:76)
if LocalPlayer():GetLocalVar("bIsHoldingObject", false) then
    -- Display UI hint
end
```

**Server-side**:
```lua
-- Get any player's local var
local money = client:GetLocalVar("money", 0)
client:SetLocalVar("money", money + 100)
```

### OnLocalVarSet Hook (CLIENT)

**Reference**: `gamemode/core/libs/cl_networking.lua:30`

Called when a local var changes on the client.

```lua
hook.Add("OnLocalVarSet", "MyPlugin", function(key, value)
    if key == "money" then
        chat.AddText("Money changed to: " .. value)
    end
end)
```

## Automatic Synchronization

**Reference**: `gamemode/core/libs/sv_networking.lua:62-90`

### player:SyncVars

When a player joins, all NetVars are automatically synchronized:

```lua
-- Automatically called when player initializes character
-- You should NOT call this manually

-- The function sends:
-- 1. All global NetVars
-- 2. All player's local NetVars
-- 3. All entity NetVars for all entities
```

This ensures new players see the current state immediately without manual synchronization.

## Complete Examples

### Example 1: Event System

```lua
-- SERVER
PLUGIN.name = "Event System"

function PLUGIN:StartEvent(eventName)
    -- Set global NetVars
    SetNetVar("eventActive", true)
    SetNetVar("eventName", eventName)
    SetNetVar("eventStart", CurTime())

    PrintMessage(HUD_PRINTTALK, "Event started: " .. eventName)
end

function PLUGIN:EndEvent()
    SetNetVar("eventActive", false)
    SetNetVar("eventName", nil)
end

ix.command.Add("StartEvent", {
    adminOnly = true,
    arguments = {ix.type.string},
    OnRun = function(self, client, eventName)
        PLUGIN:StartEvent(eventName)
        return "Event started!"
    end
})

-- CLIENT
function PLUGIN:HUDPaint()
    if not GetNetVar("eventActive", false) then return end

    local eventName = GetNetVar("eventName", "Unknown")
    local startTime = GetNetVar("eventStart", 0)
    local elapsed = math.Round(CurTime() - startTime)

    -- Draw event UI
    draw.SimpleText("EVENT: " .. eventName, "ixBigFont",
        ScrW()/2, 50, Color(255, 100, 100), TEXT_ALIGN_CENTER)

    draw.SimpleText("Time: " .. elapsed .. "s", "ixMediumFont",
        ScrW()/2, 80, color_white, TEXT_ALIGN_CENTER)
end
```

### Example 2: Entity Ownership

```lua
-- SERVER
function ENT:Initialize()
    self:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
    self:PhysicsInit(SOLID_VPHYSICS)
    self:SetUseType(SIMPLE_USE)
end

function ENT:SetOwner(character)
    -- Store owner character ID
    self:SetNetVar("owner", character:GetID())
    self:SetNetVar("ownerName", character:GetName())
end

function ENT:Use(activator)
    local char = activator:GetCharacter()
    if not char then return end

    -- Check ownership
    local ownerID = self:GetNetVar("owner", 0)

    if ownerID == 0 then
        -- Unclaimed, let player claim it
        self:SetOwner(char)
        activator:Notify("You claimed this container!")
    elseif ownerID == char:GetID() then
        -- Owner using it
        activator:Notify("Opening your container...")
    else
        activator:Notify("This belongs to " .. self:GetNetVar("ownerName", "someone"))
    end
end

-- CLIENT
function ENT:Draw()
    self:DrawModel()

    local ownerName = self:GetNetVar("ownerName", "Unclaimed")

    -- Draw owner name above entity
    local pos = self:GetPos() + Vector(0, 0, 50)
    local ang = (LocalPlayer():GetPos() - pos):Angle()
    ang:RotateAroundAxis(ang:Forward(), 90)
    ang:RotateAroundAxis(ang:Right(), 90)

    cam.Start3D2D(pos, ang, 0.1)
        draw.SimpleText(ownerName, "ixMediumFont", 0, 0, color_white, TEXT_ALIGN_CENTER)
    cam.End3D2D()
end
```

### Example 3: Character Variables

```lua
-- SERVER
-- Set character NetVars when character loads
function PLUGIN:PlayerLoadedCharacter(client, character)
    -- Set entity NetVars on player entity
    client:SetNetVar("name", character:GetName())
    client:SetNetVar("desc", character:GetDesc())
    client:SetNetVar("class", character:GetClass())

    -- Set local NetVars (only this player sees)
    client:SetLocalVar("money", character:GetMoney())
    client:SetLocalVar("charID", character:GetID())
end

-- Update money when it changes
function PLUGIN:CharacterMoneyGiven(character, amount)
    local client = character:GetPlayer()
    if not IsValid(client) then return end

    client:SetLocalVar("money", character:GetMoney())
end

-- CLIENT
-- Display character info looking at player
function PLUGIN:HUDPaint()
    local trace = LocalPlayer():GetEyeTrace()
    local target = trace.Entity

    if not IsValid(target) or not target:IsPlayer() then return end

    -- Get public info (entity NetVars)
    local name = target:GetNetVar("name", "Unknown")
    local desc = target:GetNetVar("desc", "No description")
    local class = target:GetNetVar("class", "")

    -- Draw character info
    local x, y = ScrW()/2, ScrH()/2 + 50

    draw.SimpleText(name, "ixBigFont", x, y, color_white, TEXT_ALIGN_CENTER)
    draw.SimpleText(class, "ixMediumFont", x, y + 30, Color(200, 200, 200), TEXT_ALIGN_CENTER)
    draw.SimpleText(desc, "ixSmallFont", x, y + 50, Color(150, 150, 150), TEXT_ALIGN_CENTER)
end

-- Display own money (local NetVar)
function PLUGIN:HUDPaint()
    local money = LocalPlayer():GetLocalVar("money", 0)

    draw.SimpleText("Money: $" .. money, "ixMediumFont",
        50, ScrH() - 50, Color(100, 255, 100), TEXT_ALIGN_LEFT)
end
```

### Example 4: Door Lock System

```lua
-- SERVER
function PLUGIN:InitializedPlugins()
    for _, door in ipairs(ents.FindByClass("prop_door_rotating")) do
        door:SetNetVar("locked", false)
        door:SetNetVar("owner", 0)
    end
end

ix.command.Add("DoorLock", {
    description = "Lock/unlock a door you own",
    OnRun = function(self, client)
        local door = client:GetEyeTrace().Entity

        if not IsValid(door) or not door:IsDoor() then
            return "Not looking at a door"
        end

        local character = client:GetCharacter()
        local ownerID = door:GetNetVar("owner", 0)

        -- Check ownership
        if ownerID ~= character:GetID() then
            return "You don't own this door"
        end

        -- Toggle lock
        local locked = door:GetNetVar("locked", false)
        door:SetNetVar("locked", not locked)

        door:EmitSound(locked and "doors/door_latch3.wav" or "doors/door_latch1.wav")

        return locked and "Door unlocked" or "Door locked"
    end
})

-- Prevent use when locked
function PLUGIN:PlayerUse(client, door)
    if door:IsDoor() and door:GetNetVar("locked", false) then
        client:Notify("This door is locked!")
        return false
    end
end

-- CLIENT
-- Draw lock icon on locked doors
function PLUGIN:HUDPaint()
    local trace = LocalPlayer():GetEyeTrace()
    local door = trace.Entity

    if not IsValid(door) or not door:IsDoor() then return end

    if door:GetNetVar("locked", false) then
        -- Draw lock icon
        local scrPos = door:GetPos():ToScreen()

        draw.SimpleText("🔒", "ixIconsLarge", scrPos.x, scrPos.y,
            Color(255, 100, 100), TEXT_ALIGN_CENTER, TEXT_ALIGN_CENTER)
    end
end
```

## Best Practices

### ✅ DO

- Use NetVars for any data that needs to be visible on clients
- Use global NetVars for server-wide state
- Use entity NetVars for entity-specific state
- Use local NetVars for player-private data
- Always provide default values in Get calls
- Set NetVars only from server
- Use descriptive key names
- Clear NetVars when entities are removed

### ❌ DON'T

- Don't try to set NetVars from client (won't work)
- Don't send functions or metatable objects
- Don't use NetVars for temporary/frequent values (use net messages)
- Don't forget default values in GetNetVar calls
- Don't set NetVars every frame (only on change)
- Don't use NetVars for large data (use network messages with compression)
- Don't access NetVars without validating entity first
- Don't set NetVars to nil to remove them (use specific value like 0 or "")

## Common Patterns

### Pattern 1: Character ID Sync

```lua
-- Server: Set when character loads
client:SetNetVar("char", character:GetID())

-- Client: Get character from player
local charID = client:GetNetVar("char", 0)
local character = ix.char.loaded[charID]
```
**Used in**: `gamemode/core/meta/sh_character.lua:150`

### Pattern 2: Entity Ownership

```lua
-- Server: Set owner
entity:SetNetVar("owner", character:GetID())

-- Client: Check ownership
local ownerID = entity:GetNetVar("owner", 0)
if ownerID == LocalPlayer():GetCharacter():GetID() then
    -- I own this
end
```
**Used in**: `entities/entities/ix_shipment.lua:41`

### Pattern 3: State Flags

```lua
-- Server: Set state
client:SetLocalVar("bIsHoldingObject", true)

-- Client: Check state
if LocalPlayer():GetLocalVar("bIsHoldingObject", false) then
    -- Holding object, show UI hint
end
```
**Used in**: `entities/weapons/ix_hands.lua:76`

## Common Issues

### Issue: NetVar returns default on client

**Cause**: NetVar not set on server, or client hasn't received update yet
**Fix**: Always set NetVar on server, check timing

```lua
-- WRONG: Setting on client
if CLIENT then
    LocalPlayer():SetNetVar("test", true)  -- Does nothing!
end

-- CORRECT: Set on server
if SERVER then
    client:SetNetVar("test", true)
end
```

### Issue: NetVar nil instead of value

**Cause**: Not providing default value
**Fix**: Always provide default in Get calls

```lua
-- WRONG: No default
local money = client:GetLocalVar("money")
if money then  -- Always check!
    UseMoneyHere(money)
end

-- CORRECT: With default
local money = client:GetLocalVar("money", 0)
-- Always safe to use, worst case is 0
```

### Issue: Function sent error

**Cause**: Trying to send function or table with function
**Fix**: Only send serializable data

```lua
-- WRONG: Contains function
entity:SetNetVar("data", {
    value = 100,
    callback = function() end  -- ERROR!
})

-- CORRECT: Only data
entity:SetNetVar("data", {
    value = 100,
    name = "Test"
})
```

### Issue: NetVar not updating on client

**Cause**: Setting to same value (optimization skips send), or entity not valid
**Fix**: Check entity validity, force update if needed

```lua
-- Tables always update even if "same"
entity:SetNetVar("data", someTable)  -- Always sends

-- Primitives skip if unchanged
entity:SetNetVar("count", 5)
entity:SetNetVar("count", 5)  -- Second call does nothing

-- Force update by toggling or using SendNetVar
entity:SendNetVar("count", receiver)
```

## NetVars vs Net Messages

### Use NetVars for:
- ✅ State synchronization
- ✅ Variables that change occasionally
- ✅ Values clients need to read
- ✅ Automatic sync to new players

### Use Net Messages for:
- ✅ One-time events
- ✅ Large data transfers
- ✅ Frequent updates
- ✅ Client-to-server communication
- ✅ Custom protocols

## Performance Considerations

- NetVars are sent individually when changed
- Setting same value to primitive skips network send (line 41, 158)
- Tables always send even if "same value"
- New players receive ALL NetVars on join (can be hundreds of messages)
- Consider using net messages for very large or frequently updated data

## Real Framework Usage

NetVars are used extensively in Helix core:

**Character ID**: `gamemode/core/meta/sh_character.lua:150`
```lua
client:SetNetVar("char", self:GetID())
```

**Item Data**: `entities/entities/ix_item.lua:77`
```lua
self:SetNetVar("data", itemTable.data)
```

**Holding State**: `entities/weapons/ix_hands.lua`
```lua
client:SetLocalVar("bIsHoldingObject", true)
if LocalPlayer():GetLocalVar("bIsHoldingObject", false) then
```

## See Also

- [Character System](../systems/character.md) - Uses NetVars for character data sync
- [Items System](../systems/items.md) - Item entities use NetVars
- [Plugin Development](../plugins/creating-plugins.md) - Using NetVars in plugins
- Source: `gamemode/core/libs/sv_networking.lua`, `gamemode/core/libs/cl_networking.lua`
