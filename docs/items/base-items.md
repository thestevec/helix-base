# Base Items

> **Reference**: `gamemode/items/base/`

The base items directory contains the foundational item types that ship with Helix. These serve as parent classes for creating custom items in your schema.

## ⚠️ Important: Use Base Items as Templates

**Always extend from Helix's base items** rather than creating item definitions from scratch. The framework provides:
- Automatic database persistence
- Network synchronization
- Built-in equip/unequip functionality
- Inventory management integration
- PAC3 support (for applicable items)

## Available Base Items

Helix provides five base item types:

| Base Item | File | Purpose |
|-----------|------|---------|
| **Weapons** | `sh_weapons.lua` | Equippable weapons with ammo persistence |
| **Ammunition** | `sh_ammo.lua` | Ammo boxes that add rounds to player |
| **Bags** | `sh_bags.lua` | Container items with their own inventory |
| **Outfits** | `sh_outfit.lua` | Model-changing clothing items |
| **PAC Outfits** | `sh_pacoutfit.lua` | PAC3-based attachments and accessories |

## Core Concepts

### What are Base Items?

Base items are template item definitions that provide common functionality. When you create a custom item, you inherit from one of these bases:

```lua
-- Your custom item inherits from a base
ITEM.base = "base_bags"  -- Extends the bags base item
```

### Item Properties

All base items share these common properties:

- `name` - Display name
- `description` - Item description
- `model` - World/inventory model
- `width` - Inventory width (grid cells)
- `height` - Inventory height (grid cells)
- `category` - Grouping category

### Item Functions

Items can have interactive functions that appear in context menus:

```lua
ITEM.functions.Use = {
    name = "use",
    OnRun = function(item)
        -- Function logic
        return true  -- Remove item after use
    end,
    OnCanRun = function(item)
        return true  -- Can this function be run?
    end
}
```

## Using Base Items

### Extending a Base Item

**Reference**: All base item files in `gamemode/items/base/`

To create a custom item, create a new file in your schema's `items/` directory:

```lua
-- schema/items/sh_medkit.lua
ITEM.name = "Medical Kit"
ITEM.description = "Restores health when used."
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Medical"
ITEM.base = "base_bags"  -- Extend the bags base

ITEM.functions.Use = {
    OnRun = function(item)
        item.player:SetHealth(100)
        item.player:Notify("You used the medkit.")
        return true  -- Remove after use
    end
}
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create items without a base
ITEM.name = "Medkit"
-- Missing ITEM.base = "base_..."
-- This won't have proper inventory integration!
```

### Item Data Persistence

**Reference**: `gamemode/items/base/sh_weapons.lua:173`

Use `SetData()` and `GetData()` to store persistent item-specific data:

```lua
-- Store data (persists to database)
item:SetData("ammo", weapon:Clip1())
item:SetData("equip", true)

-- Retrieve data
local ammo = item:GetData("ammo", 0)  -- 0 is default if not set
local equipped = item:GetData("equip")
```

This data automatically saves to the database and syncs to clients.

### Item Hooks

**Reference**: `gamemode/items/base/sh_weapons.lua:32`

Items can hook into events using `ITEM:Hook()`:

```lua
ITEM:Hook("drop", function(item)
    -- Called when item is dropped
    if item:GetData("equip") then
        item:SetData("equip", nil)
    end
end)
```

Available hooks:
- `drop` - Item dropped from inventory
- `take` - Item picked up
- `transfer` - Item moved between inventories

## Item Lifecycle

### Creation

When an item is created:

1. `OnInstanced(invID, x, y)` - Item created in inventory
2. `OnRegistered()` - Item type registered (once per item type)

### Loading

When a player connects:

1. Items loaded from database
2. `OnLoadout()` - Restore equipped items (weapons, outfits)
3. `OnSendData()` - Send item data to client

### Saving

**Reference**: `gamemode/items/base/sh_weapons.lua:264`

When character saves:

```lua
function ITEM:OnSave()
    local weapon = self.player:GetWeapon(self.class)

    if IsValid(weapon) and self:GetData("equip") then
        -- Save current ammo before saving
        self:SetData("ammo", weapon:Clip1())
    end
end
```

### Removal

**Reference**: `gamemode/items/base/sh_weapons.lua:272`

When item is permanently deleted:

```lua
function ITEM:OnRemoved()
    local inventory = ix.item.inventories[self.invID]
    local owner = inventory.GetOwner and inventory:GetOwner()

    if IsValid(owner) and owner:IsPlayer() then
        -- Clean up equipped item
        local weapon = owner:GetWeapon(self.class)
        if IsValid(weapon) then
            weapon:Remove()
        end
    end
end
```

## Best Practices

### ✅ DO

- Always set `ITEM.base` to extend a base item type
- Use `GetData()`/`SetData()` for persistent item properties
- Implement `OnCanRun` to validate item function usage
- Return `true` from `OnRun` to remove item after use
- Return `false` from `OnRun` to keep item in inventory
- Check `IsValid(item.player)` before accessing player
- Use `item:GetOwner()` to get the owning player

### ❌ DON'T

- Don't create items without extending a base type
- Don't store item data in global tables
- Don't manually save item data to database
- Don't forget to validate player/item state
- Don't access `item.player` without checking validity
- Don't implement custom networking for item data

## Common Patterns

### Pattern 1: Consumable Item

```lua
ITEM.name = "Food"
ITEM.model = "models/food.mdl"
ITEM.base = "base_consumable"  -- If available, or no base for simple items

ITEM.functions.Eat = {
    OnRun = function(item)
        item.player:SetHealth(math.min(item.player:Health() + 20, 100))
        return true  -- Remove after eating
    end,
    OnCanRun = function(item)
        return IsValid(item.player) and item.player:Health() < 100
    end
}
```

### Pattern 2: Item with Durability

```lua
ITEM.functions.Use = {
    OnRun = function(item)
        local uses = item:GetData("uses", 3)

        -- Do something
        item.player:Notify("Used item. " .. (uses - 1) .. " uses remain.")

        if uses <= 1 then
            return true  -- Remove when uses depleted
        end

        item:SetData("uses", uses - 1)
        return false  -- Keep in inventory
    end
}

function ITEM:GetDescription()
    local uses = self:GetData("uses", 3)
    return self.description .. "\nUses remaining: " .. uses
end
```

### Pattern 3: Checking Item State

```lua
ITEM.functions.Equip = {
    OnCanRun = function(item)
        local client = item.player

        -- Validate player exists
        if !IsValid(client) then return false end

        -- Check if already equipped
        if item:GetData("equip") == true then return false end

        -- Check if item is dropped in world
        if IsValid(item.entity) then return false end

        -- Check hooks
        if hook.Run("CanPlayerEquipItem", client, item) == false then
            return false
        end

        return true
    end
}
```

## Common Issues

### Issue: Item Data Not Saving

**Cause**: Using regular Lua variables instead of `SetData()`
**Fix**: Always use `item:SetData()` for persistent properties

```lua
-- WRONG
item.ammoCount = 30  -- Won't persist!

-- CORRECT
item:SetData("ammoCount", 30)  -- Saves to database
```

### Issue: Item Function Not Appearing

**Cause**: `OnCanRun` returning false or not defined
**Fix**: Ensure `OnCanRun` allows the function to run

```lua
ITEM.functions.Use = {
    OnRun = function(item)
        -- ...
    end,
    -- Add this if missing
    OnCanRun = function(item)
        return !IsValid(item.entity) and IsValid(item.player)
    end
}
```

### Issue: Item Not Loading on Reconnect

**Cause**: Not implementing `OnLoadout` for equipped items
**Fix**: Implement `OnLoadout` to restore equipped state

```lua
function ITEM:OnLoadout()
    if self:GetData("equip") then
        -- Re-equip the item
        self:Equip(self.player)
    end
end
```

## See Also

- [Weapons](weapons.md) - Weapon item type documentation
- [Ammunition](ammunition.md) - Ammunition item type documentation
- [Bags](bags.md) - Container item type documentation
- [Outfits](outfits.md) - Outfit item type documentation
- [PAC Outfits](pac-outfits.md) - PAC outfit item type documentation
- [Custom Items](custom-items.md) - Guide to creating custom items
- [Item System](../systems/items.md) - Core item system documentation
- [Inventory System](../systems/inventory.md) - Inventory management
