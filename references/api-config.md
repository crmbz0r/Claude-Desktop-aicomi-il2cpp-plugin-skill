# BepInEx Configuration API — Reference

## ConfigFile

```csharp
public class ConfigFile : IDictionary<ConfigDefinition, ConfigEntryBase>
{
    string ConfigFilePath { get; }
    bool SaveOnConfigSet { get; set; }       // default: true
    bool GenerateSettingDescriptions { get; set; } // Comments in .cfg

    // Create new setting or load existing one
    ConfigEntry<T> Bind<T>(string section, string key, T defaultValue, string description)
    ConfigEntry<T> Bind<T>(string section, string key, T defaultValue, ConfigDescription desc)
    ConfigEntry<T> Bind<T>(ConfigDefinition definition, T defaultValue, ConfigDescription desc)

    // Search for setting (null if not found)
    bool TryGetEntry<T>(string section, string key, out ConfigEntry<T> entry)

    void Save()    // Manually write to disk
    void Reload()  // Reload from disk (discards unsaved changes)

    event EventHandler ConfigReloaded
    event EventHandler<SettingChangedEventArgs> SettingChanged
}
```

## ConfigEntry\<T\>

```csharp
public sealed class ConfigEntry<T> : ConfigEntryBase
{
    T Value { get; set; }        // Read/set value
    object BoxedValue { get; set; }
    object DefaultValue { get; }

    ConfigFile ConfigFile { get; }
    ConfigDefinition Definition { get; }  // Section + Key
    ConfigDescription Description { get; }
    Type SettingType { get; }

    string GetSerializedValue()
    void SetSerializedValue(string value)

    event EventHandler SettingChanged
}
```

## ConfigDefinition

```csharp
// Section = group, Key = setting name
public class ConfigDefinition : IEquatable<ConfigDefinition>
{
    string Section { get; }
    string Key { get; }

    ConfigDefinition(string section, string key)
}
```

## ConfigDescription

```csharp
public class ConfigDescription
{
    static ConfigDescription Empty { get; }

    string Description { get; }
    AcceptableValueBase AcceptableValues { get; }
    object[] Tags { get; }

    ConfigDescription(string description,
        AcceptableValueBase acceptableValues = null,
        params object[] tags)
}
```

## Value Restrictions

```csharp
// Value range (for numeric types)
new AcceptableValueRange<float>(0f, 100f)
new AcceptableValueRange<int>(1, 50)

// Value list (for Enums, Strings, etc.)
new AcceptableValueList<string>("Option1", "Option2", "Option3")
new AcceptableValueList<KeyCode>(KeyCode.F1, KeyCode.F2, KeyCode.F3)

// Combination
Config.Bind("Section", "Volume", 80,
    new ConfigDescription("Volume",
        new AcceptableValueRange<int>(0, 100)));
```

## KeyboardShortcut

```csharp
public struct KeyboardShortcut
{
    static KeyboardShortcut Empty { get; }
    static IEnumerable<KeyCode> AllKeyCodes { get; }

    KeyCode MainKey { get; }
    IEnumerable<KeyCode> Modifiers { get; }

    KeyboardShortcut(KeyCode mainKey, params KeyCode[] modifiers)

    bool IsDown()    // Currently pressed (GetKeyDown)
    bool IsPressed() // Held down (GetKey)
    bool IsUp()      // Released (GetKeyUp)

    string Serialize()
    static KeyboardShortcut Deserialize(string str)
}
```

**Examples:**
```csharp
// Simple key
var hotkey = Config.Bind("Hotkeys", "Toggle",
    new KeyboardShortcut(KeyCode.F5), "Toggle");

// Key combination
var combo = Config.Bind("Hotkeys", "Save",
    new KeyboardShortcut(KeyCode.S, KeyCode.LeftControl), "Save");

// Usage in Update()
void Update()
{
    if (hotkey.Value.IsDown()) Toggle();
    if (combo.Value.IsDown()) Save();
}
```

## Supported Types (automatically serialized)

BepInEx can directly use the following types as config values:
- `bool`, `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`
- `float`, `double`, `decimal`
- `string`, `char`
- `Enum` (all enum types)
- `Color`, `Vector2`, `Vector3`, `Vector4`, `Quaternion` (Unity types)
- `KeyCode`, `KeyboardShortcut`

For custom types: use `TomlTypeConverter.AddConverter(Type, TypeConverter)`.

## Practical Patterns

```csharp
// Organize config in separate class
public static class PluginConfig
{
    public static ConfigEntry<bool>   Enabled;
    public static ConfigEntry<float>  Multiplier;

    public static void Init(ConfigFile config)
    {
        Enabled = config.Bind("General", "Enabled", true, "Enable the mod");
        Multiplier = config.Bind("Values", "Multiplier", 1.0f,
            new ConfigDescription("Multiplier", new AcceptableValueRange<float>(0.1f, 5f)));
    }
}

// In Plugin.Load():
PluginConfig.Init(Config);

// Throughout code:
if (PluginConfig.Enabled.Value) { ... }
float val = PluginConfig.Multiplier.Value;