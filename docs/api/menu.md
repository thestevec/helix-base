# Menu API (ix.menu)

> **Reference**: `gamemode/core/libs/sh_menu.lua`

The menu API manages the F1 main menu with tabs for character, inventory, business, and custom sections.

## Core Concepts

The F1 menu:
- Opens with F1 key
- Has multiple tabs
- Extensible with plugins
- Handles character selection
- Shows inventory management

## Hooks

### CreateMenuButtons

**Realm**: Client

```lua
hook.Add("CreateMenuButtons", "MyMenu", function(tabs)
    tabs["myTab"] = {
        Create = function(info, container)
            local panel = container:Add("DPanel")
            panel:Dock(FILL)
            return panel
        end,
        
        bDefault = false,  -- Don't show by default
        
        OnSelected = function(info, container)
            -- Called when tab selected
        end
    }
end)
```

## Complete Example

```lua
-- Add custom tab to F1 menu
hook.Add("CreateMenuButtons", "CustomTab", function(tabs)
    tabs["stats"] = function(container)
        local panel = container:Add("DPanel")
        panel:Dock(FILL)

        local char = LocalPlayer():GetCharacter()
        if not char then return panel end

        -- Show character stats
        local stats = panel:Add("DLabel")
        stats:SetText("Character Stats")
        stats:SetFont("DermaLarge")
        stats:Dock(TOP)

        -- Show attributes
        for k, v in pairs(char:GetAttributes()) do
            local attrPanel = panel:Add("DLabel")
            attrPanel:SetText(k .. ": " .. v)
            attrPanel:Dock(TOP)
        end

        return panel
    end
end)
```

## See Also

- [Character API](character.md) - Character menu
- [Inventory API](inventory.md) - Inventory menu
- Source: `gamemode/core/libs/sh_menu.lua`
