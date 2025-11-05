# Item API (ix.item)

> **Reference**: `gamemode/core/libs/sh_item.lua`, `gamemode/core/meta/sh_item.lua`

The item API manages item definitions, instances, and operations. Items are the foundation of inventory systems.

## Core Concepts

Items in Helix:
- Defined in `items/` directories
- Have base types (weapons, consumables, etc.)
- Store custom data per instance
- Can have functions (OnUse, OnEquipped, etc.)
- Automatically sync to clients

## Item Structure

```lua
-- items/sh_health_kit.lua
ITEM.name = "Health Kit"
ITEM.description = "Restores 50 HP"
ITEM.model = "models/items/healthkit.mdl"
ITEM.category = "Medical"
ITEM.width = 1
ITEM.height = 1
ITEM.price = 100

function ITEM:OnUse(client)
    client:SetHealth(math.min(client:Health() + 50, client:GetMaxHealth()))
    client:Notify("Used health kit")
end
```

## Base Item Types

Located in `gamemode/items/base/`:
- `sh_base.lua` - Base for all items
- `sh_weapons.lua` - Weapon items
- `sh_ammo.lua` - Ammunition
- `sh_outfit.lua` - Clothing/armor
- `sh_bags.lua` - Containers

## Item Functions

### Defining Items

```lua
ITEM.name = "Name"
ITEM.description = "Description"
ITEM.model = "models/path.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Category"
ITEM.price = 100
ITEM.bDropOnDeath = true
```

### Item Hooks

```lua
-- Called when item is used
function ITEM:OnUse(client)
    client:Notify("Used item")
    return false  -- Don't remove item
end

-- Called when item is equipped
function ITEM:OnEquipped(client)
    client:Give(self.weaponClass)
end

-- Called when item is unequipped
function ITEM:OnUnequipped(client)
    client:StripWeapon(self.weaponClass)
end

-- Called when item is dropped
function ITEM:OnDropped(x, y, client)
    print("Item dropped")
end

-- Can item be transferred?
function ITEM:CanTransfer(oldInv, newInv)
    return true
end
```

## Item Methods

### item:GetName

```lua
local name = item:GetName()
```

Gets item display name.

### item:GetData

```lua
local value = item:GetData(key, default)
```

Gets item instance data.

**Example**:
```lua
-- Store custom data
item:SetData("durability", 100)

-- Retrieve data
local durability = item:GetData("durability", 100)
```

### item:Remove

**Realm**: Server

```lua
item:Remove()
```

Removes item from inventory.

### item:Transfer

**Realm**: Server

```lua
item:Transfer(newInv, x, y, client)
```

Moves item to another inventory.

## Complete Examples

### Consumable Item

```lua
ITEM.name = "Bread"
ITEM.description = "Restores hunger"
ITEM.model = "models/food/bread.mdl"
ITEM.category = "Food"

function ITEM:OnUse(client)
    local char = client:GetCharacter()
    local hunger = char:GetData("hunger", 100)

    char:SetData("hunger", math.min(hunger + 25, 100))
    client:Notify("You ate bread")

    return true  -- Remove item after use
end
```

### Weapon Item

```lua
ITEM.name = "Pistol"
ITEM.description = "9mm pistol"
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.base = "base_weapons"
ITEM.weaponClass = "weapon_pistol"
ITEM.width = 2
ITEM.height = 1
```

### Container Item

```lua
ITEM.name = "Backpack"
ITEM.description = "Stores items"
ITEM.model = "models/props_c17/suitcase_passenger_physics.mdl"
ITEM.base = "base_bags"
ITEM.width = 4
ITEM.height = 3
ITEM.invWidth = 6
ITEM.invHeight = 5
```

## See Also

- [Inventory API](inventory.md) - Item containers
- [Character API](character.md) - Character items
- [Storage API](storage.md) - Item transfer
- Source: `gamemode/core/libs/sh_item.lua`
