# Complete Item Example

> **Reference**: `gamemode/items/base/sh_weapons.lua`, `gamemode/items/base/sh_ammo.lua`

This document provides complete, working examples of Helix items from simple to complex.

## ⚠️ Important: Use Built-in Helix Item System

**Always use Helix's item system** rather than creating custom entity-based items. The framework provides:
- Automatic database persistence
- Inventory management
- Network synchronization
- Item function system (use, drop, equip, etc.)
- Icon rendering in inventory
- Item hooks for custom behavior

## Example 1: Simple Consumable Item

The simplest item type - a one-use consumable that gives health.

### File Location

**Place in**: `schema/items/consumables/sh_medkit.lua` or `plugins/yourplugin/items/sh_medkit.lua`

### Complete Code

```lua
ITEM.name = "Medical Kit"
ITEM.description = "A medical kit that restores 50 health when used"
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 2  -- Inventory grid width
ITEM.height = 2  -- Inventory grid height
ITEM.category = "Medical"
ITEM.price = 150  -- Optional: price for vendors

-- Use function - consumed when used
ITEM.functions.use = {
    name = "Use",
    tip = "useTip",
    icon = "icon16/heart.png",
    OnRun = function(item)
        local client = item.player

        -- Heal the player
        local health = client:Health()
        local maxHealth = client:GetMaxHealth()

        if health >= maxHealth then
            client:Notify("You are already at full health!")
            return false  -- Don't consume item
        end

        client:SetHealth(math.min(health + 50, maxHealth))
        client:EmitSound("items/medshot4.wav")
        client:Notify("You restored 50 health")

        return true  -- Consume item (remove from inventory)
    end,
    OnCanRun = function(item)
        -- Only allow use if not at full health
        local client = item.player
        return IsValid(client) and client:Health() < client:GetMaxHealth()
    end
}
```

### Key Points

- **`return true`** in `OnRun` = item is consumed (removed)
- **`return false`** in `OnRun` = item is NOT consumed (stays in inventory)
- **`OnCanRun`** determines if function appears in context menu
- **`item.player`** is automatically set to the player interacting with item

## Example 2: Stackable Consumable with Data

A food item that stacks and tracks freshness.

### Complete Code

```lua
ITEM.name = "Ration"
ITEM.description = "A packaged ration. Restores hunger."
ITEM.model = "models/props_junk/garbage_metalcan001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Food"
ITEM.price = 25

-- Dynamic description based on item data
function ITEM:GetDescription()
    local freshness = self:GetData("freshness", 100)
    local desc = self.description

    if freshness < 50 then
        desc = desc .. "\nThis ration is starting to spoil."
    elseif freshness < 25 then
        desc = desc .. "\nThis ration is very old and spoiled!"
    end

    return desc
end

-- Called when item is spawned
function ITEM:OnInstanced(invID, x, y)
    -- Set initial freshness
    self:SetData("freshness", 100)
    self:SetData("packDate", os.time())
end

-- Use function
ITEM.functions.use = {
    name = "Eat",
    tip = "Consume this ration",
    icon = "icon16/cup.png",
    OnRun = function(item)
        local client = item.player
        local freshness = item:GetData("freshness", 100)

        -- Restore hunger (if using hunger system)
        local hungerAmount = math.floor(50 * (freshness / 100))

        client:EmitSound("npc/barnacle/barnacle_gulp2.wav")

        if freshness < 25 then
            -- Spoiled food might hurt you
            client:Notify("The ration was spoiled! You feel sick...")
            client:SetHealth(math.max(1, client:Health() - 10))
        else
            client:Notify("You consumed the ration and restored " .. hungerAmount .. " hunger")

            -- If using hunger plugin:
            -- local char = client:GetCharacter()
            -- char:SetData("hunger", math.min(100, char:GetData("hunger", 0) + hungerAmount))
        end

        return true  -- Consume item
    end
}

-- Inspect function to check freshness
ITEM.functions.inspect = {
    name = "Inspect",
    tip = "Check the ration's freshness",
    icon = "icon16/magnifier.png",
    OnRun = function(item)
        local freshness = item:GetData("freshness", 100)
        local packDate = item:GetData("packDate", os.time())
        local age = math.floor((os.time() - packDate) / 3600)  -- Hours old

        item.player:Notify("Freshness: " .. freshness .. "% (Packed " .. age .. " hours ago)")

        return false  -- Don't consume item
    end
}
```

### Key Points

- **`GetData()`/`SetData()`**: Store per-item data
- **`OnInstanced()`**: Called when item is created
- **`GetDescription()`**: Dynamic descriptions based on item state
- **Multiple functions**: Items can have many context menu actions

## Example 3: Equippable Weapon Item

A weapon that can be equipped and unequipped.

### Complete Code

```lua
ITEM.name = "9mm Pistol"
ITEM.description = "A reliable sidearm"
ITEM.model = "models/weapons/w_pist_fiveseven.mdl"
ITEM.class = "weapon_pistol"  -- Weapon class to give
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.weaponCategory = "sidearm"
ITEM.isWeapon = true
ITEM.price = 300

-- Visual indicator when equipped
if CLIENT then
    function ITEM:PaintOver(item, w, h)
        if item:GetData("equip") then
            -- Draw green indicator
            surface.SetDrawColor(110, 255, 110, 100)
            surface.DrawRect(w - 14, h - 14, 8, 8)
        end
    end

    function ITEM:PopulateTooltip(tooltip)
        if self:GetData("equip") then
            local name = tooltip:GetRow("name")
            name:SetBackgroundColor(derma.GetColor("Success", tooltip))
        end
    end
end

-- Equip function
ITEM.functions.Equip = {
    name = "Equip",
    tip = "Equip this weapon",
    icon = "icon16/tick.png",
    OnRun = function(item)
        local client = item.player

        -- Remove any other equipped weapon in this category
        client.carryWeapons = client.carryWeapons or {}

        local weapon = client.carryWeapons[item.weaponCategory]
        if IsValid(weapon) then
            -- Unequip previous weapon in this slot
            client:StripWeapon(weapon:GetClass())
        end

        -- Give the weapon
        local wep = client:Give(item.class)

        if IsValid(wep) then
            client.carryWeapons[item.weaponCategory] = wep

            -- Restore ammo
            local ammo = item:GetData("ammo", 0)
            wep:SetClip1(ammo)

            item:SetData("equip", true)
            client:EmitSound("items/ammo_pickup.wav", 80)
        end

        return false  -- Don't consume
    end,
    OnCanRun = function(item)
        local client = item.player

        return !IsValid(item.entity) and IsValid(client) and
               item:GetData("equip") != true and
               hook.Run("CanPlayerEquipItem", client, item) != false
    end
}

-- Unequip function
ITEM.functions.EquipUn = {
    name = "Unequip",
    tip = "Unequip this weapon",
    icon = "icon16/cross.png",
    OnRun = function(item)
        local client = item.player

        client.carryWeapons = client.carryWeapons or {}

        local weapon = client.carryWeapons[item.weaponCategory]

        if !IsValid(weapon) then
            weapon = client:GetWeapon(item.class)
        end

        if IsValid(weapon) then
            -- Save ammo
            item:SetData("ammo", weapon:Clip1())

            client:StripWeapon(item.class)
            client.carryWeapons[item.weaponCategory] = nil
            client:EmitSound("items/ammo_pickup.wav", 80)
        end

        item:SetData("equip", nil)

        return false  -- Don't consume
    end,
    OnCanRun = function(item)
        local client = item.player

        return !IsValid(item.entity) and IsValid(client) and
               item:GetData("equip") == true and
               hook.Run("CanPlayerUnequipItem", client, item) != false
    end
}

-- Auto-unequip when dropped
ITEM:Hook("drop", function(item)
    if item:GetData("equip") then
        local client = item.player

        if IsValid(client) then
            client.carryWeapons = client.carryWeapons or {}

            local weapon = client.carryWeapons[item.weaponCategory]

            if IsValid(weapon) then
                item:SetData("ammo", weapon:Clip1())
                client:StripWeapon(item.class)
                client.carryWeapons[item.weaponCategory] = nil
            end
        end

        item:SetData("equip", nil)
    end
end)
```

### Key Points

- **`item:GetData("equip")`**: Track equip state
- **`ITEM:Hook("drop")`**: Custom behavior on item events
- **Client-side `PaintOver()`**: Visual indicators in inventory
- **`OnCanRun`**: Show/hide functions based on state

## Example 4: Container Item (Bag)

An item that holds other items (creates nested inventory).

### Complete Code

```lua
ITEM.name = "Backpack"
ITEM.description = "A backpack that can hold items"
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Storage"
ITEM.invWidth = 4  -- Internal inventory width
ITEM.invHeight = 4  -- Internal inventory height
ITEM.price = 200

-- Create inventory when item is spawned
function ITEM:OnInstanced(invID, x, y, item)
    -- Create a new inventory for this bag
    ix.inventory.New(0, self.uniqueID, function(inventory)
        if !inventory then
            return
        end

        inventory:SetSize(self.invWidth, self.invHeight)

        local itemObj = ix.item.instances[item]

        if itemObj then
            itemObj:SetData("invID", inventory:GetID())
        end
    end)
end

-- Open the bag's inventory
ITEM.functions.open = {
    name = "Open",
    tip = "Open this bag",
    icon = "icon16/briefcase.png",
    OnRun = function(item)
        local client = item.player
        local invID = item:GetData("invID")

        if invID then
            local inventory = ix.item.inventories[invID]

            if inventory then
                -- Open the inventory
                ix.storage.Open(client, inventory, {
                    name = item.name,
                    entity = item.entity
                })

                return false  -- Don't consume
            end
        end

        client:Notify("This bag has no storage!")
        return false
    end,
    OnCanRun = function(item)
        return !IsValid(item.entity) and item:GetData("invID")
    end
}

-- Delete inventory when bag is removed
function ITEM:OnRemoved()
    local invID = self:GetData("invID")

    if invID then
        local inventory = ix.item.inventories[invID]

        if inventory then
            -- Remove all items from bag inventory
            inventory:Destroy()
        end
    end
end

-- Prevent bag from being placed in another bag
function ITEM:CanTransfer(oldInventory, newInventory)
    -- Check if target inventory belongs to another bag
    if newInventory then
        for _, item in pairs(ix.item.instances) do
            if item:GetData("invID") == newInventory:GetID() then
                return false  -- Can't put bag in bag
            end
        end
    end

    return true
end
```

### Key Points

- **`ix.inventory.New()`**: Create nested inventory
- **`ix.storage.Open()`**: Open inventory for player
- **`OnRemoved()`**: Clean up when item deleted
- **`CanTransfer()`**: Prevent bags in bags

## Example 5: Craftable Item with Hooks

An item that uses hooks for complex behavior.

### Complete Code

```lua
ITEM.name = "Lockpick"
ITEM.description = "Used to pick locked doors"
ITEM.model = "models/props_c17/tools_wrench01a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Tools"
ITEM.useSound = "doors/door_locked2.wav"
ITEM.price = 75

-- Use on door entities
ITEM.functions.use = {
    name = "Use on Door",
    tip = "Attempt to pick a lock",
    icon = "icon16/door_open.png",
    OnRun = function(item)
        local client = item.player
        local trace = client:GetEyeTrace()
        local entity = trace.Entity

        if !IsValid(entity) or !entity:isDoor() then
            client:Notify("You must be looking at a door!")
            return false
        end

        if entity:GetPos():Distance(client:GetPos()) > 100 then
            client:Notify("You are too far from the door!")
            return false
        end

        if !entity:GetNetVar("locked") then
            client:Notify("This door is not locked!")
            return false
        end

        -- Start lockpicking
        client:SetAction("Lockpicking...", 5, function()
            if IsValid(entity) then
                -- Success chance based on item data
                local uses = item:GetData("uses", 0)
                local successChance = math.max(30, 70 - (uses * 5))

                if math.random(100) <= successChance then
                    entity:Fire("unlock")
                    client:EmitSound("doors/door_latch3.wav")
                    client:Notify("Successfully picked the lock!")

                    -- Track uses
                    item:SetData("uses", uses + 1)

                    -- Break lockpick after too many uses
                    if uses >= 10 then
                        client:Notify("Your lockpick broke!")
                        return true  -- Consume item
                    end
                else
                    client:EmitSound(item.useSound)
                    client:Notify("Failed to pick the lock!")
                    item:SetData("uses", uses + 1)
                end
            end
        end)

        return false  -- Don't consume (unless it breaks)
    end
}

-- Show durability
function ITEM:GetDescription()
    local uses = self:GetData("uses", 0)
    local durability = math.max(0, 100 - (uses * 10))

    return self.description .. "\nDurability: " .. durability .. "%"
end

-- Paint durability on item
if CLIENT then
    function ITEM:PaintOver(item, w, h)
        local uses = item:GetData("uses", 0)
        local durability = math.max(0, 10 - uses)

        if durability <= 3 then
            -- Draw red warning
            surface.SetDrawColor(255, 0, 0, 200)
            surface.DrawRect(2, 2, 8, 8)
        end

        draw.SimpleText(durability .. "/10", "DermaDefault", w - 5, h - 5, color_white, TEXT_ALIGN_RIGHT, TEXT_ALIGN_BOTTOM, 1, color_black)
    end
end
```

### Key Points

- **`client:SetAction()`**: Progress bar for timed actions
- **Item durability**: Track uses with `GetData()`/`SetData()`
- **Conditional consumption**: Return `true` only when broken
- **Visual feedback**: Paint durability on icon

## ⚠️ Do NOT

```lua
-- WRONG: Don't create custom item entities
local item = ents.Create("prop_physics")
item.itemData = {health = 50}
-- Use ix.item.Spawn() instead!

-- WRONG: Don't store items in tables
MyItems = {}
MyItems[client:SteamID()] = {medkit = 5}
-- Use character inventory system!

-- WRONG: Don't give items without inventory system
client:GiveItem("medkit")  -- This function doesn't exist
-- Use inventory:Add("medkit") instead!

-- WRONG: Don't forget to return in OnRun
ITEM.functions.use = {
    OnRun = function(item)
        item.player:SetHealth(100)
        -- Missing return! Item won't be consumed
    end
}
```

## Best Practices

### ✅ DO

- Return `true` in `OnRun` to consume item
- Return `false` in `OnRun` to keep item
- Use `GetData()`/`SetData()` for per-item state
- Check `IsValid()` before using entities
- Use `OnCanRun` to show/hide functions
- Implement `GetDescription()` for dynamic descriptions
- Use `PaintOver()` for visual indicators
- Clean up in `OnRemoved()`
- Place items in proper categories
- Set appropriate `width` and `height`

### ❌ DON'T

- Don't create custom item entities
- Don't bypass inventory system
- Don't forget realm checks for CLIENT code
- Don't trust `item.player` without `IsValid()`
- Don't store critical data only on client
- Don't forget to register items in proper folders
- Don't use global item tables
- Don't forget to handle edge cases

## See Also

- [Item System](../systems/items.md) - Complete item documentation
- [Inventory System](../systems/inventory.md) - Working with inventories
- [Item Base Classes](../items/base-items.md) - Base item types
- [Storage System](../libraries/storage.md) - Container items
- Source: `gamemode/items/base/sh_weapons.lua` - Weapon items
- Source: `gamemode/items/base/sh_ammo.lua` - Ammo items
- Source: `gamemode/items/base/sh_bags.lua` - Container items
