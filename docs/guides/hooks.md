# Understanding and Using Hooks

> **Reference**: `docs/hooks/`, `gamemode/core/libs/sh_plugin.lua`

Complete guide to using Garry's Mod and Helix hooks for event-driven programming.

## What Are Hooks?

Hooks are callback functions that run when specific events occur. Helix uses hooks extensively for plugins and schema code.

## Hook Basics

### Adding Hooks

```lua
-- In plugin or schema
function PLUGIN:PlayerSpawn(client)
    client:SetHealth(100)
    client:ChatPrint("You spawned!")
end

-- Or using hook.Add
hook.Add("PlayerSpawn", "MyUniqueID", function(client)
    client:SetHealth(100)
end)
```

### Removing Hooks

```lua
-- Remove specific hook
hook.Remove("PlayerSpawn", "MyUniqueID")

-- Plugin hooks are auto-removed when plugin unloads
```

## Common Helix Hooks

### Character Hooks

```lua
-- When character is created
function PLUGIN:OnCharacterCreated(client, character)
    character:GiveMoney(100)  -- Starting money
end

-- When character loads
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    client:ChatPrint("Welcome, " .. character:GetName())
end

-- Before character saves
function PLUGIN:CharacterPreSave(character)
    -- Save custom data
    local stats = character:GetData("stats", {})
    -- Modify stats before save
end
```

### Player Hooks

```lua
-- Player connects
function PLUGIN:PlayerInitialSpawn(client)
    print(client:Name() .. " connected")
end

-- Player fully loaded
function PLUGIN:PlayerLoadout(client)
    -- Give starting weapons
    client:Give("weapon_pistol")
end

-- Player dies
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    if attacker:IsPlayer() and attacker != victim then
        local char = attacker:GetCharacter()
        if char then
            char:SetData("kills", char:GetData("kills", 0) + 1)
        end
    end
end

-- Player says something
function PLUGIN:PlayerSay(client, text)
    if text == "!help" then
        client:ChatPrint("Type /help for commands")
        return ""  -- Block original message
    end
end
```

### Item Hooks

```lua
-- Item used
function PLUGIN:OnItemTransferred(item, oldInv, newInv)
    print(item:GetName() .. " moved inventories")
end

-- Before item transfer
function PLUGIN:CanItemTransfer(item, oldInv, newInv)
    if item.flag and not item.player:GetCharacter():HasFlags(item.flag) then
        return false  -- Block transfer
    end
end
```

### Initialization Hooks

```lua
-- All plugins loaded
function PLUGIN:InitializedPlugins()
    -- Register commands, config, etc.
    ix.command.Add("MyCommand", {...})
end

-- Schema loaded
function PLUGIN:InitializedSchema()
    -- Schema-specific setup
end
```

## Return Values

Some hooks use return values to modify behavior:

```lua
-- Block player spawn
function PLUGIN:PlayerSpawn(client)
    if client:GetCharacter():GetData("jailed") then
        return false  -- Prevent spawn
    end
end

-- Modify damage
function PLUGIN:EntityTakeDamage(entity, dmgInfo)
    if entity:IsPlayer() then
        local damage = dmgInfo:GetDamage()
        dmgInfo:SetDamage(damage * 0.5)  -- Half damage
    end
end

-- Modify chat text
function PLUGIN:OnChatReceived(client, chatType, text, anonymous, receivers, rawText)
    return string.upper(text)  -- ALL CAPS
end
```

## Practical Examples

### Example 1: AFK System

```lua
PLUGIN.afkPlayers = PLUGIN.afkPlayers or {}

function PLUGIN:StartCommand(client, cmd)
    if cmd:GetButtons() != 0 then
        self.afkPlayers[client] = nil
    else
        self.afkPlayers[client] = (self.afkPlayers[client] or 0) + 1

        if self.afkPlayers[client] > 600 then  -- 10 minutes
            client:Kick("AFK too long")
        end
    end
end
```

### Example 2: Custom Death Messages

```lua
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    local victimName = victim:GetCharacter():GetName()

    if attacker:IsPlayer() and attacker != victim then
        local attackerName = attacker:GetCharacter():GetName()
        local weapon = attacker:GetActiveWeapon():GetClass()

        ix.chat.Send(nil, "event", attackerName .. " killed " .. victimName .. " with " .. weapon)
    else
        ix.chat.Send(nil, "event", victimName .. " died")
    end
end
```

### Example 3: Playtime Tracking

```lua
PLUGIN.joinTimes = PLUGIN.joinTimes or {}

function PLUGIN:PlayerLoadedCharacter(client, character)
    self.joinTimes[character:GetID()] = os.time()
end

function PLUGIN:CharacterPreSave(character)
    local charID = character:GetID()
    local joinTime = self.joinTimes[charID]

    if joinTime then
        local sessionTime = os.time() - joinTime
        local totalTime = character:GetData("playtime", 0)
        character:SetData("playtime", totalTime + sessionTime)
        self.joinTimes[charID] = os.time()
    end
end
```

## Best Practices

### ✅ DO
- Use PLUGIN: syntax in plugins
- Return false to block events
- Check entity validity
- Clean up in PlayerDisconnect
- Use specific hooks (not Think)

### ❌ DON'T
- Don't use generic hook names
- Don't forget return values
- Don't run heavy code in Think
- Don't forget realm checks
- Don't access removed entities

## Common Hooks Reference

### Player
- `PlayerInitialSpawn` - Connects
- `PlayerSpawn` - Spawns
- `PlayerLoadout` - Gets weapons
- `PlayerDeath` - Dies
- `PlayerDisconnected` - Leaves

### Character
- `OnCharacterCreated` - New character
- `PlayerLoadedCharacter` - Loads character
- `CharacterPreSave` - Before save

### Items/Inventory
- `OnItemTransferred` - Item moves
- `CanItemTransfer` - Check transfer
- `OnItemSpawned` - Item drops

### Chat
- `PlayerSay` - Player sends chat
- `OnChatReceived` - Client receives

## See Also

- [First Plugin Tutorial](first-plugin.md)
- [Commands Guide](commands.md)
- Hook Reference: `docs/hooks/`
