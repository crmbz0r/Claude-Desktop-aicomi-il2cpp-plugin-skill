# BepInEx Logging API — Reference

## ManualLogSource (BepInEx.Logging)

The standard logger for plugins. Provided in `BasePlugin` as `base.Log`.

```csharp
public class ManualLogSource : ILogSource, IDisposable
{
    string SourceName { get; }

    // Standard logging (overloaded for Object and interpolated strings)
    void Log(LogLevel level, object data)
    void LogDebug(object data)     // Only visible when debug level is enabled
    void LogInfo(object data)
    void LogMessage(object data)   // Important messages, always visible
    void LogWarning(object data)
    void LogError(object data)
    void LogFatal(object data)

    // Create own log source (outside a plugin)
    // Better: use Logger.CreateLogSource(name)
}
```

## LogLevel Enum

```csharp
[Flags]
public enum LogLevel
{
    None    = 0,
    Fatal   = 1,   // Critical error, no recovery possible
    Error   = 2,   // Error, recovery possible
    Warning = 4,   // Warning
    Message = 8,   // Important user info (always displayed)
    Info    = 16,  // General info
    Debug   = 32,  // Developer details
    All     = 63   // All levels
}
```

## Logger (static class)

```csharp
public static class Logger
{
    ICollection<ILogListener> Listeners { get; }
    ICollection<ILogSource> Sources { get; }
    LogLevel ListenedLogLevels { get; }  // Which levels have active listeners

    // Register new log source
    ManualLogSource CreateLogSource(string sourceName)
}
```

**Create external log source:**
```csharp
// In a non-plugin class
var log = Logger.CreateLogSource("MySubsystem");
log.LogInfo("Initialized");
// When cleaning up:
log.Dispose();
```

## ILogListener — Custom Log Target

Only needed when logs should be forwarded to custom output (file, network, UI):

```csharp
public class MyLogListener : ILogListener
{
    public LogLevel LogLevelFilter => LogLevel.Warning | LogLevel.Error | LogLevel.Fatal;

    public void LogEvent(object sender, LogEventArgs eventArgs)
    {
        string message = eventArgs.Data?.ToString() ?? "";
        LogLevel level = eventArgs.Level;
        string source = eventArgs.Source.SourceName;

        // e.g., write to own file
        File.AppendAllText("my_log.txt", eventArgs.ToStringLine());
    }

    public void Dispose() { }
}

// Register in Load():
Logger.Listeners.Add(new MyLogListener());
```

## LogEventArgs

```csharp
public class LogEventArgs : EventArgs
{
    object Data { get; }       // Log data (usually string)
    LogLevel Level { get; }    // Log level
    ILogSource Source { get; } // Which source logged

    string ToString()          // "[Level : Source] Data"
    string ToStringLine()      // Like ToString() + newline
}
```

## DiskLogListener

Pre-configured by BepInEx. Use for custom disk logging if needed:

```csharp
var diskListener = new DiskLogListener(
    localPath: Path.Combine(Paths.BepInExRootPath, "MyPlugin.log"),
    displayedLogLevel: LogLevel.Info,
    appendLog: false,
    delayedFlushing: true,
    fileLimit: 3
);
Logger.Listeners.Add(diskListener);
```

## Interpolated String Logging (efficient)

BepInEx supports C# interpolated string handlers — string interpolation is skipped when the log level is not active:

```csharp
// Efficient — costs nothing when debug is disabled
log.LogDebug($"Player: {player.name}, HP: {player.health:F2}");

// Equivalent to (but shorter):
if ((Logger.ListenedLogLevels & LogLevel.Debug) != 0)
    log.LogDebug($"Player: {player.name}, HP: {player.health:F2}");