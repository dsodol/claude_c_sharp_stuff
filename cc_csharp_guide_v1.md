# C# Development Guidelines for Claude Code

---

## CRITICAL: Project Initialization

When you start working on any project, FIRST check if `CLAUDE.md` exists in the project root.

**If CLAUDE.md does NOT exist, create it immediately with this content:**

```markdown
# CLAUDE.md

## Rules

1. **STOP on issues.** When you find a problem, DO NOT fix it automatically. Instead:
   - Explain the issue clearly
   - Propose a solution
   - Wait for my approval before proceeding

2. **Report errors immediately.** Never ignore failed commands or tool errors.

3. **Verify before proceeding.** After each major step, confirm success before moving on.
```

**If CLAUDE.md exists, read it first** and follow any project-specific rules it contains.

---

## CRITICAL: Reading Instructions

After reading this guide or any spec, you MUST:

1. **Report understanding:** State what you understood and what tools/plugins are available
2. **Confirm plugin visibility:** If a plugin is mentioned, confirm you can see it in your available tools
3. **Ask if unclear:** If anything is ambiguous, ask before proceeding

Example acknowledgment:
> "I've read the guide. I see the cc_win_plugin is available with tools: list_files, run_process, read_file, close_window. I understand I should use `dotnet publish` with `-r win-x64` for standalone builds. Ready to proceed."

---

## CRITICAL: Checklists for Verification

When you create instructions for yourself or receive instructions, you MUST create explicit post-verification checklists.

**Example — after building:**
```
Build verification checklist:
- [ ] dotnet publish completed without errors
- [ ] out\bin\Debug\net10.0-windows\win-x64\publish\ directory exists
- [ ] ProjectName.exe exists in publish folder
- [ ] Copied exe to project root
- [ ] Project root contains ProjectName.exe
```

**Example — after running a process:**
```
Process verification checklist:
- [ ] run_process returned a PID (not an error)
- [ ] PID is a positive integer
- [ ] Expected output file/log exists (if applicable)
- [ ] No error messages in response
```

Always run through the checklist and report results before proceeding.

---

## Build

- **.NET version:** 10
- **Configuration:** Debug only — **NEVER use Release**
- **After build:** Copy exe to project root

### Building Standalone Executables

**This is critical.** To create a standalone `.exe` that runs without .NET installed:

```
dotnet publish src/ProjectName.csproj -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true
```

**What each flag means:**
- `-c Debug` — Use Debug configuration (REQUIRED — never use Release)
- `-r win-x64` — Target Windows 64-bit runtime
- `--self-contained true` — Include .NET runtime in the exe (no .NET install needed)
- `-p:PublishSingleFile=true` — Bundle everything into ONE exe file

**Why all four flags?**
- Without `--self-contained true` → requires .NET runtime installed
- Without `-p:PublishSingleFile=true` → creates multiple DLLs alongside exe

**Output location:** `out\bin\Debug\net10.0\win-x64\publish\ProjectName.exe`

**For WPF apps** (target framework `net10.0-windows`):
```
dotnet publish src/ProjectName.csproj -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true
```
Output: `out\bin\Debug\net10.0-windows\win-x64\publish\ProjectName.exe`

**After publish, copy to project root:**
```
copy out\bin\Debug\net10.0\win-x64\publish\ProjectName.exe .
```

Or for WPF:
```
copy out\bin\Debug\net10.0-windows\win-x64\publish\ProjectName.exe .
```

### Common Mistakes

| Wrong | Right |
|-------|-------|
| `dotnet build` | `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true` |
| `dotnet publish -r win-x64` (missing flags) | `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true` |
| `dotnet publish -c Release` | `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true` |

**`dotnet build` does NOT create a standalone exe.** It creates a .dll that requires `dotnet` to run. Always use the full publish command with all four flags.

## Project Structure

```
project_name/
├── src/
│   ├── ProjectName.csproj
│   └── *.cs
├── out/                          (generated)
│   ├── bin/
│   └── obj/
├── logs/                         (runtime)
├── .claude/
│   └── settings.json
├── ProjectName.exe               (copied after publish)
├── .gitignore
└── README.md
```

## csproj Output Paths

```xml
<PropertyGroup>
  <BaseOutputPath>..\out\bin\</BaseOutputPath>
  <BaseIntermediateOutputPath>..\out\obj\</BaseIntermediateOutputPath>
  <PublishSingleFile>true</PublishSingleFile>
</PropertyGroup>
```

## .gitignore

Always include:

```
out/
logs/
*.exe
*.dll
*.pdb
*.user
.vs/
```

## Logging

- **Location:** `logs/` subdirectory (relative to project root)
- **Rotation:** On startup, rename existing log to timestamped version
- **Cleanup:** Keep only 3 log files, delete oldest first
- **Format:** `[yyyy-MM-dd HH:mm:ss.fff] [LEVEL] Message`

## Process Management

### Directories

- **Plugin directory:** Use `AppDomain.CurrentDomain.BaseDirectory` for plugin's own files (logs)
- **Project directory:** Use `Directory.GetCurrentDirectory()` for sandbox (file operations within user's project)
- **Working directory:** When starting child processes, set to project directory

### Path Sandboxing

- Never allow file access outside project directory
- Validate all paths before use
- Resolve relative paths against project root
- Block `..` escaping and absolute paths outside project

### CRITICAL: Process Execution Error Handling

When you run a process (via `run_process` or direct command), you MUST check the result:

**For `run_process` (plugin tool):**
```
1. Check response for "error" field
2. If error exists → STOP and report the error
3. If success → verify PID is a positive integer
4. If PID is missing or invalid → STOP and report
```

**For direct commands (`dotnet`, `copy`, etc.):**
```
1. Check exit code (0 = success, non-zero = failure)
2. Check stderr for error messages
3. If failure → STOP and report the error with full output
4. Do NOT proceed to next step if previous step failed
```

**Example error handling flow:**
```
I ran: dotnet publish src/App.csproj -r win-x64

Result: Exit code 1
Stderr: error CS1002: ; expected

STOPPING. Build failed with compiler error. Need to fix the source code before proceeding.
```

**Never ignore errors.** If a command or tool call fails, you must:
1. Report the failure immediately
2. Include the full error message
3. Stop the current workflow
4. Suggest how to fix the issue

## Naming

- Use `snake_case` for project/folder names
- Use `PascalCase` for C# namespaces, classes, methods

## Build Number Tracking

**All apps must track and display a build number.**

### Build Number Format

```
YYYY_MM_DD__HH_mm__NNN
```

Example: `2025_12_16__14_30__001`

- Date and time of build
- Counter (3 digits, zero-padded)
- Increase counter every build

### Implementation

Create a static class `BuildInfo`:

```csharp
public static class BuildInfo
{
    public const string Number = "2025_12_16__14_30__001";  // Update every build
    
    public static string PluginDirectory => AppDomain.CurrentDomain.BaseDirectory.TrimEnd(Path.DirectorySeparatorChar);
    public static string ProjectDirectory => Directory.GetCurrentDirectory();
}
```

### Logging

**Always log build number at startup:**

```csharp
AppLogger.Info($"Build: {BuildInfo.Number}");
AppLogger.Info($"Plugin directory: {BuildInfo.PluginDirectory}");
AppLogger.Info($"Project directory: {BuildInfo.ProjectDirectory}");
```

### CLI Apps

Add an optional command line parameter `--version` or `-v`:

```csharp
if (args.Contains("--version") || args.Contains("-v"))
{
    Console.WriteLine($"Build: {BuildInfo.Number}");
    Console.WriteLine($"Plugin directory: {BuildInfo.PluginDirectory}");
    Console.WriteLine($"Project directory: {BuildInfo.ProjectDirectory}");
    return;
}
```

### UI Apps (WPF)

Show build number in the window title or a status bar:

```csharp
// In MainWindow constructor or Loaded event
Title = $"App Name - Build {BuildInfo.Number}";
```

Or add a small label in the corner of the window.

### Updating the Build Number

**Every time you build:**
1. Update the `BuildInfo.Number` constant
2. Use current date/time
3. Increment the counter

Example sequence:
```
2025_12_16__14_30__001
2025_12_16__14_35__002
2025_12_16__15_00__003
```

## Permissions

File: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Edit",
      "Read",
      "Bash(mkdir:*)",
      "Bash(copy:*)",
      "Bash(move:*)",
      "Bash(del:*)",
      "Bash(type:*)",
      "Bash(dir:*)",
      "Bash(dotnet:*)",
      "Bash(git:*)"
    ],
    "deny": [
      "Bash(rmdir /s:*)",
      "Bash(format:*)",
      "Bash(del /s /q:*)"
    ]
  }
}
```

**Note:** Bash permissions are prefix-matched only. They don't restrict paths — `mkdir:*` allows `mkdir C:\anywhere`. This is fallback for when cc_win plugin is unavailable. Plugin has proper path sandboxing.
