# Creating Custom Items

> **Guide**: Step-by-step tutorial for creating custom items in Helix

This guide walks you through creating custom items for your Helix schema, from simple consumables to complex equippable items.

## ⚠️ Important: Always Use Base Items

**Never create items from scratch.** Always extend from one of Helix's base item types:

- `base_weapons` - For equippable weapons
- `base_ammo` - For ammunition boxes
- `base_bags` - For container items
- `base_outfit` - For model-changing clothing
- `base_pacoutfit` - For PAC3 accessories

Base items provide automatic persistence, networking, and integration with Helix's systems.

## Quick Start: Simple Consumable

Let's create a simple health pack item.

### Step 1: Create the File

Create a new file in your schema's `items/` directory:

```
schema/items/sh_healthpack.lua
```

The `sh_` prefix means it runs on both server and client (shared).

### Step 2: Define Basic Properties

```lua
-- schema/items/sh_healthpack.lua
ITEM.name = "Health Pack"
ITEM.description = "Restores 50 health when used."
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 1   -- Inventory width (grid cells)
ITEM.height = 1  -- Inventory height (grid cells)
ITEM.category = "Medical"  -- Item category
```

### Step 3: Add Use Function

```lua
ITEM.functions.Use = {
    name = "Use",
    tip = "useTip",  -- Translation key
    icon = "icon16/heart.png",
    OnRun = function(item)
        local client = item.player
        local health = client:Health()

        -- Restore 50 health (max 100)
        client:SetHealth(math.min(health + 50, 100))
        client:EmitSound("items/medshot4.wav")

        return true  -- Remove item after use
    end,
    OnCanRun = function(item)
        -- Only allow if player exists and is injured
        return IsValid(item.player) and item.player:Health() < 100
    end
}
```

### Step 4: Test the Item

1. Start your server
2. Spawn the item: `/itemgive yourname healthpack`
3. Use it from inventory

That's it! You've created a basic consumable item.

## Item Properties Reference

### Required Properties

```lua
ITEM.name = "Item Name"              -- Display name
ITEM.description = "Description"     -- Item description
ITEM.model = "models/item.mdl"       -- World/inventory model
```

### Common Properties

```lua
ITEM.width = 1                       -- Inventory width (default: 1)
ITEM.height = 1                      -- Inventory height (default: 1)
ITEM.category = "Category"           -- Item category
ITEM.price = 100                     -- Vendor price
ITEM.flag = "v"                      -- Required flag to use
ITEM.base = "base_weapons"           -- Base item type
ITEM.uniqueID = "custom_item"        -- Unique identifier (auto-generated)
```

### Optional Properties

```lua
ITEM.iconCam = {                     -- Custom icon camera
    pos = Vector(0, 0, 0),
    ang = Angle(0, 0, 0),
    fov = 45
}

ITEM.skin = 0                        -- Model skin
ITEM.bodyGroups = {}                 -- Model bodygroups

-- Sounds
ITEM.useSound = "items/battery_pickup.wav"
ITEM.dropSound = "physics/body/body_medium_impact_soft1.wav"
```

## Item Functions

Item functions appear in the context menu when right-clicking an item.

### Function Structure

```lua
ITEM.functions.FunctionName = {
    name = "Display Name",           -- Shown in menu
    tip = "translationKey",          -- Tooltip translation key
    icon = "icon16/accept.png",      -- Icon path
    OnRun = function(item)
        -- Function logic here
        return true  -- true = remove item, false = keep item
    end,
    OnCanRun = function(item)
        -- Return true if function can run
        return true
    end
}
```

### OnRun Return Values

```lua
-- Remove item after use
return true

-- Keep item in inventory
return false

-- Keep item and update it
item:SetData("uses", uses - 1)
return false
```

### Accessing Player and Character

```lua
OnRun = function(item)
    local client = item.player           -- The player entity
    local character = client:GetCharacter()  -- The character instance

    -- Do something with player/character
    character:SetMoney(character:GetMoney() + 100)

    return true
end
```

## Item Data Persistence

Use `GetData()` and `SetData()` to store persistent item properties.

### Storing Data

```lua
-- Store any serializable data
item:SetData("uses", 3)
item:SetData("owner", client:SteamID())
item:SetData("customName", "Bob's Sword")
item:SetData("equipped", true)

-- Data is automatically saved to database
```

### Retrieving Data

```lua
-- Get data with default value
local uses = item:GetData("uses", 3)  -- Returns 3 if not set
local owner = item:GetData("owner")   -- Returns nil if not set

-- Use in description
function ITEM:GetDescription()
    local uses = self:GetData("uses", 3)
    return self.description .. "\nUses: " .. uses
end
```

### Example: Item with Durability

```lua
ITEM.name = "Lockpick"
ITEM.description = "A tool for picking locks."
ITEM.model = "models/props/lockpick.mdl"

ITEM.functions.Use = {
    name = "Pick Lock",
    OnRun = function(item)
        local uses = item:GetData("uses", 3)

        -- Use the lockpick
        item.player:ChatPrint("You picked the lock!")

        if uses <= 1 then
            item.player:Notify("The lockpick broke!")
            return true  -- Remove item
        end

        -- Decrease uses
        item:SetData("uses", uses - 1)
        return false  -- Keep item
    end,
    OnCanRun = function(item)
        -- Check if looking at a locked door
        local trace = item.player:GetEyeTrace()
        local entity = trace.Entity

        return IsValid(entity) and entity:GetClass() == "prop_door_rotating"
               and entity:GetInternalVariable("m_bLocked")
    end
}

-- Show uses in description
function ITEM:GetDescription()
    local uses = self:GetData("uses", 3)
    return self.description .. string.format("\n\nDurability: %d/3", uses)
end
```

## Item Hooks

Hooks allow items to react to events.

### Available Hooks

```lua
-- When item is dropped
ITEM:Hook("drop", function(item)
    print("Item dropped!")
end)

-- When item is picked up
ITEM:Hook("take", function(item)
    print("Item picked up!")
end)

-- When item is transferred
ITEM:Hook("transfer", function(item, oldInv, newInv)
    print("Item transferred!")
end)
```

### Example: Self-Destructing Item

```lua
ITEM.name = "Timed Explosive"
ITEM.description = "Explodes 30 seconds after being picked up!"

ITEM:Hook("take", function(item)
    -- Start a timer when picked up
    timer.Simple(30, function()
        if item and item.player and IsValid(item.player) then
            -- Explode!
            local pos = item.player:GetPos()
            local effect = EffectData()
            effect:SetOrigin(pos)
            util.Effect("Explosion", effect)

            util.BlastDamage(item.player, item.player, pos, 200, 50)

            -- Remove item
            item:Remove()
        end
    end)
end)
```

## Item Lifecycle Functions

These functions are called automatically by the framework.

### OnInstanced(invID, x, y)

Called when a new instance is created:

```lua
function ITEM:OnInstanced(invID, x, y)
    -- Set default data for new items
    self:SetData("uses", 5)
    self:SetData("created", os.time())
end
```

### OnLoadout()

Called when player spawns with equipped item:

```lua
function ITEM:OnLoadout()
    if self:GetData("equip") then
        -- Re-equip weapon on spawn
        self:Equip(self.player)
    end
end
```

### OnSave()

Called before character saves:

```lua
function ITEM:OnSave()
    -- Save current state
    local weapon = self.player:GetWeapon(self.class)
    if IsValid(weapon) then
        self:SetData("ammo", weapon:Clip1())
    end
end
```

### OnRemoved()

Called when item is permanently deleted:

```lua
function ITEM:OnRemoved()
    -- Cleanup on deletion
    local owner = self:GetOwner()
    if IsValid(owner) then
        owner:ChatPrint("Your item was destroyed!")
    end
end
```

### CanTransfer(oldInventory, newInventory)

Control whether item can be moved:

```lua
function ITEM:CanTransfer(oldInventory, newInventory)
    -- Prevent equipped items from transferring
    if self:GetData("equip") then
        return false
    end

    -- Prevent dropping quest items
    if self.isQuestItem then
        return false
    end

    return true
end
```

## Creating Items Based on Base Types

### Weapon Item

```lua
ITEM.name = "AK-47"
ITEM.description = "A powerful assault rifle."
ITEM.model = "models/weapons/w_rif_ak47.mdl"
ITEM.class = "weapon_ak47"  -- Weapon class
ITEM.width = 4
ITEM.height = 2
ITEM.category = "Weapons"
ITEM.weaponCategory = "primary"  -- Weapon slot
ITEM.base = "base_weapons"
```

See [Weapons](weapons.md) for full details.

### Ammunition Item

```lua
ITEM.name = "Rifle Ammo"
ITEM.description = "A box containing %s rifle rounds."
ITEM.model = "models/Items/BoxMRounds.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Ammunition"
ITEM.ammo = "ar2"          -- Ammo type
ITEM.ammoAmount = 30       -- Rounds per box
ITEM.base = "base_ammo"
```

See [Ammunition](ammunition.md) for full details.

### Bag Item

```lua
ITEM.name = "Backpack"
ITEM.description = "A sturdy backpack."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 3
ITEM.height = 3
ITEM.invWidth = 6          -- Internal width
ITEM.invHeight = 5         -- Internal height
ITEM.category = "Storage"
ITEM.isBag = true
ITEM.base = "base_bags"
```

See [Bags](bags.md) for full details.

### Outfit Item

```lua
ITEM.name = "Police Uniform"
ITEM.description = "A police officer's uniform."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 3
ITEM.category = "Clothing"
ITEM.outfitCategory = "model"  -- Outfit slot
ITEM.replacement = "models/player/police.mdl"  -- New model
ITEM.base = "base_outfit"
```

See [Outfits](outfits.md) for full details.

## Advanced Patterns

### Multi-Function Item

Create items with multiple use options:

```lua
ITEM.name = "Swiss Army Knife"
ITEM.description = "A versatile tool with multiple functions."

ITEM.functions.Cut = {
    name = "Cut Rope",
    OnRun = function(item)
        item.player:ChatPrint("You cut the rope.")
        return false
    end
}

ITEM.functions.OpenCan = {
    name = "Open Can",
    OnRun = function(item)
        item.player:ChatPrint("You opened the can.")
        return false
    end
}

ITEM.functions.Screwdriver = {
    name = "Use Screwdriver",
    OnRun = function(item)
        item.player:ChatPrint("You unscrewed the panel.")
        return false
    end
}
```

### Stackable Items

Items with the same uniqueID stack automatically. Control stacking:

```lua
-- Items stack by default if they have same uniqueID
-- Prevent stacking by making each instance unique:

function ITEM:OnInstanced()
    -- Make each instance unique (prevents stacking)
    self:SetData("instanceID", util.CRC(tostring(self)))
end
```

### Tradeable Items

Allow player-to-player trading:

```lua
ITEM.functions.Give = {
    name = "Give to Player",
    OnRun = function(item)
        local client = item.player

        -- Get player looking at
        local trace = client:GetEyeTrace()
        local target = trace.Entity

        if IsValid(target) and target:IsPlayer() and target:GetCharacter() then
            local targetInv = target:GetCharacter():GetInventory()

            -- Transfer to target
            item:Transfer(targetInv:GetID(), nil, nil, client)

            client:Notify("You gave the item to " .. target:Name())
            target:Notify(client:Name() .. " gave you " .. item.name)
        else
            client:Notify("No player in range!")
        end

        return false
    end,
    OnCanRun = function(item)
        return !IsValid(item.entity)
    end
}
```

### Craftable Items

Items that can be combined:

```lua
-- Wood item
ITEM.name = "Wood Planks"
ITEM.uniqueID = "wood"

ITEM.functions.Combine = {
    name = "Craft Wooden Box",
    OnRun = function(item, data)
        local nails = data[1]  -- Second item ID

        -- Remove both items
        ix.item.instances[nails]:Remove()

        -- Give crafted item
        local inv = item.player:GetCharacter():GetInventory()
        inv:Add("wooden_box")

        item.player:Notify("You crafted a wooden box!")

        return true  -- Remove wood
    end,
    OnCanRun = function(item, data)
        if not data or not data[1] then return false end

        local targetItem = ix.item.instances[data[1]]
        return targetItem and targetItem.uniqueID == "nails"
    end
}
```

## Best Practices

### ✅ DO

- Always extend from a base item type
- Use `GetData()`/`SetData()` for item properties
- Validate `item.player` before using it
- Return `true` to remove item, `false` to keep it
- Use `OnCanRun` to check if function can execute
- Test items thoroughly before deploying
- Use descriptive `uniqueID` values
- Document complex item behaviors

### ❌ DON'T

- Don't create items without a base type
- Don't store item data in global tables
- Don't access `item.player` without checking `IsValid()`
- Don't forget to set `width` and `height`
- Don't hardcode values (use item properties)
- Don't bypass the item system with custom tables
- Don't forget that `OnRun` can return false
- Don't make items too large (keep them balanced)

## Testing Items

### Spawning Items for Testing

```lua
-- In-game console or admin command
/itemgive yourname itemuniqueID
/itemgive yourname healthpack 5  -- Give 5 healthpacks
```

### Debugging Items

```lua
-- Add debug prints
ITEM.functions.Use = {
    OnRun = function(item)
        print("Item used by:", item.player:Name())
        print("Item data:", table.ToString(item.data, "Item Data", true))

        -- Your logic here

        return true
    end
}
```

### Common Issues

**Item not appearing**: Check console for errors, verify file path is correct.

**Function not showing**: Ensure `OnCanRun` returns true, check for typos.

**Data not saving**: Use `SetData()` not regular Lua tables.

**Item not stacking**: Items stack by uniqueID, make sure they match.

## See Also

- [Base Items](base-items.md) - Base item types overview
- [Weapons](weapons.md) - Weapon items
- [Ammunition](ammunition.md) - Ammo items
- [Bags](bags.md) - Container items
- [Outfits](outfits.md) - Clothing items
- [PAC Outfits](pac-outfits.md) - PAC3 accessory items
- [Item System](../systems/items.md) - Core item system documentation
- [Inventory System](../systems/inventory.md) - Inventory management
