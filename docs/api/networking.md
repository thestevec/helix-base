# Networking API (ix.net)

> **Reference**: `gamemode/core/libs/sv_networking.lua`, `gamemode/core/libs/cl_networking.lua`

The networking API provides extensions to Garry's Mod's networking system with automatic compression and entity streaming.

## Core Concepts

Helix networking:
- Extends net library
- Compresses large data
- Handles entity streaming
- Manages inventory sync
- Character data sync

## Functions

Network operations use standard GMod net library:

```lua
-- Server to client
net.Start("myMessage")
    net.WriteString("data")
    net.WriteInt(123, 32)
net.Send(client)

-- Client to server
net.Start("myMessage")
    net.WriteString("data")
net.SendToServer()

-- Receive
net.Receive("myMessage", function(len, client)
    local data = net.ReadString()
    print(data)
end)
```

## Helix Networking

Framework uses these internal net messages:
- `ixInventoryData` - Inventory sync
- `ixInventoryAdd` - Item added
- `ixInventoryRemove` - Item removed
- `ixCharacterData` - Character sync
- `ixCharacterVar` - Character variable update
- `ixNotify` - Notifications
- `ixChatMessage` - Chat messages

## Best Practices

### ✅ DO
- Use util.AddNetworkString on server
- Keep net messages small
- Use appropriate data types
- Check IsValid before networking entities

### ❌ DON'T
- Don't send large tables every frame
- Don't forget to register net strings
- Don't send unnecessary data
- Don't spam net messages

## See Also

- [Character API](character.md) - Character sync
- [Inventory API](inventory.md) - Inventory sync
- Source: `gamemode/core/libs/sv_networking.lua`
