# Complete Hook Examples

> **Reference**: `docs/hooks/plugin.lua`, `gamemode/core/libs/sh_plugin.lua`

This document provides complete, working examples of commonly used Helix hooks.

## ⚠️ Important: Use Plugin Hooks Correctly

**Always use plugin hooks with `PLUGIN:HookName()`** rather than using `hook.Add()` directly. The framework provides:
- Automatic hook registration
- Hook caching for performance
- Integration with plugin system
- Proper hook priority
- Clean hook removal when plugin unloads

## Hook Structure

### Basic Hook Syntax

```lua
-- In plugin file (sh_plugin.lua, sv_hooks.lua, cl_hooks.lua)
function PLUGIN:HookName(arg1, arg2)
    -- Your code here

    -- Return value (if hook expects one)
    return result
end
```

### Return Values

- **Return `nil`**: Allow other hooks to run, use default behavior
- **Return value**: Override default behavior (varies by hook)
- **Return `false`**: Usually prevents action (check hook docs)
- **Return `true`**: Usually allows action (check hook docs)

## Character Hooks

### PlayerLoadedCharacter

Called when a player selects/loads a character.

```lua
-- When a player loads a character
function PLUGIN:PlayerLoadedCharacter(client, character, lastChar)
    -- Give starting items for new characters
    if !character:GetData("hasStarterKit") then
        local inventory = character:GetInventory()

        inventory:Add("item_medkit", 1)
        inventory:Add("item_flashlight", 1)

        character:SetData("hasStarterKit", true)
        client:Notify("You received a starter kit!")
    end

    -- Track playtime
    local steamID = client:SteamID()
    local uniqueID = "ixPlaytime_" .. steamID

    timer.Create(uniqueID, 60, 0, function()
        if IsValid(client) and client:GetCharacter() == character then
            local playtime = character:GetData("playtime", 0)
            character:SetData("playtime", playtime + 1)
        end
    end)

    -- Welcome message
    timer.Simple(2, function()
        if IsValid(client) then
            client:ChatPrint("Welcome back, " .. character:GetName() .. "!")
        end
    end)
end
```

### OnCharacterCreated

Called when a new character is created.

```lua
-- When a new character is first created
function PLUGIN:OnCharacterCreated(client, character)
    -- Set default data
    character:SetData("creationDate", os.time())
    character:SetData("level", 1)
    character:SetData("experience", 0)

    -- Give faction-specific items
    local faction = ix.faction.indices[character:GetFaction()]

    if faction.uniqueID == "citizen" then
        character:GiveMoney(100)
    elseif faction.uniqueID == "police" then
        character:GiveMoney(200)
        character:SetData("rank", 1)
    end

    -- Log creation
    ix.log.Add(client, "characterCreate", character:GetName())

    -- Notify admins
    for _, admin in ipairs(player.GetAll()) do
        if admin:IsAdmin() then
            admin:ChatPrint(client:Name() .. " created character: " .. character:GetName())
        end
    end
end
```

### CharacterPreSave

Called before character data is saved to database.

```lua
-- Before character is saved to database
function PLUGIN:CharacterPreSave(character)
    local client = character:GetPlayer()

    if !IsValid(client) then return end

    -- Save current position
    character:SetData("lastPosition", client:GetPos())
    character:SetData("lastAngles", client:GetAngles())

    -- Save current health/armor
    character:SetData("health", client:Health())
    character:SetData("armor", client:Armor())

    -- Save current stamina (if using stamina plugin)
    character:SetData("stamina", client:GetLocalVar("stm", 0))
end
```

### CharacterDeleted

Called when a character is deleted.

```lua
-- When character is deleted
function PLUGIN:CharacterDeleted(client, character)
    local steamID = client:SteamID()
    local charID = character:GetID()

    -- Clean up character-specific data
    local query = mysql:Delete("ix_plugin_data")
    query:Where("character_id", charID)
    query:Execute()

    -- Remove from leaderboards
    if self.leaderboard[charID] then
        self.leaderboard[charID] = nil
    end

    -- Log deletion
    ix.log.Add(client, "characterDelete", character:GetName())

    -- Notify player
    client:Notify("Character " .. character:GetName() .. " has been deleted")
end
```

## Combat Hooks

### PlayerDeath

Called when a player dies.

```lua
-- When a player dies
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    local character = victim:GetCharacter()

    if !character then return end

    -- Track deaths
    local deaths = character:GetData("deaths", 0)
    character:SetData("deaths", deaths + 1)

    -- Penalty for death
    local money = character:GetMoney()
    local penalty = math.floor(money * 0.1)  -- Lose 10% of money

    if penalty > 0 then
        character:TakeMoney(penalty)
        victim:Notify("You lost $" .. penalty .. " on death")
    end

    -- PvP tracking
    if IsValid(attacker) and attacker:IsPlayer() and attacker != victim then
        local attackerChar = attacker:GetCharacter()

        if attackerChar then
            local kills = attackerChar:GetData("kills", 0)
            attackerChar:SetData("kills", kills + 1)

            -- Reward killer
            attackerChar:GiveMoney(50)
            attacker:Notify("You received $50 for the kill")
        end

        -- Log PvP death
        ix.log.Add(victim, "pvpDeath", attacker:Name(), character:GetName())
    end

    -- Drop inventory on death (optional)
    if ix.config.Get("dropInventoryOnDeath", false) then
        character:GetInventory():DropAll(victim:GetPos())
    end
end
```

### PlayerHurt

Called when a player takes damage.

```lua
-- When player takes damage
function PLUGIN:PlayerHurt(client, attacker, health, damage)
    local character = client:GetCharacter()

    if !character then return end

    -- Track total damage taken
    local damageTaken = character:GetData("damageTaken", 0)
    character:SetData("damageTaken", damageTaken + damage)

    -- Low health warning
    if health < 30 and health > 0 then
        client:EmitSound("player/heartbeat1.wav")
    end

    -- Combat log
    if IsValid(attacker) and attacker:IsPlayer() and attacker != client then
        local attackerName = attacker:Name()
        local victimName = client:Name()

        ix.log.Add(client, "playerHurt", attackerName, victimName, math.Round(damage))
    end
end
```

### CanPlayerTakeDamage

Called to check if player can take damage.

```lua
-- Check if player can take damage
function PLUGIN:CanPlayerTakeDamage(client, attacker)
    local character = client:GetCharacter()

    if !character then
        return false  -- No character = invincible
    end

    -- Safe zones
    if client:GetPos():WithinAABox(Vector(-500, -500, 0), Vector(500, 500, 200)) then
        if IsValid(attacker) and attacker:IsPlayer() then
            attacker:Notify("You cannot attack players in the safe zone!")
        end

        return false
    end

    -- God mode flag
    if character:HasFlags("g") then
        return false
    end

    -- PvP toggle
    if IsValid(attacker) and attacker:IsPlayer() then
        local victimPvP = character:GetData("pvpEnabled", true)
        local attackerChar = attacker:GetCharacter()
        local attackerPvP = attackerChar and attackerChar:GetData("pvpEnabled", true)

        if !victimPvP or !attackerPvP then
            attacker:Notify("PvP is disabled for this player")
            return false
        end
    end

    -- Allow damage
    return true
end
```

## Item Hooks

### CanPlayerInteractItem

Called when player tries to use/drop/interact with item.

```lua
-- Check if player can interact with item
function PLUGIN:CanPlayerInteractItem(client, action, item, data)
    local character = client:GetCharacter()

    if !character then return false end

    -- Prevent dropping quest items
    if action == "drop" and item:GetData("isQuestItem") then
        client:Notify("You cannot drop quest items!")
        return false
    end

    -- Cooldown on using items
    if action == "use" then
        local lastUse = item:GetData("lastUse", 0)

        if CurTime() - lastUse < 5 then
            client:Notify("You must wait before using this item again")
            return false
        end

        item:SetData("lastUse", CurTime())
    end

    -- Class restrictions
    if item.requiredClass and character:GetClass() != item.requiredClass then
        client:Notify("Your class cannot use this item")
        return false
    end

    -- Level requirements
    if item.requiredLevel then
        local level = character:GetData("level", 1)

        if level < item.requiredLevel then
            client:Notify("You need level " .. item.requiredLevel .. " to use this")
            return false
        end
    end
end
```

### OnItemTransferred

Called when item moves between inventories.

```lua
-- When item is moved between inventories
function PLUGIN:OnItemTransferred(item, oldInventory, newInventory)
    if !oldInventory or !newInventory then return end

    -- Track item transfers
    local fromChar = oldInventory.owner
    local toChar = newInventory.owner

    if fromChar and toChar and fromChar != toChar then
        -- Log trade
        ix.log.Add(nil, "itemTransfer", item.name, fromChar, toChar)

        -- Achievement for first trade
        local char = ix.char.loaded[toChar]

        if char then
            local trades = char:GetData("tradesReceived", 0)

            if trades == 0 then
                char:SetData("unlockedTrading", true)
                local client = char:GetPlayer()

                if IsValid(client) then
                    client:Notify("Achievement: First Trade!")
                end
            end

            char:SetData("tradesReceived", trades + 1)
        end
    end
end
```

## Chat Hooks

### PlayerSay

Called when player sends a chat message.

```lua
-- When player sends chat message
function PLUGIN:PlayerSay(client, text, bTeamChat)
    local character = client:GetCharacter()

    if !character then return "" end  -- Block message if no character

    -- Command prefix filtering (let command system handle it)
    if string.sub(text, 1, 1) == "/" or string.sub(text, 1, 1) == "!" then
        return  -- Let command system handle
    end

    -- Mute system
    if character:GetData("isMuted") then
        local muteEnd = character:GetData("muteEnd", 0)

        if muteEnd > os.time() then
            local remaining = muteEnd - os.time()
            client:Notify("You are muted for " .. remaining .. " more seconds")
            return ""  -- Block message
        else
            -- Unmute
            character:SetData("isMuted", nil)
            character:SetData("muteEnd", nil)
        end
    end

    -- Anti-spam
    local lastMessage = client.ixLastMessage or 0

    if CurTime() - lastMessage < 1 then
        client:Notify("Please wait before sending another message")
        return ""
    end

    client.ixLastMessage = CurTime()

    -- Profanity filter (basic example)
    local lower = string.lower(text)
    local badWords = {"badword1", "badword2", "badword3"}

    for _, word in ipairs(badWords) do
        if string.find(lower, word, 1, true) then
            client:Notify("Your message contains inappropriate language")
            ix.log.Add(client, "profanity", text)
            return ""
        end
    end

    -- Allow message
    return
end
```

### ChatboxCreated

Called when chatbox is created (CLIENT).

```lua
-- CLIENT: When chatbox is created
if CLIENT then
    function PLUGIN:ChatboxCreated()
        -- Add custom tab
        ix.chat.Register("admin", {
            format = "[ADMIN] %s: %s",
            GetColor = function(self, speaker, text)
                return Color(255, 100, 100)
            end,
            CanSay = function(self, speaker, text)
                return speaker:IsAdmin()
            end,
            OnChatAdd = function(self, speaker, text)
                chat.AddText(Color(255, 100, 100), "[ADMIN] ", speaker, ": ", text)
            end,
            prefix = {"@"}
        })
    end
end
```

## UI Hooks (Client)

### HUDPaint

Called every frame to draw HUD (CLIENT).

```lua
if CLIENT then
    -- Draw custom HUD elements
    function PLUGIN:HUDPaint()
        local client = LocalPlayer()
        local character = client:GetCharacter()

        if !character then return end

        local scrW, scrH = ScrW(), ScrH()

        -- Draw level
        local level = character:GetData("level", 1)
        local exp = character:GetData("experience", 0)
        local expNeeded = level * 100

        draw.SimpleText("Level: " .. level, "ixMediumFont", scrW - 150, scrH - 80, color_white, TEXT_ALIGN_LEFT)
        draw.SimpleText("EXP: " .. exp .. "/" .. expNeeded, "ixSmallFont", scrW - 150, scrH - 60, Color(200, 200, 200), TEXT_ALIGN_LEFT)

        -- Draw EXP bar
        local barWidth = 200
        local barHeight = 10
        local barX = scrW - 160
        local barY = scrH - 45

        surface.SetDrawColor(0, 0, 0, 200)
        surface.DrawRect(barX, barY, barWidth, barHeight)

        local fillWidth = math.Clamp((exp / expNeeded) * barWidth, 0, barWidth)
        surface.SetDrawColor(100, 200, 255, 255)
        surface.DrawRect(barX, barY, fillWidth, barHeight)
    end
end
```

### CreateMenuButtons

Add buttons to F1 menu (CLIENT).

```lua
if CLIENT then
    -- Add custom menu buttons
    function PLUGIN:CreateMenuButtons(tabs)
        -- Add achievements tab
        tabs["achievements"] = function(container)
            local panel = container:Add("DPanel")
            panel:Dock(FILL)

            local label = panel:Add("DLabel")
            label:Dock(TOP)
            label:SetText("Achievements")
            label:SetFont("ixBigFont")
            label:SetContentAlignment(5)
            label:SizeToContents()

            -- Add achievement list
            local list = panel:Add("DScrollPanel")
            list:Dock(FILL)

            -- Request achievements from server
            net.Start("ixRequestAchievements")
            net.SendToServer()
        end

        -- Add stats tab
        tabs["statistics"] = function(container)
            local character = LocalPlayer():GetCharacter()

            if !character then return end

            local panel = container:Add("DPanel")
            panel:Dock(FILL)

            -- Display stats
            local stats = {
                {"Kills", character:GetData("kills", 0)},
                {"Deaths", character:GetData("deaths", 0)},
                {"Playtime", math.Round(character:GetData("playtime", 0) / 60) .. " hours"},
                {"Money Earned", "$" .. character:GetData("totalEarned", 0)}
            }

            for _, stat in ipairs(stats) do
                local row = panel:Add("DPanel")
                row:Dock(TOP)
                row:SetTall(30)

                local nameLabel = row:Add("DLabel")
                nameLabel:SetPos(10, 5)
                nameLabel:SetText(stat[1] .. ":")
                nameLabel:SetFont("ixMediumFont")
                nameLabel:SizeToContents()

                local valueLabel = row:Add("DLabel")
                valueLabel:SetPos(200, 5)
                valueLabel:SetText(tostring(stat[2]))
                valueLabel:SetFont("ixMediumFont")
                valueLabel:SizeToContents()
            end
        end
    end
end
```

## Utility Hooks

### InitializedPlugins

Called when all plugins are loaded.

```lua
-- When all plugins finish loading
function PLUGIN:InitializedPlugins()
    -- Load saved data
    local data = self:GetData()
    self.playerData = data.playerData or {}

    -- Start periodic tasks
    timer.Create("ixAutoSave", 300, 0, function()
        self:SetData({
            playerData = self.playerData,
            lastSave = os.time()
        })
    end)

    -- Register chat command
    ix.chat.Register("faction", {
        format = "[%s] %s: %s",
        CanSay = function(self, speaker, text)
            return speaker:GetCharacter() != nil
        end,
        OnChatAdd = function(self, speaker, text)
            local char = speaker:GetCharacter()
            local faction = ix.faction.indices[char:GetFaction()]

            chat.AddText(faction.color, "[" .. faction.name .. "] ", speaker, ": ", text)
        end,
        prefix = {"/f"}
    })

    print("[" .. self.name .. "] Plugin initialized successfully")
end
```

### OnPluginUnloaded

Called when plugin is being unloaded.

```lua
-- When plugin is unloaded
function PLUGIN:OnPluginUnloaded()
    -- Save data
    self:SetData({
        playerData = self.playerData,
        lastSave = os.time()
    })

    -- Remove timers
    timer.Remove("ixAutoSave")
    timer.Remove("ixDataSync")

    -- Close UI
    if CLIENT then
        if IsValid(ix.gui.myPlugin) then
            ix.gui.myPlugin:Remove()
        end
    end

    -- Clean up
    self.playerData = nil

    print("[" .. self.name .. "] Plugin unloaded")
end
```

## ⚠️ Do NOT

```lua
-- WRONG: Don't use hook.Add in plugins
hook.Add("PlayerSpawn", "MyHook", function(client)
    -- Use PLUGIN:PlayerSpawn instead!
end)

-- WRONG: Don't forget return values
function PLUGIN:CanPlayerTakeDamage(client, attacker)
    if client:HasGodMode() then
        -- Missing return false!
    end
end

-- WRONG: Don't modify without validation
function PLUGIN:PlayerSay(client, text)
    -- No validation, could break things
    return string.upper(text)
end

-- WRONG: Don't forget realm checks
function PLUGIN:HUDPaint()
    -- This is CLIENT-only hook, but no check!
    local client = LocalPlayer()  -- Will error on server
end
```

## Best Practices

### ✅ DO

- Use `PLUGIN:HookName()` for all hooks
- Return appropriate values based on hook
- Check `IsValid()` before using entities
- Validate all input and parameters
- Use realm checks (`if CLIENT`/`if SERVER`)
- Clean up timers in `OnPluginUnloaded()`
- Log important events with `ix.log.Add()`
- Check for nil characters/entities
- Use `timer.Simple()` for delayed actions
- Store plugin data with `self:SetData()`

### ❌ DON'T

- Don't use `hook.Add()` in plugins
- Don't forget to return values when needed
- Don't trust client data on server
- Don't modify global variables
- Don't create infinite loops
- Don't forget cleanup in `OnPluginUnloaded()`
- Don't use expensive operations in `Think()` or `HUDPaint()`
- Don't forget `IsValid()` checks in callbacks
- Don't bypass framework systems

## See Also

- [Plugin System](../plugins/plugin-system.md) - Complete plugin documentation
- [Hook Reference](../plugins/hooks.md) - All available hooks
- Source: `docs/hooks/plugin.lua` - Complete hook documentation
- Source: `gamemode/core/libs/sh_plugin.lua` - Plugin system implementation
