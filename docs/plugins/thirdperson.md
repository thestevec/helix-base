# Third Person Plugin

> **Reference**: `plugins/thirdperson.lua`

The Third Person plugin enables players to view their character from a third-person camera perspective. Provides fully customizable camera positioning with distance, vertical offset, and horizontal offset controls, plus classic vs modern movement modes.

## ⚠️ Important: Use Built-in Third Person

**Always use the Helix third person system**. The framework provides:
- Automatic view override and camera positioning
- Collision detection for camera placement
- Customizable camera distance and offsets
- Classic or modern movement modes
- Smooth crouch transitions
- Vehicle compatibility
- Client-side options for personalization

## Core Concepts

### What is Third Person?

Third person mode allows players to:
- See their character from behind
- Customize camera position
- Switch between first/third person
- Use classic (camera-relative) or modern (character-relative) movement

## Server Configuration

### thirdperson

**Reference**: `plugins/thirdperson.lua:8-10`

- **Type**: Boolean
- **Default**: false
- **Description**: Allow third person camera on the server
- **Console**: `ix_config_thirdperson 1`

**Must be enabled for clients to use third person.**

## Client Options

All options available in F1 Options > Third Person menu.

### thirdpersonEnabled

**Reference**: `plugins/thirdperson.lua:17-23`

- **Type**: Boolean
- **Default**: false
- **Description**: Enable third person view
- **Console**: `ix_option_thirdpersonEnabled 1`
- **Command**: `ix_togglethirdperson`

### thirdpersonClassic

**Reference**: `plugins/thirdperson.lua:25-28`

- **Type**: Boolean
- **Default**: false
- **Description**: Use classic camera-relative movement (like old third person games)

**Classic vs Modern**:
- **Classic**: Movement based on camera direction (like GTA 3)
- **Modern**: Movement based on character direction (like modern games)

### thirdpersonVertical

**Reference**: `plugins/thirdperson.lua:30-33`

- **Type**: Number
- **Default**: 10
- **Range**: 0 to 30
- **Description**: Vertical camera offset (up/down)

### thirdpersonHorizontal

**Reference**: `plugins/thirdperson.lua:35-38`

- **Type**: Number
- **Default**: 0
- **Range**: -30 to 30
- **Description**: Horizontal camera offset (left/right)

### thirdpersonDistance

**Reference**: `plugins/thirdperson.lua:40-43`

- **Type**: Number
- **Default**: 50
- **Range**: 0 to 100
- **Description**: Camera distance from character

## Using Third Person

### Toggle Third Person

```lua
-- Press bind (default none, must set)
-- Or use console command:
ix_togglethirdperson

-- Or programmatically:
ix.option.Set("thirdpersonEnabled", true)
```

### Console Command

**Reference**: `plugins/thirdperson.lua:46-50`

```
ix_togglethirdperson
```

Toggles third person on/off.

## Camera Behavior

### Collision Detection

**Reference**: `plugins/thirdperson.lua:102-112`

Camera automatically moves forward if it would clip through walls:
- Uses hull trace to detect collisions
- Camera never goes inside geometry
- Smoothly adjusts distance

### Crouch Adjustment

**Reference**: `plugins/thirdperson.lua:94-98`

Camera smoothly lowers when crouching:
- Lerps between standing/crouching heights
- Smooth 5x FrameTime transition

### Noclip Mode

**Reference**: `plugins/thirdperson.lua:92, 109`

Camera ignores world collision when in noclip:
- Allows free camera movement
- Useful for admins

## Developer Integration

### CanOverrideView

**Reference**: `plugins/thirdperson.lua:60-82`

Custom check for when third person should work:

```lua
-- Example: Disable third person in vehicles
function Player:CanOverrideView()
    if self:InVehicle() then
        return false
    end

    -- Default checks still apply
end
```

### Hooks

**ThirdPersonToggled**:
**Reference**: `plugins/thirdperson.lua:20-22`

```lua
function PLUGIN:ThirdPersonToggled(oldValue, newValue)
    if newValue then
        -- Player enabled third person
    else
        -- Player disabled third person
    end
end
```

## Complete Examples

### Example 1: Disable in Combat

```lua
function PLUGIN:CanOverrideView(client)
    if client:GetNWBool("inCombat") then
        return false
    end
end
```

### Example 2: Force Third Person for Class

```lua
hook.Add("PlayerLoadedCharacter", "ForceThirdPerson", function(client)
    local character = client:GetCharacter()

    if character:GetClass() == CLASS_OBSERVER then
        ix.option.Set(client, "thirdpersonEnabled", true)
    end
end)
```

### Example 3: Custom Camera Position for VIP

```lua
function PLUGIN:ThirdPersonToggled(oldValue, newValue)
    local client = LocalPlayer()

    if client:GetUserGroup() == "vip" and newValue then
        ix.option.Set("thirdpersonDistance", 100)  -- Farther camera
        ix.option.Set("thirdpersonVertical", 20)   -- Higher camera
    end
end
```

## Best Practices

### ✅ DO

- Let players customize camera position
- Use collision detection (built-in)
- Provide toggle command or bind
- Test with different player models
- Handle crouch transitions smoothly

### ❌ DON'T

- Don't force third person without option
- Don't allow in competitive modes (unfair advantage)
- Don't override without collision check
- Don't forget vehicle compatibility
- Don't use extreme camera distances

## Common Issues

### Third Person Not Working

**Cause**: Server config disabled
**Fix**: Enable `ix_config_thirdperson 1`

### Camera Inside Walls

**Cause**: Collision detection disabled
**Fix**: Plugin handles this automatically

### Movement Feels Wrong

**Cause**: Classic mode enabled/disabled
**Fix**: Toggle `thirdpersonClassic` option

### Can't See Character

**Cause**: Distance set to 0
**Fix**: Increase `thirdpersonDistance`

## Technical Details

### View Calculation

**Reference**: `plugins/thirdperson.lua:88-136`

Camera position calculated as:
1. Start at player position + view offset
2. Add vertical offset
3. Add horizontal offset
4. Adjust for crouch
5. Move back by distance
6. Trace for collisions
7. Use trace hit position

### Movement Adjustment

**Reference**: `plugins/thirdperson.lua:138-153`

In classic mode:
- Forward/back inputs adjusted for camera angle
- Strafing rotated to match camera
- Allows movement in camera direction

In modern mode:
- Pitch adjusted to camera pitch
- Yaw remains character yaw
- Movement relative to character

### Mouse Input

**Reference**: `plugins/thirdperson.lua:155-168`

Camera angles stored separately from player angles:
- `owner.camAng` stores camera orientation
- Pitch clamped to -85/+85 degrees
- Allows free look without turning character (in modern mode)

## See Also

- [Observer Plugin](observer.md) - Admin spectator mode
- [Options System](../systems/options.md) - Client preferences
- Source: `plugins/thirdperson.lua`
