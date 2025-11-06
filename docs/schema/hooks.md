# Implementing Schema Hooks

> **Reference**: `docs/hooks/plugin.lua`, `schema/sv_schema.lua`, `schema/cl_schema.lua`

Hooks are functions that Helix calls at specific moments during gameplay. This guide shows how to implement hooks in your schema.

## ⚠️ Important: Use Schema Hook System

**Always use `Schema:HookName()`** syntax rather than `hook.Add()` in schema files. The framework provides:
- Automatic hook registration
- Proper calling order
- Integration with plugin system
- Clean organization

## Core Concepts

### What are Schema Hooks?

Hooks let you respond to game events:
- **PlayerLoadedCharacter** - When character is loaded
- **PlayerSpawn** - When player spawns
- **PlayerDeath** - When player dies
- **CanPlayerUseItem** - Check if player can use item
- **PlayerSay** - Process player chat messages

Your schema uses hooks to:
- Initialize characters
- Modify gameplay behavior
- Add custom restrictions
- Respond to events

## Using Hooks in Schema

### File Location

Place hooks in:
```
schema/sv_schema.lua   -- Server hooks
schema/cl_schema.lua   -- Client hooks
schema/sh_schema.lua   -- Shared hooks (rare)
```

### Basic Hook Syntax

```lua
-- schema/sv_schema.lua
function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Give starting items on first spawn
    if not character:GetData("initialized") then
        local inventory = character:GetInventory()
        inventory:Add("item_hands")

        character:GiveMoney(100)
        character:SetData("initialized", true)
    end
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use hook.Add in schema files
hook.Add("PlayerLoadedCharacter", "MySchema", function(client, character)
    -- Don't do this in schema!
end)
```

## Common Schema Hooks

### Server Hooks

#### PlayerLoadedCharacter

**Reference**: Character is loaded and active

```lua
function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Called when character is loaded
    -- Use for: Initial setup, welcome messages, data initialization

    local name = character:GetName()
    client:ChatPrint("Welcome back, " .. name)

    -- First time setup
    if not character:GetData("initialized") then
        self:InitializeNewCharacter(client, character)
    end

    -- Restore equipment
    self:RestoreEquipment(client, character)
end
```

#### PlayerSpawn

**Reference**: Player entity spawns

```lua
function Schema:PlayerSpawn(client)
    -- Called every spawn
    -- Use for: Setting speeds, health, models

    -- Set default speeds
    client:SetRunSpeed(240)
    client:SetWalkSpeed(100)
    client:SetJumpPower(200)

    -- Faction-specific spawn behavior
    local character = client:GetCharacter()
    if character then
        local faction = ix.faction.indices[character:GetFaction()]

        if faction.OnSpawn then
            faction:OnSpawn(client)
        end
    end
end
```

#### PlayerDeath

**Reference**: Player dies

```lua
function Schema:PlayerDeath(client, inflictor, attacker)
    -- Called when player dies
    -- Use for: Death penalties, logging, announcements

    local character = client:GetCharacter()

    if character then
        -- Drop money on death
        local money = character:GetMoney()
        if money > 0 then
            ix.currency.Spawn(client:GetPos(), money)
            character:SetMoney(0)
        end

        -- Log death
        ix.log.Add(client, "death", attacker:IsPlayer() and attacker:Name() or "World")
    end
end
```

#### CanPlayerUseItem

**Reference**: Check if player can use item

```lua
function Schema:CanPlayerUseItem(client, item)
    -- Return false to prevent item use
    -- Use for: Restrictions, special conditions

    local character = client:GetCharacter()

    -- Faction restrictions
    if item.faction and item.faction != character:GetFaction() then
        client:Notify("Only " .. ix.faction.Get(item.faction).name .. " can use this")
        return false
    end

    -- In combat restriction
    if character:GetData("inCombat") then
        client:Notify("Cannot use items in combat")
        return false
    end

    -- Can use item
    return true
end
```

#### PlayerSay

**Reference**: Player sends chat message

```lua
function Schema:PlayerSay(client, text)
    local character = client:GetCharacter()

    if not character then
        return ""
    end

    -- Process OOC commands
    if string.sub(text, 1, 2) == "//" then
        -- OOC message
        local message = string.sub(text, 3)
        ix.chat.Send(client, "ooc", message)
        return ""
    end

    -- Allow normal processing
end
```

### Client Hooks

#### HUDPaint

**Reference**: Draw HUD elements

```lua
-- schema/cl_schema.lua
function Schema:HUDPaint()
    -- Draw custom HUD elements
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if character then
        local stamina = character:GetData("stamina", 100)

        -- Draw stamina bar
        draw.RoundedBox(0, 50, ScrH() - 80, 200, 20, Color(0, 0, 0, 150))
        draw.RoundedBox(0, 52, ScrH() - 78, (196 * stamina / 100), 16, Color(0, 255, 0, 200))
    end
end
```

#### CreateCharacterInfo

**Reference**: Add character info to tab menu

```lua
-- schema/cl_schema.lua
function Schema:CreateCharacterInfo(panel)
    -- Add custom character information
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if character then
        local rank = character:GetData("rank", 1)

        panel:AddRow("rank")
            :SetText("Rank: " .. rank)
            :SetBackgroundColor(Color(50, 50, 50))
            :SizeToContents()
    end
end
```

#### OnCharacterMenuCreated

**Reference**: Character menu opened

```lua
-- schema/cl_schema.lua
function Schema:OnCharacterMenuCreated(panel)
    -- Customize character selection menu
    panel:SetTitle(self.name)
end
```

## Complete Examples

### New Character Setup

```lua
-- schema/sv_schema.lua
function Schema:InitializeNewCharacter(client, character)
    local inventory = character:GetInventory()
    local faction = ix.faction.indices[character:GetFaction()]

    -- Give default items
    inventory:Add("item_hands")

    -- Faction-specific setup
    if faction.uniqueID == "citizen" then
        inventory:Add("cid")
        character:GiveMoney(100)
    elseif faction.uniqueID == "police" then
        inventory:Add("cid_police")
        inventory:Add("handheld_radio")
        character:GiveMoney(200)
    end

    -- Set initial data
    character:SetData("initialized", true)
    character:SetData("rank", 1)
    character:SetData("joinDate", os.time())

    -- Welcome message
    client:ChatPrint("Welcome to " .. self.name)
end

function Schema:PlayerLoadedCharacter(client, character, lastChar)
    if not character:GetData("initialized") then
        self:InitializeNewCharacter(client, character)
    end
end
```

### Combat System

```lua
-- schema/sv_schema.lua
function Schema:PlayerHurt(client, attacker, health, damage)
    if not attacker:IsPlayer() then return end

    local victimChar = client:GetCharacter()
    local attackerChar = attacker:GetCharacter()

    if victimChar and attackerChar then
        -- Mark both as in combat
        victimChar:SetData("inCombat", true)
        attackerChar:SetData("inCombat", true)

        -- Start combat timer
        timer.Create("CombatTimer_" .. client:SteamID(), 30, 1, function()
            if victimChar then
                victimChar:SetData("inCombat", false)
            end
        end)

        timer.Create("CombatTimer_" .. attacker:SteamID(), 30, 1, function()
            if attackerChar then
                attackerChar:SetData("inCombat", false)
            end
        end)
    end
end

function Schema:CanPlayerUseItem(client, item)
    local character = client:GetCharacter()

    if character:GetData("inCombat") then
        client:Notify("Cannot use items in combat")
        return false
    end
end
```

### Stamina System

```lua
-- schema/sv_schema.lua
function Schema:Move(client, moveData)
    local character = client:GetCharacter()
    if not character then return end

    local stamina = character:GetData("stamina", 100)

    -- Drain stamina when sprinting
    if moveData:KeyDown(IN_SPEED) and stamina > 0 then
        stamina = math.max(stamina - 0.2, 0)
        character:SetData("stamina", stamina)

        if stamina <= 0 then
            moveData:SetMaxSpeed(client:GetWalkSpeed())
        end
    else
        -- Regenerate stamina
        if stamina < 100 then
            stamina = math.min(stamina + 0.1, 100)
            character:SetData("stamina", stamina)
        end
    end
end

-- schema/cl_schema.lua
function Schema:HUDPaint()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if character then
        local stamina = character:GetData("stamina", 100)

        -- Draw stamina bar
        local x, y = 50, ScrH() - 80
        local width, height = 200, 20

        draw.RoundedBox(0, x, y, width, height, Color(0, 0, 0, 150))
        draw.RoundedBox(0, x + 2, y + 2, (width - 4) * stamina / 100, height - 4, Color(0, 200, 255, 200))

        draw.SimpleText("Stamina", "DermaDefault", x + width / 2, y + height / 2, Color(255, 255, 255), TEXT_ALIGN_CENTER, TEXT_ALIGN_CENTER)
    end
end
```

### Salary System

```lua
-- schema/sv_schema.lua
function Schema:InitPostEntity()
    -- Start salary timer
    if ix.config.Get("salaryEnabled", true) then
        timer.Create("SchemaSalary", 300, 0, function()
            self:PaySalaries()
        end)
    end
end

function Schema:PaySalaries()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()

        if character then
            local faction = ix.faction.indices[character:GetFaction()]
            local class = character:GetClass()
            local salary = 0

            -- Class salary takes priority
            if class then
                local classInfo = ix.class.Get(class)
                salary = classInfo.pay or 0
            end

            -- Fallback to faction salary
            if salary == 0 then
                salary = faction.pay or 0
            end

            if salary > 0 then
                character:GiveMoney(salary)
                client:Notify("Received salary: " .. ix.currency.Get(salary))
            end
        end
    end
end
```

### Door Access System

```lua
-- schema/sh_schema.lua
function Schema:CanPlayerAccessDoor(client, door, access)
    local character = client:GetCharacter()
    if not character then return false end

    local faction = character:GetFaction()

    -- Check door faction
    local doorFaction = door:GetNetVar("faction")
    if doorFaction then
        if faction != doorFaction then
            return false, "This door belongs to another faction"
        end
    end

    -- Check door rank
    local doorRank = door:GetNetVar("rank", 0)
    if doorRank > 0 then
        local charRank = character:GetData("rank", 0)
        if charRank < doorRank then
            return false, "Your rank is too low"
        end
    end

    return true
end
```

## Essential Hook Reference

### Character Hooks

```lua
-- Character created (first time)
function Schema:OnCharacterCreated(client, character)
end

-- Character loaded (every time)
function Schema:PlayerLoadedCharacter(client, character, lastChar)
end

-- Character deleted
function Schema:OnCharacterDelete(client, character)
end

-- Before character data saved
function Schema:PreCharacterSave(character)
end

-- After character data saved
function Schema:PostCharacterSave(character)
end
```

### Player Hooks

```lua
-- Player spawned
function Schema:PlayerSpawn(client)
end

-- Player died
function Schema:PlayerDeath(client, inflictor, attacker)
end

-- Player hurt
function Schema:PlayerHurt(client, attacker, health, damage)
end

-- Player disconnecting
function Schema:PlayerDisconnected(client)
end

-- Player said something
function Schema:PlayerSay(client, text)
end
```

### Item Hooks

```lua
-- Can player use item?
function Schema:CanPlayerUseItem(client, item)
end

-- Can player drop item?
function Schema:CanPlayerDropItem(client, item)
end

-- Can player take item?
function Schema:CanPlayerTakeItem(client, item)
end

-- Can player interact with item entity?
function Schema:CanPlayerInteractItem(client, action, item)
end
```

### Inventory Hooks

```lua
-- Can transfer item?
function Schema:CanTransferItem(item, oldInv, newInv)
end

-- Item transferred
function Schema:OnItemTransferred(item, oldInv, newInv)
end
```

## Best Practices

### ✅ DO

- Use `Schema:HookName()` syntax
- Place server hooks in `sv_schema.lua`
- Place client hooks in `cl_schema.lua`
- Check IsValid() on entities
- Validate character existence
- Return meaningful values (false to prevent action)
- Clean up timers and data
- Use appropriate realm (SERVER/CLIENT)

### ❌ DON'T

- Don't use `hook.Add()` in schema files
- Don't forget to check character validity
- Don't modify core framework files
- Don't create expensive operations in frequently-called hooks
- Don't forget return values in "Can" hooks
- Don't assume entities are valid
- Don't bypass hook system

## Common Patterns

### Initialization Pattern

```lua
function Schema:InitPostEntity()
    -- SERVER: Called once after entities spawn
    if SERVER then
        -- Start timers
        timer.Create("SchemaUpdate", 60, 0, function()
            self:Update()
        end)
    end
end

function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Initialize new characters
    if not character:GetData("initialized") then
        self:SetupNewCharacter(client, character)
    end
end
```

### Validation Pattern

```lua
function Schema:CanPlayerDoAction(client, ...)
    local character = client:GetCharacter()

    -- No character = no action
    if not character then
        return false, "No character loaded"
    end

    -- Check faction
    if character:GetFaction() != FACTION_ALLOWED then
        return false, "Wrong faction"
    end

    -- Check other conditions
    if not character:HasFlags("x") then
        return false, "Missing permission"
    end

    -- Allow action
    return true
end
```

### Cleanup Pattern

```lua
function Schema:PlayerDisconnected(client)
    local character = client:GetCharacter()

    if character then
        -- Save data before disconnect
        character:Save()

        -- Clean up timers
        timer.Remove("PlayerTimer_" .. client:SteamID())

        -- Clean up temporary data
        character:SetData("temporary", nil)
    end
end
```

## See Also

- [Hook Documentation](../../docs/hooks/plugin.lua) - Complete hook reference
- [Configuration](configuration.md) - Schema configuration
- [Schema Structure](structure.md) - File organization
- Source: `docs/hooks/plugin.lua`
