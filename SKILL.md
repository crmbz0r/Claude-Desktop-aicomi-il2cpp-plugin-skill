---
name: aicomi-il2cpp-plugin
description: >
  How to create, code, and structure BepInEx IL2CPP mods and plugins for AiComi (Unity 2022 IL2CPP).
  Use this skill whenever the user wants to write, edit, or debug a BepInEx plugin or mod for AiComi,
  mentions IL2CPP plugin development, asks about BasePlugin, BepInPlugin attributes, Harmony patches,
  config bindings, coroutines, or Unity component injection in the context of AiComi modding.
  Always use this skill for any AiComi / Unity IL2CPP mod coding task, even if phrased casually
  ("just a quick plugin", "how do I hook X", "add a config option").
---

# AiComi IL2CPP Plugin Development Skill

This skill covers the complete development of BepInEx plugins for **AiComi** (Unity 2022, IL2CPP backend).

> ⚠️ **Critical:** AiComi uses IL2CPP — plugins inherit from `BasePlugin` (namespace `BepInEx.IL2CPP`), **not** from `BaseUnityPlugin`. This is the most common beginner mistake.

---

## 1. Required Imports & Namespaces

```csharp
using BepInEx;
using BepInEx.IL2CPP;           // BasePlugin, IL2CPPChainloader
using BepInEx.Configuration;    // ConfigFile, ConfigEntry<T>
using BepInEx.Logging;          // ManualLogSource, LogLevel
using HarmonyLib;               // Harmony Patching (if used)
using UnityEngine;              // MonoBehaviour, GameObject, etc.
```

---

## 2. Minimal Plugin Skeleton

```csharp
[BepInPlugin(MyPluginInfo.PLUGIN_GUID, MyPluginInfo.PLUGIN_NAME, MyPluginInfo.PLUGIN_VERSION)]
public class Plugin : BasePlugin
{
    public static ManualLogSource Log;

    public override void Load()
    {
        Log = base.Log;
        Log.LogInfo($"Plugin {MyPluginInfo.PLUGIN_GUID} loaded!");

        // Register Harmony patches
        var harmony = new Harmony(MyPluginInfo.PLUGIN_GUID);
        harmony.PatchAll();
    }

    public override bool Unload()
    {
        // Cleanup if needed
        return true;
    }
}

internal static class MyPluginInfo
{
    public const string PLUGIN_GUID    = "com.yourname.aicomi.myplugin";
    public const string PLUGIN_NAME    = "My Plugin";
    public const string PLUGIN_VERSION = "1.0.0";
}
```

**Rules:**

- `GUID` = unique, reverse domain notation, never changes
- `Load()` = entry point (equivalent to `Awake` in MonoBehaviour)
- `base.Log` = the pre-configured `ManualLogSource` of the plugin
- `Unload()` returns `bool` — `true` = successfully unloaded

---

## 3. Plugin Attributes

### `[BepInPlugin]` — Required

```csharp
[BepInPlugin("com.author.game.plugin", "Plugin Name", "1.0.0")]
```

### `[BepInDependency]` — Dependencies

```csharp
// Hard dependency (plugin won't load without this)
[BepInDependency("com.other.plugin")]

// Soft dependency (optional, plugin still runs)
[BepInDependency("com.other.plugin", BepInDependency.DependencyFlags.SoftDependency)]

// With version range
[BepInDependency("com.other.plugin", "2.0.0")]
```

### `[BepInIncompatibility]` — Conflicts

```csharp
[BepInIncompatibility("com.conflicting.plugin")]
```

### `[BepInProcess]` — Process restriction

```csharp
[BepInProcess("AiComi.exe")]  // only load for this process
```

---

## 4. Configuration (ConfigFile)

```csharp
public class Plugin : BasePlugin
{
    // Static config entries for access from patch classes
    public static ConfigEntry<bool>   CfgEnabled;
    public static ConfigEntry<float>  CfgSpeed;
    public static ConfigEntry<string> CfgText;
    public static ConfigEntry<int>    CfgCount;

    public override void Load()
    {
        // Bind<T>(section, key, defaultValue, description)
        CfgEnabled = Config.Bind("General", "Enabled", true,
            "Enable/disable plugin");

        CfgSpeed = Config.Bind("Gameplay", "Speed", 1.5f,
            new ConfigDescription("Speed multiplier",
                new AcceptableValueRange<float>(0.1f, 10f)));

        CfgCount = Config.Bind("Gameplay", "Count", 5,
            new ConfigDescription("Count",
                new AcceptableValueRange<int>(1, 100)));

        CfgText = Config.Bind("UI", "Label", "Default",
            "Display text");
    }
}
```

**Access in code:**

```csharp
if (Plugin.CfgEnabled.Value) { ... }
float mult = Plugin.CfgSpeed.Value;

// React to changes
Plugin.CfgEnabled.SettingChanged += (sender, args) =>
    Plugin.Log.LogInfo("Enabled changed to: " + Plugin.CfgEnabled.Value);
```

**Keyboard shortcut as config:**

```csharp
var shortcut = Config.Bind("Hotkeys", "OpenMenu",
    new KeyboardShortcut(KeyCode.F10, KeyCode.LeftControl),
    "Open menu");

// In Update loop:
if (shortcut.Value.IsDown()) { OpenMenu(); }
```

---

## 5. Logging

```csharp
// From Load() — use base.Log
Log.LogInfo("Info message");
Log.LogDebug("Debug message");          // only visible in Debug builds
Log.LogWarning("Warning");
Log.LogError("Error!");
Log.LogFatal("Critical error");
Log.LogMessage("Important user info"); // always visible

// Formatting with interpolation (efficient — skipped if level disabled)
Log.LogInfo($"Player HP: {player.health}, Position: {player.transform.position}");
```

---

## 6. Harmony Patching (Method Hooks)

```csharp
[HarmonyPatch(typeof(TargetClass), nameof(TargetClass.TargetMethod))]
public static class TargetMethod_Patch
{
    // Before original method — return false skips original
    [HarmonyPrefix]
    public static bool Prefix(TargetClass __instance, ref float __result, float arg1)
    {
        Plugin.Log.LogDebug($"Prefix: arg1={arg1}");
        // Set __result and return false = skip original
        return true; // true = continue with original
    }

    // After original method
    [HarmonyPostfix]
    public static void Postfix(TargetClass __instance, ref float __result)
    {
        __result *= Plugin.CfgSpeed.Value;
    }
}
```

**Parameter reference:**

| Parameter      | Meaning                             |
| -------------- | ----------------------------------- |
| `__instance`   | `this` of the patched class         |
| `__result`     | Return value (ref for modification) |
| `___fieldName` | Private field (3 underscores)       |
| Argument name  | Original argument (exact name)      |

---

## 7. Unity Components Injection in IL2CPP

```csharp
// In Load() — add MonoBehaviour to scene
public override void Load()
{
    // Via IL2CPPChainloader (global manager)
    IL2CPPChainloader.AddUnityComponent<MyBehaviour>();

    // Or via BasePlugin method
    AddComponent<MyBehaviour>();
}

public class MyBehaviour : MonoBehaviour
{
    private void Awake()
    {
        Plugin.Log.LogInfo("MyBehaviour started");
    }

    private void Update()
    {
        // Check hotkey etc.
        if (Plugin.CfgEnabled.Value && Input.GetKeyDown(KeyCode.F5))
        {
            DoSomething();
        }
    }
}
```

---

## 8. Coroutines in IL2CPP

IL2CPP requires the `WrapToIl2Cpp()` wrapper:

```csharp
using BepInEx.IL2CPP.Utils;             // StartCoroutine Extension
using BepInEx.IL2CPP.Utils.Collections; // WrapToIl2Cpp

// In MonoBehaviour
this.StartCoroutine(MyCoroutine().WrapToIl2Cpp());

private IEnumerator MyCoroutine()
{
    yield return new WaitForSeconds(1f);
    Plugin.Log.LogInfo("1 second passed");
    yield return null;
}
```

---

## 9. Paths & Resources

```csharp
using BepInEx; // Paths

// Important paths
string pluginDir   = Paths.PluginPath;          // BepInEx/plugins/
string configDir   = Paths.ConfigPath;           // BepInEx/config/
string bepinexDir  = Paths.BepInExRootPath;      // BepInEx/
string gameRoot    = Paths.GameRootPath;          // Game directory
string managedDir  = Paths.ManagedPath;           // Managed/ folder
string processName = Paths.ProcessName;           // "AiComi"
```

---

## 10. Common Errors & Solutions

| Error                                | Cause                               | Solution                                              |
| ------------------------------------ | ----------------------------------- | ----------------------------------------------------- |
| Plugin doesn't load                  | Wrong base class                    | Use `BasePlugin` instead of `BaseUnityPlugin`         |
| `NullReferenceException` in `Load()` | IL2CPP types not ready yet          | Move initialization to `MonoBehaviour.Awake()`        |
| Harmony patch doesn't work           | Method is IL2CPP internal (icall)   | Use `IL2CPPDetourMethodPatcher` or `FastNativeDetour` |
| Coroutine doesn't run                | Missing wrapper                     | Apply `.WrapToIl2Cpp()` to the `IEnumerator`          |
| Config not saved                     | `SaveOnConfigSet` off & no `Save()` | Call `Config.Save()` or set `SaveOnConfigSet = true`  |

---

## 11. Reference Files

For more complex requests, read the appropriate reference files:

- **`references/api-il2cpp.md`** → `BasePlugin`, `IL2CPPChainloader`, `DetourGenerator`, Hooks
- **`references/api-core.md`** → `BepInPlugin`, `Paths`, `ThreadingHelper`, `Utility`
- **`references/api-config.md`** → `ConfigFile`, `ConfigEntry<T>`, `AcceptableValue*`, `KeyboardShortcut`
- **`references/api-logging.md`** → `ManualLogSource`, `LogLevel`, `ILogListener`

**When to use which file:**

- Standard plugin → `api-il2cpp.md` + `api-core.md`
- Plugin with settings → additionally `api-config.md`
- Custom logging → `api-logging.md`
- Preloader/Patcher → separate workflow, not standard plugin

---

## 12. Project Structure (Visual Studio / Rider)

```plaintext
MyPlugin/
+-- MyPlugin.csproj
+-- Plugin.cs           ← Main plugin class
+-- Patches/
¦   +-- SomeClass_Patch.cs
+-- Components/
¦   +-- MyBehaviour.cs
+-- Config.cs           ← Config entries (optional separate file)
```

**.csproj — References via NuGet:**

```xml
<ItemGroup>
  <!-- Contains all AiComi game assemblies + BepInEx IL2CPP + Harmony -->
  <PackageReference Include="IllusionLibs.Aicomi.AllPackages" Version="*-*" />
</ItemGroup>
```

> `Version="*"` = always the latest available version. Alternatively `Version="*-*"` for pre-releases too, or a fixed version like `Version="2024.10.3"` for reproducible builds.