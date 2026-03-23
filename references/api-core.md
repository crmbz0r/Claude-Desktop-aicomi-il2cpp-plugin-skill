# BepInEx Core API — Reference

## BepInPlugin Attribute

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class BepInPlugin : Attribute
{
    public string GUID { get; }    // Unique ID, never changes
    public string Name { get; }    // Display name
    public SemanticVersioning.Version Version { get; }
}
```

## BepInDependency Attribute

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class BepInDependency : Attribute
{
    // Flags
    public enum DependencyFlags
    {
        HardDependency, // Plugin won't load without this dependency
        SoftDependency  // Optional, plugin still runs
    }

    public string DependencyGUID { get; }
    public DependencyFlags Flags { get; }
    public SemanticVersioning.Range VersionRange { get; }
}
```

## BepInIncompatibility Attribute

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class BepInIncompatibility : Attribute
{
    public string IncompatibilityGUID { get; }
}
```

## BepInProcess Attribute

```csharp
// Load plugin only for specific process
[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class BepInProcess : Attribute
{
    public string ProcessName { get; }
}
// Example: [BepInProcess("AiComi.exe")]
```

## Paths — Static Paths

```csharp
public static class Paths
{
    string BepInExAssemblyDirectory  // Folder containing BepInEx core DLLs
    string BepInExAssemblyPath       // Path to BepInEx.Core.dll
    string BepInExConfigPath         // BepInEx/config/BepInEx.cfg
    string BepInExRootPath           // BepInEx/ main folder
    string CachePath                 // BepInEx/cache/
    string ConfigPath                // BepInEx/config/
    string ExecutablePath            // Path to game executable
    string GameRootPath              // Game folder (parent of executable)
    string ManagedPath               // Managed/ Unity assemblies
    string PatcherPluginPath         // BepInEx/patchers/
    string PluginPath                // BepInEx/plugins/
    string ProcessName               // Name of current process
    string[] DllSearchPaths          // Mono DLL search paths
    Version BepInExVersion           // BepInEx version
}
```

## ThreadingHelper

```csharp
public sealed class ThreadingHelper : MonoBehaviour, ISynchronizeInvoke
{
    public static ThreadingHelper Instance { get; }
    public static ISynchronizeInvoke SynchronizingObject { get; }
    public bool InvokeRequired { get; } // true if not main thread

    // Execute code on background thread
    // action can optionally return an Action to run on main thread
    void StartAsyncInvoke(Func<Action> action)

    // Queue code on main thread
    void StartSyncInvoke(Action action)
}
```

**Usage example:**
```csharp
ThreadingHelper.Instance.StartAsyncInvoke(() =>
{
    var result = DoHeavyWork();
    return () => ApplyResultOnMainThread(result); // Main thread callback
});
```

## MetadataHelper

```csharp
public static class MetadataHelper
{
    // Read plugin metadata from instance/type
    BepInPlugin GetMetadata(object plugin)
    BepInPlugin GetMetadata(Type pluginType)

    // Read all attributes of a type/assembly
    IEnumerable<T> GetAttributes<T>(object plugin) where T : Attribute
    T[] GetAttributes<T>(Type pluginType) where T : Attribute
    T[] GetAttributes<T>(Assembly assembly) where T : Attribute

    // Determine dependencies of a plugin type
    IEnumerable<BepInDependency> GetDependencies(Type plugin)
}
```

## Utility — Helper Methods

```csharp
public static class Utility
{
    bool CLRSupportsDynamicAssemblies { get; }
    Encoding UTF8NoBom { get; }

    string CombinePaths(params string[] parts)
    string ParentDirectory(string path, int levels = 1)
    string ConvertToWWWFormat(string path)  // file:// URL
    string ByteArrayToString(byte[] data)   // Hex string
    string HashStream(Stream stream)        // MD5 hash
    bool IsNullOrWhiteSpace(this string self)
    bool SafeParseBool(string input, bool defaultValue = false)
    bool TryDo(Action action, out Exception exception)
    bool TryOpenFileStream(string path, FileMode mode, out FileStream fs, ...)
    string GetCommandLineArgValue(string arg) // Parse CLI arguments

    // Topological sort (used internally for dependency resolution)
    IEnumerable<TNode> TopologicalSort<TNode>(IEnumerable<TNode> nodes,
        Func<TNode, IEnumerable<TNode>> dependencySelector)
}
```

## PluginInfo

```csharp
public class PluginInfo : ICacheable
{
    BepInPlugin Metadata { get; }                       // GUID, Name, Version
    IEnumerable<BepInDependency> Dependencies { get; }
    IEnumerable<BepInIncompatibility> Incompatibilities { get; }
    IEnumerable<BepInProcess> Processes { get; }
    string Location { get; }  // Path to plugin DLL
    object Instance { get; }  // Plugin instance (null until loaded)
    string TypeName { get; }
}
```

## BaseChainloader — List Loaded Plugins

```csharp
// All loaded plugins
Dictionary<string, PluginInfo> plugins = IL2CPPChainloader.Instance.Plugins;

// Plugin info by GUID
if (plugins.TryGetValue("com.example.plugin", out PluginInfo info))
{
    Plugin.Log.LogInfo($"Version: {info.Metadata.Version}");
    Plugin.Log.LogInfo($"Path: {info.Location}");
}

// Dependency errors that occurred during loading
List<string> errors = IL2CPPChainloader.Instance.DependencyErrors;