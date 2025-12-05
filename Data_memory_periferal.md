## ⚙️ Data Memory Peripheral

### Overview
The Data Memory peripheral is an APB slave module providing internal RAM for data storage. It supports read and write operations via the APB bus. (Note: Detailed description not explicitly found in the specification; inferred from system context as a basic memory slave.)

### 🔹 Key Features
- 32-bit data bus for efficient access
- Word-aligned addressing
- Simple read/write protocol via APB
- No interrupt support
- Used for general-purpose data storage

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **RAM Block** | Core storage array for data. |
| **APB Interface** | Handles bus transactions (PADDR, PWDATA, PRDATA, PREADY). |
| **Control Logic** | Manages read/write enables and address decoding. |

### How to Use
- Map to address range (e.g., slave0 at base address).
- Use `ld` for read, `st` for write.
- Example:
```verilog
movu r1, 0x0000  // Base address for data memory
st r2, 0[r1]     // Write data from r2 to memory
ld r3, 0[r1]     // Read data into r3
```

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Data misalignment | Non-word access | Ensure 32-bit aligned addresses |
| Overwrite during multi-access | No burst support | Use single transactions |

---
