# PAC Outfit Items

> **Reference**: `gamemode/items/base/sh_pacoutfit.lua`

The PAC outfit base item provides equippable accessories and attachments using PAC3 (Player Appearance Customizer).

## ⚠️ Important: Use Built-in PAC Outfit Base

**Always extend from `base_pacoutfit`** rather than creating custom PAC systems. The framework provides:
- Automatic PAC3 part management (line 68-81)
- Equipment state tracking (line 71, 128)
- Outfit category slot management (line 8, 120)
- Attribute boosts integration (line 74-78, 131-135)
- Drop/death cleanup (line 84-88)
- Model replacement support (line 37-53)
- Bodygroup support (line 49-53)

## Core Concepts

### What is a PAC Outfit Item?

PAC outfit items are equippable accessories that attach PAC3 models to the player. Unlike regular outfits, they:
1. Add 3D models as attachments (not replace the model)
2. Attach to specific bones
3. Can be layered with other PAC outfits
4. Require PAC3 addon to be installed
5. Use outfit categories for slot management

### Key Properties

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:2-9`

```lua
ITEM.name = "PAC Outfit"
ITEM.description = "A PAC Outfit Base."
ITEM.category = "Outfit"
ITEM.model = "models/Gibs/HGIBS.mdl"  -- Icon model
ITEM.width = 1
ITEM.height = 1
ITEM.outfitCategory = "hat"           -- Equipment slot
ITEM.pacData = {}                     -- PAC3 structure (required!)
```

### PAC3 Data Structure

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:11-55`

The `pacData` table contains PAC3 part definitions:

```lua
ITEM.pacData = {
    [1] = {
        ["children"] = {
            [1] = {
                ["self"] = {
                    ["Angles"] = Angle(12.919, 0, 0),
                    ["Position"] = Vector(-2.1, 0.02, 1.01),
                    ["UniqueID"] = "4249811628",
                    ["Size"] = 1.25,
                    ["Bone"] = "eyes",              -- Attach to this bone
                    ["Model"] = "models/props/hat.mdl",  -- Model to attach
                    ["ClassName"] = "model",
                },
            },
        },
        ["self"] = {
            ["ClassName"] = "group",
            ["UniqueID"] = "907159817",
            ["EditorExpand"] = true,
        },
    },
}
```

### Outfit Categories

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:8`

The `outfitCategory` field determines equipment slots:

```lua
ITEM.outfitCategory = "hat"      -- Head accessories
ITEM.outfitCategory = "glasses"  -- Eyewear
ITEM.outfitCategory = "mask"     -- Face masks
ITEM.outfitCategory = "back"     -- Backpacks, capes
ITEM.outfitCategory = "hands"    -- Gloves, watches
```

Only **one PAC outfit per category** can be equipped at a time (line 120).

## Using PAC Outfit Items

### Creating a Simple PAC Outfit

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:2-9`

Create a PAC outfit with attachment:

```lua
-- schema/items/sh_glasses.lua
ITEM.name = "Glasses"
ITEM.description = "A pair of stylish glasses."
ITEM.model = "models/props_c17/BriefCase001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Accessories"
ITEM.outfitCategory = "glasses"
ITEM.base = "base_pacoutfit"

-- PAC3 data (export from PAC3 editor)
ITEM.pacData = {
    [1] = {
        ["children"] = {
            [1] = {
                ["self"] = {
                    ["ClassName"] = "model",
                    ["UniqueID"] = "glasses_model",
                    ["Model"] = "models/props/glasses.mdl",
                    ["Bone"] = "ValveBiped.Bip01_Head1",
                    ["Position"] = Vector(3.5, 0, 0),
                    ["Angles"] = Angle(0, 90, 0),
                    ["Size"] = 1.0,
                },
            },
        },
        ["self"] = {
            ["ClassName"] = "group",
            ["UniqueID"] = "glasses_group",
            ["EditorExpand"] = true,
        },
    },
}
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't add PAC parts manually
pac.AddPart(player, pacData)  -- Bypasses item system!
-- Won't track equipment, won't persist
```

### Exporting PAC3 Data

To get `pacData` from PAC3:

1. Open PAC3 editor in-game
2. Create your outfit/accessory
3. Right-click the outfit group
4. Select "Copy to Clipboard (Lua)"
5. Paste into your item's `pacData` field

### PAC Outfit with Model Replacement

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:37-53`

PAC outfits can also change the player model:

```lua
ITEM.name = "Armored Suit"
ITEM.outfitCategory = "model"
ITEM.base = "base_pacoutfit"

-- Change model
ITEM.replacement = "models/player/soldier.mdl"

-- Or use pattern replacement
ITEM.replacements = {"male", "female"}

-- Or multiple replacements
ITEM.replacements = {
    {"male", "female"},
    {"group01", "group02"}
}

-- With bodygroups
ITEM.bodyGroups = {
    ["helmet"] = 1,
    ["armor"] = 2
}

-- And skin
ITEM.newSkin = 1
```

## Equip and Unequip

### Equipping PAC Outfits

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:109-146`

The equip function checks category conflicts:

```lua
ITEM.functions.Equip = {
    name = "equip",
    OnRun = function(item)
        local char = item.player:GetCharacter()

        -- Check for slot conflict
        for k, _ in char:GetInventory():Iter() do
            if k.id != item.id then
                local itemTable = ix.item.instances[k.id]

                if itemTable.pacData and k.outfitCategory == item.outfitCategory
                   and itemTable:GetData("equip") then
                    item.player:NotifyLocalized(item.equippedNotify or "outfitAlreadyEquipped")
                    return false
                end
            end
        end

        -- Equip the outfit
        item:SetData("equip", true)
        item.player:AddPart(item.uniqueID, item)

        -- Add attribute boosts
        if item.attribBoosts then
            for k, v in pairs(item.attribBoosts) do
                char:AddBoost(item.uniqueID, k, v)
            end
        end

        item:OnEquipped()
        return false
    end
}
```

### ITEM:RemovePart(client)

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:68-81`

Removes the PAC outfit from player:

```lua
function ITEM:RemovePart(client)
    local char = client:GetCharacter()

    self:SetData("equip", false)
    client:RemovePart(self.uniqueID)

    -- Remove attribute boosts
    if self.attribBoosts then
        for k, _ in pairs(self.attribBoosts) do
            char:RemoveBoost(self.uniqueID, k)
        end
    end

    self:OnUnequipped()
end
```

## Attribute Boosts

### Adding Stat Bonuses

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:131-135`

PAC outfits can provide attribute boosts:

```lua
ITEM.name = "Combat Helmet"
ITEM.outfitCategory = "hat"
ITEM.base = "base_pacoutfit"

ITEM.pacData = {
    -- PAC3 helmet data
}

-- Add attribute bonuses
ITEM.attribBoosts = {
    ["stm"] = 5,    -- +5 stamina
    ["armor"] = 10  -- +10 armor
}
```

Boosts are automatically added on equip and removed on unequip.

## Custom Hooks

### OnEquipped()

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:167-168`

Called after PAC outfit is equipped:

```lua
function ITEM:OnEquipped()
    -- Custom logic after equipping
    self.player:ChatPrint("You put on the accessory.")
end
```

### OnUnequipped()

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:170-171`

Called after PAC outfit is removed:

```lua
function ITEM:OnUnequipped()
    -- Custom logic after unequipping
    self.player:ChatPrint("You removed the accessory.")
end
```

## Outfit Lifecycle

### Drop Behavior

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:84-88`

PAC outfits are automatically removed when dropped:

```lua
ITEM:Hook("drop", function(item)
    if item:GetData("equip") then
        item:RemovePart(item:GetOwner())
    end
end)
```

### Removal Cleanup

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:156-165`

When PAC outfit is deleted while equipped:

```lua
function ITEM:OnRemoved()
    local inventory = ix.item.inventories[self.invID]
    local owner = inventory.GetOwner and inventory:GetOwner()

    if IsValid(owner) and owner:IsPlayer() then
        if self:GetData("equip") then
            self:RemovePart(owner)
        end
    end
end
```

### Transfer Prevention

**Reference**: `gamemode/items/base/sh_pacoutfit.lua:148-154`

Equipped PAC outfits cannot be transferred:

```lua
function ITEM:CanTransfer(oldInventory, newInventory)
    if newInventory and self:GetData("equip") then
        return false
    end

    return true
end
```

## Best Practices

### ✅ DO

- Use `ITEM.base = "base_pacoutfit"` for all PAC outfits
- Set `outfitCategory` to control equipment slots
- Export `pacData` from PAC3 editor (don't write by hand)
- Test PAC outfits with different player models
- Use unique `UniqueID` values in PAC data
- Set appropriate bone attachments
- Use `attribBoosts` for gameplay effects
- Use `OnEquipped`/`OnUnequipped` for custom behavior

### ❌ DON'T

- Don't manually add PAC parts without item system
- Don't write `pacData` by hand (export from PAC3)
- Don't forget to set `outfitCategory`
- Don't use same `UniqueID` for different outfits
- Don't forget that PAC3 addon is required
- Don't allow multiple outfits in same category
- Don't forget to test with different models/bones

## Complete Example

```lua
-- schema/items/sh_combat_helmet.lua
ITEM.name = "Combat Helmet"
ITEM.description = "A tactical helmet with night vision attachment."
ITEM.model = "models/props_c17/BriefCase001a.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Tactical Gear"
ITEM.outfitCategory = "hat"
ITEM.base = "base_pacoutfit"

-- PAC3 data (exported from PAC3 editor)
ITEM.pacData = {
    [1] = {
        ["children"] = {
            [1] = {
                ["self"] = {
                    ["ClassName"] = "model",
                    ["UniqueID"] = "combat_helmet_main",
                    ["Model"] = "models/props/helmet.mdl",
                    ["Bone"] = "ValveBiped.Bip01_Head1",
                    ["Position"] = Vector(0, 0, 0),
                    ["Angles"] = Angle(0, 0, 0),
                    ["Size"] = 1.0,
                },
            },
            [2] = {
                ["self"] = {
                    ["ClassName"] = "model",
                    ["UniqueID"] = "combat_helmet_nvg",
                    ["Model"] = "models/props/nightvision.mdl",
                    ["Bone"] = "ValveBiped.Bip01_Head1",
                    ["Position"] = Vector(3, 0, 2),
                    ["Angles"] = Angle(0, 0, 0),
                    ["Size"] = 0.8,
                },
            },
        },
        ["self"] = {
            ["ClassName"] = "group",
            ["UniqueID"] = "combat_helmet_group",
            ["EditorExpand"] = true,
        },
    },
}

-- Attribute bonuses
ITEM.attribBoosts = {
    ["perception"] = 5,  -- Night vision bonus
    ["armor"] = 10       -- Protection
}

-- Custom equip behavior
function ITEM:OnEquipped()
    self.player:ChatPrint("You put on the combat helmet.")

    -- Give night vision ability
    local char = self.player:GetCharacter()
    char:SetData("hasNightVision", true)
end

function ITEM:OnUnequipped()
    local char = self.player:GetCharacter()
    char:SetData("hasNightVision", false)

    -- Disable night vision if active
    if char:GetData("nightVisionActive") then
        char:SetData("nightVisionActive", false)
        -- Additional cleanup...
    end
end
```

## PAC3 Tips

### Finding Bone Names

To find bone names for attachments:

```lua
-- In console or Lua run:
local ply = Entity(1)  -- Your player
for i = 0, ply:GetBoneCount() - 1 do
    print(i, ply:GetBoneName(i))
end

-- Common bones:
-- ValveBiped.Bip01_Head1 - Head
-- ValveBiped.Bip01_Spine4 - Upper torso
-- ValveBiped.Bip01_R_Hand - Right hand
-- ValveBiped.Bip01_L_Hand - Left hand
```

### Testing PAC Outfits

1. Use PAC3 editor to create and test
2. Ensure model appears correctly on different player models
3. Test bone attachments work on all models
4. Verify positioning and scaling
5. Export to Lua when satisfied
6. Paste into item definition

### Combining with Model Replacement

```lua
-- Change model AND add PAC parts
ITEM.replacement = "models/player/soldier.mdl"
ITEM.bodyGroups = {
    ["helmet"] = 0  -- Hide default helmet
}
ITEM.pacData = {
    -- Custom helmet PAC attachment
}
```

## Common Issues

### Issue: PAC Parts Not Appearing

**Cause**: PAC3 addon not installed or `pacData` incorrect
**Fix**: Ensure PAC3 is installed and data is exported correctly

```lua
-- Check if PAC is available
if not ix.pac then
    print("PAC3 addon is not installed!")
end
```

### Issue: Parts Attached to Wrong Bone

**Cause**: Bone name doesn't exist on player model
**Fix**: Verify bone exists on target model

```lua
-- Check if bone exists
local boneID = player:LookupBone("ValveBiped.Bip01_Head1")
if not boneID then
    print("Bone does not exist on this model!")
end
```

### Issue: Multiple PAC Outfits in Same Slot

**Cause**: Using different `outfitCategory` values
**Fix**: Use same category for conflicting outfits

```lua
-- Both hats should use same category
ITEM.outfitCategory = "hat"  -- Not "hat" and "head"
```

### Issue: PAC Parts Persist After Unequip

**Cause**: Not calling `RemovePart()` properly
**Fix**: Use the item's unequip function

```lua
-- WRONG
client:RemovePart(someID)

-- CORRECT
item:RemovePart(client)
-- Or right-click and select "Unequip"
```

## PAC3 Resources

- PAC3 Workshop: https://steamcommunity.com/sharedfiles/filedetails/?id=104691717
- PAC3 Documentation: In-game PAC3 editor help
- PAC3 Export: Right-click group → "Copy to Clipboard (Lua)"

## See Also

- [Base Items](base-items.md) - Overview of all base item types
- [Outfits](outfits.md) - Regular outfit items (model replacement)
- [Item System](../systems/items.md) - Core item system
- [Attributes](../systems/attributes.md) - Attribute system for boosts
- Source: `gamemode/items/base/sh_pacoutfit.lua`
