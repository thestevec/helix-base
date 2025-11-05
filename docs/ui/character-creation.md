# Character Creation UI

> **Reference**: `gamemode/core/derma/cl_charcreate.lua`, `gamemode/core/derma/cl_charload.lua`, `gamemode/core/derma/cl_character.lua`

The character creation UI provides a multi-step interface for players to create new characters, including faction selection, description/customization, and attribute allocation. It also handles character loading with a carousel display.

## ⚠️ Important: Use Built-in Character System

**Always use Helix's character creation framework** rather than creating custom character menus. The framework provides:
- Multi-step character creation with validation
- Automatic faction filtering and whitelisting
- Model preview with customization
- Attribute point allocation system
- Progress indicator
- Network synchronization
- Character carousel for selection
- Integration with schema customization

## Core Concepts

### What is the Character UI?

The character UI consists of three main components:

1. **ixCharMenu** - Main container managing character selection and creation
2. **ixCharMenuNew** - Character creation subpanel with multi-step wizard
3. **ixCharMenuLoad** - Character selection with carousel display

### Character Creation Steps

**Reference**: `gamemode/core/derma/cl_charcreate.lua:8-150`

1. **Faction Selection** - Choose character faction
2. **Description** - Set name, description, and appearance
3. **Attributes** - Allocate attribute points (if enabled)

### Key Terms

- **Payload** - Data structure holding character creation values
- **Subpanel** - Each step in the creation process
- **Progress** - Visual indicator of creation steps
- **Faction Whitelist** - Restricted factions requiring permission
- **Character Carousel** - Rotating display of existing characters

## Opening Character Menu

### vgui.Create("ixCharMenu")

**Reference**: `gamemode/core/derma/cl_character.lua:8`

```lua
-- Open character selection/creation menu
local charMenu = vgui.Create("ixCharMenu")
-- Automatically calls MakePopup() and displays
```

## Character Creation Customization

### Adding Custom Fields

**Reference**: `gamemode/core/libs/sh_character.lua` (character vars system)

Use character variables to add custom fields:

```lua
-- SHARED: Register custom character variable
ix.char.RegisterVar("age", {
    field = "age",
    fieldType = ix.type.number,
    default = 18,
    isLocal = false,
    bNoDisplay = false,

    -- Validation function
    OnValidate = function(self, value, payload, client)
        if value < 18 or value > 100 then
            return false, "invalidAge"
        end
        return value
    end,

    -- Setup UI panel
    OnPostSetup = function(self, panel, payload)
        panel:SetMin(18)
        panel:SetMax(100)
    end,

    -- Get UI panel for character creation
    OnDisplay = function(self, container, payload)
        local panel = container:Add("ixNumSlider")
        panel:Dock(TOP)
        panel:SetText("Age")
        panel:SetMin(18)
        panel:SetMax(100)
        panel:SetDecimals(0)

        panel.OnValueChanged = function(this, value)
            payload:Set("age", math.floor(value))
        end

        return panel
    end,

    -- Populate from existing character
    OnGet = function(self, character)
        return character:GetAge()
    end,

    -- Save to character
    OnSet = function(self, character, value)
        character:SetAge(value)
    end
})
```

### Custom Description Panels

**Reference**: `gamemode/core/derma/cl_charcreate.lua:69-117`

```lua
-- Add custom panels to description step
hook.Add("OnCharacterCreationSubpanelLoaded", "MyCustomFields", function(panel, subpanelName)
    if subpanelName == "description" then
        -- panel is the description subpanel
        local container = panel.descriptionPanel

        -- Add custom field
        local ageEntry = container:Add("ixNumSlider")
        ageEntry:Dock(TOP)
        ageEntry:DockMargin(0, 0, 0, 8)
        ageEntry:SetText("Age")
        ageEntry:SetMin(18)
        ageEntry:SetMax(100)

        ageEntry.OnValueChanged = function(self, value)
            -- Update payload
            panel.payload:Set("age", math.floor(value))
        end
    end
end)
```

## Faction Customization

### Default Character Names

**Reference**: `gamemode/core/hooks/sh_hooks.lua:414`

Provide default names based on faction:

```lua
-- SHARED: Set default character name
hook.Add("GetDefaultCharacterName", "MySchema", function(client, factionID)
    local faction = ix.faction.indices[factionID]

    if faction.uniqueID == "citizen" then
        -- Return random citizen name
        return "Citizen #" .. math.random(10000, 99999), true  -- true = disable editing
    elseif faction.uniqueID == "combine" then
        return "UNIT-" .. string.upper(string.RandomString(6)), true
    end

    -- Return nil to allow manual name entry
end)
```

### Faction-Specific Models

**Reference**: `gamemode/core/derma/cl_charcreate.lua:45-48`

Factions define available models:

```lua
-- In faction definition
FACTION.models = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/female_01.mdl"
}

-- With gender separation
FACTION.maleModels = {
    "models/player/group01/male_01.mdl"
}

FACTION.femaleModels = {
    "models/player/group01/female_01.mdl"
}
```

## Character Selection Carousel

### Character Display

**Reference**: `gamemode/core/derma/cl_charload.lua`

The carousel displays existing characters with smooth transitions:

```lua
-- Characters are automatically displayed in carousel
-- Players can cycle through with arrow buttons
```

### Custom Character Display

```lua
-- Modify character carousel display
hook.Add("OnCharacterMenuCreated", "CustomCarousel", function(panel)
    local loadPanel = panel.mainPanel

    if IsValid(loadPanel) and loadPanel.carousel then
        -- Customize carousel appearance
        loadPanel.carousel.animationTime = 0.3
    end
end)
```

## Complete Example: Custom Character Creation Field

```lua
-- SHARED: Add custom "backstory" field
ix.char.RegisterVar("backstory", {
    field = "backstory",
    fieldType = ix.type.text,
    default = "",
    isLocal = false,
    bNoDisplay = false,

    OnValidate = function(self, value, payload, client)
        value = string.Trim(value)

        if #value < 50 then
            return false, "Backstory must be at least 50 characters"
        end

        if #value > 500 then
            return false, "Backstory must be less than 500 characters"
        end

        return value
    end,

    OnDisplay = function(self, container, payload)
        local label = container:Add("DLabel")
        label:SetFont("ixMediumFont")
        label:SetText("Character Backstory")
        label:Dock(TOP)
        label:DockMargin(0, 16, 0, 4)

        local entry = container:Add("DTextEntry")
        entry:Dock(TOP)
        entry:SetMultiline(true)
        entry:SetTall(120)
        entry:DockMargin(0, 0, 0, 4)

        entry.OnChange = function(this)
            payload:Set("backstory", this:GetValue())
        end

        -- Character counter
        local counter = container:Add("DLabel")
        counter:SetText("0 / 500")
        counter:Dock(TOP)
        counter:SetFont("ixSmallFont")

        entry.Think = function(this)
            local text = this:GetValue()
            local len = #text
            counter:SetText(len .. " / 500")

            if len < 50 then
                counter:SetTextColor(Color(255, 100, 100))
            elseif len > 500 then
                counter:SetTextColor(Color(255, 0, 0))
            else
                counter:SetTextColor(Color(100, 255, 100))
            end
        end

        return entry
    end,

    OnGet = function(self, character)
        return character:GetData("backstory", "")
    end,

    OnSet = function(self, character, value)
        character:SetData("backstory", value)
    end
})
```

## Complete Example: Faction-Specific Creation Steps

```lua
-- CLIENT: Add faction-specific step
hook.Add("OnCharacterCreationSetup", "MilitaryRank", function(panel)
    local payload = panel.payload

    -- Only for military faction
    if payload:Get("faction") != FACTION_MILITARY then
        return
    end

    -- Add rank selection step between description and attributes
    local rankPanel = panel:AddSubpanel("rank")
    rankPanel:SetTitle("Choose Your Rank")

    local rankList = rankPanel:Add("DScrollPanel")
    rankList:Dock(FILL)

    local ranks = {
        "Private",
        "Corporal",
        "Sergeant",
        "Lieutenant",
        "Captain"
    }

    for _, rank in ipairs(ranks) do
        local button = rankList:Add("DButton")
        button:SetText(rank)
        button:Dock(TOP)
        button:DockMargin(4, 4, 4, 4)
        button:SetTall(40)

        button.DoClick = function()
            payload:Set("rank", rank)
            panel:SetActiveSubpanel("attributes")
        end
    end
end)
```

## Complete Example: Character Appearance Randomizer

```lua
-- CLIENT: Add randomize button
hook.Add("OnCharacterCreationSubpanelLoaded", "RandomizeAppearance", function(panel, subpanelName)
    if subpanelName != "description" then
        return
    end

    local container = panel.descriptionPanel

    local randomize = container:Add("ixMenuButton")
    randomize:SetText("Randomize Appearance")
    randomize:Dock(BOTTOM)
    randomize:DockMargin(0, 8, 0, 0)
    randomize:SizeToContents()

    randomize.DoClick = function()
        local payload = panel.payload
        local factionID = payload:Get("faction")
        local faction = ix.faction.indices[factionID]

        if not faction then
            return
        end

        -- Random model
        local models = faction.models or {}
        if #models > 0 then
            local model = models[math.random(#models)]
            payload:Set("model", model)
            panel.descriptionModel:SetModel(model)
        end

        -- Random skin
        local maxSkin = panel.descriptionModel:SkinCount() - 1
        if maxSkin > 0 then
            local skin = math.random(0, maxSkin)
            payload:Set("skin", skin)
            panel.descriptionModel:SetSkin(skin)
        end

        -- Random bodygroups
        local bodygroups = {}
        for i = 0, panel.descriptionModel:GetNumBodyGroups() - 1 do
            local max = panel.descriptionModel:GetBodygroupCount(i) - 1
            bodygroups[i] = math.random(0, max)
            panel.descriptionModel:SetBodygroup(i, bodygroups[i])
        end
        payload:Set("groups", bodygroups)

        LocalPlayer():Notify("Appearance randomized!")
    end
end)
```

## Hooks

### OnCharacterMenuCreated

**Reference**: `gamemode/core/derma/cl_character.lua:382`

Called when character menu is created.

```lua
hook.Add("OnCharacterMenuCreated", "MyPlugin", function(panel)
    -- panel is the ixCharMenu
    -- Customize menu appearance or behavior
    print("Character menu opened")
end)
```

### GetDefaultCharacterName

**Reference**: `gamemode/core/hooks/sh_hooks.lua:414`

Provide default character names (SHARED).

```lua
hook.Add("GetDefaultCharacterName", "MySchema", function(client, factionID)
    -- Return name and whether it's editable
    -- return "Name", true  -- true = player can't change it
    -- return "Name", false -- false = player can edit it
    -- return nil           -- Allow manual entry
end)
```

### OnCharacterCreationSubpanelLoaded

Custom hook for modifying creation steps.

```lua
hook.Add("OnCharacterCreationSubpanelLoaded", "MyPlugin", function(panel, subpanelName)
    -- subpanelName: "faction", "description", or "attributes"
    -- Add custom UI elements to the subpanel
end)
```

## Best Practices

### ✅ DO

- Use `ix.char.RegisterVar()` for custom character fields
- Validate all input in `OnValidate` callbacks
- Provide clear error messages for validation failures
- Use faction-specific models defined in faction tables
- Check faction whitelist before showing options
- Use payload:Set() and payload:Get() for temporary data
- Return proper values from GetDefaultCharacterName
- Test character creation with multiple factions

### ❌ DON'T

- Don't bypass validation checks
- Don't forget to handle faction whitelisting
- Don't hardcode faction IDs (use faction.uniqueID)
- Don't modify character data before validation passes
- Don't forget to network custom fields properly
- Don't create characters without required fields
- Don't allow invalid model paths
- Don't forget to sync character data to clients

## Common Patterns

### Pattern 1: Conditional Field Display

```lua
-- Show field only for specific factions
ix.char.RegisterVar("badge", {
    field = "badge",
    default = "",

    OnDisplay = function(self, container, payload)
        local factionID = payload:Get("faction")
        local faction = ix.faction.indices[factionID]

        -- Only show for police faction
        if faction.uniqueID != "police" then
            return
        end

        local entry = container:Add("ixTextEntry")
        entry:Dock(TOP)
        entry:SetPlaceholderText("Badge Number")

        entry.OnChange = function(this)
            payload:Set("badge", this:GetValue())
        end

        return entry
    end
})
```

### Pattern 2: Linked Fields

```lua
-- Update one field based on another
hook.Add("OnCharacterCreationSubpanelLoaded", "LinkedFields", function(panel, subpanelName)
    if subpanelName != "description" then
        return
    end

    local payload = panel.payload

    -- When faction changes, update available models
    local oldSet = payload.Set
    payload.Set = function(self, key, value)
        oldSet(self, key, value)

        if key == "faction" then
            -- Reset model to first available for new faction
            local faction = ix.faction.indices[value]
            if faction and faction.models and #faction.models > 0 then
                self:Set("model", faction.models[1])
            end
        end
    end
end)
```

### Pattern 3: Multi-Step Validation

```lua
-- Validate across multiple fields
ix.char.RegisterVar("email", {
    field = "email",
    default = "",

    OnValidate = function(self, value, payload, client)
        local name = payload:Get("name")

        -- Ensure email matches character name format
        if not value:match(name:lower():gsub("%s", "%.")) then
            return false, "Email must match character name format"
        end

        return value
    end
})
```

## Common Issues

### Character Not Saving

**Cause**: Validation failing or network error
**Fix**: Check validation callbacks and server console

```lua
-- SHARED: Debug validation
ix.char.RegisterVar("myfield", {
    OnValidate = function(self, value, payload, client)
        print("Validating myfield:", value)

        if not value or value == "" then
            print("Validation failed: empty value")
            return false, "Field cannot be empty"
        end

        print("Validation passed")
        return value
    end
})
```

### Custom Fields Not Appearing

**Cause**: Variable not registered or display callback not set
**Fix**: Ensure OnDisplay is implemented

```lua
-- WRONG
ix.char.RegisterVar("myfield", {
    field = "myfield"
    -- No OnDisplay!
})

-- CORRECT
ix.char.RegisterVar("myfield", {
    field = "myfield",
    OnDisplay = function(self, container, payload)
        local panel = container:Add("ixTextEntry")
        panel:Dock(TOP)
        return panel
    end
})
```

### Faction Models Not Loading

**Cause**: Invalid model paths or faction not properly defined
**Fix**: Verify model paths and faction configuration

```lua
-- Verify faction has models
local faction = ix.faction.indices[factionID]

if not faction then
    print("Faction doesn't exist!")
    return
end

if not faction.models or #faction.models == 0 then
    print("Faction has no models!")
    return
end

-- Check model validity
for _, model in ipairs(faction.models) do
    if not file.Exists(model, "GAME") then
        print("Invalid model:", model)
    end
end
```

## See Also

- [Character System](../systems/character.md) - Backend character management
- [Factions System](../systems/factions.md) - Faction definitions
- [Attributes System](../systems/attributes.md) - Character attributes
- [Derma Overview](derma-overview.md) - UI panel basics
- [Menus](menus.md) - Menu integration
- Source: `gamemode/core/derma/cl_charcreate.lua`
- Source: `gamemode/core/derma/cl_charload.lua`
- Source: `gamemode/core/derma/cl_character.lua`
- Source: `gamemode/core/libs/sh_character.lua` (character vars)
