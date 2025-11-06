# UI Development with Derma

> **Reference**: `gamemode/core/derma/`

Practical guide to creating custom user interfaces using Garry's Mod's Derma (VGUI) system in Helix.

## ⚠️ Important: Client-Side Only

**UI code runs on CLIENT only**. Always wrap in `if CLIENT then` blocks.

## Basic Panel Creation

### Simple Frame

```lua
if CLIENT then
    function OpenMyMenu()
        local frame = vgui.Create("DFrame")
        frame:SetTitle("My Menu")
        frame:SetSize(400, 300)
        frame:Center()
        frame:MakePopup()

        -- Add content
        local button = frame:Add("DButton")
        button:SetText("Click Me")
        button:Dock(BOTTOM)
        button.DoClick = function()
            print("Button clicked!")
        end
    end
end
```

### Common Controls

```lua
-- Button
local button = vgui.Create("DButton", parent)
button:SetText("Click")
button:SetPos(10, 10)
button:SetSize(100, 30)
button.DoClick = function()
    print("Clicked!")
end

-- Text Entry
local textEntry = vgui.Create("DTextEntry", parent)
textEntry:SetPos(10, 50)
textEntry:SetSize(200, 25)
textEntry:SetPlaceholderText("Enter text...")
textEntry.OnEnter = function(self)
    print("Entered:", self:GetValue())
end

-- Label
local label = vgui.Create("DLabel", parent)
label:SetText("Hello World")
label:SetPos(10, 90)
label:SizeToContents()

-- Checkbox
local checkbox = vgui.Create("DCheckBoxLabel", parent)
checkbox:SetText("Enable Feature")
checkbox:SetPos(10, 120)
checkbox:SizeToContents()
checkbox.OnChange = function(self, value)
    print("Checked:", value)
end

-- Slider
local slider = vgui.Create("DNumSlider", parent)
slider:SetPos(10, 150)
slider:SetSize(200, 25)
slider:SetMin(0)
slider:SetMax(100)
slider:SetDecimals(0)
slider.OnValueChanged = function(self, value)
    print("Value:", value)
end
```

## Practical Examples

### Example 1: Character Menu

```lua
if CLIENT then
    function OpenCharacterMenu()
        local frame = vgui.Create("DFrame")
        frame:SetTitle("Character Info")
        frame:SetSize(300, 400)
        frame:Center()
        frame:MakePopup()

        -- Character name
        local nameLabel = frame:Add("DLabel")
        nameLabel:SetText("Name: " .. LocalPlayer():GetCharacter():GetName())
        nameLabel:Dock(TOP)
        nameLabel:DockMargin(10, 10, 10, 0)

        -- Money
        local moneyLabel = frame:Add("DLabel")
        moneyLabel:SetText("Money: $" .. LocalPlayer():GetCharacter():GetMoney())
        moneyLabel:Dock(TOP)
        moneyLabel:DockMargin(10, 5, 10, 0)

        -- Inventory button
        local invButton = frame:Add("DButton")
        invButton:SetText("Open Inventory")
        invButton:Dock(TOP)
        invButton:DockMargin(10, 10, 10, 0)
        invButton:SetTall(30)
        invButton.DoClick = function()
            RunConsoleCommand("ix_showinv")
        end
    end
end
```

### Example 2: Vendor Interface

```lua
if CLIENT then
    function OpenVendorShop(items)
        local frame = vgui.Create("DFrame")
        frame:SetTitle("Shop")
        frame:SetSize(500, 600)
        frame:Center()
        frame:MakePopup()

        -- Item list
        local list = frame:Add("DScrollPanel")
        list:Dock(FILL)
        list:DockMargin(5, 5, 5, 50)

        for _, itemData in ipairs(items) do
            local item = list:Add("DPanel")
            item:Dock(TOP)
            item:DockMargin(0, 0, 0, 5)
            item:SetTall(60)

            -- Name
            local name = item:Add("DLabel")
            name:SetText(itemData.name)
            name:SetFont("DermaLarge")
            name:Dock(TOP)
            name:DockMargin(10, 5, 10, 0)

            -- Price
            local price = item:Add("DLabel")
            price:SetText("$" .. itemData.price)
            price:Dock(TOP)
            price:DockMargin(10, 0, 10, 5)

            -- Buy button
            local buy = item:Add("DButton")
            buy:SetText("Buy")
            buy:Dock(RIGHT)
            buy:SetWide(80)
            buy:DockMargin(5, 5, 5, 5)
            buy.DoClick = function()
                net.Start("PurchaseItem")
                net.WriteString(itemData.id)
                net.SendToServer()
                frame:Close()
            end

            item.Paint = function(self, w, h)
                draw.RoundedBox(4, 0, 0, w, h, Color(50, 50, 50))
            end
        end
    end
end
```

### Example 3: Custom Paint

```lua
if CLIENT then
    local PANEL = {}

    function PANEL:Init()
        self:SetSize(200, 100)
    end

    function PANEL:Paint(w, h)
        -- Background
        draw.RoundedBox(8, 0, 0, w, h, Color(30, 30, 30, 250))

        -- Border
        draw.RoundedBox(8, 0, 0, w, h, Color(100, 150, 200, 255))
        draw.RoundedBox(8, 2, 2, w - 4, h - 4, Color(30, 30, 30, 250))

        -- Text
        draw.SimpleText(
            "Custom Panel",
            "DermaDefault",
            w / 2, h / 2,
            color_white,
            TEXT_ALIGN_CENTER,
            TEXT_ALIGN_CENTER
        )
    end

    vgui.Register("MyCustomPanel", PANEL, "DPanel")

    -- Usage
    local panel = vgui.Create("MyCustomPanel")
end
```

## Helix Integration

### Using ix.gui

```lua
-- Register for automatic cleanup
ix.gui.myMenu = vgui.Create("DFrame")

-- Access later
if IsValid(ix.gui.myMenu) then
    ix.gui.myMenu:Close()
end
```

### Network-Triggered UI

```lua
-- SERVER
util.AddNetworkString("OpenMyMenu")

function ShowMenuToPlayer(client, data)
    net.Start("OpenMyMenu")
    net.WriteTable(data)
    net.Send(client)
end

-- CLIENT
net.Receive("OpenMyMenu", function()
    local data = net.ReadTable()
    OpenMyMenu(data)
end)
```

## Layout Methods

```lua
-- Docking
button:Dock(TOP)      -- Dock to top
button:Dock(BOTTOM)   -- Dock to bottom
button:Dock(LEFT)     -- Dock to left
button:Dock(RIGHT)    -- Dock to right
button:Dock(FILL)     -- Fill remaining space

-- Margins
button:DockMargin(left, top, right, bottom)
button:DockMargin(10, 5, 10, 5)

-- Padding
button:DockPadding(left, top, right, bottom)

-- Manual positioning
button:SetPos(x, y)
button:SetSize(width, height)
button:Center()  -- Center in parent
button:CenterHorizontal()
button:CenterVertical()
```

## Best Practices

### ✅ DO
- Store panels in `ix.gui` for cleanup
- Use `:IsValid()` before accessing panels
- Use `:Dock()` for responsive layouts
- Clean up with `:Remove()` when done
- Use `MakePopup()` for input capture

### ❌ DON'T
- Don't create UI in loops/hooks
- Don't forget CLIENT checks
- Don't hard-code positions (use dock)
- Don't create without cleanup
- Don't forget to call `:MakePopup()`

## See Also

- [Networking Guide](networking.md)
- [Helix Derma Components](../ui/derma-overview.md)
