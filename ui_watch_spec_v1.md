# ui_testing_poc — Specification

## Overview

Build a proof-of-concept UI testing toolkit for Windows. Two components:

1. **UiWatch** — HTTP server that inspects UI elements and simulates input
2. **TestApp** — Sample WPF application with logging

The goal: Claude Code can launch both apps, use UiWatch API to inspect and interact with TestApp, verify behavior through UI state and log files.

---

## Tech Stack

- **Language:** C#
- **Framework:** .NET 10
- **UI:** WPF (TestApp)
- **HTTP:** ASP.NET Core minimal API (UiWatch)
- **UI Automation:** System.Windows.Automation
- **Input Simulation:** Win32 SendInput via P/Invoke

---

## Project Structure

```
ui_testing_poc/
├── src/
│   ├── UiWatch/
│   │   ├── UiWatch.csproj
│   │   ├── Program.cs
│   │   ├── ApiEndpoints.cs
│   │   ├── UiInspector.cs
│   │   ├── InputSimulator.cs
│   │   └── AppLogger.cs
│   │
│   └── TestApp/
│       ├── TestApp.csproj
│       ├── App.xaml
│       ├── App.xaml.cs
│       ├── MainWindow.xaml
│       ├── MainWindow.xaml.cs
│       └── AppLogger.cs
│
├── logs/                        (created at runtime)
│
├── ui_testing_poc.sln
├── test_scenario.md
└── README.md
```

---

## Component 1: UiWatch

### Purpose

HTTP server that:
- Inspects UI elements of a target process via UI Automation
- Simulates user input via UI Automation patterns (clean) or SendInput (realistic)

### Project File: UiWatch.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="UIAutomationClient" Version="8.0.0" />
    <PackageReference Include="UIAutomationTypes" Version="8.0.0" />
  </ItemGroup>
</Project>
```

Note: If `UIAutomationClient` / `UIAutomationTypes` packages are not available, use framework references:
```xml
<ItemGroup>
  <Reference Include="UIAutomationClient" />
  <Reference Include="UIAutomationTypes" />
</ItemGroup>
```

### Configuration

- **Port:** 9898 (hardcoded)
- **Host:** localhost only

### API Endpoints

#### GET /health

Returns server status.

Response `200 OK`:
```json
{
  "status": "ok",
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

#### GET /inspect?pid={pid}

Inspects all UI elements of the target process window.

Parameters:
- `pid` (required): Process ID of target application

Response `200 OK`:
```json
{
  "pid": 1234,
  "window": "Test App",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "elements": [
    {
      "id": "el_0",
      "type": "Window",
      "name": "Test App",
      "bounds": { "x": 100, "y": 100, "width": 400, "height": 350 },
      "enabled": true,
      "focused": true,
      "patterns": []
    },
    {
      "id": "el_1",
      "type": "Edit",
      "name": "Username",
      "automationId": "UsernameTextBox",
      "value": "",
      "bounds": { "x": 150, "y": 150, "width": 200, "height": 25 },
      "enabled": true,
      "focused": false,
      "patterns": ["Value", "Text"]
    },
    {
      "id": "el_2",
      "type": "Edit",
      "name": "Password",
      "automationId": "PasswordTextBox",
      "value": "",
      "bounds": { "x": 150, "y": 190, "width": 200, "height": 25 },
      "enabled": true,
      "focused": false,
      "patterns": ["Value", "Text"]
    },
    {
      "id": "el_3",
      "type": "CheckBox",
      "name": "Remember me",
      "automationId": "RememberCheckBox",
      "checked": false,
      "bounds": { "x": 150, "y": 230, "width": 120, "height": 20 },
      "enabled": true,
      "focused": false,
      "patterns": ["Toggle"]
    },
    {
      "id": "el_4",
      "type": "Button",
      "name": "Submit",
      "automationId": "SubmitButton",
      "bounds": { "x": 150, "y": 270, "width": 80, "height": 30 },
      "enabled": true,
      "focused": false,
      "patterns": ["Invoke"]
    },
    {
      "id": "el_5",
      "type": "Button",
      "name": "Clear",
      "automationId": "ClearButton",
      "bounds": { "x": 250, "y": 270, "width": 80, "height": 30 },
      "enabled": true,
      "focused": false,
      "patterns": ["Invoke"]
    },
    {
      "id": "el_6",
      "type": "Text",
      "name": "Ready",
      "automationId": "StatusLabel",
      "bounds": { "x": 150, "y": 320, "width": 200, "height": 20 },
      "enabled": true,
      "focused": false,
      "patterns": []
    }
  ]
}
```

Response `404 Not Found`:
```json
{
  "error": "Process not found",
  "pid": 1234
}
```

Response `400 Bad Request`:
```json
{
  "error": "Missing required parameter: pid"
}
```

#### POST /element/invoke

Invokes a button or clickable element using InvokePattern.

Request:
```json
{
  "pid": 1234,
  "id": "el_4"
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "invoke",
  "elementId": "el_4"
}
```

Response `404 Not Found`:
```json
{
  "error": "Element not found",
  "id": "el_4"
}
```

Response `400 Bad Request`:
```json
{
  "error": "Element does not support Invoke pattern",
  "id": "el_4"
}
```

#### POST /element/setValue

Sets the value of a text input using ValuePattern.

Request:
```json
{
  "pid": 1234,
  "id": "el_1",
  "value": "testuser"
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "setValue",
  "elementId": "el_1",
  "value": "testuser"
}
```

#### POST /element/toggle

Toggles a checkbox using TogglePattern.

Request:
```json
{
  "pid": 1234,
  "id": "el_3"
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "toggle",
  "elementId": "el_3",
  "newState": true
}
```

#### POST /element/focus

Sets focus to an element using SetFocus().

Request:
```json
{
  "pid": 1234,
  "id": "el_1"
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "focus",
  "elementId": "el_1"
}
```

#### POST /window/focus

Brings the target window to foreground.

Request:
```json
{
  "pid": 1234
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "windowFocus",
  "pid": 1234
}
```

#### POST /mouse/move

Moves the mouse cursor to screen coordinates using SendInput.

Request:
```json
{
  "x": 200,
  "y": 300
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "mouseMove",
  "x": 200,
  "y": 300
}
```

#### POST /mouse/click

Clicks at screen coordinates using SendInput.

Request:
```json
{
  "x": 200,
  "y": 300,
  "button": "left"
}
```

`button` values: `"left"`, `"right"`, `"middle"` (default: `"left"`)

Response `200 OK`:
```json
{
  "success": true,
  "action": "mouseClick",
  "x": 200,
  "y": 300,
  "button": "left"
}
```

#### POST /keyboard/type

Types a string using SendInput. Target window should be focused first.

Request:
```json
{
  "text": "hello world"
}
```

Response `200 OK`:
```json
{
  "success": true,
  "action": "keyboardType",
  "text": "hello world"
}
```

#### POST /keyboard/key

Sends a single key press using SendInput.

Request:
```json
{
  "key": "Enter"
}
```

Supported keys: `Enter`, `Tab`, `Escape`, `Backspace`, `Delete`, `Up`, `Down`, `Left`, `Right`, `Home`, `End`, `PageUp`, `PageDown`, `F1`-`F12`

Response `200 OK`:
```json
{
  "success": true,
  "action": "keyboardKey",
  "key": "Enter"
}
```

### UiInspector.cs

Implementation notes:

1. Use `AutomationElement.FromHandle()` with the main window handle of the process
2. Find main window by iterating `Process.GetProcessById(pid).MainWindowHandle`
3. Use `TreeWalker.ControlViewWalker` to traverse the UI tree
4. For each element, extract:
   - `AutomationId`
   - `Name`
   - `ControlType`
   - `BoundingRectangle`
   - `IsEnabled`
   - `HasKeyboardFocus`
   - Supported patterns (check `GetSupportedPatterns()`)
5. Generate sequential IDs (`el_0`, `el_1`, etc.) and maintain a cache mapping ID to AutomationElement for subsequent commands
6. Cache should be cleared/rebuilt on each `/inspect` call

### InputSimulator.cs

Implementation notes for SendInput:

```csharp
[DllImport("user32.dll", SetLastError = true)]
static extern uint SendInput(uint nInputs, INPUT[] pInputs, int cbSize);

[DllImport("user32.dll")]
static extern bool SetCursorPos(int X, int Y);

[DllImport("user32.dll")]
static extern bool SetForegroundWindow(IntPtr hWnd);
```

For keyboard input, use `KEYEVENTF_UNICODE` flag with character codes for typing arbitrary text.

For special keys, map key names to virtual key codes.

### AppLogger.cs

- Log file location: `../../logs/ui_watch.log` (relative to executable, resolves to `ui_testing_poc/logs/`)
- On startup: if log file exists, rename to `ui_watch_{timestamp}.log`
- Log format: `[2025-01-15 10:30:00.123] [INFO] Message`
- Log levels: `INFO`, `WARN`, `ERROR`
- Log all API requests and responses
- Use file locking or queue for thread safety

---

## Component 2: TestApp

### Purpose

Simple WPF application for testing UiWatch capabilities.

### Project File: TestApp.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0-windows</TargetFramework>
    <OutputType>WinExe</OutputType>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### UI Layout: MainWindow.xaml

```
┌─────────────────────────────────────┐
│  Test App                        [X]│
├─────────────────────────────────────┤
│                                     │
│   Username:  [__________________]   │
│                                     │
│   Password:  [__________________]   │
│                                     │
│   [✓] Remember me                   │
│                                     │
│   [ Submit ]     [ Clear ]          │
│                                     │
│   Status: Ready                     │
│                                     │
└─────────────────────────────────────┘
```

### Control Names (AutomationId)

| Control | AutomationId | Type |
|---------|--------------|------|
| Username input | UsernameTextBox | TextBox |
| Password input | PasswordTextBox | PasswordBox |
| Remember checkbox | RememberCheckBox | CheckBox |
| Submit button | SubmitButton | Button |
| Clear button | ClearButton | Button |
| Status label | StatusLabel | TextBlock |

### Window Properties

- Title: "Test App"
- Size: 400 x 350
- ResizeMode: NoResize
- StartupLocation: CenterScreen

### Behavior

**Submit button:**
1. Validate Username is not empty
2. Validate Password is not empty
3. If validation fails: set Status to "Please fill all fields"
4. If validation passes: set Status to "Login successful"
5. Log all actions

**Clear button:**
1. Clear Username field
2. Clear Password field
3. Uncheck Remember checkbox
4. Set Status to "Ready"
5. Log action

**All interactions should be logged.**

### AppLogger.cs

- Log file location: `../../logs/test_app.log`
- On startup: if log file exists, rename to `test_app_{timestamp}.log`
- Log format: `[2025-01-15 10:30:00.123] [ACTION] Button clicked: Submit`
- Log categories: `ACTION`, `INPUT`, `VALIDATION`, `STATE`

**Input field logging:**
- Username TextBox: log on TextChanged with current length
- Password PasswordBox: log on PasswordChanged with length only (never log actual password)

**Log examples:**
```
[2025-01-15 10:30:00.123] [ACTION] Application started
[2025-01-15 10:30:03.456] [INPUT] Username field changed (length: 8)
[2025-01-15 10:30:04.789] [INPUT] Password field received input (length: 9)
[2025-01-15 10:30:05.456] [ACTION] Button clicked: Submit
[2025-01-15 10:30:05.457] [VALIDATION] Username empty: false
[2025-01-15 10:30:05.458] [VALIDATION] Password empty: false
[2025-01-15 10:30:05.459] [STATE] Status changed: "Ready" -> "Login successful"
[2025-01-15 10:30:10.789] [ACTION] Button clicked: Clear
[2025-01-15 10:30:10.790] [STATE] Fields cleared
[2025-01-15 10:30:10.791] [STATE] Status changed: "Login successful" -> "Ready"
```

**Important:** Never log actual password values. Log only that input was received and optionally the length.

---

## Solution File: ui_testing_poc.sln

Create a solution containing both projects.

---

## Build Instructions

```powershell
cd ui_testing_poc
dotnet build -c Release
```

Both projects should build without errors.

---

## Process Management (Critical for Claude Code)

Claude Code needs explicit instructions on how to start processes on Windows and detect when they're ready.

### Starting UiWatch

```powershell
# Build first
dotnet build src/UiWatch/UiWatch.csproj -c Release

# Start UiWatch in a new window, capture process object
$uiwatch = Start-Process -FilePath "dotnet" -ArgumentList "run --project src/UiWatch/UiWatch.csproj -c Release --no-build" -PassThru -WindowStyle Normal

# Store PID
$uiwatchPid = $uiwatch.Id
Write-Host "UiWatch started with PID: $uiwatchPid"

# Poll /health until ready (max 30 seconds)
$ready = $false
$attempts = 0
while (-not $ready -and $attempts -lt 30) {
    Start-Sleep -Seconds 1
    $attempts++
    try {
        $response = Invoke-RestMethod -Uri "http://localhost:9898/health" -Method Get -TimeoutSec 2 -ErrorAction Stop
        if ($response.status -eq "ok") {
            $ready = $true
            Write-Host "UiWatch is ready"
        }
    } catch {
        Write-Host "Waiting for UiWatch... (attempt $attempts)"
    }
}

if (-not $ready) {
    Write-Error "UiWatch failed to start within 30 seconds"
    Stop-Process -Id $uiwatchPid -Force -ErrorAction SilentlyContinue
    exit 1
}
```

### Starting TestApp

```powershell
# Build first
dotnet build src/TestApp/TestApp.csproj -c Release

# Start TestApp, capture process object
$testapp = Start-Process -FilePath "dotnet" -ArgumentList "run --project src/TestApp/TestApp.csproj -c Release --no-build" -PassThru -WindowStyle Normal

# Store PID
$testappPid = $testapp.Id
Write-Host "TestApp started with PID: $testappPid"

# Poll /inspect until window is available (max 30 seconds)
$ready = $false
$attempts = 0
while (-not $ready -and $attempts -lt 30) {
    Start-Sleep -Seconds 1
    $attempts++
    try {
        $response = Invoke-RestMethod -Uri "http://localhost:9898/inspect?pid=$testappPid" -Method Get -TimeoutSec 5 -ErrorAction Stop
        if ($response.window -eq "Test App") {
            $ready = $true
            Write-Host "TestApp window is ready"
        }
    } catch {
        Write-Host "Waiting for TestApp window... (attempt $attempts)"
    }
}

if (-not $ready) {
    Write-Error "TestApp window not detected within 30 seconds"
    Stop-Process -Id $testappPid -Force -ErrorAction SilentlyContinue
    Stop-Process -Id $uiwatchPid -Force -ErrorAction SilentlyContinue
    exit 1
}
```

### Stopping Processes

```powershell
# Graceful shutdown
Stop-Process -Id $testappPid -Force -ErrorAction SilentlyContinue
Stop-Process -Id $uiwatchPid -Force -ErrorAction SilentlyContinue
Write-Host "All processes stopped"
```

---

## Test Scenario

Execute these steps with **2 second delay** between each step so the test can be observed.

### Setup Phase

| Step | Action | Command | Verify |
|------|--------|---------|--------|
| 1 | Build solution | `dotnet build -c Release` | Exit code 0 |
| 2 | Start UiWatch | See PowerShell above | /health returns ok |
| 3 | Start TestApp | See PowerShell above | Window detected via /inspect |

### Test Phase

| Step | Action | Method | Request | Verify |
|------|--------|--------|---------|--------|
| 4 | Inspect UI | GET | `/inspect?pid={testappPid}` | All elements present |
| 5 | Focus Username field | POST | `/element/focus` with UsernameTextBox id | Success |
| 6 | Type username | POST | `/keyboard/type` with `{"text": "testuser"}` | — |
| 7 | Inspect UI | GET | `/inspect?pid={testappPid}` | Username value = "testuser" |
| 8 | Focus Password field | POST | `/element/focus` with PasswordTextBox id | Success |
| 9 | Type password | POST | `/keyboard/type` with `{"text": "secret123"}` | — |
| 9a | Read test_app.log | File read | `logs/test_app.log` | Contains: Password field received input (length: 9) |
| 10 | Toggle Remember checkbox | POST | `/element/toggle` with RememberCheckBox id | newState = true |
| 11 | Inspect UI | GET | `/inspect?pid={testappPid}` | Checkbox checked = true |
| 12 | Click Submit (automation) | POST | `/element/invoke` with SubmitButton id | Success |
| 13 | Inspect UI | GET | `/inspect?pid={testappPid}` | Status label = "Login successful" |
| 14 | Read test_app.log | File read | `logs/test_app.log` | Contains: Submit clicked, validation passed, state changed |
| 15 | Click Clear (raw input) | POST | `/window/focus` then `/mouse/click` on ClearButton bounds | Success |
| 16 | Inspect UI | GET | `/inspect?pid={testappPid}` | All fields empty, Status = "Ready" |
| 17 | Read test_app.log | File read | `logs/test_app.log` | Contains: Clear clicked, fields cleared |

### Teardown Phase

| Step | Action | Command | Verify |
|------|--------|---------|--------|
| 18 | Stop TestApp | `Stop-Process` | Process terminated |
| 19 | Stop UiWatch | `Stop-Process` | Process terminated |
| 20 | Read ui_watch.log | File read | No ERROR entries |

---

## Test Pacing Implementation

Between each step, wait 2 seconds:

```powershell
Write-Host "Step X: <description>"
Start-Sleep -Seconds 2
# execute step
```

This allows observation of the test in real-time.

---

## Error Handling Guidelines

### UiWatch should handle:

- Process not found → 404 with clear message
- Window not accessible → 403 with explanation
- Element not found → 404 with element ID
- Pattern not supported → 400 with pattern name
- Invalid request body → 400 with missing field name
- Internal errors → 500 with details

### All errors should be logged.

---

## README.md Content

Include:
- Project description
- Prerequisites (.NET 10)
- Build instructions
- How to run manually
- API documentation summary
- Troubleshooting tips

---

## Success Criteria

1. Both projects build without errors
2. UiWatch starts and responds to /health
3. TestApp window is detectable via /inspect
4. All UI elements are enumerated with correct types and properties
5. Automation patterns (invoke, setValue, toggle) work correctly
6. Raw input (mouse/keyboard via SendInput) works correctly
7. Logs rotate on startup
8. All test steps pass
9. Logs contain expected entries

---

## Notes for Claude Code

1. **Do not use `dotnet run` for background processes** — use `Start-Process` as shown
2. **Always build before running** with `--no-build` flag on run
3. **Poll endpoints** to detect readiness — don't assume instant startup
4. **Use full paths** when reading log files
5. **2 second delays** between test steps — this is intentional for observation
6. **Check HTTP status codes** — don't assume success
7. **Parse JSON responses** to verify expected values
8. **The element IDs change** on each /inspect call — always get fresh IDs before interacting
