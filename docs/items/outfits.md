# Outfit Items

> **Reference**: `gamemode/items/base/sh_outfit.lua`

The outfit base item provides equippable clothing that changes the player's model, bodygroups, skin, and submaterials.

## ⚠️ Important: Use Built-in Outfit Item Base

**Always extend from `base_outfit`** rather than creating custom outfit systems. The framework provides:
- Automatic model changing and restoration (line 41-75)
- Bodygroup saving and restoration (line 46-99, 152-168)
- Submaterial persistence (line 101-111, 136-147)
- Attribute boosts integration (line 113-117, 199-203)
- Outfit category slot management (line 8)
- Attachment system for layered outfits (line 213-232)
- Drop/death cleanup (line 234-242)

## Core Concepts

### What is an Outfit Item?

Outfit items are equippable clothing that modify a player's appearance. When equipped, they:
1. Change the player's model (optional)
2. Apply bodygroups (optional)
3. Change skin (optional)
4. Apply submaterials (optional)
5. Add attribute boosts (optional)
6. Save previous appearance for restoration

### Key Properties

**Reference**: `gamemode/items/base/sh_outfit.lua:2-9`

```lua
ITEM.name = "Outfit"
ITEM.description = "A Outfit Base."
ITEM.category = "Outfit"
ITEM.model = "models/Gibs/HGIBS.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.outfitCategory = "model"  -- Equipment slot
ITEM.pacData = {}              -- PAC3 attachments (if needed)
```

### Outfit Categories

**Reference**: `gamemode/items/base/sh_outfit.lua:8`

The `outfitCategory` field determines equipment slots:

```lua
ITEM.outfitCategory = "model"   -- Full body outfit
ITEM.outfitCategory = "hat"     -- Head slot
ITEM.outfitCategory = "torso"   -- Torso slot
ITEM.outfitCategory = "legs"    -- Legs slot
ITEM.outfitCategory = "hands"   -- Gloves slot
```

Only **one outfit per category** can be equipped at a time.

## Model Modification

### Full Model Replacement

**Reference**: `gamemode/items/base/sh_outfit.lua:11-28`

Replace the player's entire model:

```lua
-- Simple replacement
ITEM.replacement = "models/player/police.mdl"

-- Or use replacements (same thing)
ITEM.replacements = "models/player/police.mdl"
```

### Model Part Replacement

**Reference**: `gamemode/items/base/sh_outfit.lua:11-28`

Replace parts of the model path:

```lua
-- Change a specific part (e.g., male to female)
ITEM.replacements = {"male", "female"}

-- Multiple replacements
ITEM.replacements = {
    {"male", "female"},
    {"group01", "group02"}
}
```

This performs string replacement on the current model path.

### Dynamic Model Replacement

**Reference**: `gamemode/items/base/sh_outfit.lua:56-59`

Use a function for complex logic:

```lua
function ITEM:OnGetReplacement()
    local client = self.player

    -- Return different models based on conditions
    if client:GetCharacter():GetGender() == GENDER_MALE then
        return "models/player/male_outfit.mdl"
    else
        return "models/player/female_outfit.mdl"
    end
end
```

## Bodygroups

### Setting Bodygroups

**Reference**: `gamemode/items/base/sh_outfit.lua:24-28`

Apply bodygroups when equipped:

```lua
ITEM.bodyGroups = {
    ["blade"] = 1,      -- Bodygroup name = value
    ["bladeblur"] = 1
}
```

Or use indices:

```lua
ITEM.bodyGroups = {
    [0] = 1,  -- Bodygroup 0 = value 1
    [1] = 2   -- Bodygroup 1 = value 2
}
```

### Bodygroup Restoration

**Reference**: `gamemode/items/base/sh_outfit.lua:86-99`

The framework saves and restores bodygroups:

```lua
-- Save bodygroups to item
local groups = self:GetData("groups", {})

-- Apply saved bodygroups
if !table.IsEmpty(groups) and self:ShouldRestoreBodygroups() then
    for k, v in pairs(groups) do
        client:SetBodygroup(k, v)
    end
end
```

Control restoration with:

```lua
function ITEM:ShouldRestoreBodygroups()
    return true  -- Restore saved bodygroups (default)
    -- return false  -- Use default bodygroups always
end
```

## Skin Changes

### Setting Skin

**Reference**: `gamemode/items/base/sh_outfit.lua:12-13, 77-80`

Change the player's model skin:

```lua
ITEM.newSkin = 1  -- Skin index (starts at 0)
```

The original skin is saved and restored on unequip.

## Submaterials

### Applying Submaterials

**Reference**: `gamemode/items/base/sh_outfit.lua:101-111`

Submaterials modify model textures:

```lua
-- Applied automatically from saved data
local materials = self:GetData("submaterial", {})

if !table.IsEmpty(materials) and self:ShouldRestoreSubMaterials() then
    for k, v in pairs(materials) do
        if !isnumber(k) or !isstring(v) then continue end
        client:SetSubMaterial(k - 1, v)
    end
end
```

Control restoration with:

```lua
function ITEM:ShouldRestoreSubMaterials()
    return true  -- Restore saved submaterials (default)
end
```

## Using Outfit Items

### Creating a Custom Outfit

Create a simple outfit that replaces the model:

```lua
-- schema/items/sh_police_uniform.lua
ITEM.name = "Police Uniform"
ITEM.description = "A standard police officer uniform."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Clothing"
ITEM.outfitCategory = "model"
ITEM.replacement = "models/player/police.mdl"
ITEM.base = "base_outfit"
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't change models manually
function EquipOutfit(ply, model)
    ply:SetModel(model)  -- Won't save original model!
    -- Won't persist, won't restore on death
end
```

### Outfit with Bodygroups

```lua
-- schema/items/sh_tactical_vest.lua
ITEM.name = "Tactical Vest"
ITEM.description = "A protective vest with equipment pouches."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Clothing"
ITEM.outfitCategory = "torso"
ITEM.bodyGroups = {
    ["vest"] = 1,
    ["pouches"] = 2
}
ITEM.base = "base_outfit"
```

### Outfit with Attributes

**Reference**: `gamemode/items/base/sh_outfit.lua:113-117`

Add attribute boosts while worn:

```lua
ITEM.name = "Military Armor"
ITEM.outfitCategory = "torso"
ITEM.replacement = "models/player/soldier.mdl"
ITEM.base = "base_outfit"

-- Add attribute bonuses
ITEM.attribBoosts = {
    ["stm"] = 10,   -- +10 stamina
    ["str"] = 5     -- +5 strength
}
```

Boosts are automatically added on equip (line 113-117) and removed on unequip (line 199-203).

## Equip and Unequip

### ITEM:AddOutfit(client)

**Reference**: `gamemode/items/base/sh_outfit.lua:41-121`

Equips the outfit to a player:

```lua
item:AddOutfit(client)
```

This function:
1. Sets equipment state (line 44)
2. Saves original bodygroups (line 46-54)
3. Applies model replacement (line 56-75)
4. Changes skin if defined (line 77-80)
5. Restores or applies bodygroups (line 82-99)
6. Applies submaterials (line 101-111)
7. Adds attribute boosts (line 113-117)
8. Updates hands (line 119)
9. Calls `OnEquipped()` hook (line 120)

### ITEM:RemoveOutfit(client)

**Reference**: `gamemode/items/base/sh_outfit.lua:131-211`

Removes the outfit from a player:

```lua
item:RemoveOutfit(client)
```

This function:
1. Sets equipment state to false (line 134)
2. Saves current submaterials to item (line 136-147)
3. Resets submaterials (line 150)
4. Saves current bodygroups to item (line 152-165)
5. Resets bodygroups (line 168)
6. Restores original model (line 171-174)
7. Restores original skin (line 177-184)
8. Restores original bodygroups (line 187-197)
9. Removes attribute boosts (line 199-203)
10. Removes attachments (line 205-207)
11. Updates hands (line 209)
12. Calls `OnUnequipped()` hook (line 210)

## Outfit Attachments

### Adding Attachments

**Reference**: `gamemode/items/base/sh_outfit.lua:213-232`

Outfits can have dependent attachments (items that require the outfit to be worn):

```lua
-- Make another outfit depend on this outfit
function ITEM:AddAttachment(id)
    local attachments = self:GetData("outfitAttachments", {})
    attachments[id] = true
    self:SetData("outfitAttachments", attachments)
end

-- Remove an attachment
function ITEM:RemoveAttachment(id, client)
    local item = ix.item.instances[id]
    local attachments = self:GetData("outfitAttachments", {})

    if item and attachments[id] then
        item:OnDetached(client)
    end

    attachments[id] = nil
    self:SetData("outfitAttachments", attachments)
end
```

This allows creating layered clothing systems.

## Custom Hooks

### OnEquipped()

**Reference**: `gamemode/items/base/sh_outfit.lua:306-307`

Called after outfit is equipped:

```lua
function ITEM:OnEquipped()
    -- Custom logic after equipping
    self.player:ChatPrint("You put on the outfit.")
end
```

### OnUnequipped()

**Reference**: `gamemode/items/base/sh_outfit.lua:309-310`

Called after outfit is removed:

```lua
function ITEM:OnUnequipped()
    -- Custom logic after unequipping
    self.player:ChatPrint("You removed the outfit.")
end
```

### CanEquipOutfit()

**Reference**: `gamemode/items/base/sh_outfit.lua:312-314`

Control whether the outfit can be equipped:

```lua
function ITEM:CanEquipOutfit()
    local char = self.player:GetCharacter()

    -- Require certain faction
    if char:GetFaction() != FACTION_POLICE then
        self.player:Notify("Only police can wear this!")
        return false
    end

    return true
end
```

## Outfit Lifecycle

### Drop Behavior

**Reference**: `gamemode/items/base/sh_outfit.lua:234-242`

Outfits are automatically removed when dropped:

```lua
ITEM:Hook("drop", function(item)
    if item:GetData("equip") then
        local character = ix.char.loaded[item.owner]
        local client = character and character:GetPlayer() or item:GetOwner()

        item.player = client
        item:RemoveOutfit(item:GetOwner())
    end
end)
```

### Removal Cleanup

**Reference**: `gamemode/items/base/sh_outfit.lua:298-304`

When outfit is deleted while equipped, it's automatically removed:

```lua
function ITEM:OnRemoved()
    if self.invID != 0 and self:GetData("equip") then
        self.player = self:GetOwner()
        self:RemoveOutfit(self.player)
        self.player = nil
    end
end
```

## Best Practices

### ✅ DO

- Use `ITEM.base = "base_outfit"` for all outfit items
- Set `outfitCategory` to control equipment slots
- Use `replacement` for full model changes
- Use `bodyGroups` for partial appearance changes
- Use `attribBoosts` for outfit stat bonuses
- Let framework handle model/bodygroup saving
- Test outfits with different base models
- Use `OnEquipped`/`OnUnequipped` for custom behavior

### ❌ DON'T

- Don't call `player:SetModel()` directly for outfits
- Don't manually track original model/bodygroups
- Don't forget to set `outfitCategory`
- Don't allow conflicting outfit categories
- Don't modify bodygroups without using the outfit system
- Don't implement custom appearance persistence
- Don't forget that drop auto-unequips outfits

## Complete Example

```lua
-- schema/items/sh_medic_uniform.lua
ITEM.name = "Medic Uniform"
ITEM.description = "A medical professional's outfit."
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 3
ITEM.category = "Medical"
ITEM.outfitCategory = "model"
ITEM.base = "base_outfit"

-- Replace model based on gender
function ITEM:OnGetReplacement()
    local client = self.player
    local char = client:GetCharacter()

    if char:GetData("gender") == "female" then
        return "models/player/female_medic.mdl"
    else
        return "models/player/male_medic.mdl"
    end
end

-- Attribute bonuses
ITEM.attribBoosts = {
    ["medical"] = 10  -- +10 medical skill
}

-- Bodygroups for accessories
ITEM.bodyGroups = {
    ["hat"] = 1,
    ["vest"] = 1
}

-- Skin variation
ITEM.newSkin = 0

-- Custom equip behavior
function ITEM:OnEquipped()
    self.player:ChatPrint("You are now dressed as a medic.")

    -- Give medic ability
    local char = self.player:GetCharacter()
    char:SetData("canHeal", true)
end

function ITEM:OnUnequipped()
    local char = self.player:GetCharacter()
    char:SetData("canHeal", false)
end

-- Restrict to medic faction
function ITEM:CanEquipOutfit()
    local char = self.player:GetCharacter()

    if char:GetFaction() != FACTION_MEDIC then
        self.player:Notify("Only medics can wear this uniform!")
        return false
    end

    return true
end
```

## Common Issues

### Issue: Original Model Not Restored

**Cause**: Calling `player:SetModel()` instead of `RemoveOutfit()`
**Fix**: Always use the outfit system

```lua
-- WRONG
player:SetModel(originalModel)

-- CORRECT
item:RemoveOutfit(player)
```

### Issue: Bodygroups Not Applying

**Cause**: Bodygroup names don't exist on the model
**Fix**: Verify bodygroup names for the target model

```lua
-- Check model's bodygroups
for i = 0, player:GetNumBodyGroups() - 1 do
    print(i, player:GetBodygroupName(i), player:GetBodygroupCount(i))
end
```

### Issue: Multiple Outfits Equipped

**Cause**: Using different `outfitCategory` values
**Fix**: Use same category for conflicting outfits

```lua
-- Both should use same category
ITEM.outfitCategory = "model"  -- Not "torso" and "model"
```

### Issue: Outfit Persists After Death

**Cause**: Not implementing death cleanup
**Fix**: Framework handles this automatically for character deaths

If you need custom behavior, use the `PlayerDeath` hook in your schema.

## See Also

- [Base Items](base-items.md) - Overview of all base item types
- [PAC Outfits](pac-outfits.md) - PAC3-based outfit items
- [Item System](../systems/items.md) - Core item system
- [Attributes](../systems/attributes.md) - Attribute system for boosts
- Source: `gamemode/items/base/sh_outfit.lua`
