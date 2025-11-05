# Weapon Items

> **Reference**: `gamemode/items/base/sh_weapons.lua`

The weapon base item provides equippable weapons with automatic ammo persistence, equipment state tracking, and PAC3 support.

## ⚠️ Important: Use Built-in Weapon Item Base

**Always extend from `base_weapons`** rather than creating custom weapon systems. The framework provides:
- Automatic ammo saving and restoration (line 65, 179, 204)
- Equipment state persistence (line 173)
- Weapon category slot management (line 134-139)
- PAC3 attachment support (line 110-120)
- Network synchronization
- Death/disconnect cleanup (line 287-300)

## Core Concepts

### What is a Weapon Item?

Weapon items represent equippable weapons in the inventory system. When equipped, they:
1. Give the weapon entity to the player
2. Save ammo count to the item
3. Track equipment state
4. Prevent transfer while equipped
5. Restore weapon on reconnect

### Key Properties

```lua
ITEM.name = "Weapon"           -- Display name
ITEM.description = "A Weapon." -- Description
ITEM.model = "models/weapons/w_pistol.mdl"  -- World model
ITEM.class = "weapon_pistol"   -- Weapon class to spawn
ITEM.width = 2                 -- Inventory width
ITEM.height = 2                -- Inventory height
ITEM.isWeapon = true          -- Identifies as weapon
ITEM.isGrenade = false        -- Special grenade handling
ITEM.weaponCategory = "sidearm" -- Equipment slot
ITEM.useSound = "items/ammo_pickup.wav"  -- Equip/unequip sound
```

### Weapon Categories

**Reference**: `gamemode/items/base/sh_weapons.lua:134-139`

The `weaponCategory` field determines equipment slots:

```lua
ITEM.weaponCategory = "sidearm"   -- Pistol slot
ITEM.weaponCategory = "primary"   -- Primary weapon slot
ITEM.weaponCategory = "secondary" -- Secondary weapon slot
ITEM.weaponCategory = "melee"     -- Melee weapon slot
```

Only **one weapon per category** can be equipped at a time. Attempting to equip another weapon in the same category will show an error.

## Using Weapon Items

### Creating a Custom Weapon

**Reference**: `gamemode/items/base/sh_weapons.lua:2-12`

Create a new weapon item by extending the base:

```lua
-- schema/items/sh_pistol.lua
ITEM.name = "9mm Pistol"
ITEM.description = "A standard issue pistol."
ITEM.model = "models/weapons/w_pistol.mdl"
ITEM.class = "weapon_pistol"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.weaponCategory = "sidearm"
ITEM.base = "base_weapons"
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't give weapons manually without item system
function GivePlayerWeapon(ply, weapon)
    ply:Give(weapon)  -- Bypasses item system!
    -- Won't persist, won't track ammo, won't sync
end
```

### Equip Function

**Reference**: `gamemode/items/base/sh_weapons.lua:94-108`

The built-in Equip function handles weapon equipping:

```lua
-- Players right-click the item and select "Equip"
-- Internally calls:
ITEM.functions.Equip = {
    name = "equip",
    OnRun = function(item)
        item:Equip(item.player)
        return false  -- Keep in inventory
    end,
    OnCanRun = function(item)
        local client = item.player
        return !IsValid(item.entity) and IsValid(client) and
               item:GetData("equip") != true
    end
}
```

### ITEM:Equip(client, bNoSelect, bNoSound)

**Reference**: `gamemode/items/base/sh_weapons.lua:122-190`

Equips the weapon to a player:

```lua
-- Standard equip
local weapon = item:Equip(client)

-- Equip without auto-selecting weapon
local weapon = item:Equip(client, true)

-- Equip silently (no sound)
local weapon = item:Equip(client, false, true)
```

The function automatically:
- Checks for slot conflicts (line 134-139)
- Gives the weapon entity (line 147)
- Removes default ammo (line 163-164)
- Loads saved ammo from item data (line 179)
- Sets equipment state (line 173)
- Handles grenade special case (line 175-177)
- Calls `OnEquipWeapon` hook if defined (line 184-186)

### ITEM:Unequip(client, bPlaySound, bRemoveItem)

**Reference**: `gamemode/items/base/sh_weapons.lua:192-225`

Unequips the weapon from a player:

```lua
-- Standard unequip
item:Unequip(client)

-- Unequip with sound
item:Unequip(client, true)

-- Unequip and delete item
item:Unequip(client, true, true)
```

The function automatically:
- Finds the weapon entity (line 195-198)
- Saves current ammo to item (line 204)
- Strips weapon from player (line 205)
- Clears equipment state (line 215)
- Removes PAC3 parts (line 216)
- Calls `OnUnequipWeapon` hook if defined (line 218-220)

## Ammo Persistence

### How Ammo is Saved

**Reference**: `gamemode/items/base/sh_weapons.lua:264-270`

Ammo is automatically saved when:

1. **On unequip** (line 204):
```lua
function ITEM:Unequip(client, bPlaySound, bRemoveItem)
    local weapon = client:GetWeapon(self.class)
    if IsValid(weapon) then
        self:SetData("ammo", weapon:Clip1())  -- Save ammo
    end
end
```

2. **On character save** (line 264-270):
```lua
function ITEM:OnSave()
    local weapon = self.player:GetWeapon(self.class)

    if IsValid(weapon) and weapon.ixItem == self and self:GetData("equip") then
        self:SetData("ammo", weapon:Clip1())
    end
end
```

3. **On drop** (line 65):
```lua
ITEM:Hook("drop", function(item)
    if item:GetData("equip") then
        local weapon = owner:GetWeapon(item.class)
        if IsValid(weapon) then
            item:SetData("ammo", weapon:Clip1())  -- Save before drop
        end
    end
end)
```

### How Ammo is Restored

**Reference**: `gamemode/items/base/sh_weapons.lua:241-262`

Ammo is restored on player loadout:

```lua
function ITEM:OnLoadout()
    if self:GetData("equip") then
        local client = self.player
        local weapon = client:Give(self.class, true)

        if IsValid(weapon) then
            -- Remove default ammo
            client:RemoveAmmo(weapon:Clip1(), weapon:GetPrimaryAmmoType())

            -- Restore saved ammo
            weapon:SetClip1(self:GetData("ammo", 0))

            -- Link weapon to item
            weapon.ixItem = self
        end
    end
end
```

## PAC3 Support

### Adding PAC3 Data

**Reference**: `gamemode/items/base/sh_weapons.lua:110-120`

Weapons can have PAC3 attachments:

```lua
ITEM.pacData = {
    [1] = {
        ["children"] = {
            -- PAC3 structure
        },
        ["self"] = {
            ["ClassName"] = "model",
            ["UniqueID"] = "12345",
            ["Model"] = "models/attachment.mdl",
            ["Bone"] = "ValveBiped.Bip01_R_Hand"
        }
    }
}

function ITEM:WearPAC(client)
    if ix.pac and self.pacData then
        client:AddPart(self.uniqueID, self)
    end
end

function ITEM:RemovePAC(client)
    if ix.pac and self.pacData then
        client:RemovePart(self.uniqueID)
    end
end
```

## Grenade Handling

**Reference**: `gamemode/items/base/sh_weapons.lua:10, 175-177, 302-316`

Grenades have special handling:

```lua
ITEM.isGrenade = true  -- Mark as grenade

-- In Equip function:
if self.isGrenade then
    weapon:SetClip1(1)
    client:SetAmmo(0, ammoType)
end
```

Grenades are auto-removed when thrown (line 302-316):

```lua
hook.Add("EntityRemoved", "ixRemoveGrenade", function(entity)
    if entity:GetClass() == "weapon_frag" then
        local client = entity:GetOwner()
        if IsValid(client) and client:GetAmmoCount(ammoName) < 1 then
            entity.ixItem:Unequip(client, false, true)  -- Remove item
        end
    end
end)
```

## Equipment State Management

### Death Cleanup

**Reference**: `gamemode/items/base/sh_weapons.lua:287-300`

On player death, all weapons are unequipped:

```lua
hook.Add("PlayerDeath", "ixStripClip", function(client)
    client.carryWeapons = {}

    for k, _ in client:GetCharacter():GetInventory():Iter() do
        if k.isWeapon and k:GetData("equip") then
            k:SetData("ammo", nil)    -- Clear ammo
            k:SetData("equip", nil)   -- Clear equipped state

            if k.pacData then
                k:RemovePAC(client)   -- Remove PAC parts
            end
        end
    end
end)
```

### Transfer Prevention

**Reference**: `gamemode/items/base/sh_weapons.lua:227-239`

Equipped weapons cannot be transferred:

```lua
function ITEM:CanTransfer(oldInventory, newInventory)
    if newInventory and self:GetData("equip") then
        local owner = self:GetOwner()

        if IsValid(owner) then
            owner:NotifyLocalized("equippedWeapon")
        end

        return false  -- Prevent transfer
    end

    return true
end
```

## Custom Hooks

### OnEquipWeapon(client, weapon)

**Reference**: `gamemode/items/base/sh_weapons.lua:184-186`

Called after weapon is equipped:

```lua
function ITEM:OnEquipWeapon(client, weapon)
    -- Custom logic after equipping
    client:EmitSound("weapons/draw.wav")
    client:SetRunSpeed(client:GetRunSpeed() * 0.9)
end
```

### OnUnequipWeapon(client, weapon)

**Reference**: `gamemode/items/base/sh_weapons.lua:218-220`

Called after weapon is unequipped:

```lua
function ITEM:OnUnequipWeapon(client, weapon)
    -- Custom logic after unequipping
    client:SetRunSpeed(client:GetRunSpeed() / 0.9)
end
```

## Best Practices

### ✅ DO

- Use `ITEM.base = "base_weapons"` for all weapon items
- Set unique `weaponCategory` to control equipment slots
- Let framework handle ammo persistence via `GetData("ammo")`
- Use `OnEquipWeapon`/`OnUnequipWeapon` for custom behavior
- Set `isGrenade = true` for throwable weapons
- Use `item:Equip(client)` to equip weapons programmatically
- Check `item:GetData("equip")` to test if weapon is equipped

### ❌ DON'T

- Don't call `player:Give()` directly for inventory weapons
- Don't manually save/load weapon ammo
- Don't modify `client.carryWeapons` table directly
- Don't allow multiple weapons in same category
- Don't forget to set `ITEM.class` to valid weapon class
- Don't strip weapons without calling `item:Unequip()`
- Don't implement custom equipment state tracking

## Complete Example

```lua
-- schema/items/sh_combatpistol.lua
ITEM.name = "Combat Pistol"
ITEM.description = "A military-grade sidearm with high accuracy."
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.class = "weapon_glock2"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.weaponCategory = "sidearm"
ITEM.base = "base_weapons"

-- Optional: Custom equip behavior
function ITEM:OnEquipWeapon(client, weapon)
    client:ChatPrint("You equipped a combat pistol.")

    -- Give bonus attribute while equipped
    local character = client:GetCharacter()
    if character then
        character:AddBoost("pistol_accuracy", "aimskill", 5)
    end
end

-- Optional: Custom unequip behavior
function ITEM:OnUnequipWeapon(client, weapon)
    local character = client:GetCharacter()
    if character then
        character:RemoveBoost("pistol_accuracy", "aimskill")
    end
end
```

## Common Issues

### Issue: Ammo Not Saving

**Cause**: Calling `player:StripWeapon()` instead of `item:Unequip()`
**Fix**: Always use the item's Unequip method

```lua
-- WRONG
player:StripWeapon(weapon:GetClass())

-- CORRECT
item:Unequip(player)
```

### Issue: Multiple Weapons in Same Slot

**Cause**: Using same `weaponCategory` for different weapons
**Fix**: Use unique categories or check before equipping

```lua
-- Weapons with same category conflict:
ITEM.weaponCategory = "sidearm"  -- Both use "sidearm"

-- Fix: Use different categories
ITEM.weaponCategory = "sidearm"   -- Pistol
ITEM.weaponCategory = "primary"   -- Rifle
```

### Issue: Weapon Not Appearing on Reconnect

**Cause**: `OnLoadout` not being called or equipment state lost
**Fix**: Ensure `OnLoadout` is defined (it is in base_weapons)

The framework automatically calls `OnLoadout` - no custom implementation needed.

### Issue: Grenade Not Being Removed

**Cause**: Not setting `isGrenade = true`
**Fix**: Set the flag for throwable weapons

```lua
ITEM.isGrenade = true
ITEM.class = "weapon_frag"
```

## See Also

- [Base Items](base-items.md) - Overview of all base item types
- [Ammunition](ammunition.md) - Ammo items documentation
- [Item System](../systems/items.md) - Core item system
- [Inventory System](../systems/inventory.md) - Inventory management
- Source: `gamemode/items/base/sh_weapons.lua`
