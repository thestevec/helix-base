# Creating Schema-Specific UI (Derma)

> **Reference**: `schema/derma/`, `gamemode/core/derma/`

This guide shows you how to create custom user interfaces for your schema using Garry's Mod's Derma system, integrated with Helix's UI framework.

## ⚠️ Important: Use Helix Derma Panels

**Build on Helix's existing UI components** rather than creating everything from scratch. The framework provides:
- Pre-styled panels matching Helix theme
- Responsive layouts
- Network integration
- Character data binding
- Consistent look and feel

## Core Concepts

### Derma in Schemas

Create custom UI for:
- Schema-specific menus (crafting, skills, etc.)
- Information displays (HUD elements)
- Interactive panels (vendor shops, ATMs)
- Character creation extensions
- Admin tools

### File Location

Create Derma files in:
```
schema/derma/cl_panelname.lua
```

**⚠️ Derma files must have `cl_` prefix** (client-only).

### Helix UI Components

Use Helix's built-in panels:
- `ixPanel` - Base panel
- `ixButton` - Styled button
- `ixLabel` - Text label
- `ixTextEntry` - Text input
- `ixListRow` - List item
- `ixIconButton` - Icon button
- `ixColorBox` - Color display

## Creating Your First UI Panel

### Simple Information Panel

```lua
-- File: schema/derma/cl_stats.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(400, 300)
    self:Center()
    self:SetTitle("Character Statistics")
    self:MakePopup()

    -- Background
    self.paint = function(panel, width, height)
        draw.RoundedBox(8, 0, 0, width, height, Color(40, 40, 40, 240))
    end

    -- Title
    local title = self:Add("DLabel")
    title:SetText("Statistics")
    title:SetFont("ixMediumFont")
    title:SetTextColor(color_white)
    title:Dock(TOP)
    title:DockMargin(10, 10, 10, 5)
    title:SetTall(30)

    -- Stats list
    self.statsList = self:Add("DScrollPanel")
    self.statsList:Dock(FILL)
    self.statsList:DockMargin(10, 5, 10, 10)

    self:PopulateStats()
end

function PANEL:PopulateStats()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if not character then return end

    -- Add stat entries
    self:AddStat("Name", character:GetName())
    self:AddStat("Faction", ix.faction.indices[character:GetFaction()].name)
    self:AddStat("Money", "$" .. character:GetMoney())
    self:AddStat("Play Time", string.FormattedTime(character:GetData("playTime", 0)))

    -- Custom stats
    self:AddStat("Hunger", character:GetData("hunger", 100) .. "%")
    self:AddStat("Reputation", character:GetData("reputation", 0))
end

function PANEL:AddStat(name, value)
    local statPanel = self.statsList:Add("DPanel")
    statPanel:Dock(TOP)
    statPanel:DockMargin(5, 2, 5, 2)
    statPanel:SetTall(25)

    statPanel.Paint = function(panel, w, h)
        draw.RoundedBox(4, 0, 0, w, h, Color(60, 60, 60, 200))
    end

    local nameLabel = statPanel:Add("DLabel")
    nameLabel:SetText(name .. ":")
    nameLabel:SetTextColor(Color(200, 200, 200))
    nameLabel:Dock(LEFT)
    nameLabel:SetWide(150)
    nameLabel:DockMargin(10, 0, 0, 0)

    local valueLabel = statPanel:Add("DLabel")
    valueLabel:SetText(tostring(value))
    valueLabel:SetTextColor(color_white)
    valueLabel:Dock(FILL)
end

vgui.Register("ixSchemaStats", PANEL, "DFrame")

-- Command to open panel
ix.command.Add("Stats", {
    description = "View your character statistics",
    OnRun = function(self, client)
        if CLIENT then
            vgui.Create("ixSchemaStats")
        end
    end
})
```

### Interactive Menu Panel

```lua
-- File: schema/derma/cl_crafting.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(600, 400)
    self:Center()
    self:SetTitle("Crafting Menu")
    self:MakePopup()

    self.Paint = function(panel, w, h)
        draw.RoundedBox(8, 0, 0, w, h, Color(40, 40, 40, 240))
    end

    -- Recipe list (left side)
    local recipeList = self:Add("DScrollPanel")
    recipeList:Dock(LEFT)
    recipeList:SetWide(200)
    recipeList:DockMargin(10, 10, 5, 10)

    -- Recipe details (right side)
    self.detailsPanel = self:Add("DPanel")
    self.detailsPanel:Dock(FILL)
    self.detailsPanel:DockMargin(5, 10, 10, 10)

    self.detailsPanel.Paint = function(panel, w, h)
        draw.RoundedBox(4, 0, 0, w, h, Color(60, 60, 60, 150))
    end

    -- Populate recipes
    self:PopulateRecipes(recipeList)
end

function PANEL:PopulateRecipes(parent)
    local recipes = {
        {
            name = "Bandage",
            materials = {
                {item = "item_cloth", amount = 2}
            },
            result = "item_bandage"
        },
        {
            name = "Weapon Repair Kit",
            materials = {
                {item = "item_scrap", amount = 5},
                {item = "item_spring", amount = 2}
            },
            result = "item_repair_kit"
        }
    }

    for _, recipe in ipairs(recipes) do
        local button = parent:Add("DButton")
        button:Dock(TOP)
        button:DockMargin(5, 2, 5, 2)
        button:SetTall(40)
        button:SetText(recipe.name)

        button.DoClick = function()
            self:ShowRecipeDetails(recipe)
        end

        button.Paint = function(panel, w, h)
            local col = panel:IsHovered() and Color(80, 80, 80) or Color(60, 60, 60)
            draw.RoundedBox(4, 0, 0, w, h, col)
        end
    end
end

function PANEL:ShowRecipeDetails(recipe)
    self.detailsPanel:Clear()

    -- Recipe name
    local title = self.detailsPanel:Add("DLabel")
    title:SetText(recipe.name)
    title:SetFont("ixMediumFont")
    title:SetTextColor(color_white)
    title:Dock(TOP)
    title:DockMargin(10, 10, 10, 5)
    title:SetTall(30)

    -- Materials required
    local materialsLabel = self.detailsPanel:Add("DLabel")
    materialsLabel:SetText("Materials Required:")
    materialsLabel:SetTextColor(Color(200, 200, 200))
    materialsLabel:Dock(TOP)
    materialsLabel:DockMargin(10, 5, 10, 5)

    for _, mat in ipairs(recipe.materials) do
        local matLabel = self.detailsPanel:Add("DLabel")
        matLabel:SetText("  - " .. ix.item.list[mat.item].name .. " x" .. mat.amount)
        matLabel:SetTextColor(color_white)
        matLabel:Dock(TOP)
        matLabel:DockMargin(15, 2, 10, 2)
    end

    -- Craft button
    local craftButton = self.detailsPanel:Add("DButton")
    craftButton:Dock(BOTTOM)
    craftButton:DockMargin(10, 10, 10, 10)
    craftButton:SetTall(40)
    craftButton:SetText("Craft")

    craftButton.DoClick = function()
        -- Send craft request to server
        net.Start("SchemaCraftItem")
        net.WriteString(recipe.result)
        net.SendToServer()

        self:Close()
    end

    craftButton.Paint = function(panel, w, h)
        local col = panel:IsHovered() and Color(100, 150, 100) or Color(80, 120, 80)
        draw.RoundedBox(4, 0, 0, w, h, col)
    end
end

vgui.Register("ixSchemaCrafting", PANEL, "DFrame")

-- Network receiver (SERVER)
if SERVER then
    util.AddNetworkString("SchemaCraftItem")

    net.Receive("SchemaCraftItem", function(len, client)
        local itemID = net.ReadString()
        local character = client:GetCharacter()

        if not character then return end

        -- Verify materials and craft item
        -- (Add your crafting logic here)

        local inventory = character:GetInventory()
        inventory:Add(itemID)

        client:Notify("Crafted " .. ix.item.list[itemID].name)
    end)
end
```

## Using Helix UI Components

### ixPanel

```lua
local panel = vgui.Create("ixPanel", parent)
panel:SetSize(300, 200)
panel:SetTitle("My Panel")

-- ixPanel automatically has Helix styling
```

### ixButton

```lua
local button = vgui.Create("ixButton", parent)
button:SetText("Click Me")
button:SizeToContents()
button.DoClick = function()
    print("Button clicked!")
end
```

### ixTextEntry

```lua
local entry = vgui.Create("ixTextEntry", parent)
entry:SetPlaceholder("Enter text...")
entry:Dock(TOP)
entry.OnEnter = function(self)
    print("Text entered:", self:GetValue())
end
```

### ixLabel

```lua
local label = vgui.Create("ixLabel", parent)
label:SetText("Information Text")
label:SetFont("ixMediumFont")
label:SizeToContents()
```

## HUD Elements

### Custom HUD Display

```lua
-- File: schema/derma/cl_hud.lua
local PANEL = {}

function PANEL:Init()
    -- HUD panels should not have frame or be popup
    self:SetSize(200, 100)
    self:ParentToHUD()
end

function PANEL:Paint(width, height)
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if not character then return end

    -- Don't draw if option disabled
    if not ix.option.Get("showCustomHUD", true) then
        return
    end

    -- Background
    draw.RoundedBox(8, 0, 0, width, height, Color(40, 40, 40, 200))

    -- Hunger bar
    local hunger = character:GetData("hunger", 100)
    draw.RoundedBox(4, 10, 30, width - 20, 20, Color(60, 60, 60))
    draw.RoundedBox(4, 12, 32, (width - 24) * (hunger / 100), 16, Color(255, 150, 0))

    -- Text
    draw.SimpleText("Hunger", "ixSmallFont", 10, 10, color_white)
    draw.SimpleText(hunger .. "%", "ixSmallFont", width - 10, 10, color_white, TEXT_ALIGN_RIGHT)
end

function PANEL:Think()
    -- Update position
    local scrW, scrH = ScrW(), ScrH()
    self:SetPos(scrW - self:GetWide() - 20, scrH - self:GetTall() - 120)
end

vgui.Register("ixSchemaHUD", PANEL, "ixPanel")

-- Create HUD on character load
hook.Add("InitializedCharacter", "SchemaCreateHUD", function()
    if IsValid(ix.gui.schemaHUD) then
        ix.gui.schemaHUD:Remove()
    end

    ix.gui.schemaHUD = vgui.Create("ixSchemaHUD")
end)
```

### Notification System

```lua
-- File: schema/derma/cl_notify.lua
function Schema:ShowNotification(title, message, duration)
    local panel = vgui.Create("DPanel")
    panel:SetSize(300, 80)
    panel:SetPos(ScrW() - 320, ScrH() + 100)  -- Start off-screen

    panel.Paint = function(self, w, h)
        draw.RoundedBox(8, 0, 0, w, h, Color(40, 40, 40, 240))
        draw.SimpleText(title, "ixMediumFont", 10, 10, color_white)
        draw.SimpleText(message, "ixSmallFont", 10, 35, Color(200, 200, 200))
    end

    -- Slide in animation
    panel:MoveTo(ScrW() - 320, ScrH() - 100, 0.3, 0, -1)

    -- Slide out after duration
    timer.Simple(duration or 3, function()
        if IsValid(panel) then
            panel:MoveTo(ScrW() - 320, ScrH() + 100, 0.3, 0, -1, function()
                panel:Remove()
            end)
        end
    end)
end

-- Usage
Schema:ShowNotification("Achievement", "You found a secret!", 5)
```

## Advanced UI Patterns

### List with Search

```lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(400, 500)
    self:Center()
    self:MakePopup()

    -- Search bar
    self.search = self:Add("ixTextEntry")
    self.search:Dock(TOP)
    self.search:DockMargin(10, 10, 10, 5)
    self.search:SetPlaceholder("Search...")

    self.search.OnValueChange = function(entry, value)
        self:FilterList(value)
    end

    -- Item list
    self.list = self:Add("DScrollPanel")
    self.list:Dock(FILL)
    self.list:DockMargin(10, 5, 10, 10)

    self:PopulateList()
end

function PANEL:PopulateList()
    self.items = {}

    for uniqueID, itemTable in pairs(ix.item.list) do
        local item = self.list:Add("DButton")
        item:Dock(TOP)
        item:DockMargin(0, 2, 0, 2)
        item:SetTall(30)
        item:SetText(itemTable.name)
        item.itemTable = itemTable

        self.items[#self.items + 1] = item
    end
end

function PANEL:FilterList(query)
    query = string.lower(query)

    for _, item in ipairs(self.items) do
        local name = string.lower(item.itemTable.name)

        if query == "" or string.find(name, query, 1, true) then
            item:SetVisible(true)
        else
            item:SetVisible(false)
        end
    end
end

vgui.Register("ixSchemaItemList", PANEL, "DFrame")
```

### Tab System

```lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(600, 400)
    self:Center()
    self:MakePopup()

    -- Tab buttons
    local tabContainer = self:Add("DPanel")
    tabContainer:Dock(TOP)
    tabContainer:SetTall(40)

    tabContainer.Paint = function(panel, w, h)
        draw.RoundedBox(0, 0, 0, w, h, Color(60, 60, 60))
    end

    -- Content area
    self.content = self:Add("DPanel")
    self.content:Dock(FILL)

    self.content.Paint = function(panel, w, h)
        draw.RoundedBox(0, 0, 0, w, h, Color(40, 40, 40))
    end

    -- Create tabs
    self:AddTab("Inventory", tabContainer, function(content)
        local label = content:Add("DLabel")
        label:SetText("Inventory Content")
        label:Dock(FILL)
    end)

    self:AddTab("Skills", tabContainer, function(content)
        local label = content:Add("DLabel")
        label:SetText("Skills Content")
        label:Dock(FILL)
    end)

    self:AddTab("Map", tabContainer, function(content)
        local label = content:Add("DLabel")
        label:SetText("Map Content")
        label:Dock(FILL)
    end)
end

function PANEL:AddTab(name, parent, populateFunc)
    local button = parent:Add("DButton")
    button:Dock(LEFT)
    button:SetWide(100)
    button:SetText(name)

    button.Paint = function(panel, w, h)
        local col = panel.active and Color(40, 40, 40) or Color(60, 60, 60)
        if panel:IsHovered() then
            col = Color(70, 70, 70)
        end
        draw.RoundedBox(0, 0, 0, w, h, col)
    end

    button.DoClick = function()
        self:SetActiveTab(button, populateFunc)
    end

    -- Activate first tab
    if not self.activeTab then
        self:SetActiveTab(button, populateFunc)
    end
end

function PANEL:SetActiveTab(button, populateFunc)
    -- Deactivate old tab
    if self.activeTab then
        self.activeTab.active = false
    end

    -- Activate new tab
    self.activeTab = button
    button.active = true

    -- Clear and populate content
    self.content:Clear()
    populateFunc(self.content)
end

vgui.Register("ixSchemaTabs", PANEL, "DFrame")
```

## Best Practices

### ✅ DO

- Create Derma files in `schema/derma/` with `cl_` prefix
- Use Helix's built-in UI components when possible
- Make panels responsive to different screen sizes
- Test UI at different resolutions
- Use ix.option for UI customization settings
- Close panels properly to prevent memory leaks
- Use network messages for client-server communication
- Validate all user input on server
- Cache frequently accessed data
- Use appropriate fonts and colors

### ❌ DON'T

- Don't create UI in server realm
- Don't bypass network validation
- Don't forget to remove panels when done
- Don't hardcode positions (use Dock/Center)
- Don't create panels every frame
- Don't trust client-provided data
- Don't forget to check if character exists
- Don't use expensive operations in Paint
- Don't create UI without permission checks

## Performance Tips

```lua
-- Bad: Creating panels every frame
hook.Add("HUDPaint", "DrawPanel", function()
    local panel = vgui.Create("DPanel")  -- Memory leak!
    panel:SetSize(100, 100)
end)

-- Good: Create once, update as needed
hook.Add("InitializedCharacter", "CreatePanel", function()
    if IsValid(ix.gui.myPanel) then
        ix.gui.myPanel:Remove()
    end

    ix.gui.myPanel = vgui.Create("ixMyPanel")
end)

-- Cache data
local cachedData
local nextUpdate = 0

function PANEL:Think()
    if CurTime() > nextUpdate then
        cachedData = self:GetExpensiveData()
        nextUpdate = CurTime() + 1  -- Update every second
    end
end
```

## Testing UI

1. **Test Resolution**: Try different screen sizes
2. **Test with Data**: Use various character data values
3. **Test Interaction**: Click all buttons and inputs
4. **Test Network**: Ensure server communication works
5. **Test Performance**: Check frame rate impact
6. **Test Cleanup**: Verify panels close properly

## See Also

- [Derma Overview](../ui/derma-overview.md) - Garry's Mod UI basics
- [HUD System](../ui/hud.md) - HUD creation
- [Character System](../systems/character.md) - Character data access
- [Schema Structure](structure.md) - Schema file organization
- Source: `gamemode/core/derma/` - Helix UI components
