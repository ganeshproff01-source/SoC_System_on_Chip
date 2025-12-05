
## ⚙️ UART Peripheral

### Overview
UART Module Description

The UART module supports serial communication with configurable baud rates and word lengths. It includes separate transmitter and receiver blocks with baud rate generation.

### 🔹 Key Features
- Baud rates: 9600, 19200, 57600, 115200
- Word lengths: 8, 16, 24, 32 bits
- Interrupt flags for TX/RX ready
- Clock select: 50/100 MHz
- Memory-mapped at 0x00010003

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **Baud Rate Generator** | Generates clock for selected baud rate using divider chain. |
| **Transmitter (TX)** | Shifts out data with start/stop bits; handles multi-byte. |
| **Receiver (RX)** | Shifts in data, checks stop bit, buffers received bytes. |
| **Control Register** | 8-bit register for enable, baud select, clock source. |

UART Inputs
| Signal | Width | Description |
|--------|-------|-------------|
| EN_50MHz | 1 bit | Enables the internal clock path for 50MHz operation. Used in baud rate generation. If 0, internal 100MHz clock is used. |
| PENABLE | 1 bit | APB protocol signal: Enables the data phase transfer. |
| PCLK | 1 bit | APB clock input. All APB interface logic is synchronized to this clock. |
| PRESETn | 1 bit | Active-low asynchronous reset. Resets the internal state of the UART module. |
| PWRITE | 1 bit | APB control signal: High for write operations, low for read. |
| PSEL | 1 bit | APB select signal: High when peripheral is selected for an APB transaction. |
| TXEN | 1 bit | UART transmit enable: Enables baudrate generation in TX logic. |
| SPEN | 1 bit | UART serial port enable: Enables UART functionality globally. |
| RX | 1 bit | UART receive line input: Captures serial data from external source. |
| RXEN | 1 bit | UART receive enable: Enables baudrate generation in RX logic. |
| PWDATA | 32 bits | APB write data bus: Data/control info written to UART. |
| BAUD_SEL | 2 bits | Selects one of four predefined baud rates (9600, 19200, 57600, 115200). |
| uart_control | 2 bits | UART word length control (8, 16, 24, or 32 bits). |

UART Output Ports
| Signal | Width | Description |
|--------|-------|-------------|
| TXIF | 1 bit | Transmit interrupt flag / APB PREADY: Indicates readiness for APB transactions. |
| RXIF | 1 bit | Receive interrupt flag: Data has been received and is ready. |
| TX | 1 bit | UART transmit line output: Sends serial data externally. |
| PRDATA | 8 bits | APB read data output: Provides received data to APB master. |

UART Control Register  
Bit-level description:

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---| - | --- | - | - | --- | --- | - |
| DATA CONTROL | | SPEN | BAUDSEL | | RXEN | TXEN | CS |

- Bit 0 – Clock Select: Selects system clock for UART: 0 = 100 MHz, 1 = 50 MHz
- Bit 1 – TXEN (Transmit Enable): Enables the transmitter. Uses the selected baud rate.
- Bit 2 – RXEN (Receive Enable): Enables the receiver. Uses the selected baud rate.
- Bits 4:3 – BAUDSEL (Baud Rate Select): Selects baud rate: 00 = 9600, 01 = 19200, 10 = 57600, 11 = 115200
- Bit 5 – SPEN (Serial Port Enable): Enables the serial port. Must be set for TX/RX to function.
- Bits 7:6 – DATA CONTROL: Selects number of bytes per word: 00 = 1 byte, 01 = 2 bytes, 10 = 3 bytes, 11 = 4 bytes

### Transmitter
Baudrate Generator Logic
- baud_sel = 2’b00 (9600 baud): Full divider chain used: mod3 + 4×mod2 + mod7 + mod31
- baud_sel = 2’b01 (19200 baud): One mod2 is skipped; the rest of the divider chain is the same as 9600.
- baud_sel = 2’b10 (57600 baud): mod2 and mod3 stages are skipped.
- baud_sel = 2’b11 (115200 baud): Only one mod2, mod7, and mod31 stages are used.

Divider Chain Structure (9600):  
clk →[modu3] →[modu2] →[modu2] →[modu2] →[modu2] →[modu7] →[modu31] →baud

Selective bypassing of divider stages based on baud_sel enables different baud rates with minimal logic.

TXREG Block
- Loads data from PWDATA into TXREG when TXIF and write enable are high.
- 1-clock delay before TXIF is cleared.
- TXIF auto-reasserts for 8-bit mode.
- For multi-byte modes: TXIF depends on TSR Ready, uart_control match, and mode[2] = 0.
- TXREG to TSR en manages transfer handshake.

Shift Register Block
- FSM transmits bytes: TXREG[7:0], [15:8], [23:16], [31:24].
- On falling clk_ext: byte loaded into TX shift register.
- On rising clk: right-shift TX_shift_reg, output LSB.
- After 10 shifts (start + 8 data + stop), TSR returns to idle.
- TSR Ready indicates transmitter is idle.

### Receiver
Baud Rate Generator
- Fixed at 9600 baud; not configurable.

Edge Detector
- Detects start bit (logic 0) on RX line.
- Sets st_bit_detected high until shift is done.

UART Shift Register
- Shifts in 8 bits of serial data on rising baud clock.
- Uses st_bit_detected to start.
- Asserts shift_done after receiving full byte.

Checking Stop Bit
- Checks if stop bit is logic 1 after data reception.
- If valid: asserts stop_valid.
- If invalid: asserts pullup_en.

| Signal Name | Width | Description |
|-------------|-------|-------------|
| stop_valid | 1 bit | High when valid stop bit is detected. |
| pullup_en | 1 bit | High on framing error (invalid stop bit). |

RCREG
- Stores received data only if stop bit is valid.
- Asserts RXIF when data is ready to be read.
- Clears RXIF on read_en.
- Buffers received UART data.

Pull-Up Module
- Forces RX line high if stop bit is invalid.
- Uses tri-state buffers based on pullup_en.
- Restores idle condition on RX line.

### How to Use UART
UART is mapped to address 0x00010003. Configuration steps include:
- Enabling TX/RX via set uart with appropriate control byte.
- Computing the UART address using movu and addh.
- Writing the data to be transmitted using st.

Example:
```
UART : ; UART address is 0x00010003
org 0x00000000:
set uart 1,2,5,6,7
; TXEN = 1, control = 11 (32-bit), RX enabled
movu r1,0x0001
addh r1,r1,0x0003
; UART address = 0x00010003
mov r2,0xF0F0
; Data to send
st r2,0[r1]
; Send data via UART
```

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Framing error | Invalid stop bit | Check baud rate match |
| Data loss | Buffer overflow | Service RXIF promptly |

---
