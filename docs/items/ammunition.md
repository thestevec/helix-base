# Ammunition Items

> **Reference**: `gamemode/items/base/sh_ammo.lua`

The ammunition base item provides consumable ammo boxes that add ammunition to the player's ammo pool.

## ⚠️ Important: Use Built-in Ammo Item Base

**Always extend from `base_ammo`** rather than creating custom ammo systems. The framework provides:
- Automatic ammo registration with `ix.ammo` (line 42-46)
- Remaining rounds display in inventory (line 18-23)
- Dynamic description with ammo count (line 12-15)
- Built-in use function (line 27-39)
- Proper ammo type management

## Core Concepts

### What is an Ammo Item?

Ammo items are consumable items that add ammunition to a player's ammo pool. When used, they:
1. Add rounds to the player's ammo count
2. Play a pickup sound
3. Remove the item from inventory
4. Display remaining rounds in inventory

### Key Properties

**Reference**: `gamemode/items/base/sh_ammo.lua:2-10`

```lua
ITEM.name = "Ammo Base"
ITEM.model = "models/Items/BoxSRounds.mdl"  -- Inventory model
ITEM.width = 1                               -- Inventory width
ITEM.height = 1                              -- Inventory height
ITEM.ammo = "pistol"                        -- GMod ammo type
ITEM.ammoAmount = 30                        -- Rounds in box
ITEM.description = "A Box that contains %s of Pistol Ammo"  -- %s = rounds
ITEM.category = "Ammunition"
ITEM.useSound = "items/ammo_pickup.wav"     -- Sound on use
```

### Ammo Types

Common GMod ammo types:
- `"pistol"` - Pistol rounds
- `"smg1"` - SMG rounds
- `"ar2"` - Rifle rounds
- `"357"` - Magnum rounds
- `"buckshot"` - Shotgun shells
- `"grenade"` - Grenades
- `"RPG_Round"` - RPG rockets
- `"slam"` - SLAM charges

You can also define custom ammo types.

## Using Ammo Items

### Creating a Custom Ammo Item

**Reference**: `gamemode/items/base/sh_ammo.lua:2-10`

Create a new ammo item by extending the base:

```lua
-- schema/items/sh_pistol_ammo.lua
ITEM.name = "9mm Ammunition"
ITEM.description = "A box containing %s rounds of 9mm ammunition."
ITEM.model = "models/Items/BoxSRounds.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Ammunition"
ITEM.ammo = "pistol"        -- Ammo type
ITEM.ammoAmount = 30        -- Rounds per box
ITEM.base = "base_ammo"
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't give ammo manually without item system
function GivePlayerAmmo(ply, ammoType, amount)
    ply:GiveAmmo(amount, ammoType)  -- Bypasses item system!
    -- Won't consume items, won't check inventory
end
```

### Dynamic Description

**Reference**: `gamemode/items/base/sh_ammo.lua:12-15`

The description automatically shows remaining rounds:

```lua
function ITEM:GetDescription()
    local rounds = self:GetData("rounds", self.ammoAmount)
    return Format(self.description, rounds)
end
```

This displays: *"A box containing 30 rounds of 9mm ammunition."*

As rounds are consumed, it updates: *"A box containing 15 rounds of 9mm ammunition."*

### Rounds Display

**Reference**: `gamemode/items/base/sh_ammo.lua:17-24`

CLIENT-SIDE: Ammo count is painted on the inventory icon:

```lua
if CLIENT then
    function ITEM:PaintOver(item, w, h)
        draw.SimpleText(
            item:GetData("rounds", item.ammoAmount), "DermaDefault", w - 5, h - 5,
            color_white, TEXT_ALIGN_RIGHT, TEXT_ALIGN_BOTTOM, 1, color_black
        )
    end
end
```

This shows the round count in the bottom-right corner of the item's inventory icon.

### Use Function

**Reference**: `gamemode/items/base/sh_ammo.lua:27-39`

The built-in use function handles ammo loading:

```lua
ITEM.functions.use = {
    name = "Load",
    tip = "useTip",
    icon = "icon16/add.png",
    OnRun = function(item)
        local rounds = item:GetData("rounds", item.ammoAmount)

        item.player:GiveAmmo(rounds, item.ammo)
        item.player:EmitSound(item.useSound, 110)

        return true  -- Remove item after use
    end,
}
```

Players right-click the item and select "Load" to add all rounds to their ammo pool.

### Ammo Registration

**Reference**: `gamemode/items/base/sh_ammo.lua:42-46`

Ammo types are automatically registered:

```lua
function ITEM:OnRegistered()
    if ix.ammo then
        ix.ammo.Register(self.ammo)
    end
end
```

This ensures the ammo type is properly registered with the framework's ammo system.

## Advanced Usage

### Partial Ammo Consumption

You can create ammo boxes that don't consume the entire box:

```lua
-- schema/items/sh_partial_ammo.lua
ITEM.base = "base_ammo"
ITEM.name = "Ammo Box (Partial Use)"
ITEM.ammo = "pistol"
ITEM.ammoAmount = 60

-- Override the use function
ITEM.functions.use = {
    name = "Load 10 rounds",
    OnRun = function(item)
        local rounds = item:GetData("rounds", item.ammoAmount)
        local toGive = math.min(10, rounds)

        item.player:GiveAmmo(toGive, item.ammo)
        item.player:EmitSound(item.useSound, 110)

        if rounds <= 10 then
            return true  -- Remove if depleted
        end

        item:SetData("rounds", rounds - 10)
        return false  -- Keep in inventory
    end
}
```

### Multiple Use Options

Create different loading options:

```lua
ITEM.functions.LoadAll = {
    name = "Load All",
    OnRun = function(item)
        local rounds = item:GetData("rounds", item.ammoAmount)
        item.player:GiveAmmo(rounds, item.ammo)
        item.player:EmitSound(item.useSound, 110)
        return true
    end
}

ITEM.functions.LoadHalf = {
    name = "Load Half",
    OnRun = function(item)
        local rounds = item:GetData("rounds", item.ammoAmount)
        local half = math.floor(rounds / 2)

        item.player:GiveAmmo(half, item.ammo)
        item.player:EmitSound(item.useSound, 110)

        item:SetData("rounds", rounds - half)
        return false
    end,
    OnCanRun = function(item)
        return item:GetData("rounds", item.ammoAmount) > 1
    end
}
```

### Custom Ammo Types

Define schema-specific ammo types:

```lua
-- schema/sh_ammo.lua (in schema root)
-- Define custom ammo type
game.AddAmmoType({
    name = "762x39",
    dmgtype = DMG_BULLET,
    tracer = TRACER_LINE,
    plydmg = 0,
    npcdmg = 0,
    force = 2000,
    minsplash = 10,
    maxsplash = 5
})

-- Then use in item:
-- schema/items/sh_ak_ammo.lua
ITEM.name = "7.62x39mm Ammo"
ITEM.ammo = "762x39"  -- Custom ammo type
ITEM.ammoAmount = 30
ITEM.base = "base_ammo"
```

## Ammo Management

### Checking Player Ammo

```lua
-- Check if player has ammo
local ammoCount = player:GetAmmoCount("pistol")

if ammoCount < 10 then
    player:ChatPrint("You're running low on ammo!")
end
```

### Setting Ammo Limits

Control maximum ammo per type in schema:

```lua
-- schema/sh_schema.lua
function Schema:PlayerSpawn(client)
    -- Limit pistol ammo to 120 rounds
    game.SetAmmoMax("pistol", 120)
    game.SetAmmoMax("ar2", 180)
end
```

### Ammo as Loot/Rewards

Spawn ammo items as loot:

```lua
-- Give ammo item to player
local character = player:GetCharacter()
local inventory = character:GetInventory()

inventory:Add("ammo_pistol", 1)  -- Add 1 pistol ammo box
```

## Best Practices

### ✅ DO

- Use `ITEM.base = "base_ammo"` for all ammo items
- Set `ammo` property to valid GMod ammo type
- Set `ammoAmount` to define rounds per box
- Use `%s` in description for dynamic round count
- Let framework handle ammo registration via `OnRegistered`
- Use `GetData("rounds")` to track remaining ammo
- Return `true` from use function to remove empty boxes
- Return `false` to keep partially-used boxes

### ❌ DON'T

- Don't call `player:GiveAmmo()` directly for inventory ammo
- Don't create custom ammo tracking systems
- Don't forget to set `ITEM.ammo` type
- Don't hardcode ammo amount in description (use `%s`)
- Don't manually register ammo types (framework does it)
- Don't modify ammo count without `SetData()`
- Don't use negative ammo amounts

## Complete Example

```lua
-- schema/items/sh_shotgun_shells.lua
ITEM.name = "Shotgun Shells"
ITEM.description = "A box containing %s shotgun shells."
ITEM.model = "models/Items/BoxBuckshot.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Ammunition"
ITEM.ammo = "buckshot"
ITEM.ammoAmount = 12
ITEM.useSound = "items/ammo_pickup.wav"
ITEM.base = "base_ammo"

-- Optional: Custom behavior on use
ITEM.functions.use.OnRun = function(item)
    local rounds = item:GetData("rounds", item.ammoAmount)

    -- Add to player ammo pool
    item.player:GiveAmmo(rounds, item.ammo)
    item.player:EmitSound(item.useSound, 110)

    -- Log ammo pickup
    ix.log.Add(item.player, "ammoPickup", item.name, rounds)

    return true  -- Remove item
end
```

## Common Issues

### Issue: Ammo Not Adding to Player

**Cause**: Incorrect ammo type string
**Fix**: Verify ammo type matches weapon's ammo type

```lua
-- Check weapon's ammo type
local weapon = player:GetActiveWeapon()
local ammoType = weapon:GetPrimaryAmmoType()
local ammoName = game.GetAmmoName(ammoType)
print(ammoName)  -- Use this in ITEM.ammo

-- Set matching ammo type
ITEM.ammo = "pistol"  -- Must match weapon's ammo
```

### Issue: Ammo Count Not Displaying

**Cause**: Missing `PaintOver` function or CLIENT guard
**Fix**: Ensure CLIENT-side paint function exists (it's in base_ammo)

The framework provides this automatically - no custom implementation needed.

### Issue: Description Not Updating

**Cause**: Not using `%s` placeholder or `GetDescription()`
**Fix**: Use format string in description

```lua
-- WRONG
ITEM.description = "A box of 30 rounds"  -- Static

-- CORRECT
ITEM.description = "A box of %s rounds"  -- Dynamic

function ITEM:GetDescription()
    return Format(self.description, self:GetData("rounds", self.ammoAmount))
end
```

### Issue: Ammo Box Has Negative Rounds

**Cause**: Not checking rounds before subtraction
**Fix**: Validate ammo count before operations

```lua
ITEM.functions.use = {
    OnRun = function(item)
        local rounds = item:GetData("rounds", item.ammoAmount)

        if rounds <= 0 then
            return true  -- Remove if empty
        end

        item.player:GiveAmmo(rounds, item.ammo)
        return true
    end
}
```

## See Also

- [Base Items](base-items.md) - Overview of all base item types
- [Weapons](weapons.md) - Weapon items that use ammo
- [Item System](../systems/items.md) - Core item system
- [Inventory System](../systems/inventory.md) - Inventory management
- Source: `gamemode/items/base/sh_ammo.lua`
