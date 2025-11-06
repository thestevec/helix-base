# Helix Plugin Development - AI Assistant Guide

> Context-specific guidance for developing plugins in the Helix framework.

## 📚 Documentation Knowledgebase

**Primary Reference**: [docs/llms.txt](../../llms.txt) - Complete framework documentation index

## Plugin Development Essentials

### Core Documentation
1. **[Plugin System](../../plugins/plugin-system.md)** - Architecture, loading mechanism, and lifecycle
2. **[Creating Plugins](../../plugins/creating-plugins.md)** - Step-by-step development guide
3. **[Plugin Structure](../../plugins/plugin-structure.md)** - Directory organization and file layout
4. **[Hook System](../../plugins/hooks.md)** - Available hooks and event handling
5. **[Plugin Best Practices](../../plugins/best-practices.md)** - Quality and performance guidelines

### Plugin Structure Template

```
plugins/[pluginname]/
├── sh_plugin.lua          # Plugin definition (shared)
├── sv_plugin.lua          # Server-side logic (optional)
├── cl_plugin.lua          # Client-side logic (optional)
├── sv_hooks.lua           # Server hooks (optional)
├── cl_hooks.lua           # Client hooks (optional)
├── meta/                  # Meta table extensions
├── libs/                  # Libraries
├── commands/              # Commands
├── items/                 # Item definitions
├── entities/              # Entity definitions
└── derma/                 # UI elements
```

### Quick API Reference for Plugins

**Plugin Registration** ([api](../../api/plugin.md)):
- `PLUGIN:SetName(name)` - Set plugin display name
- `PLUGIN:SetAuthor(author)` - Set plugin author
- `PLUGIN:SetDescription(desc)` - Set plugin description

**Commonly Used Systems**:
- **Commands**: [ix.command](../../api/command.md) - Register custom commands
- **Configuration**: [ix.config](../../api/config.md) - Add config variables
- **Items**: [ix.item](../../api/item.md) - Register item types
- **Hooks**: [hooks.md](../../plugins/hooks.md) - Plugin lifecycle and game events
- **Networking**: [ix.net](../../api/networking.md) - Client-server communication

### Development Workflow

#### 1. Planning Phase
- [ ] Check if functionality exists in base framework
- [ ] Review similar plugins in [base plugins list](../../llms.txt#L121-L148)
- [ ] Identify required hooks and APIs
- [ ] Plan configuration options

#### 2. Implementation Phase
- [ ] Create plugin directory structure
- [ ] Define plugin in sh_plugin.lua
- [ ] Implement server logic (sv_*.lua)
- [ ] Implement client logic (cl_*.lua)
- [ ] Add configuration options
- [ ] Register commands if needed
- [ ] Add networking if needed

#### 3. Testing Phase
- [ ] Test on local server
- [ ] Test with other plugins enabled
- [ ] Test client/server sync
- [ ] Test error handling
- [ ] Test performance impact

### Common Plugin Patterns

#### Adding Configuration
```lua
ix.config.Add("pluginSetting", default, "Description", callback, {
    category = "plugin"
})
```
See: [ix.config API](../../api/config.md)

#### Registering Commands
```lua
ix.command.Add("CommandName", {
    description = "Description",
    OnRun = function(self, client)
        -- Command logic
    end
})
```
See: [Command System](../../systems/commands.md), [Creating Commands Guide](../../guides/commands.md)

#### Creating Custom Items
```lua
ITEM.name = "Item Name"
ITEM.description = "Item description"
ITEM.model = "models/path/to/model.mdl"
```
See: [Creating Custom Items](../../guides/custom-items.md), [Item Types](../../llms.txt#L103-L111)

#### Using Hooks
```lua
function PLUGIN:HookName(arguments)
    -- Hook logic
    -- Return values affect behavior
end
```
See: [Hook System](../../plugins/hooks.md), [Hook Usage Guide](../../guides/hooks.md)

### Relevant Examples
- [Example Plugin](../../examples/plugin.md) - Complete plugin implementation
- [Example Command](../../examples/command.md) - Command with validation
- [Example Item](../../examples/item.md) - Custom item implementation
- [Example Hook Usage](../../examples/hooks.md) - Common hook patterns

### Troubleshooting

**Plugin not loading?**
- Check sh_plugin.lua exists and defines PLUGIN
- Verify file naming (sh_ prefix for shared)
- Check console for Lua errors

**Hooks not firing?**
- Verify hook name spelling
- Check if hook exists in [Hook System](../../plugins/hooks.md)
- Ensure correct realm (server/client/shared)

**Network issues?**
- Register net messages before use
- Match send/receive realms correctly
- See [Networking Guide](../../guides/networking.md)

### Base Plugins Reference

Study these base plugins for examples:
- **Simple**: [3D Text](../../plugins/3dtext.md), [Crosshair](../../plugins/crosshair.md)
- **Medium**: [Doors](../../plugins/doors.md), [Containers](../../plugins/containers.md)
- **Complex**: [Business](../../systems/business.md), [Persistence](../../plugins/persistence.md)

Full list: [llms.txt lines 121-148](../../llms.txt#L121-L148)

### Best Practices Checklist

- [ ] Use PLUGIN table for plugin-specific functions
- [ ] Prefix config variables with plugin name
- [ ] Add proper error handling and validation
- [ ] Minimize network messages
- [ ] Cache frequently-used data
- [ ] Clean up resources on plugin unload
- [ ] Document configuration options
- [ ] Follow Helix coding standards ([best-practices.md](../../best-practices.md))

---

**Quick Links**:
- [Back to Main Guide](../../../CLAUDE.md)
- [Complete Documentation Index](../../llms.txt)
- [Plugin Development Examples](../../examples/)
