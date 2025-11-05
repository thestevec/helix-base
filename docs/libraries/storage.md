# Storage Library (ix.storage)

> **Reference**: `gamemode/core/libs/sh_storage.lua`

The storage library provides player interaction with inventories (chests, containers, stashes).

## ⚠️ Important: Use ix.storage Functions

**Always use `ix.storage.Open()`** to let players access storage. Don't create custom storage UIs. Framework handles:
- Opening inventory UI for both player and storage
- Distance checking
- Multiple user support
- Search time/delays
- Automatic closing on disconnect
- Money transfer UI

## Opening Storage

### ix.storage.Open()

**Reference**: `gamemode/core/libs/sh_storage.lua:79`

```lua
-- SERVER ONLY
ix.storage.Open(client, inventory, storageInfo)

-- Simple example
local inventory = ix.inventory.Get(inventoryID)
ix.storage.Open(client, inventory, {
    name = "Storage Chest",
    entity = chestEntity
})

-- With all options
ix.storage.Open(client, inventory, {
    name = "Safe",                  -- Display name
    entity = safeEntity,            -- Entity to attach to (required)
    bMultipleUsers = true,          -- Allow multiple players
    searchTime = 5,                 -- 5 second search delay
    searchText = "Cracking safe...", -- Search message
    OnPlayerClose = function(client)
        print(client:Name() .. " closed safe")
    end,
    data = {locked = false}         -- Custom data sent to client
})
```

## Storage Info Structure

```lua
{
    entity = entity,              -- Entity or player (REQUIRED)
    id = inventoryID,             -- Defaults to inventory parameter
    name = "Storage",             -- Display name (default: "Storage")
    bMultipleUsers = false,       -- Allow multiple users (default: false)
    searchTime = 0,               -- Search delay seconds (default: 0)
    searchText = "@storageSearching", -- Search message (default: localized)
    OnPlayerClose = function(client) end, -- Callback when closed
    data = {}                     -- Arbitrary data for client
}
```

## Closing Storage

```lua
-- SERVER ONLY
ix.storage.Close(client, inventory)
```

## Checking Usage

**Reference**: `gamemode/core/libs/sh_storage.lua:50-77`

```lua
-- SERVER ONLY
-- Check if any player is using inventory
if ix.storage.InUse(inventory) then
    print("Storage is being accessed")
end

-- Check if specific player is using
if ix.storage.InUseBy(inventory, client) then
    print(client:Name() .. " is using storage")
end
```

## Complete Storage Entity Example

```lua
function PLUGIN:CreateStorageChest(position)
    if CLIENT then return end

    -- Create entity
    local chest = ents.Create("prop_physics")
    chest:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
    chest:SetPos(position)
    chest:Spawn()

    -- Create inventory
    ix.inventory.New(0, "storage", function(inventory)
        if not inventory then return end

        chest.ixInventoryID = inventory:GetID()

        -- Set use function
        function chest:Use(activator)
            if not IsValid(activator) or not activator:IsPlayer() then return end

            local inv = ix.inventory.Get(self.ixInventoryID)
            if not inv then return end

            ix.storage.Open(activator, inv, {
                name = "Storage Chest",
                entity = self,
                bMultipleUsers = true
            })
        end

        -- Close storage when entity is removed
        function chest:OnRemove()
            local inv = ix.inventory.Get(self.ixInventoryID)
            if inv then
                for _, receiver in pairs(inv:GetReceivers()) do
                    ix.storage.Close(receiver, inv)
                end
            end
        end
    end)

    return chest
end
```

## Best Practices

### ✅ DO

- Use `ix.storage.Open()` for all storage access
- Attach storage to entity via `storageInfo.entity`
- Create inventory with `ix.inventory.New()`
- Close storage when entity is removed
- Use `bMultipleUsers` for shared storage

### ❌ DON'T

- Don't create custom storage UI
- Don't bypass ix.storage system
- Don't forget to close storage on entity removal
- Don't allow storage access without entity

## See Also

- [Inventory System](../systems/inventory.md)
- Storage Library: `gamemode/core/libs/sh_storage.lua`
