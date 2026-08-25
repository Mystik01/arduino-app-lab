# Arduino App Lab ↔ UNO Q (USB) Communication README

This document explains how the desktop App Lab connects to an UNO Q over USB, how apps are created/run, and where `arduino-app-cli`, Arduino CLI, and `adb` are involved.

## Short answer

- **Yes, CLI components are used**:
  - `arduino-app-cli` Go package APIs are used for board discovery, board connections, remote file/process operations, and port forwarding.
  - Arduino CLI gRPC server APIs are used during startup to initialize/install discovery tooling.
- **`adb` is used in this project**, but in specific places:
  - It is bundled/installed as a required tool during board tooling setup.
  - It is used directly when opening a board terminal (`adb -s <serial> shell`) for serial/USB boards.
  - For normal app operations (create/edit/run/stop), App Lab talks to the board orchestrator via HTTP over a forwarded tunnel; this repo does not directly run `adb` commands for every app API call.

---

## 1) Architecture overview (USB case)

For a USB-connected UNO Q, App Lab desktop has this flow:

1. **Frontend (TypeScript/React)** calls Wails bindings (Go backend methods).
2. **Go backend** discovers/selects boards and establishes a `remote.RemoteConn`.
3. For serial/USB boards, backend starts a tunnel from host to board **orchestrator port `8800`**.
4. Frontend orchestrator service sends app API requests (create/list/start/stop/etc.) to `http://localhost:<forwardedPort>`.
5. File tree/content operations are executed through backend remote filesystem calls on the selected board connection.

Key code:
- Orchestrator tunnel and URL: `standalone-apps/app-lab-desktop/internal/board/board.go`
- Tunnel forwarding logic: `standalone-apps/app-lab-desktop/internal/tunnel/tunnel.go`
- Frontend orchestrator calls: `standalone-apps/app-lab-desktop/frontend/src/services/orchestratorService.impl.standalone.ts`
- Backend file APIs: `standalone-apps/app-lab-desktop/internal/app/api.go`
- Remote FS operations: `standalone-apps/app-lab-desktop/internal/fs/fs.go`, `.../filetree.go`

---

## 2) Startup and tooling initialization

On app startup, backend runs:

- `board.InstallToolingIfMissing(...)`

What it does:

- Ensures board discovery/runtime tooling is present.
- Unpacks bundled resources for:
  - `serial-discovery`
  - `mdns-discovery`
  - `adb`
- Uses Arduino CLI gRPC server APIs (`commands.NewArduinoCoreServer`, `ConfigurationGet`, `Init`) to initialize and auto-install discovery packages when needed.

Key code:
- Startup call: `standalone-apps/app-lab-desktop/internal/app/lifecycle.go`
- Tool installation/init logic: `standalone-apps/app-lab-desktop/internal/board/operations.go`
- Resource downloader script used during build prep: `standalone-apps/app-lab-desktop/internal/board/download_resources.sh`

---

## 3) How USB board detection works

Board detection uses supported FQBNs:

- `arduino:zephyr:unoq`
- `arduino:zephyr:ventunoq`

Detection call:

- `board.FromFQBN(ctx, supportedBoards)`

The frontend maps protocol `"serial"` to connection type `"USB"` for UI display.

Key code:
- Supported boards + detect call: `standalone-apps/app-lab-desktop/internal/board/board.go`, `.../operations.go`
- UI protocol mapping: `standalone-apps/app-lab-desktop/frontend/src/services/boardService.mapper.ts`

---

## 4) What happens when you select a USB board

When the user selects a board:

1. Backend matches selected serial from detected boards.
2. Calls `EstablishConnection(...)`.
3. For `SerialProtocol`, calls `apiBoard.GetConnection()`.
4. Immediately starts an orchestrator tunnel tagged `"orchestrator"` to board port `8800`.
5. Stores the active connection and tunnel on `selectedBoard`.

Then frontend resolves orchestrator origin by calling Go `GetOrchestratorURL()` through Wails binding and uses that origin for all orchestrator HTTP API calls.

Key code:
- Selection and connection establishment: `standalone-apps/app-lab-desktop/internal/app/lifecycle.go`, `.../internal/board/board.go`
- Orchestrator URL bridge/frontend usage: `standalone-apps/app-lab-desktop/internal/app/api.go`, `.../frontend/src/services/orchestratorService.impl.standalone.ts`

---

## 5) Create project, edit files, run app: protocol-level flow

### Create app

- Frontend calls orchestrator service `createAppV1Request(...)` with the resolved origin.
- Request goes to board orchestrator over the forwarded localhost tunnel.

### Edit/save files

- File tree/content actions use Wails Go methods (`GetFileTree`, `GetFileContent`, `WriteFileContent`, etc.).
- Go backend executes remote FS operations through the selected board `RemoteConn`.
- This means edits are applied on the board-side app filesystem via the remote connection path.

### Run/stop app

- Frontend calls orchestrator start/stop stream endpoints (for logs/status updates).
- Those requests also go through orchestrator origin (`http://localhost:<forwardedPort>` in desktop USB mode).

Key code:
- Orchestrator app endpoints: `standalone-apps/app-lab-desktop/frontend/src/services/orchestratorService.impl.standalone.ts`
- File operation methods: `standalone-apps/app-lab-desktop/frontend/src/services/arduinoAppFilesService.impl.standalone.ts`, `.../internal/app/api.go`, `.../internal/fs/fs.go`

---

## 6) Where `adb` is and is not used

### Used directly in this repo

1. **Tooling setup**: `adb` package is installed/unpacked as required board tooling.
2. **Open terminal feature**:
   - For serial/USB boards: executes `adb -s <serial> shell`.
   - For network boards: uses `ssh arduino@<ip>`.

Key code:
- `standalone-apps/app-lab-desktop/internal/board/operations.go`
- `standalone-apps/app-lab-desktop/internal/terminal/terminal.go`

### Not directly used for each app API call

- App create/list/start/stop calls are HTTP requests to the board orchestrator via forwarded local tunnel.
- The frontend/backend do not run `adb push`/`adb shell` per orchestrator API action in this codebase.

---

## 7) Is it “remote” or “temporary local then upload”?

For desktop USB flow in this codebase:

- Project lifecycle operations are orchestrator-driven on the connected board side.
- File operations are performed through remote connection APIs.
- Import/export uses local file dialogs and archive upload/download operations, but normal create/edit/run is not a separate manual-upload workflow like the classic IDE sketch upload model.

---

## 8) Security and error handling notes in this path

- Tunnel creation gracefully retries with another host port if default port is unavailable.
- Error middleware maps SSH auth failures into typed errors for UI handling.
- Terminal opening validates serial/IP input before command execution.

Key code:
- Tunnel retry/port allocation: `standalone-apps/app-lab-desktop/internal/tunnel/tunnel.go`
- Error middleware: `standalone-apps/app-lab-desktop/internal/errors/errors.go`
- Terminal sanitization: `standalone-apps/app-lab-desktop/internal/terminal/terminal.go`

