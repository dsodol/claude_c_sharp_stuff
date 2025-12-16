# cc_win_plugin — Specification

## Instructions for Claude Code

You are in the `cc_win_marketplace` folder. Create all files listed in this spec inside the current directory.

After creating all files:

1. `cd cc_win_plugin`
2. `dotnet publish src/CcWin.csproj -r win-x64`
3. `copy out\bin\Debug\net10.0\win-x64\publish\CcWin.exe .`
4. `powershell -ExecutionPolicy Bypass -File test_mcp.ps1`
5. Report build result and test output

---

## Overview

MCP server plugin for Claude Code on Windows. Provides native Windows tools so CC never needs shell commands.

**MVP Scope:** Four tools — list_files, run_process, read_file, close_window

---

## Tech Stack

- **Language:** C#
- **Framework:** .NET 10
- **MCP:** Official ModelContextProtocol SDK (NuGet)
- **Transport:** stdio

---

## Project Structure

```
./                                    (cc_win_marketplace - current directory)
├── .claude-plugin/
│   └── marketplace.json
└── cc_win_plugin/
    ├── src/
    │   ├── CcWin.csproj
    │   ├── Program.cs
    │   ├── ProjectContext.cs
    │   ├── ProcessTracker.cs
    │   ├── AppLogger.cs
    │   └── Tools/
    │       ├── ListFilesTool.cs
    │       ├── RunProcessTool.cs
    │       ├── ReadFileTool.cs
    │       └── CloseWindowTool.cs
    ├── skills/
    │   └── windows-native/
    │       └── SKILL.md
    ├── out/                          (generated)
    │   ├── bin/
    │   └── obj/
    ├── logs/                         (runtime)
    ├── .claude-plugin/
    │   └── plugin.json
    ├── CcWin.exe                     (copied after build)
    ├── .gitignore
    ├── .mcp.json
    ├── test_mcp.ps1
    └── README.md
```

### .gitignore

```
out/
logs/
*.exe
*.dll
*.pdb
*.user
.vs/
```

---

## Security Constraints

### Path Sandboxing

All file operations must be within the project directory.

**Project directory** = directory where the exe is located (AppDomain.CurrentDomain.BaseDirectory).

Validation rules:
1. Resolve input path to absolute path
2. Normalize (remove `.`, resolve `..`)
3. Check if result starts with project directory
4. If not → return error, do not execute

See `ProjectContext.cs` implementation below.

### Process Tracking

Only processes started by `run_process` can be closed by `close_window`.

See `ProcessTracker.cs` implementation below.

---

## Logging

**Location:** `logs/cc_win_plugin.log` (relative to project directory)

**Rotation:** On startup, if log file exists, rename to `cc_win_plugin_{timestamp}.log`

**Cleanup:** Keep only 3 log files. Oldest deleted first.

**Format:**
```
[2025-01-15 10:30:00.123] [INFO] Message
[2025-01-15 10:30:00.456] [ERROR] Message
```

**What gets logged:**
- Plugin startup
- Every tool call (name, inputs)
- Every result (success or error)
- Process starts (PID, command)
- Process closes (PID)
- Path validation failures

---

## Tools Specification

### 1. list_files

List files and directories at a given path.

**Input:**
```json
{
  "path": "src"
}
```

- `path` (string, required): Relative path from project directory. Use `.` for project root.

**Output:**
```json
{
  "files": [
    {
      "name": "Program.cs",
      "type": "file",
      "size": 1234,
      "modified": "2025-01-15T10:30:00Z"
    },
    {
      "name": "Tools",
      "type": "directory",
      "size": 0,
      "modified": "2025-01-15T09:00:00Z"
    }
  ]
}
```

**Errors:**
- Path outside project directory → `{ "error": "Path not allowed: {path}" }`
- Path does not exist → `{ "error": "Path not found: {path}" }`

**Notes:**
- Single level only (not recursive)
- Sort: directories first, then files, alphabetically
- `size` for directories is always 0
- `modified` is ISO 8601 UTC

---

### 2. run_process

Start a process. Returns immediately with PID.

**Input:**
```json
{
  "command": "dotnet",
  "args": ["build", "-c", "Release"]
}
```

- `command` (string, required): Executable name or path
- `args` (array of strings, optional): Command line arguments

**Output:**
```json
{
  "pid": 12345
}
```

**Errors:**
- Command not found → `{ "error": "Command not found: {command}" }`
- Failed to start → `{ "error": "Failed to start process: {details}" }`

**Critical:** Working directory is ALWAYS project directory, regardless of where the executable is.

---

### 3. read_file

Read contents of a file.

**Input:**
```json
{
  "path": "logs/app.log"
}
```

- `path` (string, required): Relative path from project directory

**Output:**
```json
{
  "content": "2025-01-15T10:30:00Z Button clicked\n"
}
```

**Errors:**
- Path outside project directory → `{ "error": "Path not allowed: {path}" }`
- File not found → `{ "error": "File not found: {path}" }`
- File too large (>10MB) → `{ "error": "File too large: {size} bytes" }`

**Notes:**
- Read as UTF-8
- Max file size: 10MB
- Can read files that are open by another process

---

### 4. close_window

Gracefully close a window by sending WM_CLOSE message.

**Input:**
```json
{
  "pid": 12345
}
```

- `pid` (integer, required): Process ID returned by run_process

**Output:**
```json
{
  "success": true
}
```

**Errors:**
- PID not tracked → `{ "error": "Process not tracked: {pid}" }`
- Process not found → `{ "error": "Process not found: {pid}" }`
- No main window → `{ "error": "Process has no main window: {pid}" }`

---

## MCP Server Implementation

### CcWin.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <BaseOutputPath>..\out\bin\</BaseOutputPath>
    <BaseIntermediateOutputPath>..\out\obj\</BaseIntermediateOutputPath>
    <PublishSingleFile>true</PublishSingleFile>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="ModelContextProtocol" Version="0.5.0-preview.1" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
  </ItemGroup>
</Project>
```

### Program.cs

```csharp
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;
using CcWin;

// Get the directory where the exe is located (not current working directory)
var projectDirectory = AppDomain.CurrentDomain.BaseDirectory.TrimEnd(Path.DirectorySeparatorChar);
var logger = new AppLogger(projectDirectory);
var processTracker = new ProcessTracker();
var projectContext = new ProjectContext(projectDirectory);

logger.Info("Plugin started");
logger.Info($"Project directory: {projectDirectory}");

var builder = Host.CreateEmptyApplicationBuilder(settings: null);

builder.Services.AddSingleton(projectContext);
builder.Services.AddSingleton(processTracker);
builder.Services.AddSingleton(logger);
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

await builder.Build().RunAsync();
```

### AppLogger.cs

```csharp
namespace CcWin;

public class AppLogger
{
    private readonly string _logPath;
    private readonly string _logsDir;
    private readonly object _lock = new();
    private const int MaxLogFiles = 3;

    public AppLogger(string projectDirectory)
    {
        _logsDir = Path.Combine(projectDirectory, "logs");
        Directory.CreateDirectory(_logsDir);

        _logPath = Path.Combine(_logsDir, "cc_win_plugin.log");

        // Rotate existing log
        if (File.Exists(_logPath))
        {
            var timestamp = DateTime.Now.ToString("yyyyMMdd_HHmmss");
            var rotatedPath = Path.Combine(_logsDir, $"cc_win_plugin_{timestamp}.log");
            File.Move(_logPath, rotatedPath);
        }

        // Keep only last 3 log files
        CleanupOldLogs();
    }

    private void CleanupOldLogs()
    {
        var logFiles = Directory.GetFiles(_logsDir, "cc_win_plugin*.log")
            .Select(f => new FileInfo(f))
            .OrderByDescending(f => f.LastWriteTime)
            .Skip(MaxLogFiles)
            .ToList();

        foreach (var file in logFiles)
        {
            try { file.Delete(); } catch { }
        }
    }

    public void Info(string message) => Log("INFO", message);
    public void Error(string message) => Log("ERROR", message);

    public void ToolCall(string toolName, object? input)
    {
        var inputStr = input != null ? System.Text.Json.JsonSerializer.Serialize(input) : "null";
        Log("INFO", $"Tool call: {toolName} | Input: {inputStr}");
    }

    public void ToolResult(string toolName, bool success, string? error = null)
    {
        if (success)
            Log("INFO", $"Tool result: {toolName} | Success");
        else
            Log("ERROR", $"Tool result: {toolName} | Error: {error}");
    }

    private void Log(string level, string message)
    {
        var timestamp = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss.fff");
        var line = $"[{timestamp}] [{level}] {message}";

        lock (_lock)
        {
            File.AppendAllText(_logPath, line + Environment.NewLine);
        }
    }
}
```

### Tools Implementation

Tools are defined as static methods with attributes. The SDK auto-discovers them.

```csharp
// Tools/ListFilesTool.cs
using System.ComponentModel;
using ModelContextProtocol.Server;

namespace CcWin.Tools;

[McpServerToolType]
public static class ListFilesTool
{
    [McpServerTool]
    [Description("List files and directories at a path. Use this instead of 'dir', 'ls', or any shell command. Returns name, type (file/directory), size, and modification time.")]
    public static ListFilesResult ListFiles(
        ProjectContext context,
        AppLogger logger,
        [Description("Relative path from project directory. Use '.' for project root.")] string path)
    {
        logger.ToolCall("list_files", new { path });

        if (!context.IsPathAllowed(path, out var absolutePath))
        {
            logger.ToolResult("list_files", false, $"Path not allowed: {path}");
            return new ListFilesResult { Error = $"Path not allowed: {path}" };
        }

        if (!Directory.Exists(absolutePath))
        {
            logger.ToolResult("list_files", false, $"Path not found: {path}");
            return new ListFilesResult { Error = $"Path not found: {path}" };
        }

        var entries = new List<FileEntry>();

        foreach (var dir in Directory.GetDirectories(absolutePath).OrderBy(d => d))
        {
            var info = new DirectoryInfo(dir);
            entries.Add(new FileEntry
            {
                Name = info.Name,
                Type = "directory",
                Size = 0,
                Modified = info.LastWriteTimeUtc.ToString("o")
            });
        }

        foreach (var file in Directory.GetFiles(absolutePath).OrderBy(f => f))
        {
            var info = new FileInfo(file);
            entries.Add(new FileEntry
            {
                Name = info.Name,
                Type = "file",
                Size = info.Length,
                Modified = info.LastWriteTimeUtc.ToString("o")
            });
        }

        logger.ToolResult("list_files", true);
        return new ListFilesResult { Files = entries };
    }
}

public class ListFilesResult
{
    public List<FileEntry>? Files { get; set; }
    public string? Error { get; set; }
}

public class FileEntry
{
    public string Name { get; set; } = "";
    public string Type { get; set; } = "";
    public long Size { get; set; }
    public string Modified { get; set; } = "";
}
```

```csharp
// Tools/RunProcessTool.cs
using System.ComponentModel;
using System.Diagnostics;
using ModelContextProtocol.Server;

namespace CcWin.Tools;

[McpServerToolType]
public static class RunProcessTool
{
    [McpServerTool]
    [Description("Start a process and return its PID. Use this instead of shell commands. Working directory is always the project root.")]
    public static RunProcessResult RunProcess(
        ProjectContext context,
        ProcessTracker tracker,
        AppLogger logger,
        [Description("Executable name or path")] string command,
        [Description("Command line arguments")] string[]? args = null)
    {
        logger.ToolCall("run_process", new { command, args });

        try
        {
            var startInfo = new ProcessStartInfo
            {
                FileName = command,
                WorkingDirectory = context.ProjectDirectory,
                UseShellExecute = false,
                CreateNoWindow = false
            };

            if (args != null)
            {
                foreach (var arg in args)
                    startInfo.ArgumentList.Add(arg);
            }

            var process = Process.Start(startInfo);
            if (process == null)
            {
                logger.ToolResult("run_process", false, $"Failed to start: {command}");
                return new RunProcessResult { Error = $"Failed to start process: {command}" };
            }

            tracker.Track(process.Id);
            logger.Info($"Process started: PID={process.Id}, Command={command}");
            logger.ToolResult("run_process", true);
            return new RunProcessResult { Pid = process.Id };
        }
        catch (Exception ex)
        {
            logger.ToolResult("run_process", false, ex.Message);
            return new RunProcessResult { Error = $"Failed to start process: {ex.Message}" };
        }
    }
}

public class RunProcessResult
{
    public int? Pid { get; set; }
    public string? Error { get; set; }
}
```

```csharp
// Tools/ReadFileTool.cs
using System.ComponentModel;
using System.Text;
using ModelContextProtocol.Server;

namespace CcWin.Tools;

[McpServerToolType]
public static class ReadFileTool
{
    private const long MaxFileSize = 10 * 1024 * 1024; // 10MB

    [McpServerTool]
    [Description("Read the contents of a file. Use this instead of 'cat', 'type', or any shell command.")]
    public static ReadFileResult ReadFile(
        ProjectContext context,
        AppLogger logger,
        [Description("Relative path from project directory")] string path)
    {
        logger.ToolCall("read_file", new { path });

        if (!context.IsPathAllowed(path, out var absolutePath))
        {
            logger.ToolResult("read_file", false, $"Path not allowed: {path}");
            return new ReadFileResult { Error = $"Path not allowed: {path}" };
        }

        if (!File.Exists(absolutePath))
        {
            logger.ToolResult("read_file", false, $"File not found: {path}");
            return new ReadFileResult { Error = $"File not found: {path}" };
        }

        var fileInfo = new FileInfo(absolutePath);
        if (fileInfo.Length > MaxFileSize)
        {
            logger.ToolResult("read_file", false, $"File too large: {fileInfo.Length}");
            return new ReadFileResult { Error = $"File too large: {fileInfo.Length} bytes" };
        }

        try
        {
            using var stream = new FileStream(absolutePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
            using var reader = new StreamReader(stream, Encoding.UTF8);
            logger.ToolResult("read_file", true);
            return new ReadFileResult { Content = reader.ReadToEnd() };
        }
        catch (Exception ex)
        {
            logger.ToolResult("read_file", false, ex.Message);
            return new ReadFileResult { Error = $"Failed to read file: {ex.Message}" };
        }
    }
}

public class ReadFileResult
{
    public string? Content { get; set; }
    public string? Error { get; set; }
}
```

```csharp
// Tools/CloseWindowTool.cs
using System.ComponentModel;
using System.Diagnostics;
using System.Runtime.InteropServices;
using ModelContextProtocol.Server;

namespace CcWin.Tools;

[McpServerToolType]
public static class CloseWindowTool
{
    [DllImport("user32.dll")]
    private static extern bool PostMessage(IntPtr hWnd, uint Msg, IntPtr wParam, IntPtr lParam);

    private const uint WM_CLOSE = 0x0010;

    [McpServerTool]
    [Description("Gracefully close a window by PID. Only works for processes started by run_process. Sends WM_CLOSE so the app can clean up properly.")]
    public static CloseWindowResult CloseWindow(
        ProcessTracker tracker,
        AppLogger logger,
        [Description("Process ID returned by run_process")] int pid)
    {
        logger.ToolCall("close_window", new { pid });

        if (!tracker.IsTracked(pid))
        {
            logger.ToolResult("close_window", false, $"Process not tracked: {pid}");
            return new CloseWindowResult { Error = $"Process not tracked: {pid}" };
        }

        try
        {
            var process = Process.GetProcessById(pid);
            if (process.MainWindowHandle != IntPtr.Zero)
            {
                PostMessage(process.MainWindowHandle, WM_CLOSE, IntPtr.Zero, IntPtr.Zero);
                tracker.Remove(pid);
                logger.Info($"Process closed: PID={pid}");
                logger.ToolResult("close_window", true);
                return new CloseWindowResult { Success = true };
            }
            else
            {
                logger.ToolResult("close_window", false, $"No main window: {pid}");
                return new CloseWindowResult { Error = $"Process has no main window: {pid}" };
            }
        }
        catch (ArgumentException)
        {
            tracker.Remove(pid);
            logger.ToolResult("close_window", false, $"Process not found: {pid}");
            return new CloseWindowResult { Error = $"Process not found: {pid}" };
        }
        catch (Exception ex)
        {
            logger.ToolResult("close_window", false, ex.Message);
            return new CloseWindowResult { Error = $"Failed to close window: {ex.Message}" };
        }
    }
}

public class CloseWindowResult
{
    public bool Success { get; set; }
    public string? Error { get; set; }
}
```

```csharp
// ProjectContext.cs
namespace CcWin;

public class ProjectContext
{
    public string ProjectDirectory { get; }

    public ProjectContext(string projectDirectory)
    {
        ProjectDirectory = Path.GetFullPath(projectDirectory);
    }

    public bool IsPathAllowed(string path, out string absolutePath)
    {
        absolutePath = Path.GetFullPath(path, ProjectDirectory);
        return absolutePath.StartsWith(ProjectDirectory + Path.DirectorySeparatorChar)
            || absolutePath == ProjectDirectory;
    }
}
```

```csharp
// ProcessTracker.cs
namespace CcWin;

public class ProcessTracker
{
    private readonly HashSet<int> _trackedPids = new();

    public void Track(int pid) => _trackedPids.Add(pid);
    public bool IsTracked(int pid) => _trackedPids.Contains(pid);
    public void Remove(int pid) => _trackedPids.Remove(pid);
}
```

---

## Plugin Configuration

### .claude-plugin/marketplace.json

```json
{
  "name": "cc_win_marketplace",
  "owner": {
    "name": "Local"
  },
  "plugins": [
    {
      "name": "cc_win",
      "source": "./cc_win_plugin",
      "description": "Native Windows tools for Claude Code"
    }
  ]
}
```

### cc_win_plugin/.claude-plugin/plugin.json

```json
{
  "name": "cc_win",
  "description": "Native Windows tools for Claude Code. Use these instead of shell commands.",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

### cc_win_plugin/.mcp.json

```json
{
  "mcpServers": {
    "cc_win": {
      "command": "${CLAUDE_PLUGIN_ROOT}/CcWin.exe",
      "type": "stdio"
    }
  }
}
```

Note: `${CLAUDE_PLUGIN_ROOT}` is expanded by CC to the plugin directory.

### cc_win_plugin/skills/windows-native/SKILL.md

```markdown
# Windows Native Tools Skill

Use native Windows tools instead of shell commands.

## Tools Available

- **list_files** — List directory contents. Use instead of `dir`, `ls`.
- **run_process** — Start a process. Use instead of shell commands.
- **read_file** — Read file contents. Use instead of `cat`, `type`.
- **close_window** — Close a window by PID. Only works for processes started by run_process.

## Rules

1. NEVER use bash, sh, cmd, or powershell
2. NEVER use shell commands like dir, ls, cat, type, copy, move, del
3. ALWAYS use the tools above for file and process operations
4. Working directory for run_process is always the project root
5. All file paths must be within the project directory

## Examples

### List files
list_files(path=".")
list_files(path="src")

### Run a build
run_process(command="dotnet", args=["build"])

### Read a log file
read_file(path="logs/app.log")

### Start and close an app
pid = run_process(command="notepad")
close_window(pid=pid)
```

### cc_win_plugin/README.md

```markdown
# cc_win_plugin

MCP server plugin for Claude Code on Windows. Provides native Windows tools.

## Tools

- **list_files** — List directory contents
- **run_process** — Start a process, returns PID
- **read_file** — Read file contents
- **close_window** — Close window by PID

## Build

dotnet build src/CcWin.csproj
copy out\bin\Debug\net10.0\CcWin.exe .

## Install

See parent marketplace for installation instructions.
```

---

## Tool Descriptions for CC

These descriptions tell CC when to use each tool:

**list_files:**
> List files and directories at a path. Use this instead of 'dir', 'ls', or any shell command to see directory contents. Returns name, type (file/directory), size, and modification time.

**run_process:**
> Start a process and return its PID. Use this instead of shell commands. Working directory is always the project root. For commands like 'dotnet build', pass command="dotnet" and args=["build"].

**read_file:**
> Read the contents of a file. Use this instead of 'cat', 'type', or any shell command to read files. Returns file content as string.

**close_window:**
> Gracefully close a window by PID. Only works for processes started by run_process. Sends WM_CLOSE message so the app can clean up properly.

---

## Installation

### Step 1: Build

```
cd cc_win_plugin
dotnet publish src/CcWin.csproj -r win-x64
copy out\bin\Debug\net10.0\win-x64\publish\CcWin.exe .
```

### Step 2: Folder Structure

The plugin must be inside a marketplace folder:

```
./                                    (cc_win_marketplace)
├── .claude-plugin/
│   └── marketplace.json
└── cc_win_plugin/
    ├── .claude-plugin/
    │   └── plugin.json
    ├── .mcp.json
    ├── src/
    ├── out/
    ├── logs/
    └── CcWin.exe
```

### Step 3: Install in CC

```
/plugin marketplace add /path/to/cc_win_marketplace
/plugin install cc_win@cc_win_marketplace
```

### Step 4: Restart CC

### Step 5: Verify

```
/mcp
```

Should show `cc_win: connected`

---

## Test Scenario

### Test 1: Standalone Exe Test (before plugin install)

Test the MCP server directly without CC. From `cc_win_marketplace/cc_win_plugin` folder:

1. Create `test_mcp.ps1` with this content:

```powershell
# Test cc_win MCP server
# Sends JSON-RPC messages via stdin, reads responses from stdout

$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = ".\CcWin.exe"
$psi.RedirectStandardInput = $true
$psi.RedirectStandardOutput = $true
$psi.UseShellExecute = $false
$psi.CreateNoWindow = $true

$process = [System.Diagnostics.Process]::Start($psi)

# Send initialize
$init = '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
$process.StandardInput.WriteLine($init)
Start-Sleep -Milliseconds 500
Write-Host "=== Initialize Response ===" 

# Send initialized notification
$initialized = '{"jsonrpc":"2.0","method":"notifications/initialized"}'
$process.StandardInput.WriteLine($initialized)
Start-Sleep -Milliseconds 100

# List tools
$tools = '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
$process.StandardInput.WriteLine($tools)
Start-Sleep -Milliseconds 500
Write-Host "=== Tools List Response ==="

# Call list_files
$call = '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_files","arguments":{"path":"."}}}'
$process.StandardInput.WriteLine($call)
Start-Sleep -Milliseconds 500
Write-Host "=== list_files Response ==="

# Read all output
$process.StandardInput.Close()
$output = $process.StandardOutput.ReadToEnd()
Write-Host $output

$process.WaitForExit(5000)
$process.Kill()

Write-Host "`n=== Test Complete ==="
Write-Host "Check logs/cc_win_plugin.log for entries"
```

2. Run: `powershell -ExecutionPolicy Bypass -File test_mcp.ps1`

3. Expected output should include:
   - JSON response with `"result"` containing server info
   - JSON response with `"tools"` array listing 4 tools
   - JSON response with `"files"` array listing directory contents

4. Check `logs/cc_win_plugin.log` exists and contains tool call entries

### Test 2: Plugin Integration Test (after plugin install)

1. CC runs: `list_files` with path "."
   - Should return project contents

2. CC runs: `list_files` with path ".."
   - Should return error (outside project)

3. CC runs: `run_process` with command "notepad"
   - Should return PID, notepad opens

4. CC runs: `close_window` with that PID
   - Notepad should close

5. CC runs: `close_window` with random PID (99999)
   - Should return error (not tracked)

6. Check `logs/cc_win_plugin.log`
   - Should contain all tool calls and results

---

## Future Enhancements (not MVP)

- `write_file` — write content to file
- `delete_file` — delete a file
- `wait_process` — wait for process to exit, return exit code
- `process_status` — check if process is running
- `http_get` — HTTP GET request
- `http_post` — HTTP POST request

---

## Success Criteria

1. Plugin publishes without errors
2. Single-file exe in `out/bin/Debug/net10.0/win-x64/publish/`
3. Exe copied to cc_win_plugin root
4. Marketplace added via `/plugin marketplace add`
5. Plugin installed via `/plugin install cc_win@cc_win_marketplace`
6. `/mcp` shows `cc_win: connected`
7. `list_files` works, blocks paths outside project
8. `run_process` starts processes with correct working directory
9. `read_file` can read files while process has them open
10. `close_window` gracefully closes windows
11. Logs created in `logs/` folder
12. Log rotation works (keeps only 3 files)
13. CC can complete build-run-read-close cycle without shell commands

---

## File Checklist

All files that must be created (relative to current directory):

```
./
├── .claude-plugin/
│   └── marketplace.json                    ← from "Plugin Configuration" section
└── cc_win_plugin/
    ├── .claude-plugin/
    │   └── plugin.json                     ← from "Plugin Configuration" section
    ├── .mcp.json                           ← from "Plugin Configuration" section
    ├── .gitignore                          ← from "Project Structure" section
    ├── README.md                           ← from "Plugin Configuration" section
    ├── test_mcp.ps1                        ← from "Test Scenario" section
    ├── skills/
    │   └── windows-native/
    │       └── SKILL.md                    ← from "Plugin Configuration" section
    └── src/
        ├── CcWin.csproj                    ← from "MCP Server Implementation" section
        ├── Program.cs                      ← from "MCP Server Implementation" section
        ├── AppLogger.cs                    ← from "MCP Server Implementation" section
        ├── ProjectContext.cs               ← from "MCP Server Implementation" section
        ├── ProcessTracker.cs               ← from "MCP Server Implementation" section
        └── Tools/
            ├── ListFilesTool.cs            ← from "Tools Implementation" section
            ├── RunProcessTool.cs           ← from "Tools Implementation" section
            ├── ReadFileTool.cs             ← from "Tools Implementation" section
            └── CloseWindowTool.cs          ← from "Tools Implementation" section
```

Total: 16 files to create.
