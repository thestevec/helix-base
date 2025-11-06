# Plugin Hooks Reference

> **Reference**: `docs/hooks/plugin.lua`, `gamemode/core/libs/sh_plugin.lua:78-83`

This document provides a complete reference of all hooks available in the Helix plugin system. Hooks allow your plugin to respond to events and modify framework behavior.

## ⚠️ Important: Use Plugin Hook Methods

**Always define hooks as methods on PLUGIN** rather than using `hook.Add()` manually. The framework provides:
- Automatic hook registration when you define `function PLUGIN:HookName()`
- Hook caching for performance via `HOOKS_CACHE` table
- Proper execution order (plugins → schema → gamemode)
- Easy hook management and cleanup
- Return value handling

## How Hooks Work

**Reference**: `gamemode/core/libs/sh_plugin.lua:342-364`

When you create a function on `PLUGIN`, Helix automatically caches it:

```lua
-- You define:
function PLUGIN:PlayerSpawn(client)
    client:SetHealth(100)
end

-- Helix automatically does:
HOOKS_CACHE["PlayerSpawn"] = HOOKS_CACHE["PlayerSpawn"] or {}
HOOKS_CACHE["PlayerSpawn"][PLUGIN] = PLUGIN.PlayerSpawn

-- When event fires:
hook.Call("PlayerSpawn", nil, client)
  → Calls all plugin hooks first
  → Then Schema hook if exists
  → Finally gamemode hooks
```

### Hook Execution Order

1. **Plugin hooks** - All plugins with this hook (order not guaranteed)
2. **Schema hook** - Schema's implementation (if exists)
3. **Gamemode hooks** - Garry's Mod gamemode hooks

**First non-nil return value stops the chain** and is returned to the caller.

## Hook Categories

- [Plugin Lifecycle](#plugin-lifecycle)
- [Character Hooks](#character-hooks)
- [Player Hooks](#player-hooks)
- [Item & Inventory Hooks](#item--inventory-hooks)
- [Chat & Communication](#chat--communication)
- [Door & Entity Interaction](#door--entity-interaction)
- [Combat & Damage](#combat--damage)
- [UI & Client Hooks](#ui--client-hooks)
- [Data & Persistence](#data--persistence)
- [Configuration & Setup](#configuration--setup)

---

## Plugin Lifecycle

### InitializedPlugins

Called after ALL plugins have been loaded.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:610`

```lua
function PLUGIN:InitializedPlugins()
    -- Good for setup that depends on other plugins
    print("All plugins loaded!")

    if SERVER then
        -- Initialize plugin data
        local data = self:GetData()
        if not data.initialized then
            data.initialized = true
            self:SetData(data)
        end
    end
end
```

### InitializedSchema

Called after the schema has been loaded.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:614`

```lua
function PLUGIN:InitializedSchema()
    -- Schema-specific initialization
    if Schema.uniqueID == "hl2rp" then
        print("HL2RP schema detected!")
    end
end
```

### OnPluginLoaded

Called when ANY plugin loads.

**Realm**: Shared

```lua
function PLUGIN:OnPluginLoaded(plugin)
    print("Plugin loaded:", plugin.name)

    -- Check for dependencies
    if plugin.uniqueID == "vendor" then
        self.hasVendor = true
    end
end
```

### OnPluginUnloaded

Called when this plugin is unloaded.

**Realm**: Shared

```lua
function PLUGIN:OnPluginUnloaded()
    -- Clean up timers
    timer.Remove("MyPluginTimer")

    -- Clean up UI
    if CLIENT and IsValid(self.menuPanel) then
        self.menuPanel:Remove()
    end

    -- Save final data
    if SERVER then
        self:SetData(self:GetData())
    end
end
```

### PluginShouldLoad

Determines if a plugin should load.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:917`

```lua
function PLUGIN:PluginShouldLoad(uniqueID)
    -- Prevent loading incompatible plugin
    if uniqueID == "oldplugin" then
        return false
    end
end
```

---

## Character Hooks

### CharacterLoaded

Called when character data is loaded from database.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:401`

```lua
function PLUGIN:CharacterLoaded(character)
    -- Character data is now available
    print("Loaded character:", character:GetName())
end
```

### PlayerLoadedCharacter

Called when a player selects and loads a character.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:818`

```lua
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    client:ChatPrint("Welcome, " .. character:GetName() .. "!")

    -- Restore character-specific data
    local health = character:GetData("savedHealth", 100)
    client:SetHealth(health)
end
```

### PrePlayerLoadedCharacter

Called before a character is loaded (can prevent loading).

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:996`

```lua
function PLUGIN:PrePlayerLoadedCharacter(client, character, currentChar)
    -- Validate character before loading
    if character:GetData("banned") then
        return false -- Prevent loading
    end
end
```

### OnCharacterCreated

Called after a character is created.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:671`

```lua
function PLUGIN:OnCharacterCreated(client, character)
    if SERVER then
        -- Give starter items
        local inventory = character:GetInventory()
        inventory:Add("item_medkit")
    end
end
```

### CanPlayerCreateCharacter

Whether a player can create a new character.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:119`

**Returns**: `false` to prevent, `string` for error message, `...` for message args

```lua
function PLUGIN:CanPlayerCreateCharacter(client, payload)
    if not client:IsAdmin() and #client:GetCharacter() >= 3 then
        return false, "tooManyCharacters"
    end
end
```

### AdjustCreationPayload

Modifies data used for character creation.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:12`

```lua
function PLUGIN:AdjustCreationPayload(client, payload, newPayload)
    -- Set starting money based on faction
    local faction = ix.faction.indices[payload.faction]
    if faction then
        newPayload.money = faction.startMoney or 100
    end
end
```

### CharacterPreSave

Called before character is saved to database.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:411`

```lua
function PLUGIN:CharacterPreSave(character)
    if SERVER then
        local client = character:GetPlayer()
        if IsValid(client) then
            -- Save current health
            character:SetData("savedHealth", client:Health())
        end
    end
end
```

### CharacterPostSave

Called after character was saved.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:405`

```lua
function PLUGIN:CharacterPostSave(character)
    print("Character saved:", character:GetID())
end
```

### CharacterRestored

Called after character was restored from database.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:419`

```lua
function PLUGIN:CharacterRestored(character)
    -- Character fully loaded from database
    local data = character:GetData("customData")
end
```

### CharacterDeleted

Called when a character is deleted.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:393`

```lua
function PLUGIN:CharacterDeleted(client, id, isCurrentChar)
    if SERVER then
        print(client:Name() .. " deleted character " .. id)
    end
end
```

### PreCharacterDeleted

Called before character deletion.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:988`

```lua
function PLUGIN:PreCharacterDeleted(client, character)
    -- Cleanup character-specific data
    local charID = character:GetID()
    self:RemoveCharacterData(charID)
end
```

### OnCharacterDisconnect

Called when player with active character disconnects.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:675`

```lua
function PLUGIN:OnCharacterDisconnect(client, character)
    -- Save character state before disconnect
    character:SetData("lastPlayed", os.time())
end
```

### CharacterVarChanged

Called when a character variable changes.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:425`

```lua
function PLUGIN:CharacterVarChanged(character, key, oldVar, value)
    if key == "money" then
        print("Money changed from", oldVar, "to", value)
    end
end
```

### CanPlayerUseCharacter

Whether player can load/use a character.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:331`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerUseCharacter(client, character)
    if character:GetData("locked") then
        return false
    end
end
```

---

## Player Hooks

### PlayerInitialSpawn

Called when player first joins (before character selection).

**Realm**: Server

```lua
function PLUGIN:PlayerInitialSpawn(client)
    -- Player connected but hasn't selected character yet
    timer.Simple(1, function()
        if IsValid(client) then
            client:ChatPrint("Welcome to the server!")
        end
    end)
end
```

### PlayerDisconnected

Called when player disconnects.

**Realm**: Server

```lua
function PLUGIN:PlayerDisconnected(client)
    local character = client:GetCharacter()
    if character then
        print(character:GetName() .. " disconnected")
    end
end
```

### PostPlayerLoadout

Called after player's loadout is set.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:969`

```lua
function PLUGIN:PostPlayerLoadout(client)
    -- Modify player after spawn
    client:SetRunSpeed(250)
    client:SetWalkSpeed(150)
end
```

### PlayerModelChanged

Called when player's model changes.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:856`

```lua
function PLUGIN:PlayerModelChanged(client, oldModel)
    print("Model changed from", oldModel, "to", client:GetModel())
end
```

### OnPlayerObserve

Called when player enters/exits observer mode.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:738`

```lua
function PLUGIN:OnPlayerObserve(client, state)
    if state then
        print(client:Name() .. " entered observer")
    else
        print(client:Name() .. " exited observer")
    end
end
```

### CanPlayerEnterObserver

Whether player can enter observer mode.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:161`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerEnterObserver(client)
    -- Only allow superadmins
    if not client:IsSuperAdmin() then
        return false
    end
end
```

### OnPlayerRestricted

Called when player is restricted (tied up).

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:763`

```lua
function PLUGIN:OnPlayerRestricted(client)
    client:ChatPrint("You have been restrained!")
end
```

### OnPlayerUnRestricted

Called when player is unrestricted.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:769`

```lua
function PLUGIN:OnPlayerUnRestricted(client)
    client:ChatPrint("You are no longer restrained.")
end
```

---

## Item & Inventory Hooks

### CanPlayerInteractItem

Whether player can interact with an item.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:213`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerInteractItem(client, action, item, data)
    if action == "drop" and item.important then
        return false -- Can't drop important items
    end
end
```

### PlayerInteractItem

Called when player interacts with an item.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:799`

```lua
function PLUGIN:PlayerInteractItem(client, action, item)
    if action == "use" then
        print(client:Name() .. " used " .. item.name)
    end
end
```

### CanPlayerDropItem

Whether player can drop an item.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:136`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerDropItem(client, item)
    if item.nodrop then
        return false
    end
end
```

### CanPlayerTakeItem

Whether player can pick up an item entity.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:274`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerTakeItem(client, item)
    if client:GetMoveType() == MOVETYPE_NOCLIP then
        return false -- Can't take items in noclip
    end
end
```

### CanPlayerEquipItem

Whether player can equip an item.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:170`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerEquipItem(client, item)
    local character = client:GetCharacter()
    if character:GetAttribute("str", 0) < item.strength then
        return false -- Not strong enough
    end
end
```

### CanPlayerUnequipItem

Whether player can unequip an item.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:308`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerUnequipItem(client, item)
    if item.permanent then
        return false
    end
end
```

### OnItemTransferred

Called when item moves between inventories.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:711`

```lua
function PLUGIN:OnItemTransferred(item, curInv, inventory)
    if SERVER then
        print("Item moved from inv", curInv:GetID(), "to", inventory:GetID())
    end
end
```

### CanTransferItem

Whether item can be transferred.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:381`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanTransferItem(item, currentInv, oldInv)
    if item.locked then
        return false
    end
end
```

### OnItemSpawned

Called when item entity spawns in world.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:700`

```lua
function PLUGIN:OnItemSpawned(entity)
    local item = entity:GetItemTable()
    print("Spawned item:", item.name)
end
```

### InventoryItemAdded

Called when item added to inventory.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:618`

```lua
function PLUGIN:InventoryItemAdded(oldInv, inventory, item)
    print("Item added:", item.uniqueID)
end
```

### InventoryItemRemoved

Called when item removed from inventory.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:626`

```lua
function PLUGIN:InventoryItemRemoved(inventory, item)
    print("Item removed:", item.uniqueID)
end
```

### CanPlayerCombineItem

Whether player can combine items.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:103`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerCombineItem(client, item, other)
    local otherItem = ix.item.instances[other]
    if otherItem and otherItem.uniqueID == "invalid" then
        return false
    end
end
```

---

## Chat & Communication

### PlayerSay

Called when player sends chat message.

**Realm**: Server

**Returns**: `string` to modify text, `""` to block

```lua
function PLUGIN:PlayerSay(client, text, team)
    if text:lower() == "!help" then
        client:ChatPrint("Help menu here!")
        return "" -- Block original message
    end
end
```

### PlayerMessageSend

Called when message is about to be sent to other players.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:841`

**Returns**: `string` to replace text

```lua
function PLUGIN:PlayerMessageSend(speaker, chatType, text, anonymous, receivers, rawText)
    -- Add timestamp to messages
    return "[" .. os.date("%H:%M") .. "] " .. text
end
```

### PrePlayerMessageSend

Called before message is processed (earliest).

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1003`

**Returns**: `false` to block message

```lua
function PLUGIN:PrePlayerMessageSend(client, chatType, message, bAnonymous)
    if not client:GetCharacter() then
        return false -- Can't chat without character
    end
end
```

### PostPlayerSay

Called after message was sent.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:975`

```lua
function PLUGIN:PostPlayerSay(client, chatType, message, anonymous)
    -- Log chat messages
    print(client:Name() .. " said:", message)
end
```

### CanAutoFormatMessage

Whether message can be auto-formatted.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:45`

**Returns**: `false` to prevent formatting

```lua
function PLUGIN:CanAutoFormatMessage(speaker, chatType, text)
    if chatType == "ooc" then
        return false -- Don't format OOC
    end
end
```

### GetCharacterName

Override character name display in chat.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:506`

**Returns**: `string` for custom name

```lua
function PLUGIN:GetCharacterName(speaker, chatType)
    local character = speaker:GetCharacter()
    if character and character:GetData("disguised") then
        return "Unknown Person"
    end
end
```

### InitializedChatClasses

Called after chat classes are registered.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:586`

```lua
function PLUGIN:InitializedChatClasses()
    -- Register custom chat class
    ix.chat.Register("shout", {
        format = "%s shouts \"%s\"",
        GetColor = function() return Color(255, 100, 100) end,
        CanHear = function(self, speaker, listener)
            return speaker:GetPos():Distance(listener:GetPos()) < 1000
        end
    })
end
```

---

## Door & Entity Interaction

### CanPlayerUseDoor

Whether player can use a door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:343`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerUseDoor(client, entity)
    if entity:GetClass() == "prop_door_rotating" then
        -- Check if player has key
        if not client:GetCharacter():GetData("hasKey") then
            return false
        end
    end
end
```

### PlayerUseDoor

Called when player uses a door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:901`

```lua
function PLUGIN:PlayerUseDoor(client, entity)
    print(client:Name() .. " used door")
end
```

### CanPlayerAccessDoor

Whether player can access door abilities (lock/unlock/etc).

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:90`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerAccessDoor(client, door, access)
    -- access levels: 0=use, 1=lock/unlock, 2=purchase
    if access == 2 and not client:IsAdmin() then
        return false -- Only admins can buy doors
    end
end
```

### PlayerLockedDoor

Called when player locks a door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:826`

```lua
function PLUGIN:PlayerLockedDoor(client, door, partner)
    client:ChatPrint("Door locked")
end
```

### PlayerUnlockedDoor

Called when player unlocks a door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:879`

```lua
function PLUGIN:PlayerUnlockedDoor(client, door, partner)
    client:ChatPrint("Door unlocked")
end
```

### OnPlayerPurchaseDoor

Called when player buys/sells a door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:753`

```lua
function PLUGIN:OnPlayerPurchaseDoor(client, entity, bBuying, bCallOnDoorChild)
    if bBuying then
        print(client:Name() .. " bought a door")
    else
        print(client:Name() .. " sold a door")
    end
end
```

### CanPlayerKnock

Whether player can knock on door.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:240`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerKnock(client, entity)
    -- Can only knock on owned doors
    if not entity.ixDoorOwner then
        return false
    end
end
```

### CanPlayerInteractEntity

Whether player can interact with entity.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:194`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerInteractEntity(client, entity, option, data)
    if client:GetNetVar("tied") then
        return false -- Can't interact while tied
    end
end
```

### PlayerInteractEntity

Called when player interacts with entity.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:789`

```lua
function PLUGIN:PlayerInteractEntity(client, entity, option, data)
    print(client:Name() .. " selected option:", option)
end
```

### OnPlayerOptionSelected

Called when player selects player interaction menu option.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:745`

```lua
function PLUGIN:OnPlayerOptionSelected(client, callingClient, option)
    if option == "recognize" then
        -- Handle recognition
    end
end
```

### PlayerUse

Called when player uses an entity.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:894`

```lua
function PLUGIN:PlayerUse(client, entity)
    print(client:Name() .. " used", entity:GetClass())
end
```

### CanPlayerHoldObject

Whether player can hold object with hands SWEP.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:183`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerHoldObject(client, entity)
    if client:GetMoveType() == MOVETYPE_NOCLIP then
        return false
    end
end
```

---

## Combat & Damage

### PlayerDeath

Called when player dies.

**Realm**: Server

```lua
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    if IsValid(attacker) and attacker:IsPlayer() then
        print(attacker:Name() .. " killed " .. victim:Name())
    end
end
```

### GetPlayerDeathSound

Override player death sound.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:530`

**Returns**: `string` for custom sound, `false` for no sound

```lua
function PLUGIN:GetPlayerDeathSound(client)
    -- Custom death sound
    return "npc/zombie/zombie_die1.wav"

    -- OR disable death sound:
    -- return false
end
```

### ShouldPermakillCharacter

Whether character should be permakilled on death.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1074`

**Returns**: `false` to prevent permakill

```lua
function PLUGIN:ShouldPermakillCharacter(client, character, inflictor, attacker)
    if client:IsAdmin() then
        return false -- Admins don't permakill
    end
end
```

### ShouldSpawnClientRagdoll

Whether to spawn ragdoll on death.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1125`

**Returns**: `false` to prevent

```lua
function PLUGIN:ShouldSpawnClientRagdoll(client)
    if client:GetNetVar("noragdoll") then
        return false
    end
end
```

### ShouldRemoveRagdollOnDeath

Whether to remove ragdoll immediately.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1099`

**Returns**: `true` to remove

```lua
function PLUGIN:ShouldRemoveRagdollOnDeath(client)
    return true -- Remove ragdolls immediately
end
```

### PlayerHurt

Called when player takes damage.

**Realm**: Server

```lua
function PLUGIN:PlayerHurt(client, attacker, health, damage)
    if damage > 50 then
        client:ChatPrint("Heavy damage!")
    end
end
```

### CanPlayerTakeDamage

Whether player can take damage.

**Realm**: Server

**Returns**: `false` to prevent damage

```lua
function PLUGIN:CanPlayerTakeDamage(client, attacker)
    if client:GetNetVar("godmode") then
        return false
    end
end
```

### GetPlayerPainSound

Override player pain sound.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:555`

**Returns**: `string` for custom sound

```lua
function PLUGIN:GetPlayerPainSound(client)
    return "vo/npc/male01/pain01.wav"
end
```

### CanPlayerThrowPunch

Whether player can punch with hands SWEP.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:285`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerThrowPunch(client)
    if client:GetCharacter():GetAttribute("str", 0) < 1 then
        return false
    end
end
```

### PlayerThrowPunch

Called when player punches.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:875`

```lua
function PLUGIN:PlayerThrowPunch(client, trace)
    if SERVER and IsValid(trace.Entity) then
        print(client:Name() .. " punched", trace.Entity)
    end
end
```

### GetPlayerPunchDamage

Modify punch damage.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:565`

**Returns**: `number` for custom damage

```lua
function PLUGIN:GetPlayerPunchDamage(client, damage, context)
    local str = client:GetCharacter():GetAttribute("str", 0)
    return damage + (str * 2) -- Strength increases punch damage
end
```

### ShouldPlayerDrowned

Whether player should drown underwater.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1089`

**Returns**: `false` to prevent drowning

```lua
function PLUGIN:ShouldPlayerDrowned(client)
    if client:GetCharacter():HasFlags("d") then
        return false -- Flag 'd' prevents drowning
    end
end
```

---

## UI & Client Hooks

### HUDPaint

Called every frame to draw HUD.

**Realm**: Client

```lua
function PLUGIN:HUDPaint()
    draw.SimpleText("Custom HUD", "Default", ScrW() / 2, 50, Color(255, 255, 255), TEXT_ALIGN_CENTER)
end
```

### CreateMenuButtons

Add buttons to F1 menu.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:464`

```lua
function PLUGIN:CreateMenuButtons(tabs)
    tabs["mybutton"] = function(panel)
        -- Create your menu content
        local button = panel:Add("DButton")
        button:SetText("My Custom Button")
        button:Dock(TOP)
        button:DockMargin(0, 0, 0, 8)
    end
end
```

### CreateCharacterInfo

Add panels to character info (tab menu).

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:453`

```lua
function PLUGIN:CreateCharacterInfo(panel)
    local custom = panel:Add("DLabel")
    custom:SetText("Custom Info")
    custom:Dock(TOP)
end
```

### CanCreateCharacterInfo

Suppress default character info panels.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:57`

```lua
function PLUGIN:CanCreateCharacterInfo(suppress)
    suppress.attributes = true -- Hide attributes panel
    suppress.money = true -- Hide money display
end
```

### OnCharacterMenuCreated

Called when character menu is created.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:696`

```lua
function PLUGIN:OnCharacterMenuCreated(panel)
    -- Customize character menu
    panel:SetTitle("Custom Title")
end
```

### CreateItemInteractionMenu

Add options to item right-click menu.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:460`

```lua
function PLUGIN:CreateItemInteractionMenu(icon, menu, itemTable)
    menu:AddOption("Custom Action", function()
        net.Start("CustomItemAction")
            net.WriteUInt(itemTable:GetID(), 32)
        net.SendToServer()
    end)
end
```

### PopulateItemTooltip

Add info to item tooltip.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:940`

```lua
function PLUGIN:PopulateItemTooltip(tooltip, item)
    local row = tooltip:AddRow("weight")
    row:SetText("Weight")
    row:SetValue(item.weight .. " kg")
end
```

### CanDrawAmmoHUD

Whether to draw ammo HUD.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:78`

**Returns**: `false` to hide

```lua
function PLUGIN:CanDrawAmmoHUD(weapon)
    if weapon:GetClass() == "ix_hands" then
        return false
    end
end
```

### ShouldDrawCrosshair

Whether to draw crosshair.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:1061`

**Returns**: `false` to hide

```lua
function PLUGIN:ShouldDrawCrosshair(client, weapon)
    return false -- Hide crosshair entirely
end
```

### ShouldHideBars

Whether to hide status bars.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:1069`

**Returns**: `true` to hide

```lua
function PLUGIN:ShouldHideBars()
    if LocalPlayer():GetMoveType() == MOVETYPE_NOCLIP then
        return true -- Hide bars in noclip
    end
end
```

### CanPlayerViewInventory

Whether player can open inventory.

**Realm**: Client
**Reference**: `docs/hooks/plugin.lua:361`

**Returns**: `false` to prevent

```lua
function PLUGIN:CanPlayerViewInventory()
    if LocalPlayer():GetNetVar("tied") then
        return false
    end
end
```

---

## Data & Persistence

### LoadData

Called when server loads data.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:645`

```lua
function PLUGIN:LoadData()
    -- Load plugin data from disk
    self.myData = self:GetData().myData or {}
end
```

### SaveData

Called when server saves data.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:1019`

```lua
function PLUGIN:SaveData()
    -- Save plugin data to disk
    local data = self:GetData()
    data.myData = self.myData
    self:SetData(data)
end
```

### PostLoadData

Called after data is loaded.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:964`

```lua
function PLUGIN:PostLoadData()
    -- Process loaded data
    print("Data loaded successfully")
end
```

---

## Configuration & Setup

### InitializedConfig

Called after configuration is loaded.

**Realm**: Shared
**Reference**: `docs/hooks/plugin.lua:606`

```lua
function PLUGIN:InitializedConfig()
    -- Config options are now available
    if ix.config.Get("myCustomConfig") then
        self:Enable()
    end
end
```

### DatabaseConnected

Called when database connection succeeds.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:475`

```lua
function PLUGIN:DatabaseConnected()
    print("Database ready!")
    self:InitializeTables()
end
```

### DatabaseConnectionFailed

Called when database connection fails.

**Realm**: Server
**Reference**: `docs/hooks/plugin.lua:479`

```lua
function PLUGIN:DatabaseConnectionFailed(error)
    ErrorNoHalt("Database error: " .. error)
end
```

---

## See Also

- [Creating Plugins](creating-plugins.md) - Step-by-step tutorial
- [Plugin System](plugin-system.md) - System overview
- [Plugin Structure](plugin-structure.md) - Directory organization
- [Best Practices](best-practices.md) - Development guidelines
- Source: `docs/hooks/plugin.lua`
- Source: `gamemode/core/libs/sh_plugin.lua`
