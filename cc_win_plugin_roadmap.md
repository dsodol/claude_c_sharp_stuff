# cc_win_plugin Roadmap

## Goal

Implement all bash tool capabilities natively for Windows, eliminating the need for shell commands. Special treatment for .NET development workflows.

---

## Bash Tool Capabilities (from Anthropic docs)

| Capability | Description |
|------------|-------------|
| Persistent session | Maintains state between commands |
| Shell commands | Run any command |
| Environment variables | Access and modify |
| Working directory | Change and track |
| Command chaining | Multiple commands in sequence |

### Use Cases to Support

1. **Development workflows** — build, test, dev tools
2. **System automation** — scripts, file management
3. **Data processing** — file processing, analysis
4. **Environment setup** — package installation, config

---

## Current Tools (MVP)

| Tool | Status |
|------|--------|
| `list_files` | ✅ Done |
| `read_file` | ✅ Done |
| `run_process` | ✅ Done |
| `close_window` | ✅ Done |
| `get_build_info` | ✅ Done |

---

## Phase 1: File Operations

| Tool | Purpose | Replaces |
|------|---------|----------|
| `write_file` | Write/overwrite file | `echo > file`, `cat > file` |
| `append_file` | Append to file | `echo >> file` |
| `delete_file` | Delete a file | `del`, `rm` |
| `copy_file` | Copy file | `copy`, `cp` |
| `move_file` | Move/rename file | `move`, `mv` |
| `create_directory` | Create folder | `mkdir` |
| `delete_directory` | Delete folder | `rmdir`, `rd` |
| `file_exists` | Check if file exists | `if exist` |
| `get_file_info` | Size, modified date, attributes | `dir`, `ls -l` |

---

## Phase 2: .NET Development (Priority)

| Tool | Purpose | Replaces |
|------|---------|----------|
| `dotnet_build` | Build project | `dotnet build` |
| `dotnet_publish` | Publish standalone exe | `dotnet publish -c Debug -r win-x64` |
| `dotnet_test` | Run tests | `dotnet test` |
| `dotnet_run` | Run project | `dotnet run` |
| `dotnet_restore` | Restore packages | `dotnet restore` |
| `dotnet_clean` | Clean build outputs | `dotnet clean` |
| `dotnet_new` | Create new project | `dotnet new` |
| `dotnet_add_package` | Add NuGet package | `dotnet add package` |

### `dotnet_build` Specification

**Input:**
```json
{
  "project": "src/MyApp.csproj",
  "configuration": "Debug"
}
```

- `project` (string, required): Path to .csproj file
- `configuration` (string, optional): Debug (default) or Release — **but we enforce Debug only**

**Output:**
```json
{
  "success": true,
  "output_path": "out\\bin\\Debug\\net10.0\\MyApp.dll",
  "warnings": 0,
  "errors": 0,
  "build_time_ms": 1234,
  "stdout": "Build succeeded.\n..."
}
```

**Errors:**
- Project not found → `{ "error": "Project not found: {path}" }`
- Build failed → `{ "error": "Build failed", "errors": [...], "stdout": "..." }`

### `dotnet_publish` Specification

**Input:**
```json
{
  "project": "src/MyApp.csproj",
  "runtime": "win-x64",
  "copy_to_root": true
}
```

- `project` (string, required): Path to .csproj file
- `runtime` (string, optional): Default `win-x64`
- `copy_to_root` (boolean, optional): Copy exe to project root after publish

**Output:**
```json
{
  "success": true,
  "exe_path": "MyApp.exe",
  "publish_path": "out\\bin\\Debug\\net10.0\\win-x64\\publish\\MyApp.exe",
  "build_time_ms": 5678
}
```

### `dotnet_test` Specification

**Input:**
```json
{
  "project": "tests/MyApp.Tests.csproj",
  "filter": "FullyQualifiedName~UnitTests"
}
```

**Output:**
```json
{
  "success": true,
  "total": 42,
  "passed": 40,
  "failed": 2,
  "skipped": 0,
  "duration_ms": 3456,
  "failures": [
    {
      "name": "TestMethod1",
      "message": "Expected 5 but got 4"
    }
  ]
}
```

---

## Phase 3: Text Operations

| Tool | Purpose | Replaces |
|------|---------|----------|
| `search_in_file` | Find text in file | `grep`, `findstr` |
| `search_in_files` | Find text in multiple files | `grep -r`, `findstr /s` |
| `replace_in_file` | Find and replace | `sed` |
| `get_line` | Get specific line(s) | `head`, `tail`, `sed -n` |
| `count_lines` | Count lines in file | `wc -l` |

### `search_in_files` Specification

**Input:**
```json
{
  "pattern": "TODO",
  "path": "src",
  "file_pattern": "*.cs",
  "case_sensitive": false,
  "max_results": 100
}
```

**Output:**
```json
{
  "matches": [
    {
      "file": "src\\Program.cs",
      "line": 42,
      "content": "// TODO: implement this"
    }
  ],
  "total_matches": 5,
  "files_searched": 12
}
```

---

## Phase 4: Process Management

| Tool | Purpose | Replaces |
|------|---------|----------|
| `process_status` | Check if process is running | `tasklist`, `ps` |
| `wait_process` | Wait for process to exit | `wait` |
| `kill_process` | Force kill process | `taskkill`, `kill` |
| `get_process_output` | Get stdout/stderr from process | capture output |

### `wait_process` Specification

**Input:**
```json
{
  "pid": 12345,
  "timeout_ms": 30000
}
```

**Output:**
```json
{
  "exited": true,
  "exit_code": 0,
  "duration_ms": 5432
}
```

Or if timeout:
```json
{
  "exited": false,
  "error": "Process did not exit within 30000ms"
}
```

---

## Phase 5: Environment & System

| Tool | Purpose | Replaces |
|------|---------|----------|
| `get_env` | Get environment variable | `echo %VAR%`, `$env:VAR` |
| `set_env` | Set environment variable (session) | `set VAR=value` |
| `get_cwd` | Get current working directory | `cd`, `pwd` |
| `set_cwd` | Change working directory | `cd path` |
| `get_system_info` | OS, CPU, memory | `systeminfo` |

---

## Phase 6: Git Operations

| Tool | Purpose | Replaces |
|------|---------|----------|
| `git_status` | Repository status | `git status` |
| `git_add` | Stage files | `git add` |
| `git_commit` | Commit changes | `git commit` |
| `git_log` | View history | `git log` |
| `git_diff` | View changes | `git diff` |
| `git_branch` | List/create branches | `git branch` |
| `git_checkout` | Switch branches | `git checkout` |

---

## Phase 7: Archive Operations

| Tool | Purpose | Replaces |
|------|---------|----------|
| `create_zip` | Create zip archive | `tar -czf`, `Compress-Archive` |
| `extract_zip` | Extract zip archive | `tar -xzf`, `Expand-Archive` |
| `list_zip` | List archive contents | `tar -tzf` |

---

## Implementation Priority

1. **Phase 2: .NET Development** — Most critical for our workflow
2. **Phase 1: File Operations** — Foundation for everything
3. **Phase 4: Process Management** — Needed for app testing
4. **Phase 3: Text Operations** — Code search and editing
5. **Phase 5: Environment** — Session management
6. **Phase 6: Git** — Version control
7. **Phase 7: Archives** — Nice to have

---

## Design Principles

1. **No shell commands** — Everything is native C# / Win32 API
2. **Structured output** — JSON responses, not text parsing
3. **Error details** — Rich error information, not just "failed"
4. **Sandboxed** — All operations within project directory
5. **Debug only** — Enforce Debug configuration for .NET
6. **Logging** — Every operation logged with parameters and result

---

## Tool Naming Convention

- Use `snake_case` for all tool names
- Prefix with domain: `dotnet_`, `git_`, `file_`
- Verbs: `get_`, `set_`, `create_`, `delete_`, `list_`, `search_`

---

## Notes

- Each tool should have comprehensive error messages
- All file paths validated against sandbox
- .NET tools should parse MSBuild output for structured responses
- Consider async execution for long-running operations
