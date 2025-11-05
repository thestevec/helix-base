# Crosshair Plugin

> **Reference**: `plugins/crosshair.lua`

The Crosshair plugin provides a dynamic, customizable crosshair for aiming. The crosshair changes size and opacity based on weapon state, distance to target, and whether the weapon is raised, providing visual feedback for player aiming.

## ⚠️ Important: Use Built-in Crosshair System

**The plugin automatically provides a crosshair** for all weapons. The framework provides:
- Dynamic crosshair that responds to weapon state
- Distance-based sizing
- Weapon raised/lowered visual feedback
- Item pickup visual indicator
- Customizable via hooks
- Smooth transitions and animations

## Core Concepts

### What is the Crosshair?

The crosshair is a dynamic aiming reticle that:
- Appears in the center of the screen
- Changes size based on distance to target
- Changes opacity based on weapon raised state
- Highlights when looking at pickupable items
- Respects weapon-specific crosshair settings
- Accounts for view punch (recoil)

### Key Features

- **Dynamic Sizing**: Crosshair gap changes with distance
- **Weapon State Feedback**: Opacity changes when weapon lowered
- **Item Highlighting**: Special appearance when looking at items
- **Smooth Transitions**: Lerped animations for smooth changes
- **Vehicle Support**: Works correctly in vehicles
- **Custom Hook**: `GetCrosshairAlpha` for custom alpha values
- **Weapon Override**: Weapons can define custom crosshair drawing

## Crosshair Behavior

### Distance-Based Sizing

**Reference**: `plugins/crosshair.lua:31-34`

```lua
-- Crosshair gap calculation:
-- - Smaller gap = closer target
-- - Larger gap = farther target
-- - Maximum distance: 1000 units
-- - Scale factor: 0-0.5 based on distance

-- Close target (< 100 units): Small gap (~12 pixels)
-- Medium target (500 units): Medium gap (~18 pixels)
-- Far target (1000+ units): Large gap (~25 pixels)
```

### Weapon Raised State

**Reference**: `plugins/crosshair.lua:34-43`

```lua
-- Weapon raised (Alt held):
-- - Gap: Normal based on distance
-- - Alpha: 150 (semi-transparent)

-- Weapon lowered (Alt released):
-- - Gap: -10% smaller
-- - Alpha: 255 (fully visible)

-- Indicates "not ready to fire" when weapon lowered
```

### Item Pickup Indicator

**Reference**: `plugins/crosshair.lua:36-40`

```lua
-- When looking at ix_item entity:
-- - Gap: 0 (dots converge to center)
-- - Size: 5 pixels (larger dots)
-- - Range: 128 units

-- Visual feedback that item can be picked up
```

## Customization

### GetCrosshairAlpha Hook

**Reference**: `plugins/crosshair.lua:44`

```lua
-- Modify crosshair transparency
function PLUGIN:GetCrosshairAlpha(alpha)
    -- Return new alpha value or nil to use default

    -- Example: Force full opacity
    return 255

    -- Example: Hide when in safezone
    if LocalPlayer():GetArea() == "safezone" then
        return 0  -- Invisible
    end

    return alpha  -- Use default
end
```

### ShouldDrawCrosshair Hook

**Reference**: `plugins/crosshair.lua:70`

```lua
-- Control whether crosshair should draw
function PLUGIN:ShouldDrawCrosshair(client, weapon)
    -- Return false to hide crosshair
    -- Return true to force show crosshair
    -- Return nil to use default logic

    -- Example: Hide when in vehicle
    if client:InVehicle() then
        return false
    end

    -- Example: Force show for specific weapon
    if IsValid(weapon) and weapon:GetClass() == "weapon_special" then
        return true
    end
end
```

### Weapon-Specific Crosshair

**Reference**: `plugins/crosshair.lua:102-105`

```lua
-- In your SWEP definition
SWEP.DrawCrosshair = false  -- Hide crosshair for this weapon

-- Or custom drawing:
function SWEP:DoDrawCrosshair(x, y, trace)
    -- Draw custom crosshair
    -- x, y: Screen center position
    -- trace: Trace result from player's aim

    surface.SetDrawColor(255, 0, 0)
    surface.DrawRect(x - 2, y - 2, 4, 4)
end
```

## Complete Examples

### Example 1: Hide Crosshair in Safezones

```lua
-- In your plugin sh_plugin.lua
function PLUGIN:ShouldDrawCrosshair(client)
    local area = client:GetArea()

    if area == "spawn_safezone" or area == "nexus" then
        return false  -- No crosshair in safe areas
    end
end
```

### Example 2: Custom Alpha Based on Stamina

```lua
-- In your plugin cl_hooks.lua
function PLUGIN:GetCrosshairAlpha(alpha)
    local client = LocalPlayer()
    local stamina = client:GetLocalVar("stm", 100)

    -- Fade crosshair when out of stamina
    if stamina < 20 then
        return alpha * (stamina / 20)
    end

    return alpha
end
```

### Example 3: Different Crosshair for Medic Class

```lua
function PLUGIN:ShouldDrawCrosshair(client, weapon)
    local character = client:GetCharacter()

    if character and character:GetClass() == CLASS_MEDIC then
        -- Medics always have crosshair visible
        return true
    end
end

function PLUGIN:GetCrosshairAlpha(alpha)
    local character = LocalPlayer():GetCharacter()

    if character and character:GetClass() == CLASS_MEDIC then
        return 255  -- Full brightness for medics
    end

    return alpha
end
```

### Example 4: Pulsing Crosshair When Low Health

```lua
function PLUGIN:GetCrosshairAlpha(alpha)
    local client = LocalPlayer()
    local health = client:Health()

    if health < 25 then
        -- Pulse between 100 and 255 when low health
        local pulse = math.abs(math.sin(CurTime() * 3)) * 155 + 100
        return math.min(alpha, pulse)
    end

    return alpha
end
```

## Best Practices

### ✅ DO

- Use `ShouldDrawCrosshair` to hide/show crosshair
- Use `GetCrosshairAlpha` to modify transparency
- Return nil from hooks to use default behavior
- Test crosshair with different weapons
- Ensure crosshair is visible for gameplay-critical aiming

### ❌ DON'T

- Don't hide crosshair without player option to re-enable
- Don't make crosshair too bright or distracting
- Don't forget to handle vehicle crosshair
- Don't override without good gameplay reason
- Don't make crosshair invisible during combat

## Common Patterns

### Pattern 1: Conditional Crosshair Display

```lua
function PLUGIN:ShouldDrawCrosshair(client, weapon)
    -- Only show for specific weapon types
    if IsValid(weapon) and weapon.IsMeleeWeapon then
        return false  -- No crosshair for melee
    end
end
```

### Pattern 2: Fade Crosshair Based on Status

```lua
function PLUGIN:GetCrosshairAlpha(alpha)
    local client = LocalPlayer()

    -- Fade when bleeding
    if client:GetNWBool("ixBleeding") then
        return alpha * 0.5
    end

    return alpha
end
```

### Pattern 3: No Crosshair for Binoculars

```lua
function PLUGIN:ShouldDrawCrosshair(client, weapon)
    if IsValid(weapon) and weapon:GetClass() == "ix_binoculars" then
        return false
    end
end
```

## Common Issues

### Crosshair Not Showing

**Cause**: Weapon has DrawCrosshair = false or hook blocking it
**Fix**:
- Check weapon definition
- Check ShouldDrawCrosshair hooks
- Ensure character is loaded and alive

```lua
-- Debug: Check why crosshair not showing
hook.Add("ShouldDrawCrosshair", "Debug", function(client, weapon)
    print("Crosshair check:")
    print("Weapon:", weapon)
    print("Weapon.DrawCrosshair:", weapon.DrawCrosshair)
end)
```

### Crosshair Wrong Position

**Cause**: Screen resolution or vehicle angles
**Fix**: Plugin accounts for this, but custom crosshairs may need adjustment

```lua
-- Crosshair uses trace to hit position, converted to screen coords
-- This handles view punch and vehicle angles automatically
```

### Crosshair Too Large/Small

**Cause**: Distance calculation or custom gap values
**Fix**: Crosshair size is based on distance to target (intended)

### Custom Crosshair Not Drawing

**Cause**: DoDrawCrosshair not defined correctly
**Fix**:

```lua
-- Ensure function is defined on SWEP table
function SWEP:DoDrawCrosshair(x, y, trace)
    -- x and y are screen coordinates
    -- trace is the aim trace result
    surface.SetDrawColor(255, 255, 255)
    surface.DrawRect(x - 1, y - 1, 2, 2)
end
```

## Technical Details

### Crosshair Composition

**Reference**: `plugins/crosshair.lua:9-17`

Crosshair consists of:
- 5 dots: Center + Top + Bottom + Left + Right
- Each dot: 4x4 pixels (default size)
- Outlined with black border
- Colored fill (white/red based on state)

### Gap Calculation

**Reference**: `plugins/crosshair.lua:31-34`

```lua
-- Distance to target
distance = trace.StartPos:DistToSqr(trace.HitPos)

-- Scale: 1.0 at close range, 0.5 at max range
scaleFraction = 1 - clamp(distance / 1000000, 0, 0.5)

-- Base gap: 25 pixels
-- Weapon lowered: -10% gap (22.5 pixels)
-- Weapon raised: Full gap based on distance
crossGap = 25 * (scaleFraction - weaponLoweredPenalty)
```

### Alpha Transitions

**Reference**: `plugins/crosshair.lua:42-44`

```lua
-- Alpha lerps smoothly between states
-- - Weapon raised: 150 alpha
-- - Weapon lowered: 255 alpha
-- - Lerp speed: FrameTime * 2

-- Results in smooth fade in/out when raising/lowering weapon
```

### Trace Calculation

**Reference**: `plugins/crosshair.lua:92-96`

```lua
-- Trace from player eye position
-- Direction: Eye angles + View punch + Vehicle angles
-- Length: 65535 units (effectively infinite)
-- Filter: Player entity (and vehicle if in one)
-- Result: What player is looking at
```

### Drawing Order

**Reference**: `plugins/crosshair.lua:57-109`

1. Check if player has character and is alive
2. Check if ragdolled
3. Check if weapon is valid
4. Check ShouldDrawCrosshair hook
5. Check if context menu or character menu open
6. Calculate aim vector with punch angle
7. Perform trace
8. Call DoDrawCrosshair (weapon or plugin)
9. Draw 5 dots with calculated gap and alpha

## Performance

- **Minimal overhead**: Only draws when needed
- **Optimized calculations**: Uses squared distance to avoid sqrt
- **Cached values**: FrameTime and frequently used values cached
- **Single trace**: One trace per frame when crosshair visible

## See Also

- [Weapon Items](../items/weapons.md) - Creating weapon items
- [HUD System](../ui/hud.md) - Custom HUD elements
- [Plugin System](plugin-system.md) - Understanding Helix plugins
- Source: `plugins/crosshair.lua`
