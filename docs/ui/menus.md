# Menu System (ix.menu)

> **Reference**: `gamemode/core/libs/sh_menu.lua`, `gamemode/core/derma/cl_menu.lua`, `gamemode/core/derma/cl_entitymenu.lua`

The menu system provides two types of interactive menus: the main F1 menu with tabs for inventory, settings, and other features, and context menus for entity interactions.

## ⚠️ Important: Use Built-in Menu Functions

**Always use Helix's built-in menu system** rather than creating custom UI from scratch. The framework provides:
- Main tabbed menu (F1) with automatic tab registration
- Entity interaction context menus
- Automatic network synchronization
- Built-in animations and transitions
- Integration with permissions system (CAMI)
- Subpanel management and lifecycle hooks

## Core Concepts

### What are the Menu Systems?

Helix provides two distinct menu systems:

1. **Main Menu (ixMenu)** - The tabbed F1 menu for inventory, character info, settings, etc.
2. **Entity Menu (ixEntityMenu)** - Context menus that appear when interacting with entities

### Key Terms

- **Tab** - A button and associated panel in the main menu
- **Subpanel** - The content area displayed when a tab is selected
- **Entity Menu** - Pop-up option list for entity interactions
- **Menu Options** - Table of text/callback pairs for entity menus
- **Hook Integration** - Automatic registration through `CreateMenuButtons` hook

## Main Menu System (F1 Menu)

### Opening the Main Menu

**Reference**: `gamemode/core/derma/cl_menu.lua:10`

The main menu opens automatically when pressing F1, but can be created manually:

```lua
-- Open main menu programmatically
local menu = vgui.Create("ixMenu")
-- Menu automatically calls MakePopup() and handles input
```

### Adding a Tab to Main Menu

**Reference**: `gamemode/core/derma/cl_menu.lua:309-345`

Use the `CreateMenuButtons` hook to register tabs:

```lua
-- Simple tab with function
hook.Add("CreateMenuButtons", "MyPlugin", function(tabs)
    tabs["mytab"] = function(container)
        -- Container is the subpanel where you add your UI
        local label = container:Add("DLabel")
        label:SetText("Hello World!")
        label:Dock(TOP)
        label:SetFont("ixMediumFont")
    end
end)
```

**Advanced tab with table structure:**

```lua
hook.Add("CreateMenuButtons", "MyAdvancedTab", function(tabs)
    tabs["advanced"] = {
        -- Set as default tab
        bDefault = false,

        -- Custom button color
        buttonColor = Color(100, 150, 200),

        -- Hide background blur
        bHideBackground = false,

        -- Called once when tab is created
        Create = function(info, container)
            local panel = container:Add("DScrollPanel")
            panel:Dock(FILL)

            -- Store reference for later access
            container.myPanel = panel
        end,

        -- Called when tab is selected
        OnSelected = function(info, container)
            -- Refresh data when tab opens
            container.myPanel:Populate()
        end,

        -- Called when switching to different tab
        OnDeselected = function(info, container)
            -- Cleanup or save state
        end,

        -- Called when tab button is created
        PopulateTabButton = function(info, button)
            -- Customize the button appearance
            button:SetFont("ixMenuButtonHugeFont")
        end
    }
end)
```

### Tab Structure Properties

**Reference**: `gamemode/core/derma/cl_menu.lua:245-308`

| Property | Type | Description |
|----------|------|-------------|
| `Create` | function | Called once to create panel content |
| `OnSelected` | function | Called when tab is selected |
| `OnDeselected` | function | Called when switching away |
| `PopulateTabButton` | function | Customize tab button |
| `bDefault` | boolean | Set as default selected tab |
| `bHideBackground` | boolean | Hide menu background blur |
| `buttonColor` | Color | Custom tab button color |

### Conditional Tab Registration

```lua
-- Only show tab if player has permission
hook.Add("CreateMenuButtons", "AdminTab", function(tabs)
    if not CAMI.PlayerHasAccess(LocalPlayer(), "Helix - Admin Menu", nil) then
        return  -- Don't add tab
    end

    tabs["admin"] = function(container)
        -- Admin panel content
    end
end)

-- Only show tab if character has data
hook.Add("CreateMenuButtons", "ConditionalTab", function(tabs)
    local character = LocalPlayer():GetCharacter()

    if not character or not character:GetData("hasQuest", false) then
        return
    end

    tabs["quests"] = function(container)
        -- Quest panel
    end
end)
```

## Entity Menu System

### ix.menu.Open()

**Reference**: `gamemode/core/libs/sh_menu.lua:34`

Opens a context menu with interaction options for an entity.

```lua
ix.menu.Open(options, entity)
```

**Parameters**:
- `options` (table) - Menu options as `{text = callback}` pairs
- `entity` (Entity, optional) - Entity to associate with menu

**Returns**: `boolean` - Success (fails if menu already open)

```lua
-- CLIENT-SIDE: Open entity menu
local options = {
    ["Open Door"] = function()
        print("Opening door...")
        return true  -- Network choice to server
    end,

    ["Lock Door"] = function()
        print("Locking door...")
        return true
    end,

    ["Cancel"] = function()
        return false  -- Don't send to server
    end
}

ix.menu.Open(options, doorEntity)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom context menus manually
local menu = DermaMenu()
menu:AddOption("Option", function() end)
menu:Open()
-- This won't integrate with framework or network properly!
```

### ix.menu.IsOpen()

**Reference**: `gamemode/core/libs/sh_menu.lua:49`

Checks if an entity menu is currently displayed.

```lua
if ix.menu.IsOpen() then
    print("Entity menu is open")
end
```

### ix.menu.NetworkChoice()

**Reference**: `gamemode/core/libs/sh_menu.lua:58`

Sends menu selection to server (usually called automatically).

```lua
ix.menu.NetworkChoice(entity, choice, data)
```

**Parameters**:
- `entity` (Entity) - Entity to send choice for
- `choice` (string) - Option text that was selected
- `data` (any) - Extra data to send with choice

```lua
-- Usually called automatically, but can be used manually
ix.menu.NetworkChoice(entity, "Open Door", {locked = false})
```

## Entity Interaction System

### Defining Entity Options

**Reference**: `gamemode/core/libs/sh_menu.lua:98-103`

Entities and players can define interaction options:

```lua
-- SHARED: On an entity
ENT.PrintName = "Storage Box"

function ENT:ShowPlayerInteraction(client)
    -- Return table of options
    return {
        ["Open"] = function(ent, client)
            if IsValid(ent) and IsValid(client) then
                -- Open storage logic
                ent:OpenStorage(client)
            end
        end,

        ["Lock"] = function(ent, client)
            ent:SetLocked(true)
            client:Notify("Box locked!")
        end
    }
end
```

### Server-Side Callbacks

**Reference**: `gamemode/core/libs/sh_menu.lua:70-91`

When a client selects an option, the server receives it:

```lua
-- ENTITY: Method 1 - OnOptionSelected
function ENT:OnOptionSelected(client, option, data)
    if option == "Open" then
        self:OpenForPlayer(client)
    elseif option == "Lock" then
        self:ToggleLock(client)
    end
end

-- ENTITY: Method 2 - OnSelect<Option> (removes spaces)
function ENT:OnSelectOpenDoor(client, data)
    -- Called when "Open Door" option is selected
    self:Fire("Open")
    client:Notify("Door opened!")
end

function ENT:OnSelectLockDoor(client, data)
    -- Called when "Lock Door" option is selected
    self:Fire("Lock")
end
```

### Player Entity Menus

**Reference**: `gamemode/core/libs/sh_menu.lua:98-103`

```lua
-- CLIENT-SIDE: Add options to player interaction menu
hook.Add("GetPlayerEntityMenu", "MyPlugin", function(target, options)
    -- target is the player being interacted with
    -- options is the table to add to

    options["Trade"] = function()
        print("Trading with " .. target:Name())
        return true
    end

    options["Give Item"] = function()
        -- Open item selection
        return false  -- Handle locally, don't network
    end
end)

-- SERVER-SIDE: Handle player option selection
hook.Add("OnPlayerOptionSelected", "MyPlugin", function(target, client, option)
    if option == "Trade" then
        -- Start trade between client and target
        StartTrade(client, target)
    end
end)
```

## Complete Example: Storage Container

```lua
-- SHARED: Storage entity definition
ENT.Type = "anim"
ENT.PrintName = "Storage Container"
ENT.Category = "Helix"
ENT.Spawnable = true
ENT.AdminOnly = true

function ENT:SetupDataTables()
    self:NetworkVar("Bool", 0, "Locked")
end

-- Show interaction options
function ENT:ShowPlayerInteraction(client)
    local options = {}

    -- Only show unlock if locked
    if self:GetLocked() then
        options["Unlock"] = function()
            return true  -- Network to server
        end
    else
        options["Open"] = function()
            return true
        end

        options["Lock"] = function()
            return true
        end
    end

    return options
end

-- SERVER: Handle option callbacks
function ENT:OnSelectOpen(client, data)
    if not self:GetLocked() then
        -- Open inventory interface
        local inventory = self:GetInventory()
        if inventory then
            inventory:Sync(client)
        end
    end
end

function ENT:OnSelectLock(client, data)
    self:SetLocked(true)
    client:Notify("Container locked!")
end

function ENT:OnSelectUnlock(client, data)
    -- Check if player has key
    if client:GetCharacter():HasItem("key_storage") then
        self:SetLocked(false)
        client:Notify("Container unlocked!")
    else
        client:Notify("You don't have the key!")
    end
end
```

## Complete Example: Custom Admin Tab

```lua
-- CLIENT-SIDE: Register admin management tab
hook.Add("CreateMenuButtons", "AdminManagement", function(tabs)
    -- Check permission
    if not CAMI.PlayerHasAccess(LocalPlayer(), "Helix - Admin", nil) then
        return
    end

    tabs["admin"] = {
        buttonColor = Color(200, 50, 50),

        Create = function(info, container)
            -- Title
            local title = container:Add("DLabel")
            title:SetText("Admin Tools")
            title:SetFont("ixMediumFont")
            title:Dock(TOP)
            title:DockMargin(8, 8, 8, 16)
            title:SetContentAlignment(5)

            -- Player list
            local scroll = container:Add("DScrollPanel")
            scroll:Dock(FILL)

            container.playerList = scroll

            -- Refresh button
            local refresh = container:Add("DButton")
            refresh:SetText("Refresh Players")
            refresh:Dock(BOTTOM)
            refresh:SetTall(40)
            refresh.DoClick = function()
                container:PopulatePlayers()
            end

            -- Initial population
            container:PopulatePlayers()
        end,

        OnSelected = function(info, container)
            -- Refresh when tab is opened
            if container.PopulatePlayers then
                container:PopulatePlayers()
            end
        end
    }
end)

-- Helper function to populate player list
function PopulatePlayers(container)
    container.playerList:Clear()

    for _, ply in ipairs(player.GetAll()) do
        local panel = container.playerList:Add("DButton")
        panel:SetText(ply:Name())
        panel:Dock(TOP)
        panel:DockMargin(4, 4, 4, 4)
        panel.DoClick = function()
            -- Show player options
            ShowPlayerActions(ply)
        end
    end
end
```

## Menu Hooks

### CreateMenuButtons

**Reference**: `gamemode/core/derma/cl_menu.lua:313`

Called to populate main menu tabs.

```lua
hook.Add("CreateMenuButtons", "MyTabs", function(tabs)
    tabs["mytab"] = function(container)
        -- Create UI
    end
end)
```

### MenuSubpanelCreated

**Reference**: `gamemode/core/derma/cl_menu.lua:119`

Called after a subpanel is created and populated.

```lua
hook.Add("MenuSubpanelCreated", "MyPlugin", function(subpanelName, panel)
    if subpanelName == "inv" then
        -- Inventory subpanel was created
        print("Inventory tab created")
    end
end)
```

### GetPlayerEntityMenu

**Reference**: `gamemode/core/libs/sh_menu.lua:101`

Add options to player interaction menus (CLIENT).

```lua
hook.Add("GetPlayerEntityMenu", "MyPlugin", function(target, options)
    options["Examine"] = function()
        -- Examine player
    end
end)
```

### CanPlayerInteractEntity

**Reference**: `gamemode/core/libs/sh_menu.lua:77`

Control whether player can use entity option (SERVER).

```lua
hook.Add("CanPlayerInteractEntity", "MyPlugin", function(client, entity, option, data)
    if option == "AdminOnly" and not client:IsAdmin() then
        return false  -- Block interaction
    end
end)
```

### PlayerInteractEntity

**Reference**: `gamemode/core/libs/sh_menu.lua:82`

Called when player selects entity option (SERVER).

```lua
hook.Add("PlayerInteractEntity", "MyPlugin", function(client, entity, option, data)
    print(client:Name() .. " selected " .. option .. " on " .. tostring(entity))
end)
```

## Best Practices

### ✅ DO

- Use `CreateMenuButtons` hook to register tabs
- Return `true` from option callbacks to network to server
- Return `false` to handle locally without networking
- Check permissions before showing tabs/options
- Use `bDefault` sparingly (only one tab should be default)
- Validate entity and player in server callbacks
- Use `OnSelect<Option>` methods for clean entity code
- Refresh tab content in `OnSelected` if data may have changed

### ❌ DON'T

- Don't create multiple menus at once (check `ix.menu.IsOpen()`)
- Don't forget to check permissions for admin tabs
- Don't send sensitive data through menu options
- Don't use spaces in option names if using `OnSelect<Option>` method
- Don't create tabs that should be entity menus
- Don't forget to validate callbacks can run after entity removal
- Don't hardcode text (use localization)
- Don't bypass `ix.menu.Open()` with custom context menus

## Common Patterns

### Pattern 1: Conditional Options

```lua
function ENT:ShowPlayerInteraction(client)
    local options = {}

    -- Always show examine
    options["Examine"] = function() return true end

    -- Only show if player has permission
    if client:IsAdmin() then
        options["Edit"] = function() return true end
        options["Remove"] = function() return true end
    end

    -- Only show if not owned
    if not self:GetOwner() or self:GetOwner() == client then
        options["Claim"] = function() return true end
    end

    return options
end
```

### Pattern 2: Data in Menu Choices

```lua
-- CLIENT: Send data with choice
ix.menu.Open({
    ["Buy (10)"] = function()
        return {action = "buy", amount = 10, price = 100}
    end,

    ["Buy (50)"] = function()
        return {action = "buy", amount = 50, price = 450}
    end
}, vendor)

-- SERVER: Receive data
function ENT:OnSelectBuy(client, data)
    if data.action == "buy" and client:GetCharacter():HasMoney(data.price) then
        -- Process purchase
        client:GetCharacter():TakeMoney(data.price)
        -- Give items
    end
end
```

### Pattern 3: Dynamic Tab Content

```lua
hook.Add("CreateMenuButtons", "DynamicTab", function(tabs)
    tabs["dynamic"] = {
        Create = function(info, container)
            container.Refresh = function(self)
                self:Clear()

                -- Rebuild UI based on current data
                local data = LocalPlayer():GetCharacter():GetData("quests", {})

                for id, quest in pairs(data) do
                    local panel = self:Add("DLabel")
                    panel:SetText(quest.name)
                    panel:Dock(TOP)
                end
            end

            container:Refresh()
        end,

        OnSelected = function(info, container)
            container:Refresh()  -- Update when tab opens
        end
    }
end)
```

## Common Issues

### Tab Not Appearing

**Cause**: Hook runs before character loaded or permission check fails
**Fix**: Validate data exists before registering

```lua
hook.Add("CreateMenuButtons", "MyTab", function(tabs)
    local character = LocalPlayer():GetCharacter()

    if not character then
        return  -- Don't add tab without character
    end

    tabs["mytab"] = function(container)
        -- Tab content
    end
end)
```

### Option Not Networking to Server

**Cause**: Callback returns `false` or nothing
**Fix**: Explicitly return `true` or data

```lua
-- WRONG
options["Open"] = function()
    print("Opening")
    -- No return = nil = won't network!
end

-- CORRECT
options["Open"] = function()
    print("Opening")
    return true  -- Network to server
end
```

### Entity Menu Won't Open

**Cause**: Another menu already open or entity too far
**Fix**: Check `ix.menu.IsOpen()` and distance

```lua
-- Check before opening
if ix.menu.IsOpen() then
    print("Menu already open!")
    return
end

if LocalPlayer():GetPos():Distance(entity:GetPos()) > 256 then
    print("Too far!")
    return
end

ix.menu.Open(options, entity)
```

### Server Callback Not Called

**Cause**: Using wrong method name or entity lacks `OnOptionSelected`
**Fix**: Use exact option text (no spaces for method name)

```lua
-- Menu option: "Open Door"
-- Method should be: OnSelectOpenDoor (removes spaces)
function ENT:OnSelectOpenDoor(client, data)
    -- This will be called
end

-- OR use generic handler:
function ENT:OnOptionSelected(client, option, data)
    if option == "Open Door" then
        -- Handle here
    end
end
```

## See Also

- [Derma Overview](derma-overview.md) - UI panel system
- [Subpanel System](../api/subpanel.md) - Subpanel parent/child
- [Configuration System](../systems/configuration.md) - CAMI permissions
- [Entity System](../api/entity.md) - Entity interactions
- Source: `gamemode/core/libs/sh_menu.lua`
- Source: `gamemode/core/derma/cl_menu.lua`
- Source: `gamemode/core/derma/cl_entitymenu.lua`
