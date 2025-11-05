# Util API (ix.util)

> **Reference**: `gamemode/core/sh_util.lua`

The util API provides various helper functions for common operations including file inclusion, type checking, color manipulation, and string formatting.

## Core Functions

### ix.util.Include

**Realm**: Shared

```lua
ix.util.Include(fileName, realm)
```

Includes a Lua file with automatic realm handling.

**Example**:
```lua
ix.util.Include("libs/sh_helper.lua")
ix.util.Include("sv_admin.lua", "server")
ix.util.Include("cl_hud.lua", "client")
```

### ix.util.IncludeDir

**Realm**: Shared

```lua
ix.util.IncludeDir(directory, bFromLua)
```

Includes all files in a directory.

**Example**:
```lua
ix.util.IncludeDir("libs")
ix.util.IncludeDir("myplugin/libs", true)
```

### ix.util.FindPlayer

**Realm**: Shared

```lua
local player = ix.util.FindPlayer(identifier)
```

Finds player by name, Steam ID, or entity index.

**Example**:
```lua
local target = ix.util.FindPlayer("John")
local target = ix.util.FindPlayer("STEAM_0:1:12345")
local target = ix.util.FindPlayer("1")
```

### ix.util.IsColor

**Realm**: Shared

```lua
local isColor = ix.util.IsColor(input)
```

Checks if input is a color table.

**Example**:
```lua
print(ix.util.IsColor(Color(255, 0, 0)))  -- true
print(ix.util.IsColor({r=255, g=0, b=0})) -- true
print(ix.util.IsColor("red"))             -- false
```

### ix.util.GetTypeFromValue

**Realm**: Shared

```lua
local type = ix.util.GetTypeFromValue(value)
```

Returns ix.type of value.

**Example**:
```lua
print(ix.util.GetTypeFromValue("text"))    -- ix.type.string
print(ix.util.GetTypeFromValue(123))       -- ix.type.number
print(ix.util.GetTypeFromValue(true))      -- ix.type.bool
```

### ix.util.SanitizeType

**Realm**: Shared

```lua
local sanitized = ix.util.SanitizeType(type, input)
```

Converts input to specified type.

**Example**:
```lua
print(ix.util.SanitizeType(ix.type.number, "123"))  -- 123
print(ix.util.SanitizeType(ix.type.bool, 1))        -- true
print(ix.util.SanitizeType(ix.type.string, 123))    -- "123"
```

## See Also

- [Command API](command.md) - Uses ix.type for arguments
- [Data API](data.md) - File operations
- Source: `gamemode/core/sh_util.lua`
