# Areas Plugin

> **Reference**: `plugins/area/`

The Areas plugin allows administrators to define named 3D box regions on the map. These areas can detect when players enter or leave them, display information to players, and trigger custom gameplay logic. Perfect for defining zones like spawn areas, restricted zones, faction territories, or special event locations.

## ⚠️ Important: Use Built-in Area System

**Always use `ix.area.Create()` and the area editor** rather than creating custom zone detection systems. The framework provides:
- Visual area editor with in-game placement
- Automatic player position tracking
- Network synchronization to all clients
- Persistent storage across restarts
- Event hooks for area transitions
- Customizable properties per area
- Area type system for categorization

## Core Concepts

### What are Areas?

Areas are 3D axis-aligned bounding boxes (AABBs) placed on the map. They:
- Have unique names/IDs
- Can be categorized by type
- Detect players inside their bounds
- Persist across server restarts
- Sync automatically to clients
- Support custom properties (color, display settings, etc.)
- Trigger hooks when players enter/leave

### Key Features

- **Visual Editor**: In-game tool for creating/removing areas
- **Player Tracking**: Automatic detection of which area each player is in
- **Event System**: Hooks fire when players change areas
- **Customizable Types**: Define your own area categories
- **Custom Properties**: Add metadata to areas
- **Display System**: Visual area name display on client
- **Persistent Storage**: All areas save to database

### Area Properties

**Reference**: `plugins/area/sh_plugin.lua:28-33`

Default properties:
- `color` (Color): Display color for the area
- `display` (Boolean): Whether to show the area name on HUD

You can add custom properties using `ix.area.AddProperty()`.

## Using Areas

### Starting the Area Editor

**Reference**: `plugins/area/sh_plugin.lua:64-76`

```lua
-- Command usage (in chat):
/AreaEdit

-- Or programmatically:
RunConsoleCommand("ix", "AreaEdit")
```

**Editor Controls**:
- **Left Click**: Set first corner (or complete area if second click)
- **Right Click**: Remove area you're looking at
- **Reload (R)**: Cancel current area placement
- Visual preview shows area bounds while editing

### Creating Areas Programmatically

**Reference**: `plugins/area/sv_plugin.lua:23-44`

```lua
-- Server-side only
ix.area.Create(name, type, startPosition, endPosition, bNoReplicate, properties)

-- Example usage:
ix.area.Create(
    "spawn_zone",                    -- Unique name/ID
    "area",                          -- Type (default: "area")
    Vector(-500, -500, 0),           -- Start position (corner 1)
    Vector(500, 500, 200),           -- End position (corner 2)
    false,                           -- Network to clients
    {                                -- Custom properties
        color = Color(0, 255, 0),
        display = true
    }
)
```

**Parameters**:
- `name` (string): Unique identifier for the area
- `type` (string, optional): Area type (default: "area")
- `startPosition` (Vector): First corner of the box
- `endPosition` (Vector): Opposite corner of the box
- `bNoReplicate` (boolean, optional): If true, don't sync to clients
- `properties` (table, optional): Custom property values

### Removing Areas Programmatically

**Reference**: `plugins/area/sv_plugin.lua:46-55`

```lua
-- Server-side only
ix.area.Remove(name, bNoReplicate)

-- Example usage:
ix.area.Remove("spawn_zone")           -- Remove and sync
ix.area.Remove("temp_zone", true)      -- Remove without syncing
```

**Parameters**:
- `name` (string): Name of the area to remove
- `bNoReplicate` (boolean, optional): If true, don't sync removal to clients

## Player Area Functions

### Getting Current Area

**Reference**: `plugins/area/sh_plugin.lua:82-84`

```lua
-- Get the area the player is currently in
local areaID = client:GetArea()

-- Returns the area name/ID, or the last valid area if not currently in one
-- Returns "" if player has never been in an area
```

### Checking if in Area

**Reference**: `plugins/area/sh_plugin.lua:87-89`

```lua
-- Check if player is actually in an area right now
local inArea = client:IsInArea()

-- Returns true only if player is currently inside an area
-- Returns nil/false if player is outside all areas
```

## Area Events

### OnPlayerAreaChanged Hook

**Reference**: `plugins/area/sv_hooks.lua:73-78`

```lua
-- Called when a player moves from one area to another
function PLUGIN:OnPlayerAreaChanged(client, oldID, newID)
    print(client:Name() .. " moved from " .. oldID .. " to " .. newID)
end

-- Example: Faction-specific zones
function PLUGIN:OnPlayerAreaChanged(client, oldID, newID)
    local area = ix.area.stored[newID]
    local character = client:GetCharacter()

    if area and area.properties.faction then
        if character:GetFaction() != area.properties.faction then
            client:Notify("You are not authorized to be in this area!")
        end
    end
end
```

## Adding Custom Area Types

**Reference**: `plugins/area/sh_plugin.lua:35-40`

```lua
-- Define a new area type
ix.area.AddType(type, displayName)

-- Example usage:
ix.area.AddType("spawn", "Spawn Zone")
ix.area.AddType("restricted", "Restricted Area")
ix.area.AddType("safezone", "Safe Zone")
ix.area.AddType("pvp", "PVP Zone")
```

## Adding Custom Properties

**Reference**: `plugins/area/sh_plugin.lua:28-33`

```lua
-- Add a custom property to all areas
ix.area.AddProperty(name, type, default, data)

-- Example usage:
function PLUGIN:SetupAreaProperties()
    -- Add faction restriction property
    ix.area.AddProperty("faction", ix.type.number, 0)

    -- Add damage multiplier property
    ix.area.AddProperty("damageMultiplier", ix.type.number, 1.0)

    -- Add PVP enabled property
    ix.area.AddProperty("pvpEnabled", ix.type.bool, false)
end
```

## Complete Examples

### Example 1: Spawn Protection Zone

```lua
-- In your schema sh_plugin.lua
function SCHEMA:SetupAreaProperties()
    -- Add spawn protection property
    ix.area.AddProperty("spawnProtection", ix.type.bool, false)
end

-- Create spawn zone on map load
function SCHEMA:InitPostEntity()
    timer.Simple(2, function()
        ix.area.Create(
            "spawn_safe_zone",
            "spawn",
            Vector(100, 100, 0),
            Vector(500, 500, 200),
            false,
            {
                color = Color(0, 255, 0),
                display = true,
                spawnProtection = true
            }
        )
    end)
end

-- Prevent damage in spawn zones
function SCHEMA:EntityTakeDamage(target, dmg)
    if target:IsPlayer() then
        local areaID = target:GetArea()
        local area = ix.area.stored[areaID]

        if area and area.properties.spawnProtection then
            return true  -- Block damage
        end
    end
end
```

### Example 2: Faction Territory System

```lua
-- Add faction property
function PLUGIN:SetupAreaProperties()
    ix.area.AddProperty("faction", ix.type.number, 0)
    ix.area.AddProperty("trespassing", ix.type.bool, true)
end

-- Create faction territories
hook.Add("InitPostEntity", "CreateFactionZones", function()
    timer.Simple(2, function()
        -- Citizen territory
        ix.area.Create(
            "citizen_housing",
            "faction_zone",
            Vector(-1000, -1000, 0),
            Vector(-500, -500, 300),
            false,
            {
                color = Color(100, 150, 255),
                display = true,
                faction = FACTION_CITIZEN,
                trespassing = true
            }
        )

        -- Combine territory
        ix.area.Create(
            "combine_base",
            "faction_zone",
            Vector(1000, 1000, 0),
            Vector(1500, 1500, 300),
            false,
            {
                color = Color(255, 100, 100),
                display = true,
                faction = FACTION_COMBINE,
                trespassing = true
            }
        )
    end)
end)

-- Notify players when entering faction zones
function PLUGIN:OnPlayerAreaChanged(client, oldID, newID)
    local area = ix.area.stored[newID]
    local character = client:GetCharacter()

    if area and area.properties.faction and area.properties.faction > 0 then
        if character:GetFaction() != area.properties.faction then
            if area.properties.trespassing then
                client:Notify("WARNING: You are trespassing in restricted territory!")
            end
        else
            client:Notify("Welcome to your faction's territory.")
        end
    end
end
```

### Example 3: Dynamic Event Zones

```lua
-- Create temporary event zone
function CreateEventZone(position, radius, duration)
    local min = position - Vector(radius, radius, radius)
    local max = position + Vector(radius, radius, radius)

    ix.area.Create(
        "event_zone_" .. os.time(),
        "event",
        min,
        max,
        false,
        {
            color = Color(255, 215, 0),
            display = true
        }
    )

    -- Remove after duration
    timer.Simple(duration, function()
        ix.area.Remove("event_zone_" .. os.time())
    end)
end

-- Usage
CreateEventZone(Vector(0, 0, 100), 500, 600)  -- 500 unit radius, 10 minute duration
```

## Best Practices

### ✅ DO

- Use descriptive area names (e.g., "spawn_zone_01" not "area1")
- Create areas on server start for permanent zones
- Use area types to categorize zones
- Set appropriate colors for visual distinction
- Test area bounds in-game with editor
- Use OnPlayerAreaChanged for area-specific logic
- Clean up temporary areas when no longer needed

### ❌ DON'T

- Don't create hundreds of overlapping areas (performance)
- Don't use areas for tiny zones (< 50 units)
- Don't forget to handle players leaving areas
- Don't modify ix.area.stored directly
- Don't create areas with same name
- Don't forget bNoReplicate can cause client/server desync

## Common Patterns

### Pattern 1: Using Area Editor

```lua
1. Type /AreaEdit in chat
2. Look at first corner position
3. Left click to set first corner
4. Look at opposite corner
5. Left click again to complete area
6. Enter area name in dialog
7. Configure properties
8. Area is created and saved
```

### Pattern 2: Removing with Editor

```lua
1. Type /AreaEdit in chat
2. Look at the area you want to remove
3. Right click
4. Area is removed and saved
```

### Pattern 3: Checking Current Area

```lua
-- In any hook
function PLUGIN:PlayerSay(client, text)
    local areaID = client:GetArea()

    if areaID == "library" then
        client:Notify("Please keep quiet in the library!")
    end
end
```

## Common Issues

### Area Not Detecting Players

**Cause**: Area bounds incorrect or player not fully inside
**Fix**:
- Use editor to visualize area bounds
- Ensure area is tall enough to cover player
- Check player is at expected position

```lua
-- Debug: Print player position and area
local pos = client:GetPos() + client:OBBCenter()
print("Player at: " .. tostring(pos))
print("In area: " .. tostring(client:GetArea()))
print("Actually in area: " .. tostring(client:IsInArea()))
```

### OnPlayerAreaChanged Not Firing

**Cause**: Player moving too fast or tick rate too low
**Fix**: Adjust area tick time configuration

```lua
-- In server console or config
ix_config_areaTickTime "0.5"  -- Check twice per second instead of once
```

### Area Not Showing on Client

**Cause**: Display property set to false or area not networked
**Fix**:
- Check area properties
- Ensure bNoReplicate is false
- Verify client received area data

```lua
-- Client-side debug
PrintTable(ix.area.stored)  -- Should show all areas
```

### Area Names Conflicting

**Cause**: Creating area with existing name
**Fix**: Use unique names

```lua
-- Bad: Generic names
ix.area.Create("zone", ...)

-- Good: Descriptive unique names
ix.area.Create("nexus_spawn_zone_01", ...)
```

## Technical Details

### Area Tracking System

**Reference**: `plugins/area/sv_hooks.lua:40-71`

The AreaThink function runs on a timer:
- Default: Every 1 second (configurable)
- Checks all alive players with characters
- Tests if player's position (including OBBCenter) is within any area bounds
- Updates `client.ixArea` with current/last area
- Updates `client.ixInArea` with current status
- Fires `OnPlayerAreaChanged` hook when area changes

### Network Protocol

**Reference**: `plugins/area/sv_plugin.lua:1-8`

Five network messages:
- `ixAreaSync`: Full area list sent to connecting clients (compressed JSON)
- `ixAreaAdd`: Notifies clients when area is created
- `ixAreaRemove`: Notifies clients when area is removed
- `ixAreaChanged`: Notifies player when their area changes
- `ixAreaEditStart`: Enters area edit mode
- `ixAreaEditEnd`: Exits area edit mode

### Data Storage

**Reference**: `plugins/area/sv_hooks.lua:2-13`

Areas are stored using Helix's data system:
- Saved to plugin data on server shutdown
- Loaded on server start
- Format: `{[name] = {type, startPosition, endPosition, properties}}`

### Position Calculation

**Reference**: `plugins/area/sh_plugin.lua:55-61`

Helper function converts two world positions to:
- Center point (midpoint between corners)
- Local min (relative to center)
- Local max (relative to center)

Used for area editor and proper bound calculation.

## Configuration

### areaTickTime

**Reference**: `plugins/area/sh_plugin.lua:13-26`

- **Type**: Number
- **Default**: 1 second
- **Range**: 0.1 to 4 seconds
- **Description**: How often to check player positions
- **Console**: `ix_config_areaTickTime`

Lower values = more responsive but higher CPU usage

## Logging

**Reference**: `plugins/area/sv_plugin.lua:10-16`

Two log types:
- `areaAdd`: When admin creates an area
- `areaRemove`: When admin removes an area

Both include admin name and area name.

## Permissions

**Required Privilege**: `Helix - AreaEdit`

Required for:
- Using /AreaEdit command
- Creating areas via editor
- Removing areas via editor
- Creating areas via network messages

## See Also

- [Spawns Plugin](spawns.md) - Define spawn points in areas
- [Doors Plugin](doors.md) - Lock doors based on areas
- [Recognition Plugin](recognition.md) - Area-based player recognition
- [Plugin System](plugin-system.md) - Understanding Helix plugins
- Source: `plugins/area/`
