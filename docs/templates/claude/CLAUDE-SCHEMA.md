# Helix Schema Development - AI Assistant Guide

> Context-specific guidance for developing schemas (game modes) in the Helix framework.

## 📚 Documentation Knowledgebase

**Primary Reference**: [docs/llms.txt](../../llms.txt) - Complete framework documentation index

## Schema Development Essentials

### Core Documentation
1. **[Schema Structure](../../schema/structure.md)** - Organization and file layout
2. **[Schema Configuration](../../schema/configuration.md)** - Schema-specific settings
3. **[Creating Factions](../../schema/factions.md)** - Faction definitions
4. **[Creating Classes](../../schema/classes.md)** - Class configuration
5. **[Creating Items](../../schema/items.md)** - Schema-specific items

### Schema Structure Template

```
schema/
├── sh_schema.lua          # Schema definition (shared)
├── sv_schema.lua          # Server-side initialization
├── cl_schema.lua          # Client-side initialization
├── factions/              # Faction definitions
│   ├── sh_citizen.lua
│   └── sh_faction2.lua
├── classes/               # Class definitions
│   ├── sh_class1.lua
│   └── sh_class2.lua
├── items/                 # Schema-specific items
│   ├── sh_item1.lua
│   └── categories/
├── commands/              # Schema commands
├── hooks/                 # Schema hooks
├── derma/                 # Schema UI elements
├── plugins/               # Schema-specific plugins
└── config.lua             # Schema configuration
```

### Schema Definition (sh_schema.lua)

```lua
SCHEMA.name = "Schema Name"
SCHEMA.author = "Author Name"
SCHEMA.description = "Schema description"

-- Include files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")

-- Auto-include directories
ix.util.IncludeDir("libs")
ix.util.IncludeDir("meta")
ix.util.IncludeDir("hooks")
```

### Creating Factions

**Essential Reading**: [Creating Factions](../../schema/factions.md), [Faction System](../../systems/factions.md)

**API Reference**: [ix.faction](../../api/faction.md)

```lua
FACTION.name = "Faction Name"
FACTION.description = "Faction description"
FACTION.color = Color(r, g, b)
FACTION.models = {"models/player/group01/male_01.mdl"}
FACTION.weapons = {"weapon_pistol"}
FACTION.isDefault = true  -- Default faction for new players

FACTION_CITIZEN = FACTION.index  -- Store faction index for reference
```

**Example**: [Example Faction](../../examples/faction.md)

### Creating Classes

**Essential Reading**: [Creating Classes](../../schema/classes.md), [Class System](../../systems/classes.md)

**API Reference**: [ix.class](../../api/class.md)

```lua
CLASS.name = "Class Name"
CLASS.description = "Class description"
CLASS.faction = FACTION_CITIZEN
CLASS.weapons = {"weapon_smg1"}
CLASS.limit = 4  -- Max players in this class

function CLASS:OnSet(client)
    -- Called when player becomes this class
end

CLASS_MEDIC = CLASS.index  -- Store class index for reference
```

**Example**: [Example Class](../../examples/class.md)

### Creating Schema Items

**Essential Reading**: [Creating Items](../../schema/items.md), [Item System](../../systems/items.md)

**API Reference**: [ix.item](../../api/item.md)

#### Simple Item
```lua
ITEM.name = "Item Name"
ITEM.description = "Item description"
ITEM.model = "models/props/item.mdl"
ITEM.price = 100
ITEM.category = "Category"
```

#### Custom Functionality
```lua
ITEM.functions.Use = {
    OnRun = function(item)
        local client = item.player
        -- Use logic
        return false  -- false = don't remove, true = remove
    end
}
```

**Guides**:
- [Creating Custom Items](../../guides/custom-items.md)
- [Item Types Reference](../../llms.txt#L103-L111)

### Schema-Specific Commands

**Essential Reading**: [Custom Commands](../../schema/commands.md), [Command System](../../systems/commands.md)

```lua
ix.command.Add("SchemaCommand", {
    description = "Command description",
    adminOnly = false,
    OnRun = function(self, client, arguments)
        -- Command logic
    end
})
```

**Guide**: [Creating Commands](../../guides/commands.md)

### Schema Hooks

**Essential Reading**: [Custom Hooks](../../schema/hooks.md), [Hook System](../../plugins/hooks.md)

```lua
function SCHEMA:HookName(arguments)
    -- Hook implementation
end
```

**Examples**: [Hook Usage Examples](../../examples/hooks.md)

### Schema Configuration

**Essential Reading**: [Schema Configuration](../../schema/configuration.md)

```lua
-- In config.lua or sh_schema.lua
ix.config.SetDefault("configName", value)
ix.config.Add("schemaConfig", default, "Description", callback, {
    category = "schema"
})
```

**API**: [ix.config](../../api/config.md)

### Character & Faction Systems

**Key Documentation**:
- [Character System](../../systems/character.md) - Character lifecycle and management
- [Faction System](../../systems/factions.md) - Faction mechanics
- [Class System](../../systems/classes.md) - Class mechanics
- [Attributes](../../systems/attributes.md) - Character progression
- [Inventory System](../../systems/inventory.md) - Inventory management

### UI Customization

**Essential Reading**: [Custom Derma](../../schema/derma.md), [UI Development](../../guides/ui-development.md)

**UI Systems**:
- [Derma Overview](../../ui/derma-overview.md)
- [Character Creation UI](../../ui/character-creation.md)
- [Menu System](../../ui/menus.md)
- [HUD System](../../ui/hud.md)

### Schema Development Workflow

#### 1. Planning Phase
- [ ] Define schema concept and theme
- [ ] Plan factions and their relationships
- [ ] Design class structure per faction
- [ ] List required items and categories
- [ ] Identify needed configuration options
- [ ] Plan custom UI requirements

#### 2. Basic Setup
- [ ] Create schema directory structure
- [ ] Define schema in sh_schema.lua
- [ ] Set up configuration in config.lua
- [ ] Configure database settings

#### 3. Core Implementation
- [ ] Create all factions
- [ ] Create classes for each faction
- [ ] Implement schema-specific items
- [ ] Add custom commands
- [ ] Implement schema hooks
- [ ] Configure spawn points

#### 4. Advanced Features
- [ ] Custom UI elements
- [ ] Schema-specific plugins
- [ ] Advanced progression systems
- [ ] Custom entities
- [ ] Special mechanics

#### 5. Testing & Polish
- [ ] Test faction balance
- [ ] Test class functionality
- [ ] Test item functionality
- [ ] Test character creation flow
- [ ] Performance testing
- [ ] Balance adjustments

### Common Schema Patterns

#### Giving Items on Spawn
```lua
function SCHEMA:PlayerLoadedCharacter(client, character)
    local inventory = character:GetInventory()
    inventory:Add("item_uniqueid")
end
```

#### Custom Character Creation
```lua
function SCHEMA:OnCharacterCreated(client, character)
    -- Initialize character data
    character:SetData("customField", value)
end
```

#### Faction-Specific Logic
```lua
function SCHEMA:PlayerSpawn(client)
    local character = client:GetCharacter()
    local faction = character:GetFaction()

    if faction == FACTION_CITIZEN then
        -- Citizen-specific spawn logic
    end
end
```

### Reference Schemas

Study existing schemas for guidance:
- **HL2 RP Schema**: https://github.com/nebulouscloud/helix-hl2rp
- **Skeleton Schema**: https://github.com/nebulouscloud/helix-skeleton

### Database Considerations

**Essential Reading**: [Database Configuration](../../config/database.md), [Database Operations](../../guides/database.md)

**API**: [ix.db](../../api/database.md)

- Configure MySQL for production
- Use SQLite for development/testing
- Plan custom database tables if needed
- Implement proper data persistence

### Best Practices Checklist

- [ ] Use SCHEMA table for schema-specific functions
- [ ] Organize files by type in appropriate directories
- [ ] Provide default faction for new players
- [ ] Balance faction/class availability
- [ ] Test item economy and balance
- [ ] Implement proper error handling
- [ ] Document custom configuration
- [ ] Follow Helix coding standards ([best-practices.md](../../best-practices.md))
- [ ] Optimize database queries
- [ ] Test with multiple players

### Configuration Reference

**Key Config Files**:
- [Server Configuration](../../config/server.md)
- [Character Settings](../../config/characters.md)
- [Inventory Settings](../../config/inventory.md)
- [Database Configuration](../../config/database.md)

### Troubleshooting

**Faction not appearing?**
- Check faction file is in factions/ directory
- Verify sh_ prefix
- Check for Lua errors in console
- Ensure FACTION.isDefault is set for at least one faction

**Class issues?**
- Verify CLASS.faction matches existing faction
- Check CLASS.limit if class is unavailable
- Ensure class files are properly named

**Items not working?**
- Check item uniqueID is unique
- Verify item category exists
- Check item file location and naming
- Review item base type

---

**Quick Links**:
- [Back to Main Guide](../../../CLAUDE.md)
- [Complete Documentation Index](../../llms.txt)
- [Schema Examples](../../examples/)
- [Getting Started](../../manual/getting-started.md)
