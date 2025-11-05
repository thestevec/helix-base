# Derma UI System

> **Reference**: `gamemode/core/derma/` directory

Helix provides an extensive collection of custom Derma (VGUI) panels that extend Garry's Mod's standard UI system with framework-specific styling, animations, and functionality.

## ⚠️ Important: Use Built-in Helix Derma Panels

**Always use Helix's built-in Derma panels** rather than creating your own UI from scratch. The framework provides:
- Automatic theme/skin integration
- Built-in animations and transitions
- Consistent styling across all interfaces
- Network synchronization where needed
- Integration with ix.option for user preferences
- Responsive layouts that scale to screen resolution

## Core Concepts

### What is the Derma System?

Helix's Derma system is a collection of 29+ custom VGUI panels that provide the user interface for the framework. These panels extend Garry's Mod's standard Derma controls with:

- **Custom styling** matching the Helix theme
- **Animation support** for smooth transitions
- **Framework integration** with character, inventory, and configuration systems
- **Panel overrides** that enhance standard Derma functionality

### Built-in Panel Types

Helix registers numerous custom panels, organized by purpose:

**Generic UI Components** (`cl_generic.lua`):
- `ixTextEntry` - Styled text input
- `ixCategoryPanel` - Collapsible category containers
- `ixSegmentedProgress` - Multi-segment progress bars
- `ixListRow` - List item rows
- `ixCheckBox` - Styled checkboxes
- `ixNumSlider` - Numeric slider controls
- `ixSlider` - Generic slider controls
- `ixLabel` - Styled labels
- `ixIconTextEntry` - Text input with icon

**Main Menu System** (`cl_menu.lua`):
- `ixMenu` - Main F1 menu container
- `ixMenuButton` - Menu navigation buttons
- `ixMenuSelectionButton` - Selection list buttons
- `ixMenuSelectionList` - Selection lists

**Character System** (`cl_character.lua`, `cl_charload.lua`, `cl_charcreate.lua`):
- `ixCharMenu` - Character selection screen
- `ixCharMenuPanel` - Base character menu panel
- `ixCharMenuButtonList` - Character action buttons
- `ixCharMenuMain` - Main character display
- `ixCharMenuCarousel` - Character carousel display
- `ixCharMenuLoad` - Character loading screen
- `ixCharMenuNew` - Character creation screen

**Inventory System** (`cl_inventory.lua`):
- `ixInventory` - Inventory grid display
- `ixItemIcon` - Individual item icons

**Notification System** (`cl_notice.lua`, `cl_noticebar.lua`):
- `ixNotice` - Pop-up notifications
- `ixNoticeManager` - Notification container
- `ixNoticeBar` - Top-of-screen notice bar

**HUD Elements** (`cl_bar.lua`):
- `ixInfoBar` - Individual HUD status bar
- `ixInfoBarManager` - HUD bar container

**Settings/Config** (`cl_settings.lua`, `cl_config.lua`):
- `ixSettings` - Base settings panel
- `ixSettingsRow` - Individual setting row
- `ixSettingsRowColor`, `ixSettingsRowNumber`, etc. - Type-specific rows
- `ixConfigManager` - Server configuration interface
- `ixPluginManager` - Plugin management interface

**Other Panels**:
- `ixTooltip` - Item tooltips
- `ixScoreboard` - Player list
- `ixHelp` - Help menu
- `ixBusiness` - Business/vendor interface
- `ixStorage` - Storage container interface
- And many more...

## Using Helix Derma Panels

### Creating a Panel

**Reference**: `gamemode/core/derma/cl_generic.lua:30`

```lua
-- Create a Helix panel
local panel = vgui.Create("ixTextEntry")
panel:SetSize(200, 30)
panel:SetFont("ixSmallFont")
panel:SetPos(100, 100)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use plain DTextEntry when Helix provides styled version
local panel = vgui.Create("DTextEntry")  -- Won't match Helix theme!
```

### Panel Registration Pattern

**Reference**: `gamemode/core/derma/cl_generic.lua:5-30`

All Helix panels follow this registration pattern:

```lua
DEFINE_BASECLASS("ParentPanel")
local PANEL = {}

-- Accessors for properties
AccessorFunc(PANEL, "myProperty", "MyProperty")

function PANEL:Init()
    -- Initialization code
end

function PANEL:Paint(width, height)
    -- Custom painting
end

-- Register the panel
vgui.Register("ixPanelName", PANEL, "ParentPanel")
```

### Opening Main Menu

**Reference**: `gamemode/core/derma/cl_menu.lua:10`

```lua
-- Open the main F1 menu
local menu = vgui.Create("ixMenu")
-- Menu handles MakePopup() automatically
```

### Opening Character Menu

**Reference**: `gamemode/core/derma/cl_character.lua:541`

```lua
-- Open character selection screen
local charMenu = vgui.Create("ixCharMenu")
-- Automatically displays character carousel
```

### Creating Inventory Display

**Reference**: `gamemode/core/derma/cl_inventory.lua:740`

```lua
-- Display an inventory (usually handled by framework)
local inventoryPanel = vgui.Create("ixInventory")
inventoryPanel:SetInventory(inventoryObject)
```

## Panel Overrides

**Reference**: `gamemode/core/derma/cl_overrides.lua`

Helix overrides several standard Garry's Mod panels to add functionality:

### DMenu Enhancements

**Reference**: `gamemode/core/derma/cl_overrides.lua:60-210`

```lua
-- Standard context menus automatically get:
-- - Smooth opening/closing animations
-- - Respects ix.option.Get("disableAnimations")
-- - Better font handling
-- - Improved scrolling

-- Usage is the same as normal DMenu
local menu = DermaMenu()
menu:AddOption("Option 1", function()
    print("Selected!")
end)
menu:Open()
```

### DComboBox Improvements

**Reference**: `gamemode/core/derma/cl_overrides.lua:212-224`

```lua
-- Combo boxes automatically get:
-- - Better font inheritance
-- - Improved dropdown positioning
-- - Height limiting to fit screen
```

### DScrollPanel Fixes

**Reference**: `gamemode/core/derma/cl_overrides.lua:226-243`

```lua
-- Scroll panels get improved ScrollToChild functionality
-- Works correctly with docked panels
scrollPanel:ScrollToChild(childPanel)
```

## Complete Example: Creating a Simple Menu

```lua
-- Create custom menu panel
DEFINE_BASECLASS("DFrame")
local PANEL = {}

function PANEL:Init()
    self:SetSize(400, 300)
    self:Center()
    self:SetTitle("My Custom Menu")
    self:MakePopup()

    -- Use Helix styled text entry
    local textEntry = self:Add("ixTextEntry")
    textEntry:SetFont("ixMediumFont")
    textEntry:Dock(TOP)
    textEntry:DockMargin(8, 8, 8, 8)
    textEntry:SetPlaceholderText("Enter text...")

    -- Use Helix styled category panel
    local category = self:Add("ixCategoryPanel")
    category:SetText("Options")
    category:Dock(TOP)
    category:DockMargin(8, 8, 8, 8)

    -- Add checkbox inside category
    local checkbox = category:Add("ixCheckBox")
    checkbox:SetText("Enable Feature")
    checkbox:Dock(TOP)
    checkbox:DockMargin(4, 4, 4, 4)
    checkbox:SizeToContents()
end

function PANEL:Paint(width, height)
    -- Use derma skin function for consistent styling
    derma.SkinFunc("DrawImportantBackground", 0, 0, width, height, Color(0, 0, 0, 200))
end

vgui.Register("MyCustomMenu", PANEL, "DFrame")

-- Usage
local menu = vgui.Create("MyCustomMenu")
```

## Animation Support

**Reference**: `gamemode/core/derma/cl_overrides.lua:141-153`

Many Helix panels support smooth animations:

```lua
-- Animations automatically respect user preferences
if not ix.option.Get("disableAnimations") then
    panel:CreateAnimation(0.5, {
        target = {alpha = 255},
        easing = "outQuint",

        Think = function(animation, panel)
            panel:SetAlpha(panel.alpha)
        end,

        OnComplete = function(animation, panel)
            print("Animation complete!")
        end
    })
end
```

## Best Practices

### ✅ DO

- Use `vgui.Create("ixPanelName")` for Helix panels
- Leverage framework styling and animations
- Check `ix.option.Get("disableAnimations")` before animating
- Use `derma.SkinFunc()` for consistent styling
- Extend `ixSubpanel` for menu subpanels
- Use `ixMenuButton` for menu navigation
- Call `MakePopup()` on main panels to capture input

### ❌ DON'T

- Don't use plain Derma panels when Helix versions exist
- Don't implement custom styling that conflicts with theme
- Don't create animations without checking user preferences
- Don't access `ix.gui` table directly (use provided methods)
- Don't forget to handle panel cleanup on Remove()
- Don't hardcode positions/sizes (use Dock and scale functions)
- Don't bypass the subpanel system for menu tabs

## Common Patterns

### Pattern 1: Menu Subpanel

```lua
-- Extend ixSubpanel for menu integration
DEFINE_BASECLASS("ixSubpanel")
local PANEL = {}

AccessorFunc(PANEL, "bReadOnly", "ReadOnly", FORCE_BOOL)

function PANEL:Init()
    -- Subpanels auto-register with parent menu
end

function PANEL:OnDisplay()
    -- Called when subpanel becomes visible
    self:Populate()
end

function PANEL:Populate()
    -- Build your interface here
end

vgui.Register("MyMenuTab", PANEL, "ixSubpanel")
```

### Pattern 2: Scaled UI Element

```lua
-- Use ScreenScale for resolution-independent sizing
local panel = vgui.Create("DPanel")
panel:SetSize(ScreenScale(200), ScreenScale(150))
panel:SetPos(ScreenScale(50), ScreenScale(50))

-- Or use Helix helper
local padding = ScrH() * 0.01  -- 1% of screen height
```

### Pattern 3: Respecting User Options

```lua
-- Always check user preferences
local function CreateAnimatedPanel()
    local panel = vgui.Create("DPanel")

    if ix.option.Get("disableAnimations") then
        panel:SetAlpha(255)  -- Instant
    else
        panel:SetAlpha(0)
        panel:CreateAnimation(0.3, {
            target = {alpha = 255},
            easing = "outQuint"
        })
    end

    return panel
end
```

## Common Issues

### Panel Not Showing

**Cause**: Forgot to call `MakePopup()` or set size/position
**Fix**: Ensure panel captures input and has valid dimensions

```lua
local panel = vgui.Create("ixMenu")
panel:SetSize(ScrW(), ScrH())
panel:SetPos(0, 0)
panel:MakePopup()  -- Critical for capturing input!
```

### Styling Doesn't Match Framework

**Cause**: Using standard Derma panels instead of Helix versions
**Fix**: Use `ix` prefixed panels

```lua
-- BEFORE (wrong)
local entry = vgui.Create("DTextEntry")

-- AFTER (correct)
local entry = vgui.Create("ixTextEntry")
```

### Animation Not Working

**Cause**: User has animations disabled or animation not properly configured
**Fix**: Always check options and use correct animation structure

```lua
-- Check setting first
if not ix.option.Get("disableAnimations") then
    panel:CreateAnimation(time, {
        target = {property = value},
        easing = "outQuint",
        Think = function(anim, pnl)
            -- Update during animation
        end
    })
end
```

## See Also

- [HUD System](hud.md) - HUD bars and displays
- [Menus](menus.md) - F1 menu system
- [Inventory UI](inventory.md) - Inventory interface
- [Character Creation](character-creation.md) - Character UI
- [Notifications](notifications.md) - Notice system
- [Configuration System](../systems/configuration.md) - ix.option system
- Source: `gamemode/core/derma/*.lua`
