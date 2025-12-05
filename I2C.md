
## ⚙️ I2C Peripheral

### Overview
The I2C peripheral is referenced in the Control Status Registers for configuration and interrupt handling, but a full module description is not provided in the specification. It supports standard I2C bus operations with enable and flag bits in CSR[1].

(Note: Detailed implementation not found; based on CSR references.)

### 🔹 Key Features
- 9-bit control register in CSR[1]
- Interrupt enable (i2c_inte) and flag (i2c_intf)
- Likely master/slave support (inferred)
- Integrated with interrupt controller for event handling

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **CSR Interface** | Maps to CSR[1] for control bits [8:0]. |
| **Interrupt Logic** | Sets i2c_intf on events; enabled by i2c_inte. |
| **Bus Controller** | Handles SDA/SCL lines (not detailed). |

CSR[1] - i2c control (Bits [8:0])

From Interrupt Register (CSR[5]):
- Bit 13: i2c inte — RW. Enable bit for I2C interrupt.
- Bit 12: i2c intf — R-Clr. Flag set when an I2C interrupt occurs; cleared by software.

### How to Use
Use csr mov to access CSR[1] for configuration.
- set i2c <bits> to enable/control.
- Check/clear intf in CSR[5].

Example (inferred):
```verilog
set csr1 0-8  // Configure I2C bits
read csr5 12  // Check i2c_intf in flags
clear csr5 12 // Clear flag
```

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Interrupt missed | GIE not set | Enable GIEL/GIEH |
| No response | CSR not written | Verify csr mov usage |

---