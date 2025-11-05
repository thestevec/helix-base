# Implementing Hooks in Your Schema

> **Reference**: `schema/sh_schema.lua`, `schema/sv_schema.lua`, `schema/cl_schema.lua`

This guide shows you how to implement hooks in your schema to customize gameplay, respond to events, and add custom behavior.

## ⚠️ Important: Use Schema Hook Methods

**Always implement hooks using `Schema:HookName()`** syntax in your schema files. The framework provides:
- Automatic hook registration
- Proper calling order
- Return value handling
- Realm-specific execution
- Integration with core systems

## Core Concepts

### What Are Hooks?

Hooks are functions that run when specific events occur:
- **Player Events**: Spawn, death, character load
- **Item Events**: Use, transfer, pickup
- **Character Events**: Creation, deletion, save
- **World Events**: Entity creation, prop damage
- **UI Events**: Menu open, HUD draw

### Hook Types

**Helix Hooks** (Schema/Plugin):
```lua
function Schema:PlayerLoadedCharacter(client, character)
    -- Helix-specific hook
end
```

**GMod Hooks** (Standard Garry's Mod):
```lua
hook.Add("PlayerSpawn", "SchemaPlayerSpawn", function(client)
    -- Standard GMod hook
end)
```

**Use Schema methods for Helix hooks, use hook.Add for GMod hooks.**

### Schema Files

Organize hooks by realm:

- `schema/sh_schema.lua` - Shared hooks (both realms)
- `schema/sv_schema.lua` - Server-only hooks
- `schema/cl_schema.lua` - Client-only hooks

## Creating Your First Hook

### Character Load Hook

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Called when player loads a character

    -- Give starting items to new characters
    if not character:GetData("initialized") then
        local inventory = character:GetInventory()
        inventory:Add("item_hands")

        character:SetData("initialized", true)
        character:SetData("joinDate", os.time())
    end

    -- Restore health
    local health = character:GetData("health", 100)
    client:SetHealth(health)

    -- Greet player
    client:ChatPrint("Welcome, " .. character:GetName())
end
```

### Player Spawn Hook

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerSpawn(client)
    -- Called whenever player spawns

    local character = client:GetCharacter()
    if not character then return end

    -- Set faction-specific spawn settings
    local faction = ix.faction.indices[character:GetFaction()]

    if faction then
        client:SetRunSpeed(faction.runSpeed or 240)
        client:SetWalkSpeed(faction.walkSpeed or 100)
    end

    -- Apply stamina if schema uses it
    local stamina = character:GetData("stamina", 100)
    character:SetData("stamina", stamina)
end
```

## Common Schema Hooks

### Character Hooks

#### OnCharacterCreated

**Reference**: `docs/hooks/plugin.lua`

Called once when a new character is created:

```lua
-- File: schema/sv_schema.lua
function Schema:OnCharacterCreated(client, character)
    -- Give starting equipment
    local inventory = character:GetInventory()
    inventory:Add("item_hands")

    -- Set starting money based on faction
    local faction = ix.faction.indices[character:GetFaction()]
    local startMoney = ix.config.Get("startingMoney", 100)

    if faction.uniqueID == "police" then
        startMoney = startMoney * 2  -- Police get double
    end

    character:SetMoney(startMoney)

    -- Initialize character data
    character:SetData("playTime", 0)
    character:SetData("hunger", 100)
    character:SetData("thirst", 100)
    character:SetData("reputation", 0)

    -- Log creation
    ix.log.Add(client, "characterCreated", character:GetName())
end
```

#### CanPlayerUseCharacter

Check if player can use a character:

```lua
-- File: schema/sh_schema.lua
function Schema:CanPlayerUseCharacter(client, character)
    -- Prevent using banned characters
    if character:GetData("banned") then
        return false, "This character is banned"
    end

    -- Require whitelist for certain factions
    local faction = ix.faction.indices[character:GetFaction()]
    if faction.flag then
        if not client:GetData("whitelists", {})[faction.uniqueID] then
            return false, "You are not whitelisted for " .. faction.name
        end
    end

    -- Admin-only factions
    if faction.adminOnly and not client:IsAdmin() then
        return false, "This faction is admin-only"
    end

    return true
end
```

#### CharacterPreSave

Called before character is saved to database:

```lua
-- File: schema/sv_schema.lua
function Schema:CharacterPreSave(character)
    local client = character:GetPlayer()

    if IsValid(client) then
        -- Save current health
        character:SetData("health", client:Health())

        -- Save position
        character:SetData("lastPos", client:GetPos())

        -- Update play time
        local playTime = character:GetData("playTime", 0)
        character:SetData("playTime", playTime + 1)
    end
end
```

### Player Hooks

#### PlayerLoadedCharacter

Called when character is loaded:

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Restore saved data
    local health = character:GetData("health", 100)
    client:SetHealth(health)

    -- Apply character-specific effects
    local mutations = character:GetData("mutations", {})
    for _, mutation in ipairs(mutations) do
        self:ApplyMutation(client, mutation)
    end

    -- Check for timed effects
    local poisoned = character:GetData("poisoned", false)
    if poisoned then
        self:StartPoisonEffect(client)
    end

    -- Welcome message
    client:ChatPrint("Welcome back, " .. character:GetName())
end
```

#### PlayerDeath

Handle player death:

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerDeath(client, inflictor, attacker)
    local character = client:GetCharacter()
    if not character then return end

    -- Drop money on death
    local money = character:GetMoney()
    if money > 0 then
        local entity = ents.Create("ix_money")
        entity:SetPos(client:GetPos() + Vector(0, 0, 10))
        entity:SetMoney(money)
        entity:Spawn()

        character:SetMoney(0)
    end

    -- Save death location
    character:SetData("deathPos", client:GetPos())
    character:SetData("deathTime", os.time())

    -- Death penalties
    local experience = character:GetData("experience", 0)
    character:SetData("experience", math.max(0, experience - 100))

    -- Log death
    if IsValid(attacker) and attacker:IsPlayer() then
        ix.log.Add(attacker, "killedPlayer", client:Name())
    end
end
```

#### PlayerSpawn

Handle respawning:

```lua
-- File: schema/sv_schema.lua
function Schema:PlayerSpawn(client)
    local character = client:GetCharacter()
    if not character then return end

    -- Set faction model
    local faction = ix.faction.indices[character:GetFaction()]
    if faction then
        local models = faction:GetModels(client)
        if models and #models > 0 then
            client:SetModel(character:GetModel() or models[1])
        end
    end

    -- Give faction weapons
    timer.Simple(0.25, function()
        if IsValid(client) and character then
            local class = character:GetClass()

            if class then
                local classTable = ix.class.Get(class)
                if classTable then
                    for _, weapon in ipairs(classTable.weapons or {}) do
                        client:Give(weapon)
                    end
                end
            end
        end
    end)
end
```

### Item Hooks

#### CanPlayerUseItem

Control item usage:

```lua
-- File: schema/sh_schema.lua
function Schema:CanPlayerUseItem(client, item)
    local character = client:GetCharacter()

    -- Check if character is tied
    if character:GetData("tied") then
        return false, "You are tied up"
    end

    -- Check if character has required flag
    if item.flag and not character:HasFlags(item.flag) then
        return false, "You don't have permission to use this item"
    end

    -- Check faction restrictions
    if item.factionID and character:GetFaction() != item.factionID then
        return false, "Only specific factions can use this"
    end

    return true
end
```

#### CanPlayerDropItem

Control item dropping:

```lua
-- File: schema/sv_schema.lua
function Schema:CanPlayerDropItem(client, item)
    -- Prevent dropping quest items
    if item:GetData("questItem") then
        return false, "You cannot drop quest items"
    end

    -- Prevent dropping while in combat
    if client:GetData("inCombat") then
        return false, "You cannot drop items during combat"
    end

    return true
end
```

#### CanPlayerTakeItem

Control item pickup:

```lua
-- File: schema/sv_schema.lua
function Schema:CanPlayerTakeItem(client, item)
    local character = client:GetCharacter()

    -- Check inventory space
    if not character:GetInventory():CanItemFit(item) then
        return false, "Not enough inventory space"
    end

    -- Prevent stealing faction items
    if item:GetData("ownerFaction") then
        if character:GetFaction() != item:GetData("ownerFaction") then
            return false, "This item belongs to another faction"
        end
    end

    return true
end
```

### World Hooks

#### OnNPCKilled

Handle NPC deaths:

```lua
-- File: schema/sv_schema.lua
function Schema:OnNPCKilled(npc, attacker, inflictor)
    if not IsValid(attacker) or not attacker:IsPlayer() then return end

    local character = attacker:GetCharacter()
    if not character then return end

    -- Give experience for kills
    local experience = character:GetData("experience", 0)
    character:SetData("experience", experience + 10)

    -- Drop loot
    local lootTable = {
        "item_scrap",
        "item_battery",
        "item_components"
    }

    local loot = lootTable[math.random(#lootTable)]
    ix.item.Spawn(loot, npc:GetPos() + Vector(0, 0, 10))

    attacker:ChatPrint("+" .. 10 .. " experience")
end
```

#### EntityTakeDamage

Modify damage:

```lua
-- File: schema/sv_schema.lua
function Schema:EntityTakeDamage(entity, dmgInfo)
    if entity:IsPlayer() then
        local character = entity:GetCharacter()
        if not character then return end

        -- Reduce damage for armored characters
        local armor = character:GetData("armorRating", 0)
        if armor > 0 then
            local damage = dmgInfo:GetDamage()
            dmgInfo:SetDamage(damage * (1 - armor / 100))
        end

        -- Apply damage modifiers from mutations
        if character:GetData("hasRegeneration") then
            dmgInfo:ScaleDamage(0.75)  -- 25% damage reduction
        end
    end
end
```

### Chat Hooks

#### CanPlayerUseChat

Control chat access:

```lua
-- File: schema/sh_schema.lua
function Schema:CanPlayerUseChat(speaker, chatType, text, anonymous)
    local character = speaker:GetCharacter()

    -- Prevent chat if character is gagged
    if character:GetData("gagged") then
        return false
    end

    -- Prevent OOC if player is new
    if chatType == "ooc" then
        if character:GetData("playTime", 0) < 300 then  -- 5 minutes
            return false
        end
    end

    return true
end
```

#### OnChatReceived

Modify chat messages:

```lua
-- File: schema/sh_schema.lua
function Schema:OnChatReceived(client, chatType, text, anonymous, receivers)
    -- Add prefixes based on character data
    if chatType == "ic" then
        local character = client:GetCharacter()
        local rank = character:GetData("rank")

        if rank then
            text = "[" .. rank .. "] " .. text
        end
    end

    return text
end
```

### UI Hooks (Client)

#### HUDPaint

Custom HUD rendering:

```lua
-- File: schema/cl_schema.lua
function Schema:HUDPaint()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if not character then return end

    -- Don't draw if option disabled
    if not ix.option.Get("showCustomHUD", true) then
        return
    end

    -- Draw hunger bar
    local hunger = character:GetData("hunger", 100)
    local scrW, scrH = ScrW(), ScrH()

    draw.RoundedBox(0, scrW - 210, scrH - 60, 200, 20, Color(0, 0, 0, 150))
    draw.RoundedBox(0, scrW - 205, scrH - 55, hunger * 1.9, 10, Color(255, 150, 0))

    draw.SimpleText("Hunger", "DermaDefault", scrW - 210, scrH - 75, color_white)
end
```

#### CharacterMenuCreated

Modify character menu:

```lua
-- File: schema/cl_schema.lua
function Schema:CharacterMenuCreated(panel)
    -- Add custom button to character menu
    local button = panel:Add("ixMenuButton")
    button:SetText("Statistics")
    button:SizeToContents()
    button.DoClick = function()
        -- Open statistics panel
        vgui.Create("ixStatsPanel")
    end
end
```

## Best Practices

### ✅ DO

- Implement hooks in appropriate schema files (sh/sv/cl)
- Use `Schema:HookName()` for Helix hooks
- Check realm before realm-specific code
- Validate all data before using it
- Return appropriate values (true/false for Can hooks)
- Use character:GetData() for character-specific data
- Log important events with ix.log
- Test hooks with multiple scenarios
- Comment complex hook logic

### ❌ DON'T

- Don't use hook.Add for Helix hooks in schema
- Don't forget realm checks
- Don't modify core framework tables directly
- Don't create infinite loops in hooks
- Don't forget to return values in Can hooks
- Don't perform expensive operations in HUDPaint
- Don't trust client data without validation
- Don't forget to check if character exists

## Advanced Hook Patterns

### Timed Effects

```lua
-- File: schema/sv_schema.lua
function Schema:StartPoisonEffect(client)
    local uniqueID = "Poison_" .. client:SteamID()

    timer.Create(uniqueID, 2, 15, function()
        if IsValid(client) and client:Alive() then
            client:TakeDamage(5)

            if client:Health() <= 0 then
                timer.Remove(uniqueID)
            end
        else
            timer.Remove(uniqueID)
        end
    end)
end

function Schema:PlayerLoadedCharacter(client, character)
    if character:GetData("poisoned") then
        self:StartPoisonEffect(client)
    end
end

function Schema:PlayerDeath(client)
    timer.Remove("Poison_" .. client:SteamID())
end
```

### Data Validation

```lua
function Schema:CharacterPreSave(character)
    -- Validate and clamp data before saving
    local health = character:GetData("health", 100)
    character:SetData("health", math.Clamp(health, 0, 100))

    local hunger = character:GetData("hunger", 100)
    character:SetData("hunger", math.Clamp(hunger, 0, 100))

    -- Remove invalid data
    local items = character:GetData("customItems", {})
    for k, v in pairs(items) do
        if not isnumber(v) then
            items[k] = nil
        end
    end
end
```

### Event Broadcasting

```lua
function Schema:PlayerLoadedCharacter(client, character)
    -- Broadcast to nearby players
    for _, ply in ipairs(player.GetAll()) do
        if ply:GetPos():Distance(client:GetPos()) < 500 then
            ply:ChatPrint(character:GetName() .. " has arrived")
        end
    end
end
```

## Testing Hooks

1. **Trigger Events**: Perform actions that call the hook
2. **Check Realm**: Verify hook runs in correct realm
3. **Test Edge Cases**: nil values, disconnected players
4. **Performance**: Ensure hooks don't lag server
5. **Integration**: Test with other schema features

## See Also

- [Hook Documentation](../hooks/) - Available hook references
- [Schema Structure](structure.md) - Schema file organization
- [Character System](../systems/character.md) - Character data
- [Plugin Hooks](../hooks/plugin.lua) - Available Helix hooks
- Source: `schema/sh_schema.lua`, `schema/sv_schema.lua`, `schema/cl_schema.lua`
