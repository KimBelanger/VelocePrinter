# VelocePrinter - Agent Context

## Project Overview
**VelocePrinter** is an Electron-based network printer emulator designed to capture, render, and analyze **ESC/POS** . It acts as a virtual printer listening on TCP port 9100.

**Original REPO was handling ZPL and ESC/POS** In this fork, we focus exclusively on **ESC/POS** protocol.

**Current Focus:** Integration with **Véloce POS** system.
- **Phase 1 (Active):** Capture raw ESC/POS data streams to hexdump files for analysis.
- **Phase 2 (Planned):** Parse captured data to extract structured information (items, prices) and store in MariaDB.

## Tech Stack
- **Runtime:** Electron (Main + Renderer processes)
- **Backend Logic:** Node.js (embedded in Electron)
- **UI Framework:** jQuery, Bootstrap 5
- **Networking:** Node.js `net` module (TCP Server)
- **External Services:**
  - !! NOT NEEDED IN THIS FORK !! ZPL Rendering: `api.labelary.com`
  - ESC/POS Rendering: `test.rubyks.com` (PHP script)

## Architecture & Key Files

### 1. Main Process
- **[main.js](main.js)**: Entry point. Handles window creation, auto-updates, and native dialogs (file/folder selection).

### 2. Renderer Process (The Core)
- **[ZplEscPrinter/js/main.js](ZplEscPrinter/js/main.js)**: **CRITICAL FILE**. Contains:
  - **TCP Server**: Listens on port 9100 (`startTcpServer`).
  - **Data Handling**: Accumulates chunks, debounces processing (`processData`).
  - **Hexdump Logic**: Captures raw bytes to `logs/hexdumps/` (`captureHexdump`).
  - **Protocol Routing**: Decides if data is ZPL or ESC/POS.
  - **UI Logic**: Settings management, notifications, rendering display.

### 3. Protocol Handlers
- **[ZplEscPrinter/js/esc_commands.js](ZplEscPrinter/js/esc_commands.js)**: Handles specific ESC/POS status/control commands.
- **!! NOT NEEDED IN THIS FORK !! [ZplEscPrinter/js/zpl_commands.js](ZplEscPrinter/js/zpl_commands.js)**: Handles ZPL status/control commands.

### 4. UI Entry
- **[ZplEscPrinter/main.html](ZplEscPrinter/main.html)**: Main application window layout.

## Data Flow
1. **Input**: TCP connection on port 9100 (e.g., from a POS system).
2. **Capture**: Data is buffered in `ZplEscPrinter/js/main.js`.
3. **Hexdump**: Raw binary data is saved to `logs/hexdumps/` with metadata (Timestamp, IP).
4. **Processing**:
   - **ZPL**: !! NOT NEEDED FOR VÉLOCE !! Sent to Labelary API -> Image returned -> Displayed. 
   - **ESC/POS**: Sent to external PHP renderer -> HTML returned -> Displayed in `iframe`.
5. **Output**: Visual representation in the app window + optional file save (PDF/PNG/Raw).

## Development Notes
- **Running**: `yarn start` (starts Electron in dev mode).
- **Building**: `yarn package` or `yarn make`.
- **Configuration**: Settings are stored in `localStorage` (Port, Host, Mode).
- **Debugging**: DevTools are enabled in development mode.

## Common Tasks for Agents
- **Analyze Hexdumps**: Look in `logs/hexdumps/` to understand the structure of incoming POS data.
- **Modify Capture Logic**: Edit `captureHexdump` in `ZplEscPrinter/js/main.js`.
- **Improve Parsing**: Future work will involve parsing the raw buffers in `processData` before/instead of sending them to the external renderer.
