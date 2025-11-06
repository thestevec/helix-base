# Creating Custom Items - Complete Walkthrough

> **Reference**: `gamemode/core/libs/sh_item.lua`, `gamemode/items/base/`

Comprehensive guide to creating custom items in Helix, from basic consumables to complex interactive items.

## ⚠️ Important: Use Helix's Item System

**Always use Helix's built-in item framework** rather than creating custom entities or tables. The framework provides:
- Automatic database persistence
- Network synchronization
- Inventory integration
- Item functions and interactions
- World entity management
- Icon rendering

## What You'll Learn

This guide covers creating:
1. **Basic Consumable** - Food item that restores health
2. **Equipment Item** - Armor that provides protection
3. **Usable Item** - Key that opens doors
4. **Stackable Item** - Currency or resources
5. **Container Item** - Bag with inventory space

**Estimated time**: 30 minutes

## Prerequisites

- Schema set up (see [Setup Guide](setup.md))
- Basic Lua knowledge
- Understanding of inventory system

## Item Structure Overview

Every item needs:
```lua
ITEM.name = "Item Name"           -- Display name (required)
ITEM.description = "Description"   -- Tooltip text (required)
ITEM.model = "models/path.mdl"     -- 3D model (required)
ITEM.width = 1                     -- Grid width (default: 1)
ITEM.height = 1                    -- Grid height (default: 1)
```

Optional but common:
```lua
ITEM.category = "Category"         -- Organization
ITEM.price = 100                   -- Default price
ITEM.flag = "v"                    -- Permission flag
ITEM.base = "base_item"            -- Inherit from base
```

## Part 1: Basic Consumable (Food Item)

Let's create a food item that restores health when eaten.

### Step 1: Create Item File

**File**: `schema/items/sh_bread.lua`

```lua
ITEM.name = "Bread"
ITEM.description = "A loaf of fresh bread. Restores 25 health."
ITEM.model = "models/props_junk/garbage_bread001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Food"
ITEM.price = 10
```

### Step 2: Add Use Function

Add the interaction that happens when player uses the item:

```lua
ITEM.name = "Bread"
ITEM.description = "A loaf of fresh bread. Restores 25 health."
ITEM.model = "models/props_junk/garbage_bread001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Food"
ITEM.price = 10

-- Define the "use" function
ITEM.functions.Use = {
    name = "Eat",                    -- Button text
    tip = "Eat the bread",           -- Tooltip
    icon = "icon16/cake.png",        -- Icon (Material icons)

    -- What happens when used
    OnRun = function(item)
        local client = item.player   -- Player using the item

        -- Server-side only
        if SERVER then
            local health = client:Health()
            local maxHealth = client:GetMaxHealth()

            -- Check if already at full health
            if health >= maxHealth then
                client:Notify("You're already at full health!")
                return false  -- Don't consume item
            end

            -- Restore 25 health
            local newHealth = math.min(health + 25, maxHealth)
            client:SetHealth(newHealth)

            -- Play eating sound
            client:EmitSound("npc/barnacle/barnacle_crunch2.wav")

            -- Show message
            client:Notify("You ate the bread and restored some health.")
        end

        return true  -- Remove item after use
    end,

    -- Check if can use (optional)
    OnCanRun = function(item)
        local client = item.player

        -- Can only use if not at full health
        return client:Health() < client:GetMaxHealth()
    end
}
```

### Step 3: Test the Item

```
// Reload schema
sh_rebuildschema

// Give yourself the item
ix_giveitem "YourName" sh_bread

// Open inventory (Tab), right-click bread, click "Eat"
```

✅ **Checkpoint**: You've created a working consumable item!

## Part 2: Equipment Item (Armor)

Create armor that provides damage protection.

### Step 1: Create Armor File

**File**: `schema/items/sh_kevlar.lua`

```lua
ITEM.name = "Kevlar Vest"
ITEM.description = "Provides protection against bullets. Reduces damage by 30%."
ITEM.model = "models/props_c17/suitcase_passenger_physics.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Equipment"
ITEM.price = 500
ITEM.flag = "v"  -- Requires vendor flag to use

-- Custom icon camera
ITEM.iconCam = {
    pos = Vector(0, 0, 200),
    ang = Angle(90, 0, 0),
    fov = 12
}
```

### Step 2: Add Equip/Unequip Functions

```lua
-- Equip function
ITEM.functions.Equip = {
    name = "Equip",
    tip = "Put on the vest",
    icon = "icon16/shield_add.png",

    OnRun = function(item)
        local client = item.player
        local character = client:GetCharacter()

        if SERVER then
            -- Check if already wearing armor
            local currentArmor = character:GetData("equippedArmor")
            if currentArmor then
                client:Notify("You're already wearing armor!")
                return false
            end

            -- Mark as equipped
            character:SetData("equippedArmor", item:GetID())

            -- Set armor value
            client:SetArmor(50)

            -- Visual feedback
            client:EmitSound("items/ammopickup.wav")
            client:Notify("You equipped the kevlar vest.")
        end

        return false  -- Don't remove item, just mark as equipped
    end,

    OnCanRun = function(item)
        local client = item.player
        local character = client:GetCharacter()

        -- Can't equip if already wearing armor
        return not character:GetData("equippedArmor")
    end
}

-- Unequip function
ITEM.functions.Unequip = {
    name = "Unequip",
    tip = "Take off the vest",
    icon = "icon16/shield_delete.png",

    OnRun = function(item)
        local client = item.player
        local character = client:GetCharacter()

        if SERVER then
            -- Check if this armor is equipped
            local equippedID = character:GetData("equippedArmor")
            if equippedID ~= item:GetID() then
                client:Notify("This armor is not equipped!")
                return false
            end

            -- Remove equipped status
            character:SetData("equippedArmor", nil)

            -- Remove armor value
            client:SetArmor(0)

            -- Visual feedback
            client:EmitSound("items/ammo_pickup.wav")
            client:Notify("You removed the kevlar vest.")
        end

        return false  -- Don't remove item
    end,

    OnCanRun = function(item)
        local client = item.player
        local character = client:GetCharacter()

        -- Can only unequip if this specific armor is equipped
        return character:GetData("equippedArmor") == item:GetID()
    end
}
```

### Step 3: Add Damage Reduction Hook

To make armor actually protect, we need to create a plugin or add to schema:

**schema/sv_hooks.lua:**
```lua
function Schema:EntityTakeDamage(target, dmgInfo)
    if target:IsPlayer() then
        local character = target:GetCharacter()
        if character then
            -- Check if wearing armor
            local armorID = character:GetData("equippedArmor")
            if armorID then
                -- Get the armor item
                local armor = ix.item.instances[armorID]
                if armor then
                    -- Reduce damage by 30%
                    local damage = dmgInfo:GetDamage()
                    dmgInfo:SetDamage(damage * 0.7)

                    -- Optional: Damage the armor over time
                    -- armor:SetData("durability", armor:GetData("durability", 100) - 5)
                end
            end
        end
    end
end
```

✅ **Checkpoint**: You've created an equipable armor item!

## Part 3: Usable Item (Door Key)

Create a key that can open specific doors.

**File**: `schema/items/sh_officekey.lua`

```lua
ITEM.name = "Office Key"
ITEM.description = "A key with a tag reading 'OFFICE 205'."
ITEM.model = "models/props_lab/keycard.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Keys"
ITEM.price = 0  -- Keys shouldn't be buyable

-- Store which doors this key opens
ITEM.doorIDs = {205, 206}  -- Door IDs this key works on

ITEM.functions.Use = {
    name = "Use Key",
    tip = "Use on a door",
    icon = "icon16/key.png",

    OnRun = function(item)
        local client = item.player

        if SERVER then
            -- Get the door player is looking at
            local trace = client:GetEyeTrace()
            local door = trace.Entity

            -- Check if looking at a door
            if not IsValid(door) or not door:IsDoor() then
                client:Notify("You must be looking at a door!")
                return false
            end

            -- Check distance
            if trace.StartPos:Distance(trace.HitPos) > 100 then
                client:Notify("You're too far from the door!")
                return false
            end

            -- Get door ID (from door system)
            local doorID = door.ixDoorID
            if not doorID then
                client:Notify("This door doesn't have a lock!")
                return false
            end

            -- Check if key fits this door
            if not table.HasValue(item.doorIDs, doorID) then
                client:Notify("This key doesn't fit this door!")
                return false
            end

            -- Toggle door lock
            local isLocked = door:GetSaveTable().m_bLocked

            if isLocked then
                door:Fire("Unlock")
                door:Fire("Open")
                client:Notify("You unlocked the door.")
                door:EmitSound("doors/door_latch3.wav")
            else
                door:Fire("Close")
                door:Fire("Lock")
                client:Notify("You locked the door.")
                door:EmitSound("doors/door_latch1.wav")
            end
        end

        return false  -- Keep key after use
    end,

    OnCanRun = function(item)
        -- Always show the option
        return true
    end
}

-- Optional: Show which doors this key opens
function ITEM:GetDescription()
    local desc = self.description
    desc = desc .. "\n\nOpens doors: "

    for i, doorID in ipairs(self.doorIDs) do
        desc = desc .. "#" .. doorID
        if i < #self.doorIDs then
            desc = desc .. ", "
        end
    end

    return desc
end
```

✅ **Checkpoint**: You've created an interactive key item!

## Part 4: Dynamic Item with Data Storage

Create a water bottle that can be refilled.

**File**: `schema/items/sh_waterbottle.lua`

```lua
ITEM.name = "Water Bottle"
ITEM.model = "models/props_junk/garbage_glassbottle003a.mdl"
ITEM.width = 1
ITEM.height = 2
ITEM.category = "Consumables"
ITEM.price = 5

-- Maximum water amount
ITEM.maxWater = 100

-- Dynamic description based on water level
function ITEM:GetDescription()
    local water = self:GetData("water", self.maxWater)
    local percentage = math.Round((water / self.maxWater) * 100)

    return string.format(
        "A water bottle. Contains %d%% water (%d/%d ml).",
        percentage, water, self.maxWater
    )
end

-- Dynamic name
function ITEM:GetName()
    local water = self:GetData("water", self.maxWater)

    if water == 0 then
        return self.name .. " (Empty)"
    elseif water < self.maxWater * 0.3 then
        return self.name .. " (Low)"
    elseif water < self.maxWater then
        return self.name .. " (Partial)"
    else
        return self.name .. " (Full)"
    end
end

-- Initialize with full water
function ITEM:OnInstanced(invID, x, y)
    self:SetData("water", self.maxWater)
end

-- Drink function
ITEM.functions.Drink = {
    name = "Drink",
    icon = "icon16/cup.png",

    OnRun = function(item)
        local client = item.player

        if SERVER then
            local water = item:GetData("water", 0)

            -- Check if empty
            if water <= 0 then
                client:Notify("The bottle is empty!")
                return false
            end

            -- Drink amount (10ml per use)
            local drinkAmount = math.min(10, water)

            -- Restore stamina or health
            local character = client:GetCharacter()
            character:SetData("thirst", math.min(
                character:GetData("thirst", 0) + drinkAmount,
                100
            ))

            -- Reduce water in bottle
            item:SetData("water", water - drinkAmount)

            -- Sound and message
            client:EmitSound("player/drink.wav")
            client:Notify("You drank some water.")

            -- Remove bottle if empty
            if item:GetData("water") <= 0 then
                client:Notify("The bottle is now empty.")
            end
        end

        return false  -- Keep bottle
    end,

    OnCanRun = function(item)
        return item:GetData("water", 0) > 0
    end
}

-- Refill function
ITEM.functions.Refill = {
    name = "Refill",
    icon = "icon16/arrow_refresh.png",

    OnRun = function(item)
        local client = item.player

        if SERVER then
            -- Check if near water source
            local trace = client:GetEyeTrace()

            -- Check if looking at water (basic check)
            if trace.MatType ~= MAT_SLOSH then
                client:Notify("You need to be near a water source!")
                return false
            end

            -- Refill bottle
            item:SetData("water", item.maxWater)
            client:EmitSound("ambient/water/water_splash1.wav")
            client:Notify("You refilled the water bottle.")
        end

        return false  -- Keep bottle
    end,

    OnCanRun = function(item)
        return item:GetData("water", item.maxWater) < item.maxWater
    end
}
```

**Optional: Visual indicator on icon**

```lua
-- Show water level on icon
if CLIENT then
    function ITEM:PaintOver(item, w, h)
        local water = item:GetData("water", item.maxWater)
        local percentage = math.Round((water / item.maxWater) * 100)

        -- Draw percentage
        draw.SimpleText(
            percentage .. "%",
            "DermaDefault",
            w - 5,
            h - 5,
            color_white,
            TEXT_ALIGN_RIGHT,
            TEXT_ALIGN_BOTTOM,
            1,
            color_black
        )

        -- Draw water level bar
        local barHeight = 3
        local barWidth = math.Round((water / item.maxWater) * w)

        surface.SetDrawColor(50, 150, 255, 200)
        surface.DrawRect(0, h - barHeight, barWidth, barHeight)
    end
end
```

✅ **Checkpoint**: You've created an item with dynamic data!

## Part 5: Using Base Items

Instead of creating from scratch, inherit from base items.

### Example: Custom Weapon

**File**: `schema/items/sh_customgun.lua`

```lua
ITEM.name = "Custom Pistol"
ITEM.description = "A modified 9mm pistol"
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.base = "base_weapons"  -- Inherit weapon functionality
ITEM.weapon = "weapon_pistol"  -- Weapon class
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.flag = "v"
ITEM.price = 300

-- Override to add custom behavior
function ITEM:OnEquipWeapon(client, weapon)
    -- Add custom modifications
    weapon:SetClip1(weapon:GetMaxClip1())
    client:Notify("You equipped your custom pistol.")
end
```

### Example: Custom Outfit

**File**: `schema/items/sh_policeu uniform.lua`

```lua
ITEM.name = "Police Uniform"
ITEM.description = "Standard issue police uniform"
ITEM.model = "models/props_c17/suitcase_passenger_physics.mdl"
ITEM.base = "base_outfit"  -- Inherit outfit functionality
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Clothing"
ITEM.outfitCategory = "uniform"

-- Model replacement
ITEM.replacements = {
    "models/police.mdl"
}

-- Allowed bodygroups
ITEM.bodyGroups = {
    ["head"] = 0,
    ["body"] = 1
}
```

✅ **Checkpoint**: You're using base items efficiently!

## Common Patterns

### Pattern 1: Item with Durability

```lua
function ITEM:OnInstanced()
    self:SetData("durability", 100)
end

function ITEM:GetDescription()
    local durability = self:GetData("durability", 100)
    return self.description .. string.format("\nDurability: %d%%", durability)
end

-- Damage the item when used
ITEM.functions.Use = {
    OnRun = function(item)
        local durability = item:GetData("durability", 100)
        durability = durability - 10

        if durability <= 0 then
            item.player:Notify("The item broke!")
            return true  -- Remove item
        end

        item:SetData("durability", durability)
        return false
    end
}
```

### Pattern 2: Item with Cooldown

```lua
ITEM.functions.Use = {
    OnRun = function(item)
        local client = item.player
        local lastUse = item:GetData("lastUse", 0)
        local currentTime = CurTime()

        -- 60 second cooldown
        if currentTime - lastUse < 60 then
            local remaining = math.ceil(60 - (currentTime - lastUse))
            client:Notify("Wait " .. remaining .. " seconds!")
            return false
        end

        -- Do action
        -- ...

        item:SetData("lastUse", currentTime)
        return false
    end
}
```

### Pattern 3: Item with Charges

```lua
function ITEM:OnInstanced()
    self:SetData("charges", 3)
end

function ITEM:GetName()
    local charges = self:GetData("charges", 0)
    return self.name .. " (" .. charges .. " charges)"
end

ITEM.functions.Use = {
    OnRun = function(item)
        local charges = item:GetData("charges", 0)

        if charges <= 0 then
            item.player:Notify("No charges remaining!")
            return true  -- Remove empty item
        end

        -- Use item
        -- ...

        charges = charges - 1
        item:SetData("charges", charges)

        if charges <= 0 then
            return true  -- Remove when empty
        end

        return false
    end
}
```

## Best Practices

### ✅ DO

- Store per-instance data with `:SetData()` and `:GetData()`
- Use `if SERVER` for actions that modify game state
- Provide clear feedback with `client:Notify()`
- Check conditions in `OnCanRun` before allowing use
- Return `true` from `OnRun` to remove item
- Return `false` from `OnRun` to keep item
- Use base items when possible
- Add visual indicators for item state

### ❌ DON'T

- Don't store data in `self.variable` (won't save)
- Don't forget SERVER/CLIENT checks
- Don't modify player state on client
- Don't create items without width/height
- Don't forget to validate data
- Don't use invalid models (check model exists)
- Don't create overpowered items

## Testing Your Items

```lua
// Give item to yourself
ix_giveitem "YourName" sh_itemname

// Give specific quantity
ix_giveitem "YourName" sh_itemname 5

// Check item registered correctly
lua_run PrintTable(ix.item.list.sh_itemname)

// Debug item data
lua_run_cl PrintTable(LocalPlayer():GetCharacter():GetInventory():GetItems()[1]:GetData())
```

## Troubleshooting

### Item doesn't appear

**Check**:
1. File is in `items/` folder
2. File named `sh_itemname.lua`
3. Ran `sh_rebuildschema`
4. No syntax errors in console

### Item has no icon

**Cause**: Invalid model path

**Fix**:
```lua
-- Check model exists
ITEM.model = "models/valid/path.mdl"

-- Or use custom icon
ITEM.icon = "vgui/inventory/icon.png"
```

### Functions don't work

**Cause**: Wrong function structure

**Fix**:
```lua
// ✅ CORRECT
ITEM.functions.Use = {
    OnRun = function(item)
        -- code
    end
}

// ❌ WRONG
ITEM.functions.Use = function(item)
    -- Won't work
end
```

## See Also

- [Item System Documentation](../systems/items.md)
- [Inventory System](../systems/inventory.md)
- [Inventory Programming Guide](inventories.md)
- [First Plugin Tutorial](first-plugin.md)
- Base Items: `gamemode/items/base/`
