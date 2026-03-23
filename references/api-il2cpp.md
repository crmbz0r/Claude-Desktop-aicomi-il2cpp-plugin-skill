# BepInEx IL2CPP API — Reference

## BasePlugin (BepInEx.IL2CPP)

Base class for **all** AiComi plugins. Assembly: `BepInEx.IL2CPP.dll`

```csharp
public abstract class BasePlugin
{
    public ConfigFile Config { get; }
    public ManualLogSource Log { get; }

    public abstract void Load();
    public virtual bool Unload() { return false; }

    // Add Unity component to global manager
    public T AddComponent<T>() where T : Il2CppObjectBase
}
```

## IL2CPPChainloader (BepInEx.IL2CPP)

```csharp
public class IL2CPPChainloader : BaseChainloader<BasePlugin>
{
    public static IL2CPPChainloader Instance { get; set; }

    // Add component globally (preferred for manager objects)
    public static Il2CppObjectBase AddUnityComponent(Type t)
    public static T AddUnityComponent<T>() where T : Il2CppObjectBase
}
```

**Access loaded plugins:**
```csharp
var plugins = IL2CPPChainloader.Instance.Plugins;
// Dictionary<string, PluginInfo> — Key = GUID
if (plugins.TryGetValue("com.other.plugin", out var info))
    Debug.Log(info.Metadata.Version);
```

## DetourGenerator (BepInEx.IL2CPP)

For native detours on IL2CPP methods (low-level):

```csharp
// Apply detour to native method
DetourGenerator.ApplyDetour(
    functionPtr,   // IntPtr to original function
    detourPtr,     // IntPtr to replacement function
    Architecture.X64
);

// Create trampoline (call original after detour)
IntPtr trampoline = DetourGenerator.CreateTrampolineFromFunction(
    originalPtr, out int trampolineLength, out int jmpLength);
```

## FastNativeDetour (BepInEx.IL2CPP.Hook)

Higher-level wrapper for native detours:

```csharp
// Recommended for native/icall methods that Harmony cannot patch
delegate void OriginalDelegate(IntPtr instance);
static OriginalDelegate _originalMethod;

var detour = FastNativeDetour.CreateAndApply(
    nativeFunctionPtr,
    new OriginalDelegate(MyReplacement),
    out _originalMethod
);

static void MyReplacement(IntPtr instance)
{
    // Custom logic
    _originalMethod(instance); // Call original
}
```

## MonoBehaviourExtensions (BepInEx.IL2CPP.Utils)

```csharp
// Extension method for IL2CPP coroutines
using BepInEx.IL2CPP.Utils;

monoBehaviour.StartCoroutine(MyCoroutine());
// Internally WrapToIl2Cpp is automatically applied
```

## CollectionExtensions (BepInEx.IL2CPP.Utils.Collections)

Conversion between Managed and IL2CPP Collections:

```csharp
using BepInEx.IL2CPP.Utils.Collections;

// Managed → IL2CPP (for Unity APIs expecting Il2Cpp enumerables)
IEnumerator il2cppEnum = managedEnumerator.WrapToIl2Cpp();
IEnumerable il2cppEnumerable = managedEnumerable.WrapToIl2Cpp();

// IL2CPP → Managed
IEnumerator managedEnum = il2cppEnumerator.WrapToManaged();
```

## Preloader (BepInEx.IL2CPP)

```csharp
// Paths known by the preloader
string unhollowedPath = Preloader.IL2CPPUnhollowedPath;
Version unityVersion  = Preloader.UnityVersion;