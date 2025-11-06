# Complete Derma UI Examples

> **Reference**: `gamemode/core/derma/cl_notice.lua`, `gamemode/core/derma/cl_inventory.lua`

This document provides complete, working examples of Helix Derma (UI) panels from simple to complex.

## ⚠️ Important: Use Helix Derma Best Practices

**Always follow Helix UI conventions** rather than creating inconsistent interfaces. The framework provides:
- Theme system with `derma.GetColor()`
- Animation helpers with `CreateAnimation()`
- Custom panel types (`ixPanel`, `ixButton`, etc.)
- Automatic cleanup on panel removal
- Network integration helpers

**All Derma code is CLIENT-ONLY**. Always wrap in `if CLIENT then ... end`.

## Example 1: Simple Information Panel

The simplest panel - displays text with a close button.

### File Location

**Place in**: `plugins/yourplugin/derma/cl_info.lua` or `schema/derma/cl_info.lua`

### Complete Code

```lua
if CLIENT then
    -- Define custom panel type
    local PANEL = {}

    function PANEL:Init()
        -- Set panel properties
        self:SetSize(400, 200)
        self:Center()
        self:SetTitle("Information")
        self:MakePopup()  -- Focus and accept input

        -- Create text label
        self.label = self:Add("DLabel")
        self.label:Dock(FILL)
        self.label:DockMargin(10, 10, 10, 10)
        self.label:SetText("This is an information panel")
        self.label:SetFont("ixMediumFont")
        self.label:SetContentAlignment(5)  -- Center
        self.label:SetTextColor(color_white)
        self.label:SetWrap(true)
        self.label:SetAutoStretchVertical(true)

        -- Create close button
        self.closeButton = self:Add("ixButton")
        self.closeButton:Dock(BOTTOM)
        self.closeButton:SetText("Close")
        self.closeButton:SizeToContents()
        self.closeButton.DoClick = function()
            self:Close()
        end
    end

    function PANEL:SetInformationText(text)
        self.label:SetText(text)
    end

    -- Register panel type
    vgui.Register("ixInfoPanel", PANEL, "ixFrame")

    -- Helper function to create the panel
    function ix.ShowInfoPanel(text)
        local panel = vgui.Create("ixInfoPanel")
        panel:SetInformationText(text)
        return panel
    end
end
```

### Usage

```lua
-- Create panel from console command
concommand.Add("test_info", function()
    ix.ShowInfoPanel("This is a test message!")
end)

-- Create panel from hook
hook.Add("PlayerButtonDown", "TestInfo", function(ply, button)
    if button == KEY_F3 then
        ix.ShowInfoPanel("You pressed F3!")
    end
end)
```

### Key Points

- **`if CLIENT`**: All Derma code is client-only
- **`vgui.Register()`**: Register custom panel types
- **`ixFrame`**: Use Helix base panel for theming
- **`MakePopup()`**: Focus panel and accept input
- **`Dock()`**: Use docking for layout

## Example 2: Interactive Menu with Buttons

A menu with multiple options.

### Complete Code

```lua
if CLIENT then
    local PANEL = {}

    function PANEL:Init()
        self:SetSize(500, 400)
        self:Center()
        self:SetTitle("Admin Menu")
        self:MakePopup()

        -- Create button container
        self.buttonList = self:Add("DScrollPanel")
        self.buttonList:Dock(FILL)
        self.buttonList:DockMargin(10, 10, 10, 10)

        -- Add buttons
        self:AddMenuButton("Teleport to Player", "icon16/user_go.png", function()
            Derma_StringRequest(
                "Teleport",
                "Enter player name:",
                "",
                function(text)
                    RunConsoleCommand("say", "/tp " .. text)
                    self:Close()
                end
            )
        end)

        self:AddMenuButton("Spawn Item", "icon16/package.png", function()
            Derma_StringRequest(
                "Spawn Item",
                "Enter item ID:",
                "",
                function(text)
                    RunConsoleCommand("say", "/giveitem " .. text)
                    self:Close()
                end
            )
        end)

        self:AddMenuButton("Heal Self", "icon16/heart.png", function()
            RunConsoleCommand("say", "/heal")
            self:Close()
        end)

        self:AddMenuButton("Return to Spawn", "icon16/house.png", function()
            RunConsoleCommand("say", "/return")
            self:Close()
        end)
    end

    function PANEL:AddMenuButton(text, icon, callback)
        local button = self.buttonList:Add("ixButton")
        button:Dock(TOP)
        button:DockMargin(5, 5, 5, 0)
        button:SetText(text)
        button:SetFont("ixMediumFont")
        button:SizeToContents()

        -- Add icon
        if icon then
            button:SetIcon(icon)
        end

        button.DoClick = callback
    end

    vgui.Register("ixAdminMenu", PANEL, "ixFrame")

    -- Bind to key
    concommand.Add("ix_adminmenu", function()
        if IsValid(ix.gui.adminMenu) then
            ix.gui.adminMenu:Remove()
        end

        ix.gui.adminMenu = vgui.Create("ixAdminMenu")
    end)
end

-- Bind key (in cl_hooks.lua or similar)
if CLIENT then
    hook.Add("PlayerButtonDown", "ixAdminMenuBind", function(ply, button)
        if button == KEY_F2 and LocalPlayer():IsAdmin() then
            RunConsoleCommand("ix_adminmenu")
        end
    end)
end
```

### Key Points

- **`DScrollPanel`**: Scrollable container for many items
- **`Dock(TOP)`**: Stack buttons vertically
- **`Derma_StringRequest()`**: Built-in input dialog
- **`ix.gui.adminMenu`**: Store reference for cleanup
- **`IsValid()`**: Check if panel exists before recreating

## Example 3: Form with Input Fields

A panel that collects user input.

### Complete Code

```lua
if CLIENT then
    local PANEL = {}

    function PANEL:Init()
        self:SetSize(450, 350)
        self:Center()
        self:SetTitle("Character Biography")
        self:MakePopup()

        -- Name field
        self.nameLabel = self:Add("DLabel")
        self.nameLabel:Dock(TOP)
        self.nameLabel:DockMargin(10, 10, 10, 0)
        self.nameLabel:SetText("Character Name:")
        self.nameLabel:SetFont("ixSmallFont")

        self.nameEntry = self:Add("ixTextEntry")
        self.nameEntry:Dock(TOP)
        self.nameEntry:DockMargin(10, 5, 10, 10)
        self.nameEntry:SetPlaceholderText("Enter name...")

        -- Age field
        self.ageLabel = self:Add("DLabel")
        self.ageLabel:Dock(TOP)
        self.ageLabel:DockMargin(10, 0, 10, 0)
        self.ageLabel:SetText("Age:")
        self.ageLabel:SetFont("ixSmallFont")

        self.ageEntry = self:Add("ixTextEntry")
        self.ageEntry:Dock(TOP)
        self.ageEntry:DockMargin(10, 5, 10, 10)
        self.ageEntry:SetPlaceholderText("Enter age...")
        self.ageEntry:SetNumeric(true)  -- Only numbers

        -- Description field
        self.descLabel = self:Add("DLabel")
        self.descLabel:Dock(TOP)
        self.descLabel:DockMargin(10, 0, 10, 0)
        self.descLabel:SetText("Description:")
        self.descLabel:SetFont("ixSmallFont")

        self.descEntry = self:Add("DTextEntry")
        self.descEntry:Dock(FILL)
        self.descEntry:DockMargin(10, 5, 10, 10)
        self.descEntry:SetMultiline(true)

        -- Submit button
        self.submitButton = self:Add("ixButton")
        self.submitButton:Dock(BOTTOM)
        self.submitButton:DockMargin(10, 0, 10, 10)
        self.submitButton:SetText("Submit")
        self.submitButton.DoClick = function()
            self:OnSubmit()
        end
    end

    function PANEL:OnSubmit()
        local name = self.nameEntry:GetText()
        local age = tonumber(self.ageEntry:GetText())
        local description = self.descEntry:GetText()

        -- Validate input
        if name == "" or #name < 3 then
            Derma_Message("Name must be at least 3 characters", "Error", "OK")
            return
        end

        if !age or age < 18 or age > 100 then
            Derma_Message("Age must be between 18 and 100", "Error", "OK")
            return
        end

        if description == "" or #description < 20 then
            Derma_Message("Description must be at least 20 characters", "Error", "OK")
            return
        end

        -- Send to server
        net.Start("ixSetCharacterBio")
            net.WriteString(name)
            net.WriteUInt(age, 8)
            net.WriteString(description)
        net.SendToServer()

        self:Close()
    end

    vgui.Register("ixBiographyPanel", PANEL, "ixFrame")
end

-- Server-side receiver
if SERVER then
    util.AddNetworkString("ixSetCharacterBio")

    net.Receive("ixSetCharacterBio", function(len, client)
        local name = net.ReadString()
        local age = net.ReadUInt(8)
        local description = net.ReadString()

        local character = client:GetCharacter()

        if !character then return end

        -- Validate server-side too (never trust client!)
        if #name < 3 or #name > 50 then return end
        if age < 18 or age > 100 then return end
        if #description < 20 or #description > 1000 then return end

        -- Save data
        character:SetData("bioName", name)
        character:SetData("bioAge", age)
        character:SetData("bioDesc", description)

        client:Notify("Biography updated successfully")
    end)
end
```

### Key Points

- **Validation**: Check input on both client AND server
- **`SetNumeric()`**: Restrict to numbers only
- **`SetMultiline()`**: Multi-line text entry
- **Network**: Send data to server with `net` library
- **Never trust client**: Always validate on server

## Example 4: List Panel with Scrolling

Display a scrollable list of items.

### Complete Code

```lua
if CLIENT then
    local PANEL = {}

    function PANEL:Init()
        self:SetSize(600, 500)
        self:Center()
        self:SetTitle("Player List")
        self:MakePopup()

        -- Search box
        self.searchEntry = self:Add("ixTextEntry")
        self.searchEntry:Dock(TOP)
        self.searchEntry:DockMargin(10, 10, 10, 5)
        self.searchEntry:SetPlaceholderText("Search players...")
        self.searchEntry.OnChange = function(this)
            self:PopulateList(this:GetText())
        end

        -- Player list
        self.playerList = self:Add("DScrollPanel")
        self.playerList:Dock(FILL)
        self.playerList:DockMargin(10, 5, 10, 10)

        -- Populate initially
        self:PopulateList()
    end

    function PANEL:PopulateList(search)
        search = search and search:lower() or ""

        -- Clear existing
        self.playerList:Clear()

        -- Add players
        for _, ply in ipairs(player.GetAll()) do
            local name = ply:Name()

            -- Filter by search
            if search != "" and !name:lower():find(search, 1, true) then
                continue
            end

            local playerPanel = self.playerList:Add("DPanel")
            playerPanel:Dock(TOP)
            playerPanel:DockMargin(0, 0, 0, 5)
            playerPanel:SetTall(60)

            -- Background
            playerPanel.Paint = function(this, w, h)
                surface.SetDrawColor(0, 0, 0, 100)
                surface.DrawRect(0, 0, w, h)

                surface.SetDrawColor(255, 255, 255, 50)
                surface.DrawOutlinedRect(0, 0, w, h)
            end

            -- Player model
            local modelPanel = playerPanel:Add("SpawnIcon")
            modelPanel:SetPos(5, 5)
            modelPanel:SetSize(50, 50)
            modelPanel:SetModel(ply:GetModel())

            -- Player name
            local nameLabel = playerPanel:Add("DLabel")
            nameLabel:SetPos(65, 5)
            nameLabel:SetText(name)
            nameLabel:SetFont("ixMediumFont")
            nameLabel:SizeToContents()

            -- Player info
            local character = ply:GetCharacter()
            local infoText = "No Character"

            if character then
                local faction = ix.faction.indices[character:GetFaction()]
                infoText = faction.name .. " - Level " .. (character:GetData("level", 1))
            end

            local infoLabel = playerPanel:Add("DLabel")
            infoLabel:SetPos(65, 28)
            infoLabel:SetText(infoText)
            infoLabel:SetFont("ixSmallFont")
            infoLabel:SizeToContents()

            -- Action buttons
            local teleportButton = playerPanel:Add("ixButton")
            teleportButton:SetPos(400, 10)
            teleportButton:SetSize(80, 20)
            teleportButton:SetText("Teleport")
            teleportButton.DoClick = function()
                RunConsoleCommand("say", "/tp \"" .. name .. "\"")
            end

            local kickButton = playerPanel:Add("ixButton")
            kickButton:SetPos(490, 10)
            kickButton:SetSize(80, 20)
            kickButton:SetText("Kick")
            kickButton.DoClick = function()
                Derma_StringRequest(
                    "Kick Player",
                    "Enter reason:",
                    "",
                    function(reason)
                        RunConsoleCommand("say", "/kick \"" .. name .. "\" " .. reason)
                        self:PopulateList(search)
                    end
                )
            end
        end
    end

    vgui.Register("ixPlayerListPanel", PANEL, "ixFrame")
end
```

### Key Points

- **Search functionality**: Filter list dynamically
- **`DPanel`**: Generic container for custom layouts
- **`Paint()`**: Custom drawing for panels
- **`SpawnIcon`**: Display player models
- **`Clear()`**: Remove all children from panel

## Example 5: Animated Panel with Progress Bar

A panel with animations and progress tracking.

### Complete Code

```lua
if CLIENT then
    local PANEL = {}

    function PANEL:Init()
        self:SetSize(400, 150)
        self:Center()
        self:SetTitle("Loading...")
        self:ShowCloseButton(false)
        self:MakePopup()

        -- Progress label
        self.progressLabel = self:Add("DLabel")
        self.progressLabel:Dock(TOP)
        self.progressLabel:DockMargin(10, 10, 10, 5)
        self.progressLabel:SetText("Initializing...")
        self.progressLabel:SetFont("ixMediumFont")
        self.progressLabel:SetContentAlignment(5)

        -- Progress bar
        self.progressBar = self:Add("DProgress")
        self.progressBar:Dock(TOP)
        self.progressBar:DockMargin(10, 5, 10, 10)
        self.progressBar:SetFraction(0)

        -- Status label
        self.statusLabel = self:Add("DLabel")
        self.statusLabel:Dock(FILL)
        self.statusLabel:DockMargin(10, 5, 10, 10)
        self.statusLabel:SetText("")
        self.statusLabel:SetFont("ixSmallFont")
        self.statusLabel:SetContentAlignment(5)
        self.statusLabel:SetTextColor(Color(150, 150, 150))

        self.currentProgress = 0
        self.targetProgress = 0
    end

    function PANEL:SetProgress(fraction, status)
        self.targetProgress = math.Clamp(fraction, 0, 1)

        if status then
            self.statusLabel:SetText(status)
        end

        -- Smooth animation
        self:CreateAnimation(0.3, {
            target = {currentProgress = self.targetProgress},
            easing = "outQuint"
        })
    end

    function PANEL:Think()
        -- Update progress bar smoothly
        self.progressBar:SetFraction(self.currentProgress)

        -- Update label
        local percent = math.Round(self.currentProgress * 100)
        self.progressLabel:SetText("Loading... " .. percent .. "%")

        -- Auto-close when complete
        if self.currentProgress >= 1 and !self.bCompleted then
            self.bCompleted = true

            timer.Simple(1, function()
                if IsValid(self) then
                    self:Remove()
                end
            end)
        end
    end

    vgui.Register("ixLoadingPanel", PANEL, "ixFrame")

    -- Example usage
    concommand.Add("test_loading", function()
        local panel = vgui.Create("ixLoadingPanel")

        -- Simulate loading steps
        timer.Simple(0.5, function()
            if IsValid(panel) then
                panel:SetProgress(0.25, "Loading assets...")
            end
        end)

        timer.Simple(1.5, function()
            if IsValid(panel) then
                panel:SetProgress(0.5, "Connecting to server...")
            end
        end)

        timer.Simple(2.5, function()
            if IsValid(panel) then
                panel:SetProgress(0.75, "Syncing data...")
            end
        end)

        timer.Simple(3.5, function()
            if IsValid(panel) then
                panel:SetProgress(1.0, "Complete!")
            end
        end)
    end)
end
```

### Key Points

- **`CreateAnimation()`**: Smooth transitions
- **`Think()`**: Called every frame
- **Progress bar**: Visual feedback for operations
- **Auto-close**: Remove panel when done
- **`IsValid()`**: Check panel exists in timers

## ⚠️ Do NOT

```lua
-- WRONG: Don't forget CLIENT check
local PANEL = {}  -- Will run on server too!

-- WRONG: Don't create global tables
MyPanel = vgui.Create("DFrame")  -- Pollutes global scope!

-- WRONG: Don't forget to validate input
function PANEL:OnSubmit()
    local text = self.entry:GetText()
    RunConsoleCommand("say", "/command " .. text)  -- No validation!
end

-- WRONG: Don't trust client data on server
net.Receive("SetMoney", function(len, client)
    local amount = net.ReadInt(32)
    client:GetCharacter():SetMoney(amount)  -- Client can send any amount!
end)

-- WRONG: Don't forget cleanup
function PANEL:DoStuff()
    timer.Create("MyTimer", 1, 0, function()
        -- Timer continues after panel removed!
    end)
end
-- Should remove timer in OnRemove
```

## Best Practices

### ✅ DO

- Wrap all Derma code in `if CLIENT`
- Use `vgui.Register()` for reusable panels
- Use `ixFrame`, `ixButton`, `ixTextEntry` for themed panels
- Validate input on both client AND server
- Use `IsValid()` before accessing panels in callbacks
- Clean up timers in `OnRemove()`
- Use `Dock()` for responsive layouts
- Store panel references in `ix.gui.*`
- Use `MakePopup()` to focus panels
- Close panels with `:Remove()` not `:Close()`

### ❌ DON'T

- Don't forget `if CLIENT` wrapper
- Don't create globals for panels
- Don't trust client input on server
- Don't forget to validate user input
- Don't create timers without cleanup
- Don't use fixed positions (use `Dock()`)
- Don't forget `IsValid()` in timers/callbacks
- Don't send excessive network messages
- Don't create memory leaks

## Advanced Tips

### Custom Paint Function

```lua
function PANEL:Paint(w, h)
    -- Custom background
    draw.RoundedBox(8, 0, 0, w, h, Color(0, 0, 0, 200))

    -- Border
    surface.SetDrawColor(255, 255, 255, 100)
    surface.DrawOutlinedRect(0, 0, w, h, 2)
end
```

### Cleanup on Remove

```lua
function PANEL:OnRemove()
    -- Remove timers
    timer.Remove("MyPanelTimer_" .. self:GetCreationID())

    -- Close child panels
    if IsValid(self.childPanel) then
        self.childPanel:Remove()
    end

    -- Clear references
    ix.gui.myPanel = nil
end
```

### Network Integration

```lua
-- CLIENT: Request data from server
net.Start("ixRequestData")
net.SendToServer()

-- SERVER: Send data to client
net.Receive("ixRequestData", function(len, client)
    net.Start("ixSendData")
        net.WriteTable(someData)
    net.Send(client)
end)

-- CLIENT: Receive and display
net.Receive("ixSendData", function()
    local data = net.ReadTable()

    local panel = vgui.Create("ixDataPanel")
    panel:SetData(data)
end)
```

## See Also

- [Derma Overview](../ui/derma-overview.md) - Complete UI system
- [HUD System](../ui/hud.md) - Drawing on HUD
- [Notifications](../ui/notifications.md) - Notice system
- [Menu System](../ui/menus.md) - Menu integration
- Source: `gamemode/core/derma/cl_notice.lua` - Notice panels
- Source: `gamemode/core/derma/cl_inventory.lua` - Complex UI example
