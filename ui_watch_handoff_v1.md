# ui_watch Project Handoff

**Version:** 1
**Date:** 2025-12-16

---

## How to Use This Document

When starting a new chat:

1. Attach this handoff document
2. Attach the latest versioned spec files:
   - `cc_win_plugin_spec_v{N}.md`
   - `cc_csharp_guide_v{N}.md`
   - `ui_watch_spec_v{N}.md` (when created)
3. Say: "Continue work on ui_watch project. Increment file versions."

Claude must:
1. Read this entire document first
2. Copy attached files and rename with incremented counter (v1 → v2)
3. Work on the new versioned files only
4. Update this handoff document version if significant decisions are made

---

## Project Overview

### Project Name: ui_watch

### Core Purpose
Enable Claude Code (CC) to test UI applications by providing:
- HTTP API to inspect UI elements
- Text descriptions CC can understand
- Input simulation capabilities

### Components

| Component | Description | Status |
|-----------|-------------|--------|
| ui_watch App | Main deliverable. HTTP API for UI automation + WPF test app | Early spec exists, needs review |
| CC Collaboration Framework | Standards for working with CC (guides, permissions, handoffs) | In progress |
| cc_win_plugin | Separate tool. Solves CC's Windows shell problems | Spec complete |

---

## Origin Story

The project started as a question about screenshot description software but evolved significantly.

### Original Request
> "is there any software which can take a screenshot and describe in detail what is there"

### Clarification 1: Purpose
**Q:** What's the use case?
**A:** Software testing. Not interested in identifying dogs or general images.

### Clarification 2: Language
**Q:** Python?
**A:** "Not Python at all."

**Conclusion:** C# on Windows for UI testing automation.

---

## ui_watch App Decisions

### What It Does
1. Inspects running Windows applications using UI Automation API
2. Returns structured data about UI elements (buttons, text fields, labels, etc.)
3. Provides text descriptions CC can parse
4. Simulates user input (mouse clicks, keyboard input)
5. Captures screenshots

### Test Application
A simple WPF login form for validating the automation works:
- Username field
- Password field
- Login button
- Status label

### API Design (Early)
```
GET  /health              — Check if running
GET  /inspect?pid={pid}   — Get UI tree for process
POST /click               — Click element
POST /type                — Type text into element
POST /keys                — Send keyboard shortcuts
GET  /screenshot          — Capture screenshot
```

### Password Field Note
**Issue raised:** UI Automation intentionally hides password values for security.
**Options discussed:**
1. Accept limitation — test only knows "something was typed"
2. Use regular TextBox for POC (not realistic)
3. Verify via log file instead

**Decision:** Not finalized. Revisit when building.

---

## CC Windows Problems (Led to cc_win_plugin)

### The Problem
User is on Windows. CC has serious issues:

1. CC tries `bash` → **Fails** (no bash on Windows unless WSL)
2. CC tries `pwsh` → **Fails** (PowerShell Core not installed by default)
3. CC tries `powershell.exe` → **Works but asks permission every time**

### Additional Issues
- CC's permission system uses "Bash(...)" naming even on Windows
- No consistent Windows tooling
- Process management is awkward

### Solution Identified
Build an MCP plugin that gives CC native Windows tools. Bypass shell entirely.

---

## cc_win_plugin Decisions

### MVP Scope

**Critical learning:** User had to emphasize "MINIMAL" multiple times. I kept overcomplicating.

**Q:** What tools for MVP?
**A:** After much back-and-forth:
1. `list_files` — directory listing (replaces dir, ls)
2. `run_process` — start process, return PID (replaces shell execution)
3. `read_file` — read file contents (replaces cat, type)
4. `close_window` — gracefully close window by PID

That's it. Four tools. No more.

### Tool Details

#### list_files
**Q:** What should it return?
**A:** Name, type (file/directory), size, modified date. Single level only, not recursive. Sorted: directories first, then files, alphabetically.

#### run_process
**Q:** Should it wait for completion?
**A:** No. Start and return PID immediately. Working directory is ALWAYS project root.

#### read_file
**Q:** Any limits?
**A:** Max 10MB. UTF-8. Must be able to read files open by another process (FileShare.ReadWrite).

#### close_window
**Q:** How to close?
**A:** Send WM_CLOSE message (graceful). Only works for processes started by run_process (tracked in memory).

### Security: Path Sandboxing

**Decision:** All file operations restricted to project directory.

**Rules:**
1. Resolve input path to absolute
2. Normalize (remove `.`, resolve `..`)
3. Check if result starts with project directory
4. If not → return error, do not execute

### Logging

**Q:** Where?
**A:** `logs/` subdirectory relative to project root

**Q:** Rotation?
**A:** On startup, rename existing log to timestamped. Keep only 3 files.

**Q:** Format?
**A:** `[yyyy-MM-dd HH:mm:ss.fff] [LEVEL] Message`

**Q:** What to log?
**A:** Plugin startup, every tool call (name + inputs), every result (success/error), process starts/closes, path validation failures.

---

## Project Structure Decisions

### Nested Folder Debate

**My initial structure:**
```
project_name/
├── src/
│   └── ProjectName/
│       └── *.cs
```

**User's correction:** "You already have project name in the structure" — no need to repeat.

**Final structure:**
```
project_name/
├── src/
│   ├── ProjectName.csproj
│   └── *.cs
├── out/
│   ├── bin/
│   └── obj/
├── logs/
└── ProjectName.exe
```

### Output Directory

**Q:** Where should bin/obj go?
**A:** NOT under src/. Use `out/` directory at project root.

**csproj settings:**
```xml
<BaseOutputPath>..\out\bin\</BaseOutputPath>
<BaseIntermediateOutputPath>..\out\obj\</BaseIntermediateOutputPath>
```

### Working Directory

**Original wording:** "Always set to project root, never exe directory"

**User's correction:** Now that exe is in project root, they're the same. Simplify to: "Always set to project root"

---

## Plugin/Marketplace Structure

### My Failures
I gave FOUR different wrong answers about how to install the plugin:
1. Edit `~/.claude.json` directly ❌
2. Use `/plugin install path` ❌
3. Use `claude mcp add-json` ❌
4. Create marketplace (finally correct ✓)

### Correct Structure
Plugin must be INSIDE a marketplace folder:

```
cc_win_marketplace/           ← User creates this, CC starts here
├── .claude-plugin/
│   └── marketplace.json      ← Marketplace manifest
└── cc_win_plugin/            ← The actual plugin
    ├── .claude-plugin/
    │   └── plugin.json       ← Plugin manifest
    ├── .mcp.json             ← MCP server config
    ├── skills/
    │   └── windows-native/
    │       └── SKILL.md      ← Tells CC to use plugin tools
    ├── src/
    ├── out/
    ├── logs/
    └── CcWin.exe
```

### Installation Commands
```
/plugin marketplace add /path/to/cc_win_marketplace
/plugin install cc_win@cc_win_marketplace
```
Then restart CC.

---

## CC Permissions (Fallback)

### Purpose
When cc_win_plugin is unavailable, allow basic shell commands.

### File Location
`.claude/settings.json` in project

### Content
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

### Important Limitation
**Q:** Can I restrict commands to project directory?
**A:** No. Bash permissions are prefix-matched only. `mkdir:*` allows `mkdir C:\anywhere`. This is fallback. Plugin has proper sandboxing.

### How Patterns Work
- `Bash(mkdir:*)` = prefix match, `:*` means "any continuation"
- CC is aware of `&&` — won't allow `mkdir foo && rm -rf /`
- Wildcard only works at END of pattern

---

## Skills

### Purpose
Tell CC to use plugin tools instead of shell commands.

### Location
`cc_win_plugin/skills/windows-native/SKILL.md`

### Content
```markdown
# Windows Native Tools Skill

Use native Windows tools instead of shell commands.

## Tools Available
- **list_files** — List directory contents. Use instead of `dir`, `ls`.
- **run_process** — Start a process. Use instead of shell commands.
- **read_file** — Read file contents. Use instead of `cat`, `type`.
- **close_window** — Close a window by PID.

## Rules
1. NEVER use bash, sh, cmd, or powershell
2. NEVER use shell commands like dir, ls, cat, type, copy, move, del
3. ALWAYS use the tools above for file and process operations
4. Working directory for run_process is always the project root
5. All file paths must be within the project directory
```

---

## Testing

### Standalone Exe Test
PowerShell script to test MCP server without CC. Sends JSON-RPC via stdin, reads responses from stdout.

See spec for full `test_mcp.ps1` content.

### Plugin Integration Test
After installing in CC:
1. `list_files` with path "." → returns contents
2. `list_files` with path ".." → error (outside project)
3. `run_process` with command "notepad" → returns PID, notepad opens
4. `close_window` with that PID → notepad closes
5. `close_window` with random PID → error (not tracked)
6. Check logs/cc_win_plugin.log → all events logged

---

## Tech Stack

- **Language:** C#
- **Framework:** .NET 10
- **MCP SDK:** `ModelContextProtocol` NuGet package (0.5.0-preview.1)
- **Also needs:** `Microsoft.Extensions.Hosting`
- **Transport:** stdio
- **Build:** Debug only (no Release builds needed)

---

## Build Commands

```
cd cc_win_marketplace/cc_win_plugin
dotnet publish src/CcWin.csproj -r win-x64
copy out\bin\Debug\net10.0\win-x64\publish\CcWin.exe .
```

**Why publish instead of build:**
- `dotnet build` outputs exe + many DLLs
- .NET exe expects DLLs in same folder
- `dotnet publish -p:PublishSingleFile=true` bundles into one exe
- Single file is cleaner for plugin root
- Takes ~15-30 sec vs 5-10 sec for build, but worth it

---

## Behavioral Rules for Claude

### From This Chat

1. **Read specs BEFORE coding** — Don't jump ahead
2. **MVP means MINIMAL** — Don't add features not requested
3. **Listen to corrections** — User had to repeat things multiple times
4. **Don't use profanity** — Even if user does, Claude must not
5. **Create actual files** — Don't just update docs when user needs files
6. **Read documentation** — I gave 4 wrong answers about plugin installation because I didn't read properly
7. **Confirm understanding** — Before creating anything, verify scope

### File Versioning
All spec files must have version counter. When continuing in new chat:
- Attached: `cc_win_plugin_spec_v1.md`
- Create: `cc_win_plugin_spec_v2.md`
- Work on v2 only

---

## Files Created This Session

| File | Purpose |
|------|---------|
| cc_win_plugin_spec_v1.md | Complete plugin specification with all code |
| cc_csharp_guide_v1.md | Reusable C# guidelines for CC |
| settings.json | Permissions fallback file |
| This handoff document | Continuity across chats |

---

## Open Items

1. **ui_watch spec** — Needs review and update, was created early before CC problems were understood
2. **Password field handling** — Decision pending
3. **Test the plugin** — Build and verify it works before moving to ui_watch

---

## Summary of User's Communication Style

- Direct and concise
- Uses caps for emphasis when I'm not listening
- Gets frustrated when I overcomplicate or don't follow instructions
- Values efficiency — no unnecessary features or explanations
- Expects Claude to read documentation properly
- Expects Claude to confirm understanding before acting
