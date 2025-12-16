# cc_win_plugin Debug Instructions

## Problem

The plugin crashes when `tools/list` is called. The log shows only "Plugin started" — nothing after.

The test script (`test_mcp.ps1`) sends:
1. initialize → works
2. initialized notification → works  
3. tools/list → process crashes

## Task

Add diagnostic logging to find where the crash occurs. Make these changes incrementally, rebuilding and testing after each step.

---

## Step 1: Wrap the entire startup in try-catch

Edit `Program.cs`. Wrap everything after logger creation in a try-catch that logs exceptions.

**Before:**
```csharp
logger.Info("Plugin started");

var builder = Host.CreateEmptyApplicationBuilder(settings: null);

builder.Services.AddSingleton(projectContext);
// ... rest of code
```

**After:**
```csharp
logger.Info("Plugin started");

try
{
    logger.Info("Creating host builder");
    var builder = Host.CreateEmptyApplicationBuilder(settings: null);

    logger.Info("Adding singletons");
    builder.Services.AddSingleton(projectContext);
    builder.Services.AddSingleton(processTracker);
    builder.Services.AddSingleton(logger);

    logger.Info("Adding MCP server");
    builder.Services
        .AddMcpServer()
        .WithStdioServerTransport()
        .WithToolsFromAssembly();

    logger.Info("Building host");
    var host = builder.Build();

    logger.Info("Running host");
    await host.RunAsync();
}
catch (Exception ex)
{
    logger.Error($"Fatal error: {ex.GetType().Name}: {ex.Message}");
    logger.Error($"Stack trace: {ex.StackTrace}");
    throw;
}
```

**Test:**
1. Rebuild: `dotnet publish src/CcWin.csproj -r win-x64`
2. Copy: `copy out\bin\Debug\net10.0\win-x64\publish\CcWin.exe .`
3. Run: `powershell -ExecutionPolicy Bypass -File test_mcp.ps1`
4. Check `logs/cc_win_plugin.log`

**Report:** What's the last log entry? Did it catch an exception?

---

## Step 2: If Step 1 shows no exception

The crash might be in the MCP SDK's tool discovery. Add logging before and after `WithToolsFromAssembly()`.

Also test if `WithToolsFromAssembly()` is the problem by temporarily removing it:

**Test without tools:**
```csharp
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport();
    // .WithToolsFromAssembly();  // Comment out
```

Rebuild and test. If it works without tools, the problem is in tool registration.

---

## Step 3: If tool registration is the problem

The SDK uses reflection to find tools. One of the tool classes might have an error.

Create a minimal test tool to isolate the issue:

**Create `Tools/TestTool.cs`:**
```csharp
using System.ComponentModel;
using ModelContextProtocol.Server;

namespace CcWin.Tools;

[McpServerToolType]
public static class TestTool
{
    [McpServerTool]
    [Description("Test tool that returns hello")]
    public static string Test()
    {
        return "hello";
    }
}
```

Then comment out ALL other tools (ListFilesTool, RunProcessTool, ReadFileTool, CloseWindowTool) and test with just TestTool.

If that works, re-enable tools one by one to find which one crashes.

---

## Step 4: Check tool method signatures

The MCP SDK is picky about tool method signatures. Common issues:

1. **Return type:** Must be a type the SDK can serialize
2. **Parameters:** Complex parameters might not serialize correctly
3. **Dependency injection:** Services must be registered correctly

Check each tool:
- Does it inject `ProjectContext`, `ProcessTracker`, `AppLogger`?
- Are all these registered before `AddMcpServer()`?

---

## Step 5: Check for Console.WriteLine

**IMPORTANT:** MCP uses stdout for JSON-RPC. Any `Console.WriteLine` will corrupt the protocol.

Search all `.cs` files for:
- `Console.WriteLine`
- `Console.Write`
- `Console.Out`

Remove any found. All output must go through the logger.

---

## Reporting

After each step, report:
1. Last line in `logs/cc_win_plugin.log`
2. Any error messages
3. Did test_mcp.ps1 complete or crash?
