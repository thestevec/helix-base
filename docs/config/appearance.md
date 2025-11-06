# Appearance Configuration

> **Reference**: `gamemode/config/sh_config.lua:18-133`

Configuration options for visual customization including colors, fonts, character menu settings, and UI elements.

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Get()` for appearance settings** rather than hardcoding visual elements. The framework provides:
- Centralized theme management
- Automatic UI updates when colors change
- Font loading and caching
- Client-side synchronization
- Admin-adjustable branding

## Theme Colors

### color

**Reference**: `gamemode/config/sh_config.lua:18`

```lua
ix.config.Add("color", Color(75, 119, 190, 255), "The main color theme for the framework.", function(oldValue, newValue)
    if (newValue.a != 255) then
        ix.config.Set("color", ColorAlpha(newValue, 255))
        return
    end

    if (CLIENT) then
        hook.Run("ColorSchemeChanged", newValue)
    end
end, {category = "appearance"})
```

**Type**: Color
**Default**: `Color(75, 119, 190, 255)` (Blue)
**Description**: Primary color theme used throughout the framework UI

**Callback**:
- Forces alpha to 255 (opaque)
- Triggers `ColorSchemeChanged` hook on clients
- Automatically updates all UI elements

**Usage**:
```lua
-- Get main theme color
local themeColor = ix.config.Get("color", Color(75, 119, 190, 255))

-- Use in custom UI
local panel = vgui.Create("DPanel")
panel:SetBackgroundColor(themeColor)

-- Respond to color changes
hook.Add("ColorSchemeChanged", "MyPlugin", function(newColor)
    -- Update custom UI elements
    UpdateCustomPanels(newColor)
end)
```

---

## Font Settings

### font

**Reference**: `gamemode/config/sh_config.lua:28`

```lua
ix.config.Add("font", "Roboto Th", "The font used to display titles.", function(oldValue, newValue)
    if (CLIENT) then
        hook.Run("LoadFonts", newValue, ix.config.Get("genericFont"))
    end
end, {category = "appearance"})
```

**Type**: String
**Default**: `"Roboto Th"`
**Description**: Font family used for titles and headers in the UI

**Callback**: Triggers `LoadFonts` hook to reload font definitions

**Usage**:
```lua
-- Get title font
local titleFont = ix.config.Get("font", "Roboto Th")

-- Create custom font using theme font
surface.CreateFont("MyTitleFont", {
    font = titleFont,
    size = 32,
    weight = 500
})
```

---

### genericFont

**Reference**: `gamemode/config/sh_config.lua:34`

```lua
ix.config.Add("genericFont", "Roboto", "The font used to display generic texts.", function(oldValue, newValue)
    if (CLIENT) then
        hook.Run("LoadFonts", ix.config.Get("font"), newValue)
    end
end, {category = "appearance"})
```

**Type**: String
**Default**: `"Roboto"`
**Description**: Font family used for body text and general UI elements

**Callback**: Triggers `LoadFonts` hook to reload font definitions

**Usage**:
```lua
-- Get body font
local bodyFont = ix.config.Get("genericFont", "Roboto")

-- Create custom font using theme font
surface.CreateFont("MyBodyFont", {
    font = bodyFont,
    size = 18,
    weight = 400
})
```

---

## Character Menu

### intro

**Reference**: `gamemode/config/sh_config.lua:118`

```lua
ix.config.Add("intro", true, "Whether or not the Helix intro is enabled for new players.", nil, {
    category = "appearance"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Shows the Helix intro cinematic to new players

**Usage**:
```lua
-- Check if intro should play
function PLUGIN:PlayerInitialSpawn(client)
    if ix.config.Get("intro", true) and client:IsNewPlayer() then
        -- Show intro
        ShowIntro(client)
    end
end
```

---

### music

**Reference**: `gamemode/config/sh_config.lua:121`

```lua
ix.config.Add("music", "music/hl2_song2.mp3", "The default music played in the character menu.", nil, {
    category = "appearance"
})
```

**Type**: String (File path)
**Default**: `"music/hl2_song2.mp3"`
**Description**: Sound file path for character selection menu background music

**Usage**:
```lua
-- Play configured menu music
function PLUGIN:CharacterMenuMusicStart()
    local musicPath = ix.config.Get("music", "music/hl2_song2.mp3")

    if musicPath and musicPath != "" then
        surface.PlaySound(musicPath)
    end
end
```

---

### communityURL

**Reference**: `gamemode/config/sh_config.lua:124`

```lua
ix.config.Add("communityURL", "https://nebulous.cloud/", "The URL to navigate to when the community button is clicked.", nil, {
    category = "appearance"
})
```

**Type**: String (URL)
**Default**: `"https://nebulous.cloud/"`
**Description**: Website URL opened when clicking community button in character menu

**Usage**:
```lua
-- Open community URL when button clicked
function PLUGIN:OnCommunityButtonPressed()
    local url = ix.config.Get("communityURL", "https://nebulous.cloud/")

    if url and url != "" then
        gui.OpenURL(url)
    end
end
```

---

### communityText

**Reference**: `gamemode/config/sh_config.lua:127`

```lua
ix.config.Add("communityText", "@community",
    "The text to display on the community button. You can use language phrases by prefixing with @", nil, {
    category = "appearance"
})
```

**Type**: String
**Default**: `"@community"` (uses language phrase)
**Description**: Text displayed on community button. Prefix with `@` to use language phrases

**Usage**:
```lua
-- Get button text (supports localization)
local buttonText = ix.config.Get("communityText", "@community")

if buttonText:sub(1, 1) == "@" then
    -- Use language phrase
    buttonText = L(buttonText:sub(2))
end

button:SetText(buttonText)
```

---

### vignette

**Reference**: `gamemode/config/sh_config.lua:131`

```lua
ix.config.Add("vignette", true, "Whether or not the vignette is shown.", nil, {
    category = "appearance"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Enables or disables screen vignette effect (darkened edges)

**Usage**:
```lua
-- Draw vignette if enabled
hook.Add("HUDPaint", "DrawVignette", function()
    if ix.config.Get("vignette", true) then
        DrawVignetteEffect()
    end
end)
```

---

## Complete Example: Themed UI Panel

```lua
-- Create panel that respects theme configuration
local PANEL = {}

function PANEL:Init()
    self:LoadTheme()

    -- Update when theme changes
    hook.Add("ColorSchemeChanged", self, function(panel, newColor)
        panel:LoadTheme()
    end)
end

function PANEL:LoadTheme()
    -- Get theme colors
    self.themeColor = ix.config.Get("color", Color(75, 119, 190, 255))

    -- Derive other colors from theme
    self.headerColor = Color(
        self.themeColor.r * 0.8,
        self.themeColor.g * 0.8,
        self.themeColor.b * 0.8
    )

    self.textColor = Color(255, 255, 255)

    -- Get fonts
    local titleFont = ix.config.Get("font", "Roboto Th")
    local bodyFont = ix.config.Get("genericFont", "Roboto")

    -- Create themed fonts
    surface.CreateFont("MyPanel.Title", {
        font = titleFont,
        size = 28,
        weight = 600
    })

    surface.CreateFont("MyPanel.Body", {
        font = bodyFont,
        size = 18,
        weight = 400
    })

    self:InvalidateLayout()
end

function PANEL:Paint(w, h)
    -- Use theme colors
    surface.SetDrawColor(self.themeColor)
    surface.DrawRect(0, 0, w, h)

    -- Header
    surface.SetDrawColor(self.headerColor)
    surface.DrawRect(0, 0, w, 40)

    return true
end

vgui.Register("ixThemedPanel", PANEL, "DPanel")

-- Character menu with theme
function PLUGIN:CharacterMenuOpened()
    local frame = vgui.Create("DFrame")
    frame:SetTitle(ix.config.Get("communityText", "@community"))

    -- Apply theme
    local color = ix.config.Get("color")
    frame:SetBackgroundColor(color)

    -- Play music
    local music = ix.config.Get("music", "music/hl2_song2.mp3")
    if music != "" then
        surface.PlaySound(music)
    end

    -- Show intro if enabled
    if ix.config.Get("intro", true) and LocalPlayer():IsNewPlayer() then
        ShowIntroSequence()
    end
end

-- Custom vignette drawer
hook.Add("HUDPaint", "CustomVignette", function()
    if not ix.config.Get("vignette", true) then return end

    local w, h = ScrW(), ScrH()
    local themeColor = ix.config.Get("color", Color(75, 119, 190))

    -- Draw vignette with theme tint
    surface.SetDrawColor(themeColor.r, themeColor.g, themeColor.b, 30)
    surface.DrawRect(0, 0, w, h)
end)
```

## Best Practices

### ✅ DO

- Use `ix.config.Get()` for all theme colors and fonts
- Listen to `ColorSchemeChanged` hook for dynamic updates
- Derive UI colors from main theme color
- Use language phrases (prefix with `@`) for text
- Check vignette config before drawing effects
- Validate URLs before opening them
- Create fonts using configured font families
- Provide fallback values to `Get()` calls

### ❌ DON'T

- Don't hardcode colors in UI elements
- Don't ignore font configuration
- Don't bypass theme system for custom panels
- Don't forget to reload UI when theme changes
- Don't use custom fonts without checking config
- Don't disable vignette without checking setting
- Don't play music without checking config
- Don't hardcode community branding

## Common Patterns

### Pattern 1: Responsive UI to Theme Changes

```lua
-- Panel that updates automatically with theme
function PANEL:Init()
    self.BaseClass.Init(self)
    self:UpdateTheme()

    hook.Add("ColorSchemeChanged", self, function(panel, color)
        panel:UpdateTheme()
    end)
end

function PANEL:UpdateTheme()
    self.color = ix.config.Get("color")
    self.font = ix.config.Get("font")

    -- Recreate fonts if needed
    surface.CreateFont("MyPanel.Title", {
        font = self.font,
        size = 24
    })

    self:InvalidateLayout()
end
```

### Pattern 2: Dynamic Color Derivation

```lua
-- Generate color scheme from main theme color
function GetColorScheme()
    local base = ix.config.Get("color", Color(75, 119, 190))

    return {
        primary = base,
        secondary = Color(base.r * 0.7, base.g * 0.7, base.b * 0.7),
        accent = Color(base.r * 1.2, base.g * 1.2, base.b * 1.2),
        text = Color(255, 255, 255),
        textDark = Color(200, 200, 200)
    }
end

-- Use in UI
local colors = GetColorScheme()
panel:SetBackgroundColor(colors.primary)
```

### Pattern 3: Localized UI Text

```lua
-- Handle localized text from config
function GetLocalizedText(configKey, default)
    local text = ix.config.Get(configKey, default)

    -- Check if it's a language phrase
    if text:sub(1, 1) == "@" then
        text = L(text:sub(2))
    end

    return text
end

-- Usage
button:SetText(GetLocalizedText("communityText", "@community"))
```

## Common Issues

### Issue: UI Not Updating After Theme Change

**Cause**: Not listening to `ColorSchemeChanged` hook
**Fix**: Hook into theme changes and reload UI

```lua
-- Listen for theme changes
hook.Add("ColorSchemeChanged", "UpdateMyUI", function(newColor)
    -- Update all UI elements
    for _, panel in ipairs(myPanels) do
        panel:SetBackgroundColor(newColor)
    end
end)
```

### Issue: Custom Fonts Not Loading

**Cause**: Fonts created before config loaded
**Fix**: Create fonts in `LoadFonts` hook or after config ready

```lua
-- Create fonts when config loads
hook.Add("LoadFonts", "MyCustomFonts", function(titleFont, bodyFont)
    surface.CreateFont("MyCustomFont", {
        font = titleFont,
        size = 20
    })
end)
```

### Issue: Vignette Always Showing

**Cause**: Not checking vignette config
**Fix**: Check config before drawing vignette

```lua
-- Only draw if enabled
hook.Add("HUDPaint", "DrawVignette", function()
    if not ix.config.Get("vignette", true) then return end

    -- Draw vignette
end)
```

## Adding Custom Appearance Options

To add your own appearance options in schemas or plugins:

```lua
-- Add custom appearance config
ix.config.Add("accentColor", Color(255, 100, 100), "Accent color for highlights", function(oldValue, newValue)
    if (CLIENT) then
        hook.Run("AccentColorChanged", newValue)
    end
end, {
    category = "appearance"
})

-- Add custom font option
ix.config.Add("titleFont", "Bebas Neue", "Custom title font", function(oldValue, newValue)
    if (CLIENT) then
        hook.Run("LoadFonts", newValue, ix.config.Get("genericFont"))
    end
end, {
    category = "appearance"
})
```

## See Also

- [Configuration System](../systems/configuration.md) - Overview of config system
- [UI Development Guide](../guides/ui-development.md) - Creating custom UI
- [Server Configuration](server.md) - Server-wide settings
- [Character Configuration](characters.md) - Character settings
- Source: `gamemode/config/sh_config.lua`
