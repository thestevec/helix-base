# Creating Schema UI (Derma)

> **Reference**: `gamemode/core/derma/`, `schema/derma/`

Derma is Garry's Mod's UI system. This guide shows how to create custom user interfaces for your schema.

## ⚠️ Important: Use Helix Derma Integration

**Always use Helix's Derma helpers and patterns** rather than creating standalone UI. The framework provides:
- Consistent styling and theming
- Network integration
- Permission checking
- Menu system integration

## Core Concepts

### What is Schema Derma?

Schema Derma includes:
- Custom menus and panels
- Information displays
- Interaction interfaces
- Admin tools

Examples:
- **HL2RP**: Combine terminals, ration dispensers, permit viewers
- **DarkRP**: Job selection, shop menus, ATMs
- **Medieval RP**: Quest boards, crafting interfaces, market stalls

## Creating Derma Panels

### File Location

**Place Derma files** in:
```
schema/derma/cl_menuname.lua
```

**File naming convention**:
- `cl_` prefix (client-only)
- Descriptive menu name
- `.lua` extension

Examples:
- `cl_terminal.lua` → Combine terminal interface
- `cl_vendor.lua` → Vendor shop menu
- `cl_crafting.lua` → Crafting interface

### Basic Panel Template

```lua
-- schema/derma/cl_custom_menu.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(600, 400)
    self:Center()
    self:MakePopup()
    self:SetTitle("Custom Menu")

    -- Create close button
    self.closeButton = self:Add("DButton")
    self.closeButton:SetText("Close")
    self.closeButton:SetPos(10, self:GetTall() - 40)
    self.closeButton:SetSize(100, 30)
    self.closeButton.DoClick = function()
        self:Close()
    end
end

function PANEL:Paint(w, h)
    -- Background
    draw.RoundedBox(0, 0, 0, w, h, Color(40, 40, 40, 250))

    -- Title bar
    draw.RoundedBox(0, 0, 0, w, 30, Color(60, 60, 60))
    draw.SimpleText("Custom Menu", "DermaLarge", w / 2, 15, Color(255, 255, 255), TEXT_ALIGN_CENTER, TEXT_ALIGN_CENTER)
end

function PANEL:Close()
    self:Remove()
end

vgui.Register("ixCustomMenu", PANEL, "DFrame")

-- Usage: vgui.Create("ixCustomMenu")
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create panels without proper integration
local frame = vgui.Create("DFrame")
frame:SetSize(500, 400)
frame:MakePopup()
-- This works but doesn't integrate with Helix theming
```

## Common Derma Examples

### Vendor Shop Menu

```lua
-- schema/derma/cl_vendor.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(700, 500)
    self:Center()
    self:MakePopup()
    self:SetTitle("Vendor")

    self.items = {}

    -- Item list
    self.itemList = self:Add("DScrollPanel")
    self.itemList:SetPos(10, 40)
    self.itemList:SetSize(self:GetWide() - 20, self:GetTall() - 100)

    -- Buy button
    self.buyButton = self:Add("DButton")
    self.buyButton:SetText("Buy Selected")
    self.buyButton:SetPos(10, self:GetTall() - 50)
    self.buyButton:SetSize(150, 35)
    self.buyButton.DoClick = function()
        if self.selectedItem then
            self:BuyItem(self.selectedItem)
        end
    end

    -- Close button
    self.closeButton = self:Add("DButton")
    self.closeButton:SetText("Close")
    self.closeButton:SetPos(self:GetWide() - 160, self:GetTall() - 50)
    self.closeButton:SetSize(150, 35)
    self.closeButton.DoClick = function()
        self:Close()
    end
end

function PANEL:SetVendorItems(items)
    self.items = items
    self:PopulateItems()
end

function PANEL:PopulateItems()
    self.itemList:Clear()

    for uniqueID, price in pairs(self.items) do
        local itemTable = ix.item.list[uniqueID]

        if itemTable then
            local itemPanel = self.itemList:Add("DButton")
            itemPanel:Dock(TOP)
            itemPanel:SetTall(60)
            itemPanel:DockMargin(0, 0, 0, 5)
            itemPanel:SetText("")

            itemPanel.Paint = function(pnl, w, h)
                local color = pnl == self.selectedItem and Color(80, 120, 180) or Color(50, 50, 50)
                draw.RoundedBox(4, 0, 0, w, h, color)

                -- Draw item name
                draw.SimpleText(itemTable.name, "DermaDefault", 10, 10, Color(255, 255, 255))

                -- Draw price
                draw.SimpleText(ix.currency.Get(price), "DermaDefault", w - 10, 10, Color(100, 255, 100), TEXT_ALIGN_RIGHT)

                -- Draw description
                draw.SimpleText(itemTable.description, "DermaDefaultBold", 10, 30, Color(200, 200, 200))
            end

            itemPanel.DoClick = function()
                self.selectedItem = itemPanel
                self.selectedItemID = uniqueID
                self.selectedPrice = price
            end

            itemPanel.uniqueID = uniqueID
        end
    end
end

function PANEL:BuyItem(itemPanel)
    if not itemPanel.uniqueID then return end

    netstream.Start("ixVendorBuy", itemPanel.uniqueID)
end

function PANEL:Close()
    self:Remove()
end

vgui.Register("ixVendorMenu", PANEL, "DFrame")
```

### Information Terminal

```lua
-- schema/derma/cl_terminal.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(800, 600)
    self:Center()
    self:MakePopup()
    self:SetTitle("Combine Terminal")

    -- Header
    self.header = self:Add("DLabel")
    self.header:SetPos(0, 30)
    self.header:SetSize(self:GetWide(), 40)
    self.header:SetFont("DermaLarge")
    self.header:SetText("COMBINE OVERWATCH - TERMINAL ACCESS")
    self.header:SetContentAlignment(5)
    self.header:SetTextColor(Color(200, 50, 50))

    -- Tab control
    self.tabs = self:Add("DPropertySheet")
    self.tabs:SetPos(10, 80)
    self.tabs:SetSize(self:GetWide() - 20, self:GetTall() - 130)

    -- Citizens tab
    local citizenPanel = vgui.Create("DPanel")
    self.tabs:AddSheet("Citizens", citizenPanel, "icon16/user.png")
    self:CreateCitizenList(citizenPanel)

    -- Alerts tab
    local alertPanel = vgui.Create("DPanel")
    self.tabs:AddSheet("Alerts", alertPanel, "icon16/error.png")
    self:CreateAlertList(alertPanel)

    -- Close button
    self.closeButton = self:Add("DButton")
    self.closeButton:SetText("DISCONNECT")
    self.closeButton:SetPos(self:GetWide() - 160, self:GetTall() - 40)
    self.closeButton:SetSize(150, 30)
    self.closeButton.DoClick = function()
        self:Close()
    end
end

function PANEL:CreateCitizenList(parent)
    local list = parent:Add("DListView")
    list:Dock(FILL)
    list:AddColumn("CID")
    list:AddColumn("Name")
    list:AddColumn("Faction")
    list:AddColumn("Status")

    -- Request citizen data from server
    netstream.Start("ixTerminalRequest", "citizens")

    netstream.Hook("ixTerminalData", function(data)
        list:Clear()

        for _, citizen in ipairs(data) do
            list:AddLine(citizen.cid, citizen.name, citizen.faction, citizen.status)
        end
    end)
end

function PANEL:CreateAlertList(parent)
    local scroll = parent:Add("DScrollPanel")
    scroll:Dock(FILL)

    -- Request alerts from server
    netstream.Start("ixTerminalRequest", "alerts")

    netstream.Hook("ixTerminalAlerts", function(alerts)
        for _, alert in ipairs(alerts) do
            local alertPanel = scroll:Add("DPanel")
            alertPanel:Dock(TOP)
            alertPanel:SetTall(60)
            alertPanel:DockMargin(5, 5, 5, 0)

            alertPanel.Paint = function(pnl, w, h)
                draw.RoundedBox(4, 0, 0, w, h, Color(80, 20, 20))

                draw.SimpleText(alert.title, "DermaDefaultBold", 10, 10, Color(255, 100, 100))
                draw.SimpleText(alert.message, "DermaDefault", 10, 30, Color(255, 255, 255))
                draw.SimpleText(alert.time, "DermaDefault", w - 10, 10, Color(200, 200, 200), TEXT_ALIGN_RIGHT)
            end
        end
    end)
end

function PANEL:Paint(w, h)
    -- Dark background
    draw.RoundedBox(0, 0, 0, w, h, Color(20, 20, 20, 250))

    -- Red border
    surface.SetDrawColor(150, 30, 30)
    surface.DrawOutlinedRect(0, 0, w, h, 2)
end

function PANEL:Close()
    self:Remove()
end

vgui.Register("ixCombineTerminal", PANEL, "DFrame")
```

### Crafting Interface

```lua
-- schema/derma/cl_crafting.lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(900, 600)
    self:Center()
    self:MakePopup()
    self:SetTitle("Crafting")

    -- Recipe list (left)
    self.recipeList = self:Add("DScrollPanel")
    self.recipeList:SetPos(10, 40)
    self.recipeList:SetSize(300, self:GetTall() - 100)

    -- Recipe details (right)
    self.recipeDetails = self:Add("DPanel")
    self.recipeDetails:SetPos(320, 40)
    self.recipeDetails:SetSize(self:GetWide() - 330, self:GetTall() - 100)

    -- Craft button
    self.craftButton = self:Add("DButton")
    self.craftButton:SetText("Craft")
    self.craftButton:SetPos(320, self:GetTall() - 50)
    self.craftButton:SetSize(200, 35)
    self.craftButton:SetEnabled(false)
    self.craftButton.DoClick = function()
        if self.selectedRecipe then
            netstream.Start("ixCraft", self.selectedRecipe.uniqueID)
        end
    end

    -- Close button
    self.closeButton = self:Add("DButton")
    self.closeButton:SetText("Close")
    self.closeButton:SetPos(self:GetWide() - 160, self:GetTall() - 50)
    self.closeButton:SetSize(150, 35)
    self.closeButton.DoClick = function()
        self:Close()
    end

    self:LoadRecipes()
end

function PANEL:LoadRecipes()
    -- In real implementation, get from server
    local recipes = {
        {
            uniqueID = "bandage",
            name = "Bandage",
            description = "Crafted medical bandage",
            result = "item_bandage",
            requirements = {
                {item = "cloth", amount = 2},
                {item = "alcohol", amount = 1}
            }
        },
        -- More recipes...
    }

    for _, recipe in ipairs(recipes) do
        self:AddRecipeButton(recipe)
    end
end

function PANEL:AddRecipeButton(recipe)
    local button = self.recipeList:Add("DButton")
    button:Dock(TOP)
    button:SetTall(80)
    button:DockMargin(0, 0, 0, 5)
    button:SetText("")

    button.Paint = function(pnl, w, h)
        local color = pnl.selected and Color(80, 120, 180) or Color(50, 50, 50)
        draw.RoundedBox(4, 0, 0, w, h, color)

        draw.SimpleText(recipe.name, "DermaDefaultBold", 10, 10, Color(255, 255, 255))
        draw.SimpleText(recipe.description, "DermaDefault", 10, 30, Color(200, 200, 200))

        -- Can craft indicator
        local canCraft = self:CanCraft(recipe)
        local indicatorColor = canCraft and Color(0, 255, 0) or Color(255, 0, 0)
        draw.RoundedBox(4, w - 20, 10, 10, 10, indicatorColor)
    end

    button.DoClick = function()
        self:SelectRecipe(recipe)
    end

    button.recipe = recipe
end

function PANEL:SelectRecipe(recipe)
    self.selectedRecipe = recipe

    -- Update all buttons
    for _, child in ipairs(self.recipeList:GetChildren()) do
        if child.recipe then
            child.selected = (child.recipe == recipe)
        end
    end

    -- Show recipe details
    self:ShowRecipeDetails(recipe)

    -- Enable craft button if can craft
    self.craftButton:SetEnabled(self:CanCraft(recipe))
end

function PANEL:ShowRecipeDetails(recipe)
    self.recipeDetails:Clear()

    -- Title
    local title = self.recipeDetails:Add("DLabel")
    title:SetText(recipe.name)
    title:SetFont("DermaLarge")
    title:Dock(TOP)
    title:DockMargin(10, 10, 10, 5)
    title:SizeToContents()

    -- Description
    local desc = self.recipeDetails:Add("DLabel")
    desc:SetText(recipe.description)
    desc:Dock(TOP)
    desc:DockMargin(10, 0, 10, 10)
    desc:SizeToContents()

    -- Requirements
    local reqLabel = self.recipeDetails:Add("DLabel")
    reqLabel:SetText("Requirements:")
    reqLabel:SetFont("DermaDefaultBold")
    reqLabel:Dock(TOP)
    reqLabel:DockMargin(10, 10, 10, 5)
    reqLabel:SizeToContents()

    for _, req in ipairs(recipe.requirements) do
        local reqPanel = self.recipeDetails:Add("DPanel")
        reqPanel:Dock(TOP)
        reqPanel:SetTall(30)
        reqPanel:DockMargin(10, 0, 10, 2)

        reqPanel.Paint = function(pnl, w, h)
            local hasItem = self:HasItem(req.item, req.amount)
            local color = hasItem and Color(40, 80, 40) or Color(80, 40, 40)
            draw.RoundedBox(4, 0, 0, w, h, color)

            local itemTable = ix.item.list[req.item]
            local itemName = itemTable and itemTable.name or req.item

            draw.SimpleText(itemName .. " x" .. req.amount, "DermaDefault", 10, h / 2, Color(255, 255, 255), TEXT_ALIGN_LEFT, TEXT_ALIGN_CENTER)

            local statusText = hasItem and "✓" or "✗"
            draw.SimpleText(statusText, "DermaDefaultBold", w - 10, h / 2, Color(255, 255, 255), TEXT_ALIGN_RIGHT, TEXT_ALIGN_CENTER)
        end
    end
end

function PANEL:CanCraft(recipe)
    for _, req in ipairs(recipe.requirements) do
        if not self:HasItem(req.item, req.amount) then
            return false
        end
    end
    return true
end

function PANEL:HasItem(itemID, amount)
    local client = LocalPlayer()
    local character = client:GetCharacter()
    if not character then return false end

    local inventory = character:GetInventory()
    local count = 0

    for _, item in pairs(inventory:GetItems()) do
        if item.uniqueID == itemID then
            count = count + (item:GetData("amount", 1))
        end
    end

    return count >= amount
end

function PANEL:Close()
    self:Remove()
end

vgui.Register("ixCraftingMenu", PANEL, "DFrame")
```

## Integrating with Helix

### Opening Menus via Commands

```lua
-- schema/commands/sh_terminal.lua
ix.command.Add("Terminal", {
    description = "Access Combine terminal",
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        return character and character:GetFaction() == FACTION_CP
    end,
    OnRun = function(self, client)
        -- SERVER: Send network message
        if SERVER then
            netstream.Start(client, "ixOpenTerminal")
        end
    end
})

-- CLIENT: Receive and open
netstream.Hook("ixOpenTerminal", function()
    vgui.Create("ixCombineTerminal")
end)
```

### Opening Menus via Entity Use

```lua
-- In entity file
function ENT:Use(activator)
    if activator:IsPlayer() then
        netstream.Start(activator, "ixOpenVendor", {
            items = self.items,
            vendorName = self.vendorName
        })
    end
end

-- CLIENT: Receive and open
netstream.Hook("ixOpenVendor", function(data)
    local menu = vgui.Create("ixVendorMenu")
    menu:SetVendorItems(data.items)
    menu:SetTitle(data.vendorName or "Vendor")
end)
```

## Best Practices

### ✅ DO

- Place derma files in `schema/derma/` folder
- Use `cl_` prefix for client-only files
- Register panels with `vgui.Register()`
- Use consistent theming and colors
- Validate data from server
- Clean up panels with `:Remove()`
- Use `:MakePopup()` for interactive menus
- Handle network disconnects gracefully

### ❌ DON'T

- Don't create derma on server (client-only)
- Don't trust client-side data without validation
- Don't forget to clean up timers/hooks
- Don't hardcode positions (use `:Dock()` when possible)
- Don't bypass Helix styling
- Don't create memory leaks

## Common Patterns

### Network Data Pattern

```lua
-- SERVER: schema/sv_schema.lua
netstream.Hook("ixRequestData", function(client, dataType)
    -- Validate client permission
    local character = client:GetCharacter()
    if not character or character:GetFaction() != FACTION_ADMIN then
        return
    end

    -- Gather data
    local data = {}
    -- ... fill data ...

    -- Send to client
    netstream.Start(client, "ixReceiveData", data)
end)

-- CLIENT: schema/derma/cl_menu.lua
function PANEL:RequestData()
    netstream.Start("ixRequestData", "citizens")
end

netstream.Hook("ixReceiveData", function(data)
    if IsValid(myPanel) then
        myPanel:UpdateData(data)
    end
end)
```

### Helper Function Pattern

```lua
-- schema/cl_schema.lua
function Schema:OpenCustomMenu(title, data)
    local menu = vgui.Create("ixCustomMenu")
    menu:SetTitle(title)
    menu:SetData(data)
    return menu
end

-- Usage anywhere
Schema:OpenCustomMenu("My Menu", {...})
```

## See Also

- [Derma Documentation](https://wiki.facepunch.com/gmod/Category:VGUI) - Garry's Mod Derma reference
- [UI Development Guide](../guides/ui-development.md) - Step-by-step UI guide
- [Helix Derma](../../gamemode/core/derma/) - Framework derma files for reference
- Source: `gamemode/core/derma/`
