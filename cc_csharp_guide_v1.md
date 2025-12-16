# C# Development Guide for Claude and Claude Code

## About This Document

This is a **live document**. Claude must update it when new principles, patterns, or preferences are identified during our work together.

**Rules for updating:**
1. When a new pattern, mistake, or preference is discovered, **propose adding it** to this guide
2. **Wait for approval** before pushing changes
3. **When in doubt, ask** if something should be added
4. **Always report** exactly what was added after each update
5. **Present this file as an artifact** after cloning and after every change

## Handoff Protocol

Claude (Opus/claude.ai) builds **prototypes only** by default. Once a project is handed off to the user for Claude Code to work on:

1. **DO NOT push any files** to that project's repo without explicit discussion first
2. Claude Code does the actual implementation work locally
3. If both push independently, it creates merge conflicts and mess
4. To contribute after handoff, discuss changes first, then coordinate

## Collaboration with Claude Code

Claude (Opus) and Claude Code collaborate via escalations:

1. **Claude Code** works independently under user's guidance
2. When CC encounters serious spec/tech issues, it creates an escalation in `escalations/` directory
3. User tells Claude (Opus) "esc" — meaning there's a new escalation to review
4. **Opus summarizes the issue for user** (user doesn't read CC's analysis)
5. User decides, Opus writes resolution in the same file
6. CC reads the resolution and continues

**Escalation file format:** `YYYY_MM_DD__NNN__short_description.md`

**Escalation template:**
```markdown
# Issue: [Short description]

## Problem
[What the issue is]

## Why Blocking
[Why can't proceed]

## Analysis
[Options considered]

## Status
OPEN
```

**Resolution (added by Opus):**
```markdown
## Resolution
[Decision and rationale]

## Status
RESOLVED
```

---

## Table of Contents

1. [Critical Rules](#1-critical-rules)
   1. [Project Initialization](#11-project-initialization)
   2. [Reading Instructions](#12-reading-instructions)
   3. [Checklists for Verification](#13-checklists-for-verification)
2. [Build](#2-build)
   1. [Standalone Executables](#21-standalone-executables)
   2. [Common Mistakes](#22-common-mistakes)
3. [Project Structure](#3-project-structure)
   1. [Directory Layout](#31-directory-layout)
   2. [csproj Output Paths](#32-csproj-output-paths)
   3. [.gitignore](#33-gitignore)
4. [Architecture](#4-architecture)
5. [UI Design Principles](#5-ui-design-principles)
   1. [Configuration Persistence](#51-configuration-persistence)
   2. [Use System UI Components](#52-use-system-ui-components)
   3. [Honor All User Choices from Option Dialogs](#53-honor-all-user-choices-from-option-dialogs)
   4. [Use System Look and Feel](#54-use-system-look-and-feel)
   5. [Match System Theme](#55-match-system-theme)
   6. [Log Panel by Default](#56-log-panel-by-default)
   7. [Panels Must Be Resizable and Collapsible](#57-panels-must-be-resizable-and-collapsible)
6. [Logging](#6-logging)
7. [Process Management](#7-process-management)
   1. [Directories](#71-directories)
   2. [Path Sandboxing](#72-path-sandboxing)
   3. [Error Handling](#73-error-handling)
8. [Silent Tool Calls](#8-silent-tool-calls)
9. [Naming](#9-naming)
10. [Build Number Tracking](#10-build-number-tracking)
11. [Permissions](#11-permissions)

---

## 1. Critical Rules

### 1.1 Project Initialization

When you start working on any project, FIRST check if `CLAUDE.md` exists in the project root.

**If CLAUDE.md does NOT exist, create it immediately:**

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

**If CLAUDE.md exists, read it first** and follow any project-specific rules.

### 1.2 Reading Instructions

After reading this guide or any spec, you MUST:

1. **Report understanding:** State what you understood and what tools/plugins are available
2. **Confirm plugin visibility:** If a plugin is mentioned, confirm you can see it
3. **Ask if unclear:** If anything is ambiguous, ask before proceeding

Example:
> "I've read the guide. I see cc_win_plugin with tools: list_files, run_process, read_file, close_window, get_build_info, wait_for_pattern. Ready to proceed."

### 1.3 Checklists for Verification

Create explicit post-verification checklists for every major step.

**Example — after building:**
```
- [ ] dotnet publish completed without errors
- [ ] publish directory exists
- [ ] exe exists in publish folder
- [ ] exe copied to project root
```

Always run through the checklist and report results before proceeding.

---

## 2. Build

- **.NET version:** 10
- **Configuration:** Debug only — **NEVER use Release**
- **After build:** Copy exe to project root

### 2.1 Standalone Executables

**Required command:**
```
dotnet publish src/ProjectName.csproj -c Debug -r win-x64 --self-contained true -p:PublishSingleFile=true
```

**Flags:**
- `-c Debug` — Debug configuration (REQUIRED)
- `-r win-x64` — Windows 64-bit
- `--self-contained true` — Include .NET runtime
- `-p:PublishSingleFile=true` — Single exe file

**Output:** `out\bin\Debug\net10.0-windows\win-x64\publish\ProjectName.exe`

**After publish:**
```
copy out\bin\Debug\net10.0-windows\win-x64\publish\ProjectName.exe .
```

### 2.2 Common Mistakes

| Wrong | Right |
|-------|-------|
| `dotnet build` | Full publish command |
| `dotnet publish -r win-x64` (missing flags) | Full publish command |
| `-c Release` | `-c Debug` |

---

## 3. Project Structure

### 3.1 Directory Layout

```
project_name/
├── src/
│   ├── ProjectName.csproj
│   └── *.cs
├── out/                          (generated)
├── logs/                         (runtime)
├── .claude/
│   └── settings.json
├── ProjectName.exe               (copied after publish)
├── .gitignore
└── README.md
```

### 3.2 csproj Output Paths

```xml
<PropertyGroup>
  <BaseOutputPath>..\out\bin\</BaseOutputPath>
  <BaseIntermediateOutputPath>..\out\obj\</BaseIntermediateOutputPath>
  <PublishSingleFile>true</PublishSingleFile>
</PropertyGroup>
```

### 3.3 .gitignore

```
out/
logs/
refs/
*.exe
*.dll
*.pdb
*.user
.vs/
```

### 3.4 Reference Repositories

External reference repos (documentation, shared components like `ds_csharp_file_browser`) should be cloned into a `refs/` subdirectory inside the project. This directory is gitignored — reference repos are not committed to the project repo.

```
project_name/
├── refs/                         (gitignored)
│   ├── ds_csharp_file_browser/
│   └── claude_c_sharp_stuff/
├── src/
└── ...
```

---

## 4. Architecture

- **UI Framework:** WPF — unless WinForms is explicitly requested
- **Pattern:** Code-Behind + Service Layer — do NOT use MVVM unless explicitly requested

---

## 5. UI Design Principles

### 5.1 Configuration Persistence

**Key Principle:** If a user can resize, move, collapse, or configure any UI element, that configuration MUST survive application restart. Same with all application options.

When implementing any UI element with user-configurable state, it MUST be:
1. **Saved automatically** when the user changes it
2. **Restored automatically** when the application starts
3. **Stored in AppSettings** with a sensible default value

| UI Element | Properties to Persist |
|------------|----------------------|
| Resizable panels | Width/Height |
| Splitters/Dividers | Position |
| Collapsible sections | Expanded/Collapsed |
| Column widths | Width of each column |
| Tab selection | Active tab index |
| Sort order | Column and direction |
| Filter state | Active filters |
| Window | Size, Position, Maximized state |

### 5.2 Use System UI Components

Use system UI components whenever functionality is close to desired. E.g., font selection must use the system font dialog, file selection must use the system file picker.

### 5.3 Honor All User Choices from Option Dialogs

When using option dialogs (file picker, font picker, color picker, etc.), ALL user selections MUST be captured and applied. If a font dialog allows selecting font family, size, and style — all three must be saved and used, not just the family name.

### 5.4 Use System Look and Feel

UI elements should match native Windows appearance. Use system icons (via Shell API), system colors, and default control styles. Do not use custom styling that deviates from platform visual language.

### 5.5 Match System Theme

All UI must match system theme. Use system colors, fonts, and visual styles that correspond to user's Windows theme settings. Never hardcode colors — use system color resources (e.g., `SystemColors` in WPF) so UI automatically adapts to user's theme.

### 5.6 Log Panel by Default

Include a log panel by default in UI applications. This aids debugging and helps users understand what the application is doing. Log key events, errors, and state changes. Log panel follows the same theme as the rest of the application.

### 5.7 Panels Must Be Resizable and Collapsible

Every panel must be:
- **Resizable** via splitter/drag handle
- **Collapsible** via toggle button or header click
- **Persistent** — size and collapsed state saved per 5.1

### 5.8 File Browser Requirements

Unless specified otherwise, a file browser must show the complete filesystem starting from root (drives on Windows). The tree should expand to reveal the target location (last used folder or user's home directory by default). Users should never be restricted to a subset of the filesystem.

Use the reusable file browser control from [ds_csharp_file_browser](https://github.com/dsodol/ds_csharp_file_browser) — a WPF TreeView-based component with system icons and on-demand loading.

---

## 6. Logging

- **Location:** `logs/` subdirectory
- **Rotation:** On startup, rename existing log to timestamped version
- **Cleanup:** Keep only 3 log files
- **Format:** `[yyyy-MM-dd HH:mm:ss.fff] [LEVEL] Message`

---

## 7. Process Management

### 7.1 Directories

- **Plugin directory:** `AppDomain.CurrentDomain.BaseDirectory`
- **Project directory:** `Directory.GetCurrentDirectory()`

### 7.2 Path Sandboxing

- Never allow file access outside project directory
- Validate all paths before use
- Block `..` escaping and absolute paths outside project

### 7.3 Error Handling

**For `run_process` (plugin tool):**
1. Check response for "error" field
2. If error → STOP and report
3. Verify PID is positive integer

**For direct commands:**
1. Check exit code (0 = success)
2. Check stderr for errors
3. If failure → STOP and report

**Never ignore errors.**

---

## 8. Silent Tool Calls

When polling or performing routine operations, do NOT output text for each tool call.

**Only output when:**
- Reporting results
- An error occurs
- User input needed
- User requests verbose mode

---

## 9. Naming

- `snake_case` for project/folder names
- `PascalCase` for C# namespaces, classes, methods

---

## 10. Build Number Tracking

**All apps must track and display a build number.**

### Format

```
YYYY_MM_DD__HH_mm__NNN
```

Example: `2025_12_16__14_30__001`

### Implementation

```csharp
public static class BuildInfo
{
    public const string Number = "2025_12_16__14_30__001";  // Update every build
    public static string PluginDirectory => AppDomain.CurrentDomain.BaseDirectory;
    public static string ProjectDirectory => Directory.GetCurrentDirectory();
}
```

### Display

- **CLI:** Support `--version` flag
- **UI:** Show in window title or status bar
- **Logs:** Log at startup

---

## 11. Permissions

File: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "mcp__plugin_cc_win_cc_win__*",
      "Edit",
      "Read"
    ]
  }
}
```

`mcp__plugin_cc_win_cc_win__*` — Auto-approve all cc_win plugin tools.
