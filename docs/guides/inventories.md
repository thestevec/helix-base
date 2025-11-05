# Working with Inventories Programmatically

> **Reference**: `gamemode/core/libs/sh_inventory.lua`, `gamemode/core/meta/sh_inventory.lua`

Practical guide to creating, managing, and manipulating inventories in Helix using code.

## ⚠️ Important: Use Helix's Inventory System

**Always use `ix.inventory` functions** rather than creating custom storage systems. The framework provides:
- Automatic database persistence via `ix.inventory.Restore()` (line 38)
- Network synchronization to clients
- Grid-based positioning with collision detection
- Item stacking and organization
- Nested inventory support (bags)

## Prerequisites

- Understanding of [Item System](../systems/items.md)
- Basic Lua and async programming knowledge
- Server-side scripting experience

## Part 1: Basic Inventory Operations

### Getting a Character's Inventory

```lua
local character = client:GetCharacter()
local inventory = character:GetInventory()

if inventory then
    print("Inventory ID:", inventory:GetID())
    print("Size:", inventory.w .. "x" .. inventory.h)
    print("Items:", table.Count(inventory:GetItems()))
end
```

### Adding Items

```lua
-- SERVER ONLY
local inventory = character:GetInventory()

-- Add item (auto-find position)
inventory:Add("sh_medkit", 1, nil, nil, function(item)
    if item then
        client:Notify("Added " .. item:GetName())
    else
        client:Notify("Inventory full!")
    end
end)

-- Add at specific position
inventory:Add("sh_medkit", 1, 0, 0)  -- Top-left corner

-- Add multiple
inventory:Add("sh_ammo_pistol", 5)  -- Add 5 instances
```

### Removing Items

```lua
-- SERVER ONLY
-- Remove by item instance
item:Remove()

-- Remove all items of type
for _, item in pairs(inventory:GetItems()) do
    if item.uniqueID == "sh_medkit" then
        item:Remove()
    end
end
```

### Checking if Item Can Fit

```lua
-- Check if item can fit at position
local canFit = inventory:CanItemFit(x, y, width, height, itemToIgnore)

if canFit then
    inventory:Add("sh_largeitem", 1, x, y)
else
    client:Notify("No space at that position!")
end
```

## Part 2: Creating Custom Inventories

### Creating Storage Container

```lua
-- SERVER ONLY
function PLUGIN:CreateStorageChest(position, width, height)
    -- Spawn entity
    local chest = ents.Create("prop_physics")
    chest:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
    chest:SetPos(position)
    chest:Spawn()

    -- Create inventory
    ix.inventory.New(0, "storage", function(inventory)
        if not inventory then
            ErrorNoHalt("Failed to create inventory!\n")
            chest:Remove()
            return
        end

        -- Set custom size
        inventory.w = width or 6
        inventory.h = height or 4

        -- Link to entity
        chest.ixInventoryID = inventory:GetID()
        inventory:SetOwner(chest:EntIndex(), true)

        print("Created storage with inventory:", inventory:GetID())
    end)

    return chest
end

-- Usage
local chest = PLUGIN:CreateStorageChest(Vector(0, 0, 0), 8, 6)
```

### Creating Player Stash

```lua
-- Per-character persistent storage
function PLUGIN:CreateStash(character)
    local stashID = character:GetData("stashInventory")

    if stashID then
        -- Restore existing stash
        ix.inventory.Restore(stashID, 8, 4, function(inventory)
            character:SetData("stash", inventory)
            print("Restored stash:", stashID)
        end)
    else
        -- Create new stash
        ix.inventory.New(character:GetID(), "stash", function(inventory)
            if inventory then
                character:SetData("stashInventory", inventory:GetID())
                character:SetData("stash", inventory)
                print("Created new stash:", inventory:GetID())
            end
        end)
    end
end

-- Access stash
local stash = character:GetData("stash")
if stash then
    -- Open UI, manipulate items, etc.
    stash:Add("sh_medkit")
end
```

## Part 3: Inventory Manipulation

### Moving Items Between Inventories

```lua
-- Transfer item to different inventory
local function TransferItem(item, targetInventory)
    if not item or not targetInventory then
        return false, "Invalid item or inventory"
    end

    -- Check if target can fit item
    local canFit, x, y = targetInventory:FindEmptySlot(item.width, item.height)

    if not canFit then
        return false, "Target inventory is full"
    end

    -- Transfer
    local success, error = item:Transfer(targetInventory:GetID(), x, y)

    if success then
        return true, "Item transferred"
    else
        return false, error or "Transfer failed"
    end
end

-- Usage
local sourceInv = character:GetInventory()
local targetInv = ix.inventory.Get(targetID)
local item = sourceInv:GetItems()[1]

local success, msg = TransferItem(item, targetInv)
print(msg)
```

### Swapping Items

```lua
function SwapItems(item1, item2)
    if not item1 or not item2 then
        return false
    end

    local inv1 = ix.inventory.Get(item1.invID)
    local inv2 = ix.inventory.Get(item2.invID)

    if inv1 == inv2 then
        -- Same inventory - swap positions
        local x1, y1 = item1.gridX, item1.gridY
        local x2, y2 = item2.gridX, item2.gridY

        item1:SetPos(x2, y2)
        item2:SetPos(x1, y1)

        return true
    else
        -- Different inventories - transfer both
        local success1 = item1:Transfer(inv2:GetID())
        local success2 = item2:Transfer(inv1:GetID())

        return success1 and success2
    end
end
```

### Finding Items

```lua
-- Find item by uniqueID
function FindItemInInventory(inventory, uniqueID)
    for _, item in pairs(inventory:GetItems()) do
        if item.uniqueID == uniqueID then
            return item
        end
    end
    return nil
end

-- Find all items matching criteria
function FindItemsByCategory(inventory, category)
    local results = {}

    for _, item in pairs(inventory:GetItems()) do
        if item.category == category then
            table.insert(results, item)
        end
    end

    return results
end

-- Usage
local medkit = FindItemInInventory(inventory, "sh_medkit")
local weapons = FindItemsByCategory(inventory, "Weapons")
```

## Part 4: Advanced Inventory Features

### Custom Inventory Types

```lua
-- Register custom inventory type
ix.item.RegisterInv("locker", 4, 8)  -- 4x8 grid

-- Create inventory with custom type
ix.inventory.New(0, "locker", function(inventory)
    -- inventory will be 4x8
    print("Created locker:", inventory:GetID())
end)
```

### Inventory Sync to Client

```lua
-- SERVER
-- Sync inventory to specific client
inventory:Sync(client)

-- Sync to all clients who can see it
local receivers = inventory:GetReceivers()
inventory:Sync(receivers)

-- CLIENT
-- Listen for inventory updates
function PLUGIN:InventoryItemAdded(inventory, item)
    print("Item added to inventory:", item:GetName())
end

function PLUGIN:InventoryItemRemoved(inventory, item)
    print("Item removed from inventory:", item:GetName())
end
```

### Searching Inventories

```lua
-- Create searchable storage
function CreateSearchableStorage(entity)
    local storage = ix.storage.New(entity, {
        name = "Locked Cabinet",
        description = "A secure storage locker",
        searchText = "Searching...",
        searchTime = 3  -- 3 seconds to open
    })

    storage:SetInventory(inventoryID)

    -- Only certain players can access
    function storage:OnAccess(client)
        local character = client:GetCharacter()

        if character:HasFlags("s") then
            return true
        end

        client:Notify("You don't have access!")
        return false
    end

    return storage
end
```

## Part 5: Practical Examples

### Example 1: Vendor System

```lua
-- Create vendor inventory with items
function CreateVendor(position, items)
    local vendor = ents.Create("ix_vendor")
    vendor:SetPos(position)
    vendor:Spawn()

    ix.inventory.New(0, "vendor", function(inventory)
        vendor.ixInventoryID = inventory:GetID()

        -- Add vendor items
        for _, itemID in ipairs(items) do
            inventory:Add(itemID, 1)
        end

        -- Make items infinite stock
        for _, item in pairs(inventory:GetItems()) do
            item:SetData("infinite", true)
        end
    end)

    return vendor
end

-- Usage
local vendorItems = {
    "sh_medkit",
    "sh_ammo_pistol",
    "sh_food_bread"
}

CreateVendor(Vector(0, 0, 0), vendorItems)
```

### Example 2: Item Durability System

```lua
-- Degrade items on use
function PLUGIN:ItemFunctionRan(item, funcName)
    if funcName ~= "Use" then return end

    local durability = item:GetData("durability", 100)
    durability = durability - 10

    if durability <= 0 then
        -- Item breaks
        item.player:Notify(item:GetName() .. " broke!")
        item:Remove()
    else
        item:SetData("durability", durability)
        item.player:Notify("Durability: " .. durability .. "%")
    end
end
```

### Example 3: Inventory Weight Limit

```lua
-- Add weight calculation
function GetInventoryWeight(inventory)
    local totalWeight = 0

    for _, item in pairs(inventory:GetItems()) do
        local weight = item.weight or 1
        totalWeight = totalWeight + weight
    end

    return totalWeight
end

-- Check weight before adding
function PLUGIN:CanItemTransfer(item, oldInv, newInv)
    if not newInv then return end

    local maxWeight = 50  -- Max 50kg
    local currentWeight = GetInventoryWeight(newInv)
    local itemWeight = item.weight or 1

    if currentWeight + itemWeight > maxWeight then
        if item.player then
            item.player:Notify("Inventory too heavy!")
        end
        return false
    end
end
```

### Example 4: Shared Storage

```lua
-- Create faction shared storage
function PLUGIN:CreateFactionStorage(factionID)
    local storageKey = "factionStorage_" .. factionID
    local invID = self:GetData(storageKey)

    if invID then
        -- Restore existing
        ix.inventory.Restore(invID, 10, 8)
    else
        -- Create new
        ix.inventory.New(0, "faction", function(inventory)
            self:SetData(storageKey, inventory:GetID())
        end)
    end
end

-- Allow access
function PLUGIN:CanPlayerAccessStorage(client, storage)
    local character = client:GetCharacter()
    local faction = character:GetFaction()

    local invID = storage.ixInventoryID
    local factionInvID = self:GetData("factionStorage_" .. faction)

    return invID == factionInvID
end
```

## Best Practices

### ✅ DO

- Use `ix.inventory.New()` to create inventories
- Use `ix.inventory.Restore()` to load from database
- Check callbacks for success: `if inventory then`
- Validate before transferring: `CanItemFit()`
- Use inventory methods: `:Add()`, `:Remove()`
- Sync to clients after changes: `inventory:Sync()`

### ❌ DON'T

- Don't create manual inventory tables
- Don't directly modify `inventory.slots`
- Don't forget SERVER checks for operations
- Don't access items without validation
- Don't create inventories in loops
- Don't forget to clean up unused inventories

## Debugging Inventories

```lua
-- Print inventory info
function PrintInventoryInfo(inventory)
    print("=== Inventory", inventory:GetID(), "===")
    print("Size:", inventory.w .. "x" .. inventory.h)
    print("Owner:", inventory:GetOwner())

    local items = inventory:GetItems()
    print("Items:", table.Count(items))

    for _, item in pairs(items) do
        print(string.format("  - %s at (%d,%d) [%dx%d]",
            item:GetName(),
            item.gridX, item.gridY,
            item.width, item.height
        ))
    end
end

-- Check for inventory issues
function ValidateInventory(inventory)
    local items = inventory:GetItems()
    local occupied = {}

    for _, item in pairs(items) do
        -- Check bounds
        if item.gridX < 0 or item.gridY < 0 then
            print("ERROR: Item out of bounds:", item:GetName())
        end

        if item.gridX + item.width > inventory.w then
            print("ERROR: Item exceeds width:", item:GetName())
        end

        if item.gridY + item.height > inventory.h then
            print("ERROR: Item exceeds height:", item:GetName())
        end

        -- Check overlaps
        for x = item.gridX, item.gridX + item.width - 1 do
            for y = item.gridY, item.gridY + item.height - 1 do
                local key = x .. "," .. y
                if occupied[key] then
                    print("ERROR: Overlapping items at", x, y)
                end
                occupied[key] = true
            end
        end
    end

    print("Validation complete")
end
```

## Troubleshooting

### Inventory not saving

**Cause**: Not created with `ix.inventory.New()`

**Fix**: Use proper creation function with callback

### Items disappearing

**Cause**: Not syncing to client after changes

**Fix**: Call `inventory:Sync(client)` after modifications

### "Inventory full" error

**Cause**: No empty slots for item size

**Fix**: Check with `FindEmptySlot()` before adding

### Items overlapping

**Cause**: Manual position setting without checks

**Fix**: Use `CanItemFit()` before positioning

## See Also

- [Inventory System Documentation](../systems/inventory.md)
- [Custom Items Guide](custom-items.md)
- [Item System Documentation](../systems/items.md)
- [Storage Library](../libraries/storage.md)
- Source: `gamemode/core/libs/sh_inventory.lua`
