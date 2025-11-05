# Inventory System

> **Reference**: `gamemode/core/libs/sh_inventory.lua`, `gamemode/core/meta/sh_inventory.lua`

The Helix inventory system provides grid-based inventory management with support for drag-and-drop UI, item positioning, and nested inventories (bags).

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's built-in inventory functions** rather than creating custom inventory systems. The framework provides comprehensive inventory management that handles:
- Database persistence
- Network synchronization
- Item positioning and collision detection
- UI updates
- Storage contexts (chests, containers, etc.)
- Bag/nested inventories

## Core Concepts

### Inventory Structure

Inventories in Helix are grid-based with configurable dimensions:

```lua
-- Default character inventory (from config)
width = ix.config.Get("inv Width", 6)
height = ix.config.Get("invHeight", 4)

-- Storage inventory (custom size)
width = 8
height = 6
```

### Inventory Types

**Character Inventory**: Every character has a persistent inventory
- Created automatically on character creation
- Loaded from database when character loads
- Saved automatically on disconnect/server shutdown

**Storage Inventory**: Containers, chests, stashes
- Attached to entities via `ix.storage` system
- Can be accessed by multiple players
- Supports search time and custom UI

**Bag Inventory**: Nested inventories inside items
- Created by items with `base_bags` base class
- Cannot contain other bags (prevent infinite nesting)
- Share storage context with parent inventory

## Creating Inventories

### Using Built-in Functions

**Reference**: `gamemode/core/libs/sh_inventory.lua:133`

```lua
-- SERVER ONLY
-- Create new inventory in database
ix.inventory.New(ownerID, invType, callback)

-- Example: Create character inventory
ix.inventory.New(character:GetID(), "grid", function(inventory)
    if inventory then
        print("Created inventory:", inventory:GetID())
        character:SetInventory(inventory)
    end
end)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create inventory tables manually
local inventory = {
    w = 6,
    h = 4,
    slots = {}
}
-- This will not save to database or sync to client!
```

### Restoring Inventories from Database

**Reference**: `gamemode/core/libs/sh_inventory.lua:38`

```lua
-- SERVER ONLY
-- Restore single inventory
ix.inventory.Restore(invID, width, height, callback)

-- Example
ix.inventory.Restore(inventory ID, 6, 4, function(inventory)
    print("Restored inventory with items")
end)

-- Restore multiple inventories
ix.inventory.Restore({
    [10] = {6, 4},  -- Inventory 10: 6x4 grid
    [11] = {8, 6}   -- Inventory 11: 8x6 grid
}, nil, nil, function(inventory)
    -- Called for each inventory
    print("Restored:", inventory:GetID())
end)
```

## Getting Inventories

### Use Built-in Getters

**Reference**: `gamemode/core/libs/sh_inventory.lua:15`

```lua
-- Get inventory by ID
local inventory = ix.inventory.Get(inventoryID)

-- Get character's inventory
local character = client:GetCharacter()
local inventory = character:GetInventory()

-- Get inventory ID
local invID = inventory:GetID()
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't access internal tables directly
local inventory = ix.item.inventories[invID]  -- Use ix.inventory.Get() instead
```

## Adding Items to Inventory

### Use Inventory Methods

**Reference**: `gamemode/core/meta/sh_inventory.lua`

```lua
-- SERVER ONLY
-- Add item by unique ID
inventory:Add(itemID, quantity, x, y, callback)

-- Examples:

-- Add item (auto-find position)
inventory:Add("sh_healthkit")

-- Add item at specific position
inventory:Add("sh_healthkit", 1, 0, 0)  -- x=0, y=0

-- Add with callback
inventory:Add("sh_healthkit", 1, nil, nil, function(item)
    if item then
        item:SetData("custom", "value")
        print("Added item:", item:GetID())
    else
        print("Failed to add item - no space?")
    end
end)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create items manually
local item = {uniqueID = "sh_healthkit"}
inventory.slots[0][0] = item  -- Won't save to database!
```

## Removing Items from Inventory

```lua
-- SERVER ONLY
-- Remove item by instance ID
inventory:Remove(itemID, bNoDelete, callback)

-- Example
local item = inventory:GetItemAt(0, 0)
if item then
    inventory:Remove(item:GetID())
end

-- Or use item method
item:Remove()  -- Automatically removes from inventory
```

## Checking Inventory Space

### Use Built-in Space Checking

**Reference**: `gamemode/core/meta/sh_inventory.lua`

```lua
-- Check if specific item can fit
local canFit, x, y = inventory:CanItemFit(item)
if canFit then
    inventory:Add(item.uniqueID, 1, x, y)
end

-- Find free position for item size
local x, y = inventory:FindEmptySlot(itemWidth, itemHeight)
if x and y then
    print("Found space at:", x, y)
else
    print("No space available")
end

-- Check if position is free
local isFree = inventory:IsFree(x, y, width, height)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't implement your own space checking
local hasSpace = true
for x = 0, width - 1 do
    for y = 0, height - 1 do
        if slots[x] and slots[x][y] then
            hasSpace = false
        end
    end
end
-- Use built-in CanItemFit() instead!
```

## Iterating Items

### Use Built-in Iterator

**Reference**: `gamemode/core/meta/sh_inventory.lua`

```lua
-- Get all items (returns table)
local items = inventory:GetItems()
for _, item in pairs(items) do
    print(item:GetName())
end

-- Use iterator (more efficient)
for item, x, y in inventory:Iter() do
    print(string.format("%s at (%d, %d)", item:GetName(), x, y))
end

-- Get item at specific position
local item = inventory:GetItemAt(x, y)
if item then
    print("Found:", item:GetName())
end
```

## Storage System

### Opening Storage for Players

**Reference**: `gamemode/core/libs/sh_storage.lua:79`

**Always use `ix.storage.Open()`** to let players access storage inventories:

```lua
-- SERVER ONLY
ix.storage.Open(client, inventory, storageInfo)

-- Example: Simple chest
local inventory = ix.inventory.Get(chestInventoryID)
ix.storage.Open(client, inventory, {
    name = "Wooden Chest",
    entity = chestEntity,
    bMultipleUsers = true  -- Allow multiple players
})

-- Example: Timed search
ix.storage.Open(client, inventory, {
    name = "Filing Cabinet",
    entity = cabinetEntity,
    searchTime = 4,  -- 4 second search time
    searchText = "Rummaging through files..."
})

-- Example: With callback
ix.storage.Open(client, inventory, {
    name = "Safe",
    entity = safeEntity,
    OnPlayerClose = function(client)
        print(client:Name() .. " closed the safe")
    end
})
```

### Storage Info Structure

**Reference**: `gamemode/core/libs/sh_storage.lua:20-34`

```lua
{
    entity = entityOrPlayer,      -- Entity to attach inventory to (required)
    id = inventoryID,              -- Inventory ID (defaults to inventory parameter)
    name = "Storage Name",         -- Display name (default: "Storage")
    bMultipleUsers = false,        -- Allow multiple players (default: false)
    searchTime = 0,                -- Search delay in seconds (default: 0)
    searchText = "@storageSearching", -- Search message (default: localized)
    OnPlayerClose = function(client) end,  -- Callback when player closes
    data = {}                      -- Arbitrary data sent to client
}
```

### Closing Storage

```lua
-- SERVER ONLY
ix.storage.Close(client, inventory)

-- Example
local inventory = client:GetCharacter():GetInventory()
ix.storage.Close(client, inventory)
```

### Checking Storage Usage

**Reference**: `gamemode/core/libs/sh_storage.lua:50-77`

```lua
-- SERVER ONLY
-- Check if any player is using inventory
local inUse = ix.storage.InUse(inventory)

-- Check if specific player is using inventory
local isUsing = ix.storage.InUseBy(inventory, client)
```

## Transferring Items

### Use Built-in Transfer

**Reference**: `gamemode/core/meta/sh_item.lua`

```lua
-- SERVER ONLY
-- Transfer item to different inventory
item:Transfer(newInventoryID, x, y, client)

-- Example: Move item to storage
local item = character:GetInventory():GetItemAt(0, 0)
local storageInv = ix.inventory.Get(storageInventoryID)

item:Transfer(storageInv:GetID(), 0, 0, client)
```

**The framework automatically**:
- Checks if item can fit
- Calls `ITEM:CanTransfer()` hook
- Updates database
- Syncs to clients
- Calls `ITEM:OnTransferred()` hook

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually move items
inventory1.slots[x][y] = nil
inventory2.slots[x2][y2] = item
-- Won't save or sync properly!
```

## Inventory Hooks

### Use These Hooks in Items/Plugins

```lua
-- Item can prevent transfers
function ITEM:CanTransfer(oldInventory, newInventory)
    if self:GetData("locked") then
        return false  -- Prevent transfer
    end
    return true
end

-- Item notified of transfer
function ITEM:OnTransferred(oldInventory, newInventory)
    local owner = self:GetOwner()
    if IsValid(owner) then
        owner:ChatPrint("Item moved!")
    end
end

-- Plugin can intercept transfers
function PLUGIN:CanItemTransfer(item, oldInv, newInv)
    -- Return false to prevent
    return true
end
```

## Inventory UI

### Opening Inventory UI

**CLIENT** - Use Helix's built-in UI system:

```lua
-- Open main inventory
vgui.Create("ixInventory")

-- Framework automatically handles:
-- - Creating grid display
-- - Drag-and-drop
-- - Item tooltips
-- - Action buttons
-- - Network synchronization
```

**⚠️ Do NOT** create custom inventory UI unless absolutely necessary. The built-in system handles all edge cases.

## Money in Inventories

### Storage Money System

**Reference**: `gamemode/core/libs/sh_storage.lua:42-44`

```lua
-- SERVER ONLY
-- Storage inventories can hold money
-- Network strings: ixStorageMoneyTake, ixStorageMoneyGive, ixStorageMoneyUpdate

-- The framework handles money UI automatically when storage is opened
-- Players can drag money into storage inventories
```

## Bags (Nested Inventories)

### Use Base Bags Item Class

**Reference**: `gamemode/items/base/sh_bags.lua`

```lua
-- Create bag item
ITEM.name = "Backpack"
ITEM.base = "base_bags"
ITEM.width = 2
ITEM.height = 2
ITEM.invWidth = 5   -- Internal inventory width
ITEM.invHeight = 5  -- Internal inventory height

-- Framework automatically:
-- - Creates nested inventory
-- - Prevents bag-in-bag
-- - Handles UI
-- - Saves bag contents
```

**⚠️ Do NOT** manually create nested inventories. Use `base_bags` class.

## Database Schema

**Reference**: Framework automatically manages these tables:

```sql
-- ix_inventories table
CREATE TABLE ix_inventories (
    inventory_id INT AUTO_INCREMENT PRIMARY KEY,
    inventory_type VARCHAR(32),
    character_id INT
);

-- ix_items table (stores item positions)
CREATE TABLE ix_items (
    item_id INT AUTO_INCREMENT PRIMARY KEY,
    inventory_id INT,
    unique_id VARCHAR(60),
    data TEXT,
    character_id INT,
    player_id VARCHAR(20),
    x INT,
    y INT
);
```

**⚠️ Do NOT** modify these tables directly. Use Helix functions.

## Common Patterns

### Give Item to Character

```lua
-- CORRECT WAY
local character = client:GetCharacter()
local inventory = character:GetInventory()

if inventory then
    inventory:Add("sh_healthkit", 1, nil, nil, function(item)
        if item then
            client:Notify("Received health kit!")
        else
            client:Notify("Inventory full!")
        end
    end)
end
```

### Create Storage Container

```lua
-- CORRECT WAY - Complete example
function PLUGIN:CreateStorageEntity(position)
    if CLIENT then return end

    -- Create entity
    local entity = ents.Create("prop_physics")
    entity:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
    entity:SetPos(position)
    entity:Spawn()

    -- Create inventory in database
    ix.inventory.New(0, "storage", function(inventory)
        if not inventory then return end

        -- Store inventory ID on entity
        entity.ixInventoryID = inventory:GetID()

        -- Setup use function
        entity:SetUse(function(ent, activator)
            if not IsValid(activator) or not activator:IsPlayer() then return end

            local inv = ix.inventory.Get(ent.ixInventoryID)
            if inv then
                ix.storage.Open(activator, inv, {
                    name = "Storage Container",
                    entity = ent,
                    bMultipleUsers = true
                })
            end
        end)
    end)

    return entity
end
```

### Check and Add Item

```lua
-- CORRECT WAY
local inventory = character:GetInventory()

-- First check if it will fit
local item = ix.item.list["sh_heavyweapon"]
if inventory:CanItemFit(item) then
    -- It fits, add it
    inventory:Add(item.uniqueID)
else
    client:Notify("Not enough inventory space!")
end
```

## Performance Considerations

### Database Operations

The framework batches inventory operations for efficiency:
- Item additions are queued and written in batches
- Inventory sync is throttled to prevent network spam
- Item data changes are lazy-saved on disconnect

**Trust the framework's optimization**. Don't implement custom saving logic.

### Network Sync

Inventories are automatically synced to clients:
- Only synced to players who can see the inventory
- Item data is compressed using PON serialization
- Updates are sent only when inventory changes

## Best Practices

### ✅ DO

- Use `ix.inventory.Get()` to get inventories
- Use `inventory:Add()` to add items
- Use `ix.storage.Open()` for storage access
- Use `base_bags` for container items
- Let the framework handle database operations
- Trust built-in space checking
- Use provided hooks for item behavior

### ❌ DON'T

- Don't create inventory tables manually
- Don't access `ix.item.inventories` directly
- Don't implement custom space checking
- Don't manually update database
- Don't create custom inventory UI without good reason
- Don't allow bag-in-bag (framework prevents this)
- Don't bypass `ix.storage` system for containers

## Common Issues

### "Inventory full" but appears empty

**Cause**: Items are in database but not loaded
**Fix**: Use `ix.inventory.Restore()` to load items from database

### Items disappearing

**Cause**: Not using proper Helix functions
**Fix**: Always use `inventory:Add()` and `item:Remove()`

### Items not saving

**Cause**: Custom inventory not created via `ix.inventory.New()`
**Fix**: Use built-in inventory creation functions

### Multiple players can't access storage

**Cause**: `bMultipleUsers` not set to true
**Fix**: Set `bMultipleUsers = true` in storage info

## See Also

- [Item System](items.md) - Creating and managing items
- [Character System](character.md) - Character inventory access
- [Storage Library](../libraries/storage.md) - Storage system details
- [Base Bags Item](../items/bags.md) - Container items
- Item System Reference: `gamemode/core/libs/sh_item.lua`
- Inventory Meta: `gamemode/core/meta/sh_inventory.lua`
- Storage System: `gamemode/core/libs/sh_storage.lua`
