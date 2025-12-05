
## ⚙️ SPI Flash Loader

### Overview
The SPI Flash Loader (Controller) is responsible for loading program instructions from external SPI flash into internal memory during boot. (Note: Detailed description not explicitly found in the specification; inferred from system context as part of boot process.)

### 🔹 Key Features
- Automatic boot loading from SPI flash
- Supports standard SPI read command
- Programming mode flag (prg_mode)
- Uses dummy PC for addressing
- Ends on 0xFFFFFFFF (EOF)

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **FSM** | States: IDLE, START, READ, END for flash read sequence. |
| **Shift Register** | Handles serial command and data shift. |
| **Data Buffer** | Buffers received instructions before write to memory. |
| **Timer** | Counts bits/clocks for command and data phases. |

### How to Use
- Automatic on reset; no user code needed for boot.
- After loading, prg_mode = 1 to start processor.
- Example boot flow (hardware): Reset → Load flash → Run from memory.

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Incomplete load | Invalid flash data | Check EOF marker |
| Stall after boot | prg_mode not set | Ensure FSM ends properly |

---
