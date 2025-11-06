# Shipment Entity (ix_shipment)

> **Reference**: `entities/entities/ix_shipment.lua`

The `ix_shipment` entity represents ordered items that arrive in a delivery crate. It's used by the business system for players to order items in bulk.

## ⚠️ Important: Use Built-in Business System

**Always use the business system** rather than creating shipment entities manually. The framework provides:
- Automatic item ordering and delivery
- Payment processing
- Ownership tracking
- Timed delivery system
- Item extraction UI

## Core Concepts

### What is a Shipment Entity?

A shipment entity (`ix_shipment`) is a temporary crate that appears when players order items through the business system. Players can interact with the crate to retrieve ordered items either into their inventory or drop them in the world.

Key features:
- Contains multiple items
- Auto-despawns after 120 seconds (line 30-36)
- Shows countdown timer (3D2D display)
- Owned by specific character
- Plays break effects when removed
- Interactive UI for item extraction

### Key Properties

**Entity Variables** (line 11-13):
- `DeliveryTime` (Int): When the shipment will despawn

**Internal Properties**:
- `items` (Table): Dictionary of item uniqueIDs to quantities
- Owner tracked via NetVar "owner" (character ID)

## Creating Shipments

### Via Business System

**Reference**: `gamemode/core/libs/sh_business.lua:55-74`

The primary way shipments are created is through the business system:

```lua
-- Players use the business menu to order items
-- Framework automatically:
-- 1. Calculates total cost
-- 2. Takes money from character
-- 3. Creates shipment entity
-- 4. Associates shipment with character
-- 5. Starts 120 second timer
```

**Manual creation** (for advanced use):

```lua
-- Create a shipment entity
local shipment = ents.Create("ix_shipment")
shipment:Spawn()
shipment:SetPos(position)

-- Set items in the shipment
local items = {
    ["item_medkit"] = 5,      -- 5 medkits
    ["ammo_pistol"] = 10,     -- 10 pistol ammo
    ["food_bread"] = 3        -- 3 bread
}
shipment:SetItems(items)

-- Set owner
shipment:SetNetVar("owner", character:GetID())

-- Add to character's entity list (for cleanup on disconnect)
local charEnts = character:GetVar("charEnts") or {}
table.insert(charEnts, shipment)
character:SetVar("charEnts", charEnts, true)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Creating shipment without items or owner
local shipment = ents.Create("ix_shipment")
shipment:Spawn()  -- No items set, no owner!

-- WRONG: Setting invalid item IDs
shipment:SetItems({
    ["invalid_item"] = 5  -- Item doesn't exist!
})
```

## Entity Behavior

### Initialization

**Reference**: `entities/entities/ix_shipment.lua:16-37`

```lua
function ENT:Initialize()
    -- Sets model to item crate
    -- Sets up physics
    -- Sets delivery time to current time + 120 seconds
    -- Creates timer to remove after 120 seconds
end
```

### Player Interaction

**Reference**: `entities/entities/ix_shipment.lua:39-54`

When a player uses (presses E on) a shipment:

```lua
function ENT:Use(activator)
    -- Framework checks:
    -- 1. Is player the owner?
    -- 2. Can player open shipment? (CanPlayerOpenShipment hook)
    -- 3. Opens shipment UI
    -- 4. Shows available items
end
```

**Custom ownership check**:

```lua
-- Only the owner can open the shipment
hook.Add("CanPlayerOpenShipment", "MyShipmentCheck", function(client, shipment)
    local character = client:GetCharacter()

    if character and shipment:GetNetVar("owner", 0) == character:GetID() then
        return true  -- Allow
    end

    client:Notify("This shipment belongs to someone else!")
    return false  -- Deny
end)
```

### Item Extraction

**Reference**: `gamemode/core/libs/sh_business.lua:77-123`

Players can extract items in two ways:

1. **Add to Inventory**: Directly adds item to character's inventory
2. **Drop**: Spawns item entity near shipment

```lua
-- This happens automatically via the UI
-- When player clicks "Add" or "Drop" button:

net.Receive("ixShipmentUse", function(length, client)
    local uniqueID = net.ReadString()
    local drop = net.ReadBool()

    -- Validates ownership and distance
    -- Spawns item or adds to inventory
    -- Decrements item count
    -- Removes shipment when empty
end)
```

### Auto-Despawn Timer

**Reference**: `entities/entities/ix_shipment.lua:30-36`

```lua
-- Shipment automatically removes after 120 seconds
self:SetDeliveryTime(CurTime() + 120)

timer.Simple(120, function()
    if IsValid(self) then
        self:Remove()
    end
end)
```

**Customizing despawn time**:

```lua
-- In a schema or plugin
hook.Add("CreateShipment", "CustomShipmentTimer", function(client, shipment)
    -- Change timer to 5 minutes
    shipment:SetDeliveryTime(CurTime() + 300)

    -- Remove existing timer and create new one
    timer.Simple(300, function()
        if IsValid(shipment) then
            shipment:Remove()
        end
    end)
end)
```

## Client-Side Features

### 3D2D Countdown Display

**Reference**: `entities/entities/ix_shipment.lua:91-121`

The shipment displays a countdown timer on its sides:

```lua
function ENT:Draw()
    -- Shows delivery icon
    -- Shows remaining time until despawn
    -- Rendered on both sides of crate
    -- Uses 3D2D rendering
end
```

### Tooltip

**Reference**: `entities/entities/ix_shipment.lua:123-136`

```lua
function ENT:OnPopulateEntityInfo(container)
    -- Shows "Shipment" as name
    -- Shows owner's name
    -- Example: "Shipment for John Doe"
end
```

## Complete Examples

### Custom Business Order Hook

```lua
-- Modify shipment on creation
hook.Add("CreateShipment", "CustomShipment", function(client, shipment)
    local character = client:GetCharacter()

    -- Notify nearby players
    for _, ply in ipairs(ents.FindInSphere(shipment:GetPos(), 500)) do
        if ply:IsPlayer() then
            ply:ChatPrint(character:GetName() .. " received a shipment delivery!")
        end
    end

    -- Play delivery sound
    shipment:EmitSound("ambient/alarms/warningbell1.wav")

    -- Add visual effect
    local effect = EffectData()
    effect:SetOrigin(shipment:GetPos())
    effect:SetScale(5)
    util.Effect("Explosion", effect)
end)
```

### Restricting Business Items

```lua
-- Only allow certain items to be ordered
hook.Add("CanPlayerUseBusiness", "RestrictBusinessItems", function(client, itemID)
    local character = client:GetCharacter()

    -- Only cops can order weapons
    local weaponItems = {
        ["weapon_pistol"] = true,
        ["weapon_rifle"] = true
    }

    if weaponItems[itemID] then
        local faction = ix.faction.indices[client:Team()]

        if faction.uniqueID != "police" then
            client:Notify("Only police can order weapons!")
            return false
        end
    end

    -- Check if character has business license
    if not character:GetData("hasBusinessLicense", false) then
        client:Notify("You need a business license!")
        return false
    end

    return true
end)
```

### Tracking Shipment Items

```lua
-- Log what items are taken from shipments
hook.Add("ShipmentItemTaken", "LogShipmentItems", function(client, itemID, amount)
    local character = client:GetCharacter()

    ix.log.Add(client, "shipmentTake", itemID, amount)

    -- Track statistics
    local stats = character:GetData("shipmentStats", {})
    stats[itemID] = (stats[itemID] or 0) + 1
    character:SetData("shipmentStats", stats)
end)
```

### Shared Shipment System

```lua
-- Allow faction members to access shipments
hook.Add("CanPlayerOpenShipment", "FactionShipments", function(client, shipment)
    local ownerID = shipment:GetNetVar("owner", 0)
    local ownerChar = ix.char.loaded[ownerID]

    if not ownerChar then
        return false
    end

    local character = client:GetCharacter()

    if not character then
        return false
    end

    -- Allow same faction members
    if ownerChar:GetFaction() == character:GetFaction() then
        return true
    end

    return false
end)
```

### Emergency Shipment Spawn

```lua
-- Admin command to spawn shipment for player
ix.command.Add("GiveShipment", {
    description = "Spawn a shipment for a player.",
    adminOnly = true,
    arguments = ix.type.player,
    OnRun = function(self, client, target)
        local character = target:GetCharacter()

        if not character then
            return "@charNoExist"
        end

        -- Create shipment
        local shipment = ents.Create("ix_shipment")
        shipment:Spawn()
        shipment:SetPos(target:GetItemDropPos(shipment))

        -- Add items
        local items = {
            ["item_medkit"] = 5,
            ["food_bread"] = 10
        }
        shipment:SetItems(items)
        shipment:SetNetVar("owner", character:GetID())

        -- Track for cleanup
        local charEnts = character:GetVar("charEnts") or {}
        table.insert(charEnts, shipment)
        character:SetVar("charEnts", charEnts, true)

        target:Notify("You received an emergency shipment!")

        return "@cmdGiveShipment", target:Name()
    end
})
```

## Best Practices

### ✅ DO

- Let players use the business system naturally
- Validate ownership before allowing interaction
- Track shipments in character's `charEnts` for cleanup
- Use hooks to customize shipment behavior
- Check distance before allowing item extraction
- Clean up shipments when character disconnects

### ❌ DON'T

- Don't create shipments without setting items
- Don't forget to set owner NetVar
- Don't bypass the business payment system
- Don't allow infinite items from shipments
- Don't forget the auto-despawn timer
- Don't let players access other players' shipments without reason

## Common Patterns

### Delivery Delay System

```lua
-- Add delivery delay before shipment arrives
hook.Add("CreateShipment", "DelayedDelivery", function(client, shipment)
    -- Hide shipment initially
    shipment:SetNoDraw(true)
    shipment:SetNotSolid(true)

    -- Store original position
    local targetPos = shipment:GetPos()

    -- Move it out of the map temporarily
    shipment:SetPos(Vector(0, 0, -10000))

    -- Deliver after 30 seconds
    timer.Simple(30, function()
        if IsValid(shipment) then
            shipment:SetPos(targetPos)
            shipment:SetNoDraw(false)
            shipment:SetNotSolid(false)

            -- Notify owner
            local owner = ix.char.loaded[shipment:GetNetVar("owner", 0)]

            if owner then
                local player = owner:GetPlayer()

                if IsValid(player) then
                    player:Notify("Your shipment has arrived!")
                end
            end
        end
    end)
end)
```

### Item Count Display

```lua
-- Show total item count in tooltip
hook.Add("OnPopulateEntityInfo", "ShipmentItemCount", function(entity, container)
    if entity:GetClass() == "ix_shipment" then
        local count = entity:GetItemCount()

        local row = container:AddRow("items")
        row:SetText("Total Items: " .. count)
        row:SizeToContents()
    end
end)
```

## Common Issues

### Shipment Disappears Too Quickly

**Cause**: 120 second timer expires
**Fix**: Extend the timer or remove auto-despawn

```lua
-- Extend to 10 minutes
hook.Add("CreateShipment", "LongerShipments", function(client, shipment)
    shipment:SetDeliveryTime(CurTime() + 600)
end)
```

### Can't Open Shipment

**Cause**: Ownership mismatch or hook blocking
**Fix**: Check owner NetVar and hooks

```lua
-- Debug ownership
print("Shipment owner:", shipment:GetNetVar("owner", 0))
print("Character ID:", character:GetID())
```

### Items Disappear From Shipment

**Cause**: Multiple players accessing or network desync
**Fix**: Validate on server, use proper networking

```lua
-- Server-side validation happens automatically
-- Don't modify shipment.items on client
```

### Shipment Spawns Inside Player

**Cause**: Poor spawn position calculation
**Fix**: Use `client:GetItemDropPos()`

```lua
-- Proper positioning
shipment:SetPos(client:GetItemDropPos(shipment))
```

## See Also

- [Business System](../systems/business.md) - Business and ordering system (if documented)
- [Item System](../systems/items.md) - Core item system
- [Character System](../systems/character.md) - Character ownership
- [ix_item Entity](ix_item.md) - Item entity documentation
- Source: `entities/entities/ix_shipment.lua`
- Source: `gamemode/core/libs/sh_business.lua`
