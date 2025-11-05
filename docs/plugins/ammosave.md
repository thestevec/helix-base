# Ammo Saver Plugin

> **Reference**: `plugins/ammosave.lua`

The Ammo Saver plugin automatically saves and restores ammunition counts for each character. When a player disconnects or switches characters, their ammo is saved to character data and restored when they rejoin, ensuring ammunition persistence across sessions.

## ⚠️ Important: Use Built-in Ammo System

**Always use `ix.ammo.Register()`** to register custom ammunition types. The framework provides:
- Automatic saving of ammo on character save
- Automatic restoration on character load
- Support for all Source engine ammo types
- Character-specific ammo storage (not shared between characters)
- Integration with Helix data system

## Core Concepts

### What is Ammo Saver?

Ammo Saver is a persistence plugin that:
- Saves each character's ammunition counts to database
- Restores ammo when the character loads
- Tracks multiple ammo types simultaneously
- Integrates with Helix character data system
- Only saves ammo types that have been registered

### Key Features

- **Automatic Persistence**: No commands needed, works automatically
- **Character-Specific**: Each character has separate ammo storage
- **Multi-Ammo Support**: Handles all registered ammunition types
- **HL2 Ammo Included**: All Half-Life 2 ammo types pre-registered
- **Extensible**: Easy to register custom ammo types

## How It Works

### Automatic Saving

**Reference**: `plugins/ammosave.lua:46-64`

When a character is saved (disconnect, character switch, server shutdown):
1. Plugin checks if player is valid
2. Loops through all registered ammo types
3. Gets current ammo count for each type
4. Saves non-zero counts to character data
5. Empty ammo types are not saved (optimization)

### Automatic Loading

**Reference**: `plugins/ammosave.lua:67-89`

When a character loads (login, character selection):
1. Waits 0.25 seconds for loadout to complete
2. Gets saved ammo data from character
3. Restores each ammo type to saved amount
4. If no saved data exists, player starts with default ammo

## Registering Custom Ammo

### ix.ammo.Register()

**Reference**: `plugins/ammosave.lua:11-17`

```lua
-- Register a custom ammo type
ix.ammo.Register(name)

-- Example usage:
ix.ammo.Register("my_custom_ammo")
ix.ammo.Register("laser_cell")
ix.ammo.Register("plasma_round")
```

**Parameters**:
- `name` (string): The ammo type name (will be converted to lowercase)

**How it works**:
- Converts name to lowercase for consistency
- Checks if ammo is already registered
- Adds to the plugin's ammo list if new
- Prevents duplicate registrations

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually modify the ammoList table
PLUGIN.ammoList[#PLUGIN.ammoList + 1] = "myammo"

-- CORRECT: Use the registration function
ix.ammo.Register("myammo")
```

## Complete Example

```lua
-- In your schema or plugin sh_plugin.lua
PLUGIN.name = "Custom Weapons"
PLUGIN.author = "Your Name"
PLUGIN.description = "Adds custom weapons with custom ammo"

-- Register your custom ammo types
ix.ammo.Register("energy_cell")
ix.ammo.Register("plasma_round")
ix.ammo.Register("railgun_slug")

-- Now when players use weapons with these ammo types,
-- their ammo will automatically save and restore
```

## Pre-Registered Ammo Types

### Half-Life 2 Ammo

**Reference**: `plugins/ammosave.lua:19-30`

All standard HL2 ammo types are pre-registered:
- `ar2` - Pulse Rifle ammo
- `pistol` - Pistol ammo
- `357` - Magnum ammo
- `smg1` - SMG ammo
- `xbowbolt` - Crossbow bolts
- `buckshot` - Shotgun shells
- `rpg_round` - RPG rockets
- `smg1_grenade` - SMG grenades
- `grenade` - Hand grenades
- `ar2altfire` - Pulse Rifle energy balls
- `slam` - SLAM mines

### Cut HL2 Ammo

**Reference**: `plugins/ammosave.lua:32-43`

Cut/unused HL2 ammo types also registered:
- `alyxgun` - Alyx's Gun
- `sniperround` - Sniper rounds
- `sniperpenetratedround` - Penetrating sniper rounds
- `thumper` - Thumper ammo
- `gravity` - Gravity Gun "ammo"
- `battery` - Suit battery
- `gaussenergy` - Gauss Gun energy
- `combinecannon` - Combine cannon ammo
- `airboatgun` - Airboat gun ammo
- `striderminigun` - Strider minigun ammo
- `helicoptergun` - Helicopter gun ammo

## Best Practices

### ✅ DO

- Register custom ammo types in your schema's sh_plugin.lua
- Use lowercase ammo names consistently
- Register ammo before players spawn
- Test that ammo saves/restores correctly
- Use standard HL2 ammo types when possible
- Register ammo types once (in shared file)

### ❌ DON'T

- Don't rely on ammo saving for unregistered types
- Don't register ammo in PlayerLoadedCharacter (too late)
- Don't modify ammo during CharacterPreSave hook
- Don't assume ammo will save if not registered
- Don't register ammo types multiple times
- Don't use uppercase in ammo names (will be lowercased anyway)

## Common Patterns

### Pattern 1: Schema with Custom Ammo

```lua
-- schema/sh_schema.lua or schema/sh_plugin.lua
SCHEMA.name = "Sci-Fi RP"
SCHEMA.author = "Your Name"

-- Register our custom ammo types
ix.ammo.Register("laser_cell")
ix.ammo.Register("plasma_charge")
ix.ammo.Register("fusion_core")
```

### Pattern 2: Weapon Plugin with Custom Ammo

```lua
-- plugins/energyweapons/sh_plugin.lua
PLUGIN.name = "Energy Weapons"
PLUGIN.author = "Your Name"

-- Register the ammo types used by our weapons
ix.ammo.Register("energy_small")
ix.ammo.Register("energy_large")
```

### Pattern 3: Converting Existing Schema

```lua
-- If you already have custom weapons with custom ammo,
-- just register the ammo types and they'll auto-save

-- List all custom ammo types
ix.ammo.Register("bolts")
ix.ammo.Register("arrows")
ix.ammo.Register("shells_12gauge")
ix.ammo.Register("shells_20gauge")
```

## Common Issues

### Ammo Not Saving

**Cause**: Ammo type not registered
**Fix**: Register the ammo type using `ix.ammo.Register()`

```lua
-- If your weapon uses "my_special_ammo" ammo type
ix.ammo.Register("my_special_ammo")
```

### Ammo Not Restoring

**Cause**: Character data not loading properly or timing issue
**Fix**:
- Verify ammo is registered
- Check character data is saving (look in database)
- 0.25 second delay should handle most loading issues

### Duplicate Registration Warning

**Cause**: Registering same ammo multiple times
**Fix**: Only register once in shared file

```lua
-- WRONG: Registering in multiple files
-- sv_hooks.lua:
ix.ammo.Register("custom")
-- cl_hooks.lua:
ix.ammo.Register("custom")

-- CORRECT: Register once in shared file
-- sh_plugin.lua:
ix.ammo.Register("custom")
```

### Wrong Case Causing Issues

**Cause**: Using uppercase in GetAmmoCount/SetAmmo
**Fix**: Use lowercase (plugin auto-converts)

```lua
-- This works fine, plugin handles case conversion
ix.ammo.Register("MyAmmo")

-- Both of these will work in game
client:SetAmmo(100, "MyAmmo")
client:SetAmmo(100, "myammo")
```

## Technical Details

### Data Storage

**Reference**: `plugins/ammosave.lua:62`

Ammo is stored in character data:
- Key: `"ammo"`
- Value: Table of `{ammoType = count}`
- Only non-zero ammo is saved
- Example: `{pistol = 45, smg1 = 120, buckshot = 18}`

### Timing

**Reference**: `plugins/ammosave.lua:68`

- **Save**: Immediate (during CharacterPreSave)
- **Load**: 0.25 second delay after PlayerLoadedCharacter
- Delay ensures player loadout is set before restoring ammo
- Prevents race conditions with spawn hooks

### Hook Integration

The plugin uses two hooks:
1. `CharacterPreSave` - Saves ammo before character data is written
2. `PlayerLoadedCharacter` - Restores ammo after character loads

### Ammo Storage Format

```lua
-- What gets saved to character data:
{
    ["ar2"] = 60,
    ["pistol"] = 45,
    ["smg1"] = 120,
    ["buckshot"] = 18
    -- Ammo types with 0 count are not saved
}
```

## Integration with Other Systems

### Weapon Items

If you use Helix weapon items (base/sh_weapons.lua), ammo saving works automatically:
- Player picks up weapon item
- Weapon gives ammo
- Ammo saves with character
- On next login, ammo is restored

### Inventory Ammo Items

If you have ammo items in inventory:
- Items stay in inventory (handled by inventory system)
- Ammo Saver handles the player's loaded ammo
- Both systems work together

## Configuration

No configuration needed. The plugin works automatically once enabled.

## Disabling the Plugin

To disable ammo persistence:
1. Remove or disable the plugin
2. Ammo will no longer save between sessions
3. Existing saved ammo data remains in database but won't be loaded

## See Also

- [Inventory System](../systems/inventory.md) - Item-based inventory
- [Items: Weapons](../items/weapons.md) - Weapon item types
- [Items: Ammunition](../items/ammunition.md) - Ammo item types
- [Character System](../systems/characters.md) - Character data storage
- [Data System](../libraries/data.md) - Persistent data storage
- Source: `plugins/ammosave.lua`
