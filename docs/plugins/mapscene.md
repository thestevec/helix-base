# Map Scenes Plugin

> **Reference**: `plugins/mapscene.lua`

The Map Scenes plugin creates cinematic camera views during character selection. Admins can define static or moving camera paths that showcase different areas of the map, providing visual interest during character loading.

## ⚠️ Important: Use Built-in Scene System

**Always use MapSceneAdd/MapSceneRemove commands**. The framework provides:
- Static camera positions
- Moving camera paths (start to end)
- Automatic transitions between scenes
- Mouse-based parallax effect
- Persistent storage

## Using Map Scenes

### Adding Static Scene

```lua
/MapSceneAdd
```

Creates scene at your current eye position and angles.

### Adding Moving Scene

```lua
/MapSceneAdd 1
```

1. Use command at first position
2. Move to second position
3. Use command again
4. Scene transitions between both points over 30 seconds

### Removing Scenes

```lua
/MapSceneRemove [radius]
```

Removes scenes within radius (default: 280 units).

## Best Practices

### ✅ DO

- Create multiple scenes for variety
- Use cinematic angles
- Show interesting map areas
- Test scene transitions

### ❌ DON'T

- Don't create too many scenes (5-10 is good)
- Don't use boring angles
- Don't place scenes inside walls

## See Also

- [Character System](../systems/characters.md) - Character selection
- Source: `plugins/mapscene.lua`
