# Inventory Configuration

> **Reference**: `gamemode/config/sh_config.lua:74-184`

Configuration options for inventory dimensions, item interaction, weight limits, and physics-based item manipulation.

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Get()` for inventory settings** rather than hardcoding inventory behavior. The framework provides:
- Centralized inventory configuration
- Admin-adjustable limits and timings
- Automatic validation of inventory operations
- Consistent item interaction mechanics
- Physics-based item handling

## Inventory Dimensions

These settings define the default size of character inventories.

### inventoryWidth

**Reference**: `gamemode/config/sh_config.lua:74`

```lua
ix.config.Add("inventoryWidth", 6, "How many slots in a row there is in a default inventory.", nil, {
    data = {min = 0, max = 20},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `6`
**Range**: 0 - 20 slots
**Description**: Number of horizontal slots in default character inventories

---

### inventoryHeight

**Reference**: `gamemode/config/sh_config.lua:78`

```lua
ix.config.Add("inventoryHeight", 4, "How many slots in a column there is in a default inventory.", nil, {
    data = {min = 0, max = 20},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `4`
**Range**: 0 - 20 slots
**Description**: Number of vertical slots in default character inventories

**Usage**:
```lua
-- Create inventory with configured dimensions
local width = ix.config.Get("inventoryWidth", 6)
local height = ix.config.Get("inventoryHeight", 4)

ix.inventory.Create(width, height, characterID, function(inventory)
    if inventory then
        character:SetInventory(inventory)
    end
end)
```

---

## Item Interaction

### itemPickupTime

**Reference**: `gamemode/config/sh_config.lua:181`

```lua
ix.config.Add("itemPickupTime", 0.5, "How long it takes to pick up and put an item in your inventory.", nil, {
    data = {min = 0, max = 5, decimals = 1},
    category = "interaction"
})
```

**Type**: Number (Float)
**Default**: `0.5` seconds
**Range**: 0.0 - 5.0 seconds
**Description**: Delay in seconds before an item is picked up and added to inventory

**Usage**:
```lua
-- Use configured pickup time
function PLUGIN:StartItemPickup(client, item)
    local pickupTime = ix.config.Get("itemPickupTime", 0.5)

    client:SetAction("Picking up item...", pickupTime, function()
        -- Add item to inventory
        item:Transfer(client:GetCharacter():GetInventory())
    end)
end
```

---

## Physics Interaction

### maxHoldWeight

**Reference**: `gamemode/config/sh_config.lua:170`

```lua
ix.config.Add("maxHoldWeight", 100, "The maximum weight that a player can carry in their hands.", nil, {
    data = {min = 1, max = 500},
    category = "interaction"
})
```

**Type**: Number (Integer)
**Default**: `100`
**Range**: 1 - 500 (weight units)
**Description**: Maximum weight of entities that can be picked up with the use key

**Usage**:
```lua
-- Check if player can hold entity
function PLUGIN:CanPlayerHoldEntity(client, entity)
    local weight = entity:GetPhysicsObject():GetMass()
    local maxWeight = ix.config.Get("maxHoldWeight", 100)

    if weight > maxWeight then
        client:Notify("This object is too heavy to carry!")
        return false
    end

    return true
end
```

---

### throwForce

**Reference**: `gamemode/config/sh_config.lua:174`

```lua
ix.config.Add("throwForce", 732, "How hard a player can throw the item that they're holding.", nil, {
    data = {min = 0, max = 8192},
    category = "interaction"
})
```

**Type**: Number (Integer)
**Default**: `732`
**Range**: 0 - 8192 force units
**Description**: Force applied when throwing held entities

**Usage**:
```lua
-- Throw entity with configured force
function PLUGIN:PlayerThrowEntity(client, entity)
    local force = ix.config.Get("throwForce", 732)
    local direction = client:GetAimVector()

    local phys = entity:GetPhysicsObject()
    if IsValid(phys) then
        phys:ApplyForceCenter(direction * force)
    end
end
```

---

### allowPush

**Reference**: `gamemode/config/sh_config.lua:178`

```lua
ix.config.Add("allowPush", true, "Whether or not pushing with hands is allowed.", nil, {
    category = "interaction"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Enables or disables player-to-player pushing with the use key

**Usage**:
```lua
-- Check if pushing is enabled
function PLUGIN:CanPlayerPushPlayer(client, target)
    if not ix.config.Get("allowPush", true) then
        return false
    end

    -- Additional checks...
    return true
end
```

---

## Complete Example: Custom Inventory System

```lua
-- Create faction-specific inventory with size modifiers
function PLUGIN:OnCharacterCreated(client, character)
    local faction = ix.faction.Get(character:GetFaction())

    -- Get base dimensions
    local baseWidth = ix.config.Get("inventoryWidth", 6)
    local baseHeight = ix.config.Get("inventoryHeight", 4)

    -- Modify based on faction
    local width = baseWidth
    local height = baseHeight

    if faction.inventoryMultiplier then
        width = math.floor(baseWidth * faction.inventoryMultiplier)
        height = math.floor(baseHeight * faction.inventoryMultiplier)
    end

    -- Create inventory
    ix.inventory.Create(width, height, character:GetID(), function(inventory)
        character:SetInventory(inventory)
    end)
end

-- Item pickup with progress bar
function PLUGIN:PlayerUse(client, entity)
    if not entity.ixItemID then return end

    local pickupTime = ix.config.Get("itemPickupTime", 0.5)

    -- Show progress bar
    client:SetAction("Picking up " .. entity.ixItemName, pickupTime, function()
        local inventory = client:GetCharacter():GetInventory()

        -- Try to add to inventory
        local item = ix.item.instances[entity.ixItemID]
        if item then
            item:Transfer(inventory:GetID(), nil, nil, client)
        end
    end)

    return false
end

-- Weight-based carry system
function PLUGIN:KeyPress(client, key)
    if key != IN_USE then return end

    local trace = client:GetEyeTrace()
    local entity = trace.Entity

    if not IsValid(entity) or not entity:GetPhysicsObject():IsValid() then
        return
    end

    -- Check weight limit
    local weight = entity:GetPhysicsObject():GetMass()
    local maxWeight = ix.config.Get("maxHoldWeight", 100)

    if weight > maxWeight then
        client:Notify("Too heavy to carry (limit: " .. maxWeight .. " kg)")
        return
    end

    -- Pick up entity
    client:PickupObject(entity)
end

-- Throw held entity
function PLUGIN:KeyRelease(client, key)
    if key != IN_ATTACK then return end

    local entity = client:GetCarriedEntity()
    if not IsValid(entity) then return end

    -- Drop with throw force
    client:DropObject()

    local force = ix.config.Get("throwForce", 732)
    local direction = client:GetAimVector()

    local phys = entity:GetPhysicsObject()
    if IsValid(phys) then
        phys:SetVelocity(direction * force)
    end
end

-- Player push system
function PLUGIN:KeyPress(client, key)
    if key != IN_USE then return end

    if not ix.config.Get("allowPush", true) then
        return
    end

    local trace = client:GetEyeTrace()
    local target = trace.Entity

    if IsValid(target) and target:IsPlayer() then
        -- Push target player
        local force = client:GetAimVector() * 300
        target:SetVelocity(force)
    end
end

-- Dynamic inventory size based on class
function PLUGIN:CharacterLoaded(character)
    local class = ix.class.Get(character:GetClass())

    if class and class.inventorySize then
        -- Custom inventory size for class
        local width, height = class.inventorySize[1], class.inventorySize[2]

        ix.inventory.Create(width, height, character:GetID(), function(inventory)
            character:SetInventory(inventory)
        end)
    else
        -- Use default size
        local width = ix.config.Get("inventoryWidth", 6)
        local height = ix.config.Get("inventoryHeight", 4)

        ix.inventory.Create(width, height, character:GetID(), function(inventory)
            character:SetInventory(inventory)
        end)
    end
end
```

## Best Practices

### ✅ DO

- Use `ix.config.Get()` for all inventory dimensions
- Check `itemPickupTime` for pickup delays
- Validate weight against `maxHoldWeight`
- Check `allowPush` before pushing players
- Use `throwForce` for consistent throw mechanics
- Create inventories with configured dimensions
- Provide feedback when weight limits exceeded
- Allow admins to adjust timings and limits

### ❌ DON'T

- Don't hardcode inventory dimensions
- Don't bypass pickup timings
- Don't ignore weight limits
- Don't allow pushing when disabled
- Don't use custom throw forces
- Don't create oversized inventories without validation
- Don't skip interaction checks
- Don't forget to provide user feedback

## Common Patterns

### Pattern 1: Faction-Specific Inventories

```lua
-- Different inventory sizes per faction
FACTION.inventoryMultiplier = 1.5

-- In character creation
function CreateFactionInventory(character)
    local faction = ix.faction.Get(character:GetFaction())
    local width = ix.config.Get("inventoryWidth", 6)
    local height = ix.config.Get("inventoryHeight", 4)

    if faction.inventoryMultiplier then
        width = math.ceil(width * faction.inventoryMultiplier)
        height = math.ceil(height * faction.inventoryMultiplier)
    end

    ix.inventory.Create(width, height, character:GetID())
end
```

### Pattern 2: Progressive Pickup System

```lua
-- Show progress bar for item pickup
function PickupItemWithProgress(client, item)
    local pickupTime = ix.config.Get("itemPickupTime", 0.5)

    client:SetAction("@pickingUp", pickupTime, function()
        -- Verify still valid
        if IsValid(item) then
            item:Transfer(client:GetCharacter():GetInventory():GetID())
        end
    end, function()
        -- Cancelled
        client:Notify("Pickup cancelled")
    end)
end
```

### Pattern 3: Strength-Based Weight

```lua
-- Modify max weight based on character strength
function GetEffectiveMaxWeight(character)
    local baseWeight = ix.config.Get("maxHoldWeight", 100)
    local strength = character:GetAttribute("strength", 0)

    -- 1% increase per strength point
    local modifier = 1 + (strength * 0.01)

    return math.floor(baseWeight * modifier)
end

-- Check if player can carry
function CanCarryEntity(client, entity)
    local weight = entity:GetPhysicsObject():GetMass()
    local maxWeight = GetEffectiveMaxWeight(client:GetCharacter())

    return weight <= maxWeight
end
```

## Common Issues

### Issue: Inventory Too Small

**Cause**: Default dimensions may not fit all items
**Fix**: Adjust config or use faction/class specific sizes

```lua
-- Larger inventory for specific faction
if character:GetFaction() == FACTION_MERCHANT then
    local width = ix.config.Get("inventoryWidth", 6) * 2
    local height = ix.config.Get("inventoryHeight", 4) * 2
    ix.inventory.Create(width, height, character:GetID())
end
```

### Issue: Items Pickup Instantly

**Cause**: Not implementing pickup delay
**Fix**: Use configured pickup time

```lua
-- Always use pickup delay
function PickupItem(client, item)
    local delay = ix.config.Get("itemPickupTime", 0.5)

    client:SetAction("@pickingUp", delay, function()
        item:Transfer(client:GetCharacter():GetInventory():GetID())
    end)
end
```

### Issue: Players Carry Heavy Objects

**Cause**: Not checking weight limit
**Fix**: Validate weight before allowing pickup

```lua
-- Check weight limit
function PLUGIN:OnEntityPickup(client, entity)
    local phys = entity:GetPhysicsObject()
    if not IsValid(phys) then return end

    local weight = phys:GetMass()
    local maxWeight = ix.config.Get("maxHoldWeight", 100)

    if weight > maxWeight then
        client:Notify("Too heavy!")
        return false
    end
end
```

## Adding Custom Inventory Options

To add your own inventory options in schemas or plugins:

```lua
-- Add custom inventory config
ix.config.Add("maxInventoryWeight", 50, "Maximum weight inventory can hold", nil, {
    category = "interaction",
    data = {min = 10, max = 500}
})

-- Add custom pickup config
ix.config.Add("quickPickup", false, "Allow instant pickup for small items", nil, {
    category = "interaction"
})

-- Use in code
function CanPickupQuickly(item)
    if not ix.config.Get("quickPickup", false) then
        return false
    end

    return item.weight and item.weight < 1
end
```

## See Also

- [Configuration System](../systems/configuration.md) - Overview of config system
- [Inventory System](../systems/inventory.md) - Inventory management
- [Items System](../systems/items.md) - Item types and usage
- [Character Configuration](characters.md) - Character settings
- [Storage Library](../libraries/storage.md) - Storage containers
- Source: `gamemode/config/sh_config.lua`
