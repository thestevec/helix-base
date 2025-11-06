# Item Entity (ix_item)

> **Reference**: `entities/entities/ix_item.lua`

The `ix_item` entity is the physical world representation of items in the Helix framework. It spawns when items are dropped, removed from inventories, or spawned in the world.

## ⚠️ Important: Use Built-in Item Spawning

**Always use `ix.item.Spawn()`** rather than creating ix_item entities directly. The framework provides:
- Automatic item instance creation
- Database persistence
- Network synchronization
- Proper physics initialization
- Item data management

## Core Concepts

### What is an Item Entity?

An item entity (`ix_item`) is a physical entity that represents an item object in the game world. When players drop items, spawn them via commands, or remove them from their inventory, the framework creates these entities automatically.

Key features:
- Visual representation using item's model
- Physics-based interaction
- Automatic pickup handling
- Item data synchronization
- Tooltip display on mouse-over
- Health system (can be damaged/destroyed)

### Key Properties

**Entity Variables**:
- `ItemID` (String): The unique identifier of the item type (e.g., "weapon_pistol")
- `ixItemID` (Number): The database ID of this specific item instance

**Internal Properties** (lines 28-36):
- `health`: Default 50, can be damaged and destroyed
- Model, skin, and material from item definition
- Physics object for world interaction

## Spawning Item Entities

### Using ix.item.Spawn()

**Reference**: `gamemode/core/libs/sh_item.lua:812`

```lua
-- Spawn an item in the world
ix.item.Spawn(uniqueID, position, callback, angles, data)
```

**Complete Working Example**:

```lua
-- Spawn a medkit at a specific position
local spawnPos = Vector(100, 200, 50)

ix.item.Spawn("item_medkit", spawnPos, function(item, entity)
    if IsValid(entity) then
        print("Spawned medkit with ID: " .. item:GetID())
        -- Item entity is now in the world
        -- Players can walk up and pick it up
    end
end)

-- Spawn with custom angle and data
local customData = {quality = 95}
ix.item.Spawn("weapon_pistol", spawnPos, nil, Angle(0, 90, 0), customData)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create ix_item entities manually
local ent = ents.Create("ix_item")
ent:SetPos(position)
ent:Spawn()  -- This won't have item data, inventory linking, or proper setup!
```

### From Item Instance

**Reference**: `entities/entities/ix_item.lua:58-96`

Items can also spawn their own entities:

```lua
-- Get an item instance and spawn its entity
local item = character:GetInventory():GetItems()[1]
if item then
    local entity = item:Spawn(Vector(100, 200, 50), Angle(0, 0, 0))

    if IsValid(entity) then
        print("Item entity spawned!")
    end
end
```

## Entity Behavior

### Player Interaction

**Reference**: `entities/entities/ix_item.lua:38-56`

When a player uses (presses E on) an item entity:

```lua
function ENT:Use(activator, caller)
    -- Framework automatically:
    -- 1. Gets the item table
    -- 2. Checks if player can take it (functions.take.OnCanRun)
    -- 3. Shows interaction progress bar
    -- 4. Adds to player's inventory
    -- 5. Removes entity from world
end
```

**Customizing pickup behavior in your item**:

```lua
ITEM.functions.take = {
    OnCanRun = function(itemTable)
        local entity = itemTable.entity
        local player = itemTable.player

        -- Custom logic: Only allow pickup during daytime
        if ix.date.GetHour() < 6 or ix.date.GetHour() > 18 then
            player:Notify("You can only pick this up during the day!")
            return false
        end

        return true
    end
}
```

### Entity Menu (Right-Click)

**Reference**: `entities/entities/ix_item.lua:269-317`

Players can right-click item entities to see available functions:

```lua
-- In your item definition
ITEM.functions.Examine = {
    name = "Examine",
    OnCanRun = function(itemTable)
        return true  -- Always allow examination
    end,
    OnClick = function(itemTable)
        local entity = itemTable.entity
        local player = itemTable.player

        player:ChatPrint("You examine the " .. itemTable.name)
        return false  -- Don't send to server (client-side only)
    end
}
```

### Damage and Destruction

**Reference**: `entities/entities/ix_item.lua:107-123`

Item entities can take damage and be destroyed:

```lua
function ENT:OnTakeDamage(damageInfo)
    -- Items have 50 health by default
    -- When health reaches 0:
    -- 1. Plays break sound
    -- 2. Shows glass impact effect
    -- 3. Calls item.OnDestroyed callback
    -- 4. Logs the destruction
    -- 5. Removes from database
end
```

**Handling destruction in your item**:

```lua
-- In your item definition
function ITEM:OnDestroyed(entity)
    -- Create explosion, spawn fragments, etc.
    local pos = entity:GetPos()

    local effect = EffectData()
    effect:SetOrigin(pos)
    util.Effect("Explosion", effect)

    -- Spawn smaller items
    ix.item.Spawn("item_scrap", pos + Vector(0, 0, 10))
end

function ITEM:OnEntityTakeDamage(entity, damageInfo)
    -- Return false to prevent normal damage behavior
    if damageInfo:GetDamage() < 10 then
        return false  -- Immune to small damage
    end

    -- Return nothing to allow normal damage
end
```

### Entity Created Callback

**Reference**: `entities/entities/ix_item.lua:92-94`

```lua
-- In your item definition
function ITEM:OnEntityCreated(entity)
    -- Called when the item entity is created
    -- Useful for special effects, custom physics, etc.

    entity:SetColor(Color(255, 0, 0))  -- Make it red
    entity:SetMaterial("models/wireframe")  -- Wireframe material

    -- Start a timer
    timer.Simple(30, function()
        if IsValid(entity) then
            entity:Remove()  -- Remove after 30 seconds
        end
    end)
end
```

### Entity Removal

**Reference**: `entities/entities/ix_item.lua:125-156`

```lua
function ENT:OnRemove()
    -- Framework automatically:
    -- 1. Plays sound and effect if destroyed
    -- 2. Calls item.OnRemoved callback
    -- 3. Deletes from database (if not safe removal)
    -- 4. Logs destruction if damaged
end
```

**Safe vs. Unsafe Removal**:

```lua
-- Safe removal (picking up item) - preserves database entry
entity.ixIsSafe = true
entity:Remove()

-- Unsafe removal (destroying item) - deletes from database
entity:Remove()  -- Default behavior
```

## Client-Side Features

### Tooltip Display

**Reference**: `entities/entities/ix_item.lua:186-254`

When players look at item entities, they see tooltips:

```lua
function ENT:OnPopulateEntityInfo(tooltip)
    -- Framework automatically:
    -- 1. Shows item name with rarity color
    -- 2. Displays item description
    -- 3. Shows grid size visualization
    -- 4. Displays custom tooltip rows from item
end
```

**Customizing tooltips in your item**:

```lua
function ITEM:PopulateTooltip(tooltip)
    local entity = self.entity

    if entity then
        local row = tooltip:AddRow("condition")
        row:SetText("Condition: " .. (self:GetData("condition", 100)) .. "%")
        row:SizeToContents()
    end
end
```

### Custom Rendering

**Reference**: `entities/entities/ix_item.lua:256-266`

```lua
-- In your item definition
function ITEM:DrawEntity(entity)
    -- Custom rendering for your item
    -- Called every frame on client

    -- Example: Glow effect
    render.SetColorModulation(1, 0.5, 0)
    entity:DrawModel()
    render.SetColorModulation(1, 1, 1)
end
```

## Complete Example: Custom Item with Entity Behavior

```lua
ITEM.name = "Explosive Barrel"
ITEM.description = "A dangerous barrel that explodes when damaged."
ITEM.model = "models/props_c17/oildrum001_explosive.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.health = 30  -- Less durable than normal items

function ITEM:OnEntityCreated(entity)
    -- Make it more obvious this is dangerous
    entity:SetColor(Color(255, 50, 50))
    entity.health = self.health or 30
end

function ITEM:OnEntityTakeDamage(entity, damageInfo)
    -- Explode when health is low
    if entity:Health() <= 10 then
        local pos = entity:GetPos()

        -- Create explosion
        local explode = ents.Create("env_explosion")
        explode:SetPos(pos)
        explode:SetKeyValue("iMagnitude", "100")
        explode:Spawn()
        explode:Fire("Explode")

        -- Damage nearby players
        for _, ply in ipairs(ents.FindInSphere(pos, 200)) do
            if ply:IsPlayer() then
                ply:TakeDamage(50, damageInfo:GetAttacker(), entity)
            end
        end

        entity.ixIsDestroying = true
        entity:Remove()

        return false  -- Prevent normal damage behavior
    end
end

function ITEM:Think(entity)
    -- Custom think logic - beep when damaged
    if entity:Health() < entity.health / 2 then
        if not entity.nextBeep or entity.nextBeep < CurTime() then
            entity:EmitSound("buttons/button17.wav")
            entity.nextBeep = CurTime() + 1
        end
    end
end
```

## Best Practices

### ✅ DO

- Use `ix.item.Spawn()` to create item entities
- Let the framework handle pickup logic
- Use item callbacks (`OnEntityCreated`, `OnDestroyed`, etc.)
- Check entity validity before operations
- Use `entity.ixIsSafe = true` when preserving items
- Implement `OnCanRun` in item functions for custom logic

### ❌ DON'T

- Don't create `ix_item` entities with `ents.Create()`
- Don't modify `entity.ixItemID` manually
- Don't delete items from database directly
- Don't forget to validate entity and item existence
- Don't bypass `item.functions.take` logic
- Don't access `ix.item.instances` directly without checks

## Common Patterns

### Spawning Multiple Items

```lua
-- Spawn a pile of items
local items = {"item_medkit", "ammo_pistol", "food_bread"}
local basePos = Vector(100, 200, 50)

for i, uniqueID in ipairs(items) do
    local offset = Vector(i * 20, 0, 0)

    ix.item.Spawn(uniqueID, basePos + offset, function(item, entity)
        print("Spawned: " .. item.name)
    end)
end
```

### Temporary Item Entities

```lua
-- Spawn an item that disappears after 60 seconds
ix.item.Spawn("item_healthvial", position, function(item, entity)
    if IsValid(entity) then
        timer.Simple(60, function()
            if IsValid(entity) then
                entity.ixIsSafe = true  -- Don't log as destroyed
                entity:Remove()
                item:Remove()  -- Remove item instance too
            end
        end)
    end
end)
```

### Converting Dropped Items

```lua
-- When player drops an item
function ITEM:OnDrop(item, position)
    -- Item has been dropped, entity will be created
    -- This is called BEFORE entity creation
end

-- Access the entity after it's created
hook.Add("OnItemSpawned", "MyItemSpawnedHook", function(item, entity)
    if item.uniqueID == "special_item" then
        -- Do something with the entity
        entity:SetColor(Color(255, 215, 0))
    end
end)
```

## Common Issues

### Item Entity Has No Item Data

**Cause**: Entity created without proper `SetItem()` call
**Fix**: Always use `ix.item.Spawn()` instead of manual entity creation

```lua
-- WRONG
local ent = ents.Create("ix_item")
ent:Spawn()

-- CORRECT
ix.item.Spawn("item_medkit", position)
```

### Item Duplicates When Picked Up

**Cause**: Not marking entity as safe during inventory operations
**Fix**: Set `ixIsSafe` flag before removal

```lua
-- When moving item to inventory
entity.ixIsSafe = true
entity:Remove()
```

### Entity Invalid in Callback

**Cause**: Trying to use entity before it's created
**Fix**: Use the callback parameter

```lua
-- WRONG
ix.item.Spawn("item", pos)
-- entity doesn't exist yet!

-- CORRECT
ix.item.Spawn("item", pos, function(item, entity)
    if IsValid(entity) then
        -- Now entity exists
        entity:SetColor(Color(255, 0, 0))
    end
end)
```

### Item Disappears on Server Restart

**Cause**: `bNoPersist = true` means entities don't save
**Fix**: Items should be in inventories for persistence. World items are temporary by design.

```lua
-- Items in the world are meant to be temporary
-- For persistent storage, use:
-- 1. Player inventories
-- 2. Storage containers
-- 3. Custom persistence plugin
```

## See Also

- [Item System](../systems/items.md) - Core item system documentation
- [Inventory System](../systems/inventory.md) - Inventory management
- [Storage Library](../libraries/storage.md) - Persistent item storage
- [ix_money Entity](ix_money.md) - Money entity documentation
- Source: `entities/entities/ix_item.lua`
