# C# Development Guidelines for Claude Code

## Build

- **.NET version:** 10
- **Configuration:** Debug only (no Release builds)
- **After build:** Copy exe to project root
- **Build command:** `dotnet publish src/ProjectName.csproj -r win-x64`
- **Copy command:** `copy out\bin\Debug\net10.0\win-x64\publish\ProjectName.exe .`

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

- **Plugin directory:** Use `AppDomain.CurrentDomain.BaseDirectory` for plugin's own files (logs)
- **Project directory:** Use `Directory.GetCurrentDirectory()` for sandbox (file operations within user's project)
- **Working directory:** When starting child processes, set to project directory
- **Path sandboxing:** Never allow file access outside project directory
- Validate all paths before use
- Resolve relative paths against project root
- Block `..` escaping and absolute paths outside project

## Naming

- Use `snake_case` for project/folder names
- Use `PascalCase` for C# namespaces, classes, methods

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
