# Weapon Select Plugin

> **Reference**: `plugins/wepselect.lua`

The Weapon Select plugin replaces Garry's Mod's default weapon selection with a stylish radial menu. Features smooth animations, weapon instructions display, and integrated sound effects for an immersive weapon switching experience.

## ⚠️ Important: Use Built-in Weapon Selection

**The plugin automatically replaces default weapon selection**. The framework provides:
- Radial weapon menu with smooth animations
- Weapon name and instructions display
- Scroll wheel and number key support
- Auto-fade after selection
- Custom sound hooks

## Using Weapon Selection

### Controls

- **Mouse Wheel Up**: Scroll to next weapon
- **Mouse Wheel Down**: Scroll to previous weapon
- **Number Keys (1-9)**: Direct weapon selection
- **Left Click**: Confirm selection
- **Wait 5 seconds**: Auto-hide menu

### Visual Elements

- **Radial Display**: Weapons arranged in arc
- **Highlight**: Current weapon shown in theme color
- **Instructions**: Weapon usage instructions below name
- **Fade Effects**: Smooth transitions in/out

## Developer Hooks

### WeaponCycleSound

**Reference**: `plugins/wepselect.lua:119-120`

```lua
-- Custom sound when scrolling weapons
function PLUGIN:WeaponCycleSound()
    -- Return sound path and pitch
    return "custom/scroll.wav", 100
end
```

### WeaponSelectSound

**Reference**: `plugins/wepselect.lua:174`

```lua
-- Custom sound when selecting weapon
function PLUGIN:WeaponSelectSound(weapon)
    -- Return sound path for this weapon
    if weapon:GetClass() == "weapon_special" then
        return "custom/select_special.wav"
    end

    -- Return nil for default sound
end
```

## Complete Examples

### Example 1: Disable for Specific Weapon

```lua
function PLUGIN:PlayerBindPress(client, bind, pressed)
    local weapon = client:GetActiveWeapon()

    if IsValid(weapon) and weapon:GetClass() == "gmod_camera" then
        return  -- Allow default behavior
    end
end
```

### Example 2: Custom Sound Per Weapon Class

```lua
function PLUGIN:WeaponSelectSound(weapon)
    local class = weapon:GetClass()

    if class:StartWith("weapon_") then
        return "HL2Player.Use"
    elseif class:StartWith("ix_") then
        return "items/battery_pickup.wav"
    end
end
```

## Best Practices

### ✅ DO

- Set weapon.Instructions for helpful tooltips
- Use custom sounds for immersion
- Test with many weapons

### ❌ DON'T

- Don't override without good reason
- Don't forget weapon instructions
- Don't use excessive weapon counts (> 20)

## See Also

- [Crosshair Plugin](crosshair.md) - Weapon aiming
- [Inventory System](../systems/inventory.md) - Item-based weapons
- Source: `plugins/wepselect.lua`
