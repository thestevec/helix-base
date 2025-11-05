# Storage API (ix.storage)

> **Reference**: `gamemode/core/libs/sh_storage.lua`

The storage API allows players to interact with external inventories like containers, chests, NPCs, or other players. It displays both inventories side-by-side for item transfer.

## Core Concepts

Storage provides inventory interaction:
- Opens dual-inventory UI
- Supports money transfer
- Optional search time
- Multiple user support
- Distance checking

## Functions

### ix.storage.Open

**Realm**: Server

```lua
ix.storage.Open(client, inventory, info)
```

Opens an inventory for player interaction.

**Parameters**:
- `client` (Player) - Player opening storage
- `inventory` (Inventory) - Inventory to open
- `info` (table) - Storage configuration:
  - `entity` - Entity associated with storage
  - `name` - Display name
  - `bMultipleUsers` - Allow multiple players
  - `searchTime` - Delay before opening
  - `searchText` - Text during search
  - `OnPlayerClose` - Callback when closed

**Example**:
```lua
-- Simple storage
ix.storage.Open(client, inventory, {
    entity = container,
    name = "Filing Cabinet"
})

-- With search time
ix.storage.Open(client, inventory, {
    entity = chest,
    name = "Locked Chest",
    searchTime = 5,
    searchText = "Picking lock...",
    OnPlayerClose = function(ply)
        print(ply:Name(), "closed chest")
    end
})

-- Multiple users
ix.storage.Open(client, inventory, {
    entity = shopNPC,
    name = "Shop",
    bMultipleUsers = true
})
```

### ix.storage.Close

**Realm**: Server

```lua
ix.storage.Close(inventory)
```

Forces all players to close inventory.

**Example**:
```lua
-- Close when entity removed
hook.Add("EntityRemoved", "CloseStorage", function(ent)
    if ent.inventory then
        ix.storage.Close(ent.inventory)
    end
end)
```

## Complete Example

### Container Entity

```lua
ENT.Type = "anim"
ENT.PrintName = "Storage Container"
ENT.Spawnable = true
ENT.Model = "models/props_c17/FurnitureDrawer001a.mdl"

function ENT:Initialize()
    if SERVER then
        self:SetModel(self.Model)
        self:PhysicsInit(SOLID_VBOX)
        self:SetUseType(SIMPLE_USE)

        -- Create inventory
        local inv = ix.inventory.Create(5, 5)
        inv.OnAuthorizeTransfer = function(inventory, client, oldInv, item)
            -- Check distance
            return client:GetPos():DistToSqr(self:GetPos()) < 10000
        end

        self.inventory = inv
    end
end

function ENT:Use(client)
    if SERVER and self.inventory then
        ix.storage.Open(client, self.inventory, {
            entity = self,
            name = "Storage Container",
            searchTime = 2,
            searchText = "Opening container..."
        })
    end
end

function ENT:OnRemove()
    if SERVER and self.inventory then
        ix.storage.Close(self.inventory)
    end
end
```

### NPC Vendor

```lua
-- Open shop inventory
function OpenShop(client, npc)
    local inv = npc.shopInventory

    ix.storage.Open(client, inv, {
        entity = npc,
        name = "Weapon Vendor",
        bMultipleUsers = true,
        data = {
            money = npc:GetMoney()
        }
    })
end
```

## See Also

- [Inventory API](inventory.md) - Inventory management
- [Item API](item.md) - Item system
- Source: `gamemode/core/libs/sh_storage.lua`
