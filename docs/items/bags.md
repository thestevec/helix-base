# Bag Items

> **Reference**: `gamemode/items/base/sh_bags.lua`

The bag base item provides container items with their own internal inventory, allowing players to carry additional items.

## ⚠️ Important: Use Built-in Bag Item Base

**Always extend from `base_bags`** rather than creating custom container systems. The framework provides:
- Automatic inventory creation and registration (line 98-108)
- Database persistence for bag contents (line 186-193)
- Network synchronization (line 129, 144)
- Owner tracking and permissions (line 221-243)
- Nested bag prevention (line 201-218)
- Automatic cleanup on deletion (line 182-194)

## Core Concepts

### What is a Bag Item?

Bag items are container items that have their own inventory. When created, they:
1. Create a new inventory instance
2. Link to the bag item via unique ID
3. Allow viewing/managing contents via UI
4. Persist contents to database
5. Transfer ownership when bag moves

### Key Properties

**Reference**: `gamemode/items/base/sh_bags.lua:6-14`

```lua
ITEM.name = "Bag"
ITEM.description = "A bag to hold items."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.category = "Storage"
ITEM.width = 2          -- Bag's size in inventory
ITEM.height = 2
ITEM.invWidth = 4       -- Internal inventory width
ITEM.invHeight = 2      -- Internal inventory height
ITEM.isBag = true       -- Identifies as bag item
```

The bag itself takes `2x2` cells in the player's inventory, but provides a `4x2` internal inventory for storage.

## Using Bag Items

### Creating a Custom Bag

**Reference**: `gamemode/items/base/sh_bags.lua:6-14`

Create a new bag item by extending the base:

```lua
-- schema/items/sh_backpack.lua
ITEM.name = "Backpack"
ITEM.description = "A sturdy backpack for carrying supplies."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 3
ITEM.invWidth = 6       -- 6 cells wide
ITEM.invHeight = 4      -- 4 cells tall
ITEM.category = "Storage"
ITEM.isBag = true
ITEM.base = "base_bags"
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom container systems
local customStorage = {}
customStorage[player] = {}
-- This won't persist, won't sync, won't integrate with inventory!
```

### Inventory Creation

**Reference**: `gamemode/items/base/sh_bags.lua:95-108`

Bags automatically create an inventory when instantiated:

```lua
function ITEM:OnInstanced(invID, x, y)
    local inventory = ix.item.inventories[invID]

    ix.inventory.New(inventory and inventory.owner or 0, self.uniqueID, function(inv)
        local client = inv:GetOwner()

        inv.vars.isBag = self.uniqueID
        self:SetData("id", inv:GetID())

        if IsValid(client) then
            inv:AddReceiver(client)
        end
    end)
end
```

The bag's internal inventory ID is stored in `item:GetData("id")`.

### Viewing Bag Contents

**Reference**: `gamemode/items/base/sh_bags.lua:15-56`

Players can view a bag's contents via the View function:

```lua
ITEM.functions.View = {
    icon = "icon16/briefcase.png",
    OnClick = function(item)
        local index = item:GetData("id", "")

        if index then
            local inventory = ix.item.inventories[index]

            -- Create inventory UI panel
            local panel = vgui.Create("ixInventory")
            panel:SetInventory(inventory)
            panel:SetTitle(item:GetName())
            -- ...
        end

        return false  -- Don't remove item
    end,
    OnCanRun = function(item)
        return !IsValid(item.entity) and item:GetData("id") and
               !IsValid(ix.gui["inv" .. item:GetData("id", "")])
    end
}
```

Right-clicking the bag and selecting "View" opens the bag's inventory UI.

### Getting Bag Inventory

**Reference**: `gamemode/items/base/sh_bags.lua:110-118`

Access a bag's inventory programmatically:

```lua
local bag = item  -- The bag item instance
local inventory = bag:GetInventory()

if inventory then
    -- Work with the inventory
    for item, data in inventory:Iter() do
        print(item.name)
    end
end

-- Alias also available:
local inv = bag:GetInv()
```

### Transfer Items to Bag

**Reference**: `gamemode/items/base/sh_bags.lua:57-76`

The framework provides a "combine" function:

```lua
ITEM.functions.combine = {
    OnRun = function(item, data)
        ix.item.instances[data[1]]:Transfer(item:GetData("id"), nil, nil, item.player)
        return false
    end,
    OnCanRun = function(item, data)
        local index = item:GetData("id", "")

        if index then
            local inventory = ix.item.inventories[index]
            if inventory then
                return true
            end
        end

        return false
    end
}
```

This allows dragging items onto the bag to transfer them.

## Bag Ownership and Transfers

### Ownership Tracking

**Reference**: `gamemode/items/base/sh_bags.lua:221-243`

When a bag moves between inventories, ownership updates:

```lua
function ITEM:OnTransferred(curInv, inventory)
    local bagInventory = self:GetInventory()

    if isfunction(curInv.GetOwner) then
        local owner = curInv:GetOwner()

        if IsValid(owner) then
            bagInventory:RemoveReceiver(owner)  -- Remove old owner
        end
    end

    if isfunction(inventory.GetOwner) then
        local owner = inventory:GetOwner()

        if IsValid(owner) then
            bagInventory:AddReceiver(owner)     -- Add new owner
            bagInventory:SetOwner(owner)
        end
    else
        -- No valid owner (dropped in world)
        bagInventory:SetOwner(nil)
    end
end
```

### Dropping Bags

**Reference**: `gamemode/items/base/sh_bags.lua:157-168`

When a bag is dropped, database ownership is cleared:

```lua
ITEM.postHooks.drop = function(item, result)
    local index = item:GetData("id")

    local query = mysql:Update("ix_inventories")
        query:Update("character_id", 0)
        query:Where("inventory_id", index)
    query:Execute()

    -- Notify client to close bag UI
    net.Start("ixBagDrop")
        net.WriteUInt(index, 32)
    net.Send(item.player)
end
```

This ensures the bag contents are no longer tied to the character.

## Nested Bag Prevention

**Reference**: `gamemode/items/base/sh_bags.lua:197-219`

Bags cannot be placed inside other bags:

```lua
function ITEM:CanTransfer(oldInventory, newInventory)
    local index = self:GetData("id")

    if newInventory then
        -- Can't put bag inside another bag
        if newInventory.vars and newInventory.vars.isBag then
            return false
        end

        local index2 = newInventory:GetID()

        -- Can't put bag inside itself
        if index == index2 then
            return false
        end

        -- Can't create circular reference
        for k, _ in self:GetInventory():Iter() do
            if k:GetData("id") == index2 then
                return false
            end
        end
    end

    return !newInventory or newInventory:GetID() != oldInventory:GetID() or newInventory.vars.isBag
end
```

This prevents:
- Bags inside bags
- Bags inside themselves
- Circular bag references

## Database Persistence

### Bag Inventory Registration

**Reference**: `gamemode/items/base/sh_bags.lua:246-248`

Bag inventories are registered with the framework:

```lua
function ITEM:OnRegistered()
    ix.inventory.Register(self.uniqueID, self.invWidth, self.invHeight, true)
end
```

This creates the inventory type for this bag.

### Bag Deletion

**Reference**: `gamemode/items/base/sh_bags.lua:182-194`

When a bag is removed, its contents are deleted:

```lua
function ITEM:OnRemoved()
    local index = self:GetData("id")

    if index then
        -- Delete all items in bag
        local query = mysql:Delete("ix_items")
            query:Where("inventory_id", index)
        query:Execute()

        -- Delete bag inventory
        query = mysql:Delete("ix_inventories")
            query:Where("inventory_id", index)
        query:Execute()
    end
end
```

**⚠️ WARNING**: Deleting a bag permanently deletes all items inside!

### Loading Bags

**Reference**: `gamemode/items/base/sh_bags.lua:121-155`

Bags are restored from database on load:

```lua
function ITEM:OnSendData()
    local index = self:GetData("id")

    if index then
        local inventory = ix.item.inventories[index]

        if inventory then
            inventory.vars.isBag = self.uniqueID
            inventory:Sync(self.player)
            inventory:AddReceiver(self.player)
        else
            -- Restore from database
            local owner = self.player:GetCharacter():GetID()

            ix.inventory.Restore(self:GetData("id"), self.invWidth, self.invHeight, function(inv)
                inv.vars.isBag = self.uniqueID
                inv:SetOwner(owner, true)
                -- ...
            end)
        end
    end
end
```

## UI Integration

### Inventory Highlighting

**Reference**: `gamemode/items/base/sh_bags.lua:78-92`

CLIENT-SIDE: Bags highlight when hovered:

```lua
if CLIENT then
    function ITEM:PaintOver(item, width, height)
        local panel = ix.gui["inv" .. item:GetData("id", "")]

        if !IsValid(panel) then
            return
        end

        if vgui.GetHoveredPanel() == self then
            panel:SetHighlighted(true)
        else
            panel:SetHighlighted(false)
        end
    end
end
```

This provides visual feedback when hovering over the bag in inventory.

## Best Practices

### ✅ DO

- Use `ITEM.base = "base_bags"` for all bag items
- Set `invWidth` and `invHeight` to define internal capacity
- Set `isBag = true` to identify as bag item
- Use `item:GetInventory()` to access bag contents
- Let framework handle ownership and permissions
- Warn players before deleting bags (contents are lost)
- Use reasonable inventory sizes (don't make bags too large)

### ❌ DON'T

- Don't create custom container systems
- Don't manually manage bag inventory IDs
- Don't allow bags inside bags (framework prevents this)
- Don't delete bags without warning players
- Don't modify bag inventory database directly
- Don't forget that deleting a bag deletes all contents
- Don't make bags smaller than their contained items

## Complete Example

```lua
-- schema/items/sh_military_backpack.lua
ITEM.name = "Military Backpack"
ITEM.description = "A large tactical backpack with multiple compartments."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 3
ITEM.height = 3
ITEM.invWidth = 8       -- Large internal capacity
ITEM.invHeight = 6
ITEM.category = "Storage"
ITEM.isBag = true
ITEM.base = "base_bags"

-- Optional: Limit what can go in bag
function ITEM:CanTransfer(oldInventory, newInventory)
    -- Call parent first
    if not baseclass.Get("base_bags").CanTransfer(self, oldInventory, newInventory) then
        return false
    end

    -- Additional restrictions
    if newInventory and self:GetData("locked") then
        local owner = self:GetOwner()
        if IsValid(owner) then
            owner:Notify("This bag is locked!")
        end
        return false
    end

    return true
end

-- Optional: Add lock/unlock functions
ITEM.functions.Lock = {
    name = "Lock",
    OnRun = function(item)
        item:SetData("locked", true)
        item.player:Notify("You locked the backpack.")
        return false
    end,
    OnCanRun = function(item)
        return !item:GetData("locked")
    end
}

ITEM.functions.Unlock = {
    name = "Unlock",
    OnRun = function(item)
        item:SetData("locked", false)
        item.player:Notify("You unlocked the backpack.")
        return false
    end,
    OnCanRun = function(item)
        return item:GetData("locked")
    end
}
```

## Common Issues

### Issue: Bag Contents Not Saving

**Cause**: Bag inventory not properly linked to character
**Fix**: Framework handles this automatically - ensure `OnSendData` is not overridden incorrectly

The base implementation handles saving automatically.

### Issue: Bag Inventory Not Opening

**Cause**: Bag inventory not created or ID not stored
**Fix**: Ensure bag has inventory ID

```lua
local bag = item
local invID = bag:GetData("id")

if not invID then
    print("ERROR: Bag has no inventory ID!")
end
```

### Issue: Items Disappearing from Bag

**Cause**: Bag deleted or database issue
**Fix**: Check database and warn before deleting bags

```lua
-- Before deleting bag
local bagInv = item:GetInventory()
if bagInv then
    local itemCount = 0
    for _ in bagInv:Iter() do
        itemCount = itemCount + 1
    end

    if itemCount > 0 then
        player:Notify("Warning: Deleting this bag will destroy " .. itemCount .. " items!")
    end
end
```

### Issue: Bag Inside Bag

**Cause**: Trying to circumvent CanTransfer
**Fix**: Framework prevents this - don't override `CanTransfer` without calling parent

```lua
-- CORRECT: Call parent function
function ITEM:CanTransfer(oldInventory, newInventory)
    if not baseclass.Get("base_bags").CanTransfer(self, oldInventory, newInventory) then
        return false
    end

    -- Your additional checks
    return true
end
```

## See Also

- [Base Items](base-items.md) - Overview of all base item types
- [Item System](../systems/items.md) - Core item system
- [Inventory System](../systems/inventory.md) - Inventory management
- [Storage Library](../libraries/storage.md) - Storage container system
- Source: `gamemode/items/base/sh_bags.lua`
