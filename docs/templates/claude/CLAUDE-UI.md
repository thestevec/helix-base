# Helix UI Development - AI Assistant Guide

> Context-specific guidance for developing user interfaces in the Helix framework.

## 📚 Documentation Knowledgebase

**Primary Reference**: [docs/llms.txt](../../llms.txt) - Complete framework documentation index

## UI Development Essentials

### Core Documentation
1. **[Derma Overview](../../ui/derma-overview.md)** - UI component system and framework
2. **[UI Development Guide](../../guides/ui-development.md)** - Creating Derma interfaces
3. **[Menu System](../../ui/menus.md)** - Main menu, F1 menu, tab navigation
4. **[HUD System](../../ui/hud.md)** - Status bars, health, stamina displays
5. **[Markup System](../../ui/markup.md)** - Rich text rendering

### UI Systems Reference

**Core UI Components**:
- [Derma Overview](../../ui/derma-overview.md) - Base component system
- [Menu System](../../ui/menus.md) - Menu registration and tabs
- [HUD System](../../ui/hud.md) - On-screen displays
- [Inventory UI](../../ui/inventory.md) - Drag-and-drop inventory
- [Character Creation](../../ui/character-creation.md) - Character screens
- [Notifications](../../ui/notifications.md) - Notice system

### Common Derma Components

#### Base Panel
```lua
local PANEL = {}

function PANEL:Init()
    -- Initialize panel
    self:SetSize(400, 300)
    self:Center()
    self:MakePopup()
end

function PANEL:Paint(w, h)
    -- Draw panel background
    draw.RoundedBox(0, 0, 0, w, h, Color(50, 50, 50, 250))
end

vgui.Register("ixCustomPanel", PANEL, "EditablePanel")
```

#### Creating Buttons
```lua
local button = self:Add("DButton")
button:SetText("Button Text")
button:SetSize(100, 30)
button:SetPos(10, 10)
button.DoClick = function()
    -- Button click handler
end
```

#### Creating Text Entry
```lua
local textEntry = self:Add("DTextEntry")
textEntry:SetSize(200, 25)
textEntry:SetPos(10, 50)
textEntry:SetPlaceholderText("Enter text...")
textEntry.OnEnter = function(entry)
    local text = entry:GetValue()
    -- Handle text entry
end
```

#### Creating Lists
```lua
local list = self:Add("DScrollPanel")
list:SetSize(300, 200)
list:SetPos(10, 90)

-- Add items to list
for i = 1, 10 do
    local item = list:Add("DButton")
    item:SetText("Item " .. i)
    item:Dock(TOP)
    item:DockMargin(5, 5, 5, 0)
end
```

### Menu Registration

**API Reference**: [ix.menu](../../api/menu.md)

```lua
ix.menu.Add("MenuName", "Tab Display Name", "icon_path", function(panel)
    -- Create menu contents
    local container = panel:Add("Panel")
    container:Dock(FILL)

    -- Add UI elements
end, function()
    -- Optional: Return true/false to show/hide tab
    return true
end)
```

### HUD Elements

**Essential Reading**: [HUD System](../../ui/hud.md)

**API Reference**: [ix.bar](../../api/hud.md)

#### Adding Status Bars
```lua
ix.bar.Add(function()
    return LocalPlayer():Health() / LocalPlayer():GetMaxHealth()
end, Color(200, 50, 50), nil, "health")

ix.bar.Add(function()
    return LocalPlayer():Armor() / 100
end, Color(50, 150, 200), nil, "armor")
```

#### Custom HUD Elements
```lua
hook.Add("HUDPaint", "ixCustomHUD", function()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if character then
        -- Draw custom HUD elements
        draw.SimpleText("Text", "Font", x, y, color, TEXT_ALIGN_LEFT)
    end
end)
```

### Networking for UI

**Essential Reading**: [Networking Guide](../../guides/networking.md)

**API Reference**: [ix.net](../../api/networking.md)

#### Opening UI from Server
```lua
-- Server-side
netstream.Start(client, "OpenCustomUI", data)

-- Client-side
netstream.Hook("OpenCustomUI", function(data)
    local panel = vgui.Create("ixCustomPanel")
    panel:SetData(data)
end)
```

#### Sending Data to Server
```lua
-- Client-side
netstream.Start("CustomUIAction", {
    action = "something",
    value = 123
})

-- Server-side
netstream.Hook("CustomUIAction", function(client, data)
    -- Handle action from client UI
end)
```

### Styling and Theming

#### Custom Colors
```lua
-- Define color scheme
local PANEL = {}

PANEL.BackgroundColor = Color(40, 40, 40, 240)
PANEL.AccentColor = Color(100, 150, 255)
PANEL.TextColor = Color(255, 255, 255)

function PANEL:Paint(w, h)
    draw.RoundedBox(8, 0, 0, w, h, self.BackgroundColor)

    -- Accent bar
    draw.RoundedBox(0, 0, 0, w, 3, self.AccentColor)
end
```

#### Using Helix Theme
```lua
-- Use built-in theme colors
surface.SetDrawColor(ix.config.Get("color"))

-- Use theme fonts
surface.SetFont("ixMenuButtonFont")
```

### Markup System

**Essential Reading**: [Markup System](../../ui/markup.md)

```lua
local markup = ix.markup.Parse("<font=ixBigFont>Title</font>\nRegular text")
markup:Draw(x, y, TEXT_ALIGN_LEFT, TEXT_ALIGN_TOP, alpha)
```

### Inventory UI Integration

**Essential Reading**: [Inventory UI](../../ui/inventory.md), [Inventory System](../../systems/inventory.md)

```lua
-- Open inventory
local inventory = character:GetInventory()
ix.gui.inv1 = vgui.Create("ixInventory")
ix.gui.inv1:SetInventory(inventory)
```

### Character Creation UI

**Essential Reading**: [Character Creation](../../ui/character-creation.md)

Customize character creation screens:
```lua
function PANEL:OnCharacterCreation()
    -- Add custom fields to character creation
end
```

### Notifications

**Essential Reading**: [Notifications](../../ui/notifications.md)

**API Reference**: [ix.notice](../../api/notice.md)

```lua
-- Client-side notification
ix.util.Notify("Message text")

-- Server-side to specific client
client:Notify("Message text")

-- Server-side to all clients
ix.util.NotifyAll("Message text")
```

### UI Development Workflow

#### 1. Planning Phase
- [ ] Sketch UI layout and flow
- [ ] Identify required components
- [ ] Plan data flow (client/server)
- [ ] Consider mobile/different resolutions
- [ ] Check if similar UI exists in base framework

#### 2. Implementation Phase
- [ ] Create PANEL structure
- [ ] Implement Init() function
- [ ] Add UI components (buttons, text, etc.)
- [ ] Implement Paint() for custom drawing
- [ ] Add event handlers (DoClick, OnEnter, etc.)
- [ ] Register panel with vgui.Register()

#### 3. Integration Phase
- [ ] Set up networking if needed
- [ ] Integrate with game systems
- [ ] Add to menu system if applicable
- [ ] Test data synchronization

#### 4. Polish Phase
- [ ] Add animations and transitions
- [ ] Implement proper scaling
- [ ] Add sound effects
- [ ] Test on different resolutions
- [ ] Optimize performance

### Common UI Patterns

#### Modal Dialog
```lua
local function ShowDialog(title, message, callback)
    local frame = vgui.Create("DFrame")
    frame:SetSize(300, 150)
    frame:SetTitle(title)
    frame:Center()
    frame:MakePopup()

    local label = frame:Add("DLabel")
    label:SetText(message)
    label:Dock(FILL)
    label:SetContentAlignment(5)

    local button = frame:Add("DButton")
    button:SetText("OK")
    button:Dock(BOTTOM)
    button.DoClick = function()
        frame:Remove()
        if callback then callback() end
    end
end
```

#### Confirmation Dialog
```lua
Derma_Query(
    "Are you sure?",
    "Confirmation",
    "Yes", function() -- Yes callback end,
    "No", function() -- No callback end
)
```

#### Context Menu
```lua
local menu = DermaMenu()
menu:AddOption("Option 1", function() -- Action end)
menu:AddOption("Option 2", function() -- Action end)
menu:AddSpacer()
menu:AddOption("Cancel", function() end)
menu:Open()
```

### Performance Optimization

#### Caching
```lua
-- Cache expensive calculations
function PANEL:Init()
    self.cachedData = {}
end

function PANEL:GetCachedValue(key)
    if !self.cachedData[key] then
        self.cachedData[key] = ExpensiveCalculation()
    end
    return self.cachedData[key]
end
```

#### Think Function
```lua
-- Use Think() for periodic updates, not Paint()
function PANEL:Think()
    if (self.nextUpdate or 0) < CurTime() then
        self:UpdateData()
        self.nextUpdate = CurTime() + 1  -- Update every second
    end
end
```

#### Avoid Paint() Spam
```lua
-- Bad: Creating objects in Paint()
function PANEL:Paint(w, h)
    local color = Color(255, 0, 0)  -- New object every frame!
    draw.RoundedBox(0, 0, 0, w, h, color)
end

-- Good: Cache in Init()
function PANEL:Init()
    self.bgColor = Color(255, 0, 0)
end

function PANEL:Paint(w, h)
    draw.RoundedBox(0, 0, 0, w, h, self.bgColor)
end
```

### Accessibility Considerations

- Use readable font sizes
- Ensure sufficient color contrast
- Provide keyboard navigation where possible
- Test at different screen resolutions
- Consider colorblind-friendly palettes

### Debugging UI

```lua
-- Debug panel bounds
function PANEL:Paint(w, h)
    -- Your normal paint code

    -- Debug outline
    surface.SetDrawColor(255, 0, 0, 255)
    surface.DrawOutlinedRect(0, 0, w, h)
end

-- Print panel hierarchy
PrintTable(panel:GetChildren())

-- Check panel position and size
print(panel:GetPos(), panel:GetSize())
```

### Example References

- [Example Derma Panel](../../examples/derma.md) - Complete panel implementation
- Base plugin UIs for reference:
  - [Chatbox UI](../../plugins/chatbox.md)
  - [Vendor UI](../../plugins/vendor.md)
  - [Container UI](../../plugins/containers.md)

### Troubleshooting

**Panel not showing?**
- Check if MakePopup() is called
- Verify panel size is set correctly
- Check z-order (newer panels on top)
- Ensure panel isn't created server-side

**Paint not working?**
- Verify Paint(w, h) signature
- Check if panel has size
- Ensure drawing code uses w, h parameters

**Networking issues?**
- Verify net message is registered
- Check realm (client vs server)
- See [Networking Guide](../../guides/networking.md)

### Best Practices Checklist

- [ ] Use Helix theme colors where appropriate
- [ ] Implement proper cleanup in OnRemove()
- [ ] Cache expensive operations
- [ ] Use Think() instead of Paint() for updates
- [ ] Test at multiple resolutions
- [ ] Add proper error handling
- [ ] Follow Helix UI conventions
- [ ] Optimize paint operations
- [ ] Implement keyboard shortcuts
- [ ] Add helpful tooltips

---

**Quick Links**:
- [Back to Main Guide](../../../CLAUDE.md)
- [Complete Documentation Index](../../llms.txt)
- [UI Examples](../../examples/derma.md)
- [UI Development Guide](../../guides/ui-development.md)
