# 3D Panels Plugin

> **Reference**: `plugins/3dpanel.lua`

The 3D Panels plugin allows administrators to place 2D image panels in 3D space on the map. These panels display images from URLs and are synchronized across all clients, making them perfect for displaying posters, signs, artwork, or information boards in your gameworld.

## ⚠️ Important: Use Built-in Panel System

**Always use the built-in PanelAdd/PanelRemove commands** rather than creating custom image display systems. The framework provides:
- Automatic network synchronization to all clients
- Persistent storage of panel data
- Image caching for performance
- Live preview while placing panels
- Distance-based rendering optimization

## Core Concepts

### What are 3D Panels?

3D Panels are 2D images positioned in 3D world space. They:
- Display images from direct image URLs (PNG, JPG, etc.)
- Persist across server restarts
- Automatically sync to connecting players
- Render only when players are nearby (within ~2048 units)
- Support custom scale and brightness settings

### Key Features

- **Automatic Caching**: Images are downloaded once and cached locally
- **Network Sync**: All panels automatically sync to clients on join
- **Live Preview**: See exactly where the panel will appear while typing the command
- **Optimized Rendering**: Panels only render when players are within range
- **Persistent Storage**: All panels save automatically via Helix data system

## Using 3D Panels

### Adding a Panel

**Reference**: `plugins/3dpanel.lua:351-372`

```lua
-- Command usage (in chat):
/PanelAdd <url> [scale] [brightness]

-- Examples:
/PanelAdd https://example.com/poster.png
/PanelAdd https://example.com/sign.jpg 1.5
/PanelAdd https://example.com/art.png 2 80
```

**Parameters**:
- `url` (required): Direct link to image file (must be PNG, JPG, etc.)
- `scale` (optional): Size multiplier (default: 1.0, clamped to 0.001-5.0)
- `brightness` (optional): Brightness percentage (default: 100, clamped to 1-255)

**How it works**:
1. Look at the surface where you want to place the panel
2. Type the command with the image URL
3. See a live preview as you type
4. Press enter to place the panel
5. Panel automatically saves and syncs to all clients

**Placement**:
- The panel appears where your crosshair is looking
- It automatically aligns perpendicular to the surface
- Position is offset slightly from the surface to prevent z-fighting

### Removing Panels

**Reference**: `plugins/3dpanel.lua:374-388`

```lua
-- Command usage (in chat):
/PanelRemove [radius]

-- Examples:
/PanelRemove          -- Removes panels within 100 units
/PanelRemove 200      -- Removes panels within 200 units
/PanelRemove 50       -- Removes panels within 50 units
```

**Parameters**:
- `radius` (optional): Distance in units to search for panels (default: 100)

**How it works**:
1. Look near the panel(s) you want to remove
2. Use /PanelRemove with optional radius
3. All panels within radius of your crosshair position are deleted
4. Returns the number of panels removed

### Programmatic Access

**Reference**: `plugins/3dpanel.lua:34-56`

```lua
-- Add a panel programmatically (server-side)
function PLUGIN:AddPanel(position, angles, url, scale, brightness)
    -- Example usage:
    local pos = Vector(0, 0, 100)
    local ang = Angle(0, 90, 90)
    local url = "https://example.com/image.png"

    PLUGIN:AddPanel(pos, ang, url, 1.0, 100)
end
```

**Reference**: `plugins/3dpanel.lua:59-97`

```lua
-- Remove panels near a position (server-side)
function PLUGIN:RemovePanel(position, radius)
    -- Example usage:
    local pos = Vector(0, 0, 100)
    local deleted = PLUGIN:RemovePanel(pos, 150)
    print("Removed " .. deleted .. " panels")
end
```

## Complete Example

```lua
-- Add a welcome poster in your schema
hook.Add("InitPostEntity", "AddWelcomePoster", function()
    timer.Simple(2, function()
        local plugin = ix.plugin.list["3dpanel"]

        if plugin then
            local pos = Vector(100, 200, 64)
            local ang = Angle(0, 180, 90)

            plugin:AddPanel(
                pos,
                ang,
                "https://example.com/welcome-poster.png",
                2.0,  -- 2x scale
                100   -- Full brightness
            )
        end
    end)
end)
```

## Best Practices

### ✅ DO

- Use direct image URLs (ending in .png, .jpg, etc.)
- Use HTTPS URLs when possible
- Test image URLs in browser first
- Keep panel count reasonable (< 50 per map)
- Use appropriate scale for readability
- Place panels on flat surfaces for best results
- Use the live preview to verify placement

### ❌ DON'T

- Don't use HTML page URLs (must be direct image links)
- Don't use extremely large images (> 2048x2048)
- Don't place hundreds of panels (impacts performance)
- Don't rely on panels for critical gameplay information
- Don't use panels for animated content (only static images)
- Don't forget to test URLs before placing

## Common Patterns

### Pattern 1: Adding Informational Signs

```lua
-- Place a rules sign at spawn
/PanelAdd https://yourserver.com/images/rules.png 1.5
```

### Pattern 2: Creating an Art Gallery

```lua
-- Place multiple art pieces on walls
/PanelAdd https://example.com/art1.png 1.0
-- Move to next wall
/PanelAdd https://example.com/art2.png 1.0
-- Continue for each piece
```

### Pattern 3: Cleaning Up Old Panels

```lua
-- Remove all panels in a large area
/PanelRemove 500

-- Or remove specific panel with small radius
/PanelRemove 10
```

## Common Issues

### Panel Not Showing

**Cause**: Invalid URL, HTTPS blocked, or wrong file type
**Fix**:
- Verify URL works in browser
- Ensure URL points directly to image file
- Check if server allows HTTPS connections
- Use PNG or JPG format

```lua
-- Incorrect:
/PanelAdd https://example.com/page-with-image.html

-- Correct:
/PanelAdd https://example.com/direct-image.png
```

### Panel Too Small/Large

**Cause**: Incorrect scale parameter
**Fix**: Adjust scale parameter (0.001 to 5.0 range)

```lua
-- Too small
/PanelAdd https://example.com/image.png 0.1

-- Better
/PanelAdd https://example.com/image.png 1.5
```

### Panel Not Aligned Properly

**Cause**: Surface angle or placement position
**Fix**:
- Look directly at flat surface
- Panel aligns perpendicular to surface normal
- Move closer/farther for better angle

### Can't Remove Panel

**Cause**: Looking at wrong location or radius too small
**Fix**: Increase radius parameter

```lua
-- Increase search radius
/PanelRemove 300
```

## Technical Details

### Data Storage

**Reference**: `plugins/3dpanel.lua:100-110`

Panels are stored using Helix's data system:
- Automatically saves when panels are added/removed
- Loads on server start
- Persists across restarts
- Stored in plugin data file

### Network Protocol

**Reference**: `plugins/3dpanel.lua:12-30`

Three network messages:
- `ixPanelList`: Full panel list sent to connecting clients
- `ixPanelAdd`: Notifies clients when panel is added
- `ixPanelRemove`: Notifies clients when panel is removed

### Image Caching

**Reference**: `plugins/3dpanel.lua:118-155`

Client-side caching system:
- Downloads images on demand
- Stores in `data/helix/[schema]/3dpanel/`
- Reuses cached images on reconnect
- Validates materials before rendering

### Rendering Optimization

**Reference**: `plugins/3dpanel.lua:261-295`

Performance optimizations:
- Distance check: 4,194,304 squared units (~2048 units)
- Anisotropic filtering for quality
- Only renders visible panels
- Brightness control via surface color

## Permissions

**Required Privilege**: `Manage Panels`

This privilege is required to use PanelAdd and PanelRemove commands. Configure in your admin settings.

## Configuration

No configuration file needed. All settings are per-panel via command parameters.

## See Also

- [3D Text Plugin](3dtext.md) - For placing text instead of images
- [Plugin System](plugin-system.md) - Understanding Helix plugins
- [Commands System](../systems/commands.md) - Creating custom commands
- Source: `plugins/3dpanel.lua`
