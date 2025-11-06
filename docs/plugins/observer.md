# Observer Plugin

> **Reference**: `plugins/observer.lua`

The Observer plugin enhances noclip mode for administrators, providing an invisible spectator mode with ESP visualization of players, automatic return-to-position functionality, and logging. Perfect for monitoring gameplay without interfering.

## ⚠️ Important: Use Built-in Observer Mode

**Always use the Helix observer system** via noclip. The framework provides:
- Invisible spectator mode (players can't see you)
- ESP showing all players with health, names, and aim direction
- Optional teleport-back to original position
- Prevents vehicle entry while observing
- Automatic logging of observer sessions
- God mode and no-target while observing

## Core Concepts

### What is Observer Mode?

Observer mode is an enhanced noclip that:
- Makes admin completely invisible
- Shows ESP overlay of all players
- Saves admin's position before observing
- Optionally teleports back when exiting
- Prevents interaction with world
- Logs entry/exit for accountability

## Client Options

### observerESP

**Reference**: `plugins/observer.lua:20-25`

- **Type**: Boolean
- **Default**: true
- **Description**: Show player ESP while in observer mode
- **Visible**: Only to players with "Helix - Observer" privilege

### observerTeleportBack

**Reference**: `plugins/observer.lua:11-17`

- **Type**: Boolean
- **Default**: true
- **Networked**: true
- **Description**: Teleport back to original position when exiting observer
- **Visible**: Only to players with "Helix - Observer" privilege

## Using Observer Mode

### Entering Observer

Press V key (default noclip bind) to toggle observer mode.

**What Happens**:
- Position and angles saved
- Player becomes invisible
- Player becomes non-solid
- God mode enabled
- No-target enabled (NPCs ignore you)
- ESP activates (if enabled)

### Exiting Observer

Press V again to exit.

**What Happens**:
- If teleportBack enabled: Returns to original position
- If teleportBack disabled: Stays at current position
- Player becomes visible
- Player becomes solid
- God mode disabled
- No-target disabled

## ESP Features

**Reference**: `plugins/observer.lua:31-92`

When in observer mode with ESP enabled, you see:
- **Player Boxes**: Team-colored squares showing player positions
- **Player Names**: Above each player
- **Health Bars**: Visual health indicator above name
- **Aim Lines**: Shows where each player is looking
- **Distance Fading**: Farther players appear dimmer
- **Obstruction Check**: Aim lines dimmed if view obstructed

### ESP Range

- **Maximum Range**: 1024 units
- **Fade Start**: Players beyond 512 units fade
- **Minimum Alpha**: 80 (always somewhat visible within range)

## Developer Hooks

### CanPlayerEnterObserver

**Reference**: `plugins/observer.lua:126-130`

```lua
-- Control who can enter observer mode
function PLUGIN:CanPlayerEnterObserver(client)
    -- Return true to allow, false/nil to deny

    -- Check custom permission
    if client:GetUserGroup() == "moderator" then
        return true
    end

    -- Default: Checks for "Helix - Observer" privilege
end
```

### OnPlayerObserve

**Reference**: `plugins/observer.lua:184-190`

```lua
-- Called when player enters/exits observer
function PLUGIN:OnPlayerObserve(client, state)
    -- state = true when entering
    -- state = false when exiting

    print(client:Name() .. (state and " started" or " stopped") .. " observing")
end
```

### ShouldPermakillCharacter Hook Example

While not specific to observer, this is useful for preventing permakill while testing:

```lua
function PLUGIN:ShouldPermakillCharacter(client, character, inflictor, attacker)
    if client:GetMoveType() == MOVETYPE_NOCLIP then
        return false  -- Don't permakill while in observer
    end
end
```

## Complete Example

```lua
-- Custom observer ESP colors
hook.Add("HUDPaint", "CustomObserverESP", function()
    -- Add custom ESP elements while in observer
    local client = LocalPlayer()

    if client:GetMoveType() == MOVETYPE_NOCLIP and
       CAMI.PlayerHasAccess(client, "Helix - Observer", nil) then
        -- Draw custom information
    end
end)
```

## Best Practices

### ✅ DO

- Use observer mode for non-intrusive monitoring
- Enable teleportBack for quick returns
- Use ESP to track player activity
- Log observer sessions for accountability
- Check CanPlayerEnterObserver for custom permissions

### ❌ DON'T

- Don't interfere with gameplay while observing
- Don't use observer to gain unfair advantage
- Don't disable logging without good reason
- Don't spawn props while in observer
- Don't forget you're invisible to players

## Common Issues

### Can't Enter Observer Mode

**Cause**: Missing "Helix - Observer" privilege
**Fix**: Grant privilege via admin mod

### ESP Not Showing

**Cause**: observerESP option disabled
**Fix**: Enable in F1 Options > Observer

### Not Teleporting Back

**Cause**: observerTeleportBack option disabled
**Fix**: Enable option or manually navigate back

## Technical Details

**Reference**: `plugins/observer.lua:138-182`

When entering observer:
1. Saves position and angles to `client.ixObsData`
2. Sets NoDraw, NotSolid
3. Disables world model and shadow
4. Enables god mode
5. Sets NoTarget flag

When exiting:
1. Reads saved position from `ixObsData`
2. Teleports player (if option enabled)
3. Restores visibility
4. Restores collision
5. Disables god mode

## See Also

- [Recognition Plugin](recognition.md) - Player recognition system
- [Logging Plugin](logging.md) - Event logging
- Source: `plugins/observer.lua`
