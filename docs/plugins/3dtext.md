# 3D Text Plugin

> **Reference**: `plugins/3dtext.lua`

The 3D Text plugin allows administrators to place text labels in 3D space on the map. These text displays are synchronized across all clients and persist across server restarts, making them perfect for signs, labels, instructions, or decorative text in your gameworld.

## ⚠️ Important: Use Built-in Text System

**Always use the built-in TextAdd/TextRemove commands** rather than creating custom 3D text systems. The framework provides:
- Automatic network synchronization to all clients
- Persistent storage of text data
- Beautiful markup rendering with shadows
- Live preview while placing text
- Distance-based alpha fading
- Undo support for accidentally placed text
- Visual removal preview showing what will be deleted

## Core Concepts

### What is 3D Text?

3D Text displays are rendered text positioned in 3D world space. They:
- Display any text string with newline support
- Persist across server restarts
- Automatically sync to connecting players
- Render with distance-based fading (within ~1024 units)
- Support custom scale settings
- Include text shadows for readability
- Can be undone with GMod's undo system

### Key Features

- **Markup Rendering**: Uses Helix markup system for rich text rendering
- **Text Shadows**: Automatic shadows for better visibility
- **Network Sync**: All text automatically syncs to clients on join
- **Live Preview**: See exactly where text will appear while typing command
- **Undo Support**: Use GMod undo to remove last placed text
- **Distance Fading**: Text fades out as you move away
- **Persistent Storage**: All text saves automatically via Helix data system
- **Newline Support**: Use `\n` in text for multi-line displays

## Using 3D Text

### Adding Text

**Reference**: `plugins/3dtext.lua:301-328`

```lua
-- Command usage (in chat):
/TextAdd <text> [scale]

-- Examples:
/TextAdd "Welcome to the City"
/TextAdd "No Entry" 1.5
/TextAdd "Office Hours:\n9 AM - 5 PM" 1.2
```

**Parameters**:
- `text` (required): The text to display (use `\n` for newlines)
- `scale` (optional): Size multiplier (default: 1.0, clamped to 0.001-5.0)

**How it works**:
1. Look at the surface where you want to place the text
2. Type the command with your text
3. See a live preview as you type
4. Press enter to place the text
5. Text automatically saves and syncs to all clients
6. Use GMod undo (`Z` key) to remove if needed

**Placement**:
- Text appears where your crosshair is looking
- Automatically aligns perpendicular to the surface
- Position is offset slightly from surface to prevent z-fighting
- Text is centered at the placement point

### Removing Text

**Reference**: `plugins/3dtext.lua:330-341`

```lua
-- Command usage (in chat):
/TextRemove [radius]

-- Examples:
/TextRemove          -- Removes text within 100 units
/TextRemove 200      -- Removes text within 200 units
/TextRemove 50       -- Removes text within 50 units
```

**Parameters**:
- `radius` (optional): Distance in units to search for text (default: 100)

**How it works**:
1. Look near the text you want to remove
2. Type /TextRemove with optional radius
3. Red lines show from screen center to all text that will be deleted
4. Counter shows how many will be removed
5. Press enter to confirm removal
6. Returns the number of text objects removed

**Visual Feedback**:

**Reference**: `plugins/3dtext.lua:215-251`

When typing the TextRemove command, you'll see:
- Red lines from crosshair to each text in range
- Counter showing how many will be deleted
- Real-time update as you change radius parameter

### Programmatic Access

**Reference**: `plugins/3dtext.lua:37-53`

```lua
-- Add text programmatically (server-side)
function PLUGIN:AddText(position, angles, text, scale)
    -- Example usage:
    local pos = Vector(0, 0, 100)
    local ang = Angle(0, 90, 90)
    local text = "Information Board"

    local index = PLUGIN:AddText(pos, ang, text, 1.5)
    print("Added text with ID: " .. index)
    return index
end
```

**Reference**: `plugins/3dtext.lua:56-87`

```lua
-- Remove text near a position (server-side)
function PLUGIN:RemoveText(position, radius)
    -- Example usage:
    local pos = Vector(0, 0, 100)
    local deleted = PLUGIN:RemoveText(pos, 150)
    print("Removed " .. deleted .. " text objects")
end
```

**Reference**: `plugins/3dtext.lua:89-102`

```lua
-- Remove specific text by ID (server-side)
function PLUGIN:RemoveTextByID(id)
    -- Returns true if text existed and was removed
    local success = PLUGIN:RemoveTextByID(5)
    if success then
        print("Text removed successfully")
    end
end
```

## Complete Example

```lua
-- Add directional signs in your schema
hook.Add("InitPostEntity", "AddDirectionalSigns", function()
    timer.Simple(2, function()
        local plugin = ix.plugin.list["3dtext"]

        if plugin then
            -- Main entrance sign
            plugin:AddText(
                Vector(100, 200, 80),
                Angle(0, 180, 90),
                "City Hall\nEnter Here",
                2.0  -- 2x scale
            )

            -- Small room label
            plugin:AddText(
                Vector(50, 100, 70),
                Angle(0, 90, 90),
                "Storage",
                0.8  -- Smaller scale
            )
        end
    end)
end)
```

## Best Practices

### ✅ DO

- Use `\n` for multi-line text (e.g., "Line 1\nLine 2")
- Keep text concise and readable
- Use appropriate scale for viewing distance
- Place text on flat surfaces for best alignment
- Use the live preview to verify placement
- Use GMod undo for quick corrections
- Test readability from expected viewing distance
- Use shadows are automatic and improve visibility

### ❌ DON'T

- Don't create extremely long text strings (performance impact)
- Don't place hundreds of text objects (< 50 per map recommended)
- Don't use tiny scale for important information
- Don't forget to check visibility from multiple angles
- Don't rely on text for critical real-time information
- Don't place text in areas with heavy visual clutter

## Common Patterns

### Pattern 1: Multi-Line Signs

```lua
-- Create a detailed information sign
/TextAdd "Reception\n\nHours: 9 AM - 5 PM\nKnock if closed" 1.5
```

### Pattern 2: Room Labels

```lua
-- Label different rooms
/TextAdd "Storage Room" 1.0
-- Move to next room
/TextAdd "Office" 1.0
-- Continue for each room
```

### Pattern 3: Quick Corrections with Undo

```lua
-- Place text
/TextAdd "Office"
-- Oops, wrong position!
-- Press Z to undo
-- Reposition and try again
/TextAdd "Office" 1.2
```

### Pattern 4: Cleaning Up Old Text

```lua
-- Remove all text in a large area
/TextRemove 500

-- Or remove specific text with small radius
/TextRemove 10
```

## Common Issues

### Text Not Showing

**Cause**: Too far away or invalid font
**Fix**:
- Move closer (render distance is ~1024 units)
- Check for font errors in console
- Verify text string is not empty

```lua
-- Incorrect (empty text):
/TextAdd ""

-- Correct:
/TextAdd "Sign Text"
```

### Text Too Small/Large

**Cause**: Incorrect scale parameter
**Fix**: Adjust scale parameter (0.001 to 5.0 range)

```lua
-- Too small
/TextAdd "Info" 0.1

-- Better
/TextAdd "Info" 1.5
```

### Text Not Aligned Properly

**Cause**: Surface angle or complex geometry
**Fix**:
- Look directly at flat surface
- Text aligns perpendicular to surface normal
- Use undo and try different angles
- Move closer/farther for better placement

### Newlines Not Working

**Cause**: Using literal newline instead of `\n`
**Fix**: Use `\n` escape sequence

```lua
-- Incorrect:
/TextAdd "Line 1
Line 2"

-- Correct:
/TextAdd "Line 1\nLine 2"
```

### Can't Remove Specific Text

**Cause**: Wrong location or radius too small
**Fix**:
- Look directly at the text
- Increase radius parameter
- Use visual feedback (red lines) to verify

```lua
-- Increase search radius
/TextRemove 300
```

## Technical Details

### Data Storage

**Reference**: `plugins/3dtext.lua:105-115`

Text objects are stored using Helix's data system:
- Automatically saves when text is added/removed
- Loads on server start via LoadData hook
- Persists across restarts
- Stored in plugin data file
- Legacy format support via table.ClearKeys

### Network Protocol

**Reference**: `plugins/3dtext.lua:12-34`

Three network messages:
- `ixTextList`: Full text list sent to connecting clients (compressed JSON)
- `ixTextAdd`: Notifies clients when text is added
- `ixTextRemove`: Notifies clients when text is removed

### Markup System

**Reference**: `plugins/3dtext.lua:122-139`

Text rendering uses Helix markup:
- Font: `ix3D2DFont`
- Custom text shadows (1 pixel offset)
- Supports color and formatting
- Newline character support (`\n`)
- Error handling for invalid markup

### Rendering Optimization

**Reference**: `plugins/3dtext.lua:253-298`

Performance optimizations:
- Distance check: 1,048,576 squared units (~1024 units max)
- Alpha fading starts at ~256 units (65536 squared)
- Fully visible within ~256 units
- Fully transparent beyond ~1024 units
- Only renders visible text objects
- Alpha calculation: `(1 - ((distance - 65536) / 768432)) * 255`

### Undo Integration

**Reference**: `plugins/3dtext.lua:317-324`

GMod undo system integration:
- Each placed text gets an undo entry
- Undo name: `ix3dText`
- Logs when text is undone
- Only affects text placed by that player
- Standard GMod undo key (`Z`) works

## Permissions

**Required**: Admin only (adminOnly = true)

Only administrators can use TextAdd and TextRemove commands. No separate privilege required.

## Configuration

No configuration file needed. All settings are per-text via command parameters.

## Logging

**Reference**: `plugins/3dtext.lua:16-18`

The plugin logs when players undo their 3D text:
- Log type: `undo3dText`
- Format: `"%s has removed their last 3D text."`
- Includes player name

## See Also

- [3D Panels Plugin](3dpanel.md) - For placing images instead of text
- [Markup System](../libraries/markup.md) - Text rendering system
- [Plugin System](plugin-system.md) - Understanding Helix plugins
- [Commands System](../systems/commands.md) - Creating custom commands
- Source: `plugins/3dtext.lua`
