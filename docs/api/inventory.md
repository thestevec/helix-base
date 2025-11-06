# Inventory API (ix.inventory)

> **Reference**: `gamemode/core/libs/sh_inventory.lua`, `gamemode/core/meta/sh_inventory.lua`

The inventory API manages item containers with grid-based storage, weight limits, and transfer authorization.

## Core Concepts

Inventories are grid-based containers:
- Defined by width and height
- Store items with positions
- Have optional weight limits
- Sync to authorized clients
- Can be nested (bags)

## Functions

### ix.inventory.Create

**Realm**: Server

```lua
local inv = ix.inventory.Create(width, height, id)
```

Creates new inventory.

**Example**:
```lua
-- Create 5x5 inventory
local inv = ix.inventory.Create(5, 5)

-- Create with specific ID
local inv = ix.inventory.Create(10, 10, "storage_123")
```

## Inventory Methods

### inventory:Add

**Realm**: Server

```lua
local item = inventory:Add(itemID, quantity, data, x, y)
```

Adds item to inventory.

**Example**:
```lua
-- Add item
inventory:Add("health_kit")

-- Add with quantity
inventory:Add("ammo_pistol", 50)

-- Add at specific position
inventory:Add("weapon_pistol", 1, {}, 0, 0)
```

### inventory:Remove

**Realm**: Server

```lua
local success = inventory:Remove(itemID, quantity)
```

Removes item from inventory.

### inventory:HasItem

**Realm**: Shared

```lua
local has = inventory:HasItem(itemID, quantity)
```

Checks if inventory contains item.

**Example**:
```lua
if inventory:HasItem("health_kit") then
    print("Has health kit")
end

if inventory:HasItem("ammo_pistol", 50) then
    print("Has at least 50 pistol ammo")
end
```

### inventory:GetItems

**Realm**: Shared

```lua
local items = inventory:GetItems()
```

Gets all items in inventory.

**Example**:
```lua
for _, item in pairs(inventory:GetItems()) do
    print(item:GetName(), item:GetQuantity())
end
```

## Complete Example

```lua
-- Create character inventory
local char = client:GetCharacter()
local inv = ix.inventory.Create(6, 4)

-- Give to character
char:SetInventory(inv)

-- Add starting items
inv:Add("health_kit", 2)
inv:Add("weapon_pistol")
inv:Add("ammo_pistol", 30)

-- Check contents
for _, item in pairs(inv:GetItems()) do
    print("Item:", item:GetName())
end
```

## See Also

- [Item API](item.md) - Item definitions
- [Storage API](storage.md) - Inventory interaction
- [Character API](character.md) - Character inventories
- Source: `gamemode/core/libs/sh_inventory.lua`
