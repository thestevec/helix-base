# Creating Items for Your Schema

> **Reference**: `gamemode/core/libs/sh_item.lua`, `schema/items/`

This guide shows you how to create custom items for your schema. Items are the objects characters can carry, use, equip, and trade.

## ⚠️ Important: Use Helix Item System

**Always create items using Helix's item system** rather than creating custom inventory solutions. The framework provides:
- Automatic item registration from files
- Grid-based inventory system
- Database persistence for item instances
- Network synchronization
- Item actions (use, equip, drop)
- Data storage per item instance

## Core Concepts

### Items in Your Schema

Items define the objects in your world:
- **Half-Life 2 RP**: Rations, medkits, weapons, clothing, tokens
- **Star Wars RP**: Lightsabers, blasters, credits, datapads
- **Medieval RP**: Swords, shields, potions, gold coins

Each item has:
- Visual representation (model, icon)
- Physical properties (size, weight)
- Behavior (what happens when used)
- Per-instance data (durability, owner, etc.)

### File-Based Auto-Loading

**Reference**: `gamemode/core/libs/sh_item.lua`

Helix automatically loads item files from:
```
schema/items/sh_itemname.lua
```

The file prefix `sh_` makes it shared (loads on both server and client).

## Creating Your First Item

### Step 1: Create Item File

Create a new file in `schema/items/`:

```lua
-- File: schema/items/sh_medkit.lua
ITEM.name = "Medical Kit"
ITEM.description = "A standard medical kit that restores 50 health."
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Medical"
ITEM.price = 150

ITEM.functions.Use = {
    name = "Use",
    icon = "icon16/heart.png",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            local health = client:Health()
            local maxHealth = client:GetMaxHealth()

            if health >= maxHealth then
                client:Notify("You are already at full health")
                return false  -- Don't remove item
            end

            client:SetHealth(math.min(health + 50, maxHealth))
            client:EmitSound("items/medshot4.wav")
        end

        return true  -- Remove item after use
    end,
    OnCanRun = function(item)
        return item.player:Health() < item.player:GetMaxHealth()
    end
}
```

### Step 2: Test Your Item

Use console or admin command to give yourself the item:

```lua
-- In server console
lua_run player.GetByID(1):GetCharacter():GetInventory():Add("medkit")
```

Or create a test command:

```lua
-- File: schema/commands/sh_giveitem.lua
ix.command.Add("GiveItem", {
    description = "Give yourself a test item",
    arguments = {ix.type.string},
    OnRun = function(self, client, itemID)
        local inventory = client:GetCharacter():GetInventory()
        inventory:Add(itemID)
        return "Gave " .. itemID
    end
})
```

## Item Properties

### Required Properties

```lua
ITEM.name = "Item Name"          -- REQUIRED: Display name
ITEM.description = "Description"  -- REQUIRED: Tooltip text
ITEM.model = "models/path.mdl"   -- REQUIRED: World model
```

### Common Properties

```lua
ITEM.width = 1                   -- Grid width (default: 1)
ITEM.height = 1                  -- Grid height (default: 1)
ITEM.category = "Miscellaneous"  -- Organization category
ITEM.price = 100                 -- Default vendor price
ITEM.flag = "v"                  -- Required flag to use
ITEM.rarity = "rare"             -- Rarity tier (affects color)
ITEM.weight = 1                  -- Weight in kg (future use)
ITEM.skin = 0                    -- Model skin
ITEM.material = ""               -- Override material
```

### Icon Customization

```lua
-- Custom camera for icon render
ITEM.iconCam = {
    pos = Vector(0, 0, 200),
    ang = Angle(90, 0, 0),
    fov = 45
}

-- Or use a custom icon image
ITEM.icon = "vgui/inventory/item_icon.png"
```

## Complete Item Examples

### Consumable Item (Food)

```lua
-- File: schema/items/sh_food_bread.lua
ITEM.name = "Bread Loaf"
ITEM.description = "A fresh loaf of bread. Restores 25 hunger."
ITEM.model = "models/props_junk/garbage_bread001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Food"
ITEM.price = 20

ITEM.functions.Eat = {
    name = "Eat",
    icon = "icon16/cake.png",
    OnRun = function(item)
        local client = item.player
        local character = client:GetCharacter()

        if SERVER then
            local hunger = character:GetData("hunger", 100)

            if hunger >= 100 then
                client:Notify("You are not hungry")
                return false
            end

            character:SetData("hunger", math.min(hunger + 25, 100))
            client:EmitSound("npc/barnacle/barnacle_crunch" .. math.random(2, 3) .. ".wav")
            client:ChatPrint("You ate the bread and feel less hungry.")
        end

        return true  -- Remove item
    end,
    OnCanRun = function(item)
        local character = item.player:GetCharacter()
        return character:GetData("hunger", 100) < 100
    end
}
```

### Weapon Item

```lua
-- File: schema/items/sh_weapon_pistol.lua
ITEM.name = "9mm Pistol"
ITEM.description = "A standard 9mm semi-automatic pistol."
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.base = "base_weapons"  -- Inherit weapon functionality
ITEM.weapon = "weapon_pistol"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.price = 500
ITEM.flag = "v"  -- Requires vendor flag

-- Weapon items get Equip/Unequip automatically from base
```

### Armor Item (Outfit)

```lua
-- File: schema/items/sh_outfit_vest.lua
ITEM.name = "Kevlar Vest"
ITEM.description = "Ballistic protection vest. Provides 50 armor."
ITEM.model = "models/items/item_item_crate.mdl"
ITEM.base = "base_outfit"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Clothing"
ITEM.price = 300

ITEM.outfitCategory = "torso"
ITEM.bodyGroups = {
    ["torso"] = 1
}

-- Override OnSet to give armor
function ITEM:OnEquipped(client)
    client:SetArmor(50)
end

function ITEM:OnUnequipped(client)
    client:SetArmor(0)
end
```

### Container Item (Backpack)

```lua
-- File: schema/items/sh_bag_backpack.lua
ITEM.name = "Backpack"
ITEM.description = "A sturdy backpack for carrying extra items."
ITEM.model = "models/props_c17/suitcase_passenger_physics.mdl"
ITEM.base = "base_bags"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Storage"
ITEM.price = 200

-- Internal inventory size
ITEM.invWidth = 5
ITEM.invHeight = 5

-- Bags get Open action automatically from base
```

### Multi-Use Item (Advanced)

```lua
-- File: schema/items/sh_tool_multi.lua
ITEM.name = "Multi-Tool"
ITEM.description = "A versatile tool with multiple functions."
ITEM.model = "models/props_c17/tools_wrench01a.mdl"
ITEM.width = 1
ITEM.height = 2
ITEM.category = "Tools"
ITEM.price = 350

-- Initialize durability
function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("durability", 100)
    item:SetData("uses", 0)
end

-- Show durability in description
function ITEM:GetDescription()
    local durability = self:GetData("durability", 100)
    local uses = self:GetData("uses", 0)
    return self.description .. "\nDurability: " .. durability .. "%\nUses: " .. uses
end

-- Repair action
ITEM.functions.Repair = {
    name = "Repair Vehicle",
    icon = "icon16/wrench.png",
    OnRun = function(item)
        local client = item.player
        local trace = client:GetEyeTrace()
        local entity = trace.Entity

        if not IsValid(entity) or not entity:IsVehicle() then
            client:Notify("You must look at a vehicle")
            return false
        end

        if trace.HitPos:Distance(client:GetPos()) > 100 then
            client:Notify("Too far away")
            return false
        end

        if SERVER then
            -- Repair vehicle
            entity:SetHealth(entity:GetMaxHealth())
            client:EmitSound("items/battery_pickup.wav")

            -- Damage durability
            local durability = item:GetData("durability", 100) - 10
            item:SetData("durability", math.max(durability, 0))
            item:SetData("uses", item:GetData("uses", 0) + 1)

            -- Break tool if durability reaches 0
            if durability <= 0 then
                client:Notify("Your multi-tool broke!")
                return true  -- Remove item
            end
        end

        return false  -- Keep item
    end
}

-- Lock pick action
ITEM.functions.Lockpick = {
    name = "Pick Lock",
    icon = "icon16/key.png",
    OnRun = function(item)
        local client = item.player
        local trace = client:GetEyeTrace()
        local door = trace.Entity

        if not IsValid(door) or not door:IsDoor() then
            client:Notify("You must look at a door")
            return false
        end

        if SERVER then
            door:Fire("unlock")
            client:EmitSound("npc/metropolice/gear" .. math.random(1, 6) .. ".wav")

            -- Damage durability
            local durability = item:GetData("durability", 100) - 5
            item:SetData("durability", math.max(durability, 0))

            if durability <= 0 then
                return true  -- Break and remove
            end
        end

        return false
    end
}

-- Repair tool action
ITEM.functions.RepairTool = {
    name = "Repair Tool",
    icon = "icon16/cog.png",
    OnRun = function(item)
        if SERVER then
            local durability = item:GetData("durability", 0)

            if durability >= 100 then
                item.player:Notify("Tool is already at full durability")
                return false
            end

            item:SetData("durability", 100)
            item.player:EmitSound("items/battery_pickup.wav")
            item.player:Notify("Repaired multi-tool to 100%")
        end

        return false
    end,
    OnCanRun = function(item)
        return item:GetData("durability", 100) < 100
    end
}

-- Draw durability bar on icon
function ITEM:PaintOver(item, w, h)
    local durability = item:GetData("durability", 100) / 100
    local barColor = Color(255 * (1 - durability), 255 * durability, 0)

    draw.RoundedBox(0, 4, h - 8, w - 8, 4, Color(0, 0, 0, 200))
    draw.RoundedBox(0, 5, h - 7, (w - 10) * durability, 2, barColor)
end
```

## Using Base Items

### Available Base Items

**Reference**: `gamemode/items/base/`

- `base_weapons` - Equippable weapons
- `base_ammo` - Ammunition for weapons
- `base_bags` - Container items
- `base_outfit` - Clothing that changes appearance
- `base_pacoutfit` - PAC3 outfit items

### Inheriting from Base

```lua
-- Weapon inherits equip/unequip functionality
ITEM.base = "base_weapons"
ITEM.weapon = "weapon_ar2"

-- Ammo inherits load functionality
ITEM.base = "base_ammo"
ITEM.ammo = "ar2"
ITEM.ammoAmount = 30

-- Bag inherits open/close functionality
ITEM.base = "base_bags"
ITEM.invWidth = 6
ITEM.invHeight = 4
```

## Item Functions (Actions)

### Defining Actions

```lua
ITEM.functions.ActionName = {
    name = "Display Name",
    icon = "icon16/icon.png",
    OnRun = function(item)
        -- item.player = player who clicked
        -- Return true to remove item
        -- Return false to keep item
    end,
    OnCanRun = function(item)
        -- Return true to show button
        -- Return false to disable/hide button
    end,
    bShowPrompt = false  -- Show confirmation dialog
}
```

### Common Action Patterns

```lua
-- Use action (consumable)
ITEM.functions.Use = {
    name = "Use",
    OnRun = function(item)
        -- Apply effect
        if SERVER then
            item.player:SetHealth(100)
        end
        return true  -- Remove after use
    end
}

-- Equip action (already in base_weapons)
ITEM.functions.Equip = {
    name = "Equip",
    OnRun = function(item)
        if SERVER then
            item.player:Give(item.weapon)
        end
        return false  -- Don't remove
    end
}

-- Inspect action (no effect)
ITEM.functions.Inspect = {
    name = "Inspect",
    OnRun = function(item)
        item.player:ChatPrint("This item looks interesting...")
        return false  -- Don't remove
    end
}

-- Destroy action (with confirmation)
ITEM.functions.Destroy = {
    name = "Destroy",
    icon = "icon16/bin.png",
    bShowPrompt = true,  -- Ask for confirmation
    OnRun = function(item)
        return true  -- Remove item
    end
}
```

## Item Data Storage

### Setting and Getting Data

```lua
-- OnInstanced - called when item is created
function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("condition", 100)
    item:SetData("owner", "")
    item:SetData("timestamp", os.time())
end

-- Getting data
local condition = item:GetData("condition", 100)  -- Default to 100

-- Setting data (automatically saved)
item:SetData("condition", 50)
```

### Dynamic Item Name

```lua
function ITEM:GetName()
    local condition = self:GetData("condition", 100)
    local name = self.name

    if condition < 25 then
        name = name .. " (Broken)"
    elseif condition < 50 then
        name = name .. " (Damaged)"
    elseif condition < 75 then
        name = name .. " (Worn)"
    end

    return name
end
```

### Dynamic Description

```lua
function ITEM:GetDescription()
    local desc = self.description
    local owner = self:GetData("owner", "")

    desc = desc .. "\nCondition: " .. self:GetData("condition", 100) .. "%"

    if owner != "" then
        desc = desc .. "\nOwner: " .. owner
    end

    return desc
end
```

## Best Practices

### ✅ DO

- Create items in `schema/items/` with `sh_` prefix
- Always set name, description, and model
- Use appropriate base items when available
- Set reasonable width/height for grid
- Validate player input in SERVER realm
- Return true/false correctly in OnRun
- Use OnCanRun to disable invalid actions
- Test items in inventory
- Provide meaningful descriptions
- Use categories for organization

### ❌ DON'T

- Don't modify ITEM table at runtime (use SetData)
- Don't create items without checking inventory space
- Don't forget SERVER checks for important logic
- Don't trust client-side item data
- Don't create circular bag references
- Don't forget to return value in OnRun
- Don't bypass item system with custom tables
- Don't use item properties for instance data

## Common Patterns

### Stackable Items

```lua
function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("stack", 1)
end

function ITEM:GetName()
    local stack = self:GetData("stack", 1)
    if stack > 1 then
        return self.name .. " (x" .. stack .. ")"
    end
    return self.name
end

-- Merge stacks when transferring
function ITEM:CanTransfer(oldInv, newInv)
    for _, item in pairs(newInv:GetItems()) do
        if item.uniqueID == self.uniqueID and item:GetID() != self:GetID() then
            local theirStack = item:GetData("stack", 1)
            local myStack = self:GetData("stack", 1)
            local maxStack = self.maxStack or 99

            if theirStack + myStack <= maxStack then
                item:SetData("stack", theirStack + myStack)
                self:Remove()
                return false
            end
        end
    end

    return true
end
```

### Timed Effects

```lua
ITEM.functions.Use = {
    name = "Use",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            -- Immediate effect
            client:SetRunSpeed(300)

            -- Reset after duration
            timer.Simple(30, function()
                if IsValid(client) then
                    client:SetRunSpeed(240)
                    client:Notify("Speed boost wore off")
                end
            end)
        end

        return true
    end
}
```

### Faction-Specific Items

```lua
-- Set during item definition
ITEM.factionID = FACTION_POLICE

-- Or check in hooks
function ITEM:CanTransfer(oldInv, newInv)
    local owner = newInv:GetOwner()
    if owner then
        local character = owner:GetCharacter()
        if character:GetFaction() != FACTION_POLICE then
            return false
        end
    end
    return true
end
```

## Testing Items

1. **Give Item**: Use command or console to add item
2. **Test Actions**: Click all action buttons
3. **Test Transfer**: Move between inventories
4. **Test Drop**: Drop item in world
5. **Test Pickup**: Pick up dropped item
6. **Test Persistence**: Disconnect and reconnect
7. **Test with Multiple Players**: Trade items

## See Also

- [Item System](../systems/items.md) - Core item system reference
- [Inventory System](../systems/inventory.md) - Inventory management
- [Character System](../systems/character.md) - Character inventories
- [Schema Structure](structure.md) - Schema directory layout
- Source: `gamemode/core/libs/sh_item.lua`
- Source: `gamemode/items/base/` - Base item types
