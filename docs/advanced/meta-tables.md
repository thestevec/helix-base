# Meta Tables

> **Reference**: `gamemode/core/meta/sh_*.lua`

Meta tables extend Garry's Mod base classes (Player, Entity, etc.) and Helix classes (Character, Item, etc.) with additional methods and functionality specific to the Helix framework.

## ⚠️ Important: Use Built-in Meta Methods

**Always use Helix's meta table extensions** rather than creating standalone utility functions. The framework provides:
- Extended Player methods for character interaction
- Entity helper methods for common checks
- Character accessor methods for data management
- Item methods for inventory operations
- Inventory methods for item management
- Automatic integration with Helix systems
- Consistent API across the framework

## Core Concepts

### What are Meta Tables?

Meta tables in Lua allow you to extend existing classes with new methods. Helix extends both Garry's Mod base classes and its own custom classes to provide convenient methods for common operations.

**Helix extends these classes**:
- **Player** - Methods for character, inventory, and player-specific operations
- **Entity** - Helper methods for doors, chairs, locks, and item checks
- **Character** - Data accessors, inventory management, faction/class operations
- **Item** - Item properties, transfer operations, and hooks
- **Inventory** - Grid-based item storage and manipulation

### Key Terms

- **Meta Table**: A table that defines the methods and behavior of a class
- **Extension**: Adding new methods to existing Garry's Mod classes
- **Accessor**: Getter/setter methods for class properties
- **Meta Method**: A function attached to a class via its meta table

## Player Meta Extensions

**Reference**: `gamemode/core/meta/sh_player.lua`

### Common Player Methods

```lua
-- Character access
local character = player:GetCharacter()  -- Get player's active character

-- Play time
local seconds = player:GetPlayTime()  -- Time played on server

-- State checks
local raised = player:IsWepRaised()  -- Is weapon raised
local restricted = player:IsRestricted()  -- Is player bound/cuffed
local canShoot = player:CanShootWeapon()  -- Can fire weapon
local running = player:IsRunning()  -- Is player running
local stuck = player:IsStuck()  -- Is player stuck in geometry

-- Model checks
local female = player:IsFemale()  -- Does player have female model

-- Name accessors
local steamName = player:SteamName()  -- Steam name
local name = player:Name()  -- Character name (if loaded)
```

**Complete Example**:
```lua
-- Check player state before action
function PLUGIN:CanPlayerUseItem(client, item)
    if client:IsRestricted() then
        client:Notify("You cannot use items while restrained!")
        return false
    end

    if not client:GetCharacter() then
        return false
    end

    return true
end

-- Restrict player movement
function PLUGIN:HandcuffPlayer(client)
    client:SetNetVar("restricted", true)
    client:SetNetVar("canShoot", false)

    timer.Simple(60, function()
        if IsValid(client) then
            client:SetNetVar("restricted", false)
            client:SetNetVar("canShoot", true)
        end
    end)
end

-- Check if player is actively moving
function PLUGIN:IsPlayerActive(client)
    local character = client:GetCharacter()

    if not character then
        return false
    end

    if client:IsRestricted() then
        return false
    end

    if client:IsRunning() then
        return true
    end

    return false
end

-- Gender-specific interactions
function PLUGIN:GetPronouns(client)
    if client:IsFemale() then
        return "she", "her"
    else
        return "he", "him"
    end
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom utility functions for player checks
function IsPlayerRestricted(ply)
    return ply:GetNetVar("restricted")  -- Use ply:IsRestricted() instead!
end

-- WRONG: Don't bypass meta methods
if ply:GetNetVar("raised") then  -- Use ply:IsWepRaised() instead!

-- WRONG: Don't create custom name getters
function GetPlayerName(ply)
    local char = ply:GetCharacter()
    return char and char:GetName() or "Unknown"  -- Use ply:Name() instead!
end
```

## Entity Meta Extensions

**Reference**: `gamemode/core/meta/sh_entity.lua`

### Common Entity Methods

```lua
-- Type checks
local isChair = entity:IsChair()  -- Is entity a chair
local isDoor = entity:IsDoor()  -- Is entity a door

-- Door operations (SERVER)
local locked = entity:IsLocked()  -- Is door/button locked
```

**Complete Example**:
```lua
-- Door interaction system
function PLUGIN:PlayerUse(client, entity)
    if entity:IsDoor() then
        if entity:IsLocked() then
            client:Notify("This door is locked!")
            return false
        end

        local doorOwner = entity:GetNetVar("owner")

        if doorOwner and doorOwner != client then
            client:Notify("You don't own this door!")
            return false
        end
    end

    if entity:IsChair() then
        local sitPos = entity:GetPos() + Vector(0, 0, 10)
        client:SetPos(sitPos)
        client:SetMoveType(MOVETYPE_NONE)

        timer.Simple(0.1, function()
            if IsValid(client) then
                client:ForceSequence("sit")
            end
        end)

        return false
    end
end

-- Lock/unlock door
function PLUGIN:ToggleDoorLock(client, entity)
    if not entity:IsDoor() then
        return false
    end

    local doorOwner = entity:GetNetVar("owner")

    if doorOwner != client then
        client:Notify("You don't own this door!")
        return false
    end

    if entity:IsLocked() then
        entity:Fire("unlock")
        client:Notify("Door unlocked")
    else
        entity:Fire("lock")
        client:Notify("Door locked")
    end
end

-- Chair placement validation
function PLUGIN:CanPlayerPlaceChair(client, model)
    if not CHAIR_CACHE[model] then
        -- Check if it's a chair using entity meta
        local tempEnt = ents.Create("prop_physics")
        tempEnt:SetModel(model)

        if tempEnt:IsChair() then
            CHAIR_CACHE[model] = true
        end

        tempEnt:Remove()
    end

    return CHAIR_CACHE[model]
end
```

## Character Meta Extensions

**Reference**: `gamemode/core/meta/sh_character.lua`

### Common Character Methods

```lua
-- Identity
local id = character:GetID()  -- Unique database ID
local name = character:GetName()  -- Character name
local desc = character:GetDescription()  -- Character description
local model = character:GetModel()  -- Character model

-- Faction & Class
local faction = character:GetFaction()  -- Faction ID
local class = character:GetClass()  -- Class ID

-- Inventory
local inv = character:GetInventory()  -- Main inventory

-- Money
local money = character:GetMoney()  -- Current money
character:GiveMoney(amount)  -- Add money
character:TakeMoney(amount)  -- Remove money

-- Data storage
local value = character:GetData(key, default)  -- Get persistent data
character:SetData(key, value)  -- Set persistent data

-- Attributes
local value = character:GetAttribute(key, default)  -- Get attribute value
character:SetAttribute(key, value)  -- Set attribute value

-- Flags
local hasFlag = character:HasFlags(flags)  -- Check if has flag(s)
character:GiveFlags(flags)  -- Grant flags
character:TakeFlags(flags)  -- Remove flags

-- Player access
local player = character:GetPlayer()  -- Get owning player

-- Saving (SERVER)
character:Save(callback)  -- Save to database
```

**Complete Example**:
```lua
-- Character info display
function PLUGIN:ShowCharacterInfo(viewer, target)
    local character = target:GetCharacter()

    if not character then
        return
    end

    local panel = vgui.Create("DFrame")
    panel:SetTitle(character:GetName())
    panel:SetSize(400, 300)
    panel:Center()

    local info = panel:Add("DLabel")
    info:SetText(string.format([[
Name: %s
Description: %s
Faction: %s
Money: %s
    ]],
        character:GetName(),
        character:GetDescription(),
        ix.faction.indices[character:GetFaction()].name,
        ix.currency.Get(character:GetMoney())
    ))
    info:Dock(FILL)
end

-- Give reward with notification
function PLUGIN:GiveReward(client, amount, itemID)
    local character = client:GetCharacter()

    if not character then
        return
    end

    -- Give money
    character:GiveMoney(amount)
    client:Notify("You received " .. ix.currency.Get(amount))

    -- Give item
    if itemID then
        character:GetInventory():Add(itemID, 1, nil, nil, function(item)
            if item then
                client:Notify("You received " .. item:GetName())
            end
        end)
    end

    -- Save character
    character:Save()
end

-- Skill system using attributes
function PLUGIN:InitCharacterAttributes()
    ix.char.RegisterVar("strength", {
        field = "strength",
        default = 0,
        isLocal = false,
        bSaveLoadInitialOnly = false
    })
end

function PLUGIN:IncreaseSkill(client, skill, amount)
    local character = client:GetCharacter()

    if not character then
        return
    end

    local currentValue = character:GetAttribute(skill, 0)
    character:SetAttribute(skill, currentValue + amount)

    client:Notify(string.format("%s increased to %d",
        skill,
        currentValue + amount
    ))
end

-- Flag-based permissions
function PLUGIN:CanAccessArea(client, area)
    local character = client:GetCharacter()

    if not character then
        return false
    end

    local requiredFlags = area.flags

    if requiredFlags and not character:HasFlags(requiredFlags) then
        client:Notify("You don't have permission to enter this area!")
        return false
    end

    return true
end
```

## Item Meta Extensions

**Reference**: `gamemode/core/meta/sh_item.lua`

### Common Item Methods

```lua
-- Identity
local id = item:GetID()  -- Unique instance ID
local name = item:GetName()  -- Item name
local desc = item:GetDescription()  -- Item description
local model = item:GetModel()  -- World model

-- Properties
local data = item:GetData(key, default)  -- Get item data
item:SetData(key, value, receivers)  -- Set item data

-- Inventory
local inv = item:GetInventory()  -- Parent inventory
local owner = item:GetOwner()  -- Inventory owner (player/entity)

-- Position
local x, y = item:GetGridPos()  -- Grid position in inventory
item:SetGridPos(x, y)  -- Set grid position

-- Hooks
item:Call(hookName, ...)  -- Call item hook
item:Hook(hookName, function)  -- Add hook function
```

**Complete Example**:
```lua
-- Durability system
ITEM.name = "Weapon"
ITEM.maxDurability = 100

function ITEM:GetDurability()
    return self:GetData("durability", self.maxDurability)
end

function ITEM:SetDurability(value)
    self:SetData("durability", math.max(0, value))
end

function ITEM:DamageWeapon(damage)
    local durability = self:GetDurability()
    self:SetDurability(durability - damage)

    if self:GetDurability() <= 0 then
        self:OnBroken()
    end
end

function ITEM:OnBroken()
    local owner = self:GetOwner()

    if IsValid(owner) and owner:IsPlayer() then
        owner:Notify(self:GetName() .. " has broken!")
    end

    self:Remove()
end

-- Container item with nested inventory
ITEM.name = "Backpack"
ITEM.invWidth = 4
ITEM.invHeight = 4

function ITEM:OnInstanced()
    -- Create nested inventory
    ix.inventory.New(0, self.invWidth, self.invHeight, function(inv)
        self:SetData("inventory", inv:GetID())
    end)
end

function ITEM:GetContainerInventory()
    local invID = self:GetData("inventory")

    if invID then
        return ix.inventory.instances[invID]
    end
end

-- Custom item action
ITEM.functions.Consume = {
    OnRun = function(item)
        local owner = item:GetOwner()

        if IsValid(owner) and owner:IsPlayer() then
            local character = owner:GetCharacter()

            if character then
                character:SetData("hunger", 0)
                owner:Notify("You ate " .. item:GetName())
            end
        end

        return true  -- Remove item
    end
}
```

## Inventory Meta Extensions

**Reference**: `gamemode/core/meta/sh_inventory.lua`

### Common Inventory Methods

```lua
-- Identity
local id = inv:GetID()  -- Unique inventory ID
local owner = inv:GetOwner()  -- Owner (player or entity)

-- Size
local width, height = inv:GetSize()  -- Grid dimensions

-- Items
local items = inv:GetItems()  -- Table of all items
local item = inv:GetItemAt(x, y)  -- Get item at position

-- Operations
inv:Add(uniqueID, quantity, data, x, y, callback)  -- Add item
inv:Remove(itemID, bNoDelete)  -- Remove item
inv:Transfer(itemID, targetInv, x, y, callback)  -- Move item

-- Checks
local canFit = inv:CanItemFit(x, y, width, height, item)  -- Check space
local hasItem = inv:HasItem(uniqueID, quantity)  -- Check for item
```

**Complete Example**:
```lua
-- Loot distribution system
function PLUGIN:DistributeLoot(chest, items)
    local invID = chest:GetNetVar("inventory")
    local inventory = ix.inventory.instances[invID]

    if not inventory then
        return
    end

    for _, itemData in ipairs(items) do
        inventory:Add(itemData.uniqueID, itemData.quantity or 1)
    end
end

-- Crafting system with inventory checks
function PLUGIN:CraftItem(client, recipe)
    local character = client:GetCharacter()
    local inventory = character:GetInventory()

    -- Check if player has materials
    for _, material in ipairs(recipe.materials) do
        if not inventory:HasItem(material.uniqueID, material.quantity) then
            client:Notify("Missing: " .. material.name)
            return false
        end
    end

    -- Remove materials
    for _, material in ipairs(recipe.materials) do
        local items = inventory:GetItems()

        local removed = 0
        for _, item in pairs(items) do
            if item.uniqueID == material.uniqueID and removed < material.quantity then
                item:Remove()
                removed = removed + 1
            end
        end
    end

    -- Give crafted item
    inventory:Add(recipe.result, 1, nil, nil, function(item)
        if item then
            client:Notify("Crafted: " .. item:GetName())
        end
    end)

    return true
end

-- Search container
function PLUGIN:SearchContainer(client, entity)
    local invID = entity:GetNetVar("inventory")
    local inventory = ix.inventory.instances[invID]

    if not inventory then
        client:Notify("This container is empty")
        return
    end

    local items = inventory:GetItems()
    local itemList = {}

    for _, item in pairs(items) do
        table.insert(itemList, item:GetName())
    end

    if #itemList > 0 then
        client:Notify("Contains: " .. table.concat(itemList, ", "))
    else
        client:Notify("Container is empty")
    end
end
```

## Best Practices

### ✅ DO

- Use meta methods instead of creating utility functions
- Check if player has character before accessing character methods
- Use `:IsValid()` checks before calling meta methods
- Use meta methods for consistent API across codebase
- Leverage built-in accessors for character data
- Use inventory meta methods for item operations
- Check return values from meta methods

### ❌ DON'T

- Don't bypass meta methods to access internal data directly
- Don't create custom utility functions that duplicate meta methods
- Don't forget to validate entities before calling meta methods
- Don't modify meta tables from plugins (use hooks instead)
- Don't assume player always has character
- Don't call server-only meta methods on client
- Don't create inconsistent naming conventions

## Common Patterns

### Pattern 1: Safe Character Access

```lua
-- Always check if character exists
function PLUGIN:DoSomething(client)
    local character = client:GetCharacter()

    if not character then
        return
    end

    -- Safe to use character methods now
    character:GiveMoney(100)
end
```

### Pattern 2: Entity Type Checking

```lua
-- Use meta methods for entity checks
function PLUGIN:PlayerUse(client, entity)
    if entity:IsDoor() then
        self:HandleDoor(client, entity)
    elseif entity:IsChair() then
        self:HandleChair(client, entity)
    end
end
```

### Pattern 3: Item Data Management

```lua
-- Use item meta for data storage
function ITEM:SetOwnerName(name)
    self:SetData("ownerName", name)
end

function ITEM:GetOwnerName()
    return self:GetData("ownerName", "Unknown")
end
```

## See Also

- [Character System](../systems/character.md) - Character management
- [Inventory System](../systems/inventory.md) - Inventory operations
- [Items System](../systems/items.md) - Item creation and usage
- [Player API](../api/player.md) - Full player method reference
- Garry's Mod Wiki: https://wiki.garrysmod.com/page/Category:Player
- Source: `gamemode/core/meta/sh_player.lua`
- Source: `gamemode/core/meta/sh_character.lua`
- Source: `gamemode/core/meta/sh_entity.lua`
- Source: `gamemode/core/meta/sh_item.lua`
- Source: `gamemode/core/meta/sh_inventory.lua`
