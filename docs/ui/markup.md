# Markup System (ix.markup)

> **Reference**: `gamemode/core/libs/cl_markup.lua`

The markup system provides pseudo-HTML text rendering with support for colors, fonts, images, and automatic text wrapping. It's used extensively in the chatbox and 3D text displays.

## ⚠️ Important: Use Built-in Markup Functions

**Always use ix.markup.Parse()** rather than manually drawing formatted text. The framework provides:
- Automatic text wrapping to specified widths
- HTML-like tag parsing for fonts and colors
- Support for embedded images/materials
- UTF-8 text handling
- Newline (`\n`) and tab (`\t`) character support
- Proper text sizing and measurement

## Core Concepts

### What is the Markup System?

The markup system parses pseudo-HTML strings into renderable markup objects that can be drawn with proper formatting, colors, and fonts. It's similar to Garry's Mod's built-in markup but integrated with Helix's systems.

### Key Terms

- **Markup Object** - Parsed representation of formatted text ready to render
- **Tags** - HTML-like syntax for formatting (`<color>`, `<font>`, `<img>`)
- **Wrapping** - Automatic line breaking at specified widths
- **Color Stack** - Nested color context for hierarchical formatting
- **Font Stack** - Nested font context for hierarchical formatting

## Using Markup

### ix.markup.Parse()

**Reference**: `gamemode/core/libs/cl_markup.lua:263`

Parses pseudo-HTML markup into a drawable markup object.

```lua
ix.markup.Parse(text, maxWidth)
```

**Parameters**:
- `text` (string) - Text with markup tags
- `maxWidth` (number, optional) - Maximum width before wrapping

**Returns**: MarkupObject - Object with drawing and sizing methods

```lua
-- Basic text
local markup = ix.markup.Parse("Hello World")

-- With color
local markup = ix.markup.Parse("<color=255,0,0>Red Text</color>")

-- With font
local markup = ix.markup.Parse("<font=ixBigFont>Large Text</font>")

-- With wrapping
local markup = ix.markup.Parse("Long text that will wrap", 200)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually draw formatted text
surface.SetFont("Font1")
surface.DrawText("Text")
surface.SetFont("Font2")
surface.DrawText("More text")
-- This is tedious and doesn't handle wrapping!

-- Use ix.markup.Parse() instead
local markup = ix.markup.Parse("<font=Font1>Text</font> <font=Font2>More text</font>")
markup:draw(x, y)
```

### MarkupObject:draw()

**Reference**: `gamemode/core/libs/cl_markup.lua:206`

Draws the markup to the screen.

```lua
markupObject:draw(x, y, halign, valign, alphaOverride)
```

**Parameters**:
- `x` (number) - X position
- `y` (number) - Y position
- `halign` (number, optional) - Horizontal alignment (TEXT_ALIGN_LEFT/CENTER/RIGHT)
- `valign` (number, optional) - Vertical alignment (TEXT_ALIGN_TOP/CENTER/BOTTOM)
- `alphaOverride` (number, optional) - Override alpha value (0-255)

```lua
-- Draw at position
markup:draw(100, 50)

-- Draw centered
markup:draw(ScrW() / 2, ScrH() / 2, TEXT_ALIGN_CENTER, TEXT_ALIGN_CENTER)

-- Draw with transparency
markup:draw(100, 50, TEXT_ALIGN_LEFT, TEXT_ALIGN_TOP, 128)
```

### MarkupObject:GetWidth()

**Reference**: `gamemode/core/libs/cl_markup.lua:181`

Returns the total width of the markup.

```lua
local width = markupObject:GetWidth()
```

### MarkupObject:GetHeight()

**Reference**: `gamemode/core/libs/cl_markup.lua:190`

Returns the total height of the markup.

```lua
local height = markupObject:GetHeight()
```

### MarkupObject:size()

**Reference**: `gamemode/core/libs/cl_markup.lua:194`

Returns both width and height.

```lua
local width, height = markupObject:size()
```

## Markup Tags

### Color Tag

**Reference**: `gamemode/core/libs/cl_markup.lua:79-93`

Changes text color using RGB(A) values or named colors.

```lua
-- RGB format
local markup = ix.markup.Parse("<color=255,0,0>Red text</color>")

-- RGBA format (with alpha)
local markup = ix.markup.Parse("<color=255,0,0,128>Transparent red</color>")

-- Named colors
local markup = ix.markup.Parse("<color=red>Red</color> <color=blue>Blue</color>")

-- Nested colors
local markup = ix.markup.Parse("<color=red>Red <color=blue>Blue</color> Red again</color>")
```

**Available Named Colors**:
- Basic: `black`, `white`, `grey`/`gray`, `red`, `green`, `blue`, `yellow`, `purple`, `cyan`/`turq`
- Dark: `dkgrey`, `dkred`, `dkgreen`, `dkblue`, `dkyellow`, `dkpurple`, `dkcyan`
- Light: `ltgrey`, `ltred`, `ltgreen`, `ltblue`, `ltyellow`, `ltpurple`, `ltcyan`

### Font Tag

**Reference**: `gamemode/core/libs/cl_markup.lua:95-97`

Changes text font.

```lua
-- Change font
local markup = ix.markup.Parse("<font=ixBigFont>Large Text</font>")

-- Nested fonts
local markup = ix.markup.Parse("<font=ixMediumFont>Medium <font=ixBigFont>Big</font> Medium</font>")

-- Multiple fonts
local markup = ix.markup.Parse("<font=ixSmallFont>Small</font> <font=ixMediumFont>Medium</font>")
```

### Image Tag

**Reference**: `gamemode/core/libs/cl_markup.lua:98-121`

Embeds a material/image in the text.

```lua
-- Basic image (16x16 default)
local markup = ix.markup.Parse("<img=vgui/icon.png> Icon text")

-- Custom size
local markup = ix.markup.Parse("<img=vgui/icon.png,32x32> Larger icon")

-- Without extension (auto-detected)
local markup = ix.markup.Parse("<img=vgui/icon,24x24> Icon")
```

## Special Characters

### Newlines and Tabs

```lua
-- Newline character
local markup = ix.markup.Parse("Line 1\nLine 2\nLine 3")

-- Tab character
local markup = ix.markup.Parse("Column1\tColumn2\tColumn3")

-- Combined
local markup = ix.markup.Parse("Header\n\tIndented line\n\tAnother indent")
```

### HTML Entities

**Reference**: `gamemode/core/libs/cl_markup.lua:295-297`

```lua
-- Greater than: &gt; becomes >
local markup = ix.markup.Parse("5 &gt; 3")

-- Less than: &lt; becomes <
local markup = ix.markup.Parse("3 &lt; 5")

-- Ampersand: &amp; becomes &
local markup = ix.markup.Parse("You &amp; Me")
```

## Custom Drawing Callback

### onDrawText Callback

**Reference**: `gamemode/core/libs/cl_markup.lua:233-234`

Override how text is drawn for custom effects.

```lua
local markup = ix.markup.Parse("<color=255,255,255>Text with shadow</color>")

-- Custom drawing function
markup.onDrawText = function(text, font, x, y, color, halign, valign, alpha, block)
    -- Draw shadow
    surface.SetFont(font)
    surface.SetTextColor(0, 0, 0, alpha or 255)
    surface.SetTextPos(x + 2, y + 2)
    surface.DrawText(text)

    -- Draw main text
    surface.SetTextColor(color.r, color.g, color.b, alpha or color.a)
    surface.SetTextPos(x, y)
    surface.DrawText(text)
end

markup:draw(100, 100)
```

## Complete Example: Chat Message

```lua
-- CLIENT-SIDE: Format chat message with colors and fonts
function FormatChatMessage(speaker, text)
    local speakerColor = team.GetColor(speaker:Team())
    local r, g, b = speakerColor.r, speakerColor.g, speakerColor.b

    -- Build markup string
    local markup = string.format(
        "<color=%d,%d,%d><font=ixMediumFont>%s</font></color>: %s",
        r, g, b,
        speaker:Name(),
        text
    )

    -- Parse with wrapping
    local parsed = ix.markup.Parse(markup, ScrW() * 0.4)

    return parsed
end

-- Usage in HUD
hook.Add("HUDPaint", "DrawChatMessages", function()
    local y = ScrH() - 200

    for i, msg in ipairs(chatMessages) do
        msg.markup:draw(10, y, TEXT_ALIGN_LEFT, TEXT_ALIGN_TOP)
        y = y - msg.markup:GetHeight() - 5
    end
end)
```

## Complete Example: 3D Text with Shadow

```lua
-- CLIENT-SIDE: Create 3D text sign
function Create3DTextSign(text, position, angle)
    local markup = ix.markup.Parse("<font=ix3D2DFont>" .. text:gsub("\\n", "\n"))

    -- Custom shadow drawing
    markup.onDrawText = function(surfaceText, font, x, y, color, alignX, alignY, alpha)
        -- Shadow
        surface.SetTextPos(x + 1, y + 1)
        surface.SetTextColor(0, 0, 0, alpha or 255)
        surface.SetFont(font)
        surface.DrawText(surfaceText)

        -- Main text
        surface.SetTextPos(x, y)
        surface.SetTextColor(color.r, color.g, color.b, alpha or color.a)
        surface.DrawText(surfaceText)
    end

    return {
        markup = markup,
        pos = position,
        ang = angle
    }
end

-- Draw 3D text
local sign = Create3DTextSign("Welcome\nto the\nserver!", Vector(0, 0, 100), Angle(0, 0, 0))

hook.Add("PostDrawTranslucentRenderables", "Draw3DSign", function()
    cam.Start3D2D(sign.pos, sign.ang, 0.1)
        sign.markup:draw(0, 0, TEXT_ALIGN_CENTER, TEXT_ALIGN_CENTER)
    cam.End3D2D()
end)
```

## Complete Example: Wrapped Description Panel

```lua
-- CLIENT-SIDE: Create item description with word wrap
function CreateDescriptionPanel(item)
    local frame = vgui.Create("DFrame")
    frame:SetSize(400, 300)
    frame:Center()
    frame:SetTitle(item:GetName())
    frame:MakePopup()

    local panel = frame:Add("DPanel")
    panel:Dock(FILL)

    -- Parse description with wrapping
    local description = ix.markup.Parse(
        "<font=ixMediumFont><color=white>" .. item:GetDescription() .. "</color></font>",
        380  -- Wrap at panel width minus padding
    )

    panel.Paint = function(self, w, h)
        surface.SetDrawColor(50, 50, 50, 255)
        surface.DrawRect(0, 0, w, h)

        -- Draw wrapped description
        description:draw(10, 10)
    end

    -- Resize panel to fit content
    panel:SetTall(description:GetHeight() + 20)

    return frame
end
```

## Best Practices

### ✅ DO

- Use `ix.markup.Parse()` for formatted text
- Specify `maxWidth` for automatic wrapping
- Use named colors for common colors
- Close all tags properly (`</color>`, `</font>`)
- Store parsed markup for reuse (don't reparse every frame)
- Use `onDrawText` callback for custom effects
- Check dimensions with `GetWidth()` and `GetHeight()`
- Use UTF-8 characters safely

### ❌ DON'T

- Don't parse markup every frame (cache it)
- Don't forget to close tags
- Don't use very wide text without wrapping
- Don't forget `<` and `>` need `&lt;` and `&gt;`
- Don't use invalid color names
- Don't use non-existent fonts
- Don't draw outside Paint hooks (use manual rendering)

## Common Patterns

### Pattern 1: Colored Player Name

```lua
-- Format player name with faction color
function GetColoredName(player)
    local color = team.GetColor(player:Team())
    return ix.markup.Parse(string.format(
        "<color=%d,%d,%d>%s</color>",
        color.r, color.g, color.b,
        player:Name()
    ))
end
```

### Pattern 2: Multi-line Text Box

```lua
-- Create wrapped text display
function CreateTextBox(container, text, width)
    local markup = ix.markup.Parse(text, width)

    local panel = container:Add("DPanel")
    panel:SetSize(width, markup:GetHeight())
    panel.Paint = function(self, w, h)
        markup:draw(0, 0)
    end

    return panel
end
```

### Pattern 3: Animated Fade

```lua
-- Create fading markup
local markup = ix.markup.Parse("<color=white>Fading text</color>")
local alpha = 255
local startTime = CurTime()

hook.Add("HUDPaint", "FadeText", function()
    local elapsed = CurTime() - startTime
    alpha = math.max(0, 255 - elapsed * 50)

    if alpha > 0 then
        markup:draw(100, 100, TEXT_ALIGN_LEFT, TEXT_ALIGN_TOP, alpha)
    else
        hook.Remove("HUDPaint", "FadeText")
    end
end)
```

## Common Issues

### Tags Not Closing Properly

**Cause**: Forgot closing tag or mismatched tags
**Fix**: Always close tags in proper order

```lua
-- WRONG
local markup = ix.markup.Parse("<color=red>Text <font=Big>Large</color></font>")

-- CORRECT
local markup = ix.markup.Parse("<color=red>Text <font=Big>Large</font></color>")
```

### Text Not Wrapping

**Cause**: Forgot to specify maxWidth parameter
**Fix**: Always provide width for wrapping

```lua
-- WRONG (no wrapping)
local markup = ix.markup.Parse("Very long text that should wrap...")

-- CORRECT (wraps at 300 units)
local markup = ix.markup.Parse("Very long text that should wrap...", 300)
```

### Performance Issues

**Cause**: Parsing markup every frame
**Fix**: Cache parsed markup and reuse

```lua
-- WRONG
hook.Add("HUDPaint", "Draw", function()
    local markup = ix.markup.Parse("<color=red>Text</color>")  -- Parsing every frame!
    markup:draw(10, 10)
end)

-- CORRECT
local markup = ix.markup.Parse("<color=red>Text</color>")  -- Parse once
hook.Add("HUDPaint", "Draw", function()
    markup:draw(10, 10)  -- Reuse parsed markup
end)
```

### Colors Not Working

**Cause**: Invalid color name or format
**Fix**: Use valid RGB values or named colors

```lua
-- WRONG
local markup = ix.markup.Parse("<color=redd>Text</color>")  -- Typo!
local markup = ix.markup.Parse("<color=255>Text</color>")   -- Need RGB!

-- CORRECT
local markup = ix.markup.Parse("<color=red>Text</color>")
local markup = ix.markup.Parse("<color=255,0,0>Text</color>")
```

## See Also

- [Chat System](../systems/chat.md) - Uses markup for formatted messages
- [Derma Overview](derma-overview.md) - UI panels that can display markup
- [HUD System](hud.md) - Drawing on-screen elements
- [3D Text Plugin](../plugins/3dtext.md) - 3D world text using markup
- Source: `gamemode/core/libs/cl_markup.lua`
- Usage: `plugins/chatbox/derma/cl_chatbox.lua` (lines 46, 68)
- Usage: `plugins/3dtext.lua` (lines 123, 190)
