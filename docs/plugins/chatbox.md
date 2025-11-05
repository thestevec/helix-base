# Chatbox Plugin

> **Reference**: `plugins/chatbox/`

The Chatbox plugin replaces Garry's Mod's default chat interface with a customizable, feature-rich chatbox. It provides command autocomplete, chat history, customizable tabs and filters, adjustable sizing and positioning, timestamps, and much more for an enhanced roleplay communication experience.

## ⚠️ Important: Use Built-in Chatbox

**This plugin automatically replaces the default chat**. The framework provides:
- Automatic command autocomplete
- Chat history (up arrow to recall previous messages)
- Customizable tabs with filters
- Adjustable font size and styling
- Resizable and repositionable window
- Timestamps and notice integration
- Tab-based message filtering
- Integration with all Helix chat systems

## Core Concepts

### What is the Chatbox?

The chatbox is a custom VGUI panel that:
- Displays all chat messages and IC/OOC communications
- Provides command suggestions as you type
- Remembers your chat history
- Allows custom tabs for filtering messages
- Integrates with Helix's chat types
- Saves preferences per-client

### Key Features

- **Command Autocomplete**: Shows available commands as you type `/`
- **Chat History**: Press UP arrow to recall previous messages
- **Custom Tabs**: Create tabs to filter specific chat types
- **Resizable**: Drag corners to resize the chatbox
- **Moveable**: Drag to reposition anywhere on screen
- **Font Scaling**: Adjust text size for readability
- **Timestamps**: Optional timestamps on messages
- **Notice Integration**: System notices can appear in chat
- **Text Outline**: Optional text outlines for better contrast

## Client Options

### chatNotices

**Reference**: `plugins/chatbox/sh_plugin.lua:13-15`

- **Type**: Boolean
- **Default**: false
- **Description**: Show system notices in the chatbox

```lua
-- In F1 menu > Options > Chat
-- Or via console:
ix_option_chatNotices "1"  -- Enable
ix_option_chatNotices "0"  -- Disable
```

### chatTimestamps

**Reference**: `plugins/chatbox/sh_plugin.lua:17-19`

- **Type**: Boolean
- **Default**: false
- **Description**: Show timestamps on chat messages

```lua
ix_option_chatTimestamps "1"  -- Enable timestamps
```

### chatFontScale

**Reference**: `plugins/chatbox/sh_plugin.lua:21-27`

- **Type**: Number
- **Default**: 1.0
- **Range**: 0.1 to 2.0
- **Description**: Scale factor for chat font size

```lua
ix_option_chatFontScale "1.5"   -- 150% size
ix_option_chatFontScale "0.8"   -- 80% size
```

### chatOutline

**Reference**: `plugins/chatbox/sh_plugin.lua:29-31`

- **Type**: Boolean
- **Default**: false
- **Description**: Add outline to chat text for better readability

```lua
ix_option_chatOutline "1"  -- Enable text outlines
```

### chatTabs

**Reference**: `plugins/chatbox/sh_plugin.lua:34-39`

- **Type**: String (JSON)
- **Hidden**: true (managed by UI)
- **Description**: Stores custom tab configurations

### chatPosition

**Reference**: `plugins/chatbox/sh_plugin.lua:42-47`

- **Type**: String (JSON)
- **Hidden**: true (managed by UI)
- **Description**: Stores chatbox position and size

## Using the Chatbox

### Opening Chat

```lua
-- Press Y (default) or T to open chat
-- Or programmatically:
PLUGIN.panel:SetActive(true)
```

### Chat History

```lua
-- While chat is open:
-- Press UP arrow: Previous message
-- Press DOWN arrow: Next message (if you went back)
-- Cycles through all previously sent messages
```

### Command Autocomplete

```lua
-- Type "/" to see available commands
-- Continue typing to filter commands
-- Use arrow keys to navigate suggestions
-- Press TAB or ENTER to select command
```

### Creating Custom Tabs

1. Click the `+` button on the chatbox
2. Name your tab
3. Set up filters for what messages to show
4. Tab saves automatically

### Resizing Chatbox

1. Hover over bottom-right corner
2. Drag to resize
3. Position saves automatically

### Moving Chatbox

1. Click and drag the chatbox
2. Position anywhere on screen
3. Position saves automatically

## Developer Functions

### Adding Messages

**Reference**: `plugins/chatbox/sh_plugin.lua:127-150`

```lua
-- Client-side only
chat.AddText(...)

-- Examples:
chat.AddText("Simple message")
chat.AddText(Color(255, 0, 0), "Red message")
chat.AddText(player, " says: ", Color(255, 255, 0), "Hello!")
```

**Parameters**:
- Varargs of colors, strings, players, or other types
- Colors apply to subsequent text
- Player entities automatically converted to colored name
- All messages also logged to console

### Custom Panel Access

**Reference**: `plugins/chatbox/sh_plugin.lua:49-59`

```lua
-- Access the chatbox panel
local chatbox = PLUGIN.panel

-- Check if chatbox is valid
if IsValid(PLUGIN.panel) then
    -- Do something with chatbox
end
```

### Hook: ChatboxCreated

**Reference**: `plugins/chatbox/sh_plugin.lua:58`

```lua
-- Called after chatbox is created/recreated
function PLUGIN:ChatboxCreated()
    -- Access the chatbox
    local chatbox = PLUGIN.panel

    -- Add custom functionality
end
```

## Complete Examples

### Example 1: Custom Chat Tab for Faction

```lua
-- In your plugin cl_hooks.lua
function PLUGIN:ChatboxCreated()
    local chatbox = PLUGIN.panel

    -- Add a faction-only tab
    -- (This would typically be done through the UI)
    -- Example shows the concept
end
```

### Example 2: Colored Announcement

```lua
-- Server-side: Send colored message to all players
local function SendAnnouncement(message)
    local color = ix.config.Get("color")

    for _, client in ipairs(player.GetAll()) do
        client:ChatPrint(message)  -- Or use chat.AddText on client
    end
end

-- Client-side: Display in chat
hook.Add("SomeEvent", "ShowAnnouncement", function()
    chat.AddText(
        Color(255, 215, 0),
        "[ANNOUNCEMENT] ",
        color_white,
        "Server restart in 5 minutes!"
    )
end)
```

### Example 3: Chat History Programmatically

```lua
-- Access chat history
local history = ix.chat.history

-- Get last message sent
local lastMessage = history[#history]

-- Add to history programmatically
table.insert(ix.chat.history, "Some command")
```

## Best Practices

### ✅ DO

- Use chat.AddText() for client-side messages
- Provide color for important messages
- Keep messages concise
- Use existing chat types when possible
- Let players customize their chatbox
- Respect chatNotices option for system messages

### ❌ DON'T

- Don't spam chat with too many messages
- Don't use excessive colors (readability)
- Don't override the chatbox without good reason
- Don't forget client-side vs server-side context
- Don't hardcode chat positions/sizes
- Don't disable player customization options

## Common Patterns

### Pattern 1: Adding Colored Message

```lua
-- Client-side
chat.AddText(
    Color(100, 200, 255),
    "[Info] ",
    color_white,
    "This is an informational message."
)
```

### Pattern 2: Player Name in Message

```lua
-- Client-side
local ply = LocalPlayer()
chat.AddText(
    ply,  -- Automatically colored by team
    " has joined the server!"
)
```

### Pattern 3: Multi-Color Message

```lua
chat.AddText(
    Color(255, 0, 0), "Error: ",
    Color(255, 255, 255), "Could not find ",
    Color(255, 255, 0), "target player",
    Color(255, 255, 255), "."
)
```

## Common Issues

### Chatbox Not Appearing

**Cause**: Resolution change or chatbox removed
**Fix**: Recreate chatbox

```lua
-- Console command
lua_run_cl PLUGIN:CreateChat()
```

### Chat History Not Saving

**Cause**: History is session-only, not persistent
**Fix**: This is intended behavior. History clears on disconnect.

### Custom Tabs Disappeared

**Cause**: Option data corruption or reset
**Fix**: Recreate tabs through UI. They save automatically.

### Messages Not Showing in Custom Tab

**Cause**: Filter not configured correctly
**Fix**: Check tab filter settings match message types you want to see.

### Chatbox Position Reset

**Cause**: Resolution change or option reset
**Fix**: Reposition and it will save automatically.

## Technical Details

### Network Protocol

**Reference**: `plugins/chatbox/sh_plugin.lua:152-167`

- `ixChatMessage`: Client sends chat messages to server
- Rate limited to one message per 0.5 seconds
- Max length enforced via `chatMax` config
- Automatically truncates over-length messages

### Chat History Storage

**Reference**: `plugins/chatbox/sh_plugin.lua:9`

- Stored in `ix.chat.history` table
- Array of strings
- Session-only (not persistent)
- Used for UP/DOWN arrow navigation

### Current Command Tracking

**Reference**: `plugins/chatbox/sh_plugin.lua:10-11`

- `ix.chat.currentCommand`: String of current command being typed
- `ix.chat.currentArguments`: Table of parsed arguments
- Updated in real-time as player types
- Used by autocomplete and other plugins

### Panel Recreation

**Reference**: `plugins/chatbox/sh_plugin.lua:49-59`

Chatbox is recreated when:
- Font scale changes
- Screen resolution changes
- InitPostEntity (first load)
- Manually triggered

### Integration with Default Chat

**Reference**: `plugins/chatbox/sh_plugin.lua:110-114`

- Hides default GMod chatbox (`CHudChat`)
- Intercepts messagemode bind
- Redirects all chat through custom panel
- Logs all messages to console

## Customization

### Custom Tabs via UI

1. Click `+` button on chatbox
2. Enter tab name
3. Configure filters:
   - Include specific chat types
   - Exclude others
   - Use wildcards for patterns
4. Tab appears and auto-saves

### Adjusting Appearance

All appearance settings in F1 Options > Chat:
- Font Scale: Size of text
- Timestamps: Show message times
- Outlines: Text outline for contrast
- Notices: Show system notices in chat

## See Also

- [Chat System](../systems/chat.md) - Creating custom chat types
- [Commands System](../systems/commands.md) - Chat command system
- [Typing Plugin](typing.md) - Shows typing indicators
- [Plugin System](plugin-system.md) - Understanding Helix plugins
- Source: `plugins/chatbox/`
