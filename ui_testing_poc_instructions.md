# ui_testing_poc — Instructions for Claude Code

---

## CRITICAL: Project Initialization

**Before doing anything else:**

1. Check if `CLAUDE.md` exists in the project root
2. If NOT, create it with this content:

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

3. If CLAUDE.md already exists, read it and follow its rules

---

## CRITICAL: First Steps After Reading

After reading these instructions, you MUST:

1. **Query plugin build info:** Call `get_build_info()` to verify the plugin is working
2. **Report plugin visibility:** "I can see cc_win_plugin with tools: [list them]" OR "I cannot see cc_win_plugin"
3. **Confirm understanding:** State that you understand the build command: `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true`
4. **Confirm error handling:** State that you will stop and report any errors immediately

Example acknowledgment:
> "I've read the ui_testing_poc instructions. CLAUDE.md exists/created.
> 
> Plugin build info:
> - Build: 2025_12_16__14_30__001
> - Plugin directory: C:\Users\user\.claude\plugins\cache\cc_win_marketplace\cc_win\1.0.0
> - Project directory: C:\Projects\ui_testing_poc
> 
> I can see cc_win_plugin with tools: list_files, run_process, read_file, close_window, get_build_info. I understand I need to use `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true` to build a standalone exe. I will stop and report any errors immediately. Ready to proceed."

If you CANNOT see the plugin or `get_build_info()` fails, say so immediately.

---

## Purpose

This project tests the cc_win_plugin. You will build a simple WPF application, run it, monitor its behavior through log files, and close it gracefully.

---

## cc_win_plugin Guidelines

You have access to the cc_win_plugin which provides native Windows tools. Use these instead of shell commands for file and process operations.

### Available Tools

| Tool | Purpose | Use When |
|------|---------|----------|
| `get_build_info` | Get plugin build number and directories | Starting a new chat, verifying plugin works |
| `list_files` | List directory contents | You need to see what files exist in a folder |
| `run_process` | Launch an application | You need to start a UI app or background process |
| `read_file` | Read file contents | You need to check logs, config files, or any text file |
| `wait_for_pattern` | Block until pattern appears in file | Waiting for a specific log entry (e.g., button click) |
| `close_window` | Gracefully close a window | You need to shut down an application |

### When to Use Plugin vs Direct Commands

**Use direct commands for:**
- Building and compiling (`dotnet publish -r win-x64`)
- File copy/move operations (`copy`)
- Any operation where you need to see output and wait for completion

Run these commands directly — do NOT use bash, cmd, powershell, or any shell. Just run `dotnet` or `copy` as commands.

**Use plugin tools for:**
- Listing directory contents
- Launching applications
- Reading files (especially while another app is running)
- Closing applications

### Important Behavior

- `run_process` starts the application and returns immediately — it does not wait for the app to exit
- `close_window` sends a graceful close message to the window
- All file paths are relative to project root
- Plugin operations are sandboxed to the project directory

### Silent Tool Calls

When polling or performing routine operations, do NOT output text for each tool call.

**Only output text when:**
- Reporting results to the user
- An error occurs
- User input is needed
- User explicitly requests verbose mode

**Example — polling for button click:**
```
# BAD (too verbose):
"Checking log file..."
[calls read_file]
"No click yet. Checking again..."
[calls read_file]
"Still waiting..."
[calls read_file]

# GOOD (silent polling):
[calls read_file silently]
[calls read_file silently]
[calls read_file silently]
"Button click detected!"
```

During Phase 3 (waiting for button click), call `read_file` repeatedly without commentary until the click is detected, then report once.

### CRITICAL: Error Handling

**Every command and tool call MUST be checked for errors.**

For direct commands:
- Exit code 0 = success, non-zero = failure
- If failure → STOP and report the full error

For plugin tools:
- Check response for "error" field
- If error exists → STOP and report
- For run_process: verify PID is a positive integer

**Never ignore errors. Never proceed after a failure.**

---

## Tech Stack

- **Language:** C#
- **Framework:** .NET 10
- **UI:** WPF (Windows Presentation Foundation)
- **Build configuration:** Debug only

---

## Project Structure

```
ui_testing_poc/
├── src/
│   ├── UiTestingPoc.csproj
│   ├── App.xaml
│   ├── App.xaml.cs
│   ├── MainWindow.xaml
│   ├── MainWindow.xaml.cs
│   └── AppLogger.cs
├── out/                          (generated by build)
│   ├── bin/
│   └── obj/
├── logs/                         (created at runtime by app)
│   └── app.log
├── CLAUDE.md                     (created at project start)
├── .gitignore
└── UiTestingPoc.exe              (copied to project root after publish)
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

## csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0-windows</TargetFramework>
    <OutputType>WinExe</OutputType>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <BaseOutputPath>..\out\bin\</BaseOutputPath>
    <BaseIntermediateOutputPath>..\out\obj\</BaseIntermediateOutputPath>
    <PublishSingleFile>true</PublishSingleFile>
  </PropertyGroup>
</Project>
```

---

## Build Commands

**IMPORTANT:** You must use the full publish command to create a standalone single-file executable.

```
dotnet publish src/UiTestingPoc.csproj -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true
copy out\bin\Debug\net10.0-windows\win-x64\publish\UiTestingPoc.exe .
```

**What each flag means:**
- `-c Debug` — Use Debug configuration (REQUIRED — **NEVER use Release**)
- `-r win-x64` — Target Windows 64-bit
- `--self-contained true` — Include .NET runtime (no .NET install needed on target machine)
- `-p:PublishSingleFile=true` — Bundle everything into ONE exe file

**Why all four flags?**
- Without `--self-contained true` → requires .NET runtime installed
- Without `-p:PublishSingleFile=true` → creates multiple DLLs alongside exe

**Common mistake:** Using `dotnet build` or missing flags. This will NOT create a working standalone exe.

**Output location:** `out\bin\Debug\net10.0-windows\win-x64\publish\UiTestingPoc.exe`

After publishing, copy the exe to project root so `run_process` can find it easily.

---

## Application Specification

### Main Window

| Property | Value |
|----------|-------|
| Width | 400 pixels |
| Height | 200 pixels |
| Title | "UI Testing POC" |
| ResizeMode | NoResize |
| WindowStartupLocation | CenterScreen |

### UI Elements

The window contains a single button:

| Property | Value |
|----------|-------|
| Content | "Click Me" |
| Width | 120 pixels |
| Height | 40 pixels |
| Position | Centered (HorizontalAlignment="Center", VerticalAlignment="Center") |

### Behavior

When the button is clicked, the app writes a log entry. That's all it does.

---

## Logging Specification

### AppLogger Class

Create a static class `AppLogger` with:

- **Log file path:** `logs/app.log` relative to the **current working directory** (which is project root when launched via `run_process`)
- **Directory creation:** Create `logs/` folder if it doesn't exist
- **Log format:** `[yyyy-MM-dd HH:mm:ss.fff] [INFO] {message}`
- **Write mode:** Append to file

**Note:** The exe is copied to project root and `run_process` runs it from there, so `logs/` will be created in the project root (e.g., `ui_testing_poc/logs/app.log`).

### Events to Log

| When | Log Message |
|------|-------------|
| Application starts (in App.xaml.cs OnStartup or MainWindow constructor) | `App started` |
| Button is clicked | `Button clicked` |

### Expected Log File Content

After starting the app:
```
[2025-12-16 14:30:00.123] [INFO] App started
```

After clicking the button once:
```
[2025-12-16 14:30:00.123] [INFO] App started
[2025-12-16 14:30:05.456] [INFO] Button clicked
```

---

## Task: Build, Run, and Test

### Phase 1: Build

1. Check/create `CLAUDE.md` (see Project Initialization section)
2. Create `.gitignore`
3. Create all source files listed in the project structure
4. Run `dotnet publish src/UiTestingPoc.csproj -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true`
5. Copy the exe to project root: `copy out\bin\Debug\net10.0-windows\win-x64\publish\UiTestingPoc.exe .`
6. Verify the exe exists

**Phase 1 Verification Checklist:**
```
- [ ] CLAUDE.md exists (created or already present)
- [ ] .gitignore created
- [ ] All source files created (App.xaml, App.xaml.cs, MainWindow.xaml, MainWindow.xaml.cs, AppLogger.cs, UiTestingPoc.csproj)
- [ ] dotnet publish with all flags completed with exit code 0
- [ ] out\bin\Debug\net10.0-windows\win-x64\publish\ directory exists
- [ ] UiTestingPoc.exe exists in publish folder (should be single file, ~60-150MB)
- [ ] copy command succeeded
- [ ] UiTestingPoc.exe exists in project root
```

Report checklist results before proceeding to Phase 2.

### Phase 2: Launch and Verify Startup

1. Use `run_process` to launch `UiTestingPoc.exe`
2. Check the response for errors
3. Verify PID is a positive integer
4. Check that `logs/app.log` exists using `list_files`
5. Read the log file using `read_file` and verify it contains "App started"
6. Report: "App is running. Log shows startup. Waiting for you to click the button."

**Phase 2 Verification Checklist:**
```
- [ ] run_process returned successfully (no "error" field)
- [ ] PID is a positive integer: ____
- [ ] logs/ directory exists
- [ ] logs/app.log file exists
- [ ] Log contains "App started"
```

**If run_process returns an error:**
- STOP immediately
- Report the full error message
- Check if the exe file exists
- Check if the exe path is correct
- Do NOT proceed to Phase 3

### Phase 3: Wait for Button Click

1. Tell me the app is running and waiting
2. Use `wait_for_pattern` to wait for "Button clicked" in the log
3. When pattern is found, proceed to Phase 4

**Use this single call:**
```
wait_for_pattern(path="logs/app.log", pattern="Button clicked", timeout_ms=120000)
```

This blocks internally until the pattern appears — no polling loop needed, no output spam.

**Phase 3 Verification Checklist:**
```
- [ ] Informed user that app is running and waiting for click
- [ ] Called wait_for_pattern (single tool call)
- [ ] Pattern found (or timeout with clear error)
```

### Phase 4: Close and Report

1. Close the application gracefully using `close_window` with the PID
2. Read the final log file content
3. Report what happened:
   - Did the app start successfully?
   - Did you detect the button click?
   - Did the app close gracefully?
   - Show the final log file contents

**Phase 4 Verification Checklist:**
```
- [ ] close_window called with correct PID
- [ ] close_window returned successfully
- [ ] Final log read successfully
- [ ] Report includes: startup status, click detection, close status, log contents
```

---

---

## Permissions Setup

To auto-approve cc_win plugin tools (no prompts), add to `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__plugin_cc_win_cc_win__*"
    ]
  }
}
```

This allows all cc_win plugin tools without asking for permission each time.

---

## Success Criteria

- [ ] CLAUDE.md exists in project root
- [ ] App built with full command: `dotnet publish -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true`
- [ ] Published exe is a single file (no DLLs alongside it)
- [ ] App window appears on screen
- [ ] Log file is created with startup message
- [ ] You detect when I click the button (by reading the log)
- [ ] App closes without crashing
- [ ] You report the full sequence of events
- [ ] You stopped and asked on any issues (did not auto-fix)
