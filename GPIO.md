
## ⚙️ GPIO Peripheral

### Overview
Bidirectional General Purpose Input/Output (GPIO) pin with programmable direction and data control. It allows an external system to configure the pin as either an input or an output, write data to it when set as an output, and read its value when set as an input.

The WR TRIS signal is used to enable writing to the TRIS register, which determines the direction of the GPIO pin. When the TRIS register is set to logic 0, the pin is configured as an output. When it is set to logic 1, the pin becomes an input.

### 🔹 Key Features
- 16-bit bidirectional ports
- Programmable direction (TRIS register)
- Data read/write (PORT register)
- Tri-state control for inputs
- Latch for stable reads

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **TRIS Register** | Controls pin direction (0 = output, 1 = input). |
| **PORT Register** | Holds output data when pin is output. |
| **Read Latch** | Captures input data from pin on read enable. |
| **Tri-State Buffer** | Drives data to pin only in output mode. |

### How to Use
The TRIS register address is 0x00010001, and the data (PORT) register address is 0x00010011.
The following example shows how to declare all 16 GPIO ports as output and store the data 0xF0F0 to the data register.

Example:
```
gpio : GPIO address is 0x00010001 for TRIS register
and 0x00010011 for data register

org 0x00000000
movu r1,0x0001
addh r1,r1,0x0001
; r1 = 0x00010001 (TRIS address)

movu r2,0x0001
addh r2,r2,0x0011
; r2 = 0x00010011 (PORT address)

mov r3,0x00000000
; Declaring all 16 ports as output (TRIS = 0)
st r3,0[r1]
; Write to TRIS register

mov r3,0xF0F0
; Data to write to PORT
st r3,0[r2]
; Store data to PORT register

nop
nop
```

Step 1: Pin Configuration (Input/Output)  
The first operation in the GPIO system is to configure the direction of each pin. This is done by writing to a specific address (address = 94 or 95) using the APB interface. When the PSEL and PWRITE signals are high and the PADDR value is 5’b11110, it indicates a write operation to the direction (TRIS) registers. In this case, each bit of the 16-bit PWDATA input determines the mode of the corresponding GPIO pin. If a bit is 0, the corresponding pin is configured as an output; if it is 1, the pin becomes an input. This configuration is applied to all 16 pins simultaneously.

Step 2: Sending Data to Output Pins  
Once the pins are configured, the processor can send data to those configured as outputs. This is done by writing to the GPIO PORT registers. If PSEL and PWRITE are high and the PADDR value is 5’b11111, the module interprets this as a write to the output data registers. Each bit of the PWDATA signal is then written to the corresponding GPIO pin, but only if that pin is set as an output.

Step 3: Reading Data from Input Pins  
To read data from the GPIO pins configured as inputs, the processor issues a read request. This occurs when PSEL is high, PWRITE is low, and PADDR is 5’b11111. The module then reads the actual logic levels from all GPIO pins and returns their states through the 16-bit PRDATA output. Each bit in PRDATA reflects the real-time logic level of the corresponding pin.

Summary:  
The CPU first sets the direction for each pin (input or output), then writes data to the output pins or reads data from the input pins.

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Bus contention | Read during write | Use PREADY for synchronization |
| Incorrect direction | Wrong TRIS write | Verify address and data before st |

---